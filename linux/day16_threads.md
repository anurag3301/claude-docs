# Day 16 — Threads in Linux

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand exactly what a "thread" is at the kernel level, what `CLONE_*` flags share, how `task_struct` instances relate within a thread group, and what the kernel does differently for threads vs processes.

---

## 1. Linux Threads Are Not Special

The single most important thing to understand: **Linux has no separate "thread" abstraction**. Every thread is a full `task_struct`. What makes threads different from processes is what they *share* — controlled by `clone()` flags.

```
User perspective:          Kernel perspective:
                           
Process A                  task_struct A (pid=100, tgid=100)
  Thread 1 ─────────────► task_struct A.1 (pid=100, tgid=100) ← group_leader
  Thread 2 ─────────────► task_struct A.2 (pid=101, tgid=100)
  Thread 3 ─────────────► task_struct A.3 (pid=102, tgid=100)
```

All three task_structs have:
- Different `pid` (TID, kernel thread ID)
- Same `tgid` (= the PID user space sees = 100)
- Same `mm` pointer (share address space)
- Same `files` pointer (share file descriptors)
- Same `sighand` pointer (share signal handlers)
- Same `signal` pointer (share pending signals, process group, session)
- Different `stack` (each has its own kernel stack)
- Different `thread` struct (CPU register state, TLS)

---

## 2. `pthread_create()` Under the Hood

When glibc's `pthread_create()` is called, it calls `clone()` with specific flags:

```c
// glibc/nptl/pthread_create.c (simplified)
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg)
{
    // Allocate new stack for the thread:
    pd = allocate_stack(attr, &stackaddr, &stacksize);
    
    // Clone with thread flags:
    int flags = CLONE_VM        // Share address space
              | CLONE_FS        // Share filesystem state (cwd, root)
              | CLONE_FILES     // Share open file descriptors
              | CLONE_SIGHAND   // Share signal handlers
              | CLONE_THREAD    // Same thread group (same TGID)
              | CLONE_SETTLS    // Set thread-local storage
              | CLONE_PARENT_SETTID  // Write TID to parent's pthread_t
              | CLONE_CHILD_CLEARTID // Clear TID in futex on exit (join)
              | CLONE_SYSVSEM;  // Share SysV semaphore undo list
    
    pid_t tid = clone(start_thread,    // Thread entry function (glibc wrapper)
                      stackaddr + stacksize,  // Stack top
                      flags,
                      pd,              // Argument to start_thread
                      &pd->tid,        // parent_tidptr → CLONE_PARENT_SETTID
                      tp,              // TLS pointer → CLONE_SETTLS
                      &pd->tid);       // child_tidptr → CLONE_CHILD_CLEARTID
    
    *thread = (pthread_t)pd;           // Return pthread handle
    return 0;
}
```

The `start_thread` wrapper in glibc:
1. Calls `start_routine(arg)` (your function)
2. On return, calls `pthread_exit()` which calls `exit_thread()` → `sys_exit()` (not `sys_exit_group()`)

---

## 3. What Each `CLONE_*` Flag Means in Detail

### 3.1 `CLONE_VM` — Shared Address Space

Without `CLONE_VM` (fork): `copy_mm()` creates a new `mm_struct` with COW page tables.

With `CLONE_VM` (thread): `copy_mm()` simply increments `mm->mm_users` and sets `p->mm = current->mm`.

```c
// kernel/fork.c
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
    if (clone_flags & CLONE_VM) {
        // Share: just increment refcount
        mmget(current->mm);
        tsk->mm = current->mm;
        tsk->active_mm = tsk->mm;
        return 0;
    }
    // Fork: create COW copy
    tsk->mm = dup_mm(tsk, current->mm);
    // ...
}
```

Result: all threads see the same virtual address space. A `malloc()` in thread 1 is visible to thread 2.

### 3.2 `CLONE_FILES` — Shared File Descriptor Table

Without `CLONE_FILES` (fork): `copy_files()` creates a new `files_struct` with a copy of the file descriptor array (same `struct file *` pointers, different table).

With `CLONE_FILES` (thread): same `files_struct` pointer, `files->count` incremented.

```c
static int copy_files(unsigned long clone_flags, struct task_struct *tsk, ...)
{
    struct files_struct *oldf = current->files;
    if (clone_flags & CLONE_FILES) {
        atomic_inc(&oldf->count);
        tsk->files = oldf;  // Point to same table
        return 0;
    }
    // Fork: duplicate the fd table
    tsk->files = dup_fd(oldf, sysctl_nr_open, &error);
}
```

