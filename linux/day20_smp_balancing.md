# Day 20 — SMP & Load Balancing

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand per-CPU run queues, the scheduling domain hierarchy, load balancing algorithms, NUMA-aware scheduling, and CPU hotplug.

---

## 1. One Run Queue Per CPU

The fundamental SMP scheduling design: **every CPU has its own run queue**. There is no global lock protecting a single shared queue.

```c
// kernel/sched/sched.h
struct rq {
    // This run queue's statistics:
    unsigned int nr_running;       // Total runnable tasks
    unsigned long cpu_load[5];     // Load averages (1, 4, 8, 16, 32 ticks)
    
    // The three scheduler sub-queues:
    struct cfs_rq cfs;     // CFS (normal) tasks
    struct rt_rq  rt;      // RT tasks
    struct dl_rq  dl;      // Deadline tasks
    
    struct task_struct *curr;     // Currently running task
    struct task_struct *idle;     // This CPU's idle task (swapper/N)
    struct task_struct *stop;     // stop_machine task (migration/N)
    
    unsigned long next_balance;   // When to next check for imbalance
    
    // Clocks:
    u64 clock;                    // rq clock (monotonic, per-CPU)
    u64 clock_task;               // Task clock (only advances when a task runs)
    
    // Migration:
    struct list_head cfs_tasks;   // CFS tasks on this CPU
    
    // CPU:
    int cpu;                      // Which CPU this rq belongs to
    int online;                   // CPU is online
    
    // Lock:
    raw_spinlock_t lock;          // Protects this run queue
};

// Per-CPU run queues:
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);
#define cpu_rq(cpu)     (&per_cpu(runqueues, (cpu)))
#define this_rq()       this_cpu_ptr(&runqueues)
#define task_rq(p)      cpu_rq(task_cpu(p))
```

**Benefits of per-CPU queues**:
- No global lock contention (huge scalability win)
- Tasks tend to stay on their CPU (better cache utilization)
- Each CPU can make scheduling decisions independently

**Cost**: tasks might be unevenly distributed → need periodic load balancing.

---

## 2. The Scheduling Domain Hierarchy

The kernel models the CPU topology (caches, sockets, NUMA) as a hierarchy of **scheduling domains**:

```
Example: 2-socket system, 8 cores each, HT disabled

NUMA Domain (span=all 16 CPUs)
├── MC (Multi-Core) Domain, Node 0 (span=CPUs 0-7)
│   ├── CPU 0 run queue
│   ├── CPU 1 run queue
│   ├── ...
│   └── CPU 7 run queue
└── MC Domain, Node 1 (span=CPUs 8-15)
    ├── CPU 8 run queue
    ├── ...
    └── CPU 15 run queue

With HyperThreading:
SMT Domain per core (2 HT siblings per domain)
└── MC Domain per socket
    └── NUMA Domain
```

```c
// include/linux/sched/topology.h
struct sched_domain {
    struct sched_domain __rcu *parent;  // Next level up
    struct sched_domain __rcu *child;   // Next level down
    struct sched_group *groups;         // Groups of CPUs at this level
    
    unsigned long min_interval;   // Minimum load balance interval (ms)
    unsigned long max_interval;   // Maximum load balance interval
    
    unsigned int busy_factor;     // Imbalance factor for busy CPUs
    unsigned int imbalance_pct;   // Imbalance threshold (%)
    
    unsigned int flags;           // SD_LOAD_BALANCE, SD_BALANCE_NEWIDLE, etc.
    
    cpumask_var_t span;           // CPUs covered by this domain
    
    struct rq *rq;                // Run queue for this domain's CPUs
    int id;                       // Domain ID
};

struct sched_group {
    struct sched_group *next;     // Next group in same domain
    atomic_t ref;
    unsigned int group_weight;    // Total CPU weight in group
    cpumask_var_t cpumask;        // CPUs in this group
    struct sched_group_capacity *sgc;  // Capacity info
};
```

---

## 3. Load Balancing Algorithm

Load balancing runs periodically (controlled by timers) and on specific events:

