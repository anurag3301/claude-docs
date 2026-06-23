# Day 10 — Interrupt Handling: Software Path

> **Estimated read time:** 90–120 minutes  
> **Goal:** Master softirqs, tasklets, workqueues, threaded IRQs, and the design philosophy behind deferred interrupt work.

---

## 1. The Problem: Long Work in Interrupt Context

Hardware interrupt handlers must be fast — they run with interrupts disabled (or at minimum with the same interrupt masked), blocking all other work on that CPU. A handler that takes 1ms is catastrophic at 1000 Hz timer frequency.

But real device work is non-trivial:
- A network card delivers a packet. Parsing the TCP/IP headers, finding the socket, copying to user buffer — that's hundreds of microseconds.
- A storage controller completes a disk read. Marking the page as uptodate, waking the waiting process, handling readahead — more complex still.

**Solution**: split interrupt work into two halves:

1. **Top half** (hardirq handler): runs immediately. Acknowledges the device, saves minimum state, schedules the bottom half. Returns in microseconds.

2. **Bottom half** (deferred): runs soon after, at a lower priority, with more flexibility. Can run with interrupts re-enabled.

Linux has three bottom-half mechanisms with different tradeoffs: **softirqs**, **tasklets**, and **workqueues**.

---

## 2. Softirqs — The Foundation

Softirqs are the kernel's lowest-level deferred mechanism. They run per-CPU, cannot sleep, but have interrupts *enabled* (unlike hardirq handlers).

### 2.1 The Fixed Softirq Table

Unlike the dynamic IRQ subsystem, softirqs are statically defined at compile time:

```c
// include/linux/interrupt.h
enum {
    HI_SOFTIRQ = 0,        // High-priority tasklets
    TIMER_SOFTIRQ,          // Timer wheel callbacks
    NET_TX_SOFTIRQ,         // Network packet transmission
    NET_RX_SOFTIRQ,         // Network packet receive (NAPI polling)
    BLOCK_SOFTIRQ,          // Block I/O completions
    IRQ_POLL_SOFTIRQ,       // I/O polling (replaces old BLOCK_IOPOLL)
    TASKLET_SOFTIRQ,        // Low-priority tasklets
    SCHED_SOFTIRQ,          // Scheduler (load balancing, RT pull)
    HRTIMER_SOFTIRQ,        // High-resolution timer callbacks
    RCU_SOFTIRQ,            // RCU callbacks (grace period processing)
    NR_SOFTIRQS             // Total count
};
```

These are the *only* softirqs. You cannot add new ones. If your subsystem needs deferred work, use tasklets (which build on `TASKLET_SOFTIRQ`/`HI_SOFTIRQ`) or workqueues.

### 2.2 Raising a Softirq

From hardirq context, you *raise* (schedule) a softirq:

```c
// Raise a softirq on the current CPU (to run soon):
raise_softirq(NET_RX_SOFTIRQ);

// Or, if you've already disabled interrupts:
raise_softirq_irqoff(NET_RX_SOFTIRQ);

// For network: NAPI scheduler uses:
__raise_softirq_irqoff(NET_RX_SOFTIRQ);
```

Raising sets a bit in the per-CPU `__softirq_pending` bitmask:

```c
// kernel/softirq.c
void raise_softirq_irqoff(unsigned int nr)
{
    __raise_softirq_irqoff(nr);  // or_softirq_pending(1UL << nr)
    
    // If not currently in interrupt context, trigger immediate processing
    if (!in_interrupt())
        wakeup_softirqd();
}
```

### 2.3 When Softirqs Run

Softirqs are processed at specific points:
1. At the tail of hardware interrupt handling (`irq_exit_rcu()`)
2. After `local_bh_enable()` (re-enabling bottom halves)
3. In the `ksoftirqd` kernel thread (for high-load conditions)

