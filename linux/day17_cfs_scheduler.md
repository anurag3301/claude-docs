# Day 17 — CPU Scheduling: The Completely Fair Scheduler (CFS)

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand CFS internals — vruntime, the run queue red-black tree, wake-up preemption, load balancing basics, and how to observe scheduler behavior.

---

## 1. The Scheduler's Job

The scheduler answers one question, millions of times per second: **which task runs next on this CPU?**

It must balance:
- **Fairness**: every runnable task gets CPU proportional to its weight
- **Latency**: interactive tasks must wake up quickly (low response time)
- **Throughput**: batch jobs should not constantly yield to interactive tasks
- **Scalability**: decision must be fast (O(log n) or better) even with thousands of tasks

Before CFS (pre-2.6.23), Linux used an O(1) scheduler with fixed time slices. CFS replaced it with a fundamentally different model based on virtual time.

---

## 2. The CFS Mental Model: Virtual Time

CFS's key innovation: instead of fixed time slices, it tracks how much CPU time each task has received and always runs the one with the **least virtual runtime** (`vruntime`).

```
Imagine a perfect fair CPU that serves all tasks simultaneously:
  - 4 tasks with equal weight: each gets 25% of CPU
  - After 1 second of wall time: each has 0.25 seconds of CPU time

CFS emulates this: run the most-behind task first.
```

```
vruntime of 4 equal tasks over time:
              Task A ──────────────────────────────── (runs 0→1ms, vruntime=1)
              Task B         ──────────────────────── (runs 1→2ms, vruntime=1)
              Task C                  ──────────────── (runs 2→3ms, vruntime=1)
              Task D                          ──────── (runs 3→4ms, vruntime=1)
              ← All converge to same vruntime over time →
```

**Critical property**: tasks with higher nice values (lower priority) advance faster in vruntime per unit of CPU time — so they run less often.

---

## 3. The `sched_entity` and `cfs_rq`

```c
// include/linux/sched.h — scheduling entity (one per task)
struct sched_entity {
    struct load_weight     load;       // Weight (based on nice value)
    struct rb_node         run_node;   // Node in CFS run queue rbtree
    struct list_head       group_node; // For task groups
    unsigned int           on_rq;     // Is this entity on a run queue?
    
    u64                    exec_start; // When this task last started running
    u64                    sum_exec_runtime; // Total CPU time accumulated
    u64                    vruntime;   // Virtual runtime (THE key metric)
    u64                    prev_sum_exec_runtime; // At last context switch
    
    u64                    nr_migrations;  // How many times task moved between CPUs
    
    struct sched_statistics statistics;    // Latency stats (with CONFIG_SCHEDSTATS)
    // ...
};

// Per-CPU runnable task queue for CFS:
struct cfs_rq {
    struct load_weight     load;       // Total weight of all runnable tasks
    unsigned int           nr_running; // Number of tasks on this queue
    unsigned int           h_nr_running; // Including task group children
    
    u64                    exec_clock; // CFS clock (advances as tasks run)
    u64                    min_vruntime; // Smallest vruntime on this queue
                                        // Used as baseline for new tasks
    
    struct rb_root_cached  tasks_timeline; // Rbtree of tasks, sorted by vruntime
    // rb_root_cached caches the leftmost node = task with min vruntime
    
    struct sched_entity   *curr;       // Currently running task's se
    struct sched_entity   *next;       // Next task to run (after curr)
    struct sched_entity   *last;       // Task that just ran (buddy for preemption)
    struct sched_entity   *skip;       // Skip this task for one round
    // ...
};
```

---

## 4. `vruntime` Computation

The `vruntime` increase per unit of wall-clock time is weighted by nice/priority:

```c
// kernel/sched/fair.c

// Weight table indexed by nice value (-20 to 19):
// Nice -20 → weight 88761 (highest priority, slowest vruntime advance)
// Nice   0 → weight 1024  (baseline)
// Nice  19 → weight 15    (lowest priority, fastest vruntime advance)
const int sched_prio_to_weight[40] = {
    /* -20 */    88761,  71755,  56483,  46273,  36291,
    /* -15 */    29154,  23254,  18705,  14949,  11916,
    /* -10 */     9548,   7620,   6100,   4904,   3906,
    /*  -5 */     3121,   2501,   1991,   1586,   1277,
    /*   0 */     1024,    820,    655,    526,    423,
    /*   5 */      335,    272,    215,    172,    137,
    /*  10 */      110,     87,     70,     56,     45,
    /*  15 */       36,     29,     23,     18,     15,
};

// vruntime update formula:
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;
    
    if (unlikely(!curr))
        return;
    
    // Wall-clock time since last update:
    delta_exec = now - curr->exec_start;
    curr->exec_start = now;
    
    // Update total CPU time:
    curr->sum_exec_runtime += delta_exec;
    
    // Compute vruntime delta:
    // delta_vruntime = delta_exec * (NICE_0_LOAD / curr->load.weight)
    // = delta_exec * 1024 / curr_weight
    // Higher weight → smaller vruntime increment → runs longer before preempted
    
    u64 delta_vruntime = calc_delta_fair(delta_exec, curr);
    curr->vruntime += delta_vruntime;
    
    // Update queue's min_vruntime:
    update_min_vruntime(cfs_rq);
}
```

### 4.1 Effect of Nice Values

```
Task A: nice= 0, weight=1024
Task B: nice=+5, weight= 335

For 10ms of wall-clock time running:
  Task A vruntime += 10ms * (1024/1024) = 10ms
  Task B vruntime += 10ms * (1024/335)  = 30.5ms

Result: Task B accumulates vruntime 3x faster.
After Task B runs 10ms, it's "behind" by 30.5ms vs Task A's 10ms.
CFS picks Task A next (smallest vruntime).
Net effect: Task A gets ~3x more CPU than Task B.
```

---

## 5. The Run Queue: Red-Black Tree

All runnable tasks are in a red-black tree keyed by `vruntime`:

```
                     [50ms]
                    /       \
               [30ms]       [70ms]
              /     \       /     \
           [20ms] [40ms] [60ms] [80ms]
           
Leftmost node = smallest vruntime = next task to run
```

```c
// Inserting a task into the run queue:
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_root.rb_node;
    struct rb_node *parent = NULL;
    struct sched_entity *entry;
    bool leftmost = true;
    
    while (*link) {
        parent = *link;
        entry = rb_entry(parent, struct sched_entity, run_node);
        
        if (entity_before(se, entry)) {  // se->vruntime < entry->vruntime
            link = &parent->rb_left;
        } else {
            link = &parent->rb_right;
            leftmost = false;  // Not the leftmost node
        }
    }
    
    rb_link_node(&se->run_node, parent, link);
    rb_insert_color_cached(&se->run_node, &cfs_rq->tasks_timeline, leftmost);
    // rb_insert_color_cached maintains the leftmost pointer if leftmost=true
}

// Pick next task to run (leftmost = min vruntime):
static struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);
    if (!left)
        return NULL;
    return rb_entry(left, struct sched_entity, run_node);
}
```

The `rb_root_cached` maintains a pointer to the leftmost node, making `__pick_first_entity()` O(1) (just dereference the cached pointer).

---

## 6. `pick_next_task_fair()` — Choosing the Next Task

```c
// kernel/sched/fair.c
static struct task_struct *
pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;
    struct task_struct *p;
    
    // 1. Put previous task back into the run queue (if still runnable):
    if (prev->sched_class == &fair_sched_class)
        put_prev_task_fair(rq, prev);
    
    // 2. Pick the leftmost entity (min vruntime):
    se = pick_next_entity(cfs_rq, NULL);
    // pick_next_entity considers:
    // - The leftmost node (min vruntime)
    // - The "buddy" system: cfs_rq->next and cfs_rq->last
    //   (to reduce unnecessary preemptions)
    
    // 3. Handle task groups (cgroup CPU scheduling):
    // Groups have their own cfs_rq; walk down the hierarchy
    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        set_next_entity(cfs_rq, se);
    }
    
    p = task_of(se);  // Get task_struct from sched_entity
    
    return p;
}
```

---

## 7. Preemption Decision

CFS doesn't use fixed time slices. Instead, it preempts the current task when:

1. A new task wakes up with a smaller `vruntime` (it's more "behind")
2. The current task has run for at least `sysctl_sched_min_granularity`

```c
// kernel/sched/fair.c
static void check_preempt_wakeup_fair(struct rq *rq, struct task_struct *p, int wake_flags)
{
    struct task_struct *curr = rq->curr;
    struct sched_entity *se = &curr->se;
    struct sched_entity *pse = &p->se;
    
    // Don't preempt if current task has a wakeup buddy (to reduce bouncing):
    if (test_tsk_need_resched(curr))
        return;
    
    // Compute how much the waking task is "behind":
    // If waking task's vruntime < curr->vruntime - threshold → preempt
    int scale = cfs_rq->nr_running >= sched_nr_latency;
    
    if (wakeup_preempt_entity(se, pse) == 1) {
        // Waking task deserves to run NOW
        resched_curr(rq);  // Sets TIF_NEED_RESCHED flag
        // Actual preemption happens at next schedule() call or interrupt return
    }
}

// The vruntime comparison:
static int wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se)
{
    s64 gran = wakeup_gran(curr);
    s64 vdiff = curr->vruntime - se->vruntime;
    
    if (vdiff <= 0)        // Waking task is ahead — don't preempt
        return -1;
    
    if (vdiff > gran)      // Waking task is significantly behind — preempt
        return 1;
    
    return 0;              // Similar vruntime — let current task continue
}
```

---

## 8. `schedule()` — The Context Switch

```c
// kernel/sched/core.c
asmlinkage __visible void __sched schedule(void)
{
    struct task_struct *tsk = current;
    
    sched_submit_work(tsk);  // Flush pending work before sleeping
    
    do {
        preempt_disable();
        __schedule(SM_NONE);  // The real work
        sched_preempt_enable_no_resched();
    } while (need_resched());
}

static void __sched notrace __schedule(unsigned int sched_mode)
{
    struct task_struct *prev, *next;
    struct rq_flags rf;
    struct rq *rq;
    int cpu;
    
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);     // This CPU's run queue
    prev = rq->curr;
    
    local_irq_disable();
    rq_lock(rq, &rf);
    
    // Update clock:
    update_rq_clock(rq);
    
    // Dequeue current task if it's going to sleep:
    if (prev->state != TASK_RUNNING) {
        // Task is going to sleep: remove from run queue
        deactivate_task(rq, prev, DEQUEUE_SLEEP);
    }
    
    // Pick next task:
    next = pick_next_task(rq, prev, &rf);
    
    clear_tsk_need_resched(prev);
    
    if (likely(prev != next)) {
        // Do the actual context switch:
        rq->nr_switches++;
        rq->curr = next;
        context_switch(rq, prev, next, &rf);
        // context_switch() does NOT return in 'prev' context!
        // It returns in 'next' context
    } else {
        // Same task — just unlock and return
        rq_unpin_lock(rq, &rf);
        __balance_callbacks(rq);
        local_irq_enable();
    }
}
```

### 8.1 `context_switch()` — The Assembly Level

```c
// kernel/sched/core.c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
               struct task_struct *next, struct rq_flags *rf)
{
    // Switch memory context:
    if (!next->mm) {
        // Kernel thread: borrow prev's address space
        next->active_mm = prev->active_mm;
        if (prev->mm)
            mmgrab_lazy_tlb(prev->active_mm);
        enter_lazy_tlb(prev->active_mm, next);
    } else {
        // User process: switch page tables (change CR3 on x86)
        switch_mm_irqs_off(prev->active_mm, next->mm, next);
        lru_gen_use_mm(next->mm);
    }
    
    // Switch CPU register state (TSS, FPU, etc.):
    // arch/x86/kernel/process_64.c:
    switch_to(prev, next, prev);
    // This is a macro that:
    // 1. Saves prev's RSP (stack pointer)
    // 2. Saves prev's IP (instruction pointer, via pusha/call)
    // 3. Loads next's RSP
    // 4. Returns (using next's saved IP)
    // After switch_to: we're now executing in 'next's context
    
    return finish_task_switch(prev);  // Cleanup of 'prev' after switch
}
```

```asm
// arch/x86/include/asm/switch_to.h (simplified):
#define switch_to(prev, next, last)                 \
do {                                                \
    asm volatile(                                   \
    "pushq %%rbp\n\t"           /* Save prev's RBP */ \
    "movq %%rsp, %P[prev_sp](%[prev])\n\t"  /* Save prev's RSP */ \
    "movq %P[next_sp](%[next]), %%rsp\n\t"  /* Load next's RSP */ \
    "movq $1f, %P[prev_ip](%[prev])\n\t"   /* Save prev's IP */   \
    "pushq %P[next_ip](%[next])\n\t"        /* Push next's IP */   \
    "ret\n\t"                   /* Jump to next task! */           \
    "1:\t"                      /* prev resumes here next time */  \
    "popq %%rbp\n\t"            /* Restore prev's RBP */          \
    ...
    );                                              \
} while (0)
```

After `switch_to`, execution continues in the `next` task's context. `prev` is saved and will resume from the `1:` label when it's scheduled again.

---

## 9. Task Wake-Up: `wake_up_process()`

```c
// kernel/sched/core.c
int wake_up_process(struct task_struct *p)
{
    return try_to_wake_up(p, TASK_NORMAL, 0);
}

static int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    struct rq *rq;
    
    // 1. Ensure the sleeping task's stores are visible:
    smp_mb__before_spinlock();
    
    // 2. Acquire the run queue lock:
    rq = __task_rq_lock(p, &rf);
    
    // 3. Verify task is in the right state:
    if (!(p->state & state))
        goto out;
    
    // 4. Place task on a CPU's run queue:
    ttwu_queue(p, cpu, wake_flags);
    
    // ttwu_queue → ttwu_do_activate → activate_task:
    //   enqueue_task_fair(rq, p, ...)
    //   → enqueue_entity(cfs_rq, &p->se, ...)
    //   → __enqueue_entity(cfs_rq, se) [insert into rbtree]
    
    // 5. Check if new task should preempt current:
    check_preempt_curr(rq, p, wake_flags);
    // → check_preempt_wakeup_fair() → possibly sets TIF_NEED_RESCHED
    
    p->state = TASK_RUNNING;
    
out:
    __task_rq_unlock(rq, &rf);
    return success;
}
```

---

## 10. vruntime Initialization for New Tasks

When a task is created or woken after sleeping, its `vruntime` must be set correctly:

```c
// kernel/sched/fair.c

// New task: start at current min_vruntime (don't let it run forever)
static void place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se,
                          int flags)
{
    u64 vruntime = cfs_rq->min_vruntime;
    
    // Waking up after sleep: subtract a bonus (to reduce latency):
    if (flags & ENQUEUE_WAKEUP) {
        // Wakeup preemption bonus: allow waking task to run before the
        // current task if it's sufficiently behind
        vruntime -= sched_latency_ns;
        // (capped to avoid going too far back)
    }
    
    // Never set vruntime below min_vruntime:
    se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```

The sleep bonus (`-= sched_latency_ns`) is why interactive tasks (which frequently sleep and wake) get low latency — each wake gives them a credit.

---

## 11. CFS Tunables

```bash
# Key tunables in /proc/sys/kernel/:

# Target latency — all runnable tasks get one turn within this window:
cat /proc/sys/kernel/sched_latency_ns
# 6000000 (6ms) — how long it takes all tasks to get a turn

# Minimum granularity — minimum time a task runs before preemption:
cat /proc/sys/kernel/sched_min_granularity_ns
# 750000 (0.75ms) — prevents thrashing

# Wakeup preemption granularity:
cat /proc/sys/kernel/sched_wakeup_granularity_ns
# 1000000 (1ms) — waking task must be this much ahead to preempt

# How to tune for different workloads:
# Low-latency interactive (e.g., audio):
echo 1000000 > /proc/sys/kernel/sched_latency_ns
echo 100000 > /proc/sys/kernel/sched_min_granularity_ns

# High-throughput batch:
echo 24000000 > /proc/sys/kernel/sched_latency_ns
echo 3000000 > /proc/sys/kernel/sched_min_granularity_ns
```

---

## 12. Nice Values and Weight

```bash
# Set nice value at launch:
nice -n 10 myprogram      # Lower priority (nice=10)
nice -n -5 myprogram      # Higher priority (nice=-5, requires CAP_SYS_NICE)

# Change running process nice value:
renice -n 5 -p $$         # Set current shell to nice=5
renice -n -10 -p 1234     # Set PID 1234 to nice=-10 (root only)

# See current nice values:
ps -o pid,ni,comm aux | head -20
# NI column: -20 (highest priority) to 19 (lowest)
```

```c
// Kernel: setting priority from nice value
static int effective_prio(struct task_struct *p)
{
    // CFS tasks: prio = DEFAULT_PRIO + nice = 120 + nice
    // Range: 100 (nice=-20) to 139 (nice=19)
    return NICE_TO_PRIO(task_nice(p));
}

// CFS weight from prio:
static void set_load_weight(struct task_struct *p, bool update_load)
{
    int prio = p->static_prio - MAX_RT_PRIO;  // CFS range: 0-39
    struct load_weight *load = &p->se.load;
    
    load->weight = scale_load(sched_prio_to_weight[prio]);
    load->inv_weight = sched_prio_to_wmult[prio];  // Precomputed inverse
}
```

---

## 13. Task Groups and Cgroup CPU Scheduling

With `CONFIG_FAIR_GROUP_SCHED`, groups of tasks are scheduled as a unit:

```
CPU bandwidth allocation with cgroups:
  /sys/fs/cgroup/cpu/database/cpu.weight = 200   (2x weight)
  /sys/fs/cgroup/cpu/webapp/cpu.weight   = 100   (1x weight)

Result: database tasks get ~66% CPU, webapp tasks ~33%

# Set CPU weight (cgroup v2):
echo 200 > /sys/fs/cgroup/mygroup/cpu.weight

# Set CPU bandwidth (hard limit):
echo "50000 100000" > /sys/fs/cgroup/mygroup/cpu.max
# = 50ms per 100ms period = 50% max CPU for the group
```

Internally, the scheduler uses a two-level structure: each cgroup has its own `cfs_rq`, and tasks within the group are scheduled against the group's `cfs_rq`. The groups themselves are scheduled against the CPU's `cfs_rq`.

---

## 14. Observing the Scheduler

```bash
# Per-task scheduling statistics:
cat /proc/$$/sched
# Fields include:
# se.exec_start         — when task last started executing
# se.vruntime           — current virtual runtime
# se.sum_exec_runtime   — total CPU time
# nr_switches           — total context switches
# nr_voluntary_switches — task called schedule() itself (I/O waits, etc.)
# nr_involuntary_switches — preempted by scheduler

# System-wide scheduler stats:
cat /proc/schedstat
# Per-CPU statistics about scheduling decisions

# perf: sample scheduler events
sudo perf sched record -- sleep 5
sudo perf sched latency           # Show scheduling latency histogram
sudo perf sched timehist          # Timeline of scheduling events

# bpftrace: trace schedule() calls
sudo bpftrace -e '
kprobe:__schedule {
    @[comm] = count();
}'

# Context switch rate:
sudo bpftrace -e '
tracepoint:sched:sched_switch {
    @cs_rate = count();
}
interval:s:1 { print(@cs_rate); clear(@cs_rate); }'

# Latency tracing (how long tasks wait on the run queue):
sudo perf stat -e sched:sched_switch,sched:sched_wakeup -a sleep 5
```

---

## 15. `SCHED_BATCH` and `SCHED_IDLE` Policies

```c
// SCHED_BATCH (policy=3):
// Like SCHED_NORMAL (CFS) but:
// - Never preempts SCHED_NORMAL tasks (wakeup won't preempt)
// - Hint to the scheduler: "this is CPU-bound batch work, not interactive"
// - CPU affinity is relaxed (migrated more freely between CPUs)
// Use for: background compression, backup, batch processing

// SCHED_IDLE (policy=5):
// Even lower than nice+19
// Only runs when nothing else is runnable
// Used for: truly idle work, seL4 idle task, desktop cube effects

// Setting from user space:
struct sched_param param = { .sched_priority = 0 };
sched_setscheduler(pid, SCHED_BATCH, &param);
sched_setscheduler(pid, SCHED_IDLE, &param);

// Equivalent with chrt:
chrt --batch 0 ./myjob
chrt --idle  0 ./myjob
```

---

## 16. The Scheduler Class Hierarchy

CFS is one of several scheduler classes. They run in strict priority order:

```c
// kernel/sched/sched.h
// Scheduler classes (highest to lowest priority):
// 1. stop_sched_class    — stop_machine(), migration threads
// 2. dl_sched_class      — SCHED_DEADLINE (EDF)
// 3. rt_sched_class      — SCHED_FIFO, SCHED_RR
// 4. fair_sched_class    — SCHED_NORMAL, SCHED_BATCH, SCHED_IDLE (CFS)
// 5. idle_sched_class    — swapper/N (idle threads)

// pick_next_task() tries each class in order:
static struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    const struct sched_class *class;
    
    // Optimization: if all runnable tasks are CFS:
    if (likely(prev->sched_class <= &fair_sched_class &&
               rq->nr_running == rq->cfs.h_nr_running)) {
        return pick_next_task_fair(rq, prev, rf);
    }
    
    // Full search: RT/DL tasks may be present:
    for_each_class(class) {
        p = class->pick_next_task(rq, prev, rf);
        if (p)
            return p;
    }
}
```

Any runnable RT or deadline task always preempts CFS tasks.

---

## 17. Scheduler Tracing with `ftrace`

```bash
# Enable the scheduler wakeup tracer (measures wake-to-run latency):
cd /sys/kernel/debug/tracing
echo wakeup > current_tracer
echo 1 > tracing_on
sleep 1
echo 0 > tracing_on
cat trace | head -50

# Output shows time from wake event to task actually running:
#   TASK-PID    CPU#    |||||    TIMESTAMP   FUNCTION
#   bash-1234   [001]   .....   1234.567890: #1: prio=120
#   bash-1234   [001]   .....   1234.567891: waking up bash:1234
#   bash-1234   [001]   .....   1234.567910: running         ← 19µs latency

# Function graph tracer to see schedule() internals:
echo function_graph > current_tracer
echo schedule > set_graph_function
cat trace | head -30
```

---

## 18. Mental Model Checkpoint

After Day 17, you should be able to:

1. Explain `vruntime` — what it is, how it advances faster for low-priority tasks.
2. Why does CFS use a red-black tree instead of a sorted array or linked list?
3. Trace `pick_next_task_fair()` — what is `rb_first_cached()` and why is it O(1)?
4. What happens to a task's `vruntime` when it wakes from sleep? Why?
5. What is `TIF_NEED_RESCHED` and what sets/clears it?
6. Explain the context switch: what does `switch_to()` save and restore?
7. What is `sched_latency_ns` and how does it affect responsiveness vs throughput?
8. Why does SCHED_BATCH exist? What's different about its wakeup behavior?
9. In what order do scheduler classes run? What preempts a CFS task?
10. What is the scheduler wakeup tracer and what does it measure?

---

## Key Source Files

```bash
kernel/sched/fair.c         # CFS: pick_next_task_fair, update_curr, place_entity
kernel/sched/core.c         # schedule(), context_switch(), try_to_wake_up()
kernel/sched/sched.h        # struct cfs_rq, struct sched_class, rq
include/linux/sched.h       # struct sched_entity, task_struct scheduling fields
arch/x86/kernel/process_64.c  # switch_to (context switch assembly)
Documentation/scheduler/    # Official CFS documentation
Documentation/scheduler/sched-design-CFS.rst  # CFS design document (READ THIS)
```

---

## Summary

CFS elegantly solves scheduling fairness through a single abstraction: **virtual runtime**. Tasks with small `vruntime` are "behind" and run next; tasks that run accumulate `vruntime` at a rate inversely proportional to their priority weight.

The implementation is:
- **Run queue**: a red-black tree sorted by `vruntime`, O(log n) insert/delete, O(1) min-find
- **Preemption**: when a high-priority task wakes, its `vruntime` is small enough to trigger `TIF_NEED_RESCHED`
- **Context switch**: saves RSP/RBP/IP, switches mm (CR3 on x86), switches to new task's kernel stack
- **Sleep bonus**: waking tasks get a `vruntime` credit proportional to `sched_latency_ns`

Tomorrow: real-time and deadline scheduling — `SCHED_FIFO`, `SCHED_RR`, and `SCHED_DEADLINE`.
