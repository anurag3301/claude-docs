# Day 11 — Kernel Locking: Basics

> **Estimated read time:** 90–120 minutes  
> **Goal:** Master spinlocks, mutexes, semaphores, reader-writer locks — when to use each and how they work internally.

---

## 1. Why Locking Exists

The kernel runs on multiple CPUs simultaneously and is preemptible (with `CONFIG_PREEMPT`). Without synchronization, two CPUs modifying the same data structure simultaneously cause corruption:

```c
// UNSAFE: two CPUs increment a counter simultaneously
static int counter = 0;

void increment(void) {
    counter++;   // This is NOT atomic!
    // Compiles to:
    // 1. LOAD: r1 = counter  (CPU A reads 5)
    // 2. ADD:  r1 = r1 + 1   (CPU A computes 6)
    //                         (CPU B reads 5, computes 6 simultaneously)
    // 3. STORE: counter = r1  (CPU A writes 6)
    //                         (CPU B writes 6)
    // Result: counter = 6, but should be 7!
}
```

This is a **data race** — undefined behavior in C, and in practice produces wrong results.

### 1.1 Sources of Concurrency in the Kernel

1. **SMP**: Multiple CPUs running kernel code simultaneously
2. **Preemption**: With `CONFIG_PREEMPT`, a higher-priority task can preempt kernel code mid-operation
3. **Interrupts**: A hardware interrupt can fire and interrupt kernel code at any point
4. **Softirqs**: Can preempt process context on the same CPU

Each of these requires different defensive measures.

---

## 2. The Synchronization Toolkit at a Glance

```
Scenario                               Solution
──────────────────────────────────────────────────────────────────
Fast path, interrupt context           spinlock (with irqsave)
Fast path, process context only        spinlock (plain)
Long/slow critical section             mutex
Allowing concurrent reads              rwlock or rwsem
Protecting per-CPU data               per-CPU variables (no lock!)
High-read, low-write patterns          RCU (Day 12)
Simple counter (no struct)            atomic_t / atomic64_t
Wait for an event                     wait_queue or completion
Sequence of related updates (reads)   seqlock
```

---

## 3. Spinlocks

A spinlock is a **busy-wait** lock: if the lock is taken, the waiter *spins* (loops checking the lock) until it's released. This means:

- **No sleeping**: the CPU is 100% busy while waiting
- **Minimum hold time**: ideal for very short critical sections (single digit microseconds)
- **Required in interrupt context**: because interrupts can't sleep

### 3.1 Declaration and Initialization

```c
#include <linux/spinlock.h>

// Static initialization:
DEFINE_SPINLOCK(my_lock);              // Declares + initializes
static DEFINE_SPINLOCK(priv_lock);     // Module-private

// Dynamic initialization (for locks embedded in structs):
spinlock_t my_lock;
spin_lock_init(&my_lock);

// In a struct:
struct my_device {
    spinlock_t lock;
    u32 state;
    // ...
};

// In init:
spin_lock_init(&dev->lock);
```

### 3.2 Basic Lock/Unlock

```c
spin_lock(&lock);
    // CRITICAL SECTION — protected against:
    // - Other CPUs taking the same lock
    // - Preemption disabled implicitly
critical_code();
spin_unlock(&lock);
```

`spin_lock()` disables kernel preemption on the local CPU (`preempt_disable()`). This prevents the thread from being preempted while holding a spinlock.

### 3.3 Spinlock with IRQ Disable

**Problem**: if process context takes a spinlock, and an interrupt fires that tries to take the same spinlock → **deadlock** (interrupt waits for spinlock, but process can't release it because it's interrupted).

```c
// WRONG — interrupt can deadlock:
spin_lock(&dev->lock);
// ← Interrupt fires here, tries spin_lock(&dev->lock) → DEADLOCK
modify_data(dev);
spin_unlock(&dev->lock);

// CORRECT — disable IRQs while lock is held:
unsigned long flags;
spin_lock_irqsave(&dev->lock, flags);   // Disables IRQs, saves RFLAGS
modify_data(dev);
spin_unlock_irqrestore(&dev->lock, flags);  // Restores IRQs (re-enable if they were on)
```

