# Day 13 — `task_struct` Deep Dive

> **Estimated read time:** 90–120 minutes  
> **Kernel version referenced:** 6.x  
> **Goal:** Understand the process descriptor completely — every major field, what it tracks, and how the kernel uses it.

---

## 1. What `task_struct` Is

Every process and thread in the Linux kernel is represented by a `struct task_struct`. It is the kernel's complete record of a running entity — its identity, state, memory, open files, scheduling information, signal handling, and more.

```bash
# How many task_structs exist on your system right now:
ls /proc | grep '^[0-9]' | wc -l   # One per process
cat /proc/sys/kernel/pid_max        # Max possible: 32768 to 4194304
```

`task_struct` is defined in `include/linux/sched.h` and is one of the largest structs in the kernel — around 7–9 KB on x86_64 (it varies with config options). This entire day is about understanding what lives in those 9 KB and why.

---

## 2. The `current` Macro

Before looking at fields, understand how you access the running task's `task_struct`:

```c
// include/asm-generic/current.h (or arch-specific version)
#define current get_current()

// x86_64 implementation: stored in per-CPU variable
static __always_inline struct task_struct *get_current(void)
{
    return this_cpu_read_stable(pcpu_hot.current_task);
}
```

`current` is valid only in **process context** (syscalls, kernel threads). In hardirq context, `current` still points to the interrupted task — but it's meaningless for interrupt work.

```c
// Common uses:
current->pid          // PID of the running process
current->comm         // Process name (max 15 chars + null)
current->mm           // Memory descriptor
current->files        // Open file table
current->signal       // Signal handling
current->cred         // Credentials (uid, gid, capabilities)
```

---

## 3. Identity Fields

```c
struct task_struct {
    // ─── Identity ─────────────────────────────────────────────────────
    pid_t pid;              // Thread ID (TID): unique per thread
    pid_t tgid;             // Thread Group ID: PID visible to user space
                            // For single-threaded processes: pid == tgid
                            // For threads: pid = unique TID, tgid = thread group's PID
    
    char comm[TASK_COMM_LEN]; // Executable name (15 chars max + \0)
                               // Set by execve or prctl(PR_SET_NAME)
    
    // Pointer to the thread group leader's task_struct:
    struct task_struct *group_leader;
    // For single-threaded: group_leader == self
    // For threads: group_leader == the task that called pthread_create originally
```

```bash
# See pid vs tgid:
# ps shows TGID (user-visible PID), not TID
# Use /proc/<pid>/task/ to see all threads:
ls /proc/$(pgrep -n mysqld)/task/
# 12345/  12346/  12347/   ← each is a separate TID
# All share tgid = 12345

# TID directly:
cat /proc/$(pgrep -n mysqld)/task/12346/status | grep -E "Pid|Tgid"
# Pid:   12346    ← TID
# Tgid:  12345    ← TGID (= user-visible PID)
```

---

## 4. Process Relationships

```c
    // ─── Family Relationships ──────────────────────────────────────────
    struct task_struct __rcu *real_parent; // Biological parent (creator)
    struct task_struct __rcu *parent;      // Current parent (adoptive if reparented)
                                           // Differs from real_parent if parent died
                                           // and task was reparented to init/subreaper
    
    struct list_head children;   // List of child processes
    struct list_head sibling;    // Entry in parent's children list
    
    // Thread group:
    struct list_head thread_group;   // All threads in the same thread group
    struct list_head thread_node;    // Entry in signal->thread_head list
```

```bash
# Process family tree:
pstree -p $(pgrep sshd | head -1)
# sshd(1234)─┬─sshd(5678)───bash(9012)─┬─ps(1111)
#             │                          └─grep(2222)
#             └─sshd(6789)───bash(9013)

# In kernel (from process context):
struct task_struct *child;
list_for_each_entry(child, &current->children, sibling) {
    pr_info("child: pid=%d name=%s\n", child->pid, child->comm);
}
```

---

## 5. State Machine

