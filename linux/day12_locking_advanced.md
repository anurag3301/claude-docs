# Day 12 — Kernel Locking: Advanced

> **Estimated read time:** 90–120 minutes  
> **Goal:** Master RCU (Read-Copy-Update), seqlocks, memory barriers, atomic operations, and the full mental model for lock-free kernel programming.

---

## 1. The Limits of Traditional Locking

Day 11 covered spinlocks and mutexes — they work, but have costs:

- **Scalability ceiling**: every reader of a structure must acquire a lock, even if nobody is writing. On a 128-CPU server, serializing 128 readers on a spinlock is catastrophic.
- **Cache line bouncing**: the lock cache line bounces between CPUs even when all are just reading.
- **Starvation**: writers can starve under reader pressure (rwlock) or readers can starve under write pressure.

The kernel's answer to read-heavy data is **RCU** (Read-Copy-Update) — a synchronization mechanism where readers need no locks at all.

---

## 2. RCU — Read-Copy-Update

RCU is the most important concurrency technique unique to the Linux kernel. It appears in almost every major subsystem: networking (routing tables, netfilter), VFS (dentry cache), process management (task_struct lookup), networking filters, and more.

### 2.1 The Core Idea

RCU achieves **lock-free reads** by making updates follow three steps:

1. **Read** a copy of the data you're modifying
2. **Copy** it and modify the copy
3. **Update** the pointer to the new copy atomically

Old readers continue reading the *old* version. The old version is freed only after all readers that started before the update are done (**grace period**).

```
Before update:      
  [pointer] → [old data]
  
  Reader A starts: sees [old data]         ← Still valid
  
During update:
  [pointer] → [new data]    ← Atomic pointer swap
              [old data]    ← Still alive! Reader A might be using it
  
  Reader B starts: sees [new data]
  
After grace period (Reader A finishes):
  [pointer] → [new data]
  free([old data])          ← Now safe to free
```

### 2.2 The Three Guarantees

1. **Readers never block** — no locks, no atomic operations in the read path (just a barrier)
2. **Readers never starve writers** — after all current readers finish, the old version is freed
3. **Readers see a consistent snapshot** — they either see the old OR new version, never a half-updated one (pointer swap is atomic)

### 2.3 When to Use RCU

RCU is ideal when:
- Reads vastly outnumber writes (routing tables: 100K reads/sec, 1 write/sec)
- Readers are frequent but short (no sleeping inside RCU read-side critical section)
- Memory for old versions is acceptable (you keep old copies during the grace period)

**Not** suitable for:
- Write-heavy workloads
- Very large data where keeping old copies is expensive
- Code where readers need to sleep (use rwsem instead)

---

## 3. RCU API — Readers

```c
#include <linux/rcupdate.h>

// The read-side critical section:
rcu_read_lock();
    // INSIDE: you may access RCU-protected data
    // - Can run on any CPU
    // - Cannot sleep (no schedule(), no mutex_lock, no GFP_KERNEL alloc)
    // - Can be preempted (with CONFIG_PREEMPT_RCU)
    // - Marks a "quiescent state" boundary for the grace period
rcu_read_unlock();

// Accessing an RCU-protected pointer:
struct my_data *data;
rcu_read_lock();
data = rcu_dereference(my_global_ptr);  // Atomic read + memory barrier
if (data) {
    use_data(data);   // data is guaranteed valid until rcu_read_unlock
}
rcu_read_unlock();

// Why rcu_dereference() and not just *my_global_ptr?
// It includes a data dependency barrier on DEC Alpha (where memory ordering
// is very weak) and documents intent for tools like sparse/lockdep.
// On x86 it compiles to a plain load, but never skip it — it's documentation.
```

### 3.1 Nested RCU Read Locks

```c
// RCU read locks are nestable and cheap (often just a compiler barrier):
rcu_read_lock();
outer_data = rcu_dereference(outer_ptr);
    rcu_read_lock();    // Nested — OK
    inner_data = rcu_dereference(inner_ptr);
    rcu_read_unlock();
rcu_read_unlock();
```

---

## 4. RCU API — Writers/Updaters

Writers do the real work: copying, modifying, atomically swapping, then waiting for old readers.