```c
// kernel/softirq.c — The softirq processing loop
static void __do_softirq(void)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;  // Don't run > 2ms
    int max_restart = MAX_SOFTIRQ_RESTART;           // Max 10 iterations
    
    pending = local_softirq_pending();  // Read the bitmask
    
restart:
    set_softirq_pending(0);   // Clear pending bits before processing
    local_irq_enable();       // Interrupts ON during softirq processing!
    
    while ((softirq_bit = ffs(pending))) {  // Find first set bit
        unsigned int vec_nr = softirq_bit - 1;
        struct softirq_action *h = &softirq_vec[vec_nr];
        
        h->action();    // Call the softirq handler
        pending >>= softirq_bit;
    }
    
    local_irq_disable();
    
    // Check if more softirqs were raised while we were processing:
    pending = local_softirq_pending();
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() &&
            --max_restart)
            goto restart;
        
        // Ran too long — hand off to ksoftirqd:
        wakeup_softirqd();
    }
}
```

Key observations:
- **Interrupts are re-enabled** during softirq processing — hardware IRQs can interrupt softirq handlers
- **Softirqs cannot preempt each other**: two instances of the same softirq can run simultaneously on different CPUs (they must be SMP-safe), but on one CPU they're sequential
- **Time-bounded**: softirqs won't run longer than 2ms before yielding to `ksoftirqd`

### 2.4 Registering a Softirq

```c
// Called once at boot time (not from modules):
open_softirq(NET_RX_SOFTIRQ, net_rx_action);

// The registration table:
struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp = {
    [NET_RX_SOFTIRQ] = { .action = net_rx_action },
    [NET_TX_SOFTIRQ] = { .action = net_tx_action },
    // ...
};
```

---

## 3. `ksoftirqd` — The Softirq Kernel Thread

When softirqs are arriving faster than they can be processed (e.g., network flood), processing them all inline in `irq_exit()` would starve user processes. The kernel hands off to a dedicated kernel thread:

```bash
# One ksoftirqd per CPU:
ps aux | grep ksoftirq
# root    10  0.0  0.0      0     0 ?  S    Jan01   5:32 [ksoftirqd/0]
# root    18  0.0  0.0      0     0 ?  S    Jan01   2:11 [ksoftirqd/1]
# root    26  0.0  0.0      0     0 ?  S    Jan01   4:55 [ksoftirqd/2]
```

`ksoftirqd` runs at `SCHED_NORMAL` nice=0, meaning it competes with user processes for CPU time. When a network flood hits, `ksoftirqd/X` will show up in `top` consuming significant CPU on the relevant cores.

---

## 4. Disabling Softirqs: `local_bh_disable` / `local_bh_enable`

```c
// "bh" = bottom half (historical name for softirqs)

local_bh_disable();  // Prevent softirqs from running on current CPU
// ... do work that must not be interrupted by softirqs ...
local_bh_enable();   // Re-enable softirqs (runs pending ones if any)

// These are safe to nest:
local_bh_disable();  // depth = 1
local_bh_disable();  // depth = 2
local_bh_enable();   // depth = 1 — still disabled
local_bh_enable();   // depth = 0 — now processes any pending softirqs
```

**When to use it**: In code that shares data with softirq handlers but runs in process context. Instead of a full spinlock with IRQ save, if you only need to protect against softirqs (not hardirqs) on the same CPU:

```c
// Protects against softirq on the same CPU, but not other CPUs:
local_bh_disable();
my_per_cpu_counter++;
local_bh_enable();

// Full spinlock needed for multi-CPU protection:
spin_lock_bh(&my_lock);    // = spin_lock + local_bh_disable
// ...
spin_unlock_bh(&my_lock);  // = local_bh_enable + spin_unlock
```

---

## 5. Tasklets — Dynamic Softirqs

Tasklets build on top of `TASKLET_SOFTIRQ` / `HI_SOFTIRQ` and let you create deferred work without adding to the fixed softirq table.

**Key property**: A tasklet instance runs on only one CPU at a time (serialized), even on SMP. This is the main difference from softirqs, which can run concurrently on different CPUs.

### 5.1 Declaring a Tasklet

```c
#include <linux/interrupt.h>

// Static initialization:
DECLARE_TASKLET(my_tasklet, my_tasklet_handler);      // Enabled by default
DECLARE_TASKLET_DISABLED(my_tasklet, my_tasklet_func); // Starts disabled

// Dynamic initialization:
struct tasklet_struct my_tasklet;
tasklet_init(&my_tasklet, my_tasklet_handler, (unsigned long)dev);
// Last arg is passed to handler as 'unsigned long data'
```

### 5.2 The Handler

