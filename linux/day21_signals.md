# Day 21 — Signals

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand signal delivery internals, `sigaction`, real-time signals, the complete delivery path from kernel to user handler and back.

---

## 1. Signals: Asynchronous Software Interrupts

Signals are the kernel's mechanism for notifying a process of an event — a software interrupt that can arrive at any time. They originate from:

- **Hardware exceptions**: `SIGSEGV` (segfault), `SIGBUS`, `SIGFPE` (divide-by-zero), `SIGILL`
- **Terminal**: `SIGINT` (Ctrl+C), `SIGTSTP` (Ctrl+Z), `SIGQUIT` (Ctrl+\\)
- **Other processes**: `kill(pid, sig)`, `tkill(tid, sig)`
- **The kernel itself**: `SIGKILL` (when process is OOM-killed), `SIGALRM` (timer)
- **Child exit**: `SIGCHLD` (when a child changes state)

---

## 2. Signal Numbers

```c
// include/uapi/asm-generic/signal.h
#define SIGHUP    1   // Terminal hangup
#define SIGINT    2   // Interrupt (Ctrl+C)
#define SIGQUIT   3   // Quit (Ctrl+\) — generates core dump
#define SIGILL    4   // Illegal instruction
#define SIGTRAP   5   // Trace/breakpoint trap
#define SIGABRT   6   // Abort (from abort())
#define SIGBUS    7   // Bus error (alignment, non-existent memory)
#define SIGFPE    8   // Floating-point/arithmetic exception
#define SIGKILL   9   // Kill — CANNOT be caught or ignored!
#define SIGUSR1  10   // User-defined 1
#define SIGSEGV  11   // Segmentation violation
#define SIGUSR2  12   // User-defined 2
#define SIGPIPE  13   // Broken pipe (write to pipe with no readers)
#define SIGALRM  14   // Alarm (from alarm(), setitimer())
#define SIGTERM  15   // Termination (default kill command)
#define SIGSTKFLT 16  // Stack fault on coprocessor (rarely used)
#define SIGCHLD  17   // Child stopped or exited
#define SIGCONT  18   // Continue if stopped
#define SIGSTOP  19   // Stop — CANNOT be caught or ignored!
#define SIGTSTP  20   // Terminal stop (Ctrl+Z) — CAN be caught
#define SIGTTIN  21   // Terminal read from background process
#define SIGTTOU  22   // Terminal write from background process
#define SIGURG   23   // Urgent data on socket (OOB data)
#define SIGXCPU  24   // CPU time limit exceeded (RLIMIT_CPU)
#define SIGXFSZ  25   // File size limit exceeded (RLIMIT_FSIZE)
#define SIGVTALRM 26  // Virtual timer expired
#define SIGPROF  27   // Profiling timer expired
#define SIGWINCH 28   // Terminal window size changed
#define SIGIO    29   // I/O now possible (O_ASYNC)
#define SIGPWR   30   // Power failure
#define SIGSYS   31   // Bad system call (seccomp)

// Real-time signals: 32–64 (SIGRTMIN to SIGRTMAX)
// - Multiple instances can be queued (unlike standard signals)
// - Delivered in order (FIFO)
// - Carry a payload (sigval_t)
#define SIGRTMIN  32
#define SIGRTMAX  64
```

---

## 3. Signal Data Structures in `task_struct`

```c
// Per-thread:
struct task_struct {
    sigset_t blocked;              // Currently blocked signals (signal mask)
    sigset_t real_blocked;         // Saved mask (for sigtimedwait)
    sigset_t saved_sigmask;        // Saved for sigsuspend
    struct sigpending pending;     // Per-thread pending signals (from tkill)
    // ...
};

// Per-thread-group (shared among all threads):
struct signal_struct {
    struct sigpending shared_pending;  // Process-directed signals
    
    struct list_head thread_head;      // All threads in this group
    wait_queue_head_t wait_chldexit;   // For wait() on children
    
    int group_exit_code;               // Exit code if killing group
    struct task_struct *group_exec_task;  // Thread doing exec
    
    struct posix_cputimers posix_cputimers;   // POSIX timers
    
    // For job control:
    struct pid *pids[PIDTYPE_MAX];  // PGID, SID pids
    struct tty_struct *tty;         // Controlling terminal
    
    // Rlimits:
    struct rlimit rlim[RLIM_NLIMITS];
};

// Signal actions (shared between threads via sighand_struct):
struct sighand_struct {
    refcount_t count;
    struct k_sigaction action[_NSIG];  // One action per signal (64 entries)
    spinlock_t siglock;
    wait_queue_head_t signalfd_wqh;
};

// One signal action:
struct k_sigaction {
    struct sigaction sa;
    // sa.sa_handler: SIG_DFL, SIG_IGN, or function pointer
    // sa.sa_mask: signals to block during handler
    // sa.sa_flags: SA_RESTART, SA_ONESHOT, SA_SIGINFO, etc.
    // sa.sa_restorer: trampoline function for return from handler
};
```

