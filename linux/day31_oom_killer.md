# Day 31 — OOM Killer

> **Estimated read time:** 90 minutes  
> **Goal:** Understand when the OOM killer fires, how `oom_badness()` scores processes, how `oom_score_adj` controls it, and how cgroup-level OOM works.

---

## 1. When Does the OOM Killer Fire?

The OOM (Out Of Memory) killer is a last resort. The kernel reaches it only after exhausting:

1. `kswapd` background reclaim
2. Direct reclaim (allocating task reclaims pages itself)
3. Memory compaction (rearranging pages to satisfy high-order allocations)
4. Retry loops with progressively lower watermarks

```c
// mm/page_alloc.c — the slowpath eventually calls:
static page *__alloc_pages_may_oom(gfp_t gfp_mask, unsigned int order,
                                    const struct alloc_context *ac,
                                    unsigned long *did_some_progress)
{
    struct oom_control oc = {
        .zonelist  = ac->zonelist,
        .nodemask  = ac->nodemask,
        .memcg     = NULL,
        .gfp_mask  = gfp_mask,
        .order     = order,
    };
    
    // Only one OOM killer at a time system-wide:
    if (!mutex_trylock(&oom_lock))
        return NULL;  // Another OOM is already running; wait
    
    // Can we reclaim enough before killing?
    if (oom_reaping_in_progress()) {
        // Previous OOM victim is being reaped; give it time
        schedule_timeout_uninterruptible(1);
        goto out;
    }
    
    // Commit to OOM kill:
    out_of_memory(&oc);
    
out:
    mutex_unlock(&oom_lock);
    return NULL;
}
```

---

## 2. `out_of_memory()` — The Entry Point

```c
// mm/oom_kill.c
bool out_of_memory(struct oom_control *oc)
{
    unsigned long freed = 0;
    
    // Last-chance: try notifiers first (registered by drivers/subsystems):
    blocking_notifier_call_chain(&oom_notify_list, 0, &freed);
    if (freed > 0)
        return true;  // Someone freed memory — no kill needed!
    
    // Check for kernel thread causing OOM (memory leak in driver etc.):
    if (is_memoryless_node(task_to_cpumask(current)) &&
        !should_oom_kill_current(oc))
        return true;
    
    // Try to find and kill a process:
    select_bad_process(oc);
    
    if (!oc->chosen) {
        // Nothing to kill — perhaps all processes are kernel threads
        // or are already being killed
        dump_header(oc, NULL);
        pr_warn("Out of memory and no killable processes...\n");
        panic("System is deadlocked on memory\n");
    }
    
    // Kill the chosen victim:
    oom_kill_process(oc, "Out of memory");
    return true;
}
```

---

## 3. `select_bad_process()` — Choosing the Victim

```c
// mm/oom_kill.c
static void select_bad_process(struct oom_control *oc)
{
    oc->chosen_points = LONG_MIN;
    
    // Iterate over all tasks:
    rcu_read_lock();
    for_each_process_thread(g, p) {
        long points;
        
        // Skip kernel threads (no mm):
        if (is_global_init(p) || p->flags & PF_KTHREAD)
            continue;
        
        // Skip processes that are already dying:
        if (task_will_free_mem(p))
            continue;
        
        // Skip processes that are memcg-immune:
        if (oc->memcg && !task_in_mem_cgroup(p, oc->memcg))
            continue;
        
        // Compute score:
        points = oom_badness(p, oc->totalpages);
        
        // Choose process with highest score:
        if (points > oc->chosen_points) {
            oc->chosen_points = points;
            oc->chosen = p;
        }
    }
    rcu_read_unlock();
}
```

---

## 4. `oom_badness()` — The Scoring Algorithm

This is the core of the OOM killer. It scores each process from 0 to 1000:

```c
// mm/oom_kill.c
long oom_badness(struct task_struct *p, unsigned long totalpages)
{
    long points;
    long adj;
    
    // oom_score_adj: user-controllable bias (-1000 to +1000)
    // -1000 = never kill this process
    // +1000 = kill this process first
    adj = (long)p->signal->oom_score_adj;
    
    // oom_score_adj = -1000: absolutely exempt from OOM killing
    if (adj == OOM_SCORE_ADJ_MIN)  // -1000
        return LONG_MIN;
    
    // Base score: memory usage as fraction of total memory
    // Count: RSS + swap usage + page table memory
    points = get_mm_rss(p->mm) +    // Pages in RAM
             get_mm_counter(p->mm, MM_SWAPENTS) +  // Pages in swap
             mm_pgtables_bytes(p->mm) / PAGE_SIZE;  // Page table pages
    
    // Adjust based on oom_score_adj:
    // Scaled: adj/1000 × totalpages added to points
    // adj = +1000: adds totalpages to score (always highest score)
    // adj = -999: subtracts almost all points
    adj *= totalpages / 1000;
    points += adj;
    
    // Result: 0 to (totalpages × 2) typically
    // Normalized to 0-1000 range:
    return points * 1000 / totalpages;
}
```