```c
// Updating an RCU-protected pointer:
struct my_data *old_data;
struct my_data *new_data;

// Step 1: Create new version
new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
*new_data = *old_data;      // Copy old data
new_data->value = new_value; // Modify copy

// Step 2: Atomic pointer update (readers now see new_data)
old_data = rcu_dereference_protected(my_global_ptr, lockdep_is_held(&my_lock));
rcu_assign_pointer(my_global_ptr, new_data);
// rcu_assign_pointer() includes a write memory barrier (smp_wmb())
// to ensure new_data's contents are visible before the pointer update

// Step 3: Wait for all pre-update readers to finish
synchronize_rcu();   // BLOCKING: sleeps until grace period ends
// After this returns: no reader can hold a reference to old_data

// Step 4: Free the old version
kfree(old_data);
```

### 4.1 `synchronize_rcu` vs `call_rcu`

`synchronize_rcu()` blocks until the grace period ends. In some contexts (interrupt, atomic), you can't block. Use `call_rcu()` to register a callback:

```c
// Deferred freeing via callback:
struct my_data {
    // ... your data fields ...
    struct rcu_head rcu;  // Embedded for call_rcu
};

// The callback (runs after grace period, in softirq or kthread context):
static void my_data_free_rcu(struct rcu_head *head)
{
    struct my_data *data = container_of(head, struct my_data, rcu);
    kfree(data);
}

// After rcu_assign_pointer:
call_rcu(&old_data->rcu, my_data_free_rcu);
// Returns immediately; callback is invoked after grace period
// Do NOT touch old_data after this call!
```

### 4.2 `kfree_rcu` — Shortcut for Simple Freeing

When the callback just calls `kfree`, use the convenience macro:

```c
// Instead of call_rcu + callback:
kfree_rcu(old_data, rcu);  // old_data->rcu must be struct rcu_head
// Equivalent to: call_rcu(&old_data->rcu, kfree_callback)

// Even simpler in recent kernels (no embedded rcu_head needed):
kfree_rcu(ptr, rcu_field);
// or:
kfree_rcu(ptr);  // If struct has no dependencies
```

---

## 5. RCU in Practice: Replacing a Table Entry

Complete example — a routing table lookup that's called millions of times per second:

```c
// Protected structure:
struct route_entry {
    u32 dest;
    u32 gateway;
    struct rcu_head rcu;
};

// Global pointer (RCU-protected):
struct route_entry __rcu *current_route = NULL;
DEFINE_MUTEX(route_update_lock);  // Serializes writers (not readers!)

// READER PATH (called from packet processing — must be fast):
int lookup_route(u32 dest, u32 *gateway)
{
    struct route_entry *entry;
    
    rcu_read_lock();
    entry = rcu_dereference(current_route);
    if (entry && entry->dest == dest) {
        *gateway = entry->gateway;
        rcu_read_unlock();
        return 0;  // Found
    }
    rcu_read_unlock();
    return -ENOENT;
}

// WRITER PATH (called infrequently when routing changes):
int update_route(u32 dest, u32 gateway)
{
    struct route_entry *new_entry, *old_entry;
    
    new_entry = kmalloc(sizeof(*new_entry), GFP_KERNEL);
    if (!new_entry) return -ENOMEM;
    
    new_entry->dest = dest;
    new_entry->gateway = gateway;
    
    // Serialize writes:
    mutex_lock(&route_update_lock);
    old_entry = rcu_dereference_protected(current_route,
                    lockdep_is_held(&route_update_lock));
    rcu_assign_pointer(current_route, new_entry);
    mutex_unlock(&route_update_lock);
    
    // Wait for readers that saw old_entry to finish:
    if (old_entry) {
        synchronize_rcu();
        kfree(old_entry);
    }
    return 0;
}
```

---

## 6. RCU Lists — The Most Common Pattern

Most RCU usage involves linked lists. The kernel provides RCU-safe list operations:

```c
// Initialization:
LIST_HEAD(my_rcu_list);  // Same as regular list

// RCU list operations for WRITERS:
list_add_rcu(&new->list, &my_rcu_list);      // Add at head
list_add_tail_rcu(&new->list, &my_rcu_list); // Add at tail
list_del_rcu(&entry->list);    // Delete (entry still accessible by old readers)
list_replace_rcu(&old->list, &new->list);    // Replace old with new

// After list_del_rcu, wait before freeing:
synchronize_rcu();
kfree(entry);  // Now safe
// OR:
call_rcu(&entry->rcu, my_free_func);

// RCU list iteration for READERS:
rcu_read_lock();
list_for_each_entry_rcu(item, &my_rcu_list, list) {
    // item is safe to access here
    // Do NOT sleep inside this loop
}
rcu_read_unlock();

// Can also iterate outside rcu_read_lock (with lock held):
mutex_lock(&my_mutex);  // Writer holds this lock
list_for_each_entry(item, &my_rcu_list, list) {  // NOT _rcu version needed
    // Can modify items here
}
mutex_unlock(&my_mutex);
```

### 6.1 Kernel Example: Network Filter Rules

```c
// net/netfilter/core.c — registering a hook
// The hooks list is RCU-protected so packet processing never blocks
struct nf_hook_entry {
    nf_hookfn *hook;
    void *priv;
    struct nf_hook_entry __rcu *next;
};

// Packet processing (called millions of times per second):
int nf_hook(u_int8_t pf, unsigned int hook, struct net *net, ...)
{
    struct nf_hook_entry *entry;
    
    rcu_read_lock();
    entry = rcu_dereference(net->nf.hooks[pf][hook]);
    while (entry) {
        ret = entry->hook(entry->priv, skb, state);
        if (ret != NF_ACCEPT) break;
        entry = rcu_dereference(entry->next);
    }
    rcu_read_unlock();
    return ret;
}
```

---

## 7. Grace Periods and Quiescent States

The "grace period" ends when every CPU has passed through at least one **quiescent state** — a point where no RCU read-side critical section is active:

- On non-preemptible kernels: any context switch is a quiescent state (a task can't be in `rcu_read_lock()` during a context switch, since it can't sleep there)
- On preemptible kernels: `rcu_read_unlock()` is the explicit quiescent state marker

```bash
# RCU statistics:
cat /sys/kernel/debug/rcu/rcu_preempt/rcudata  # Per-CPU RCU state
cat /proc/sys/kernel/nmi_watchdog               # NMI watchdog (uses RCU GP mechanism)

# RCU stalls warning (if grace period takes > 21 seconds):
# dmesg: INFO: rcu_sched self-detected stall on CPU 3 (soft lockup?)
# Causes: CPU stuck in long loop, RCU read-side critical section sleeping (bug!),
#         CPU offline while in RCU critical section
```

---

## 8. Sleepable RCU (SRCU)

Regular RCU readers cannot sleep. If you need RCU semantics but your read-side code must sleep (e.g., it calls user-space or does I/O), use **SRCU**:

```c
#include <linux/srcu.h>

// Must define a per-subsystem srcu_struct:
DEFINE_SRCU(my_srcu);
// OR:
struct srcu_struct my_srcu;
init_srcu_struct(&my_srcu);

// Reader (CAN sleep!):
int idx = srcu_read_lock(&my_srcu);
// ... access protected data ... CAN sleep here ...
srcu_read_unlock(&my_srcu, idx);

// Writer:
synchronize_srcu(&my_srcu);   // Wait for grace period
// or:
call_srcu(&my_srcu, &old->rcu, my_free_callback);
```

SRCU is heavier than RCU — readers must record which "generation" they started in (the `idx` value). Used by: kernel module unloading, ACPI, some filesystem operations.

---

## 9. Seqlocks — The Writer-Favored Fast Path

A seqlock allows **lock-free reads** for frequently-read, infrequently-written data where readers can detect if a write occurred and retry.

Ideal for: timestamps, system time, quickly-changing counters.

