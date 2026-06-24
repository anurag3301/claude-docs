# Day 15 — Process States & Lifecycle

> **Estimated read time:** 90 minutes  
> **Goal:** Understand every state a process can be in, the complete exit path, zombie reaping, signal-based stops, and the orphan/subreaper model.

---

## 1. The State Machine

A process moves through a well-defined set of states during its lifetime:

```
                     fork()
                        │
                        ▼
                   TASK_NEW (just created, not yet scheduled)
                        │
              wake_up_new_task()
                        │
                        ▼
              ┌── TASK_RUNNING ──────────────────────────────┐
              │   (on run queue or on CPU)                    │
              │                                               │
              │   schedule()          schedule()              │
              │   ↓ (blocked)         ↓ (preempted)          │
              │                                               │
        TASK_INTERRUPTIBLE      TASK_UNINTERRUPTIBLE          │
        (woken by signal        (D state: must wait for I/O   │
         or event)              CANNOT be woken by signal)    │
              │                        │                      │
         signal or              event completes               │
         wait_event done             │                        │
              │                      │                        │
              └──────────────────────┘                        │
                                                              │
              SIGSTOP / ptrace              SIGCONT           │
              ───────────────→ __TASK_STOPPED ──────────────► │
              ptrace TRAP      __TASK_TRACED                   │
                                                              │
              do_exit()                                        │
              ──────────────────────────────────────────────► ▼
                                                        EXIT_ZOMBIE
                                                     (task_struct still alive)
                                                              │
                                                       parent wait()
                                                              │
                                                              ▼
                                                        EXIT_DEAD
                                                     (task_struct freed)
```

---

## 2. State Details

### 2.1 `TASK_RUNNING` (R)

The task is either:
- On a CPU running right now, OR
- On a run queue waiting for CPU time

**Both cases use the same state value** — you can't tell from the state whether it's actually executing. The `task_struct::on_cpu` field distinguishes: `1` = currently executing, `0` = queued.

```c
// Check if task is on a CPU right now:
bool executing = task->on_cpu;  // 1 = running, 0 = runnable but waiting
```

### 2.2 `TASK_INTERRUPTIBLE` (S)

The most common sleeping state. The task is waiting for:
- An event (disk I/O, network data, mutex, timer)
- And can be interrupted by any signal

```c
// Setting interruptible sleep (inside wait_event_interruptible):
set_current_state(TASK_INTERRUPTIBLE);
if (!condition)
    schedule();  // Actually sleep; wake up when condition changes or signal arrives

// After wake: check if woken by signal or condition:
if (signal_pending(current))
    return -ERESTARTSYS;  // Interrupted by signal
```

### 2.3 `TASK_UNINTERRUPTIBLE` (D)

"Disk sleep" — waiting for I/O that cannot be interrupted. CANNOT be killed even with `kill -9`.

Why it exists: if a process is in the middle of a non-interruptible operation (e.g., updating filesystem metadata), killing it mid-operation would leave the FS in a corrupt state.

```c
// Example: waiting for page I/O (not interruptible):
set_current_state(TASK_UNINTERRUPTIBLE);
schedule();  // Will only wake when I/O completes

// D state processes are dangerous in large numbers:
# Count D-state processes:
ps aux | awk '$8 ~ /^D/ {count++} END {print count, "D-state processes"}'

# D state often means:
# - NFS mount hung (server unreachable)
# - Hung disk/SAN
# - Kernel bug in driver
```

### 2.4 `__TASK_STOPPED` (T) and `__TASK_TRACED` (t)

Stopped by `SIGSTOP` or `SIGTSTP` (Ctrl+Z), or being traced by ptrace.

```bash
# Stop a process:
kill -STOP $PID    # SIGSTOP: cannot be caught or ignored
kill -TSTP $PID    # SIGTSTP: can be caught (terminal stop, e.g. Ctrl+Z in shell)

# Resume:
kill -CONT $PID    # SIGCONT

# ptrace trace (what strace/gdb does):
# strace attaches → sets TASK_TRACED
# Each syscall entry/exit: task stops, strace reads state, resumes task
```

### 2.5 `TASK_KILLABLE`

A hybrid: like `TASK_UNINTERRUPTIBLE` but wakes for fatal signals (kill -9):

```c
// kernel/sched/core.c
set_current_state(TASK_KILLABLE);
schedule();
if (fatal_signal_pending(current))
    return -EINTR;  // Was killed
```

Used where we want to be mostly uninterruptible (not interrupted by SIGTERM/SIGHUP) but still allow SIGKILL to work. Example: `wait_event_killable()`.