---

## 4. Sending a Signal: `kill()` Path

```c
// SYSCALL_DEFINE2(kill, pid_t, pid, int, sig):
// kernel/signal.c

SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
    struct kernel_siginfo info;
    
    // Build siginfo:
    prepare_kill_siginfo(sig, &info);
    // info.si_signo = sig
    // info.si_errno = 0
    // info.si_code = SI_USER
    // info.si_pid = task_tgid_vnr(current)
    // info.si_uid = from_kuid_munged(...)
    
    return kill_something_info(sig, &info, pid);
}

static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
    if (pid > 0) {
        // Send to specific process (thread group):
        return kill_pid_info(sig, info, find_vpid(pid));
    }
    if (pid == 0) {
        // Send to all processes in same process group:
        return kill_pgrp_info(sig, info, task_pgrp(current));
    }
    if (pid == -1) {
        // Send to all processes (except init):
        return kill_proc_info_as_uid(sig, info, ...);
    }
    // pid < -1: send to process group abs(pid)
    return kill_pgrp_info(sig, info, find_vpid(-pid));
}
```

### 4.1 `send_signal_locked()` — The Core

```c
// kernel/signal.c
static int send_signal_locked(int sig, struct kernel_siginfo *info,
                               struct task_struct *t, enum pid_type type)
{
    struct sigpending *pending;
    struct sigqueue *q;
    
    // Check permission: can sender send to target?
    if (!si_fromuser(info) || kill_ok_as_cred(info, t))
        // OK
    else
        return -EPERM;
    
    // SIGKILL and SIGSTOP: always succeed, even if blocked:
    if (sig_fatal(t, sig))
        goto ret;
    
    // Decide: process-directed or thread-directed?
    pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending
                                     : &t->pending;
    
    // For standard signals 1-31: coalesce duplicates
    // (don't queue if already pending)
    if (legacy_queue(pending, sig))
        goto ret;  // Already pending — don't queue again
    
    // For RT signals (32-64): always queue
    q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit, sigqueue_flags);
    if (q) {
        list_add_tail(&q->list, &pending->list);
        // Copy siginfo:
        copy_siginfo(&q->info, info);
    }
    
    // Set the "pending" bit in the sigset:
    sigaddset(&pending->signal, sig);
    
    // Wake the target task:
    complete_signal(sig, t, type);
    
    return 0;
}
```

### 4.2 `complete_signal()` — Finding the Target Thread

```c
// For process-directed signals: choose which thread gets it
static void complete_signal(int sig, struct task_struct *p,
                             enum pid_type type)
{
    struct signal_struct *signal = p->signal;
    struct task_struct *t;
    
    // Can this task handle the signal right now?
    if (wants_signal(sig, p))
        t = p;
    else if (type == PIDTYPE_PID)
        // Thread-directed: only that thread
        goto out_set;
    else {
        // Try other threads in the group:
        t = signal->curr_target;
        while (!wants_signal(sig, t)) {
            t = next_thread(t);
            if (t == signal->curr_target)
                goto out_set;  // No thread can handle it
        }
        signal->curr_target = t;
    }
    
    // Set TIF_SIGPENDING on the chosen thread:
    signal_wake_up(t, sig == SIGKILL);
    // This sets TIF_SIGPENDING in thread_info
    // And wakes the task if it's sleeping (send_signal to TASK_INTERRUPTIBLE task)
    
out_set:
    sigaddset(&signal->shared_pending.signal, sig);
}
```

---

## 5. Signal Delivery: The Exit-to-Userspace Path

Signals are delivered when a task is about to return to user space:

```c
// arch/x86/entry/common.c
// Called at the end of every syscall and interrupt that returns to user space:
static void exit_to_user_mode_loop(struct pt_regs *regs, u32 cached_flags)
{
    while (true) {
        local_irq_enable();
        
        if (cached_flags & _TIF_NEED_RESCHED)
            schedule();
        
        // Handle signals:
        if (cached_flags & _TIF_SIGPENDING)
            handle_signal_work(regs, cached_flags);
        
        // ... other work ...
        
        local_irq_disable();
        cached_flags = read_thread_flags();
        if (!(cached_flags & EXIT_TO_USER_MODE_WORK))
            break;
    }
}
```

### 5.1 `do_signal()` — The Delivery Function

```c
// arch/x86/kernel/signal.c
static void do_signal(struct pt_regs *regs)
{
    struct ksignal ksig;
    
    // Get next pending signal:
    if (get_signal(&ksig)) {
        // A signal is ready to deliver
        handle_signal(&ksig, regs);
        return;
    }
    
    // No signal: check if a system call should restart:
    if (syscall_get_error(current, regs) == -ERESTART_RESTARTBLOCK) {
        // Restart system call via restart_block
        regs->ax = get_nr_restart_syscall(regs);
        regs->ip -= 2;  // Back up to SYSCALL instruction
    }
}

static void handle_signal(struct ksignal *ksig, struct pt_regs *regs)
{
    struct k_sigaction *ka = &ksig->ka;
    
    // Case 1: SIG_DFL (default action):
    if (ka->sa.sa_handler == SIG_DFL) {
        default_signal_action(ksig->sig);
        // For SIGTERM/SIGHUP/SIGUSR1/etc.: do_exit(sig)
        // For SIGSTOP: do_signal_stop()
        // For SIGCONT: resume if stopped
        // For SIGIGN-equivalent signals: just return
        return;
    }
    
    // Case 2: User-space handler:
    setup_rt_frame(ksig, regs);
    // This function modifies pt_regs to "redirect" execution
    // to the signal handler in user space
}
```

---

## 6. Signal Frame: Redirecting to the Handler

`setup_rt_frame()` is how the kernel installs a signal handler call on the user stack:

```c
// arch/x86/kernel/signal.c
static int setup_rt_frame(struct ksignal *ksig, struct pt_regs *regs)
{
    // The signal frame on the user stack:
    struct rt_sigframe __user *frame;
    
    // Align the user stack:
    frame = get_sigframe(&ksig->ka, regs, sizeof(*frame));
    
    // Build the ucontext (saved CPU state):
    put_sigset_t(&frame->uc.uc_sigmask, &current->blocked);
    save_sigcontext(regs, &frame->uc.uc_mcontext, ...);
    // Saves: rip, rsp, rax, rbx, ... (all registers)
    // This is the state to restore when the handler returns
    
    // Put siginfo on the stack:
    copy_siginfo_to_user(&frame->info, &ksig->info);
    
    // Set up the trampoline return address:
    // The signal handler will "return" by calling sigreturn
    frame->pretcode = ksig->ka.sa.sa_restorer;
    // sa_restorer is a small piece of code in vdso or glibc:
    // mov rax, SYS_rt_sigreturn; syscall
    
    // Block signals during handler (as specified by sa_mask):
    set_current_blocked(&ksig->ka.sa.sa_mask);
    
    // Redirect execution to the handler:
    regs->ip = (unsigned long)ksig->ka.sa.sa_handler;  // Handler address
    regs->sp = (unsigned long)frame;                    // New stack pointer
    regs->di = ksig->sig;                               // First arg: signal number
    regs->si = (unsigned long)&frame->info;             // Second arg: siginfo
    regs->dx = (unsigned long)&frame->uc;               // Third arg: ucontext
    
    return 0;
}
```

After `setup_rt_frame()`:
```
The user-space execution will be:

signal_handler(sig, &siginfo, &ucontext):
    /* User's handler code */
    return;
    
    /* Return lands at sa_restorer: */
mov rax, SYS_rt_sigreturn
syscall         ← calls sys_rt_sigreturn()
```

### 6.1 `sys_rt_sigreturn()` — Return from Handler

```c
// kernel/signal.c / arch/x86/kernel/signal.c
SYSCALL_DEFINE0(rt_sigreturn)
{
    struct pt_regs *regs = current_pt_regs();
    struct rt_sigframe __user *frame;
    
    // Get the sigframe we built:
    frame = (struct rt_sigframe __user *)(regs->sp - sizeof(long));
    
    // Restore the signal mask:
    set_current_blocked(&frame->uc.uc_sigmask);
    
    // Restore the CPU state (all registers):
    restore_sigcontext(regs, &frame->uc.uc_mcontext);
    // Restores: rip (original code location), rsp, rax, ... 
    
    // Returns to the kernel → exits syscall → resumes original user code
    return regs->ax;  // Return value of interrupted syscall (if SA_RESTART)
}
```

