# Day 7 — Interrupt Handling: ISRs, Top/Bottom Halves, Threaded IRQs & Latency
**Environment:** 🔵 RPi4 (button on GPIO27)
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [What Is an Interrupt?](#1-what-is-an-interrupt)
2. [The Interrupt Controller — GIC on RPi4](#2-the-interrupt-controller--gic-on-rpi4)
3. [Requesting an IRQ — `request_irq()`](#3-requesting-an-irq--request_irq)
4. [Writing an Interrupt Service Routine](#4-writing-an-interrupt-service-routine)
5. [Top Half vs Bottom Half](#5-top-half-vs-bottom-half)
6. [Softirqs](#6-softirqs)
7. [Tasklets](#7-tasklets)
8. [Workqueues](#8-workqueues)
9. [Threaded IRQ Handlers](#9-threaded-irq-handlers)
10. [Interrupt Sharing](#10-interrupt-sharing)
11. [Measuring IRQ Latency with ftrace](#11-measuring-irq-latency-with-ftrace)
12. [Hands-On Lab — Button Driver with IRQ on RPi4](#12-hands-on-lab--button-driver-with-irq-on-rpi4)
13. [Interview Questions](#13-interview-questions)

---

## 1. What Is an Interrupt?

An interrupt is a hardware signal that causes the CPU to stop whatever it's doing, save its state, and run a special handler function — the **Interrupt Service Routine (ISR)**. After the ISR finishes, the CPU resumes the interrupted work.

### Types of Interrupts

```
Hardware Interrupts (asynchronous — fired by hardware at any time):
  ├── External IRQ: GPIO pin change, UART RX data, timer overflow
  ├── DMA completion: DMA controller finished a transfer
  └── PCIe/USB events: device sending MSI (Message-Signaled Interrupt)

Software Interrupts (synchronous — triggered by CPU itself):
  ├── System calls: svc/syscall instruction
  ├── Exceptions: page fault, undefined instruction, alignment fault
  └── Software IRQs: raised via raise_softirq() in kernel code

Inter-Processor Interrupts (IPI — between CPU cores):
  ├── TLB shootdown: one CPU invalidates another's TLB
  ├── Scheduler reschedule: wake up a task on another CPU
  └── Function call: run a function on a specific CPU
```

### The Interrupt Flow on ARM64 (RPi4)

```
Hardware Event (e.g., button pressed → GPIO27 goes low)
         │
         ▼
BCM2711 GPIO Controller detects falling edge
         │ signals interrupt to GIC
         ▼
GIC-400 (Generic Interrupt Controller)
         │ routes IRQ to CPU0 (or any CPU configured via affinity)
         │ raises IRQ exception
         ▼
ARM64 CPU — takes exception to EL1
         │ saves all registers to kernel stack (exception frame)
         │ jumps to kernel's exception vector (vectors in el1h_irq handler)
         ▼
Kernel IRQ dispatcher (kernel/irq/handle.c)
         │ reads GIC to find which interrupt fired
         │ looks up handler registered for this IRQ number
         ▼
Your ISR function (irq_handler in your driver)
         │ does minimal work, returns IRQ_HANDLED
         ▼
Return from exception
         │ restore registers, return to interrupted code
         ▼
Interrupted code resumes
```

### Why Speed Matters in ISRs

While an ISR runs on a CPU core:
- That core's interrupts of equal or lower priority are **disabled**
- The interrupted process cannot run on that core
- Any other interrupt that fires on that core is deferred

A slow ISR increases **interrupt latency** — the time other interrupts wait before being serviced. For a real-time system playing audio, a 1ms ISR causes an audible glitch. For a network driver, a slow ISR means dropped packets.

**Rule: Do the absolute minimum in an ISR. Defer everything else.**

---

## 2. The Interrupt Controller — GIC on RPi4

RPi4 uses a **GIC-400** (Generic Interrupt Controller v2) connected to the BCM2711 SoC's interrupt lines.

### IRQ Numbers

Linux assigns each interrupt source a virtual IRQ number. The mapping between hardware signal and IRQ number comes from the Device Tree. You don't hardcode IRQ numbers — you ask the GPIO subsystem:

```c
int irq = gpio_to_irq(gpio_pin);
// OR for DT-based drivers:
int irq = platform_get_irq(pdev, 0);
// OR:
int irq = of_irq_get(np, 0);
```

### Viewing Active IRQs

```bash
# On RPi4
cat /proc/interrupts
# CPU0      CPU1      CPU2      CPU3
# 16:  4821231   4695312   4634812   4701024  GICv2  27 Level  arch_timer
# 23:        0         0         0         0  GICv2  34 Level  DMA IRQ
# 64:      438         0         0         0  pinctrl-bcm2835   4 Edge  eth0
# ...
# Columns: IRQ_num, counts per CPU, controller, hw_irq, trigger, name

# Check GPIO interrupts specifically
cat /proc/interrupts | grep gpio
```

---

## 3. Requesting an IRQ — `request_irq()`

```c
#include <linux/interrupt.h>

int request_irq(unsigned int irq,
                irq_handler_t handler,
                unsigned long flags,
                const char *name,
                void *dev_id);

void free_irq(unsigned int irq, void *dev_id);
```

### Parameters

**`irq`** — the IRQ number (from `gpio_to_irq()` etc.)

**`handler`** — your ISR function pointer, signature:
```c
irqreturn_t my_handler(int irq, void *dev_id);
```

**`flags`** — controls IRQ behavior (can OR multiple flags):

| Flag | Meaning |
|------|---------|
| `IRQF_TRIGGER_RISING` | Fire on rising edge (low→high) |
| `IRQF_TRIGGER_FALLING` | Fire on falling edge (high→low) |
| `IRQF_TRIGGER_BOTH` | Fire on both edges |
| `IRQF_TRIGGER_HIGH` | Fire when signal is high (level trigger) |
| `IRQF_TRIGGER_LOW` | Fire when signal is low (level trigger) |
| `IRQF_SHARED` | IRQ line shared with other devices |
| `IRQF_ONESHOT` | Disable IRQ line until handler thread completes |
| `IRQF_NO_AUTOEN` | Don't automatically enable IRQ after request |

**`name`** — appears in `/proc/interrupts`

**`dev_id`** — passed to handler as second argument. For `IRQF_SHARED`, must be unique (used to identify which handler to call). Always pass your device struct pointer here.

### Full Example

```c
struct my_dev {
    int irq;
    int gpio;
    // ...
};

static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    // minimal work here
    return IRQ_HANDLED;
}

// In probe/init:
dev->gpio = 27;
dev->irq  = gpio_to_irq(dev->gpio);
if (dev->irq < 0) {
    dev_err(dev, "Failed to get IRQ for GPIO %d\n", dev->gpio);
    return dev->irq;
}

ret = request_irq(dev->irq,
                  my_handler,
                  IRQF_TRIGGER_FALLING,
                  "my_button",
                  dev);
if (ret) {
    dev_err(dev, "request_irq failed: %d\n", ret);
    return ret;
}

// In remove/exit:
free_irq(dev->irq, dev);
```

### `devm_request_irq()` — Managed IRQ Request

For platform drivers, use the managed version that auto-frees on device removal:

```c
ret = devm_request_irq(&pdev->dev, irq, my_handler,
                        IRQF_TRIGGER_FALLING, "my_button", dev);
// No need to call free_irq() — kernel does it automatically on device removal
```

### Return Value of ISR

```c
IRQ_HANDLED    // we handled this interrupt
IRQ_NONE       // this interrupt wasn't ours (important for IRQF_SHARED)
IRQ_WAKE_THREAD // handled hard-IRQ part, wake the threaded handler
```

---

## 4. Writing an Interrupt Service Routine

### The Golden Rules of ISR Code

1. **No sleeping** — no `msleep()`, `mutex_lock()`, `kmalloc(GFP_KERNEL)`, `schedule()`, `copy_to/from_user()`
2. **No blocking I/O** — no file operations, no disk access
3. **Minimal work** — read a status register, acknowledge the interrupt, queue work for later
4. **No blocking** of any kind
5. **Fast** — aim for < 1 µs in the ISR body
6. **Use `GFP_ATOMIC`** for any memory allocation (and prefer to avoid even that)
7. **Use `spin_lock_irqsave`** for any shared data (IRQs on this CPU are already disabled for the same IRQ level, but not necessarily for all levels)

### What ISRs Typically Do

```c
static irqreturn_t good_isr(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    u32 status;

    /* 1. Read interrupt status register to determine what happened */
    status = readl(dev->base + IRQ_STATUS_REG);

    /* 2. If not our interrupt (for shared IRQs), return IRQ_NONE */
    if (!(status & MY_IRQ_BITS))
        return IRQ_NONE;

    /* 3. Acknowledge / clear the interrupt (critical! prevents re-firing) */
    writel(status & MY_IRQ_BITS, dev->base + IRQ_CLEAR_REG);

    /* 4. Record what happened (minimal state update) */
    dev->last_status = status;
    atomic_inc(&dev->irq_count);

    /* 5. Wake up any sleeping process / schedule deferred work */
    wake_up_interruptible(&dev->wait_queue);
    /* OR: tasklet_schedule(&dev->tasklet); */
    /* OR: schedule_work(&dev->work); */

    return IRQ_HANDLED;
}
```

### Enabling / Disabling Interrupts

```c
// Disable a specific IRQ line
disable_irq(irq);        // waits for any running handler to finish
disable_irq_nosync(irq); // doesn't wait (safe from ISR)

// Re-enable
enable_irq(irq);

// Disable ALL interrupts on current CPU (save/restore)
unsigned long flags;
local_irq_save(flags);    // save flags, disable IRQs
/* ... critical section ... */
local_irq_restore(flags); // restore flags

// Check if IRQs are enabled
irqs_disabled()   // true if IRQs disabled on current CPU
in_interrupt()    // true if in interrupt context
in_irq()          // true if in hard IRQ context
in_softirq()      // true if in softirq context
```

---

## 5. Top Half vs Bottom Half

The **top half** is the actual ISR — it runs immediately when the interrupt fires with interrupts disabled (or at elevated priority). It does the absolute minimum.

The **bottom half** is deferred work — it runs later with interrupts re-enabled, handling the actual processing of the interrupt event. There are three mechanisms: **softirqs**, **tasklets**, and **workqueues**.

```
Hardware Interrupt fires
         │
         ▼
TOP HALF (ISR)
  ├── Acknowledge hardware interrupt           │
  ├── Read status registers                    │  Runs with IRQs
  ├── Save minimal state                       │  disabled/masked
  ├── Schedule bottom half                     │  MUST be fast
  └── Return IRQ_HANDLED                       │
         │
         ▼  (return from interrupt context)
BOTTOM HALF (deferred work)
  ├── Process the data read in top half        │  Runs with IRQs
  ├── Copy data to user buffers                │  enabled (mostly)
  ├── Allocate memory (GFP_KERNEL ok)          │  CAN be slow
  ├── Wake up sleeping processes               │  CAN sleep
  └── Submit I/O                               │  (workqueues only)
```

---

## 6. Softirqs

Softirqs are the lowest-level bottom-half mechanism. They run with interrupts **enabled** but are not allowed to sleep. Softirqs are **statically compiled** — there is a fixed set defined at compile time, not dynamically registered by drivers.

```c
// Fixed set of softirq types (include/linux/interrupt.h):
enum {
    HI_SOFTIRQ = 0,       // high-priority tasklets
    TIMER_SOFTIRQ,         // timer wheel processing
    NET_TX_SOFTIRQ,        // network transmit
    NET_RX_SOFTIRQ,        // network receive
    BLOCK_SOFTIRQ,         // block device completion
    IRQ_POLL_SOFTIRQ,      // IRQ polling
    TASKLET_SOFTIRQ,       // tasklets (general bottom half)
    SCHED_SOFTIRQ,         // scheduler
    HRTIMER_SOFTIRQ,       // high-resolution timers
    RCU_SOFTIRQ,           // RCU callbacks
    NR_SOFTIRQS
};
```

**Drivers almost never use softirqs directly.** Use tasklets (which run on top of softirqs) or workqueues instead. The network subsystem and block layer use softirqs internally because they're performance-critical.

```bash
# See softirq statistics
cat /proc/softirqs
#               CPU0       CPU1       CPU2       CPU3
# HI:             0          0          0          0
# TIMER:   12834567   12654321   12701234   12812345
# NET_TX:     45123      23456      34567      12345
# NET_RX:    456789     234567     345678     123456
```

---

## 7. Tasklets

Tasklets are the simplest bottom-half mechanism for driver writers. They run in softirq context (after the hard IRQ returns), with interrupts enabled, but **cannot sleep**.

```c
#include <linux/interrupt.h>

/* Define the tasklet function */
static void my_tasklet_func(struct tasklet_struct *t)
{
    struct my_dev *dev = from_tasklet(dev, t, tasklet);
    /*
     * Runs in softirq context:
     * - Interrupts ENABLED
     * - Cannot sleep
     * - Cannot call kmalloc(GFP_KERNEL) — use GFP_ATOMIC
     * - Can use spin_lock_bh() or spin_lock_irqsave()
     */
    pr_info("Tasklet processing IRQ data: status=0x%x\n", dev->last_status);
    process_data(dev);
    wake_up_interruptible(&dev->wq);
}

/* In device struct */
struct my_dev {
    struct tasklet_struct tasklet;
    u32 last_status;
    // ...
};

/* Initialize */
tasklet_setup(&dev->tasklet, my_tasklet_func);

/* Schedule from ISR (top half) */
static irqreturn_t my_isr(int irq, void *id)
{
    struct my_dev *dev = id;
    dev->last_status = readl(dev->base + STATUS);
    writel(dev->last_status, dev->base + CLEAR);
    tasklet_schedule(&dev->tasklet);   /* queue for bottom half */
    return IRQ_HANDLED;
}

/* Cleanup */
tasklet_kill(&dev->tasklet);  /* wait for tasklet to finish, then disable */
```

### Tasklet Guarantees

- Never runs in parallel with itself on multiple CPUs (serialized)
- Runs on the same CPU that scheduled it (cache friendly)
- Runs before workqueue callbacks (lower latency than workqueues)
- Can be scheduled multiple times but runs only once per "batch"

### Tasklet vs Softirq

| | Tasklet | Softirq |
|--|---------|---------|
| Dynamic registration | Yes | No (fixed set) |
| Parallelism | Serialized (same tasklet, multiple CPUs can't run it in parallel) | Parallel (same softirq CAN run on multiple CPUs) |
| Typical use | Driver bottom halves | Network stack, timers |
| Sleep allowed | No | No |

---

## 8. Workqueues

Workqueues are the most flexible bottom-half mechanism. They run in the context of kernel threads (`kworker`), which means they **can sleep**, can use `GFP_KERNEL`, and can do I/O.

```c
#include <linux/workqueue.h>

/* Define work item */
static void my_work_func(struct work_struct *work)
{
    struct my_dev *dev = container_of(work, struct my_dev, work);
    /*
     * Runs in kworker thread context:
     * - Interrupts ENABLED
     * - CAN sleep (mutex_lock, msleep, etc.)
     * - CAN kmalloc(GFP_KERNEL)
     * - CAN do I/O
     */
    pr_info("Work running for device\n");
    mutex_lock(&dev->lock);
    process_data(dev);
    mutex_unlock(&dev->lock);
    wake_up_interruptible(&dev->wq);
}

/* In device struct */
struct my_dev {
    struct work_struct work;
    // ...
};

/* Initialize */
INIT_WORK(&dev->work, my_work_func);

/* Schedule from ISR */
static irqreturn_t my_isr(int irq, void *id)
{
    struct my_dev *dev = id;
    dev->last_status = readl(dev->base + STATUS);
    writel(dev->last_status, dev->base + CLEAR);
    schedule_work(&dev->work);    /* queue onto system workqueue */
    return IRQ_HANDLED;
}

/* Cleanup — wait for work to finish before freeing resources */
cancel_work_sync(&dev->work);  /* cancel and wait for completion */
flush_work(&dev->work);        /* just wait, don't cancel */
```

### Delayed Work

```c
struct delayed_work {
    struct work_struct work;
    struct timer_list timer;
};

INIT_DELAYED_WORK(&dev->dwork, my_delayed_func);

/* Schedule to run after 100ms */
schedule_delayed_work(&dev->dwork, msecs_to_jiffies(100));

/* Cancel */
cancel_delayed_work_sync(&dev->dwork);
```

### Custom Workqueues

The system workqueue (`schedule_work()`) is shared. For time-critical or high-volume work, create a dedicated workqueue:

```c
/* Create dedicated single-thread workqueue */
struct workqueue_struct *my_wq;
my_wq = alloc_ordered_workqueue("my_driver_wq", 0);
// OR multi-threaded:
my_wq = alloc_workqueue("my_driver_wq", WQ_MEM_RECLAIM, 0);

/* Queue work on your private workqueue */
queue_work(my_wq, &dev->work);

/* Cleanup */
flush_workqueue(my_wq);
destroy_workqueue(my_wq);
```

### Workqueue Flags

| Flag | Meaning |
|------|---------|
| `WQ_HIGHPRI` | Run on high-priority worker threads |
| `WQ_MEM_RECLAIM` | Has a rescue worker for memory-pressure situations |
| `WQ_UNBOUND` | Not bound to any CPU (for long-running work) |
| `WQ_FREEZABLE` | Frozen during system suspend |
| `WQ_SYSFS` | Expose in sysfs for tuning |

---

## 9. Threaded IRQ Handlers

**Threaded IRQs** are the modern, preferred approach for most drivers. They split the handler into two functions:

1. **Hard IRQ handler** (top half) — runs in interrupt context, does the absolute minimum (acknowledge hardware)
2. **Thread handler** (bottom half) — runs in a dedicated kernel thread, CAN sleep, can do anything

```c
int request_threaded_irq(unsigned int irq,
                          irq_handler_t handler,       // hard IRQ (top half)
                          irq_handler_t thread_fn,     // thread (bottom half)
                          unsigned long irqflags,
                          const char *devname,
                          void *dev_id);
```

### Example: Button with Threaded IRQ

```c
/* Hard IRQ handler — runs in interrupt context */
static irqreturn_t btn_hard_irq(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;

    /* Just acknowledge and request thread to run */
    /* Disable the IRQ line until thread finishes (IRQF_ONESHOT) */
    dev->raw_value = gpio_get_value(dev->gpio);
    return IRQ_WAKE_THREAD;   /* request thread handler */
}

/* Thread handler — runs in kernel thread context (can sleep!) */
static irqreturn_t btn_thread_fn(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;

    /* Can sleep here — e.g., debounce via msleep */
    msleep(20);   /* debounce: wait 20ms */

    /* Re-read GPIO after debounce */
    int val = gpio_get_value(dev->gpio);

    /* Can use mutex, kmalloc(GFP_KERNEL), etc. */
    mutex_lock(&dev->lock);
    dev->button_state = val;
    dev->press_count++;
    mutex_unlock(&dev->lock);

    wake_up_interruptible(&dev->wq);
    return IRQ_HANDLED;
}

/* Request threaded IRQ */
ret = request_threaded_irq(
    irq,
    btn_hard_irq,             /* hard IRQ: minimal, returns IRQ_WAKE_THREAD */
    btn_thread_fn,            /* thread: does the real work, can sleep */
    IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
    /* IRQF_ONESHOT: keep IRQ line disabled until thread completes */
    "my_button",
    dev
);
```

### `IRQF_ONESHOT` is Critical for Threaded IRQs

Without `IRQF_ONESHOT`, the IRQ line is re-enabled when the hard IRQ handler returns. If the hardware is level-triggered (IRQ line stays asserted until handled), the hard IRQ fires again immediately, before the thread had a chance to clear it — infinite interrupt storm.

`IRQF_ONESHOT` keeps the IRQ line disabled until the thread handler returns, giving it time to clear the hardware condition.

### Threaded IRQ vs Workqueue

| | Threaded IRQ | Workqueue |
|--|-------------|-----------|
| Handler is | A kernel thread per IRQ | Shared `kworker` thread pool |
| Priority | Real-time capable | Normal kernel thread priority |
| Latency | Lower (dedicated thread) | Higher (shared pool) |
| Can sleep | Yes | Yes |
| Scheduling | When IRQ fires | When `schedule_work()` called |
| Use case | IRQ bottom halves | Deferred work from anywhere |

---

## 10. Interrupt Sharing

Multiple devices can share a single IRQ line (common with legacy PCI INTx). When a shared IRQ fires, **every handler registered for that IRQ is called** in turn. Each handler must identify if the interrupt is from its device.

```c
/* Register with IRQF_SHARED */
request_irq(irq, my_handler,
            IRQF_SHARED | IRQF_TRIGGER_LOW,
            "my_device", dev);   /* dev must be unique, non-NULL */

/* In the handler, check if interrupt is ours */
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    u32 status;

    /* Read interrupt status register */
    status = readl(dev->regs + IRQ_STATUS);

    /* If not our interrupt, return IRQ_NONE immediately */
    if (!(status & MY_IRQ_BIT))
        return IRQ_NONE;

    /* Clear our interrupt */
    writel(MY_IRQ_BIT, dev->regs + IRQ_CLEAR);

    /* Handle it */
    tasklet_schedule(&dev->tasklet);
    return IRQ_HANDLED;
}
```

**Rules for shared IRQs:**
- All handlers sharing an IRQ must use `IRQF_SHARED`
- `dev_id` must be non-NULL (used to identify handlers on `free_irq`)
- Handlers must return `IRQ_NONE` quickly if the interrupt isn't theirs
- All handlers sharing an IRQ must agree on trigger type

---

## 11. Measuring IRQ Latency with ftrace

ftrace is the kernel's built-in tracing framework. It can measure interrupt handling latency with microsecond precision.

### Set Up ftrace

```bash
# On RPi4 — ftrace is mounted at /sys/kernel/debug/tracing/
ls /sys/kernel/debug/tracing/

# Check available tracers
cat /sys/kernel/debug/tracing/available_tracers
# blk function_graph function nop

# Check available events
ls /sys/kernel/debug/tracing/events/irq/
```

### Measure IRQ Latency (irqsoff tracer)

The `irqsoff` tracer records the maximum time interrupts were disabled:

```bash
# Enable irqsoff tracer
echo irqsoff > /sys/kernel/debug/tracing/current_tracer

# Start tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Run some workload on RPi4 (e.g., compile something, run a loop)
for i in $(seq 1 100000); do echo $i > /dev/null; done

# Stop tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on

# View results
cat /sys/kernel/debug/tracing/trace | head -50
# Shows the maximum latency and the call stack that caused it
```

### Trace Specific IRQ Events

```bash
# Trace all interrupt entry/exit
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Press the button several times
# ...

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# Example output:
# <idle>-0   [001] d..1  1234.567890: irq_handler_entry: irq=189 name=my_button
# <idle>-0   [001] d..1  1234.567891: irq_handler_exit: irq=189 ret=handled
# Time between entry and exit is your ISR latency
```

### Function Graph Tracer for Your Module

```bash
# Trace only your module's functions
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo "my_module_name" > /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on
# trigger IRQ (press button)
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
# Shows call graph with timing
```

### Using `perf` for IRQ Statistics

```bash
# Count interrupts for 5 seconds
sudo perf stat -e irq:irq_handler_entry sleep 5

# Show per-interrupt statistics
sudo perf top -e irq:irq_handler_entry

# Record a trace and analyze
sudo perf record -e irq:irq_handler_entry,irq:irq_handler_exit -a sleep 5
sudo perf report
```

---

## 12. Hands-On Lab — Button Driver with IRQ on RPi4

### Hardware Setup

```
RPi4 Pin 1  (3.3V)  ──┤10kΩ├── GPIO27 (Pin 13)  ← pull-up keeps line HIGH
RPi4 Pin 13 (GPIO27) ─────────── Button Pin A
Button Pin B ──────────────────── GND (Pin 14)

When button pressed: GPIO27 pulled LOW → falling edge → IRQ fires
```

### Full Driver: `btn_irq.c`

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/gpio.h>
#include <linux/interrupt.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/wait.h>
#include <linux/spinlock.h>
#include <linux/workqueue.h>
#include <linux/ktime.h>

#define DRIVER_NAME   "btn_irq"
#define BUTTON_GPIO   27

struct btn_dev {
    /* GPIO / IRQ */
    int                gpio;
    int                irq;

    /* State */
    volatile bool      event_pending;
    ktime_t            last_press_time;
    unsigned long      press_count;
    spinlock_t         lock;

    /* Workqueue for bottom-half (debounce) */
    struct work_struct dwork;

    /* Wait queue for blocking read */
    wait_queue_head_t  wq;

    /* Char device */
    struct cdev        cdev;
};

static dev_t dev_num;
static struct btn_dev *bdev;
static struct class *bclass;

/* ── Bottom half: workqueue handler ─────────────────── */
static void btn_work(struct work_struct *w)
{
    struct btn_dev *dev = container_of(w, struct btn_dev, dwork);
    unsigned long flags;
    int gpio_val;

    /* Debounce: wait 20ms then re-read GPIO */
    msleep(20);
    gpio_val = gpio_get_value(dev->gpio);

    /* Only count if GPIO is still low (button still pressed) */
    if (gpio_val == 0) {
        spin_lock_irqsave(&dev->lock, flags);
        dev->event_pending  = true;
        dev->last_press_time = ktime_get();
        dev->press_count++;
        spin_unlock_irqrestore(&dev->lock, flags);

        pr_info("btn_irq: button press #%lu confirmed (GPIO=%d)\n",
                dev->press_count, gpio_val);

        wake_up_interruptible(&dev->wq);
    } else {
        pr_debug("btn_irq: bounce filtered out\n");
    }
}

/* ── Top half: hard IRQ handler ──────────────────────── */
static irqreturn_t btn_hard_irq(int irq, void *dev_id)
{
    struct btn_dev *dev = dev_id;

    /* Critical: keep this TINY */
    /* Just schedule the work — debounce in the bottom half */
    schedule_work(&dev->dwork);

    return IRQ_HANDLED;
}

/* ── File operations ─────────────────────────────────── */
static int btn_open(struct inode *inode, struct file *filp)
{
    filp->private_data = container_of(inode->i_cdev, struct btn_dev, cdev);
    return 0;
}

static ssize_t btn_read(struct file *filp, char __user *ubuf,
                         size_t count, loff_t *pos)
{
    struct btn_dev *dev = filp->private_data;
    char buf[64];
    int len, ret;
    unsigned long flags;
    ktime_t press_time;
    unsigned long press_count;

    /* Blocking: wait until a button event is available */
    if (filp->f_flags & O_NONBLOCK) {
        if (!dev->event_pending) return -EAGAIN;
    } else {
        ret = wait_event_interruptible(dev->wq, dev->event_pending);
        if (ret) return -EINTR;
    }

    /* Consume the event */
    spin_lock_irqsave(&dev->lock, flags);
    dev->event_pending = false;
    press_time  = dev->last_press_time;
    press_count = dev->press_count;
    spin_unlock_irqrestore(&dev->lock, flags);

    len = snprintf(buf, sizeof(buf),
                   "press=%lu time_ns=%lld\n",
                   press_count, ktime_to_ns(press_time));

    if (copy_to_user(ubuf, buf, len)) return -EFAULT;
    return len;
}

static const struct file_operations btn_fops = {
    .owner   = THIS_MODULE,
    .open    = btn_open,
    .read    = btn_read,
};

/* ── Init / Exit ─────────────────────────────────────── */
static int __init btn_irq_init(void)
{
    int ret;

    bdev = kzalloc(sizeof(*bdev), GFP_KERNEL);
    if (!bdev) return -ENOMEM;

    bdev->gpio = BUTTON_GPIO;
    spin_lock_init(&bdev->lock);
    init_waitqueue_head(&bdev->wq);
    INIT_WORK(&bdev->dwork, btn_work);

    /* Request GPIO */
    ret = gpio_request(bdev->gpio, "btn_irq");
    if (ret) { pr_err("gpio_request failed: %d\n", ret); goto err_gpio; }

    ret = gpio_direction_input(bdev->gpio);
    if (ret) { pr_err("gpio_direction_input failed\n"); goto err_dir; }

    /* Get IRQ number */
    bdev->irq = gpio_to_irq(bdev->gpio);
    if (bdev->irq < 0) {
        pr_err("gpio_to_irq failed: %d\n", bdev->irq);
        ret = bdev->irq; goto err_dir;
    }
    pr_info("btn_irq: GPIO%d → IRQ%d\n", bdev->gpio, bdev->irq);

    /* Request IRQ — falling edge (button press pulls GPIO low) */
    ret = request_irq(bdev->irq, btn_hard_irq,
                      IRQF_TRIGGER_FALLING,
                      "btn_irq", bdev);
    if (ret) { pr_err("request_irq failed: %d\n", ret); goto err_dir; }

    /* Register char device */
    ret = alloc_chrdev_region(&dev_num, 0, 1, DRIVER_NAME);
    if (ret) goto err_chrdev;

    cdev_init(&bdev->cdev, &btn_fops);
    bdev->cdev.owner = THIS_MODULE;
    ret = cdev_add(&bdev->cdev, dev_num, 1);
    if (ret) goto err_cdev;

    bclass = class_create(DRIVER_NAME);
    if (IS_ERR(bclass)) { ret = PTR_ERR(bclass); goto err_class; }

    if (IS_ERR(device_create(bclass, NULL, dev_num, NULL, DRIVER_NAME))) {
        ret = -ENODEV; goto err_device;
    }

    pr_info("btn_irq: ready on /dev/%s — press the button!\n", DRIVER_NAME);
    return 0;

err_device: class_destroy(bclass);
err_class:  cdev_del(&bdev->cdev);
err_cdev:   unregister_chrdev_region(dev_num, 1);
err_chrdev: free_irq(bdev->irq, bdev);
err_dir:    gpio_free(bdev->gpio);
err_gpio:   kfree(bdev);
    return ret;
}

static void __exit btn_irq_exit(void)
{
    cancel_work_sync(&bdev->dwork);  /* wait for any pending work */
    device_destroy(bclass, dev_num);
    class_destroy(bclass);
    cdev_del(&bdev->cdev);
    unregister_chrdev_region(dev_num, 1);
    free_irq(bdev->irq, bdev);
    gpio_free(bdev->gpio);
    kfree(bdev);
    pr_info("btn_irq: removed\n");
}

module_init(btn_irq_init);
module_exit(btn_irq_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Button IRQ driver with workqueue debounce");
MODULE_VERSION("1.0");
```

### Testing

```bash
# Build, deploy, load
make && scp btn_irq.ko pi@<RPI_IP>:/home/pi/
ssh pi@<RPI_IP> 'sudo insmod btn_irq.ko && sudo chmod 666 /dev/btn_irq'

# Verify IRQ registration
cat /proc/interrupts | grep btn_irq

# Blocking read — waits for button press
cat /dev/btn_irq
# press button → prints: press=1 time_ns=1234567890

# Non-blocking check from Python
python3 -c "
import os, errno
fd = os.open('/dev/btn_irq', os.O_RDONLY | os.O_NONBLOCK)
try:
    print(os.read(fd, 64))
except BlockingIOError:
    print('No button press yet')
os.close(fd)
"

# Measure IRQ latency with ftrace
sudo bash -c '
echo irq_handler_entry > /sys/kernel/debug/tracing/set_event
echo irq_handler_exit  >> /sys/kernel/debug/tracing/set_event
echo 1 > /sys/kernel/debug/tracing/tracing_on
'
# Press button 5 times
sudo bash -c '
echo 0 > /sys/kernel/debug/tracing/tracing_on
grep btn_irq /sys/kernel/debug/tracing/trace
'
# Calculate latency: difference between entry and exit timestamps

# Count IRQ occurrences
grep btn_irq /proc/interrupts
```

---

## 13. Interview Questions

---

**Q1. What is the difference between a top half and a bottom half in interrupt handling? Why do we need both?**

**Answer:** The **top half** is the actual interrupt service routine (ISR) registered with `request_irq()`. It runs immediately when the hardware interrupt fires, in interrupt context, with the interrupt line (and possibly all interrupts on that CPU) disabled. It must be extremely fast — typically under a microsecond. Its job is: acknowledge the hardware interrupt (clear the hardware interrupt flag), save minimal state about what happened, and schedule the bottom half.

The **bottom half** is deferred work that runs later, after the CPU returns from interrupt context. It handles the bulk of the interrupt processing: copying data, processing buffers, waking user-space processes, doing I/O. There are three mechanisms: **tasklets** (run in softirq context, can't sleep), **workqueues** (run in kernel thread context, can sleep), and **threaded IRQs** (dedicated thread per IRQ, can sleep).

We need both because the interrupt controller holds off other interrupts while the ISR runs (or at least the same interrupt). A slow ISR causes other interrupt sources to be delayed — increasing system-wide interrupt latency. By doing minimal work in the ISR and deferring the rest, we keep interrupt latency low for the entire system.

---

**Q2. What functions cannot be called from an interrupt handler and why?**

**Answer:** Any function that might sleep or block cannot be called from an interrupt handler. Specifically:

- **`kmalloc(GFP_KERNEL)`** — may sleep waiting for memory to be reclaimed. Use `GFP_ATOMIC` instead.
- **`mutex_lock()`** — calls `schedule()` when contended (sleeps).
- **`msleep()` / `schedule()` / `wait_event()`** — explicitly yield the CPU.
- **`copy_from_user()` / `copy_to_user()`** — may fault on absent user pages, triggering page fault handling which can sleep.
- **`printk()` with certain log levels** — while `printk()` itself is technically callable, it can be very slow in some configurations.
- **Any filesystem or block layer operations** — can block waiting for I/O.

The reason: interrupt handlers run with the CPU in interrupt context. The scheduler will not voluntarily run another task while in interrupt context — `schedule()` would panic with "BUG: scheduling while atomic." The interrupt must return quickly so the interrupted code (user process or kernel) can resume.

Enable `CONFIG_DEBUG_ATOMIC_SLEEP=y` during development — it will immediately print a warning + stack trace if any sleeping function is called from interrupt context.

---

**Q3. Explain `IRQF_ONESHOT`. When is it required?**

**Answer:** `IRQF_ONESHOT` instructs the kernel to keep the IRQ line **disabled** from when the hard IRQ handler returns until the threaded handler function completes. Without it, the IRQ line is re-enabled immediately when the hard IRQ handler returns (before the thread runs).

This is required for **level-triggered interrupts** with threaded handlers. A level-triggered interrupt fires continuously as long as the hardware condition is asserted (e.g., a UART RX FIFO has data). If the IRQ line is re-enabled before the thread handles the condition (reads the FIFO), the IRQ fires again immediately — creating an infinite interrupt loop that starves the system.

`IRQF_ONESHOT` breaks this loop: the IRQ line stays disabled, the thread runs, the thread reads the FIFO (clears the condition), then the kernel re-enables the IRQ. Only now can a new interrupt fire.

For edge-triggered interrupts (fire on transition, not level), `IRQF_ONESHOT` is technically not needed but is still good practice — it prevents duplicate IRQs if the hardware glitches during the threaded handler's execution.

---

**Q4. What is the difference between `tasklet_schedule()` and `schedule_work()`? When would you use each?**

**Answer:**

**`tasklet_schedule()`** queues a tasklet to run in **softirq context** (before returning to user space or returning from interrupt context). It cannot sleep. Runs on the same CPU that scheduled it. Very low latency — runs within microseconds of being scheduled. Use for: high-frequency, latency-sensitive bottom halves that don't need to sleep (e.g., network packet processing, audio sample handling).

**`schedule_work()`** queues a work item onto the global **workqueue** (`events` or similar kworker thread pool). Runs in a kernel thread — can sleep, use mutex, do I/O, allocate GFP_KERNEL memory. Latency is higher (the kworker thread must be scheduled). Use for: bottom halves that need to sleep, do significant processing, or access file systems.

In modern drivers, **threaded IRQs** (using `request_threaded_irq`) are often preferred over manual tasklets/workqueues because the kernel handles the thread lifecycle automatically. But explicit workqueues are still common when the deferred work is not directly tied to a single interrupt handler.

---

**Q5. What does `/proc/interrupts` show, and what can you diagnose from it?**

**Answer:** `/proc/interrupts` shows interrupt statistics. Each row is one interrupt:

```
 16:  4821231   4695312   GICv2  27 Level  arch_timer
 189:      42         0   pinctrl-bcm2835  27 Edge  btn_irq
```

Fields: IRQ number, count per CPU, interrupt controller, hardware IRQ number, trigger type, handler name.

What you can diagnose:

1. **Interrupt affinity**: If a NIC's IRQ only increments on CPU0 but you have 4 CPUs, you're not using multi-queue networking. Adjust via `/proc/irq/<N>/smp_affinity`.

2. **Interrupt storm**: If a counter is incrementing millions of times per second, a "screaming interrupt" might be happening — the device keeps asserting its IRQ because the handler isn't clearing it properly.

3. **Missing interrupts**: If your driver registered an IRQ but the count stays at 0 when you expect it to fire, the hardware isn't generating the interrupt, the IRQ number is wrong, or the trigger type doesn't match (e.g., expecting falling edge but hardware drives rising edge).

4. **Load balancing**: Uneven distribution across CPUs can cause latency on overloaded CPUs. Use `irqbalance` daemon or manual affinity tuning.

5. **Shared IRQ misidentification**: If `IRQ_NONE` return rate is high, your shared handler is being called for interrupts it doesn't own — find why the hardware isn't asserting your specific status bit.

---

**Q6. What happens if you call `free_irq()` while the interrupt handler is still running on another CPU?**

**Answer:** `free_irq()` is designed to handle this safely. It does NOT immediately free the handler. Instead, it:

1. Sets a flag marking the IRQ handler as being removed
2. Calls `synchronize_irq(irq)` internally, which **blocks** until any currently-executing instance of the handler completes on all CPUs
3. Only after the handler has fully returned on all CPUs does `free_irq()` remove the handler from the IRQ dispatch table and return

This is why `free_irq()` should be called **before** freeing any data structures the handler accesses. The correct sequence in cleanup is:

```c
free_irq(dev->irq, dev);     // blocks until handler is done
cancel_work_sync(&dev->work); // blocks until work is done
gpio_free(dev->gpio);
kfree(dev);                   // now safe to free
```

Never `kfree(dev)` before `free_irq(dev->irq, dev)` — the handler runs asynchronously on another CPU and dereferences `dev`.

---

> **Week 1 Complete!** You now understand the Linux kernel architecture, module development, character drivers, ioctl/poll/wait queues, memory management, synchronization primitives, and interrupt handling. Week 2 begins with the Linux device model (kobject, sysfs) and Device Tree — the foundation for all real hardware driver work.