---

## 3. `do_exit()` — The Exit Path

When a process calls `exit()` (or is killed), `do_exit()` in `kernel/exit.c` runs:

```c
void __noreturn do_exit(long code)
{
    struct task_struct *tsk = current;
    int group_dead;
    
    WARN_ON(irqs_disabled());  // Should not be called with IRQs disabled
    
    // Prevent recursive exit:
    if (unlikely(tsk->flags & PF_EXITING)) {
        pr_alert("Fixing recursive fault but reboot is needed!\n");
        tsk->flags |= PF_EXITPIDONE;
        set_current_state(TASK_UNINTERRUPTIBLE);
        schedule();
    }
    
    // Mark as exiting (prevent signals from being delivered):
    exit_signals(tsk);
    
    // Accounting and profiling:
    acct_update_integrals(tsk);
    
    // Release mm (decrement mm_users; if 0, free the mm):
    exit_mm();
    
    // Close all open files (calls close() on each fd):
    exit_files(tsk);
    
    // Notify filesystem that we're going away:
    exit_fs(tsk);
    
    // Release IPC resources:
    sem_exit(tsk);
    
    // Exit cgroups:
    cgroup_exit(tsk);
    
    // Exit namespaces (decrement refcounts):
    exit_task_namespaces(tsk);
    
    // Exit work queue state:
    exit_task_work(tsk);
    
    // Timer cleanup:
    exit_itimers(tsk);
    
    // Set exit code:
    tsk->exit_code = code;
    
    // Reparent children (orphan handling):
    exit_notify(tsk, group_dead);
    // This:
    //   1. Reparents children to init/subreaper
    //   2. Sends SIGCHLD to parent
    //   3. For threads: if last in group, marks thread group as dead
    
    // Scheduler cleanup — this task will not run again:
    do_task_dead();  // Sets state EXIT_DEAD, calls schedule()
    // Never returns
}
```

---

## 4. Zombie Processes

After `do_exit()`, the `task_struct` is NOT immediately freed. Instead:

1. The process enters `EXIT_ZOMBIE` state
2. It sends `SIGCHLD` to its parent
3. It waits for the parent to call `wait()`

**Why?** The parent might want to read the exit code and resource usage. The `task_struct` is kept alive to hold this information.

```bash
# Create a zombie:
# (In bash)
bash -c 'sleep 100 &' &   # starts sleep in background, bash exits
ps aux | grep 'sleep 100'   # shows zombie '[sleep] <defunct>'
# or:
( sleep 10 & wait -n )    # Start child, don't wait for it

# Zombies appear as:
ps aux | grep Z
# Z = EXIT_ZOMBIE state
# <defunct> marker in the command column
```

A zombie uses:
- Its `task_struct` (~7KB)
- Its PID (until reaped)
- Nothing else — memory is freed in `exit_mm()` before becoming zombie

A process cannot be killed while zombie — it's already dead. Only reaping by the parent (or reparenting to init) clears it.

---

## 5. `wait()` / `wait4()` / `waitpid()` — Reaping

```c
// include/linux/wait.h
// Syscalls:
pid_t wait(int *status);                         // Wait for any child
pid_t waitpid(pid_t pid, int *status, int opts); // Wait for specific child
pid_t wait4(pid_t pid, int *status, int opts, struct rusage *r);

// Options:
WNOHANG    // Return immediately if no child has exited (non-blocking)
WUNTRACED  // Also return for stopped children
WCONTINUED // Also return for SIGCONT'd children
```

### 5.1 Kernel Implementation

```c
// kernel/exit.c
SYSCALL_DEFINE4(wait4, pid_t, upid, int __user *, stat_addr,
                int, options, struct rusage __user *, ru)
{
    // do_wait() does the actual work:
    // 1. Search children for ones in zombie/stopped state
    // 2. If found: collect exit code, free task_struct, return PID
    // 3. If not found + WNOHANG: return 0
    // 4. If not found + blocking: sleep in wait_queue until SIGCHLD
    
    return do_wait(&wo);
}

static long do_wait(struct wait_opts *wo)
{
    struct task_struct *tsk;
    
retry:
    child_wait_callback(wo);   // Check for matching children
    
    if (wo->notask_error)
        return wo->notask_error;
    
    // No child found — block:
    set_current_state(TASK_INTERRUPTIBLE);
    if (!wo->notask_error)
        schedule();  // Sleep until SIGCHLD wakes us
    
    set_current_state(TASK_RUNNING);
    goto retry;
}
```