---

## 7. Complete Signal Delivery Timeline

```
1. Process A calls kill(pid_B, SIGTERM)
   → send_signal_locked()
   → sigaddset(&B->signal->shared_pending.signal, SIGTERM)
   → TIF_SIGPENDING set on target thread in B

2. Target thread in B returns from syscall / interrupt
   → exit_to_user_mode_loop() checks _TIF_SIGPENDING
   → do_signal() called
   → get_signal(): dequeues SIGTERM from pending list
   → sa.sa_handler = user's handler function
   → setup_rt_frame():
     - Saves all registers to signal frame on user stack
     - Sets regs->ip = handler address
     - Sets regs->sp = signal frame

3. Return to user space: instead of original code, executes handler!
   → signal_handler(SIGTERM, &info, &uc)

4. Handler executes, calls return (or falls through to restorer):
   → sa_restorer: mov rax, SYS_rt_sigreturn; syscall

5. sys_rt_sigreturn():
   → Restores original registers from signal frame
   → Resumes original user code as if nothing happened
```

---

## 8. `sigaction()` — Registering Handlers

```c
#include <signal.h>

struct sigaction sa;

// Simple handler:
sa.sa_handler = my_handler;        // void (*)(int)
sigemptyset(&sa.sa_mask);          // Don't block additional signals
sa.sa_flags = 0;
sigaction(SIGTERM, &sa, NULL);

// SA_SIGINFO handler (receives siginfo_t and ucontext):
sa.sa_sigaction = my_sigaction;    // void (*)(int, siginfo_t *, void *)
sigemptyset(&sa.sa_mask);
sigaddset(&sa.sa_mask, SIGINT);    // Block SIGINT during this handler
sa.sa_flags = SA_SIGINFO | SA_RESTART;
sigaction(SIGSEGV, &sa, NULL);

// SA_* flags:
SA_RESTART      // Restart slow syscalls if interrupted
SA_ONESHOT      // Reset to SIG_DFL after one invocation
SA_NODEFER      // Don't block the signal during its own handler
SA_ONSTACK      // Use alternate signal stack (for stack overflow handler)
SA_NOCLDSTOP    // Don't send SIGCHLD when child stops (only on exit)
SA_NOCLDWAIT    // Don't create zombies (auto-reap children)
SA_RESETHAND    // Same as SA_ONESHOT
SA_SIGINFO      // Handler receives siginfo_t and ucontext_t
```

---

## 9. Signal Masks: `sigprocmask()`

```c
sigset_t mask, old_mask;

// Block SIGINT and SIGTERM:
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigaddset(&mask, SIGTERM);
sigprocmask(SIG_BLOCK, &mask, &old_mask);

// ... critical section where we don't want to be interrupted ...

// Restore original mask:
sigprocmask(SIG_SETMASK, &old_mask, NULL);

// Atomically wait for a signal with a temporary mask:
sigsuspend(&old_mask);  // Sleep until a signal arrives; restores mask on return

// Check for pending signals:
sigset_t pending;
sigpending(&pending);
if (sigismember(&pending, SIGINT))
    printf("SIGINT is pending\n");
```

```c
// Kernel implementation:
SYSCALL_DEFINE4(rt_sigprocmask, int, how, sigset_t __user *, nset,
                sigset_t __user *, oset, size_t, sigsetsize)
{
    sigset_t old_set = current->blocked;
    
    if (nset) {
        sigset_t new_set;
        copy_from_user(&new_set, nset, sizeof(new_set));
        
        switch (how) {
        case SIG_BLOCK:
            sigandsets(&current->blocked, &current->blocked, &new_set);
            // Wait: this is OR actually:
            sigorsets(&current->blocked, &current->blocked, &new_set);
            break;
        case SIG_UNBLOCK:
            signandsets(&current->blocked, &current->blocked, &new_set);
            break;
        case SIG_SETMASK:
            current->blocked = new_set;
            break;
        }
        
        // Can't block SIGKILL or SIGSTOP:
        sigdelsetmask(&current->blocked, sigmask(SIGKILL) | sigmask(SIGSTOP));
        
        // Recalculate pending signals after mask change:
        recalc_sigpending();
    }
    
    if (oset)
        copy_to_user(oset, &old_set, sizeof(old_set));
    
    return 0;
}
```