```c
void my_tasklet_handler(struct tasklet_struct *t)
{
    // In modern kernels: tasklet_struct passed directly
    struct my_device *dev = from_tasklet(dev, t, tasklet);
    // Or the old style if using init with data:
    // struct my_device *dev = (struct my_device *)data;
    
    // Can: use spinlocks, read/write device registers, wake_up()
    // Cannot: sleep, take mutexes
    
    u32 status = dev->saved_status;  // Set by hardirq handler
    
    if (status & RX_COMPLETE) {
        process_received_data(dev);
    }
    if (status & TX_COMPLETE) {
        complete_transmit(dev);
    }
}
```

### 5.3 Scheduling and Control

```c
// Schedule for execution (safe to call from hardirq):
tasklet_schedule(&my_tasklet);     // Low priority (TASKLET_SOFTIRQ)
tasklet_hi_schedule(&my_tasklet);  // High priority (HI_SOFTIRQ)

// If already scheduled, won't be re-added (each tasklet has a "pending" bit).
// Calling tasklet_schedule() twice → runs once.

// Enable/disable (tasklet starts disabled with DECLARE_TASKLET_DISABLED):
tasklet_enable(&my_tasklet);
tasklet_disable(&my_tasklet);   // Blocks until current run completes
tasklet_disable_nosync(&my_tasklet);  // Non-blocking disable

// Kill (wait for current run to complete, prevent future scheduling):
tasklet_kill(&my_tasklet);  // MUST call this before freeing the object!
// This blocks if tasklet is currently running — do not call from atomic context
```

### 5.4 How Tasklets Work Internally

```c
// kernel/softirq.c
// TASKLET_SOFTIRQ handler:
static void tasklet_action(struct softirq_action *a)
{
    struct tasklet_struct *list = this_cpu_xchg(tasklet_vec.head, NULL);
    
    while (list) {
        struct tasklet_struct *t = list;
        list = list->next;
        
        // Try to lock the tasklet (prevents concurrent execution):
        if (tasklet_trylock(t)) {
            if (!atomic_read(&t->count)) {  // Not disabled?
                t->func(t);                 // Call the handler
                tasklet_unlock(t);
            } else {
                // Disabled — re-schedule for later
                tasklet_unlock(t);
                tasklet_schedule(t);
            }
        } else {
            // Another CPU is running this tasklet — re-queue
            tasklet_schedule(t);
        }
    }
}
```

### 5.5 Typical Pattern: Tasklet in a Driver

```c
struct my_device {
    void __iomem *base;
    struct tasklet_struct rx_tasklet;
    u32 rx_status;
    // ...
};

// In interrupt handler (hardirq — fast path):
irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_device *dev = data;
    
    dev->rx_status = readl(dev->base + STATUS_REG);
    writel(dev->rx_status, dev->base + ACK_REG);  // Acknowledge
    
    tasklet_schedule(&dev->rx_tasklet);  // Defer processing
    return IRQ_HANDLED;
}

// In tasklet handler (softirq context — still can't sleep):
void my_rx_tasklet(struct tasklet_struct *t)
{
    struct my_device *dev = from_tasklet(dev, t, rx_tasklet);
    u32 status = dev->rx_status;
    
    while (data_available(dev)) {
        receive_packet(dev);
    }
}

// In probe:
tasklet_setup(&dev->rx_tasklet, my_rx_tasklet);

// In remove:
tasklet_kill(&dev->rx_tasklet);
```

---

## 6. Workqueues — Bottom Halves That Can Sleep

If your deferred work needs to sleep (wait for locks, sleep for a timeout, call `kmalloc(GFP_KERNEL)`, do I/O), use a workqueue. Work items run in kernel thread context.

### 6.1 The Default Workqueue System

The kernel provides a shared system workqueue:

```c
#include <linux/workqueue.h>

// Work item structure:
struct work_struct my_work;
INIT_WORK(&my_work, my_work_handler);  // Initialize

// Schedule work on the system workqueue:
schedule_work(&my_work);              // On current CPU
schedule_work_on(cpu, &my_work);     // On specific CPU

// Delayed work (run after delay):
struct delayed_work my_delayed_work;
INIT_DELAYED_WORK(&my_delayed_work, my_delayed_handler);
schedule_delayed_work(&my_delayed_work, HZ);  // After 1 second (HZ = ticks per sec)
schedule_delayed_work(&my_delayed_work, msecs_to_jiffies(500));  // After 500ms

// Cancel pending work:
cancel_work_sync(&my_work);         // Cancel and wait for completion
cancel_delayed_work_sync(&my_dwork);
```

