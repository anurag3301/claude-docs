# Day 19 — Kernel Preemption

> **Estimated read time:** 90 minutes  
> **Goal:** Understand exactly when kernel code can be preempted, what prevents preemption, and how the preemption count governs scheduling decisions.

---

## 1. Three Preemption Models

The Linux kernel supports three preemption configurations, selected at compile time:

```
CONFIG_PREEMPT_NONE        (server default)
CONFIG_PREEMPT_VOLUNTARY   (desktop default, older kernels)
CONFIG_PREEMPT             (low-latency desktop)
CONFIG_PREEMPT_RT          (hard real-time — separate patchset, partially upstream)
```

```bash
# Check current kernel's preemption model:
cat /boot/config-$(uname -r) | grep -E 'PREEMPT|HZ='
# CONFIG_PREEMPT_VOLUNTARY=y
# CONFIG_HZ=250
```

---

## 2. `CONFIG_PREEMPT_NONE` — No Kernel Preemption

**Behavior**: Kernel code runs to completion unless it voluntarily calls `schedule()`. A kernel thread or syscall handler that enters a CPU-bound loop will hold the CPU until it exits or sleeps.

```
Kernel mode: runs without interruption
                    ↓
         Explicit schedule() call
                    ↓
         Or: interrupt return to user space → check TIF_NEED_RESCHED
```

**Used for**: servers where throughput matters more than latency. Fewer context switches, better cache utilization.

---

## 3. `CONFIG_PREEMPT_VOLUNTARY` — Voluntary Preemption

Adds explicit preemption points in long kernel paths via `might_resched()`:

```c
// Inserted at strategic points in long kernel loops:
void might_resched(void)
{
    if (need_resched())
        preempt_schedule();
}

// Example: VFS readahead loop checks might_resched()
// Example: filesystem journal commit
// Example: memory compaction

// Also: cond_resched() — calls schedule() if TIF_NEED_RESCHED:
cond_resched();   // Explicitly yield if needed
```

**Used for**: desktop systems needing reasonable interactivity without full preemption overhead.

---

## 4. `CONFIG_PREEMPT` — Full Kernel Preemption

**Behavior**: The kernel can be preempted at almost any point, including in the middle of a syscall or driver code, as long as the preemption count is zero.

```
Any kernel code point where preempt_count == 0:
    → Interrupt fires
    → irq_exit() checks TIF_NEED_RESCHED
    → If set: schedule() is called → current task preempted
```

This means: a high-priority task can preempt whatever is running in the kernel, as soon as the kernel is in a "preemptible" section.

---

## 5. The `preempt_count` Register

Every CPU has a `preempt_count` (stored in per-CPU hot data) that is a composite counter tracking all the reasons the kernel cannot be preempted:

```c
// include/linux/preempt.h
// preempt_count bits:
// [0:7]   = PREEMPT_MASK     (preempt_disable/enable depth)
// [8:15]  = SOFTIRQ_MASK     (in_softirq() depth / local_bh_disable depth)
// [16:19] = HARDIRQ_MASK     (in_irq() depth)
// [20]    = NMI_MASK         (in_nmi())
// [21]    = PREEMPT_NEED_RESCHED (set when resched needed, avoids atomic ops)

// Preemption is disabled when preempt_count != 0
#define preemptible() (preempt_count() == 0 && !irqs_disabled())
```

```c
// Operations that modify preempt_count:

// Explicit disable/enable (increments/decrements [0:7]):
preempt_disable();     // ++preempt_count[PREEMPT bits]
preempt_enable();      // --preempt_count[PREEMPT bits]; check resched

// spin_lock:
spin_lock(&lock);      // = preempt_disable() + acquire the lock
spin_unlock(&lock);    // = release + preempt_enable()

// local_bh_disable (softirq context):
local_bh_disable();   // ++preempt_count[SOFTIRQ bits]
local_bh_enable();    // --; if 0: run pending softirqs

// In hardirq handler:
irq_enter();           // ++preempt_count[HARDIRQ bits]
irq_exit();            // --; if 0 && pending softirqs: run them

// NMI:
nmi_enter();           // ++preempt_count[NMI bit]
nmi_exit();            // --
```

---

## 6. `preempt_disable()` and `preempt_enable()`

```c
// include/linux/preempt.h

#define preempt_disable()       \
do {                            \
    preempt_count_inc();        \
    barrier();                  \
} while (0)

#define preempt_enable()        \
do {                            \
    barrier();                  \
    if (unlikely(preempt_count_dec_and_test()))  \
        __preempt_schedule();   \
} while (0)
```