### 5.2 Decoding Exit Status

```c
// User space: what wait() returns in status:
int status;
pid_t child = wait(&status);

if (WIFEXITED(status)) {
    int code = WEXITSTATUS(status);   // 0-255
    printf("exited normally with code %d\n", code);
}

if (WIFSIGNALED(status)) {
    int sig = WTERMSIG(status);       // signal number
    bool core = WCOREDUMP(status);    // core dump produced?
    printf("killed by signal %d%s\n", sig, core ? " (core dumped)" : "");
}

if (WIFSTOPPED(status)) {
    int sig = WSTOPSIG(status);       // stopping signal
}

if (WIFCONTINUED(status)) {
    // Was SIGCONT'd
}
```

---

## 6. Orphan Processes and Reparenting

When a parent dies before its children, the children become **orphans**. The kernel reparents them automatically.

### 6.1 Reparenting Logic

```c
// kernel/exit.c — called from do_exit():
static void exit_notify(struct task_struct *tsk, bool group_dead)
{
    // Find new parent for children:
    forget_original_parent(tsk, &dead);
    // This calls find_new_reaper() for each orphaned child
}

static struct task_struct *find_new_reaper(struct task_struct *father,
                                            struct task_struct *child_reaper)
{
    struct task_struct *thread;
    
    // First try: another thread in the same thread group
    thread = find_alive_thread(father);
    if (thread)
        return thread;
    
    // Second try: the subreaper (if any)
    // (a process that called prctl(PR_SET_CHILD_SUBREAPER, 1))
    
    // Last resort: PID 1 (init/systemd)
    return child_reaper;
}
```

### 6.2 Subreapers

A subreaper is a process that has registered to "adopt" orphan processes in its subtree:

```c
// Register as a subreaper (systemd does this):
prctl(PR_SET_CHILD_SUBREAPER, 1);

// Now: if any descendant process's parent dies, it gets reparented
// to this subreaper instead of PID 1

// Check if current process is a subreaper:
int is_subreaper;
prctl(PR_GET_CHILD_SUBREAPER, &is_subreaper);
```

This is critical for containerization: `runc` sets itself as subreaper before `exec()`-ing the container init, ensuring orphaned container processes don't escape to the host's PID 1.

### 6.3 Init and Zombies

`systemd` (PID 1) has special logic to periodically `waitpid()` for any adopted orphan children that become zombies. Without this, a long-lived system would accumulate zombie entries.

```bash
# Check PID 1's children (adopted orphans):
cat /proc/1/children   # PIDs of direct children of PID 1
ls /proc/1/task/1/children
```

---

## 7. Thread Group Exit

When a thread group (multithreaded process) exits, all threads must die:

```c
// Thread calls exit() or is killed:
do_exit():
    exit_signals():
        // If this is the last thread: mark whole group as dying
        // Otherwise: send SIGKILL to all other threads in the group
        if (atomic_dec_and_test(&current->signal->live)) {
            // Last thread standing — do full cleanup
        } else {
            // More threads exist: send SIGKILL to kill them
            zap_other_threads(current);
        }
```

```bash
# Kill an entire thread group (all threads):
kill -9 <tgid>       # Sends to the thread group (all threads get it)
# vs:
kill -9 <tid>        # Sends to a specific thread (usually same effect for SIGKILL)

# tkill sends to specific thread:
syscall(SYS_tkill, tid, sig)
# tgkill sends to thread in a specific group (safer):
syscall(SYS_tgkill, tgid, tid, sig)
```

---

## 8. `SIGCHLD` and the Child Status Notification

The exit notification to parent works via `SIGCHLD`:

```c
// kernel/exit.c
static void do_notify_parent(struct task_struct *tsk, int sig)
{
    struct siginfo info;
    
    info.si_signo = sig;   // SIGCHLD
    info.si_code  = tsk->exit_code & 0x7f  ? CLD_KILLED : CLD_EXITED;
    info.si_pid   = task_pid_nr_ns(tsk, ...);
    info.si_uid   = from_kuid_munged(..., task_uid(tsk));
    info.si_status = tsk->exit_code >> 8;  // Exit code (for CLD_EXITED)
    
    // Send to parent:
    send_signal(sig, &info, tsk->parent, PIDTYPE_TGID);
    
    // Wake up parent's wait() if it's sleeping:
    wake_up_parent(tsk);
}
```

### 8.1 `SIGCHLD` Handling Options