### 6.2 The Work Handler

```c
void my_work_handler(struct work_struct *work)
{
    // THIS IS A KERNEL THREAD — CAN SLEEP!
    struct my_device *dev = container_of(work, struct my_device, work);
    
    // Can take mutexes:
    mutex_lock(&dev->lock);
    
    // Can sleep:
    msleep(10);
    
    // Can call GFP_KERNEL allocations:
    void *buf = kmalloc(1024, GFP_KERNEL);
    
    // Can wait for completions:
    wait_for_completion(&dev->io_done);
    
    // Can block on I/O:
    ret = kernel_read(dev->file, buf, 1024, &pos);
    
    mutex_unlock(&dev->lock);
    kfree(buf);
}
```

### 6.3 Workers and `kworker` Threads

```bash
# See workqueue threads:
ps aux | grep kworker
# root    34  0.0  0.0  kworker/0:1-events
# root    35  0.0  0.0  kworker/0:1H-kblockd  ← High-priority queue
# root    36  0.0  0.0  kworker/1:1-events
# root    67  0.0  0.0  kworker/u8:2-flush-8:0  ← Per-device flush

# kworker thread naming: kworker/CPU:ID-POOL_NAME
# CPU: which CPU it's bound to (or 'u' for unbound)
# ID: worker number
# POOL_NAME: which workqueue/pool
```

The workqueue infrastructure manages a pool of threads. When work is queued, it wakes a thread from the pool. Idle threads are destroyed after 5 minutes.

### 6.4 Custom Workqueues

For work that must not interfere with system work (high-priority, ordered, or isolated):

```c
// Create your own workqueue:
struct workqueue_struct *my_wq;

// Ordered (single-threaded, work items run in order):
my_wq = create_singlethread_workqueue("my_wq");

// Unordered with specific flags:
my_wq = alloc_workqueue("my_wq",
    WQ_UNBOUND |           // Not bound to a specific CPU
    WQ_HIGHPRI |           // High priority (competes with SCHED_RR)
    WQ_MEM_RECLAIM,        // Has a rescue thread for memory pressure
    0);                    // max_active: 0 = use default

// Use it:
queue_work(my_wq, &my_work);
queue_delayed_work(my_wq, &my_dwork, delay);

// Flush all pending work (blocks):
flush_workqueue(my_wq);
drain_workqueue(my_wq);

// Destroy:
destroy_workqueue(my_wq);
```

### 6.5 Workqueue Flags

```c
WQ_UNBOUND          // Not bound to a CPU — can run on any CPU
                    // Better for CPU-intensive work (avoids starvation)
WQ_HIGHPRI          // High-priority worker thread pool
WQ_CPU_INTENSIVE    // Work items may be CPU-intensive; don't hold up other work
WQ_MEM_RECLAIM      // Needs to run during memory reclaim; has rescue thread
WQ_FREEZABLE        // Freeze during system suspend
WQ_SYSFS            // Expose workqueue in /sys/bus/workqueue/
WQ_POWER_EFFICIENT  // Prefer power-efficient scheduling
```

### 6.6 Common Bug: Forgetting `cancel_work_sync` on Module Unload

```c
// WRONG — module unloads while work is running:
static void __exit mymodule_exit(void)
{
    // Work may still be running in kworker thread!
    kfree(dev);  // CRASH: work accesses freed memory
}

// CORRECT:
static void __exit mymodule_exit(void)
{
    cancel_work_sync(&dev->work);      // Wait for work to complete
    cancel_delayed_work_sync(&dev->dwork);
    kfree(dev);  // Safe now
}
```

---

## 7. Threaded IRQs — The Modern Approach

Threaded IRQs combine the top-half/bottom-half model directly in the IRQ API. They're the preferred approach for new drivers.

