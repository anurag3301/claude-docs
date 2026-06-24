# Day 6 — Kernel Synchronization: Spinlocks, Mutexes, RCU & Atomic Ops
**Environment:** 🔵 RPi4
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [Why Synchronization Matters](#1-why-synchronization-matters)
2. [Atomic Operations](#2-atomic-operations)
3. [Spinlocks](#3-spinlocks)
4. [Mutexes and Semaphores](#4-mutexes-and-semaphores)
5. [Read-Write Locks](#5-read-write-locks)
6. [RCU — Read-Copy-Update](#6-rcu--read-copy-update)
7. [Seqlocks](#7-seqlocks)
8. [Memory Barriers](#8-memory-barriers)
9. [Choosing the Right Primitive](#9-choosing-the-right-primitive)
10. [Hands-On Lab — Race Condition Demo on RPi4](#10-hands-on-lab--race-condition-demo-on-rpi4)
11. [Interview Questions](#11-interview-questions)

---

## 1. Why Synchronization Matters

The kernel is massively concurrent. On a 4-core RPi4, four kernel threads can run simultaneously. Hardware interrupt handlers preempt running code at any moment. Without synchronization, shared data gets silently corrupted.

### The Classic Race

```c
static int counter = 0;
void increment(void) { counter++; }  // LOOKS atomic. ISN'T.
```

`counter++` compiles to three instructions:

```asm
ldr  w0, [counter_addr]   // 1. Load
add  w0, w0, #1           // 2. Add
str  w0, [counter_addr]   // 3. Store
```

Two CPUs running this simultaneously:

```
CPU0: ldr w0=0
                  CPU1: ldr w0=0
CPU0: add w0=1
CPU0: str counter=1
                  CPU1: add w0=1
                  CPU1: str counter=1   ← overwrites CPU0's work!
```

Result: `counter=1` instead of `2`. The increment from CPU0 is lost.

### Sources of Concurrency in the Kernel

| Source | Description |
|--------|-------------|
| **SMP** | Multiple CPUs executing code simultaneously |
| **Kernel preemption** | Scheduler interrupts kernel code between any two instructions (`CONFIG_PREEMPT`) |
| **Hardware IRQs** | Can fire at any time, preempting any code |
| **Softirqs/Tasklets** | Software interrupt handlers deferred from hard IRQs |
| **Workqueues** | Kernel threads running concurrently |

---

## 2. Atomic Operations

The simplest synchronization form — indivisible operations on single variables. No lock needed.

```c
#include <linux/atomic.h>

// Declare and initialize
atomic_t counter = ATOMIC_INIT(0);
atomic64_t big   = ATOMIC64_INIT(0);

// Read / Write
int v = atomic_read(&counter);
atomic_set(&counter, 5);

// Arithmetic
atomic_inc(&counter);                   // counter++
atomic_dec(&counter);                   // counter--
atomic_add(5, &counter);               // counter += 5
atomic_sub(3, &counter);               // counter -= 3

// Return new value after operation
int n = atomic_inc_return(&counter);   // return ++counter
int n = atomic_dec_return(&counter);   // return --counter
int n = atomic_add_return(5, &counter);

// Test after operation
bool hit_zero = atomic_dec_and_test(&counter); // true if result==0

// Compare and exchange
int old = atomic_cmpxchg(&counter, expected, new_val);
// sets counter=new_val only if counter==expected; returns old value

// Exchange
int prev = atomic_xchg(&counter, new_val);
```

### Bit Atomics

```c
#include <linux/bitops.h>

unsigned long flags = 0;

set_bit(3, &flags);             // set bit 3
clear_bit(3, &flags);           // clear bit 3
change_bit(3, &flags);          // toggle bit 3
bool old = test_and_set_bit(3, &flags);   // atomic test+set
bool old = test_and_clear_bit(3, &flags); // atomic test+clear
bool is  = test_bit(3, &flags); // just test
```

### Use `refcount_t` for Reference Counts

Never use `atomic_t` for reference counts — use the purpose-built `refcount_t`:

```c
#include <linux/refcount.h>

refcount_t ref = REFCOUNT_INIT(1);

refcount_inc(&ref);
bool zero = refcount_dec_and_test(&ref); // true → safe to free
```

`refcount_t` detects overflow (which would cause premature freeing) and underflow, panicking instead of silently corrupting memory.

### When Atomics Are Enough vs When You Need Locks

- **Enough:** single counter, single flag, single pointer, reference count
- **Not enough:** protecting multiple related fields that must change together, linked list operations, multi-step transactions

---

## 3. Spinlocks

A spinlock is a lock where the waiter **busy-spins** (loops checking the lock) rather than sleeping. This makes it safe in interrupt context and fast for very short critical sections.

```c
#include <linux/spinlock.h>

// Declaration and initialization
DEFINE_SPINLOCK(my_lock);           // static
// OR:
spinlock_t lock;
spin_lock_init(&lock);              // dynamic

// Basic usage (process context only — no IRQ sharing)
spin_lock(&lock);
/* critical section */
spin_unlock(&lock);

// When the lock is ALSO taken from interrupt handlers
unsigned long flags;
spin_lock_irqsave(&lock, flags);   // lock + disable IRQs on this CPU
/* critical section */
spin_unlock_irqrestore(&lock, flags); // unlock + restore IRQ state

// When lock is shared with softirqs/tasklets only (not hard IRQs)
spin_lock_bh(&lock);
/* critical section */
spin_unlock_bh(&lock);

// Non-blocking attempt
if (spin_trylock(&lock)) {
    /* got the lock */
    spin_unlock(&lock);
} else {
    /* lock was busy — take alternate path */
}
```

### The `irqsave` vs Plain `spin_lock` Decision — This Is Critical

```c
// SCENARIO: shared counter, accessed from process context AND IRQ handler

static int shared = 0;
static spinlock_t lock;

/* Process context — WRONG usage */
void process_func(void) {
    spin_lock(&lock);         // acquires lock
    shared++;
    /* <<< IRQ fires HERE on same CPU >>> */
    /* IRQ handler calls spin_lock(&lock) — tries to acquire a lock  */
    /* already held by THIS CPU → CPU spins forever → DEADLOCK        */
    spin_unlock(&lock);
}

static irqreturn_t my_irq(int irq, void *data) {
    spin_lock(&lock);         // spinning on CPU that holds the lock
    shared--;
    spin_unlock(&lock);
    return IRQ_HANDLED;
}
```

Fix with `irqsave`:

```c
void process_func(void) {
    unsigned long flags;
    spin_lock_irqsave(&lock, flags);  // disables IRQs on this CPU
    shared++;                          // IRQ cannot fire here
    spin_unlock_irqrestore(&lock, flags);
}

static irqreturn_t my_irq(int irq, void *data) {
    spin_lock(&lock);    // safe: IRQs already disabled on this CPU by irqsave
    shared--;            // OR use spin_lock_irqsave for full safety
    spin_unlock(&lock);
    return IRQ_HANDLED;
}
```

### Cardinal Spinlock Rules

1. **Never sleep while holding a spinlock** — no `kmalloc(GFP_KERNEL)`, no `msleep()`, no `copy_from_user()`, no `mutex_lock()`
2. **Always use `irqsave`** if the same lock is ever acquired in an IRQ handler
3. **Never hold two locks in different orders** across code paths — causes deadlock
4. **Keep critical sections tiny** — spinning wastes CPU cycles
5. **Not reentrant** — re-acquiring a spinlock you hold → instant deadlock

Enable these during development to catch violations:
```bash
CONFIG_DEBUG_SPINLOCK=y
CONFIG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
```

---

## 4. Mutexes and Semaphores

Unlike spinlocks, mutexes **sleep** when contended. The waiting task is put to sleep and woken when the lock is available — no CPU wasted spinning.

### Mutex

```c
#include <linux/mutex.h>

DEFINE_MUTEX(my_mutex);   // static
// OR:
struct mutex m;
mutex_init(&m);           // dynamic

// Lock — sleeps if contended (MUST be in process context, NOT IRQ)
mutex_lock(&my_mutex);
/* critical section — can sleep here, can kmalloc(GFP_KERNEL) */
mutex_unlock(&my_mutex);

// Interruptible — wakes if signal arrives (Ctrl+C, SIGTERM)
int ret = mutex_lock_interruptible(&my_mutex);
if (ret == -EINTR) return -EINTR;
/* ... */
mutex_unlock(&my_mutex);

// Killable — wakes only for fatal signals
int ret = mutex_lock_killable(&my_mutex);

// Non-blocking
if (mutex_trylock(&my_mutex)) {
    /* got it */
    mutex_unlock(&my_mutex);
}
```

### Mutex Rules

1. **Process context only** — never in IRQ handlers or with IRQs/preemption disabled
2. **Not reentrant** — same task locking a mutex it holds → deadlock
3. **Owner must unlock** — the task that locked it must be the one to unlock it
4. **Never hold while calling code that might lock the same mutex** (re-entrancy deadlock)

### Semaphores (Counting Locks)

Prefer `mutex` for mutual exclusion. Use semaphores only when you need N > 1 simultaneous holders:

```c
#include <linux/semaphore.h>

struct semaphore sem;
sema_init(&sem, 1);   // binary (mutex-like)
sema_init(&sem, 4);   // allows 4 concurrent holders

down(&sem);           // decrement; sleep if 0 (non-interruptible)
down_interruptible(&sem);  // sleeps but wakes on signal
up(&sem);             // increment; wake one waiter
```

---

## 5. Read-Write Locks

Allow many simultaneous readers, but writers get exclusive access.

### `rwlock_t` — For Interrupt Context

```c
#include <linux/rwlock.h>

DEFINE_RWLOCK(rw);

// Many readers simultaneously
read_lock(&rw);
/* read shared data */
read_unlock(&rw);

// IRQ-safe variants
unsigned long flags;
read_lock_irqsave(&rw, flags);
read_unlock_irqrestore(&rw, flags);

// One writer at a time (exclusive)
write_lock(&rw);
/* modify shared data */
write_unlock(&rw);

write_lock_irqsave(&rw, flags);
write_unlock_irqrestore(&rw, flags);
```

### `rwsem` — For Process Context (Can Sleep)

```c
#include <linux/rwsem.h>

DECLARE_RWSEM(rw_sem);

down_read(&rw_sem);   /* read */   up_read(&rw_sem);
down_write(&rw_sem);  /* write */  up_write(&rw_sem);

// Downgrade write → read (release exclusivity, keep shared access)
downgrade_write(&rw_sem);
```

---

## 6. RCU — Read-Copy-Update

RCU is the highest-performance synchronization mechanism in the kernel. Used in the network stack, VFS, and device model. Reads are **completely lock-free** — zero overhead on the read side.

### The Core Idea

```
Old ptr ──► [data: A]
                           Writer makes a copy:
New ptr ──► [data: B]      New copy updated, pointer atomically switched
                           Old copy freed AFTER all readers using it finish
```

Readers never block writers. Writers never block readers. Old data is freed only after a **grace period** — a time after which no CPU holds a reference to it.

### RCU API

```c
#include <linux/rcupdate.h>

/* ──── READER SIDE ──── */
rcu_read_lock();
// Disable preemption — this CPU won't pass through a quiescent state
// Reads are lock-free and can run in any context

struct myobj *p = rcu_dereference(global_ptr);
// rcu_dereference adds a data dependency barrier preventing
// the CPU from speculating past the pointer load

if (p)
    use(p->field);  // guaranteed valid during RCU read-side CS

rcu_read_unlock();
// Preemption re-enabled — quiescent states can now occur


/* ──── WRITER SIDE ──── */
struct myobj *old = rcu_dereference_protected(global_ptr,
                        lockdep_is_held(&my_writer_lock));

struct myobj *new = kmalloc(sizeof(*new), GFP_KERNEL);
*new = *old;           // copy
new->field = new_val;  // update the copy

rcu_assign_pointer(global_ptr, new);
// Atomic pointer replacement + memory barrier
// After this, NEW readers see new pointer

synchronize_rcu();     // BLOCK until all pre-existing readers finish
                       // (grace period elapsed)
kfree(old);            // now safe to free

// Non-blocking alternative:
call_rcu(&old->rcu_head, free_old_callback);
```

### RCU Grace Period Explained

A **grace period** is a span of time after `rcu_assign_pointer()` during which every CPU passes through a **quiescent state** — a moment where it cannot be accessing old RCU-protected data. Quiescent states include:
- Returning to user space
- Context switching (calling `schedule()`)
- Being idle

`synchronize_rcu()` returns only after every CPU has passed through at least one quiescent state. After that, no reader started before the pointer update can still be running, so the old object is safe to free.

### RCU List Operations

```c
#include <linux/rculist.h>

// Add — safe for concurrent lockfree readers
spin_lock(&writer_lock);
list_add_rcu(&new->node, &my_list);
spin_unlock(&writer_lock);

// Delete
spin_lock(&writer_lock);
list_del_rcu(&entry->node);
spin_unlock(&writer_lock);
call_rcu(&entry->rcu, free_entry);  // free after grace period

// Traverse — lockfree reader
rcu_read_lock();
list_for_each_entry_rcu(item, &my_list, node) {
    process(item);
}
rcu_read_unlock();
```

### When to Use RCU

**Good fit:**
- Read-mostly data: routing tables, module lists, file system dentries
- Reads in interrupt context (rcu_read_lock is cheap and IRQ-safe)
- Performance-critical read paths

**Bad fit:**
- Write-heavy workloads (each write blocks in `synchronize_rcu()`)
- Very tight memory budget (old objects live until grace period)
- Readers need to sleep (can't sleep inside `rcu_read_lock()`)

---

## 7. Seqlocks

Seqlocks let writers proceed without waiting for readers. Readers detect if a write occurred during their read and retry. Ideal for frequently-read but rarely-written data.

```c
#include <linux/seqlock.h>

DEFINE_SEQLOCK(my_seqlock);

/* Writer — exclusive, serialized among writers */
write_seqlock(&my_seqlock);
/* update data */
write_sequnlock(&my_seqlock);

/* Reader — optimistic, retries on conflict */
unsigned int seq;
do {
    seq = read_seqbegin(&my_seqlock);
    /* read data */
} while (read_seqretry(&my_seqlock, seq));
// read_seqretry returns true if a write happened during the read
```

Used in kernel timekeeping: reading `jiffies_64` and `xtime` is done via seqlock — the counter is read millions of times per second with only occasional writes.

---

## 8. Memory Barriers

Modern CPUs and compilers reorder memory accesses for performance. This breaks concurrent code even with correct locking around it.

```c
#include <asm/barrier.h>

mb();      // full memory barrier: no loads or stores reordered across
rmb();     // read barrier: no loads reordered across
wmb();     // write barrier: no stores reordered across
smp_mb();  // SMP-safe memory barrier (compiles to nothing on UP)
smp_rmb();
smp_wmb();

// Acquire/Release (C11 semantics)
val = smp_load_acquire(&ptr);    // load; subsequent accesses can't move before
smp_store_release(&ptr, val);    // store; prior accesses can't move after
```

### Concrete Driver Example

```c
/* DMA descriptor handed to hardware */
struct dma_desc {
    u32 buf_phys;   // physical address of data buffer
    u32 length;     // byte count
    u32 flags;      // bit0 = 1 means "owned by DMA controller"
};

/* WRONG — CPU may store flags before buf_phys/length */
desc->buf_phys = phys;
desc->length   = len;
desc->flags    = DMA_OWN;   // DMA starts with stale buf_phys!

/* CORRECT */
desc->buf_phys = phys;
desc->length   = len;
wmb();                       // guarantee: stores above complete first
desc->flags    = DMA_OWN;   // DMA controller now sees correct buf_phys
```

ARM64 (RPi4's architecture) is **weakly ordered** — both loads and stores can be reordered. x86 is strongly ordered for stores but the compiler still reorders. Always use barriers when writing to hardware-visible memory.

In practice: use `iowrite32()` / `ioread32()` for MMIO registers — these include implicit barriers. Use `dma_wmb()` / `dma_rmb()` for DMA descriptor rings.

---

## 9. Choosing the Right Primitive

```
Are you in interrupt context or holding a spinlock? (Can't sleep)
  YES:
    Single integer/counter? ────────────→ atomic_t / atomic64_t
    Single flag/bit?        ────────────→ set_bit / clear_bit
    Protecting a struct?
      Accessed from IRQ handler too? ──→ spinlock + spin_lock_irqsave()
      Only softirq/tasklet?          ──→ spinlock + spin_lock_bh()

  NO (Process context, can sleep):
    Simple counter/flag?    ────────────→ atomic_t
    Protecting a struct:
      Many readers, rare writes?
        Readers can detect conflict?  ──→ seqlock
        Readers need guaranteed data? ──→ rwsem
        Reads must be truly lockfree? ──→ RCU
      Standard mutual exclusion?      ──→ mutex
      N concurrent holders?           ──→ semaphore (sema_init(N))
```

### Quick Reference Table

| Primitive | Sleeps? | IRQ-safe? | Speed | Use Case |
|-----------|---------|-----------|-------|----------|
| `atomic_t` | No | Yes | Fastest | Counters, flags |
| `spinlock_t` | No | Yes (irqsave) | Fast | Short critical sections |
| `mutex` | Yes | No | Medium | Long critical sections |
| `rwlock_t` | No | Yes | Fast reads | Read-heavy, IRQ context |
| `rwsem` | Yes | No | Medium reads | Read-heavy, process context |
| RCU | No (read) | Yes (read) | Lockfree reads | High-perf read-mostly data |
| `seqlock` | No | Yes | Fast | Rare writes, readers retry OK |

---

## 10. Hands-On Lab — Race Condition Demo on RPi4

### Part A — Intentional Race Condition

```c
// race_demo.c
#include <linux/module.h>
#include <linux/kthread.h>
#include <linux/delay.h>
#include <linux/atomic.h>
#include <linux/spinlock.h>

#define NUM_THREADS  4
#define ITERATIONS   100000

static unsigned long  unprotected = 0;
static unsigned long  spinlocked  = 0;
static atomic_long_t  atomicval   = ATOMIC_LONG_INIT(0);
static DEFINE_SPINLOCK(sp_lock);
static atomic_t threads_done = ATOMIC_INIT(0);

static int counter_thread(void *unused)
{
    int i;
    for (i = 0; i < ITERATIONS; i++) {
        /* ---- WRONG: plain increment ---- */
        unprotected++;

        /* ---- CORRECT: spinlock ---- */
        spin_lock(&sp_lock);
        spinlocked++;
        spin_unlock(&sp_lock);

        /* ---- CORRECT: atomic ---- */
        atomic_long_inc(&atomicval);
    }
    atomic_inc(&threads_done);
    return 0;
}

static int __init race_init(void)
{
    int i, expected = NUM_THREADS * ITERATIONS;

    pr_info("race_demo: launching %d threads × %d iterations = %d expected\n",
            NUM_THREADS, ITERATIONS, expected);

    for (i = 0; i < NUM_THREADS; i++)
        kthread_run(counter_thread, NULL, "race_%d", i);

    /* Wait for completion */
    while (atomic_read(&threads_done) < NUM_THREADS)
        msleep(100);

    pr_info("race_demo: RESULTS\n");
    pr_info("  Expected:     %d\n", expected);
    pr_info("  Unprotected:  %lu  ← likely WRONG due to race\n", unprotected);
    pr_info("  Spinlocked:   %lu  ← should be exact\n", spinlocked);
    pr_info("  Atomic:       %ld  ← should be exact\n",
            atomic_long_read(&atomicval));
    return 0;
}
static void __exit race_exit(void) {}

module_init(race_init);
module_exit(race_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Race condition demonstration");
```

```bash
# On RPi4
sudo insmod race_demo.ko
sleep 5
dmesg | tail -8
# Unprotected will almost certainly be LESS than 400000
# Spinlocked and Atomic will be exactly 400000
sudo rmmod race_demo
```

### Part B — Trigger a Lockdep Warning

```c
// lockdep_demo.c — violate lock ordering so lockdep catches it
#include <linux/module.h>
#include <linux/spinlock.h>

static DEFINE_SPINLOCK(lock_a);
static DEFINE_SPINLOCK(lock_b);

static int __init lockdep_init(void)
{
    unsigned long f;

    /* Establish ordering: A → B */
    spin_lock_irqsave(&lock_a, f);
    spin_lock(&lock_b);
    spin_unlock(&lock_b);
    spin_unlock_irqrestore(&lock_a, f);

    /* Now reverse: B → A  — lockdep WILL warn here */
    spin_lock_irqsave(&lock_b, f);
    spin_lock(&lock_a);   /* <<< lockdep warning: circular dependency */
    spin_unlock(&lock_a);
    spin_unlock_irqrestore(&lock_b, f);

    return 0;
}
static void __exit lockdep_exit(void) {}

module_init(lockdep_init);
module_exit(lockdep_exit);
MODULE_LICENSE("GPL");
```

```bash
# Requires kernel built with CONFIG_LOCKDEP=y, CONFIG_PROVE_LOCKING=y
sudo insmod lockdep_demo.ko
dmesg | grep -A 40 "circular locking"
# You'll see a detailed report showing both lock acquisition sites
sudo rmmod lockdep_demo
```

### Part C — Shared-Buffer Driver with Real Synchronization

```c
// sync_buf.c — character device with properly synchronized ring buffer

#include <linux/module.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/mutex.h>
#include <linux/slab.h>
#include <linux/wait.h>

#define BUF_SIZE  256
#define DEV_NAME  "sync_buf"

struct sync_buf_dev {
    struct cdev           cdev;
    char                  buf[BUF_SIZE];
    size_t                head, tail, count;   /* ring buffer state */
    struct mutex          lock;
    wait_queue_head_t     read_wq;
    wait_queue_head_t     write_wq;
};

static dev_t dev_num;
static struct sync_buf_dev *sdev;
static struct class *sclass;

static int sbuf_open(struct inode *i, struct file *f)
{
    f->private_data = container_of(i->i_cdev, struct sync_buf_dev, cdev);
    return 0;
}

static ssize_t sbuf_write(struct file *f, const char __user *u,
                           size_t count, loff_t *pos)
{
    struct sync_buf_dev *d = f->private_data;
    ssize_t written = 0;

    if (mutex_lock_interruptible(&d->lock))
        return -EINTR;

    while (count > 0 && d->count < BUF_SIZE) {
        d->buf[d->head] = 0;
        if (copy_from_user(&d->buf[d->head], u + written, 1)) {
            mutex_unlock(&d->lock);
            return -EFAULT;
        }
        d->head = (d->head + 1) % BUF_SIZE;
        d->count++;
        written++;
        count--;
    }

    mutex_unlock(&d->lock);

    if (written)
        wake_up_interruptible(&d->read_wq);

    return written ? written : -ENOBUFS;
}

static ssize_t sbuf_read(struct file *f, char __user *u,
                          size_t count, loff_t *pos)
{
    struct sync_buf_dev *d = f->private_data;
    ssize_t read_bytes = 0;
    int ret;

    /* Block until data is available */
    ret = wait_event_interruptible(d->read_wq, d->count > 0);
    if (ret) return -EINTR;

    if (mutex_lock_interruptible(&d->lock))
        return -EINTR;

    while (count > 0 && d->count > 0) {
        if (copy_to_user(u + read_bytes, &d->buf[d->tail], 1)) {
            mutex_unlock(&d->lock);
            return -EFAULT;
        }
        d->tail = (d->tail + 1) % BUF_SIZE;
        d->count--;
        read_bytes++;
        count--;
    }

    mutex_unlock(&d->lock);
    wake_up_interruptible(&d->write_wq);
    return read_bytes;
}

static const struct file_operations sbuf_fops = {
    .owner   = THIS_MODULE,
    .open    = sbuf_open,
    .read    = sbuf_read,
    .write   = sbuf_write,
};

static int __init sbuf_init(void)
{
    int ret;

    sdev = kzalloc(sizeof(*sdev), GFP_KERNEL);
    if (!sdev) return -ENOMEM;

    mutex_init(&sdev->lock);
    init_waitqueue_head(&sdev->read_wq);
    init_waitqueue_head(&sdev->write_wq);

    alloc_chrdev_region(&dev_num, 0, 1, DEV_NAME);
    cdev_init(&sdev->cdev, &sbuf_fops);
    sdev->cdev.owner = THIS_MODULE;
    cdev_add(&sdev->cdev, dev_num, 1);

    sclass = class_create(DEV_NAME);
    device_create(sclass, NULL, dev_num, NULL, DEV_NAME);

    pr_info("sync_buf: ready on /dev/%s\n", DEV_NAME);
    return 0;
}

static void __exit sbuf_exit(void)
{
    device_destroy(sclass, dev_num);
    class_destroy(sclass);
    cdev_del(&sdev->cdev);
    unregister_chrdev_region(dev_num, 1);
    kfree(sdev);
}

module_init(sbuf_init);
module_exit(sbuf_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Synchronized ring-buffer char device");
```

```bash
# On RPi4
sudo insmod sync_buf.ko
sudo chmod 666 /dev/sync_buf

# Terminal 1 — blocking reader
cat /dev/sync_buf

# Terminal 2 — writer unblocks the reader
echo "hello from kernel" > /dev/sync_buf

# Stress test: multiple concurrent writers
for i in $(seq 1 10); do
    echo "msg_${i}" > /dev/sync_buf &
done
cat /dev/sync_buf  # read all messages

sudo rmmod sync_buf
```

---

## 11. Interview Questions

---

**Q1. What is a spinlock and when MUST you use `spin_lock_irqsave` vs plain `spin_lock`?**

**Answer:** A spinlock is a mutual exclusion primitive where the contending CPU busy-waits in a loop until the lock is released. It's safe in interrupt context (since it doesn't sleep) and efficient for very short critical sections.

You **must** use `spin_lock_irqsave()` whenever the same lock is also acquired from a hardware interrupt handler. The reason is deadlock: if CPU0 holds the lock via `spin_lock()`, and a hardware interrupt fires on CPU0, the IRQ handler tries to acquire the same lock. CPU0 is now spinning waiting for a lock it itself holds — it can never release it. Deadlock.

`spin_lock_irqsave(&lock, flags)` atomically disables hardware interrupts on the current CPU before acquiring the lock, preventing the IRQ from firing. `flags` saves the current IRQ enable state (you might already be in an IRQ handler), and `spin_unlock_irqrestore()` restores it exactly. This is always the safe choice when a lock is shared with IRQ context.

---

**Q2. What is the difference between a spinlock and a mutex? Which can you use in interrupt context?**

**Answer:**

| | Spinlock | Mutex |
|---|---|---|
| Waiting behavior | Busy-spins (consumes CPU) | Sleeps (task put on wait queue) |
| Interrupt context | ✅ Safe | ❌ Never |
| Can sleep while held | ❌ No | ✅ Yes |
| Duration | Very short (nanoseconds) | Short to long |
| Ownership | None (any CPU can unlock) | Owner-only unlock |

Only spinlocks can be used in interrupt handlers — interrupt handlers cannot sleep, and `mutex_lock()` calls `schedule()` when contended, which is illegal in interrupt context. Using a mutex in an IRQ handler with `CONFIG_DEBUG_ATOMIC_SLEEP=y` will immediately print a warning and stack trace.

Use spinlocks when: in IRQ context, or when critical section is tiny (< ~50 instructions). Use mutexes when: in process context with longer critical sections where sleeping while waiting saves CPU cycles.

---

**Q3. Explain RCU. What does "grace period" mean?**

**Answer:** RCU (Read-Copy-Update) allows reads to proceed completely lock-free while writes modify a private copy and atomically replace a pointer.

**Write sequence:** (1) Copy the data structure, (2) Update the copy, (3) Use `rcu_assign_pointer()` to atomically replace the shared pointer, (4) Wait for a grace period, (5) Free the old copy.

A **grace period** is the time between the pointer replacement and when it's safe to free the old data. It's the interval after which every CPU that could have read the old pointer has passed through a **quiescent state** — a point where it can't possibly hold a reference to the old object. Quiescent states include: returning to user space, calling `schedule()`, or being idle. `synchronize_rcu()` blocks until this condition is met on all CPUs. `call_rcu()` schedules a callback asynchronously instead of blocking.

Readers use `rcu_read_lock()` (disables preemption, preventing quiescent states) and `rcu_dereference()` (includes data dependency barrier). Reading is completely lock-free — zero overhead on the common fast path. This is why RCU dominates high-performance kernel subsystems like the network routing table.

---

**Q4. What are memory barriers and give a concrete driver example where omitting one causes a bug?**

**Answer:** Memory barriers prevent CPUs and compilers from reordering memory accesses across them. Modern CPUs execute instructions out of order for performance; without barriers, the hardware-visible order of stores can differ from program order.

**Driver example — DMA descriptor ring:**

```c
struct dma_desc {
    u32 buf_addr;    // physical address of buffer
    u32 length;      // byte count
    u32 ctrl;        // bit0=1 means "DMA controller owns this descriptor"
};

/* WRONG — no barrier */
desc->buf_addr = phys_addr;   // CPU may store these in any order
desc->length   = 4096;
desc->ctrl     = DMA_OWN;     // if DMA controller sees ctrl=1 before
                               // buf_addr is written → reads garbage address
                               // → silent data corruption or system crash

/* CORRECT */
desc->buf_addr = phys_addr;
desc->length   = 4096;
wmb();                        // all stores above complete before stores below
desc->ctrl     = DMA_OWN;    // DMA controller now guaranteed to see correct addr
```

ARM64 (RPi4) is weakly ordered — both loads and stores can be reordered. The `wmb()` maps to `dsb st` on ARM64 (Data Synchronization Barrier, store only). In practice, use `dma_wmb()` for DMA descriptors and `iowrite32()`/`ioread32()` for MMIO — both include the necessary barriers.

---

**Q5. What is `atomic_t`? Why can't you just use `int + spinlock` for everything?**

**Answer:** `atomic_t` wraps an `int` with hardware-guaranteed indivisible operations. On x86, implemented with the `LOCK` prefix; on ARM64 with load-exclusive / store-exclusive (`LDXR`/`STXR`) instruction pairs.

You can use `int + spinlock` for everything, but `atomic_t` is better for simple operations because:

1. **Performance**: `atomic_inc()` is a single instruction. `spin_lock() + counter++ + spin_unlock()` is ~10+ instructions plus lock overhead and disabling preemption.
2. **IRQ safety without `irqsave`**: An atomic operation is inherently IRQ-safe without disabling interrupts — it completes in one hardware instruction.
3. **No deadlock risk**: Atomics can't deadlock.
4. **Compound check-and-act**: `atomic_dec_and_test()` atomically decrements and returns whether the result is zero in one operation — implementing this with a spinlock requires extra care to avoid races between the decrement and the test.

Use `int + spinlock` when you need to protect **multiple related fields** that must change atomically together (the spinlock guards the entire multi-field transaction). For single counters, flags, or reference counts, `atomic_t` is the right tool.

---

**Q6. What is lockdep and what kinds of bugs can it detect?**

**Answer:** lockdep (Lock Dependency Validator, enabled with `CONFIG_PROVE_LOCKING=y`) is a runtime lock correctness checker built into the kernel. It tracks the **lock dependency graph** — every time lock B is acquired while lock A is held, it records the edge A→B. It checks this graph for:

1. **Circular dependencies (deadlocks)**: If A→B and B→A both appear, that's a potential deadlock. lockdep reports this immediately with stack traces showing both orderings.

2. **IRQ-unsafe locking**: If lock A is ever taken in IRQ context without disabling IRQs, and somewhere else it's taken in process context without disabling IRQs, lockdep warns — that sequence can deadlock if an IRQ fires at the wrong moment.

3. **Mutex in atomic context**: Taking a mutex (which may sleep) while preemption is disabled or inside a spinlock critical section.

4. **Lock inversion**: Lock ordering that could cause priority inversion.

The power of lockdep is that it catches these bugs **before the actual deadlock occurs** — it just needs to see both orderings used anywhere in the kernel's execution, not in the exact concurrent scenario. This makes it excellent for catching races in rarely-executed error paths. Always develop with lockdep enabled; disable it in production due to overhead.

---

> **Next:** Day 7 — Interrupt Handling: ISRs, top/bottom halves, threaded IRQs, tasklets, workqueues, and latency measurement with ftrace on RPi4.