```c
    // ─── Task State ────────────────────────────────────────────────────
    unsigned int __state;      // Current state
    
    // Valid states:
    // TASK_RUNNING         = 0x00000000  Ready or running on a CPU
    // TASK_INTERRUPTIBLE   = 0x00000001  Sleeping, woken by signal or event
    // TASK_UNINTERRUPTIBLE = 0x00000002  Sleeping, NOT woken by signal (D state)
    // __TASK_STOPPED       = 0x00000004  Stopped (SIGSTOP/ptrace)
    // __TASK_TRACED        = 0x00000008  Being traced by ptrace
    // TASK_PARKED          = 0x00000040  kthread parked
    // TASK_DEAD            = 0x00000080  Task is exiting (do_exit)
    // TASK_WAKEKILL        = 0x00000100  Wake on fatal signals
    // TASK_WAKING          = 0x00000200  Being woken up
    // TASK_NOLOAD          = 0x00000400  Don't count in load average
    // TASK_NEW             = 0x00000800  Task just forked, not scheduled yet
    // TASK_RTLOCK_WAIT     = 0x00001000  Waiting for RT mutex (PREEMPT_RT)
    
    // The exit_state field tracks post-exit state:
    int exit_state;
    // EXIT_ZOMBIE  = 0x00000010  Exited, waiting for parent to wait()
    // EXIT_DEAD    = 0x00000020  Being reaped
```

```c
// Setting state (must use the macros, not direct assignment):
set_current_state(TASK_INTERRUPTIBLE);  // Sets + memory barrier
__set_current_state(TASK_RUNNING);      // Sets without memory barrier (faster)

// Common pattern in blocking code:
prepare_to_wait(&wq_head, &wait, TASK_INTERRUPTIBLE);
if (!condition) {
    schedule();   // Actually sleep
}
finish_wait(&wq_head, &wait);
```

```bash
# See process states:
ps aux
# S = TASK_INTERRUPTIBLE (sleeping, woken by signal)
# D = TASK_UNINTERRUPTIBLE (disk sleep — waiting for I/O, cannot be killed!)
# R = TASK_RUNNING (running or runnable)
# T = TASK_STOPPED (stopped by signal)
# Z = EXIT_ZOMBIE (zombie — exited but not reaped)
# I = Idle kernel thread (not counted in load average)

# D state processes are often a sign of hung I/O:
ps aux | awk '$8=="D" {print}'
```

---

## 6. Scheduling Fields

```c
    // ─── Scheduler Fields ──────────────────────────────────────────────
    int prio;            // Effective priority (scheduler uses this)
    int static_prio;     // Nice-based priority: 100+nice (100–139 for normal tasks)
    int normal_prio;     // Normalized priority (considers RT policy)
    unsigned int rt_priority;  // Real-time priority: 1–99 (0 for normal tasks)
    
    const struct sched_class *sched_class;  // Pointer to scheduler class
    // Points to: fair_sched_class, rt_sched_class, dl_sched_class, idle_sched_class
    
    struct sched_entity se;    // CFS scheduling entity
    struct sched_rt_entity rt; // RT scheduling entity
    struct sched_dl_entity dl; // Deadline scheduling entity
    
    unsigned int policy;    // Scheduling policy:
    // SCHED_NORMAL = 0     (CFS)
    // SCHED_FIFO   = 1     (RT, no time slice)
    // SCHED_RR     = 2     (RT, time slice)
    // SCHED_BATCH  = 3     (CFS, batch workloads)
    // SCHED_IDLE   = 5     (CFS, very low priority)
    // SCHED_DEADLINE = 6   (EDF deadline scheduler)
    
    cpumask_t cpus_mask;    // CPUs this task is allowed to run on (CPU affinity)
    
    u64 utime, stime;       // User/system time in nanoseconds
    u64 gtime;              // Guest time (time in virtual machine)
    u64 start_time;         // Monotonic time when task started
    u64 start_boottime;     // Boottime when task started
```

```bash
# Inspect scheduling:
cat /proc/$$/sched      # Detailed CFS scheduling info for current shell

# Output includes:
# se.exec_start          : 12345678.901234
# se.vruntime            : 9876543.210987   ← CFS virtual runtime
# se.sum_exec_runtime    : 123456.789       ← Total CPU time
# nr_switches            : 12345            ← Context switches
# nr_voluntary_switches  : 11000
# nr_involuntary_switches: 1345

# Set CPU affinity:
taskset -c 0,1 myprogram    # Run on CPU 0 and 1 only
taskset -p -c 2-5 $$        # Set current shell to CPUs 2-5
```

---

## 7. Memory Management: `mm_struct`