```c
// In the parent:
// Option 1: Synchronous wait (blocking):
waitpid(-1, &status, 0);

// Option 2: Non-blocking poll:
waitpid(-1, &status, WNOHANG);

// Option 3: Signal handler (async notification):
struct sigaction sa = {
    .sa_handler = sigchld_handler,
    .sa_flags   = SA_RESTART | SA_NOCLDSTOP,  // Don't notify for stops
};
sigaction(SIGCHLD, &sa, NULL);

void sigchld_handler(int sig) {
    int status;
    pid_t pid;
    // Must loop — multiple children might have exited:
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        handle_exit(pid, status);
    }
}

// Option 4: Ignore SIGCHLD (auto-reap — no zombies):
signal(SIGCHLD, SIG_IGN);
// With SIG_IGN: children become "detached" and are automatically reaped
// No zombie state, but you also can't get exit codes
```

---

## 9. Process Groups and Sessions

Linux has a hierarchy above the process: process groups and sessions, used for job control.

```
Session (SID = 5678)
├── Foreground Process Group (PGID = 9012)
│   ├── bash (PID=9012, PGID=9012) ← group leader
│   └── vim (PID=9013, PGID=9012)
│
└── Background Process Group (PGID = 9100)
    ├── sleep 100 (PID=9100, PGID=9100) ← group leader
    └── sleep 200 (PID=9101, PGID=9100)
```

```c
// Fields in task_struct:
// Accessed via:
pid_t pgid = task_pgrp_nr(task);   // Process group ID
pid_t sid  = task_session_nr(task); // Session ID

// Syscalls:
getpgrp()          // Get own PGID
setpgrp()          // Create own process group (PGID = PID)
getpgid(pid)       // Get PGID of process 'pid'
setpgid(pid, pgid) // Set process group

getsid(pid)        // Get session ID
setsid()           // Create new session (becomes session leader)
```

```bash
# See process groups and sessions:
ps -o pid,ppid,pgid,sid,comm
# PID   PPID   PGID    SID COMMAND
# 5678     1   5678   5678 bash         ← session leader
# 9012  5678   9012   5678 bash         ← subshell, new group
# 9013  9012   9012   5678 vim          ← same group as subshell

# Signals to process groups:
kill -TERM -9012    # Negative PID = PGID: kill the whole group
```

---

## 10. `/proc/<pid>/` — Observing Process Lifecycle

```bash
# Key /proc entries for lifecycle:
cat /proc/$PID/status        # State, PID, PPID, threads, memory, signals
cat /proc/$PID/stat          # Machine-readable: state, times, etc.
cat /proc/$PID/wchan         # What kernel function process is sleeping in
cat /proc/$PID/stack         # Kernel stack trace (CONFIG_STACKTRACE)
cat /proc/$PID/syscall       # Current or last syscall + arguments
cat /proc/$PID/fdinfo/0      # Info about fd 0 (stdin)

# For thread groups:
ls /proc/$PID/task/          # One subdir per thread (TID)
cat /proc/$PID/task/$TID/status  # Per-thread status

# Monitor state changes:
watch -n 0.5 "cat /proc/$PID/status | head -5"

# Stacks of all threads:
for TID in $(ls /proc/$PID/task/); do
    echo "--- Thread $TID ---"
    cat /proc/$PID/task/$TID/stack
done
```

---

## 11. Process Accounting

The kernel optionally records resource usage of every exiting process:

```c
// When configured (CONFIG_BSD_PROCESS_ACCT):
// At exit, kernel writes struct acct to a file (/var/log/pacct):
struct acct {
    char    ac_flag;     // Flags (F_FORK etc.)
    char    ac_version;  // Accounting version
    __u16   ac_uid16;    // Real UID (16-bit for compat)
    __u16   ac_gid16;    // Real GID
    __u16   ac_tty;      // Controlling terminal
    __u32   ac_btime;    // Creation time
    float   ac_utime;    // User CPU time
    float   ac_stime;    // System CPU time
    float   ac_etime;    // Elapsed wall-clock time
    float   ac_mem;      // Average memory usage
    __u32   ac_io;       // I/O in blocks
    __u32   ac_rw;       // Blocks read or written
    char    ac_comm[ACCT_COMM]; // Command name
};

// Enable/disable accounting:
syscall(SYS_acct, "/var/log/pacct");  // Enable
syscall(SYS_acct, NULL);              // Disable

// Analyze accounting data:
lastcomm                   # Recent commands
sa                         # Summary accounting
```

---

## 12. `prctl()` — Process Control

