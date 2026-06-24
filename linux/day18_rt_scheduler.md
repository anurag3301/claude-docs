# Day 18 — CPU Scheduling: Real-Time & Deadline

> **Estimated read time:** 90 minutes  
> **Goal:** Understand SCHED_FIFO, SCHED_RR, SCHED_DEADLINE, priority ranges, and what PREEMPT_RT does to the kernel.

---

## 1. What "Real-Time" Means

In the kernel context, "real-time" means **deterministic response time**, not "fast." A real-time task must complete its work within a bounded deadline, every time.

Linux offers three real-time scheduling policies:
- `SCHED_FIFO` — run until it blocks or yields (no time slice)
- `SCHED_RR` — round-robin with time slices within same priority
- `SCHED_DEADLINE` — EDF (Earliest Deadline First) with bandwidth guarantees

These run at higher priority than any CFS (`SCHED_NORMAL`) task. A single runnable RT task will starve all normal tasks.

---

## 2. Priority Ranges

```
Priority ranges (internal kernel values):
  0           = SCHED_DEADLINE tasks (dl_sched_class, separate queue)
  1–99        = RT priorities (rt_sched_class)
               1 = lowest RT priority
               99 = highest RT priority
  100         = CFS highest (nice = -20)
  120         = CFS normal (nice = 0)  ← "DEFAULT_PRIO"
  139         = CFS lowest (nice = +19)

User-space priority mapping:
  SCHED_FIFO/RR:  sched_priority 1–99 → internal priority 99–1
  (inverted! user 99 = internal 1 = highest RT)
  SCHED_NORMAL:   sched_priority = 0, nice -20 to 19
```

```bash
# See current policy and priority:
chrt -p $$          # For current shell
# scheduling policy: SCHED_OTHER (= SCHED_NORMAL)
# scheduling priority: 0

chrt -p 1234        # For PID 1234

# System RT tasks:
ps -eo pid,rtprio,cls,comm | grep -v '-'
# PID   RTPRIO CLS COMMAND
#    9       99  FF migration/0   ← SCHED_FIFO priority 99 (migration thread)
#   10       99  FF watchdog/0    ← SCHED_FIFO priority 99
#   34        1  RR kworker/...   ← SCHED_RR priority 1
```

---

## 3. `SCHED_FIFO` — Non-Preemptible RT

**Rules:**
- Runs until it voluntarily yields (`sched_yield()`), sleeps, or is preempted by a **higher-priority** RT task
- No time slice — can run indefinitely
- Tasks of the **same** priority: FIFO ordering (first one in runs until it yields)

```c
// Set SCHED_FIFO from kernel:
struct sched_param sp = { .sched_priority = 50 };
sched_setscheduler(current, SCHED_FIFO, &sp);

// From user space:
struct sched_param sp = { .sched_priority = 50 };
sched_setscheduler(0, SCHED_FIFO, &sp);  // 0 = current process
// OR:
chrt -f 50 ./myprogram    # Launch with SCHED_FIFO priority 50

// Required: CAP_SYS_NICE or RLIMIT_RTPRIO
```

**Danger**: a buggy SCHED_FIFO task that loops infinitely will freeze the system (all CPUs blocked).

Protection: `CONFIG_RT_THROTTLE` limits RT tasks to 95% of CPU by default:
```bash
cat /proc/sys/kernel/sched_rt_period_us   # 1000000 (1 second)
cat /proc/sys/kernel/sched_rt_runtime_us  # 950000 (950ms = 95%)
# Total RT runtime per second is 950ms
# Remaining 50ms goes to normal tasks (prevents full starvation)

# Disable throttling (production RT systems):
echo -1 > /proc/sys/kernel/sched_rt_runtime_us
```

---

## 4. `SCHED_RR` — Round-Robin RT

Like `SCHED_FIFO` but with a time quantum. Tasks at the same priority take turns:

```c
// Time quantum for SCHED_RR:
cat /proc/sys/kernel/sched_rr_timeslice_ms   # Default: 100ms

// When the quantum expires:
// - Task is moved to the TAIL of its priority queue
// - Next task at same priority runs
// - If no other task at same priority: original task continues

struct sched_param sp = { .sched_priority = 50 };
sched_setscheduler(0, SCHED_RR, &sp);
// OR:
chrt -r 50 ./myprogram
```

---

## 5. RT Run Queue Structure

```c
// kernel/sched/sched.h
struct rt_rq {
    struct rt_prio_array    active;      // Bitmask + queues for all 100 RT prios
    unsigned int            rt_nr_running; // Count of runnable RT tasks
    unsigned int            rr_nr_running; // Count of SCHED_RR tasks
    
    struct rt_bandwidth     rt_bandwidth; // For RT throttling
    
    int                     overloaded;  // More RT tasks than CPUs?
    struct plist_head       pushable_tasks; // RT tasks that can be migrated
};

// The active array: O(1) scheduling (bitmap + array of lists)
struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO + 1);  // 100-bit bitmap
    struct list_head queue[MAX_RT_PRIO];       // One list per priority level
};
```

```c
// pick_next_task_rt():
static struct task_struct *pick_next_task_rt(struct rq *rq, ...)
{
    struct rt_rq *rt_rq = &rq->rt;
    struct rt_prio_array *array = &rt_rq->active;
    
    // Find highest non-empty priority level:
    int idx = sched_find_first_bit(array->bitmap);
    
    // Take the first task from that priority's queue:
    struct rt_se = list_first_entry(&array->queue[idx],
                                     struct sched_rt_entity, run_list);
    return rt_task_of(se);
}
```

O(1) scheduling: `find_first_bit` on a 100-bit bitmap is essentially constant time.

---

## 6. `SCHED_DEADLINE` — EDF Scheduling

`SCHED_DEADLINE` implements the **Earliest Deadline First** (EDF) algorithm — the theoretically optimal real-time scheduling policy.

### 6.1 The Three Parameters

```
Task parameters:
  runtime  = how much CPU time the task needs per period (in ns)
  deadline = relative deadline from job activation (in ns)
  period   = how often the task activates (in ns)
  
Example: video codec needing 5ms of CPU every 33ms (30fps):
  runtime  = 5,000,000 (5ms)
  deadline = 33,333,333 (33.3ms)
  period   = 33,333,333 (33.3ms)
  utilization = runtime/period = 15% of one CPU
```

```c
// Set SCHED_DEADLINE from user space:
#include <linux/sched/types.h>

struct sched_attr attr = {
    .size           = sizeof(attr),
    .sched_policy   = SCHED_DEADLINE,
    .sched_runtime  = 5 * 1000000,     // 5ms
    .sched_deadline = 33 * 1000000,    // 33ms
    .sched_period   = 33 * 1000000,    // 33ms
};
syscall(SYS_sched_setattr, 0, &attr, 0);

// Or from command line:
chrt --deadline --sched-runtime 5000000 \
     --sched-deadline 33000000 \
     --sched-period 33000000 \
     0 ./video_codec
```

### 6.2 How EDF Works

At each scheduling decision: **run the task with the earliest absolute deadline**.

```
Example: 3 tasks, scheduling at time T=0
  Task A: deadline = T+10ms
  Task B: deadline = T+5ms   ← smallest deadline → runs first
  Task C: deadline = T+20ms

After B runs for its runtime budget:
  Task A: deadline = T+10ms  ← smallest remaining → runs next
  Task C: deadline = T+20ms  ← runs last
```

### 6.3 Admission Control

The kernel enforces that total utilization ≤ 100% per CPU (with NUMA-aware spreading):