```c
    // ─── Memory ────────────────────────────────────────────────────────
    struct mm_struct *mm;          // Process's memory descriptor
                                   // NULL for kernel threads!
    struct mm_struct *active_mm;   // Active mm (used by kernel threads borrowing
                                   // the address space of the last user task)
```

`mm_struct` is the complete description of a process's address space — covered deeply in Phase 3. Key fields you'll reference from `task_struct`:

```c
// Key mm_struct fields (for reference):
struct mm_struct {
    struct maple_tree  mm_mt;      // VMA tree (virtual memory areas)
    unsigned long      mmap_base;  // Base of mmap region
    unsigned long      task_size;  // Size of task virtual memory space
    pgd_t             *pgd;        // Top-level page table
    struct rw_semaphore mmap_lock; // Protects VMA tree
    atomic_t           mm_users;   // How many user processes use this mm
    atomic_t           mm_count;   // References to this mm (including lazy references)
    unsigned long      total_vm;   // Total virtual memory mapped (pages)
    unsigned long      locked_vm;  // Locked pages (mlock)
    unsigned long      start_code, end_code;   // .text segment bounds
    unsigned long      start_data, end_data;   // .data segment bounds
    unsigned long      start_brk,  brk;        // Heap bounds
    unsigned long      start_stack;            // Stack start
    unsigned long      arg_start, arg_end;     // Command-line arguments
    unsigned long      env_start, env_end;     // Environment variables
};
```

```bash
# Inspect a process's memory:
cat /proc/$$/maps              # Virtual memory areas
cat /proc/$$/status | grep Vm  # Memory statistics
# VmPeak:    123456 kB  ← Peak virtual memory
# VmSize:     98765 kB  ← Current virtual memory
# VmRSS:      45678 kB  ← Resident set size (physical pages)
# VmSwap:         0 kB  ← Pages in swap

# smaps gives per-VMA breakdown including private dirty pages:
cat /proc/$$/smaps | head -30
```

---

## 8. File System Context

```c
    // ─── Filesystem Context ────────────────────────────────────────────
    struct fs_struct *fs;      // Filesystem info: root dir, current dir
    struct files_struct *files; // Open file descriptors table
```

### 8.1 `fs_struct`

```c
// include/linux/fs_struct.h
struct fs_struct {
    int users;            // Reference count
    spinlock_t lock;
    seqcount_spinlock_t seq;
    int umask;            // File creation mask (umask)
    int in_exec;          // Currently executing a binary
    struct path root;     // Root directory (path = dentry + vfsmount)
    struct path pwd;      // Current working directory
};
```

```c
// Kernel access to current directory:
struct path cwd;
get_fs_pwd(current->fs, &cwd);   // Gets current->fs->pwd safely
// Use cwd.dentry, cwd.mnt
path_put(&cwd);
```

### 8.2 `files_struct`

```c
// include/linux/fdtable.h
struct files_struct {
    atomic_t count;           // Reference count
    bool resize_in_progress;
    wait_queue_head_t resize_wait;
    struct fdtable __rcu *fdt;  // Pointer to the fd table (RCU-protected)
    struct fdtable fdtab;       // Initial fd table (embedded for small fd counts)
    // ...
};

struct fdtable {
    unsigned int max_fds;
    struct file __rcu **fd;     // Array of file pointers: fd[0]=stdin, fd[1]=stdout...
    unsigned long *close_on_exec;   // Bitmask: close on exec
    unsigned long *open_fds;        // Bitmask: which fds are open
    // ...
};
```

```c
// Kernel: look up a file from an fd:
struct file *f = fget(fd);      // Increments refcount
if (!f) return -EBADF;
// use f...
fput(f);                        // Decrements refcount

// Iterate all open files of a process:
struct files_struct *files = task->files;
struct fdtable *fdt = files_fdtable(files);
unsigned int i;
rcu_read_lock();
for (i = 0; i < fdt->max_fds; i++) {
    struct file *f = rcu_dereference(fdt->fd[i]);
    if (f) {
        pr_info("fd %d: %pD\n", i, f);  // %pD prints the filename
    }
}
rcu_read_unlock();
```

```bash
# User-space view of open files:
ls -la /proc/$$/fd          # Symbolic links to open files
ls -la /proc/$$/fd | wc -l  # Count open fds
cat /proc/$$/limits | grep "open files"  # Current FD limit
```

---

## 9. Signal Handling