Many lifecycle-related settings are controlled via `prctl()`:

```c
#include <sys/prctl.h>

// Name the process (shows in ps, top):
prctl(PR_SET_NAME, "my_process_name");

// Get/set death signal (sent to children when THIS process dies):
prctl(PR_SET_PDEATHSIG, SIGTERM);

// Prevent core dumps:
prctl(PR_SET_DUMPABLE, 0);

// Set subreaper status:
prctl(PR_SET_CHILD_SUBREAPER, 1);

// Prevent setuid elevation:
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);

// Control seccomp:
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &filter);

// Timing:
prctl(PR_SET_TIMERSLACK, 1000000);  // 1ms timer slack

// CPU affinity hints:
prctl(PR_SCHED_CORE, PR_SCHED_CORE_CREATE, 0, PIDTYPE_PID, 0);
```

---

## 13. `/proc/sys/kernel/` Lifecycle Parameters

```bash
# PID limits:
cat /proc/sys/kernel/pid_max          # Max PID value (default 32768 or 4194304)
cat /proc/sys/kernel/threads-max      # Max threads system-wide
echo 4194304 > /proc/sys/kernel/pid_max  # Increase for large systems

# Core dump behavior:
cat /proc/sys/kernel/core_pattern     # Where core dumps go
# %e = executable name, %p = PID, %t = timestamp, %s = signal
echo '/tmp/cores/%e.%p.%t' > /proc/sys/kernel/core_pattern
cat /proc/sys/kernel/core_uses_pid    # Append PID to core file name

# Zombie collection:
# No sysctl — handled by PID 1's wait() calls

# Panic on kernel oops (for reliability):
cat /proc/sys/kernel/panic_on_oops    # 0=continue, 1=panic
cat /proc/sys/kernel/panic            # Reboot delay after panic (0=no reboot)
```

---

## 14. Tracing Process Lifecycle Events

```bash
# bpftrace: trace all exits with exit codes:
sudo bpftrace -e '
tracepoint:sched:sched_process_exit {
    printf("exit: pid=%d comm=%s code=%d\n",
        pid, comm, args->exit_code >> 8);
}'

# ftrace: trace do_exit calls:
echo do_exit > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace_pipe

# Watch zombies appear and disappear:
watch -n 1 "ps aux | awk '\$8==\"Z\"'"

# audit: log all process executions and exits:
auditctl -a always,exit -F arch=b64 -S execve  -k exec
auditctl -a always,exit -F arch=b64 -S exit_group -k exit
ausearch -k exec | tail -20
```

---

## 15. Mental Model Checkpoint

After Day 15, you should be able to:

1. Name all 6 process states and what triggers each transition.
2. Why does `TASK_UNINTERRUPTIBLE` exist? What does it protect against?
3. Trace `do_exit()` — name the 8 major cleanup steps.
4. What is a zombie and why isn't it immediately freed?
5. How does reparenting work? What is a subreaper?
6. Write a signal handler for `SIGCHLD` that correctly reaps all children.
7. What is the difference between a process group and a session?
8. What does `prctl(PR_SET_CHILD_SUBREAPER, 1)` do and why does `runc` use it?
9. How do you decode a wait() status value to get the exit code or signal?

---

## Key Source Files

```bash
kernel/exit.c             # do_exit(), exit_notify(), do_wait()
kernel/signal.c           # do_notify_parent(), SIGCHLD delivery
fs/proc/array.c           # /proc/<pid>/status, /proc/<pid>/stat
include/linux/sched.h     # TASK_* state constants
include/uapi/linux/wait.h # WNOHANG, WUNTRACED, W* macros
include/linux/prctl.h     # PR_SET_* constants
```

---

## Summary

Process lifecycle has four major phases:

1. **Creation** (Day 14): `fork()`/`clone()` → `copy_process()` → `wake_up_new_task()`
2. **Execution** (scheduling, Day 17): runs in `TASK_RUNNING` state, alternates with sleep
3. **Exit** (`do_exit()`): releases mm, files, fs, signal handlers, notifies parent via `SIGCHLD`
4. **Reaping** (`wait()`): parent collects exit status, kernel frees `task_struct`

The zombie state exists between exit and reaping — it's not a bug, it's a feature. The D state (TASK_UNINTERRUPTIBLE) is the kernel's way of saying "this process cannot be interrupted safely right now." Orphan reparenting ensures all processes have a parent that will eventually reap them.

Tomorrow: Linux threads — how `CLONE_VM | CLONE_THREAD` works internally and what it means for the scheduler.