### 3.1 Trigger Points

```c
// 1. Periodic rebalancing (via scheduler tick):
// kernel/sched/core.c
void scheduler_tick(void)
{
    // ...
    trigger_load_balance(rq);  // Called every tick from timer interrupt
}

// 2. CPU goes idle (pull tasks immediately):
// kernel/sched/fair.c
static void newidle_balance(struct rq *this_rq, struct rq_flags *rf)
{
    // When CPU becomes idle: immediately try to pull work from busiest CPU
}

// 3. Task is woken (push to least-loaded CPU):
// try_to_wake_up() → select_task_rq_fair()
```

### 3.2 `load_balance()` — The Core Algorithm

```c
// kernel/sched/fair.c
static int load_balance(int this_cpu, struct rq *this_rq,
                        struct sched_domain *sd,
                        enum cpu_idle_type idle,
                        int *continue_balancing)
{
    struct sched_group *group;
    struct rq *busiest;
    
    // 1. Find the busiest group in this domain:
    group = find_busiest_group(&env);
    if (!group)
        goto out_balanced;
    
    // 2. Find the busiest CPU in that group:
    busiest = find_busiest_queue(&env, group);
    if (!busiest)
        goto out_balanced;
    
    // 3. Calculate how many tasks to move:
    ld_moved = 0;
    if (busiest->nr_running > 1) {
        // Move tasks from busiest to this_rq:
        ld_moved = detach_tasks(&env);
        // detach_tasks: removes tasks from busiest's rbtree
        // attach_tasks: adds them to this_rq's rbtree
        attach_tasks(&env);
    }
    
    return ld_moved;
}
```

### 3.3 Imbalance Calculation

```c
// What counts as "imbalanced"?
// The domain's imbalance_pct threshold (default 117% = 17% imbalance OK):

// Example: 2 CPUs, domain imbalance_pct = 117
// CPU 0: 10 tasks (load=10)
// CPU 1:  5 tasks (load=5)
// Average: 7.5 tasks
// CPU 0 is 10/7.5 = 133% of average → exceeds 117% → IMBALANCED
// Move (10-5)/2 = 2-3 tasks from CPU 0 to CPU 1

// But: tasks can have CPU affinity (pinned to specific CPUs)
// And: NUMA tasks prefer to stay on their socket
// And: cache-hot tasks are not moved
```

---

## 4. CPU Affinity and Migration

### 4.1 CPU Affinity

```c
// Set which CPUs a task can run on:
// User space:
taskset -c 0,1,2,3 myprogram          # Restrict to CPUs 0-3
taskset -p 0xF 1234                    # Set affinity mask for PID 1234

// Kernel:
cpumask_t mask;
cpumask_clear(&mask);
cpumask_set_cpu(0, &mask);
cpumask_set_cpu(1, &mask);
sched_setaffinity(task->pid, &mask);

// Check/get affinity:
sched_getaffinity(pid, cpusetsize, mask);
```

```c
// In task_struct:
cpumask_t cpus_mask;   // Allowed CPUs (set by affinity API)
// load_balance will not move task to CPUs outside cpus_mask
```

### 4.2 Task Migration Between CPUs

When load balancing moves a task, it goes through migration:

```c
// kernel/sched/core.c
static int migration_cpu_stop(void *data)
{
    struct migration_arg *arg = data;
    struct task_struct *p = arg->task;
    int dest_cpu = arg->dest_cpu;
    struct rq *rq = this_rq();
    
    // Move the task:
    move_queued_task(rq, p, dest_cpu);
    // = dequeue_task(rq, p) + enqueue_task(cpu_rq(dest_cpu), p)
    
    return 0;
}

// The migration is executed via stop_machine():
// The stop machine mechanism runs a function on a specific CPU
// while stopping all other CPUs temporarily
```

```bash
# See how often tasks migrate between CPUs:
cat /proc/$$/status | grep numa_pages_migrated

# perf: count migrations:
sudo perf stat -e migrations -a sleep 5
# migrations: 234   ← tasks migrated between CPUs in 5 seconds

# Per-process migration count (in /proc/<pid>/sched):
cat /proc/$$/sched | grep nr_migrations
```