```c
    // ─── Signal Handling ───────────────────────────────────────────────
    struct signal_struct *signal;    // Shared among all threads in the group
    struct sighand_struct *sighand;  // Signal handlers (actions for each signal)
                                     // Shared across threads
    sigset_t blocked;               // Blocked signals (per-thread signal mask)
    sigset_t real_blocked;          // Real blocked mask (for sigtimedwait)
    sigset_t saved_sigmask;         // Saved for sigsuspend
    struct sigpending pending;       // Per-thread pending signal queue
    
    unsigned long sas_ss_sp;        // Signal alternative stack address
    size_t sas_ss_size;             // Signal alternative stack size
    unsigned int sas_ss_flags;
```

```c
// signal_struct: shared by all threads in the group
struct signal_struct {
    struct sigpending shared_pending;   // Process-directed signals (to any thread)
    struct list_head thread_head;       // List of threads
    int group_exit_code;               // Exit code if whole group is exiting
    // ...
    struct posix_cputimers posix_cputimers;   // POSIX CPU timers
    struct pid *pids[PIDTYPE_MAX];            // Various PID types
    // ...
};

// sighand_struct: signal handlers
struct sighand_struct {
    refcount_t count;
    struct k_sigaction action[_NSIG];  // Array of 64 signal actions
    spinlock_t siglock;
    wait_queue_head_t signalfd_wqh;
};
```

---

## 10. Credentials and Security

```c
    // ─── Credentials ───────────────────────────────────────────────────
    const struct cred __rcu *cred;   // Effective credentials (RCU-protected)
    const struct cred __rcu *real_cred;  // Real (objective) credentials
```

```c
// include/linux/cred.h
struct cred {
    atomic_t usage;
    kuid_t  uid;    // Real UID (from login)
    kgid_t  gid;    // Real GID
    kuid_t  suid;   // Saved UID (for setuid executables)
    kgid_t  sgid;   // Saved GID
    kuid_t  euid;   // Effective UID (what kernel checks for permissions)
    kgid_t  egid;   // Effective GID
    kuid_t  fsuid;  // UID for filesystem operations
    kgid_t  fsgid;  // GID for filesystem operations
    
    kernel_cap_t cap_inheritable;  // Capabilities that can be inherited
    kernel_cap_t cap_permitted;    // Permitted capabilities
    kernel_cap_t cap_effective;    // Currently effective capabilities
    kernel_cap_t cap_bset;         // Bounding set
    kernel_cap_t cap_ambient;      // Ambient capabilities
    
    struct user_namespace *user_ns;  // User namespace
    // ... SELinux context, AppArmor label, etc.
};
```

```c
// Kernel credential APIs:
uid_t uid = from_kuid(&init_user_ns, current_uid());   // Get UID
bool is_root = uid_eq(current_euid(), GLOBAL_ROOT_UID);

// Temporarily change credentials (e.g., in NFS code):
const struct cred *old_cred;
struct cred *override = prepare_creds();
override->fsuid = server_uid;
old_cred = override_creds(override);
// ... do filesystem operations as server_uid ...
revert_creds(old_cred);
put_cred(override);
```

```bash
# View process credentials:
cat /proc/$$/status | grep -E "^(Uid|Gid|CapEff)"
# Uid:    1000     1000     1000     1000
# ^real   ^eff     ^saved   ^fs
# CapEff: 0000000000000000  ← No extra capabilities
```

---

## 11. Kernel Stack and Thread Info

```c
    // ─── Stack ─────────────────────────────────────────────────────────
    void *stack;    // Pointer to the kernel stack
                    // The kernel stack is 16KB (THREAD_SIZE) per task
                    // Located immediately after the task_struct in memory
                    // (or separately with VMAP_STACK)
    
    // thread_info is embedded at the bottom of the kernel stack:
    // struct thread_info {
    //     unsigned long flags;   // TIF_* flags (signal pending, need resched, etc.)
    //     int preempt_count;     // Preemption depth
    //     u64 syscall_work;      // Syscall work flags (entry/exit hooks)
    // };
```

```c
// Access thread_info flags:
test_thread_flag(TIF_NEED_RESCHED)   // Scheduler wants to run something else
test_thread_flag(TIF_SIGPENDING)     // There are pending signals
test_thread_flag(TIF_SYSCALL_TRACE)  // ptrace is tracing syscalls

// Set/clear:
set_thread_flag(TIF_NEED_RESCHED);
clear_thread_flag(TIF_NEED_RESCHED);
```