```c
// Request a threaded IRQ:
int request_threaded_irq(unsigned int irq,
    irq_handler_t handler,      // Primary handler (hardirq context, must be fast)
    irq_handler_t thread_fn,    // Thread handler (kernel thread context, can sleep)
    unsigned long irqflags,
    const char *devname,
    void *dev_id);

// Example:
int err = request_threaded_irq(dev->irq,
    my_primary_handler,    // Fast: check if ours, disable IRQ line
    my_thread_handler,     // Slow: process data, can sleep
    IRQF_SHARED | IRQF_ONESHOT,
    "my_driver",
    dev);
```

The primary handler runs in hardirq context. If it returns `IRQ_WAKE_THREAD`, the kernel wakes a dedicated `irq/N-name` thread that runs `thread_fn`.

```c
irqreturn_t my_primary_handler(int irq, void *data)
{
    struct my_device *dev = data;
    
    // Check if this is our interrupt:
    if (!(readl(dev->base + STATUS) & OUR_INT))
        return IRQ_NONE;
    
    // Disable IRQ at device level (IRQF_ONESHOT will re-enable after thread):
    // (For level-triggered: device keeps asserting until we clear it in thread)
    return IRQ_WAKE_THREAD;  // Wake the thread handler
}

irqreturn_t my_thread_handler(int irq, void *data)
{
    struct my_device *dev = data;
    
    // Can sleep, take mutexes, do I/O:
    mutex_lock(&dev->lock);
    process_interrupt_data(dev);    // Can sleep
    clear_interrupt_source(dev);    // Acknowledge at device (safe now)
    mutex_unlock(&dev->lock);
    
    return IRQ_HANDLED;
}
```

```bash
# See threaded IRQ threads:
ps aux | grep "irq/"
# root    123  0.0  0.0  irq/24-nvme0q1    ← NVMe queue 1 thread
# root    124  0.0  0.0  irq/25-nvme0q2
# root    125  0.0  0.0  irq/26-i2c-1      ← I2C controller
```

### 7.1 `IRQF_ONESHOT` — Critical for Level-Triggered Threaded IRQs

For level-triggered interrupts with threaded handlers:
- Device asserts IRQ line (stays asserted until software clears it)
- Primary handler runs: can't clear it (would sleep)
- If we re-enable the IRQ line now, it fires immediately again — infinite loop!

`IRQF_ONESHOT` keeps the IRQ line disabled until the thread handler completes:

```
IRQ fires
  → Primary handler runs (IRQ line disabled by APIC mask)
  → Returns IRQ_WAKE_THREAD
  → Thread is woken [IRQ line stays disabled]
  → Thread processes + clears the IRQ source
  → Thread returns IRQ_HANDLED
  → Kernel re-enables the IRQ line [IRQF_ONESHOT behavior]
```

---

## 8. Choosing the Right Bottom-Half Mechanism

```
Need to defer work from hardirq context?
         │
         ▼
    Can it sleep? (e.g., mutex, I/O, kmalloc(GFP_KERNEL))
         │
    YES──┤──────────────────────── Workqueue or Threaded IRQ
         │
         NO
         │
    Is it very performance-critical and SMP-parallel OK?
         │
    YES──┤──────────────────────── Softirq (but you can't add new ones)
         │                         (Used by: network, block, timer, RCU)
         NO
         │
         ▼
    Tasklet (serialized, uses existing TASKLET_SOFTIRQ)
```

In practice for new drivers:
- **Threaded IRQ**: almost always the right choice for device drivers
- **Workqueue**: for periodic work, deferred work not tied to an IRQ
- **Tasklet**: legacy; still exists but discouraged for new code in favor of threaded IRQs

The kernel developers have been gradually replacing tasklets with threaded IRQs across the codebase.

---

## 9. Timer-Based Deferred Work

### 9.1 Kernel Timers (Low-Resolution)

```c
#include <linux/timer.h>

struct timer_list my_timer;

// Initialize:
timer_setup(&my_timer, my_timer_callback, 0 /* flags */);

// Schedule to fire:
mod_timer(&my_timer, jiffies + HZ);     // Fire in 1 second
mod_timer(&my_timer, jiffies + msecs_to_jiffies(500));  // 500ms

// Cancel:
del_timer(&my_timer);          // Non-blocking
del_timer_sync(&my_timer);     // Blocks until callback finishes (safe from race)

// Callback (runs in softirq context — TIMER_SOFTIRQ):
void my_timer_callback(struct timer_list *t)
{
    // Cannot sleep!
    struct my_device *dev = from_timer(dev, t, my_timer);
    
    // Do quick work, or schedule a workqueue item:
    schedule_work(&dev->periodic_work);
    
    // Re-arm if needed:
    mod_timer(&my_timer, jiffies + HZ);
}
```