Result: if thread 1 opens a file, thread 2 sees it. If thread 1 closes fd 5, it's closed for thread 2 too.

### 3.3 `CLONE_SIGHAND` — Shared Signal Handlers

Without `CLONE_SIGHAND` (fork): `copy_sighand()` duplicates the `sighand_struct` (64 entries of `k_sigaction`).

With `CLONE_SIGHAND` (thread): same `sighand_struct` pointer, refcount incremented.

```c
static int copy_sighand(unsigned long clone_flags, struct task_struct *tsk)
{
    struct sighand_struct *sig;
    if (clone_flags & CLONE_SIGHAND) {
        refcount_inc(&current->sighand->count);
        tsk->sighand = current->sighand;
        return 0;
    }
    // Fork: copy the handlers table
    sig = kmem_cache_alloc(sighand_cachep, GFP_KERNEL);
    *sig = *current->sighand;  // Copy all 64 handlers
    tsk->sighand = sig;
}
```

### 3.4 `CLONE_THREAD` — Thread Group Membership

This is what makes a task a "thread" rather than a "child process":

```c
static int copy_signal(unsigned long clone_flags, struct task_struct *tsk)
{
    struct signal_struct *sig;
    
    if (clone_flags & CLONE_THREAD) {
        // Share signal_struct: same thread group, same TGID
        tsk->signal = current->signal;
        tsk->tgid = current->tgid;  // Same process group ID
        tsk->group_leader = current->group_leader;
        
        // Add to thread group list:
        list_add_tail_rcu(&tsk->thread_node,
                          &tsk->signal->thread_head);
        
        // Increment thread count:
        atomic_inc(&tsk->signal->live);
        atomic_inc(&tsk->signal->nr_threads);
        return 0;
    }
    
    // Fork: new signal_struct = new thread group = new "process"
    sig = kmem_cache_zalloc(signal_cachep, GFP_KERNEL);
    tsk->tgid = tsk->pid;  // New process: TGID = own PID
    tsk->group_leader = tsk;
    // ...
}
```

**Key implication**: `CLONE_THREAD` requires `CLONE_SIGHAND` (they must share handlers if in the same thread group).

### 3.5 `CLONE_SETTLS` — Thread-Local Storage

Sets the base address of the FS or GS segment register for x86_64 TLS:

```c
// arch/x86/kernel/process_64.c
int do_arch_prctl_64(struct task_struct *task, int option, unsigned long arg2)
{
    switch (option) {
    case ARCH_SET_FS:
        // Set FS.base (used for TLS in 64-bit)
        task->thread.fsbase = arg2;
        if (task == current)
            wrmsrl(MSR_FS_BASE, arg2);  // Write to MSR immediately
        break;
    }
}
```

Thread-local variables (declared `__thread` in C, `thread_local` in C++) use FS:offset addressing on x86_64. Each thread gets its own TLS block with its own FS base.

```c
// Example TLS variable:
__thread int my_count = 0;

// Thread 1:
my_count = 42;  // Stored at FS:offset_of_my_count
// Thread 2:
printf("%d\n", my_count);  // Reads its OWN my_count (0) at its own FS base
```

---

## 4. Thread-Specific State in `task_struct`

Even though threads share most of `task_struct`'s pointed-to data, each thread has its own:

```c
struct task_struct {
    // PER-THREAD (not shared):
    pid_t pid;                  // Thread ID (TID)
    void *stack;                // Kernel stack (16KB)
    struct thread_struct thread; // CPU state: fsbase, gsbase, FPU state
    sigset_t blocked;           // Signal mask (per-thread)
    struct sigpending pending;  // Per-thread pending signals
    unsigned long sas_ss_sp;   // Signal alt stack (per-thread)
    u64 utime, stime;          // CPU time (per-thread)
    u64 nvcsw, nivcsw;         // Context switches (per-thread)
    
    struct sched_entity se;    // CFS scheduling entity (per-thread)
    int prio;                  // Scheduling priority (per-thread)
    
    // SHARED (via pointer):
    struct mm_struct    *mm;       // Address space
    struct files_struct *files;    // File descriptors
    struct fs_struct    *fs;       // Filesystem info
    struct sighand_struct *sighand; // Signal handlers
    struct signal_struct  *signal; // Thread group info
    // ...
};
```

---

## 5. TID vs PID — The Confusion

The kernel's TID (`task_struct::pid`) and the user-visible PID are different things:

```bash
# In shell (sees TGID as "PID"):
ps -T -p $(pgrep -n mysqld)   # -T shows threads
#   PID    SPID COMMAND
# 12345   12345 mysqld       ← TGID = TID = 12345 (main thread)
# 12345   12346 mysqld       ← TGID = 12345, TID = 12346 (worker thread)
# 12345   12347 mysqld       ← TGID = 12345, TID = 12347

# PID vs TID in C:
#include <sys/types.h>
#include <unistd.h>
#include <sys/syscall.h>

pid_t pid = getpid();          // Returns TGID (process PID)
pid_t tid = syscall(SYS_gettid); // Returns TID (thread ID)
// For main thread: pid == tid
// For other threads: pid != tid
```

```c
// Kernel perspective:
task->pid   // TID
task->tgid  // Thread Group ID (= user-visible PID)

// Kernel macros:
task_pid_nr(task)   // Returns task->pid (TID)
task_tgid_nr(task)  // Returns task->tgid (user PID)
```

---

## 6. Signal Delivery to Thread Groups

Signal delivery to a thread group is complex because of shared signal handling:

```c
// Signal types:
// 1. Process-directed (to tgid): can be delivered to ANY thread in the group
//    kill(pid, sig)  → stored in signal->shared_pending
// 2. Thread-directed (to tid): delivered to THAT specific thread
//    tkill(tid, sig) → stored in task->pending

// kernel/signal.c: choosing which thread gets a process-directed signal:
static struct task_struct *find_any_eligible_task(struct signal_struct *signal, ...)
{
    // Priority:
    // 1. A thread specifically waiting for this signal (sigwaitinfo)
    // 2. A thread with the signal unblocked
    // 3. The thread group leader (if eligible)
}
```

```bash
# Send to thread group (any thread can handle):
kill -SIGTERM $(pgrep mysqld)      # To TGID

# Send to specific thread (only that thread handles it):
kill -SIGTERM $(ps -T -p $(pgrep mysqld) | awk 'NR==3{print $2}')
# (tkill syscall: only delivers to specific TID)
```

---

## 7. Thread Exit and `CLONE_CHILD_CLEARTID`

`CLONE_CHILD_CLEARTID` enables the `pthread_join()` mechanism:

```c
// When clone() is called with CLONE_CHILD_CLEARTID:
// The kernel stores child_tidptr in task->clear_child_tid

// When the thread exits (kernel/fork.c):
static void mm_release(struct task_struct *tsk, struct mm_struct *mm)
{
    if (tsk->clear_child_tid) {
        // Write 0 to *clear_child_tid in userspace:
        put_user(0, tsk->clear_child_tid);
        
        // Wake up threads waiting in futex:
        do_futex(tsk->clear_child_tid, FUTEX_WAKE, 1, NULL, NULL, 0, 0);
    }
}
```

`pthread_join()` uses `futex(FUTEX_WAIT)` on this location. When the thread exits, the kernel clears it and wakes the joining thread.

```c
// pthread_join() conceptual implementation:
int pthread_join(pthread_t thread, void **retval)
{
    struct pthread *pd = (struct pthread *)thread;
    
    // Spin or wait for tid to become 0:
    while (atomic_load(&pd->tid) != 0) {
        // FUTEX_WAIT: sleep until pd->tid changes
        futex(&pd->tid, FUTEX_WAIT, pd->tid, NULL);
    }
    
    // Thread has exited; collect return value
    *retval = pd->result;
    return 0;
}
```

---

## 8. Kernel Threads vs User Threads

```c
// Kernel thread: created with kernel_thread(), has no mm
struct task_struct *kth = kthread_run(my_fn, data, "my_kthread");
// kth->mm == NULL (kernel thread uses active_mm from last user task)

// User thread: has mm, user stack, user code
// kth->mm != NULL, points to shared mm_struct with other threads
```

```bash
# Tell them apart in ps:
ps aux | grep '\['
# root       34  0.0  0.0      kworker/0:1-events  ← kernel thread (in brackets)
# root       35  0.0  0.0      ksoftirqd/0          ← kernel thread
```

---

## 9. Futexes — The Thread Synchronization Mechanism

Mutexes, condition variables, and barriers in user space (pthreads) are built on **futexes** (Fast Userspace muTEXes):