The kernel stack layout:

```
High address:
  ┌──────────────────────────────┐  ← stack bottom (SP starts here)
  │  Stack frame: current func   │
  │  ...                         │
  │  Stack frame: syscall entry  │
  │  struct pt_regs             │  ← Saved user-mode registers
  ├──────────────────────────────┤
  │  Guard page (VMAP_STACK)    │  ← Unmapped, catches stack overflow
  │  [or direct mapping]         │
  └──────────────────────────────┘  ← Low address (task_struct or thread_info)

# Stack size: THREAD_SIZE = 16384 bytes on x86_64
```

```c
// Stack overflow detection:
// With CONFIG_VMAP_STACK=y: each thread's stack is vmalloc'd with a guard page
// Overflow hits the guard page → page fault in kernel = oops/panic

// Without VMAP_STACK: stack overflow silently corrupts adjacent memory (dangerous)

// Current stack usage:
unsigned long sp = current_stack_pointer;
unsigned long stack_base = (unsigned long)task_stack_page(current);
unsigned long used = (stack_base + THREAD_SIZE) - sp;
// used / THREAD_SIZE * 100 = % stack used
```

---

## 12. PID Namespace and PID Structure

```c
    // ─── PID/Namespace ─────────────────────────────────────────────────
    struct pid *thread_pid;            // Pointer to PID structure
    struct hlist_node pid_links[PIDTYPE_MAX];  // Links in PID hash tables
    
    struct nsproxy *nsproxy;           // Namespace proxy
    // nsproxy contains pointers to all namespaces this task is in:
    // - ipc_ns:  IPC namespace
    // - mnt_ns:  Mount namespace
    // - net_ns:  Network namespace
    // - pid_ns_for_children: PID namespace for child processes
    // - uts_ns:  UTS namespace (hostname, domainname)
    // - time_ns: Time namespace
    // - cgroup_ns: cgroup namespace
```

```c
// include/linux/pid.h
struct pid {
    refcount_t count;
    unsigned int level;           // Depth in PID namespace hierarchy
    spinlock_t lock;
    struct hlist_head tasks[PIDTYPE_MAX];   // Tasks with this PID
    struct hlist_head inodes;               // Inode list (for /proc)
    struct upid numbers[1];                 // Array of (nr, ns) pairs
};

// PID types:
enum pid_type {
    PIDTYPE_PID,    // TID (thread ID)
    PIDTYPE_TGID,   // TGID (thread group = process PID)
    PIDTYPE_PGID,   // Process group ID (for job control)
    PIDTYPE_SID,    // Session ID
    PIDTYPE_MAX,
};
```

```c
// Kernel PID lookups:
struct pid *pid = task_pid(task);           // Get PID struct
pid_t nr = task_pid_nr(task);              // Get numeric PID (in init namespace)
pid_t nr = task_pid_vnr(task);             // Get numeric PID (in task's namespace)
pid_t nr = task_tgid_nr(task);             // Get TGID
pid_t nr = task_pgrp_nr_ns(task, ns);      // Get PGID in namespace

// Find task by PID number:
struct pid *pid = find_vpid(nr);           // In current namespace
struct task_struct *t = pid_task(pid, PIDTYPE_PID);
```

---

## 13. Namespaces via `nsproxy`

```c
    struct nsproxy *nsproxy;
```

```c
// include/linux/nsproxy.h
struct nsproxy {
    refcount_t count;
    struct uts_namespace    *uts_ns;   // Hostname, NIS domain
    struct ipc_namespace    *ipc_ns;   // SysV IPC, POSIX mqueues
    struct mnt_namespace    *mnt_ns;   // Filesystem mounts
    struct pid_namespace    *pid_ns_for_children;  // PID namespace for new children
    struct net              *net_ns;   // Network stack instance
    struct time_namespace   *time_ns;  // Clock offsets
    struct cgroup_namespace *cgroup_ns; // cgroup namespace
};
```

Each namespace type allows the kernel to present a different view of the system to different process groups — the foundation of containers.