`preempt_count_dec_and_test()` returns true if the count just became zero AND `TIF_NEED_RESCHED` is set. In that case, `__preempt_schedule()` calls `schedule()`.

---

## 7. When Preemption Happens

On a `CONFIG_PREEMPT` kernel, preemption can happen at:

### 7.1 Interrupt Return Path

Every time an interrupt handler completes and returns to kernel mode:

```asm
/* arch/x86/entry/entry_64.S */
/* After hardirq handler completes: */
ret_from_intr:
    testl   $3, CS(%rsp)        /* Check CPL: did we interrupt user space? */
    jnz     retint_user         /* Yes: user-space return path */
    
    /* We interrupted kernel code: */
    DISABLE_INTERRUPTS(CLBR_ANY)
    
    /* Check preempt_count: */
    movl    PER_CPU_VAR(__preempt_count), %eax
    testl   %eax, %eax
    jnz     restore_regs_and_return  /* preempt_count != 0: can't preempt */
    
    /* Check TIF_NEED_RESCHED: */
    testl   $TIF_NEED_RESCHED, TI_flags(%rsp)
    jz      restore_regs_and_return  /* No resched needed */
    
    /* Preempt! */
    call    preempt_schedule_irq     /* Calls schedule() */
    jmp     ret_from_intr
```

### 7.2 `preempt_enable()` Return Path

When preemption is re-enabled and `TIF_NEED_RESCHED` is set:

```c
void preempt_enable(void)
{
    // Decrement and check atomically:
    if (unlikely(__preempt_count_dec_and_test()))
        __preempt_schedule();
}

void __preempt_schedule(void)
{
    do {
        preempt_disable();         // Protect against recursive preemption
        __schedule(SM_PREEMPT);    // Call scheduler with preempt flag
        sched_preempt_enable_no_resched();
    } while (need_resched());      // Loop: might need to resched again
}
```

---

## 8. `TIF_NEED_RESCHED` — The Resched Flag

```c
// When a higher-priority task becomes runnable, the scheduler sets this flag:
void resched_curr(struct rq *rq)
{
    struct task_struct *curr = rq->curr;
    
    if (test_tsk_need_resched(curr))
        return;  // Already set
    
    if (cpu == smp_processor_id()) {
        // Reschedule this CPU:
        set_tsk_need_resched(curr);   // Set TIF_NEED_RESCHED in thread_info
        set_preempt_need_resched();   // Set bit in preempt_count (fast path)
        return;
    }
    
    // Reschedule another CPU:
    set_tsk_need_resched(curr);
    smp_send_reschedule(cpu);         // IPI to wake up that CPU's scheduler
}
```

`TIF_NEED_RESCHED` is checked at:
1. Every interrupt return to kernel mode (if PREEMPT enabled)
2. Every `preempt_enable()` call
3. Every explicit `cond_resched()` call
4. Every `might_resched()` call (PREEMPT_VOLUNTARY)

---

## 9. Non-Preemptible Sections

```c
// Explicit preemption disable:
preempt_disable();
// ... critical section ...
preempt_enable();  // Re-enables; may call schedule()

// Spinlock (implicit preemption disable):
spin_lock(&lock);     // preempt_disable()
// ... critical section ...
spin_unlock(&lock);   // preempt_enable()

// Hardirq context:
// irq_enter() increments HARDIRQ bits → non-preemptible
void my_irq_handler(int irq, void *dev) {
    // preempt_count() != 0 here
    // Preemption is disabled until irq_exit()
}

// Softirq context:
// in_softirq() returns true → non-preemptible by other softirqs
static void net_rx_action(struct softirq_action *h) {
    // local_bh_disable() called: SOFTIRQ bits set
}
```

---

## 10. Preempt Count Debugging

```c
// Debug prints the preempt_count:
printk("preempt_count: %d (hw:%d sw:%d nmi:%d)\n",
    preempt_count(),
    hardirq_count() >> HARDIRQ_SHIFT,
    softirq_count() >> SOFTIRQ_SHIFT,
    nmi_count()     >> NMI_SHIFT);

// Assert preemption is disabled:
lockdep_assert_preemption_disabled();  // WARN if preemptible

// Assert preemption is enabled:
lockdep_assert_preemption_enabled();   // WARN if non-preemptible

// might_sleep(): warns if called from non-sleepable context:
might_sleep();  // Called at start of functions that may sleep (mutex, alloc...)
// If preempt_count != 0 or IRQs disabled → prints a warning stack trace
```

---

## 11. `might_sleep()` and `might_resched()`