```c
// syscall: futex(uaddr, op, val, timeout, uaddr2, val3)

// FUTEX_WAIT: if *uaddr == val, sleep
// FUTEX_WAKE: wake up to val threads waiting on uaddr

// Uncontended mutex fast path (no syscall):
// Lock: CAS 0→1 in userspace. If succeeds: acquired (no kernel call).
// Unlock: set 0 in userspace. If waiters: call FUTEX_WAKE.

// Contended mutex:
// Lock: CAS 0→1 fails (mutex held) → FUTEX_WAIT(futex_addr, 1)
// Unlock: if waiters exist → FUTEX_WAKE(futex_addr, 1)
```

```c
// kernel/futex/core.c
SYSCALL_DEFINE6(futex, u32 __user *, uaddr, int, op, u32, val,
    const struct __kernel_timespec __user *, utime,
    u32 __user *, uaddr2, u32, val3)
{
    switch (cmd) {
    case FUTEX_WAIT:
        return futex_wait(uaddr, val, timeout);
        // Verifies *uaddr == val (atomic), then sleeps
        
    case FUTEX_WAKE:
        return futex_wake(uaddr, val);
        // Wakes up to val threads waiting on uaddr
        
    case FUTEX_REQUEUE:
        return futex_requeue(uaddr, flags, uaddr2, val, val2, ...);
        // pthread_cond_broadcast: wake some, requeue rest
    }
}
```

---

## 10. Thread-Local Data Structures

### 10.1 Kernel Per-Thread Data

Beyond `task_struct` fields, the kernel uses per-thread data in two ways:

```c
// 1. Direct task_struct fields (accessible via current):
current->blocked     // Signal mask
current->pending     // Pending signals  
current->thread.fsbase  // TLS base

// 2. task_work mechanism (callbacks run at syscall exit):
// Used by io_uring, perf, etc. to run callbacks in task context
int task_work_add(struct task_struct *task, struct callback_head *work, ...)
void task_work_run(void);  // Called at syscall exit
```

### 10.2 User TLS Layout

On x86_64, the TCB (Thread Control Block) layout:

```
FS:[0]      → pointer to TCB itself (self-reference)
FS:[8]      → dtv (Dynamic Thread Vector — for dynamic TLS)
FS:[0x28]   → stack canary (for stack overflow detection)
FS:[0x30]   → pointer to pthread_t (self)
...
FS:[offset] → thread-local variables
```

```bash
# Inspect TLS base for a thread:
sudo cat /proc/$(pgrep -n mysqld)/task/$(ls /proc/$(pgrep -n mysqld)/task/ | head -2 | tail -1)/status | grep fsbase
# (Not directly exposed in /proc, but accessible via PTRACE_GETREGSET)
```

---

## 11. Thread Safety in the Kernel

When writing kernel code that may run in concurrent threads, know what's protected:

```c
// Shared between all threads of a process — needs locking:
current->mm           // mm_struct: use mmap_lock
current->files        // files_struct: use file_lock or RCU
current->signal       // signal_struct: use siglock or tasklist_lock

// Per-thread — safe to access without locks from own thread:
current->blocked      // Own signal mask
current->pending      // Own pending signals
current->thread       // CPU state
```

---

## 12. `clone3()` — The Modern Interface

`clone3()` (Linux 5.3+) is a more extensible version of `clone()`:

```c
// include/uapi/linux/sched.h
struct clone_args {
    __aligned_u64 flags;      // CLONE_* flags
    __aligned_u64 pidfd;      // [out] pidfd for new process
    __aligned_u64 child_tid;  // [out] TID in child
    __aligned_u64 parent_tid; // [out] TID in parent
    __aligned_u64 exit_signal;
    __aligned_u64 stack;      // Stack address
    __aligned_u64 stack_size; // Stack size
    __aligned_u64 tls;        // TLS base
    __aligned_u64 set_tid;    // Specific TIDs (for checkpoint/restore)
    __aligned_u64 set_tid_size;
    __aligned_u64 cgroup;     // cgroup fd for new process
};

// Usage:
struct clone_args args = {
    .flags      = CLONE_VM | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD,
    .stack      = (uint64_t)stack,
    .stack_size = stack_size,
    .tls        = (uint64_t)tls_block,
};
pid_t tid = syscall(SYS_clone3, &args, sizeof(args));
```

Advantages over `clone()`:
- Can request a specific TID (for container checkpoint/restore with CRIU)
- Can set the cgroup for the new task atomically
- Extensible: new fields can be added as `size` grows

---

## 13. `unshare()` — Selective Namespace/Resource Split

`unshare()` allows a process to detach from shared resources without forking:

```c
// Create a new mount namespace for the current process:
unshare(CLONE_NEWNS);
// Now this process has its own mount namespace; changes don't affect parent

// Create a new network namespace:
unshare(CLONE_NEWNET);

// Detach from current file descriptor table:
unshare(CLONE_FILES);
// Now has a copy — closing fds doesn't affect other threads

// Used by: unshare(1) command, container runtimes
```

```bash
# User-space usage:
unshare --mount --pid --net bash  # New shell in isolated namespaces
# Inside this bash: mount, PID, and network are isolated
ip link show  # Only sees lo — completely isolated network
```

---

## 14. Thread Count Limits

```bash
# Per-process thread limit (ulimit):
ulimit -u        # Max user processes (all threads count!)
# On Ubuntu: 63363 by default

# System-wide limit:
cat /proc/sys/kernel/threads-max
# 2*(RAM_pages) typically

# Per-process limit (via RLIMIT_NPROC):
cat /proc/$$/limits | grep processes
# Max processes: 63363 (soft), 63363 (hard)

# Increase for high-thread-count applications:
echo 100000 > /proc/sys/kernel/threads-max
ulimit -u 100000   # (also needs /etc/security/limits.conf for persistence)
```

---

## 15. Debugging Thread Issues

```bash
# See all threads of a process:
ps -T -p $(pgrep mysqld)          # Linux ps with -T (threads)
top -H -p $(pgrep -n mysqld)      # top with threads
htop  # F2 → Display options → Show custom thread names

# Thread stack traces (GDB):
gdb -p $(pgrep -n mysqld)
(gdb) thread apply all bt          # Backtrace all threads

# Kernel-side thread stacks:
for tid in $(ls /proc/$(pgrep -n mysqld)/task/); do
    echo "=== TID $tid ==="
    cat /proc/$(pgrep -n mysqld)/task/$tid/wchan
    cat /proc/$(pgrep -n mysqld)/task/$tid/stack 2>/dev/null
done

# bpftrace: trace thread creation:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_clone {
    if (args->clone_flags & 0x10000) {  // CLONE_THREAD
        printf("thread created: pid=%d comm=%s\n", pid, comm);
    }
}'

# Count threads system-wide:
ps -eLf | wc -l
cat /proc/sys/kernel/threads-max
```

---

## 16. Mental Model Checkpoint

After Day 16, you should be able to:

1. Explain what makes a Linux "thread" different from a "process" at the `task_struct` level.
2. List all flags `pthread_create()` passes to `clone()` and what each shares.
3. What is the difference between `pid` and `tgid` in `task_struct`? How does user space see these?
4. How does `CLONE_CHILD_CLEARTID` enable `pthread_join()`? Trace the mechanism.
5. Explain `CLONE_SETTLS` — what does it set and where is TLS stored on x86_64?
6. Write a `futex(FUTEX_WAIT)` and `futex(FUTEX_WAKE)` pair and explain when the kernel returns immediately vs sleeps.
7. What is per-thread vs per-process in `task_struct`?
8. How does signal delivery work differently for thread-directed vs process-directed signals?
9. What does `unshare(CLONE_NEWNS)` do and how does it differ from creating a new namespace via `clone()`?

---

## Key Source Files

```bash
kernel/fork.c                    # copy_mm, copy_files, copy_signal, copy_thread
include/uapi/linux/sched.h       # CLONE_* flags
arch/x86/kernel/process_64.c     # TLS setup (copy_thread, do_arch_prctl)
kernel/futex/core.c              # futex() syscall implementation
nptl/pthread_create.c            # (glibc source) pthread_create → clone
include/linux/sched.h            # task_struct fields: thread_node, signal, sighand
kernel/signal.c                  # Thread-directed vs process-directed signal delivery
```

---

## Summary

Linux threads are `task_struct` instances that share key resources via pointer:

- **Address space** (`CLONE_VM`): same `mm_struct` → shared heap, globals, text
- **File descriptors** (`CLONE_FILES`): same `files_struct` → open/close affects all
- **Signal handlers** (`CLONE_SIGHAND`): same `sighand_struct` → `sigaction()` affects all
- **Thread group** (`CLONE_THREAD`): same `signal_struct` → same TGID, shared pending

Each thread keeps its own: TID (`pid`), kernel stack, signal mask, FPU state, TLS base, scheduling entity, CPU time counters.

The thread synchronization mechanism (`pthread_mutex`, `pthread_cond`) is built on **futexes** — a userspace-first mechanism that only calls into the kernel when there's actual contention. Most lock/unlock operations are pure userspace CAS operations.

Tomorrow: the CFS scheduler — how the kernel decides which thread runs next.