```c
#include <linux/seqlock.h>

// Declaration:
DEFINE_SEQLOCK(my_seqlock);
seqlock_t my_seqlock;
seqlock_init(&my_seqlock);

// Writer (exclusive):
write_seqlock(&my_seqlock);
my_data.x = new_x;
my_data.y = new_y;
write_sequnlock(&my_seqlock);

// Or with IRQ save:
write_seqlock_irqsave(&my_seqlock, flags);
// ...
write_sequnlock_irqrestore(&my_seqlock, flags);

// Reader (no lock — but may retry):
unsigned int seq;
do {
    seq = read_seqbegin(&my_seqlock);
    // Read the data (may be inconsistent if writer runs concurrently):
    x = my_data.x;
    y = my_data.y;
} while (read_seqretry(&my_seqlock, seq));
// After loop: x and y are guaranteed consistent

// How it works:
// write_seqlock:   increments sequence counter (odd = write in progress)
// write_sequnlock: increments again (even = no write in progress)
// read_seqbegin:   reads counter; if odd, spins until even; returns value
// read_seqretry:   reads counter; if changed since seqbegin, return true (retry)
```

### 9.1 Timekeeping — The Canonical Seqlock Example

```c
// kernel/time/timekeeping.c
struct timekeeper {
    seqcount_raw_spinlock_t seq;   // Sequence counter for vDSO
    // ... clock state ...
};

// Writer (called when clock is updated — rare):
write_seqcount_begin(&tk_core.seq);
tk->xtime_sec = new_time;
tk->xtime_nsec = new_nsec;
// ...
write_seqcount_end(&tk_core.seq);

// vDSO reader (runs in user space, no kernel trap!):
do {
    seq = vdso_read_begin(vd);
    ns = vdso_calc_ns(vd, &base, &nsec);
} while (vdso_read_retry(vd, seq));
```

---

## 10. Memory Barriers

Memory barriers are the foundation all synchronization primitives are built on. Without them, CPUs and compilers can reorder memory operations in ways that break concurrent code.

### 10.1 Why Reordering Happens

**Compiler reordering**: The compiler may reorder statements for optimization.
**CPU reordering**: Modern CPUs have store buffers and load queues — stores may not be immediately visible to other CPUs.

```c
// Without barriers (may be wrong on ARM/POWER/RISC-V):
// Thread 1 (producer):
data = 42;         // Might be reordered AFTER flag = 1 by CPU or compiler
flag = 1;

// Thread 2 (consumer):
while (!flag);     // Sees flag=1
use(data);         // But might still see data=0!

// With barrier:
data = 42;
smp_wmb();         // Write memory barrier: all stores before this are visible
flag = 1;          // before stores after this

// Consumer:
while (!flag);
smp_rmb();         // Read memory barrier: all loads after this see stores
use(data);         // that happened before smp_wmb() on the producer
```

### 10.2 Kernel Memory Barrier API

```c
// Full barriers (both read and write):
mb()       // Memory barrier (all processors)
smp_mb()   // SMP-only memory barrier (NOP on UP)

// Store/Write barrier:
wmb()      // Write memory barrier
smp_wmb()  // SMP write barrier

// Load/Read barrier:
rmb()      // Read memory barrier
smp_rmb()  // SMP read barrier

// Compiler-only barrier (no CPU barrier):
barrier()  // Prevents compiler reordering only; CPU may still reorder

// Acquire/Release semantics (C11-style):
smp_load_acquire(&var)     // Load with acquire semantics
smp_store_release(&var, val) // Store with release semantics
// These are the preferred modern API for lock-free programming
```

### 10.3 What Each Barrier Guarantees

```
smp_wmb():  All stores BEFORE the barrier complete before any store AFTER
smp_rmb():  All loads BEFORE the barrier complete before any load AFTER
smp_mb():   All loads+stores before complete before all loads+stores after

Acquire: No memory operation AFTER the acquire can move BEFORE it
Release: No memory operation BEFORE the release can move AFTER it

Classic pattern:
Thread A:              Thread B:
x = data;             while(!y) ;
smp_store_release(&y, 1);   val = smp_load_acquire(&y); // sees 1
                            use(x);  // Guaranteed to see data
```

### 10.4 x86 Memory Model

x86 has a **Total Store Order (TSO)** memory model — stronger than most:
- Stores are always visible to loads in program order
- `smp_wmb()` = NOP on x86 (hardware guarantees this)
- `smp_rmb()` = NOP on x86
- `smp_mb()` = `MFENCE` (or `LOCK XCHG` etc.) — needed for store-load reordering