---

## 5. Wake-Up CPU Selection: `select_task_rq_fair()`

When a task is woken from sleep, the scheduler must choose which CPU to place it on:

```c
// kernel/sched/fair.c
static int select_task_rq_fair(struct task_struct *p, int prev_cpu,
                                int wake_flags)
{
    int sync = !!(wake_flags & WF_SYNC);  // Waker and wakee on same CPU?
    struct sched_domain *tmp, *sd = NULL;
    int cpu = smp_processor_id();         // Current CPU
    int new_cpu = prev_cpu;               // Default: stay on previous CPU
    int want_affine = 0;
    
    // 1. Try affine wakeup (place near waker for cache sharing):
    if (wake_flags & WF_TTWU_QUEUE) {
        // The waker is on this CPU; waking on same CPU is often best
        // for producer-consumer patterns
    }
    
    // 2. Find the best idle CPU in the scheduling domain:
    for_each_domain(cpu, tmp) {
        if (want_affine && (tmp->flags & SD_WAKE_AFFINE)) {
            int target = wake_affine(tmp, p, prev_cpu, sync);
            if (target != prev_cpu)
                return target;
        }
    }
    
    // 3. Find the least-loaded CPU:
    new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);
    
    return new_cpu;
}
```

Key heuristics:
- **Idle CPU preference**: always prefer an idle CPU
- **Affinity preference**: woken tasks tend to go back to their previous CPU (good for cache reuse)
- **Waker affinity**: if waker and wakee communicate heavily (producer-consumer), place them on the same core/socket

---

## 6. NUMA-Aware Scheduling

On NUMA systems, memory bandwidth and latency depend on which NUMA node the memory is on. The scheduler tracks this:

### 6.1 NUMA Balancing

`CONFIG_NUMA_BALANCING` enables automatic NUMA migration: the kernel tracks which NUMA node a task's pages are on and tries to migrate tasks (and pages) together.

```c
// kernel/sched/fair.c — NUMA balancing:

// Periodically: fault in task's pages as NUMA hint faults
// Task accesses page → page fault → kernel records which CPU generated the fault
// If task accesses remote NUMA pages frequently → migrate task to that NUMA node

struct numa_group {
    refcount_t refcount;
    spinlock_t lock;
    int nr_tasks;
    pid_t gid;
    int active_nodes;
    
    struct rcu_head rcu;
    unsigned long total_faults;
    unsigned long max_faults_cpu;
    
    // Fault counts per node per task-group:
    unsigned long faults[];
};
```

```bash
# Enable NUMA balancing (default on NUMA systems):
cat /proc/sys/kernel/numa_balancing
# 1 = enabled

# NUMA statistics:
numastat -p $(pgrep -n mysqld)
# Per-node memory usage for mysqld

# Per-node run time:
cat /proc/$$/numa_maps | head -5

# NUMA scheduling stats:
cat /proc/schedstat | head -3
```

### 6.2 NUMA Groups

Tasks that share memory (same address space = threads, or mmap-sharing processes) can be grouped:

```c
// When two tasks access the same physical page from different CPUs:
// → They're probably communicating
// → The scheduler groups them and tries to keep them on the same NUMA node

// sched_numa_find_next_cpu() prefers CPUs on the same NUMA node as task's memory
```

---

## 7. Load vs Utilization

The scheduler tracks two distinct concepts:

### 7.1 Load (Weighted)

```c
// "Load" = sum of task weights on a CPU
// A SCHED_NORMAL task with nice=0 has weight=1024
// A nice=-10 task has weight=9548
// Load of a CPU = sum of weights of all runnable tasks

// CFS load is tracked via PELT (Per-Entity Load Tracking):
struct sched_avg {
    u64 last_update_time;
    u64 load_sum;
    u64 runnable_sum;
    u32 util_sum;
    u32 period_contrib;
    unsigned long load_avg;      // Exponential moving average of load
    unsigned long runnable_avg;  // Exponential moving average of runnability
    unsigned long util_avg;      // Exponential moving average of utilization
};
```