```c
// kernel/sched/deadline.c
static int dl_overflow(struct task_struct *p, int policy,
                       const struct sched_attr *attr)
{
    struct dl_bw *dl_b = dl_bw_of(task_cpu(p));
    
    // Total utilization check:
    // sum(runtime_i / period_i) <= 1.0 for each CPU
    u64 new_bw = to_ratio(attr->sched_period, attr->sched_runtime);
    
    if (dl_b->total_bw + new_bw > dl_b->bw) {
        // Admission refused: would overcommit this CPU
        return -EBUSY;
    }
    // ...
}
```

This is **admission control** — the kernel refuses to schedule a task that would make it impossible to meet all deadlines. Unlike SCHED_FIFO/RR, deadline tasks have mathematical guarantees.

### 6.4 `sched_yield_to_deadline()` — Replenishment

When a deadline task finishes its work before its runtime budget expires, it calls `sched_yield()`. The bandwidth budget is replenished at the next period.

```c
// Sporadic server model:
// - Task has runtime=5ms, period=33ms
// - Each period: runtime "refills" to 5ms
// - Task can only use 5ms per period, then sleeps

// Internal: hrtimer fires at period boundary, calls:
static enum hrtimer_restart dl_task_timer(struct hrtimer *timer)
{
    // Replenish runtime:
    dl_se->runtime = dl_se->dl_runtime;
    // Update absolute deadline:
    dl_se->deadline = rq_clock(rq) + dl_se->dl_deadline;
    // Re-enqueue the task:
    enqueue_task_dl(rq, p, ENQUEUE_REPLENISH);
    return HRTIMER_NORESTART;
}
```

---

## 7. RT Throttling in Detail

```bash
# Watch RT throttling in action:
grep -i throttl /proc/sched_debug | head -5

# If RT tasks get throttled:
dmesg | grep "sched: RT throttling activated"
# This means: RT tasks used their full 950ms/1000ms quota

# Per-cgroup RT bandwidth:
cat /sys/fs/cgroup/cpu/mygroup/cpu.rt_runtime_us   # 950000
cat /sys/fs/cgroup/cpu/mygroup/cpu.rt_period_us    # 1000000

# cgroup v2 doesn't expose RT bandwidth (use SCHED_DEADLINE instead)
```

---

## 8. Priority Inversion and PI Mutexes

A classic real-time problem: low-priority task L holds a mutex; high-priority H needs the mutex; medium M preempts L. H waits for M (which preempts L which holds the mutex H needs) → **priority inversion**.

Linux solution: **Priority Inheritance (PI) mutexes**:

```c
#include <pthread.h>

// Normal mutex — no PI:
pthread_mutex_t m;
pthread_mutex_init(&m, NULL);

// PI mutex:
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_t pi_mutex;
pthread_mutex_init(&pi_mutex, &attr);

// When H blocks on pi_mutex held by L:
// Kernel temporarily raises L's priority to H's priority
// L can now preempt M and release the mutex quickly
// H gets the mutex, L returns to its original priority
```

```c
// Kernel PI mutex:
#include <linux/rtmutex.h>

struct rt_mutex my_mutex;
rt_mutex_init(&my_mutex);

rt_mutex_lock(&my_mutex);
// ... critical section ...
rt_mutex_unlock(&my_mutex);
```

The Mars Pathfinder in 1997 had a priority inversion bug that caused a system reset. PI mutexes prevent this.

---

## 9. `PREEMPT_RT` — The Real-Time Kernel Patch

The `CONFIG_PREEMPT_RT` patch set (now partially upstream) converts the Linux kernel into a hard real-time OS.

### 9.1 What PREEMPT_RT Does

**Problem**: Even with `CONFIG_PREEMPT`, there are sections where interrupts are disabled, spinlocks are held, or the kernel is otherwise non-preemptible. These cause **latency spikes** in RT tasks.

PREEMPT_RT changes:

1. **Spinlocks → sleeping mutexes**: `spinlock_t` is replaced by sleeping mutexes (on RT systems). Most spinlocks in drivers become preemptible.