```bash
# See a process's namespaces:
ls -la /proc/$$/ns/
# lrwxrwxrwx 1 user user 0 Jan  1 ipc -> ipc:[4026531839]
# lrwxrwxrwx 1 user user 0 Jan  1 mnt -> mnt:[4026531840]
# lrwxrwxrwx 1 user user 0 Jan  1 net -> net:[4026531992]
# lrwxrwxrwx 1 user user 0 Jan  1 pid -> pid:[4026531836]
# lrwxrwxrwx 1 user user 0 Jan  1 uts -> uts:[4026531838]
# Numbers are inode numbers — same inode = same namespace

# Two processes in the same namespace share the inode number
```

---

## 14. cgroups

```c
    // ─── cgroups ───────────────────────────────────────────────────────
#ifdef CONFIG_CGROUPS
    struct css_set __rcu *cgroups;    // Pointer to cgroup subsystem state set
    struct list_head cg_list;         // Links into css_set's task list
#endif
```

```c
// css_set: one per unique combination of cgroup memberships
// Contains pointers to the subsystem state for each cgroup controller:
struct css_set {
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    // subsys[memory_cgrp_id]  → memory controller state
    // subsys[cpu_cgrp_id]     → CPU controller state
    // subsys[io_cgrp_id]      → I/O controller state
    // etc.
};
```

```bash
# See what cgroups a process belongs to:
cat /proc/$$/cgroup
# 0::/user.slice/user-1000.slice/session-1.scope

# With systemd cgroup v2:
systemd-cgls              # Full cgroup tree
systemctl status $$       # cgroup for current shell's service
```

---

## 15. CPU Time Accounting

```c
    // ─── Time Accounting ───────────────────────────────────────────────
    u64 utime;          // User-mode CPU time (nanoseconds)
    u64 stime;          // Kernel-mode CPU time (nanoseconds)
    u64 gtime;          // Guest time (in KVM virtual CPU)
    
    struct prev_cputime prev_cputime;  // Previous accounting interval
    
    u64 nvcsw;          // Voluntary context switches (task called schedule())
    u64 nivcsw;         // Involuntary context switches (scheduler preempted task)
    
    // High-resolution accounting (CONFIG_TASK_XACCT):
    u64 acct_rss_mem1;  // Accumulated RSS (for BSD accounting)
    u64 acct_vm_mem1;   // Accumulated VM
    u64 acct_timexpd;   // Expiration time for CPU accounting
```

```bash
# Task CPU time:
cat /proc/$$/stat
# Fields (man 5 proc):
# 14: utime — user mode jiffies
# 15: stime — kernel mode jiffies
# 16: cutime — children user time
# 17: cstime — children kernel time
# 42: delayacct_blkio_ticks — block I/O delay

# Convert jiffies to seconds: divide by HZ (usually 250 on servers)
# Or use /proc/$$/status for nanoseconds:
cat /proc/$$/status | grep -E "^(VmPeak|VmRSS|voluntary)"
```

---

## 16. Exit Information

```c
    // ─── Exit State ────────────────────────────────────────────────────
    int exit_code;          // Exit code passed to do_exit()
                            // Encodes signal + status: see WIFSIGNALED, WEXITSTATUS
    int exit_signal;        // Signal to send to parent when we die
                            // Normally SIGCHLD; threads use -1 (no signal)
    int pdeath_signal;      // Signal sent when parent dies (prctl PR_SET_PDEATHSIG)
    unsigned long jobctl;   // Job control flags (JOBCTL_STOP_*, JOBCTL_TRAP_*)
```

```c
// Exit code encoding (same as POSIX wait() status):
// Normal exit:    (exit_code << 8) | 0
// Killed:         0 | (signum & 0x7f)
// Core dumped:    0x80 | (signum & 0x7f)

// Macros to decode (from user space):
WIFEXITED(status)    // True if normal exit
WEXITSTATUS(status)  // Exit code (0-255)
WIFSIGNALED(status)  // True if killed by signal
WTERMSIG(status)     // Signal number
WIFSTOPPED(status)   // True if stopped
```

---

## 17. Key Kernel Functions That Use `task_struct`

```c
// kernel/fork.c — creating a new task_struct:
struct task_struct *copy_process(
    struct pid *pid,
    int trace,
    int node,
    struct kernel_clone_args *args);

// kernel/sched/core.c — context switch:
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
               struct task_struct *next, struct rq_flags *rf);

// kernel/exit.c — task is done:
void __noreturn do_exit(long code);

// kernel/signal.c — send a signal:
int send_signal_locked(int sig, struct kernel_siginfo *info,
                       struct task_struct *t, enum pid_type type);

// fs/proc/array.c — generating /proc/<pid>/status:
static int do_task_stat(struct seq_file *m, struct pid_namespace *ns,
                        struct pid *pid, struct task_struct *task, int whole);
```