On ARM64, POWER, and RISC-V, all barriers are real instructions. This is why code developed on x86 can break on ARM — barrier bugs are hidden on x86.

```bash
# See what smp_mb() compiles to on different architectures:
objdump -d kernel.o | grep -A2 -B2 mfence  # x86: MFENCE
# On ARM64: DMB ISH (Data Memory Barrier, Inner Shareable)
# On RISC-V: FENCE rw,rw
```

### 10.5 `READ_ONCE` / `WRITE_ONCE`

Prevent the compiler from performing optimizations that break concurrent access:

```c
// Without READ_ONCE: compiler might hoist the read out of the loop:
while (flag == 0) ;   // Compiler: "flag can't change, load it once"
// Infinite loop even if flag becomes 1!

// With READ_ONCE: forces a fresh load each iteration:
while (READ_ONCE(flag) == 0) ;  // Forces per-iteration load from memory

// WRITE_ONCE: prevents compiler from splitting the write or caching it:
WRITE_ONCE(flag, 1);  // Guaranteed single atomic store

// Also important for documented races (when you *intend* to race):
u64 counter;
// This is a data race, but READ_ONCE suppresses UBSan and documents intent:
u64 val = READ_ONCE(counter);
```

---

## 11. Lock-Free Data Structures with Atomic CAS

For performance-critical structures, CAS (compare-and-swap) operations allow building lock-free algorithms:

```c
#include <linux/atomic.h>

// Compare-and-swap: if *ptr == old, set *ptr = new, return old
// Returns the actual old value (regardless of success)
int actual_old = atomic_cmpxchg(&counter, expected_old, new_val);
bool success = (actual_old == expected_old);

// Example: lock-free push to a stack (Treiber stack):
struct node {
    struct node *next;
    int data;
};

struct node *stack_top = NULL;  // Global stack pointer

void push(struct node *new_node)
{
    struct node *old_top;
    do {
        old_top = READ_ONCE(stack_top);
        new_node->next = old_top;
    } while (cmpxchg(&stack_top, old_top, new_node) != old_top);
    // Loop until we successfully swap: if another thread changed stack_top
    // between our read and our CAS, we retry
}
```

**Warning**: CAS loops can suffer **ABA problems**: thread reads A, another thread changes A→B→A, first thread's CAS succeeds even though the structure changed. Solutions: tagged pointers, hazard pointers, or just use RCU.

---

## 12. `atomic_t` vs `refcount_t`

For reference counting specifically, use `refcount_t` instead of `atomic_t`:

```c
#include <linux/refcount.h>

struct my_obj {
    refcount_t refs;
    // ...
};

// Initialize:
refcount_set(&obj->refs, 1);

// Increment (checked — warns if incrementing from zero):
refcount_inc(&obj->refs);

// Decrement (returns true if it hit zero):
if (refcount_dec_and_test(&obj->refs)) {
    // Last reference dropped — safe to free
    kfree(obj);
}

// Try to get a reference (fails if already zero):
if (!refcount_inc_not_zero(&obj->refs))
    return -ENOENT;  // Object is being destroyed
```

`refcount_t` vs `atomic_t` for refcounting:
- `refcount_t` **saturates** at UINT_MAX (overflow becomes UINT_MAX, not 0) — prevents use-after-free from refcount overflow
- `refcount_t` warns on underflow (decrement past zero = bug)
- `atomic_t` doesn't have these protections

This matters: refcount overflow was a real CVE class (e.g., CVE-2016-0728).

---

## 13. `per_cpu` + Atomics: The Fast Counter Pattern

Combining per-CPU variables with periodic aggregation is how the kernel implements high-performance statistics:

```c
// High-frequency path (no locks, no atomic overhead):
DEFINE_PER_CPU(unsigned long, rx_packets);

// In packet receive (called millions of times/sec):
this_cpu_inc(rx_packets);  // No cache bouncing, no locking

// Low-frequency path (reading stats, called once/second):
unsigned long total = 0;
int cpu;
for_each_possible_cpu(cpu)
    total += per_cpu(rx_packets, cpu);

// Problem: reads may be non-atomic (split reads on 64-bit counters on 32-bit arch)
// Solution: use local_t or u64_stats_t for precise 64-bit per-CPU counters

#include <linux/u64_stats_sync.h>
struct u64_stats_sync stats_sync;
u64 rx_bytes;

// Writer:
u64_stats_update_begin(&stats_sync);
rx_bytes += len;
u64_stats_update_end(&stats_sync);

// Reader:
unsigned int start;
u64 bytes;
do {
    start = u64_stats_fetch_begin(&stats_sync);
    bytes = rx_bytes;
} while (u64_stats_fetch_retry(&stats_sync, start));
```

---

## 14. SRCU in Module Context — Safe Module Unloading

A classic race: module unloads while a reader is inside its code. RCU/SRCU solves this elegantly:

```c
// module.c — safe against concurrent unload:
DEFINE_SRCU(module_srcu);

// In-module reader (could be called from anywhere):
int module_function(void)
{
    int idx = srcu_read_lock(&module_srcu);
    
    // module code here — safe even if rmmod is called concurrently
    int result = do_work();
    
    srcu_read_unlock(&module_srcu, idx);
    return result;
}

// In module exit:
static void __exit mymodule_exit(void)
{
    // Prevent new readers from entering:
    // (This is done by unregistering whatever calls module_function)
    unregister_hook();
    
    // Wait for all current readers to finish:
    synchronize_srcu(&module_srcu);
    
    // Now safe to free all module resources:
    cleanup_resources();
}
```

---

## 15. `mutex_trylock` Pattern for Avoiding Lock Inversion

```c
// When you need two locks but can't guarantee ordering:
// (e.g., operating on two objects of unknown relationship)

bool transfer_safe(struct account *a, struct account *b)
{
    // Always try to lock in address order:
    struct account *first  = (uintptr_t)a < (uintptr_t)b ? a : b;
    struct account *second = (uintptr_t)a < (uintptr_t)b ? b : a;
    
    if (mutex_trylock(&first->lock)) {
        if (mutex_trylock(&second->lock)) {
            // Have both locks
            do_transfer(a, b);
            mutex_unlock(&second->lock);
            mutex_unlock(&first->lock);
            return true;
        }
        mutex_unlock(&first->lock);
    }
    return false;  // Caller should retry
}
```

---

## 16. The Full Picture: Context → Allowed Primitives

```
Execution Context    Sleep?  Primitives Available
──────────────────────────────────────────────────────────────────────
NMI handler          NO     irq_work_queue, READ_ONCE/WRITE_ONCE
                            atomic_*, per-CPU (no locking at all!)

Hardirq handler      NO     spin_lock (preemption already off)
                            atomic_*, per-CPU, kmalloc(GFP_ATOMIC)
                            tasklet_schedule, schedule_work
                            wake_up, complete

Softirq / Tasklet    NO     spin_lock, spin_lock_bh (against other softirqs)
                            atomic_*, per-CPU, kmalloc(GFP_ATOMIC)
                            schedule_work, wake_up, complete
                            RCU read-side (no sleep!)

Process context      YES    Everything above PLUS:
(with spinlock)             spin_lock (nested ok if ordered)
                            NO: schedule, mutex, sleep, GFP_KERNEL

Process context      YES    mutex, rwsem, semaphore, spin_lock
(no spinlock)               GFP_KERNEL, schedule, msleep
                            RCU read-side (and can also call synchronize_rcu)
                            SRCU (can sleep in read-side!)
                            Any synchronization primitive

Atomic context       NO     = any context where preempt_count > 0
(spin_lock held, etc.)       = same as hardirq for locking purposes
```

---

## 17. Putting It All Together: A Realistic Subsystem

A network connection table with realistic locking:

```c
// conntrack entry:
struct conn_entry {
    u32 src_ip, dst_ip;
    u16 src_port, dst_port;
    
    // Per-entry lock (protects state):
    spinlock_t lock;
    int state;
    
    // Reference counting:
    refcount_t refs;
    
    // For RCU list:
    struct hlist_node hash_node;
    struct rcu_head rcu;  // For deferred free
};

// Global hash table (RCU-protected):
#define CONN_HASH_BITS 16
struct hlist_head conn_table[1 << CONN_HASH_BITS];
DEFINE_SPINLOCK(conn_table_lock);  // Serializes writes to hash table

// FAST READ PATH (packet processing — millions/sec):
struct conn_entry *conn_lookup(u32 sip, u16 sport, u32 dip, u16 dport)
{
    u32 hash = jhash_3words(sip, dip, (sport << 16) | dport, 0)
               & ((1 << CONN_HASH_BITS) - 1);
    struct conn_entry *entry;
    
    rcu_read_lock();
    hlist_for_each_entry_rcu(entry, &conn_table[hash], hash_node) {
        if (entry->src_ip == sip && entry->dst_ip == dip &&
            entry->src_port == sport && entry->dst_port == dport) {
            if (refcount_inc_not_zero(&entry->refs)) {
                rcu_read_unlock();
                return entry;  // Caller must conn_put() when done
            }
        }
    }
    rcu_read_unlock();
    return NULL;
}

// WRITE PATH (new connection — infrequent):
int conn_insert(struct conn_entry *new_entry)
{
    u32 hash = hash_conn(new_entry);
    
    spin_lock_bh(&conn_table_lock);
    hlist_add_head_rcu(&new_entry->hash_node, &conn_table[hash]);
    spin_unlock_bh(&conn_table_lock);
    
    return 0;
}

// REMOVE PATH:
void conn_remove(struct conn_entry *entry)
{
    spin_lock_bh(&conn_table_lock);
    hlist_del_rcu(&entry->hash_node);
    spin_unlock_bh(&conn_table_lock);
    
    // Decrement ref (we held one for the table):
    conn_put(entry);
}

void conn_put(struct conn_entry *entry)
{
    if (refcount_dec_and_test(&entry->refs))
        kfree_rcu(entry, rcu);  // Free after grace period
}
```

This combines:
- **RCU** for lock-free reads of the hash table
- **refcount_t** for safe object lifetime management
- **spinlock_bh** for serializing writes (protects against softirqs, not just concurrent CPUs)
- **kfree_rcu** for deferred freeing after grace period

---

## 18. Debugging Concurrency Issues

### 18.1 KCSAN — Kernel Concurrency Sanitizer

```bash
# Enable KCSAN (kernel data race detector):
scripts/config --enable KCSAN
make -j$(nproc)

# KCSAN instruments every memory access and reports unprotected concurrent accesses.
# When a race is detected, dmesg shows:
# ==================================================================
# BUG: KCSAN: data-race in conn_lookup / conn_remove
#
# read to 0xffff888123456789 of 4 bytes by task 1234 on cpu 2:
#  conn_lookup+0x45/0x90
#
# write to 0xffff888123456789 of 4 bytes by task 5678 on cpu 3:
#  conn_remove+0x22/0x40

# Annotate intentional races (to suppress false positives):
// Reader: data race is intentional, use READ_ONCE:
state = READ_ONCE(entry->state);

// Or mark as racy with documentation:
// data_race(x = global_counter);  // Intentional approximate read
```

### 18.2 Lockdep Extended Annotations

```c
// Tell lockdep about lock classes for complex scenarios:

// Subclasses: when you lock two objects of the same type, annotate the order:
mutex_lock_nested(&parent->lock, 0);  // "outer" class
mutex_lock_nested(&child->lock, 1);   // "inner" class

// RCU lockdep: verify RCU is held when accessing RCU-protected data:
struct data *d = rcu_dereference_check(ptr,
    rcu_read_lock_held() || lockdep_is_held(&my_mutex));
// This allows access either inside RCU read-lock OR while holding my_mutex

// Assert RCU is held:
RCU_LOCKDEP_WARN(!rcu_read_lock_held(), "RCU not locked!");
```

### 18.3 `sparse` RCU Annotations

```c
// Mark RCU-protected pointers for sparse to verify correct usage:
struct my_data __rcu *my_ptr;  // __rcu tells sparse this needs rcu_dereference

// Incorrect access (sparse will warn):
my_ptr->field = 1;  // sparse: dereference of RCU-protected pointer without rcu_dereference

// Correct:
rcu_read_lock();
rcu_dereference(my_ptr)->field = 1;  // Still wrong at runtime if writer races...
rcu_read_unlock();
```