`jiffies` is the kernel's low-resolution time counter, incremented `HZ` times per second (typically 250 on server kernels, 100 on embedded).

### 9.2 High-Resolution Timers (hrtimers)

For sub-millisecond precision:

```c
#include <linux/hrtimer.h>

struct hrtimer my_hrtimer;

// Initialize:
hrtimer_init(&my_hrtimer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
my_hrtimer.function = my_hrtimer_callback;

// Start:
hrtimer_start(&my_hrtimer, ns_to_ktime(500000), HRTIMER_MODE_REL);  // 500µs
hrtimer_start(&my_hrtimer, ktime_set(0, 1000000), HRTIMER_MODE_REL); // 1ms

// Cancel:
hrtimer_cancel(&my_hrtimer);   // Cancel and wait for callback to finish

// Callback (runs in HRTIMER_SOFTIRQ — still cannot sleep):
enum hrtimer_restart my_hrtimer_callback(struct hrtimer *timer)
{
    struct my_device *dev = container_of(timer, struct my_device, my_hrtimer);
    
    // Do work...
    
    // One-shot (don't restart):
    return HRTIMER_NORESTART;
    
    // Or restart with new period:
    hrtimer_forward_now(timer, ns_to_ktime(500000));
    return HRTIMER_RESTART;
}
```

---

## 10. `irq_work` — Deferred Work in Hardirq Context

Sometimes you need to run something "from hardirq context, but deferred to the next safe point." The `irq_work` mechanism handles this:

```c
#include <linux/irq_work.h>

struct irq_work my_irq_work;
init_irq_work(&my_irq_work, my_irq_work_handler);

// Queue it (safe from any context including NMI!):
irq_work_queue(&my_irq_work);

// Handler runs in hardirq context on the current CPU:
void my_irq_work_handler(struct irq_work *work)
{
    // Still can't sleep, but interrupts may be enabled
    // Common use: printk from NMI context
}
```

`perf` uses this to safely print output from performance counter overflow handlers that fire in NMI context.

---

## 11. The `completion` API — Thread Synchronization

Often you want to wait in one thread for something to complete in another (workqueue, thread, IRQ handler):

```c
#include <linux/completion.h>

struct completion my_comp;
init_completion(&my_comp);  // Or DECLARE_COMPLETION(my_comp); for static

// In the waiter thread (can sleep):
int ret = wait_for_completion(&my_comp);         // Wait forever
int ret = wait_for_completion_interruptible(&my_comp);  // Interruptible by signal
int ret = wait_for_completion_timeout(&my_comp, HZ);  // Wait max 1 second (0=timeout)

// In the worker (hardirq, softirq, thread — any context):
complete(&my_comp);         // Wake ONE waiter
complete_all(&my_comp);     // Wake ALL waiters
```

**Pattern**: interrupt handler signals work complete, process context waits.

```c
// Driver pattern:
struct my_device {
    struct completion tx_done;
    // ...
};

// Process context: send command and wait:
int send_command(struct my_device *dev, u32 cmd)
{
    init_completion(&dev->tx_done);
    writel(cmd, dev->base + CMD_REG);  // Trigger device
    
    // Wait for IRQ handler to signal completion:
    if (!wait_for_completion_timeout(&dev->tx_done, HZ)) {
        dev_err(dev->dev, "command timeout!\n");
        return -ETIMEDOUT;
    }
    return 0;
}

// IRQ handler:
irqreturn_t my_irq(int irq, void *data)
{
    struct my_device *dev = data;
    if (readl(dev->base + STATUS) & TX_DONE)
        complete(&dev->tx_done);
    return IRQ_HANDLED;
}
```

---

## 12. `wait_queue` — General Event Waiting

For waiting on arbitrary conditions:

```c
#include <linux/wait.h>

// Declare:
DECLARE_WAIT_QUEUE_HEAD(my_wq);  // Static
wait_queue_head_t my_wq;          // Dynamic
init_waitqueue_head(&my_wq);

// Wait for condition (process context, can sleep):
wait_event(my_wq, condition);                     // Uninterruptible
wait_event_interruptible(my_wq, condition);        // Returns -ERESTARTSYS if signal
wait_event_timeout(my_wq, condition, HZ);          // Max 1 second
wait_event_interruptible_timeout(my_wq, cond, HZ);

// Wake waiters (from any context):
wake_up(&my_wq);             // Wake one waiter (TASK_NORMAL)
wake_up_all(&my_wq);         // Wake all waiters
wake_up_interruptible(&my_wq);  // Wake only TASK_INTERRUPTIBLE waiters

// Typical pattern (producer/consumer):
// Producer (e.g., IRQ handler):
new_data = true;
wake_up(&data_wq);

// Consumer (process context):
wait_event_interruptible(data_wq, new_data);
process(data);
new_data = false;
```

The condition is re-evaluated atomically with respect to `wake_up` — this avoids missed wake-ups.

---

## 13. Softirq / Interrupt Context Detection

```c
// Check what context you're in:
in_irq()              // In hardirq handler
in_softirq()          // In softirq handler
in_interrupt()        // In any interrupt context (hardirq or softirq)
in_serving_softirq()  // Currently processing a softirq
in_nmi()              // In NMI handler
in_task()             // In task/process context (not any interrupt)

preempt_count()       // Raw count (0 means preemptible task context)

// These determine what you can do:
// in_task() → can sleep, use GFP_KERNEL, take mutexes
// in_softirq() → no sleep, GFP_ATOMIC, spinlock only
// in_irq() → no sleep, GFP_ATOMIC, spinlock only, NO BH functions
// in_nmi() → even more restricted, use irq_work for any deferral
```

---

## 14. The Network Stack as a Case Study

The network receive path is the best example of all these mechanisms working together:

```
NIC receives packet
    │
    ▼
NIC DMA: packet → ring buffer in RAM
    │
    ▼
NIC asserts IRQ
    │
    ▼
[HARDIRQ] NIC driver irq_handler():
    - Read IRQ status register
    - Call napi_schedule(&napi)  ← raises NET_RX_SOFTIRQ
    - Disable NIC IRQ (NAPI mode)
    - Return IRQ_HANDLED
    │
    ▼
irq_exit_rcu(): pending NET_RX_SOFTIRQ → invoke_softirq()
    │
    ▼
[SOFTIRQ] net_rx_action():
    - Calls nic_poll() (registered by driver)
    │
    ▼
[SOFTIRQ] nic_poll(napi, budget=64):
    - Picks up packets from ring buffer (up to budget)
    - For each packet:
        netif_receive_skb(skb)
            ip_rcv(skb)          → IP layer
            tcp_v4_rcv(skb)      → TCP layer
            sk_data_ready(sock)  → wake up waiting process
    - If ring empty: napi_complete() + re-enable NIC IRQ
    - Return # packets processed
    │
    ▼
[PROCESS CONTEXT] User calls recv():
    - Was waiting in wait_queue
    - Woken by sk_data_ready
    - Copies data from socket buffer to user buffer
    - Returns to user space
```

This is why network performance is measured in packets per second, not just bytes — each packet requires multiple context transitions.

---

## 15. Tracing the Software Interrupt Path

```bash
# Watch softirq activity:
watch -n 1 cat /proc/softirqs
#                    CPU0       CPU1       CPU2       CPU3
#          HI:          0          0          0          0
#       TIMER:    4567890    4123456    3987654    4234567  ← Lots of timer ticks
#      NET_TX:      12345      11234      10987      11456
#      NET_RX:    2345678    2134567    2098765    2156789  ← Heavy network
#       BLOCK:      45678      43210      41234      44321
#    IRQ_POLL:          0          0          0          0
#    TASKLET:       5678       4321       3210       5432
#       SCHED:    1234567    1123456    1098765    1123456
#     HRTIMER:     12345      11234      10987      11456
#         RCU:    9876543    9765432    9654321    9765432  ← RCU callbacks

# Measure ksoftirqd CPU usage under load:
pidstat -p $(pgrep -d, ksoftirqd) 1

# bpftrace: trace softirq entry/exit with timing:
sudo bpftrace -e '
tracepoint:irq:softirq_entry {
    @ts[args->vec] = nsecs;
}
tracepoint:irq:softirq_exit {
    if (@ts[args->vec]) {
        @time_ns[args->vec] = hist(nsecs - @ts[args->vec]);
        delete(@ts[args->vec]);
    }
}'

# ftrace: trace workqueue execution:
echo workqueue:workqueue_execute_start > /sys/kernel/debug/tracing/set_event
cat /sys/kernel/debug/tracing/trace_pipe
# kworker/0:1-34    [000] ...  1234.567: workqueue_execute_start: work struct ...
```