### 4.1 What Gets Counted

```bash
# See the actual OOM score for processes:
cat /proc/$$/oom_score
# 15  ← Score from 0-1000 (higher = more likely to be killed)

# This is approximately: RSS / total_pages × 1000 + oom_score_adj

# Example scores:
cat /proc/1/oom_score          # init/systemd: 0 (usually)
cat /proc/$(pgrep chromium)/oom_score  # 350 (large memory user)
cat /proc/$(pgrep mysqld)/oom_score    # 200

# Historical note: before Linux 2.6.36, OOM scoring was complex
# (considered CPU time, nice value, etc.). The current algorithm
# is intentionally simple: kill the biggest memory user first.
```

---

## 5. `oom_score_adj` — Controlling the Kill Priority

```bash
# Read current adj value:
cat /proc/$$/oom_score_adj
# 0  ← Default: score unchanged

# Range: -1000 to +1000
# -1000 = NEVER kill this process (reserved for systemd, init)
# -500  = Much less likely to be killed (protect important daemons)
# 0     = Default (score based only on memory)
# +500  = More likely to be killed (expendable processes)
# +1000 = Kill this first (container sandboxes, worker processes)

# Set it:
echo -500 > /proc/$(pgrep mysqld)/oom_score_adj   # Protect mysqld
echo 1000 > /proc/$(pgrep chrome)/oom_score_adj    # Chrome is expendable
echo -1000 > /proc/1/oom_score_adj                 # Never kill init
```

```c
// In a daemon's startup code (common pattern):
// Protect a critical process:
int protect_from_oom(void)
{
    int fd = open("/proc/self/oom_score_adj", O_WRONLY);
    if (fd < 0) return -1;
    write(fd, "-1000", 5);
    close(fd);
    return 0;
}
```

### 5.1 `oom_adj` vs `oom_score_adj`

```bash
# Old interface (deprecated, Linux < 2.6.36):
cat /proc/$$/oom_adj          # Range: -17 to +15
# -17 = exempt, 0 = default, 15 = kill first

# New interface (use this):
cat /proc/$$/oom_score_adj    # Range: -1000 to +1000

# systemd sets oom_score_adj for processes it manages:
# /etc/systemd/system/myservice.service
# [Service]
# OOMScoreAdjust=-500
```

---

## 6. `oom_kill_process()` — Executing the Kill

```c
// mm/oom_kill.c
static void oom_kill_process(struct oom_control *oc, const char *message)
{
    struct task_struct *victim = oc->chosen;
    
    // Print OOM kill reason to kernel log:
    dump_header(oc, victim);
    // dmesg output:
    // Out of memory: Killed process 1234 (mysqld) total-vm:2097152kB,
    //   anon-rss:1048576kB, file-rss:65536kB, shmem-rss:0kB,
    //   UID:1000 pgtables:4096kB oom_score_adj:0
    
    // Send SIGKILL to ALL threads of the process:
    do_send_sig_info(SIGKILL, SEND_SIG_PRIV, victim, PIDTYPE_TGID);
    
    // Mark the mm as under OOM reaping (prevents future OOM kills of same mm):
    set_bit(MMF_OOM_SKIP, &victim->mm->flags);
    
    // Start oom_reaper thread to free memory quickly:
    wake_up_process(oom_reaper_th);
    
    pr_err("Killed process %d (%s), adj %hd\n",
           task_pid_nr(victim), victim->comm,
           victim->signal->oom_score_adj);
}
```

---

## 7. The OOM Reaper

Simply sending SIGKILL isn't enough — the process must run to handle the signal, which requires CPU and memory. The **OOM reaper** aggressively frees memory without waiting for the process to exit:

```c
// mm/oom_kill.c
static int oom_reaper(void *unused)
{
    for (;;) {
        // Wait for a victim to reap:
        wait_event_freezable(oom_reaper_wait, oom_reaper_list != NULL);
        
        struct task_struct *tsk = oom_reaper_list;
        
        // Aggressively unmap and free all anonymous pages:
        oom_reap_task(tsk);
        // oom_reap_task():
        //   mmap_read_trylock(mm)
        //   unmap_page_range() on all VMAs
        //   TLB flush
        //   Return pages to allocator immediately
        // Does NOT wait for process to exit!
    }
}
```

The reaper runs as a kernel thread (`oom_reaper`) and can free memory within milliseconds even if the victim process is stuck.

---

## 8. OOM in cgroup v2

cgroup memory limits cause contained OOM kills (kill within the cgroup before the global OOM killer fires):

```bash
# Set memory limit for a cgroup:
echo "512M" > /sys/fs/cgroup/myapp/memory.max

# When the cgroup hits the limit:
# 1. Kernel tries to reclaim within the cgroup
# 2. If reclaim fails: OOM kill within the cgroup

# Configure cgroup OOM behavior:
cat /sys/fs/cgroup/myapp/memory.oom.group
# 0 = kill individual task that triggered OOM (default)
# 1 = kill the entire cgroup (all processes) when any task triggers OOM

echo 1 > /sys/fs/cgroup/myapp/memory.oom.group  # Kill the whole group

# Monitor cgroup OOM events:
cat /sys/fs/cgroup/myapp/memory.events
# low 0
# high 0
# max 0        ← how many times hit limit
# oom 0        ← OOM events
# oom_kill 0   ← processes killed by OOM in this cgroup

# Subscribe to OOM events via inotify or epoll:
inotifywait -e modify /sys/fs/cgroup/myapp/memory.events
```

### 8.1 Per-cgroup `oom_score_adj`

```bash
# systemd applies OOM score adj to all processes in a service's cgroup:
# /etc/systemd/system/myservice.service
# [Service]
# OOMScoreAdjust=-200          # Protect this service (all its processes)
# OOMPolicy=continue           # Don't kill on OOM (continue running)
# OOMPolicy=stop               # Stop the unit on OOM
# OOMPolicy=kill               # Kill the unit on OOM (default)
```

---

## 9. Interpreting OOM Killer Messages

```bash
# Typical dmesg output after OOM:
dmesg | grep -A 50 "Out of memory"

# Out of memory: Killed process 4567 (java) total-vm:8388608kB,
#   anon-rss:3145728kB, file-rss:131072kB, shmem-rss:0kB,
#   UID:1001 pgtables:16384kB oom_score_adj:0

# total-vm:8388608kB = 8GB virtual address space
# anon-rss:3145728kB = 3GB anonymous pages in RAM (heap/stack)
# file-rss:131072kB  = 128MB file-backed pages in RAM
# shmem-rss:0kB      = Shared memory pages in RAM
# pgtables:16384kB   = 16MB of page tables
# oom_score_adj:0    = No manual adjustment

# Memory dump also shows:
# Active: 4096 kB    Inactive: 2048 kB    Unevictable: 512 kB
# Active(anon): 2048 kB   Inactive(anon): 512 kB
# ...
```

---

## 10. Preventing OOM: Memory Overcommit

Linux overcommits memory by default — it promises more virtual memory than physical RAM + swap:

```bash
# Overcommit policy:
cat /proc/sys/vm/overcommit_memory
# 0 = Heuristic overcommit (allow reasonable overcommit, deny obvious failures)
# 1 = Always overcommit (never fail malloc, even if impossible to honor)
# 2 = Never overcommit (strict: only commit up to swap + fraction of RAM)

cat /proc/sys/vm/overcommit_ratio
# 50  ← With mode 2: commit up to 50% of RAM + all swap

# Check how much is committed:
cat /proc/meminfo | grep Commit
# CommitLimit:   25165824 kB  ← Maximum committable (overcommit_ratio based)
# Committed_AS:  15728640 kB  ← Currently committed
```

```c
// mm/util.c — overcommit check on mmap/brk:
int __vm_enough_memory(struct mm_struct *mm, long pages, int cap_sys_admin)
{
    long allowed;
    
    // Mode 1: always allow
    if (sysctl_overcommit_memory == OVERCOMMIT_ALWAYS)
        return 0;
    
    // Mode 0: heuristic
    if (sysctl_overcommit_memory == OVERCOMMIT_GUESS) {
        // Allow if: requested < free + cached + swap - kernel_reserved
        free = global_zone_page_state(NR_FREE_PAGES);
        free += global_node_page_state(NR_FILE_PAGES);
        free += get_nr_swap_pages();
        if (pages < free)
            return 0;
        // ... more heuristics
    }
    
    // Mode 2: strict accounting
    allowed = vm_commit_limit();  // overcommit_ratio × RAM + swap
    if (pages + total_committed > allowed)
        return -ENOMEM;  // Fail the allocation
    
    return 0;
}
```