2. **Interrupt handlers → threads**: Almost all hardirq handlers become threaded IRQ handlers running in kernel threads at `SCHED_FIFO` priority.

3. **Local IRQ disable → mutex**: `local_irq_disable()` is replaced by a per-CPU mutex.

4. **Timer softirq → RT thread**: `HRTIMER_SOFTIRQ` runs in an RT kernel thread.

```bash
# Check if PREEMPT_RT is active:
uname -a
# Linux host 6.9.0-rt9 #1 SMP PREEMPT_RT ...  ← RT kernel
grep PREEMPT_RT /boot/config-$(uname -r)
# CONFIG_PREEMPT_RT=y

# Measure worst-case latency with cyclictest:
sudo apt install rt-tests
sudo cyclictest -m -p 80 -n -t1 -q --duration=60
# Min/Avg/Max latencies in microseconds
# A good RT system: max < 100µs
# Non-RT Linux: max can be milliseconds or more
```

### 9.2 PREEMPT_RT Impact on Drivers

```c
// Normal kernel: spinlock is non-sleeping:
static DEFINE_SPINLOCK(my_lock);
spin_lock(&my_lock);      // Disables preemption, non-sleeping

// PREEMPT_RT: spinlock is a sleeping lock (rt_spinlock = rt_mutex):
// spin_lock() may now sleep!
// Consequence: code that does spin_lock() is now preemptible

// Proper PREEMPT_RT-safe spinlock (when you really need non-sleeping):
raw_spinlock_t raw_lock;
raw_spin_lock_init(&raw_lock);
raw_spin_lock(&raw_lock);    // NEVER sleeps, even on PREEMPT_RT
raw_spin_unlock(&raw_lock);
```

For driver developers: use `raw_spinlock_t` only for the smallest, most critical sections (< 10µs). Everything else can use regular `spinlock_t` (which RT makes preemptible).

---

## 10. Measuring RT Latency

```bash
# cyclictest: the standard RT latency test
# Measures time from timer expiration to task actually running:
sudo cyclictest \
    --mlockall \           # Lock all memory (no page faults)
    --threads=4 \          # 4 measurement threads
    --priority=80 \        # RT priority 80
    --interval=1000 \      # 1ms timer interval
    --duration=300 \       # Run for 5 minutes
    --histfile=latency.hist # Save histogram

# Output:
# T: 0 (145678) P:80 I:1000 C:  300000 Min:      2 Act:    4 Avg:    4 Max:    38

# Key metric: MAX latency (microseconds)
# Desktop Linux: 500–5000µs max
# Server Linux + PREEMPT: 100–500µs max  
# PREEMPT_RT kernel: 10–100µs max
# Hardware RT (Xenomai/RTAI): < 10µs max

# hwlatdetect: find hardware-caused latency:
sudo hwlatdetect --duration=60 --threshold=10
# Finds: SMI (System Management Interrupts) which can steal CPU for 50-500µs
```

---

## 11. IRQ Priorities with Threaded IRQs

With threaded IRQs (PREEMPT_RT), interrupt handlers become kernel threads with `SCHED_FIFO` priority. You can tune their priority:

```bash
# See IRQ thread priorities:
ps -eo pid,rtprio,cls,comm | grep 'irq/'
# 123   50  FF irq/24-nvme0q1    ← NVMe IRQ thread, priority 50
# 124   50  FF irq/25-nvme0q2
# 125   50  FF irq/26-eth0

# Change IRQ thread priority:
# Method 1: via /proc/irq/<n>/
# (only changes CPU affinity, not priority directly)

# Method 2: chrt on the IRQ thread:
chrt -f -p 70 $(pgrep 'irq/24-nvme0q1')

# Method 3: via udev/irqbalance configuration
```

---

## 12. Practical RT System Setup