**Rule**: if a spinlock is ever taken in interrupt context (hardirq), ALL acquisitions of that lock must use `spin_lock_irqsave()` / `spin_unlock_irqrestore()`.

```c
// Variants:
spin_lock(&lock);              // Basic — disables preemption only
spin_lock_irq(&lock);          // Disables preemption + IRQs (assumes IRQs were on)
spin_lock_irqsave(&lock, flags); // Disables preemption + IRQs, saves old IRQ state
spin_lock_bh(&lock);           // Disables preemption + softirqs (not hardirqs)

spin_unlock(&lock);
spin_unlock_irq(&lock);
spin_unlock_irqrestore(&lock, flags);
spin_unlock_bh(&lock);

// Try without waiting:
if (spin_trylock(&lock)) {
    // Got it!
    spin_unlock(&lock);
} else {
    // Lock is held by someone else — do something else
}
```

### 3.4 When to Use Which Variant

```
Spinlock taken from:     Use:
──────────────────────────────────────────────────────
Process context only   → spin_lock / spin_unlock
Process + softirq      → spin_lock_bh / spin_unlock_bh
Process + hardirq      → spin_lock_irqsave / spin_unlock_irqrestore
Softirq + hardirq      → spin_lock_irqsave / spin_unlock_irqrestore
Hardirq only           → spin_lock / spin_unlock
                          (preemption is already off in hardirq)
```

### 3.5 Spinlock Internal Implementation

On x86_64, a queued spinlock (MCS-like) implementation:

```c
// include/asm-generic/qspinlock.h (simplified)
typedef struct qspinlock {
    union {
        atomic_t val;
        struct {
            u8 locked;    // 0 = free, 1 = locked
            u8 pending;   // Someone is waiting (first waiter)
            u16 tail;     // Queue of waiters
        };
    };
} arch_spinlock_t;

// Acquire (simplified):
void queued_spin_lock(struct qspinlock *lock)
{
    u32 val = 0;
    // Fast path: CAS 0 → _Q_LOCKED_VAL (= 1)
    if (atomic_try_cmpxchg_acquire(&lock->val, &val, _Q_LOCKED_VAL))
        return;  // Got the lock!
    
    // Slow path: queue ourselves and spin
    queued_spin_lock_slowpath(lock, val);
}
```

The queued spinlock (since Linux 4.2) eliminates the "thundering herd" problem where all waiting CPUs spin on the same cache line — instead, each CPU in the queue spins on its own MCS node (local variable).

### 3.6 What NOT to Do with Spinlocks

```c
// FORBIDDEN while holding a spinlock:
kmalloc(size, GFP_KERNEL);  // May sleep during reclaim
mutex_lock(&m);             // May sleep
schedule();                 // May sleep
msleep(10);                 // Sleeps
printk(KERN_INFO "...");    // OK in process context, but can be slow
copy_from_user(buf, uptr, n); // May page fault → sleep

// ALLOWED:
kmalloc(size, GFP_ATOMIC);   // Won't sleep (but may fail)
pr_err("...\n");             // OK (printk doesn't sleep in kernel 5.x+)
atomic_inc(&counter);        // OK
readl(iobase + REG);         // OK
```

---

## 4. Mutexes

A mutex (mutual exclusion lock) is a **sleeping lock** — if the lock is taken, the waiter sleeps (releases the CPU to run other tasks). This means:

- **Only in process context** (never in interrupt context, never in softirq)
- **Longer hold times** are acceptable
- **Contention is fine** — blocked threads don't waste CPU
- **Strict ownership**: only the thread that acquired the mutex can release it

### 4.1 Declaration and Initialization