---

## 10. Real-Time Signals

RT signals (32–64) differ from standard signals in important ways:

```c
// Standard signals 1-31:
// - Only one pending per signal per target (duplicates dropped)
// - No guaranteed ordering
// - No payload beyond signal number

// RT signals 32-64:
// - Multiple instances can queue
// - Delivered in order: SIGRTMIN before SIGRTMIN+1, etc.
// - Carry a sigval_t payload (int or pointer)

// Send RT signal with payload:
union sigval value;
value.sival_int = 42;
sigqueue(pid, SIGRTMIN, value);

// Or via siginfo:
struct siginfo info = {
    .si_signo = SIGRTMIN,
    .si_code  = SI_QUEUE,
    .si_pid   = getpid(),
    .si_uid   = getuid(),
    .si_value = value,
};
syscall(SYS_rt_sigqueueinfo, pid, SIGRTMIN, &info);

// Handler receives the payload:
void rt_handler(int sig, siginfo_t *info, void *ctx)
{
    printf("got RT signal, value=%d\n", info->si_value.sival_int);
}

struct sigaction sa = {
    .sa_sigaction = rt_handler,
    .sa_flags     = SA_SIGINFO,
};
sigaction(SIGRTMIN, &sa, NULL);
```

---

## 11. `signalfd` — Signals via File Descriptor

`signalfd` converts signals into file descriptor read events — useful for event-driven programs using `epoll`:

```c
#include <sys/signalfd.h>

sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGTERM);
sigaddset(&mask, SIGINT);

// Block signals first (so they don't invoke handlers):
sigprocmask(SIG_BLOCK, &mask, NULL);

// Create signalfd:
int sfd = signalfd(-1, &mask, SFD_NONBLOCK | SFD_CLOEXEC);

// Add to epoll:
struct epoll_event ev = {
    .events  = EPOLLIN,
    .data.fd = sfd,
};
epoll_ctl(epfd, EPOLL_CTL_ADD, sfd, &ev);

// When a signal arrives:
struct signalfd_siginfo si;
read(sfd, &si, sizeof(si));
// si.ssi_signo = signal number
// si.ssi_pid = sender's PID
// si.ssi_uid = sender's UID
```

---

## 12. `sigaltstack` — Alternate Signal Stack

Required for `SIGSEGV` handlers that need to diagnose stack overflow:

```c
#include <signal.h>

#define SIGSTK_SIZE (64 * 1024)  // 64KB alternate stack

void *altstack = malloc(SIGSTK_SIZE);
stack_t ss = {
    .ss_sp    = altstack,
    .ss_size  = SIGSTK_SIZE,
    .ss_flags = 0,
};
sigaltstack(&ss, NULL);

struct sigaction sa = {
    .sa_sigaction = segfault_handler,
    .sa_flags     = SA_SIGINFO | SA_ONSTACK,  // Use alternate stack
};
sigaction(SIGSEGV, &sa, NULL);

void segfault_handler(int sig, siginfo_t *info, void *ctx)
{
    // Running on alternate stack now!
    // Can diagnose why we segfaulted without worrying about stack space
    ucontext_t *uc = (ucontext_t *)ctx;
    void *fault_addr = info->si_addr;
    void *ip = (void *)uc->uc_mcontext.gregs[REG_RIP];
    // ...
    _exit(1);
}
```

---

## 13. Hardware Exception Signals

When a CPU exception occurs (segfault, illegal instruction, etc.), the kernel translates it to a signal:

```c
// arch/x86/kernel/traps.c
DEFINE_IDTENTRY_ERRORCODE(exc_general_protection)
{
    // ...
    // Send SIGSEGV or SIGBUS to the faulting process:
    force_sig_fault(SIGSEGV, SEGV_MAPERR, (void __user *)address);
}

DEFINE_IDTENTRY(exc_invalid_op)
{
    // Illegal instruction → SIGILL
    force_sig_fault(SIGILL, ILL_ILLOPN, (void __user *)regs->ip);
}

// Page fault handler:
// arch/x86/mm/fault.c
static void __bad_area_nosemaphore(struct pt_regs *regs, ...)
{
    // Access violation → SIGSEGV
    force_sig_fault_to_task(SIGSEGV, si_code, address, tsk);
}
```