---

## 19. Performance Comparison

```bash
# Benchmark different synchronization mechanisms (rough order):

# Mutex (uncontended):           ~10-30ns  (fast path: single CAS)
# Mutex (contended):             ~500ns-5µs (context switch cost)
# Spinlock (uncontended):        ~5-15ns   (single CMPXCHG)
# Spinlock (contended, 2 CPUs):  ~50-100ns (cache line bouncing)
# RCU read-side:                 ~1-3ns    (memory barrier only)
# RCU synchronize_rcu():         ~10-100ms (full grace period)
# atomic_t inc:                  ~5-15ns   (LOCK XADD on x86)
# per-CPU inc (this_cpu_inc):    ~1-2ns    (simple INCQ, no lock prefix)

# Measure in your environment:
# (use perf or a microbenchmark)
```

---

## 20. Mental Model Checkpoint

After Day 12, you should be able to:

1. Explain RCU with the "publish-subscribe" analogy. What is a grace period?
2. Write a correct RCU reader path with `rcu_read_lock` / `rcu_dereference`.
3. Write a correct RCU writer path with `rcu_assign_pointer`, `synchronize_rcu`, and `kfree`.
4. Explain the difference between `synchronize_rcu()` and `call_rcu()`. When would you choose each?
5. Use a seqlock: write the writer path and the retry-loop reader path.
6. What does `smp_wmb()` guarantee? When is it a NOP?
7. Why is `READ_ONCE()` needed even on x86?
8. When should you use `refcount_t` instead of `atomic_t`?
9. Given a struct protected by both RCU (for list membership) and a spinlock (for field updates), write the read and update paths.
10. What does KCSAN detect that lockdep does not?

---

## Key Source Files

```bash
include/linux/rcupdate.h          # Core RCU API (rcu_read_lock, rcu_dereference)
include/linux/rculist.h           # RCU list operations
kernel/rcu/tree.c                 # RCU tree implementation (grace periods, callbacks)
kernel/rcu/sync.c                 # synchronize_rcu() implementation
include/linux/seqlock.h           # Seqlock and seqcount API
include/linux/atomic.h            # Atomic operations
include/linux/compiler.h          # READ_ONCE, WRITE_ONCE, barrier()
include/asm-generic/barrier.h     # smp_mb, smp_rmb, smp_wmb
include/linux/refcount.h          # refcount_t — safe reference counting
include/linux/srcu.h              # Sleepable RCU
Documentation/RCU/                # Extensive RCU documentation (MUST READ)
Documentation/RCU/whatisRCU.rst   # RCU conceptual introduction
Documentation/memory-barriers.txt # Full memory barrier documentation (long!)
```

---

## Summary of Phase 1

Congratulations — you've covered all 12 foundational topics:

| Day | Topic | Key Takeaway |
|-----|-------|-------------|
| 1 | Architecture | Monolithic + modular; ring 0/3; four paths to kernel |
| 2 | Build system | Kconfig + Kbuild; obj-m/obj-y; compile + boot |
| 3 | Boot process | Power-on → start_kernel → init → /sbin/init |
| 4 | Syscalls mechanics | SYSCALL MSR → entry_SYSCALL_64 → sys_call_table |
| 5 | Syscalls tracing | strace/perf/bpftrace; write your own syscall |
| 6 | Kernel modules | ELF loading; EXPORT_SYMBOL; char device pattern |
| 7 | Data structures | list_head, rbtree, hashtable, xarray, percpu |
| 8 | Memory layout | physmap; zones; KASLR; buddy + SLUB allocators |
| 9 | Interrupts HW | IDT; APIC; request_irq; hardirq rules |
| 10 | Interrupts SW | Softirqs; tasklets; workqueues; threaded IRQs |
| 11 | Locking basics | Spinlock; mutex; atomic; lockdep; ordering rules |
| 12 | Locking advanced | RCU; seqlock; memory barriers; refcount_t; KCSAN |

**Phase 2 begins** with process management — `task_struct`, `fork()`, the scheduler, and SMP. The foundation you've built here directly enables everything that follows.