---

## 16. Common Bugs in Bottom-Half Code

### 16.1 Sleeping in Softirq Context

```c
// WRONG:
static void my_softirq_handler(void)
{
    mutex_lock(&dev->lock);   // BUG: may sleep in softirq!
    kmalloc(64, GFP_KERNEL);  // BUG: may sleep!
}

// CORRECT:
static void my_softirq_handler(void)
{
    spin_lock(&dev->lock);        // OK: non-sleeping spinlock
    kmalloc(64, GFP_ATOMIC);      // OK: atomic allocation
    spin_unlock(&dev->lock);
}
```

### 16.2 Missing `tasklet_kill` on Module Unload

```c
// WRONG:
static void __exit mymodule_exit(void)
{
    kfree(dev);  // tasklet may fire after this and access freed dev!
}

// CORRECT:
static void __exit mymodule_exit(void)
{
    tasklet_kill(&dev->tasklet);  // Wait for running tasklet, prevent future
    kfree(dev);
}
```

### 16.3 Touching Freed Object in Workqueue

```c
// WRONG:
static void do_remove(struct my_device *dev)
{
    kfree(dev);
    // Work item might still be pending or running!
}

// CORRECT:
static void do_remove(struct my_device *dev)
{
    cancel_work_sync(&dev->work);
    cancel_delayed_work_sync(&dev->delayed_work);
    kfree(dev);
}
```

---

## 17. Mental Model Checkpoint

After Day 10, you should be able to:

1. Explain the top-half/bottom-half split and why it's necessary.
2. Name all 10 softirqs and explain which subsystem uses each.
3. When does `ksoftirqd` run and why does it exist?
4. Write a correct tasklet: initialization, scheduling from hardirq, handler, cleanup.
5. Write a work item: initialization, scheduling, handler with sleeping, cleanup in exit.
6. What is `IRQF_ONESHOT` and when is it required?
7. Describe the flow from network packet arrival to userspace `recv()` return.
8. What is `local_bh_disable()` and when would you use it?
9. Write a correct `completion` usage to synchronize process context with an IRQ handler.
10. What is the difference between `wait_event` and `wait_event_interruptible`?

---

## Key Source Files

```bash
kernel/softirq.c            # __do_softirq(), raise_softirq(), tasklet implementation
include/linux/interrupt.h   # DECLARE_TASKLET, request_threaded_irq, softirq API
include/linux/workqueue.h   # All workqueue types and functions
kernel/workqueue.c          # Workqueue implementation
include/linux/timer.h       # Kernel timer API
include/linux/hrtimer.h     # High-resolution timer API
include/linux/wait.h        # wait_queue_head, wait_event*, wake_up*
include/linux/completion.h  # completion API
net/core/dev.c              # net_rx_action() — network softirq handler
```

---

## Summary

The software interrupt path exists because hardware interrupt handlers must complete in microseconds but real work takes longer. The kernel provides three escalating levels of deferral:

1. **Softirqs**: per-CPU, non-sleeping, parallel on SMP. Fixed 10 types only. Use for highest-performance deferred work (networking, block I/O, timers). Running in `SOFTIRQ` bit of `preempt_count`.

2. **Tasklets**: built on softirqs, serialized per-tasklet instance. More flexible than softirqs (can be created dynamically), still cannot sleep. Being phased out in favor of threaded IRQs.

3. **Workqueues**: run in `kworker` kernel threads. **Can sleep**. Highest flexibility but more overhead. Right choice for anything that might block.

**Threaded IRQs** (`request_threaded_irq`) are the modern recommended approach for device drivers — they combine a fast primary handler with a sleepable thread handler cleanly.

Tomorrow: kernel locking basics — spinlocks, mutexes, semaphores, and the rules for choosing between them.