```bash
# Isolate CPUs for RT tasks (no other tasks on CPU 2-3):
# In /etc/default/grub:
GRUB_CMDLINE_LINUX="isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3"
# Update grub and reboot

# Pin RT task to isolated CPU:
taskset -c 2 chrt -f 80 ./my_rt_task

# Disable CPU frequency scaling (constant frequency = predictable timing):
for cpu in 2 3; do
    cpufreq-set -c $cpu -g performance
done

# Disable C-states (deep sleep → high wake latency):
for cpu in 2 3; do
    echo 1 > /sys/devices/system/cpu/cpu${cpu}/cpuidle/state*/disable
done

# Lock memory (prevent page faults during RT execution):
# In your RT program:
#include <sys/mman.h>
mlockall(MCL_CURRENT | MCL_FUTURE);
```

---

## 13. Scheduler Classes and POSIX Compliance

```c
// POSIX defines:
// SCHED_OTHER  = 0  → SCHED_NORMAL (CFS)
// SCHED_FIFO   = 1  → SCHED_FIFO
// SCHED_RR     = 2  → SCHED_RR

// Linux extensions:
// SCHED_BATCH    = 3  (GNU extension)
// SCHED_IDLE     = 5  (GNU extension)
// SCHED_DEADLINE = 6  (Linux-specific)

// POSIX priority range query:
int min_prio = sched_get_priority_min(SCHED_FIFO);  // 1
int max_prio = sched_get_priority_max(SCHED_FIFO);  // 99

// Get/set scheduler:
struct sched_param sp;
sched_getscheduler(pid);                  // Returns policy
sched_getparam(pid, &sp);                 // Gets priority
sched_setscheduler(pid, policy, &sp);     // Sets both
```

---

## 14. Mental Model Checkpoint

After Day 18, you should be able to:

1. What is the difference between `SCHED_FIFO` and `SCHED_RR`?
2. How does the RT scheduler choose the next task? What data structure does it use?
3. What is RT throttling and why does it exist?
4. Explain `SCHED_DEADLINE`'s three parameters: runtime, deadline, period.
5. What is admission control in `SCHED_DEADLINE`?
6. What is priority inversion? How do PI mutexes fix it?
7. What three things does `PREEMPT_RT` change in the kernel?
8. What is `cyclictest` and what does it measure?
9. What is `raw_spinlock_t` and when should you use it instead of `spinlock_t`?

---

## Key Source Files

```bash
kernel/sched/rt.c          # RT scheduler: SCHED_FIFO, SCHED_RR
kernel/sched/deadline.c    # SCHED_DEADLINE: EDF implementation
kernel/locking/rtmutex.c   # PI mutex implementation
kernel/sched/core.c        # pick_next_task(), sched_setscheduler()
include/uapi/linux/sched.h # SCHED_* policy constants
include/linux/sched/rt.h   # RT-related scheduler declarations
Documentation/scheduler/sched-deadline.rst  # SCHED_DEADLINE spec
Documentation/timers/hrtimers.rst          # hrtimer documentation
```

---

## Summary

Linux RT scheduling layers from highest to lowest priority:

1. **`SCHED_DEADLINE`** (dl_sched_class): EDF with admission control. Guaranteed latency if utilization ≤ 100%.
2. **`SCHED_FIFO`** (rt_sched_class): Runs until voluntary yield or higher-priority preemption. No time slice.
3. **`SCHED_RR`** (rt_sched_class): Like FIFO but with a time quantum (100ms default) for equal-priority tasks.
4. **CFS** (`SCHED_NORMAL`, `SCHED_BATCH`, `SCHED_IDLE`): Only runs when no RT/DL task is runnable.

`PREEMPT_RT` converts the kernel to a hard RT system by making spinlocks sleepable and converting interrupt handlers to threads — but requires careful driver adaptation.

Tomorrow: kernel preemption — when and how the kernel can be preempted, and what the preemption count protects.