---

## 11. Protecting Processes with `mlock()`

`mlock()` pins pages in RAM and prevents OOM from swapping them:

```c
#include <sys/mman.h>

// Lock all current and future pages:
mlockall(MCL_CURRENT | MCL_FUTURE);
// MCL_CURRENT: lock all pages currently mapped
// MCL_FUTURE: lock all pages mapped in the future

// Lock a specific region:
mlock(addr, size);

// Unlock:
munlock(addr, size);
munlockall();

// Requires: CAP_IPC_LOCK or RLIMIT_MEMLOCK (user-space limit)

// Check how much is locked:
cat /proc/$$/status | grep VmLck
# VmLck:      0 kB  (0 means nothing locked)
```

---

## 12. Kernel `panic_on_oom`

For systems where killing any process is unacceptable:

```bash
# Panic instead of OOM-killing:
cat /proc/sys/vm/panic_on_oom
# 0 = No panic (default) — kill a process
# 1 = Panic if OOM (then reboot if configured)
# 2 = Force panic regardless of cpuset/mempolicy restrictions

echo 1 > /proc/sys/vm/panic_on_oom

# Combine with panic timeout for auto-reboot:
echo 10 > /proc/sys/kernel/panic  # Reboot 10 seconds after panic
```

---

## 13. Observing OOM Events

```bash
# Real-time OOM monitoring:
sudo dmesg -w | grep -i "out of memory\|oom_kill\|killed process"

# bpftrace: trace OOM events:
sudo bpftrace -e '
kprobe:out_of_memory {
    printf("OOM triggered by: %s (pid=%d)\n", comm, pid);
}
kprobe:oom_kill_process {
    printf("OOM kill: victim=%s\n",
        ((struct oom_control*)arg0)->chosen->comm);
}'

# Count OOM kills since boot:
cat /proc/vmstat | grep oom
# oom_kill   3  ← 3 OOM kills since boot

# Per-cgroup OOM events:
cat /sys/fs/cgroup/*/memory.events | grep oom_kill
```

---

## 14. Mental Model Checkpoint

After Day 31, you should be able to:

1. List the reclaim steps that occur *before* the OOM killer fires.
2. Explain `oom_badness()`: what does it score and what formula does it use?
3. What is `oom_score_adj` and what does -1000 mean?
4. Why does the OOM reaper exist? What does it do differently from SIGKILL?
5. How does cgroup memory OOM differ from global OOM?
6. What is `overcommit_memory=2` and how does it prevent OOM?
7. What does `mlockall(MCL_CURRENT | MCL_FUTURE)` do and when would you use it?
8. How do you protect a critical daemon from being OOM-killed?

---

## Key Source Files

```bash
mm/oom_kill.c              # out_of_memory(), oom_badness(), oom_kill_process()
mm/page_alloc.c            # __alloc_pages_may_oom() — OOM trigger
mm/vmscan.c                # Reclaim before OOM
include/linux/oom.h        # OOM constants, oom_control struct
kernel/cgroup/memory.c     # Per-cgroup OOM
mm/util.c                  # __vm_enough_memory() — overcommit check
Documentation/admin-guide/mm/concepts.rst  # OOM documentation
```

---

## Summary

The OOM killer is the kernel's circuit breaker for memory exhaustion. Its algorithm is intentionally simple: score each process by RSS + swap usage, adjusted by `oom_score_adj`, and kill the highest scorer.

Key controls:
- **`/proc/<pid>/oom_score_adj`**: -1000 (exempt) to +1000 (kill first)
- **`/proc/sys/vm/panic_on_oom`**: panic instead of killing (for embedded/critical systems)
- **`/proc/sys/vm/overcommit_memory=2`**: strict accounting to *prevent* reaching OOM
- **cgroup `memory.max`**: contain OOM kills within a resource group
- **`mlockall()`**: pin critical process memory against reclaim

The OOM reaper thread (not the killed process itself) rapidly frees memory by unmapping anonymous pages immediately, ensuring the system recovers quickly even if the victim process is temporarily stuck.

Tomorrow: huge pages and THP — mapping 2MB or 1GB pages to reduce TLB pressure.