---

## 18. Traversing All Processes

```c
// include/linux/sched/signal.h

// Iterate ALL processes (thread group leaders only):
struct task_struct *p;
for_each_process(p) {
    printk("pid=%d name=%s\n", p->pid, p->comm);
}

// Iterate ALL threads (every task_struct, including threads):
struct task_struct *p, *t;
for_each_process_thread(p, t) {
    printk("tgid=%d tid=%d name=%s\n", t->tgid, t->pid, t->comm);
}

// Iterate threads within one process:
struct task_struct *t;
for_each_thread(p, t) {
    printk("thread: tid=%d\n", t->pid);
}
```

These require `rcu_read_lock()` in production code and `tasklist_lock` for modifications:

```c
// Read the process list (RCU-safe read):
rcu_read_lock();
for_each_process(p) {
    // Access p safely
    // Do NOT sleep here
}
rcu_read_unlock();

// Modify the process list (requires tasklist_lock):
write_lock_irq(&tasklist_lock);
// ... add/remove task from lists ...
write_unlock_irq(&tasklist_lock);
```

---

## 19. The `task_struct` in Memory: Layout Optimization

`task_struct` frequently accessed fields are organized for cache performance:

```c
// The "hot" fields are at the beginning (most likely in L1 cache):
struct task_struct {
    struct thread_info        thread_info;  // FIRST: TIF_* flags, preempt count
    unsigned int              __state;      // Task state
    // ...
    struct sched_info         sched_info;   // Scheduling info
    struct list_head          tasks;        // All-processes list
    // ...
};
```

```bash
# See actual layout with pahole (BTF structure printer):
sudo apt install pahole
pahole -C task_struct /usr/lib/debug/boot/vmlinux-$(uname -r) | head -100
# Shows each field with its offset and size:
# struct task_struct {
#     struct thread_info         thread_info;          /*     0   256 */
#     unsigned int               __state;              /*   256     4 */
#     ...
#     /* size: 9536, cachelines: 149, members: 302 */
```

---

## 20. Mental Model Checkpoint

After Day 13, you should be able to:

1. Explain the difference between `pid` and `tgid` in `task_struct`.
2. What is `current` and why is it valid in process context but meaningless in NMI context?
3. Name 5 fields of `task_struct` and explain what each tracks.
4. What is the difference between `mm` and `active_mm`?
5. Draw the relationship between `task_struct`, `signal_struct`, and `sighand_struct` for a multithreaded process.
6. What is a `css_set` and how does cgroup membership get tracked?
7. How does the kernel safely traverse all processes? Which lock protects the list?
8. What does `nsproxy` contain and why does it have `pid_ns_for_children` rather than `pid_ns`?

---

## Key Source Files

```bash
include/linux/sched.h          # struct task_struct definition (MUST READ)
include/linux/mm_types.h       # struct mm_struct
include/linux/fs_struct.h      # struct fs_struct
include/linux/fdtable.h        # struct files_struct, fdtable
include/linux/cred.h           # struct cred
include/linux/pid.h            # struct pid, pid_type
include/linux/nsproxy.h        # struct nsproxy
kernel/fork.c                  # alloc_task_struct_node(), copy_process()
kernel/exit.c                  # do_exit()
fs/proc/array.c                # /proc/<pid>/status, stat generation
```

---

## Summary

`task_struct` is the kernel's complete dossier on every running entity. Its ~300 fields track:

- **Identity**: pid, tgid, comm, relationships (parent/children/threads)
- **State**: running/sleeping/zombie, exit state
- **Scheduling**: priority, policy, CFS entity (`se`), CPU affinity
- **Memory**: `mm_struct` pointer (virtual address space)
- **Files**: `fs_struct` (cwd/root) + `files_struct` (open file descriptors)
- **Signals**: pending signals, blocked mask, signal handlers
- **Security**: credentials (uid/gid/capabilities), LSM contexts
- **Namespace isolation**: `nsproxy` linking all 7 namespace types
- **cgroup membership**: `css_set` linking all resource controllers
- **Accounting**: CPU time, context switch counts, stack usage

Tomorrow: how `task_struct` gets created — the complete `fork()`/`clone()`/`exec()` lifecycle.