```c
// In code that can sleep:
void my_blocking_function(void)
{
    might_sleep();     // Validate context: warns if called non-sleepably
    
    mutex_lock(&m);    // Safe: we're in a sleepable context
    // ...
    mutex_unlock(&m);
}

// In code with voluntary preemption points:
void long_kernel_loop(void)
{
    for (i = 0; i < 1000000; i++) {
        process_item(i);
        cond_resched();    // Yield if TIF_NEED_RESCHED is set
        // = if (need_resched()) schedule();
    }
}

// might_sleep() is conditionally compiled:
// CONFIG_DEBUG_ATOMIC_SLEEP: enabled → real WARN_ON
// Without it: might_sleep() compiles to might_resched()
```

---

## 12. Preemption and Locking Interaction

```c
// Rule: A task holding a spinlock cannot be preempted
// Because: spinlock calls preempt_disable()

// Rule: A task holding a mutex CAN be preempted
// Because: mutex doesn't disable preemption

spin_lock(&fast_lock);
// preempt_count = 1 → non-preemptible
some_work();
spin_unlock(&fast_lock);
// preempt_count = 0 → preemptible
// If TIF_NEED_RESCHED: schedule() called here

mutex_lock(&slow_lock);
// preempt_count = 0 → STILL preemptible
// A high-priority RT task can preempt us here (with CONFIG_PREEMPT)
some_work();
// ... get preempted if needed ...
mutex_unlock(&slow_lock);
```

This is why mutex-based code is safe with PREEMPT but spinlock-based code needs care with PREEMPT_RT (where spinlocks become sleeping).

---

## 13. The `idle` Task and Preemption

Each CPU has an idle task (`swapper/N`) that runs when nothing else is runnable:

```c
// kernel/sched/idle.c
static void do_idle(void)
{
    while (!need_resched()) {
        // cpuidle: enter low-power state
        local_irq_disable();
        arch_cpu_idle();  // HLT instruction on x86
        local_irq_enable();
        
        if (need_resched())
            break;
    }
    
    // Exit idle: run the scheduler
    schedule_idle();
}
```

The idle task has preempt_count = 0 at all times (it's always preemptible). An interrupt that makes another task runnable sets `TIF_NEED_RESCHED`, wakes the CPU from HLT, and idle calls `schedule()`.

---

## 14. Observing Preemption

```bash
# Count voluntary vs involuntary context switches:
cat /proc/$$/status | grep ctxt
# voluntary_ctxt_switches:   1234  ← called schedule() explicitly
# nonvoluntary_ctxt_switches: 56   ← preempted by higher-priority task

# System-wide:
vmstat 1 5
# cs column = context switches per second

# perf: measure context switch overhead:
sudo perf stat -e context-switches,cpu-migrations -a sleep 5
# context-switches    12345
# cpu-migrations         67   ← tasks moved between CPUs

# bpftrace: trace preemptions:
sudo bpftrace -e '
tracepoint:sched:sched_switch {
    if (args->prev_state == 0) {  // TASK_RUNNING = preempted (not sleeping)
        @[args->prev_comm] = count();  // Who got preempted
    }
}'
```

---

## 15. Mental Model Checkpoint

After Day 19, you should be able to:

1. What are the four preemption models and when is each used?
2. What is `preempt_count` and what bits does it contain?
3. When does the kernel check `TIF_NEED_RESCHED`?
4. What does `spin_lock()` do to `preempt_count`?
5. Can a task holding a mutex be preempted? Can a task holding a spinlock?
6. What is the interrupt return path preemption check?
7. What does `might_sleep()` do and why is it important?
8. How does the idle task get woken from HLT when a new task becomes runnable?

---

## Key Source Files

```bash
include/linux/preempt.h           # preempt_count, preempt_disable/enable macros
kernel/sched/core.c               # preempt_schedule(), resched_curr()
arch/x86/entry/entry_64.S        # ret_from_intr: kernel preemption check
include/linux/thread_info.h       # TIF_NEED_RESCHED, set_tsk_need_resched
kernel/sched/idle.c               # Idle task and do_idle()
Documentation/locking/preemption-locking.rst  # Official docs
```

---

## Summary

Kernel preemption is controlled by a single `preempt_count` value that accumulates depth from four sources: explicit `preempt_disable()` calls, spinlocks, hardirq handlers, and NMI. When this count is zero and `TIF_NEED_RESCHED` is set, the kernel schedules another task at the next safe preemption point.

The key insight: preemption is *not* about fairness — it's about latency. With `CONFIG_PREEMPT`, a high-priority RT task can preempt kernel code that's running on behalf of a low-priority task, as soon as that code reaches a preemptible point (preempt_count = 0).

Tomorrow: SMP — how per-CPU run queues work, and how the scheduler balances load across CPUs.