### 7.2 Utilization (Capacity)

```c
// "Utilization" = how much of a CPU's capacity a task uses
// Ranges 0 to SCHED_CAPACITY_SCALE (1024)

// Used for:
// 1. Energy-aware scheduling: place task on CPU with enough capacity
//    but not more than needed (save power)
// 2. Capacity-aware scheduling: avoid overloading heterogeneous CPUs

// big.LITTLE/EAS (Energy-Aware Scheduling):
// Task with util=200 fits on a LITTLE core (capacity=256)
// Task with util=900 needs a big core (capacity=1024)
```

---

## 8. CPU Hotplug

CPUs can be brought online/offline at runtime:

```bash
# Offline CPU 3 (remove from scheduling):
echo 0 > /sys/devices/system/cpu/cpu3/online

# Online CPU 3:
echo 1 > /sys/devices/system/cpu/cpu3/online

# See online CPUs:
cat /sys/devices/system/cpu/online   # e.g., 0-2,4-7 (3 offline)
cat /sys/devices/system/cpu/present  # All CPUs that physically exist
cat /sys/devices/system/cpu/possible # CPUs that could possibly be online
```

```c
// When a CPU goes offline:
// 1. Tasks on that CPU are migrated to other CPUs
// 2. IRQ handlers are moved to other CPUs
// 3. Per-CPU resources are freed

// kernel/sched/core.c
static int sched_cpu_dying(unsigned int cpu)
{
    struct rq *rq = cpu_rq(cpu);
    struct rq_flags rf;
    
    // Migrate all tasks off this CPU:
    migrate_tasks(rq, &rf);
    
    // Clean up the run queue:
    rq->online = 0;
    
    return 0;
}
```

---

## 9. `isolcpus` and `nohz_full`

For latency-sensitive workloads, remove CPUs from the general scheduler:

```bash
# Kernel command line:
isolcpus=2,3        # Isolate CPUs 2-3 from scheduler
nohz_full=2,3       # Full dynticks: no timer interrupts when 1 task running
rcu_nocbs=2,3       # Offload RCU callbacks from these CPUs

# Effect:
# - General-purpose tasks never scheduled on CPUs 2,3
# - CPUs 2,3 only run tasks explicitly assigned via taskset
# - Timer tick suppressed on CPUs 2,3 when only 1 task runs
#   → Eliminates ~250 timer interrupts/second of jitter

# Assign RT task to isolated CPU:
taskset -c 2 chrt -f 80 ./my_rt_task

# Verify: no other tasks on CPU 2
ps -o pid,psr,comm aux | awk '$2==2'
```

---

## 10. Load Balancing Domains and Flags

```c
// SD_* flags control when load balancing happens:
SD_LOAD_BALANCE      // Do load balancing in this domain
SD_BALANCE_NEWIDLE   // Balance when CPU goes idle
SD_BALANCE_EXEC      // Rebalance when exec() is called
SD_BALANCE_FORK      // Rebalance at fork
SD_BALANCE_WAKE      // Rebalance at task wake-up
SD_WAKE_AFFINE       // Try to place woken task near waker
SD_ASYM_CPUCAPACITY  // Heterogeneous CPUs (big.LITTLE)
SD_SHARE_CPUCAPACITY // HyperThreaded siblings share capacity
SD_SHARE_LLC         // CPUs in same LLC (last-level cache)
SD_NUMA              // NUMA domain (cross-socket balancing)

// Balance intervals increase with domain level:
// SMT domain: rebalance every 1ms
// MC (core) domain: rebalance every 4ms  
// Socket (NUMA) domain: rebalance every 64ms
```

---

## 11. `sched_debug` — Observing the Scheduler