```c
#include <linux/mutex.h>

// Static:
DEFINE_MUTEX(my_mutex);
static DEFINE_MUTEX(priv_mutex);

// Dynamic:
struct mutex my_mutex;
mutex_init(&my_mutex);

// In struct:
struct my_subsystem {
    struct mutex lock;
    struct list_head items;
};
mutex_init(&sub->lock);
```

### 4.2 Locking

```c
// Acquire (blocks if locked — CANNOT call from interrupt context):
mutex_lock(&my_mutex);

// Acquire, but return -EINTR if a signal arrives:
if (mutex_lock_interruptible(&my_mutex)) {
    return -EINTR;   // User Ctrl+C'd or signal delivered
}

// Acquire, but return -ERESTARTSYS if signal arrives:
if (mutex_lock_killable(&my_mutex)) {
    return -ERESTARTSYS;
}

// Try without waiting (returns 1 if acquired, 0 if not):
if (mutex_trylock(&my_mutex)) {
    // Got it
    mutex_unlock(&my_mutex);
}

// Release:
mutex_unlock(&my_mutex);
```

`mutex_lock_interruptible` is almost always preferred over `mutex_lock` in syscall handlers — it lets the user interrupt a blocked operation with Ctrl+C (SIGINT).

### 4.3 Mutex Internal Implementation

```c
// include/linux/mutex.h
struct mutex {
    atomic_long_t       owner;   // Points to task_struct of owner, or 0
    raw_spinlock_t      wait_lock;
    struct list_head    wait_list;  // Queue of blocked waiters
#ifdef CONFIG_DEBUG_MUTEXES
    const char         *name;
    void               *magic;
#endif
};
```

Fast path (uncontended):
```c
// mutex_lock() fast path:
// CAS: if owner == 0, set owner = current → lock acquired
if (atomic_long_try_cmpxchg_acquire(&lock->owner, &owner, (long)current))
    return;  // Fast path: got the lock!

// Slow path: spin briefly (optimistic spinning), then sleep:
mutex_lock_slowpath(lock);
```

The mutex has an optimistic spinning phase: if the owner is running on another CPU, spin briefly hoping it releases soon. Only sleep if it doesn't release quickly. This reduces context switch overhead for short-held mutexes.

### 4.4 Mutex Rules (Enforced by `lockdep`)

1. **Cannot call from interrupt context** — will trigger `might_sleep()` warning
2. **Only the owner can unlock** — unlike semaphores
3. **No recursive locking** — calling `mutex_lock()` while holding it deadlocks
4. **Cannot be held across `schedule()`** — wait, actually it can, unlike spinlocks

```c
// Checking context:
might_sleep();   // Called inside mutex.c — BUG if called in interrupt context

// Debug: check if current task holds this mutex:
lockdep_assert_held(&my_mutex);  // Assert (with lockdep enabled)
```

---

## 5. Semaphores

Semaphores are counting locks: a counter initialized to N allows N concurrent holders. They're more general than mutexes (which are binary semaphores with ownership).

**In modern kernel code, semaphores are rarely used** — they were replaced by mutexes (for N=1) and other primitives. But you'll see them in older code.

```c
#include <linux/semaphore.h>

// Declare with initial count:
struct semaphore my_sem;
sema_init(&my_sem, 1);    // Binary (like mutex, but no ownership)
sema_init(&my_sem, 5);    // Allow up to 5 concurrent accessors

// Static init:
DEFINE_SEMAPHORE(my_sem);  // Count = 1

// Acquire (decrement; blocks if count == 0):
down(&my_sem);                       // Uninterruptible
int ret = down_interruptible(&my_sem);  // Returns -EINTR on signal
int ret = down_trylock(&my_sem);        // Returns 0 if acquired, 1 if not

// Release (increment; wakes waiter if any):
up(&my_sem);  // CAN be called from a different task than the down()
```