`force_sig_fault()` sends a signal that **cannot be blocked** — even if the process has blocked `SIGSEGV`, a page fault still delivers it (by temporarily removing it from the blocked mask).

---

## 14. `SIGKILL` and `SIGSTOP` — Unblockable Signals

```c
// SIGKILL: always kills the process, cannot be caught or blocked
// How: complete_signal() handles SIGKILL specially:
static void complete_signal(int sig, struct task_struct *p, ...)
{
    if (sig == SIGKILL || sig == SIGSTOP) {
        // Wake ALL threads in the group:
        for each thread t in group:
            signal_wake_up_state(t, sig == SIGKILL ? TASK_WAKEKILL : 0);
        return;
    }
}

// The process can't survive SIGKILL because:
// 1. Can't be blocked (sigdelsetmask removes it from blocked set)
// 2. Can't be caught (sa_handler set to SIG_DFL for SIGKILL)
// 3. Default action is terminate
// 4. Even if sleeping in D state, TASK_WAKEKILL wakes it
//    (most code uses wait_event_killable which checks fatal_signal_pending)

// The only way a process doesn't die from SIGKILL immediately:
// - It's in an uninterruptible wait (D state) that doesn't check SIGKILL
// - This is a bug in the driver/subsystem keeping it in that state
```

---

## 15. Tracing Signal Delivery

```bash
# strace: show signals:
strace -e signal=all ./myprogram

# See pending signals:
cat /proc/$$/status | grep -i sig
# SigBlk: 0000000000000000   ← blocked
# SigIgn: 0000000000001000   ← ignored (bit 12 = SIGPIPE)
# SigCgt: 0000000000000000   ← caught (has handler)
# SigPnd: 0000000000000000   ← thread-pending
# ShdPnd: 0000000000000000   ← process-pending

# Decode the hex mask:
python3 -c "
mask = 0x1000
for i in range(64):
    if mask & (1 << i):
        print(f'SIG{i+1}')
"
# SIG13 = SIGPIPE (ignored by default in many programs)

# bpftrace: trace signal sends:
sudo bpftrace -e '
tracepoint:signal:signal_generate {
    printf("signal %d → pid %d from pid %d\n",
        args->sig, args->pid, pid);
}'

# bpftrace: trace signal delivery:
sudo bpftrace -e '
tracepoint:signal:signal_deliver {
    printf("signal %d delivered to pid %d handler=0x%lx\n",
        args->sig, pid, args->sa_handler);
}'
```

---

## 16. Mental Model Checkpoint

After Day 21, you should be able to:

1. Trace a `kill(pid, SIGTERM)` from syscall to the target process receiving it.
2. What does `setup_rt_frame()` do to the user stack and to `pt_regs`?
3. How does a process "return" from a signal handler? What syscall does `sa_restorer` invoke?
4. What is the difference between standard and real-time signals?
5. Why can't `SIGKILL` be caught or blocked? How does the kernel enforce this?
6. What is `sigaltstack` and when is it required?
7. How does a hardware exception (segfault) become a `SIGSEGV` signal?
8. What does `SA_RESTART` do for a syscall interrupted by a signal?

---

## Key Source Files

```bash
kernel/signal.c                    # send_signal_locked(), do_signal(), get_signal()
arch/x86/kernel/signal.c          # setup_rt_frame(), sys_rt_sigreturn()
arch/x86/kernel/traps.c           # Exception → signal translation
include/linux/signal.h             # Signal data structure definitions
include/uapi/asm-generic/signal.h  # Signal numbers
kernel/sys.c                       # sigprocmask, sigpending implementations
```

---

## Summary

Signals travel a well-defined path:

1. **Generation**: `kill()` → `send_signal_locked()` → enqueue in `sigpending` + set `TIF_SIGPENDING`
2. **Selection**: at syscall/interrupt return, `get_signal()` dequeues the highest-priority pending signal
3. **Frame setup**: `setup_rt_frame()` saves registers to user stack, redirects `pt_regs` to handler
4. **Handler runs**: normal user-space code
5. **Return**: `sa_restorer` calls `sys_rt_sigreturn()` → restores original registers → original code resumes

The key architectural insight: signals are delivered *asynchronously* from the kernel's perspective but *synchronously* from the kernel code's perspective — they're checked at the defined "signal delivery points" (syscall/interrupt return), not in the middle of arbitrary kernel code.

Tomorrow: IPC mechanisms — pipes, message queues, shared memory, and semaphores.