```bash
# Comprehensive scheduler state dump:
cat /proc/sched_debug

# Output includes:
# Sched Debug Version: v0.11
# ktime: 123456789012345
# sched_clk: 123456789012345
# cpu#0, 2500.000 MHz
#   .nr_running: 3
#   .load: 3072
#   .nr_switches: 12345
#   cfs_rq[0]:/
#   .exec_clock: 98765432
#   .MIN_vruntime: 0.123456789s
#   .min_vruntime: 9876543210
#   .max_vruntime: 9876543211
#   runnable tasks:
#           task   PID         tree-key  switches  prio  wait-time...
#            bash  1234   9876.123456789     1234   120   0.000000000

# Per-task scheduling info:
cat /proc/$$/sched

# System load averages (1min, 5min, 15min):
cat /proc/loadavg
# 0.52 0.38 0.31 2/345 12345
# ↑ Running tasks / Total tasks ↑ Last PID
```

---

## 12. Tracing Load Balancing

```bash
# ftrace: trace load balance events:
cd /sys/kernel/debug/tracing
echo sched:sched_migrate_task > set_event
echo 1 > tracing_on
sleep 5
cat trace | grep sched_migrate_task | head -20
# sched_migrate_task: comm=mysqld pid=1234 orig_cpu=0 dest_cpu=3

# bpftrace: trace migrations:
sudo bpftrace -e '
tracepoint:sched:sched_migrate_task {
    @migrations[args->orig_cpu, args->dest_cpu] = count();
}'

# perf: CPU migration events:
sudo perf record -e sched:sched_migrate_task -ag sleep 10
sudo perf script | head -50
```

---

## 13. Scheduler Tuning for Different Workloads

```bash
# HPC/MPI workloads: bind processes to CPUs, disable migration:
numactl --cpunodebind=0 --membind=0 ./mpi_job
# OR: use CPU sets
cgcreate -g cpuset:/hpc
echo 0-7 > /sys/fs/cgroup/cpuset/hpc/cpuset.cpus
echo 0 > /sys/fs/cgroup/cpuset/hpc/cpuset.mems
cgclassify -g cpuset:/hpc $$

# Database workloads: NUMA-aware memory allocation
numactl --localalloc ./mysqld

# Web servers: let the scheduler do its job
# (usually default settings are fine)

# Latency-sensitive (trading, audio):
# Use isolcpus + SCHED_FIFO + taskset as shown in Day 18

# Scheduler domain interval tuning:
# For latency: reduce balance intervals
echo 1 > /proc/sys/kernel/sched_domain/cpu0/domain0/min_interval
echo 4 > /proc/sys/kernel/sched_domain/cpu0/domain0/max_interval
```

---

## 14. Mental Model Checkpoint

After Day 20, you should be able to:

1. Why does Linux use per-CPU run queues instead of a single global queue?
2. Describe the scheduling domain hierarchy on a 2-socket NUMA system.
3. When does load balancing run? Name three trigger points.
4. What is the difference between "load" and "utilization" in the scheduler?
5. How does `select_task_rq_fair()` choose which CPU to wake a task on?
6. What does `isolcpus=` do in the kernel command line?
7. What is NUMA balancing and what does it try to achieve?
8. How does a task migrate from one CPU to another? What mechanism does it use?

---

## Key Source Files

```bash
kernel/sched/fair.c              # load_balance(), select_task_rq_fair(), NUMA
kernel/sched/topology.c          # sched_domain hierarchy construction
kernel/sched/sched.h             # struct rq, struct sched_domain
kernel/sched/core.c              # scheduler_tick(), sched_cpu_dying()
include/linux/sched/topology.h   # SD_* flags
Documentation/scheduler/sched-domains.rst  # Scheduling domain documentation
```

---

## Summary

SMP scheduling in Linux is built on per-CPU run queues connected by a scheduling domain hierarchy that reflects the system's topology (SMT siblings, cache domains, NUMA nodes).

Load balancing runs periodically and on events (idle CPU, new task, task wakeup) to keep work distributed across CPUs. The algorithm considers task weights (load), actual CPU utilization, CPU affinity masks, and NUMA topology.

For latency-critical workloads, `isolcpus` removes CPUs from general scheduling, and tasks can be pinned with `taskset` + `chrt` for completely predictable behavior.

Tomorrow: signals — how the kernel delivers asynchronous events to processes.