**Use semaphore when**:
- You need a counting semaphore (N > 1 concurrent holders)
- You need a semaphore that's released from a different task (producer/consumer without completion)

**Use mutex instead** when you just need mutual exclusion (N=1).

---

## 6. Reader-Writer Locks

Many data structures are read-heavy: many readers, infrequent writers. A plain spinlock/mutex serializes readers unnecessarily.

### 6.1 `rwlock_t` — Reader-Writer Spinlock

```c
#include <linux/rwlock.h>

// Declaration:
DEFINE_RWLOCK(my_rwlock);
rwlock_t my_rwlock;
rwlock_init(&my_rwlock);

// Read (multiple concurrent readers allowed):
read_lock(&my_rwlock);
// ... read data ...
read_unlock(&my_rwlock);

// Write (exclusive):
write_lock(&my_rwlock);
// ... modify data ...
write_unlock(&my_rwlock);

// With IRQ save:
read_lock_irqsave(&my_rwlock, flags);
read_unlock_irqrestore(&my_rwlock, flags);
write_lock_irqsave(&my_rwlock, flags);
write_unlock_irqrestore(&my_rwlock, flags);
```

**Limitation**: readers can starve writers! If readers keep arriving, the writer may never get the lock. The kernel's RCU mechanism (Day 12) solves this far more elegantly.

### 6.2 `rwsem` — Reader-Writer Semaphore (Sleeping)

```c
#include <linux/rwsem.h>

// Declaration:
DECLARE_RWSEM(my_rwsem);
struct rw_semaphore my_rwsem;
init_rwsem(&my_rwsem);

// Read:
down_read(&my_rwsem);         // Blocking
down_read_trylock(&my_rwsem); // Non-blocking
up_read(&my_rwsem);

// Write:
down_write(&my_rwsem);
down_write_trylock(&my_rwsem);
up_write(&my_rwsem);

// Downgrade (write → read, without releasing):
downgrade_write(&my_rwsem);  // Reader takes over; writers still blocked

// Trylock variants:
down_read_trylock(&my_rwsem);   // 1 if acquired, 0 if not
down_write_trylock(&my_rwsem);
```

`rwsem` is used extensively in the kernel:
- `mm_struct::mmap_lock` — protects the VMA tree (heavy read, occasional write)
- `task_struct::signal->shared_pending` — signal queues
- Many filesystem locks

```c
// VMA tree: an rwsem protects reads (page faults) vs writes (mmap/munmap):
mmap_read_lock(mm);
vma = find_vma(mm, address);  // Many concurrent readers OK
mmap_read_unlock(mm);

mmap_write_lock(mm);
do_mmap(mm, ...);  // Exclusive write
mmap_write_unlock(mm);
```

---

## 7. Atomic Operations

For simple counters and flags, avoid locks entirely:

```c
#include <linux/atomic.h>

// 32-bit atomic integer:
atomic_t counter;
atomic_set(&counter, 0);   // Initialize

atomic_read(&counter)           // Read
atomic_inc(&counter)            // counter++
atomic_dec(&counter)            // counter--
atomic_add(n, &counter)         // counter += n
atomic_sub(n, &counter)         // counter -= n

// Read-after-modify:
int old = atomic_fetch_inc(&counter);     // Returns old value, then increments
int new = atomic_inc_return(&counter);    // Increments, then returns new value
int old = atomic_xchg(&counter, new_val); // Swap atomically

// Compare-and-swap (CAS):
int expected = 5;
bool success = atomic_cmpxchg(&counter, expected, new_val) == expected;
// Sets counter = new_val only if counter == expected; returns old value

// Decrement and test (used for refcounting):
if (atomic_dec_and_test(&counter)) {
    // counter hit zero — do cleanup
}

// 64-bit:
atomic64_t big_counter;
atomic64_inc(&big_counter);
atomic64_read(&big_counter);

// Bitwise atomics:
unsigned long flags_word;
set_bit(BIT_NR, &flags_word);
clear_bit(BIT_NR, &flags_word);
test_bit(BIT_NR, &flags_word);
test_and_set_bit(BIT_NR, &flags_word);   // Returns old value
```

`atomic_t` is the right choice for: refcounts, statistics counters (that don't need lock-free read), simple flags.

```bash
# Check if atomic operations are lock-free on this arch:
cat /boot/config-$(uname -r) | grep CONFIG_ARCH_HAS_ATOMIC64
```

---

## 8. Lockdep — Deadlock Detection

`lockdep` is the kernel's lock validator. Enable it with `CONFIG_PROVE_LOCKING=y`.

```bash
# Enable in kernel config:
scripts/config --enable PROVE_LOCKING
scripts/config --enable DEBUG_LOCKDEP
scripts/config --enable LOCK_STAT
make -j$(nproc)
```

Lockdep tracks:
1. **Lock ordering**: builds a directed graph of "lock A → lock B" (A taken before B). If a cycle is detected, warns about potential deadlock.
2. **Context**: warns if a sleeping lock is taken in interrupt context.
3. **Lock classes**: each unique lock type (based on its location in source code) is a "class". Two locks initialized at the same site are the same class.

```bash
# Lockdep warning example (in dmesg):
# WARNING: possible circular locking dependency detected
# 
# bash/1234 is trying to acquire lock:
# ffffc90000123456 (&dev->lock){....}-{2:2}, at: my_irq_handler+0x45/0x90
#                  ^^^^^^^^^^^^^^^^^^^^
#                  Lock type: spinlock (depth 2)
# 
# but task is already holding lock:
# ffffc90000234567 (&cpu_lock){....}-{2:2}, at: my_func+0x12/0x30
# 
# which lock already depends on the new lock.
# 
#  the existing dependency chain (in reverse order) is:
# ...

# After seeing this: look at line numbers, understand the lock ordering,
# either fix by always taking locks in the same order, or restructure code.
```

### 8.1 Lock Annotation

```c
// Tell lockdep about intentional "same-class" locks:
// (e.g., nested locking of the same mutex type in different objects)

// Lock nest:
mutex_lock_nested(&child->lock, I_MUTEX_CHILD);
// The second arg is a subclass — tells lockdep this is always taken "after" parent

// Marking a lock as IRQ-safe:
lockdep_set_class(&my_lock, &my_lock_class);

// Temporarily disable lockdep for a known-safe pattern:
lockdep_off();
// ... known safe code ...
lockdep_on();
```

### 8.2 Checking Lock State

```c
// Assert a lock is held (compile-time checks with lockdep):
lockdep_assert_held(&my_mutex);
lockdep_assert_held_read(&my_rwsem);
lockdep_assert_held_write(&my_rwsem);

// These compile to no-ops without CONFIG_PROVE_LOCKING.
// With lockdep: will WARN if the assertion fails.
```

---

## 9. Lock Ordering and Deadlock Avoidance

Deadlocks happen when lock acquisition cycles exist:

```
CPU 0:               CPU 1:
lock(A)              lock(B)
...waiting...        ...waiting...
lock(B) ← BLOCKED   lock(A) ← BLOCKED
```

**The golden rule**: always acquire locks in a consistent global order.

```c
// WRONG — different order in different code paths:
// Path 1:
mutex_lock(&lock_a);
mutex_lock(&lock_b);   // Lock order: A → B

// Path 2 (elsewhere):
mutex_lock(&lock_b);
mutex_lock(&lock_a);   // Lock order: B → A — POTENTIAL DEADLOCK

// CORRECT — always A before B:
// Document it! e.g.:
/* Lock ordering: always lock_a before lock_b */
mutex_lock(&lock_a);
mutex_lock(&lock_b);
```

### 9.1 Lock Hierarchy in the Kernel

The kernel has documented lock orderings. For example, in the network stack:

```
socket lock
  → net_device lock
    → NIC hardware lock (spinlock)
```

In the memory subsystem:
```
mmap_lock (rwsem)
  → page table lock (spinlock)
    → page lock (per-page spinlock)
```

**Never** lock in reverse order of the established hierarchy.

### 9.2 Try-Lock to Avoid Deadlock

In some cases, use `trylock` to avoid blocking when already holding another lock:

```c
// When the lock order is unknown or variable (e.g., operating on two objects
// of the same type):
void transfer(struct account *from, struct account *to, int amount)
{
retry:
    mutex_lock(&from->lock);
    
    if (!mutex_trylock(&to->lock)) {
        // Failed — release and retry in different order
        mutex_unlock(&from->lock);
        
        // Small backoff to reduce livelock:
        cond_resched();
        goto retry;
    }
    
    from->balance -= amount;
    to->balance += amount;
    
    mutex_unlock(&to->lock);
    mutex_unlock(&from->lock);
}
```

Or sort the locks by address (ensures consistent order regardless of call order):

```c
void transfer(struct account *a, struct account *b, int amount)
{
    // Sort by pointer value to ensure consistent order:
    struct account *first  = a < b ? a : b;
    struct account *second = a < b ? b : a;
    
    mutex_lock(&first->lock);
    mutex_lock(&second->lock);
    
    if (a < b) { a->balance -= amount; b->balance += amount; }
    else       { b->balance += amount; a->balance -= amount; }
    
    mutex_unlock(&second->lock);
    mutex_unlock(&first->lock);
}
```

---

## 10. Condition Variables: `wait_event` Revisited

In classical programming, condition variables + mutex are paired. In the kernel, `wait_queue` + lock serves the same purpose:

```c
// Producer (from IRQ handler or another thread):
spin_lock(&queue->lock);
enqueue(queue, item);
spin_unlock(&queue->lock);
wake_up(&queue->wait);  // Signal condition changed

// Consumer:
wait_event_interruptible(queue->wait,
    !queue_empty(queue));  // Re-checked after each wake_up

spin_lock(&queue->lock);
item = dequeue(queue);
spin_unlock(&queue->lock);
```

The `wait_event` macro correctly handles the race between checking the condition and sleeping:

```c
// wait_event(wq, condition) expands to (simplified):
for (;;) {
    prepare_to_wait(&wq, &entry, TASK_INTERRUPTIBLE);
    if (condition)
        break;
    schedule();  // Sleep
}
finish_wait(&wq, &entry);
```

The `prepare_to_wait` adds the task to the wait queue *before* checking the condition, so a `wake_up` that arrives during this window doesn't get lost.

---

## 11. `cond_resched` — Yielding in Long Loops

Long loops in kernel code should yield the CPU to allow higher-priority tasks to run:

```c
// In a long loop processing data (process context, preemption enabled):
for (i = 0; i < huge_count; i++) {
    process_item(items[i]);
    
    cond_resched();  // Yield if a higher-priority task is waiting
    // This is a no-op if no task needs the CPU.
}
```

`cond_resched()` calls `schedule()` if `need_resched()` is set. Without it, a kernel loop can starve other processes for hundreds of milliseconds.

```c
// With lock held — can't resched (scheduler might switch tasks):
// spin_lock_bh(&lock);
// for (large loop) {
//    cond_resched();  ← WRONG: you hold a lock!
// }
// spin_unlock_bh(&lock);

// Instead: drop the lock, resched, re-acquire:
for (i = 0; i < huge_count; i++) {
    spin_lock_bh(&lock);
    process_item(items[i]);
    spin_unlock_bh(&lock);
    cond_resched();  // Now safe: no lock held
}
```

---

## 12. Per-CPU Counters: Avoiding Locks Entirely

For statistics that are only summed occasionally, per-CPU variables eliminate lock contention:

```c
#include <linux/percpu.h>

// Per-CPU counter:
DEFINE_PER_CPU(unsigned long, packet_count);

// Increment (fast path, no lock needed):
this_cpu_inc(packet_count);

// Read total (slower, sums all CPUs):
unsigned long total = 0;
int cpu;
for_each_possible_cpu(cpu)
    total += per_cpu(packet_count, cpu);
```

This is how the kernel's network statistics work — `netdev->stats.rx_packets` is per-CPU in high-performance paths.

---

## 13. `local_lock` — Ensuring CPU-Local Invariants

`local_lock` is a newer abstraction (kernel 5.8+) that protects code that must run on a single CPU without preemption:

```c
#include <linux/local_lock.h>

DEFINE_LOCAL_LOCK(my_local_lock);

// Acquire (disables preemption, protects CPU-local data):
local_lock(&my_local_lock);
// ... CPU-local critical section ...
local_unlock(&my_local_lock);

// With IRQ save:
local_lock_irqsave(&my_local_lock, flags);
local_unlock_irqrestore(&my_local_lock, flags);
```

`local_lock` compiles to `preempt_disable()` on non-PREEMPT_RT kernels and to a per-CPU spinlock on PREEMPT_RT (where even preemption is driven by regular locks).

---

## 14. Lock Statistics

```bash
# Enable lock statistics at runtime:
echo 1 > /proc/sys/kernel/lock_stat

# View stats:
cat /proc/lock_stat
# class name        con-bounces   contentions   waittime-min   waittime-max   waittime-total   acquisitions   holdtime-min   holdtime-max   holdtime-total
# &mm->mmap_lock:R    1234          5678          0.00ns        123.45us       456789.00ns      1000000        0.00ns         50.00us        5000000.00ns
# &dev->lock          89            234            0.00ns        45.67us        89012.00ns       500000         0.00ns         20.00us        2000000.00ns

# Fields:
# con-bounces: times lock changed CPU while contested
# contentions: number of times lock was taken when already held
# waittime: time spent waiting for the lock
# acquisitions: total number of successful lock acquisitions
# holdtime: time the lock was held

# Reset stats:
echo 0 > /proc/sys/kernel/lock_stat
```

High contention on a lock = bottleneck. Solutions:
1. Reduce lock hold time (do less work while holding it)
2. Use finer-grained locking (one lock per object instead of one global lock)
3. Use a read-write lock if most accesses are reads
4. Use RCU for read-heavy patterns (Day 12)
5. Use per-CPU data where possible

---

## 15. Choosing the Right Lock

```
Question 1: What contexts access this data?
├─ Only process context (syscalls, workqueues)
│   └─ mutex (preferred) or rwsem
├─ Process context + softirq
│   └─ spin_lock_bh (prevents softirq on local CPU)
└─ Process context + hardirq (or softirq + hardirq)
    └─ spin_lock_irqsave

Question 2: What's the access pattern?
├─ Mostly reads, rare writes
│   ├─ Many readers simultaneously OK? → RCU (Day 12)
│   └─ Simpler, less performance-critical → rwsem
└─ Mixed reads/writes
    └─ Plain spinlock or mutex

Question 3: What's the critical section duration?
├─ Microseconds (register I/O, list manipulation)
│   └─ Spinlock
└─ Longer (memory allocation, blocking I/O, sleep)
    └─ Mutex (or threaded IRQ / workqueue)

Question 4: Is this a counter or simple flag?
└─ atomic_t — no lock needed

Question 5: Is this per-CPU data?
└─ per-CPU variable — no lock needed (just disable preemption)
```

---

## 16. Common Mistakes

### 16.1 Forgetting `_irqsave` When Needed

```c
// Bug: spin_lock not protecting against interrupt
static DEFINE_SPINLOCK(my_lock);

// This runs in process context:
void process_context_func(void)
{
    spin_lock(&my_lock);      // ← BUG if my_irq_handler also uses my_lock
    // ...
    spin_unlock(&my_lock);
}

// This runs in hardirq:
irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    spin_lock(&my_lock);      // DEADLOCK if process_context_func is interrupted here!
    // ...
    spin_unlock(&my_lock);
    return IRQ_HANDLED;
}

// Fix:
void process_context_func(void)
{
    unsigned long flags;
    spin_lock_irqsave(&my_lock, flags);
    // ...
    spin_unlock_irqrestore(&my_lock, flags);
}
```

### 16.2 Recursive Lock (Spinlock)

```c
spin_lock(&my_lock);
do_something();           // If do_something() tries to take my_lock → DEADLOCK
spin_unlock(&my_lock);

// Fix: either:
// 1. Restructure to not call do_something while holding lock
// 2. Use a different lock for the inner path
// 3. Use an already-locked version of do_something
```

### 16.3 Lock Held Across Sleep

```c
spin_lock(&my_lock);
msleep(100);     // BUG: sleeping while holding spinlock!
               // Other CPUs waiting for this lock are wasting CPU for 100ms
spin_unlock(&my_lock);

// If you need to sleep in the critical section, use a mutex:
mutex_lock(&my_mutex);
msleep(100);    // OK: mutex allows sleeping
mutex_unlock(&my_mutex);
```

---

## 17. Mental Model Checkpoint

After Day 11, you should be able to:

1. Explain why a spinlock busy-waits and when that's the right choice.
2. Use `spin_lock_irqsave` correctly — when is it required vs just `spin_lock`?
3. Explain mutex vs spinlock: when to choose each, what happens on contention.
4. Use `mutex_lock_interruptible` and handle -EINTR correctly.
5. Implement a reader-writer lock with `rwsem` for a read-heavy structure.
6. Use `atomic_t` for a simple counter and explain when it's sufficient.
7. Identify a deadlock from lock ordering analysis.
8. Interpret `lockdep` output in dmesg.
9. Choose between `spin_lock_bh` and `spin_lock_irqsave` for a given access pattern.
10. Use `local_lock` for CPU-local data.

---

## Key Source Files

```bash
include/linux/spinlock.h          # Spinlock API and macros
include/linux/mutex.h             # Mutex API
include/linux/semaphore.h         # Semaphore API
include/linux/rwlock.h            # Reader-writer spinlock
include/linux/rwsem.h             # Reader-writer semaphore
include/linux/atomic.h            # Atomic operations
include/linux/lockdep.h           # Lockdep annotation macros
kernel/locking/spinlock.c         # Spinlock slow path
kernel/locking/mutex.c            # Mutex implementation
kernel/locking/qspinlock.c        # Queued spinlock implementation
Documentation/locking/            # Official kernel locking documentation
Documentation/locking/mutex-design.rst  # Mutex design rationale
```

---

## Summary

Kernel synchronization uses different primitives for different contexts:

- **Spinlocks** (`spinlock_t`): busy-wait, interrupt-safe variants, never sleep. For short critical sections in any context. Use `_irqsave` when shared with interrupt handlers.

- **Mutexes** (`struct mutex`): sleeping lock, process context only, optimistic spinning. For longer critical sections or code that may sleep.

- **Semaphores** (`struct semaphore`): counting sleeping lock. Rarely used in modern kernel; use mutex for binary case.

- **Reader-writer variants** (`rwlock_t`, `rw_semaphore`): allow concurrent readers, exclusive writers. Use when reads dominate. For extreme read-heavy workloads, prefer RCU (Day 12).

- **Atomic operations** (`atomic_t`): lock-free, for simple values. Eliminates lock overhead for counters and flags.

The cardinal rules: no sleeping while holding a spinlock, always use `_irqsave` variants when shared with interrupt context, document and enforce lock ordering to prevent deadlocks.

Tomorrow: advanced locking — RCU, seqlocks, memory barriers, and the full picture of concurrent kernel programming.
