# Day 7 — Kernel Data Structures

> **Estimated read time:** 90–120 minutes  
> **Goal:** Master the fundamental data structures used throughout the kernel: linked lists, red-black trees, hash tables, radix trees, bitmaps, and more.

---

## 1. Why Kernel Data Structures Are Different

Kernel data structures differ from user-space ones in important ways:

1. **Intrusive design**: Instead of `list<struct my_object>`, the kernel embeds the list node *inside* the object. This eliminates separate heap allocation for list nodes and allows an object to be on multiple lists simultaneously.

2. **No generics/templates**: Pure C. Data structures use `void *` or macro tricks.

3. **Lock-free alternatives**: Many structures have RCU-safe variants (`hlist_head`, etc.).

4. **Memory discipline**: Allocations must specify GFP flags. No exceptions.

5. **Cache-awareness**: Structures are often designed to fit in cache lines (64 bytes on x86_64).

---

## 2. Doubly-Linked List (`list_head`)

This is the most pervasive data structure in the kernel. **Everything** uses it.

### 2.1 The Core Structure

```c
// include/linux/list.h

struct list_head {
    struct list_head *next, *prev;
};
```

That's it. Two pointers. The trick is embedding it in your data structure:

```c
struct task_struct {
    // ...
    struct list_head tasks;       // All processes linked here
    struct list_head children;    // List of child processes
    struct list_head sibling;     // Sibling process list
    // ...
};

struct net_device {
    // ...
    struct list_head dev_list;    // All network devices
    // ...
};
```

### 2.2 Getting the Container Object

The magic is `container_of()` — given a pointer to a `list_head`, get the pointer to the enclosing struct:

```c
// include/linux/kernel.h
#define container_of(ptr, type, member) ({          \
    void *__mptr = (void *)(ptr);                   \
    ((type *)(__mptr - offsetof(type, member))); })

// Example:
struct list_head *node = ...; // Some node in the tasks list
struct task_struct *task = list_entry(node, struct task_struct, tasks);
// Equivalent: container_of(node, struct task_struct, tasks)
// = (struct task_struct *)((char *)node - offsetof(struct task_struct, tasks))
```

`offsetof(type, member)` returns the byte offset of `member` within `type`. Subtracting that from the `list_head` pointer gives the start of the containing struct.

### 2.3 List Operations

```c
// Initialization
LIST_HEAD(my_list);                  // Statically declare + init
INIT_LIST_HEAD(&my_list);            // Runtime init of existing list_head

// Insertion
list_add(&node, &head);              // Add after head (stack behavior)
list_add_tail(&node, &head);         // Add before head (queue behavior)

// Removal
list_del(&node);                     // Remove from list
list_del_init(&node);                // Remove + reinitialize (prevents use-after-free bugs)

// Move
list_move(&node, &new_head);         // Remove from old list, add to front of new
list_move_tail(&node, &new_head);    // Remove from old list, add to tail of new

// Splice (merge lists)
list_splice(&list_to_add, &head);    // Insert list_to_add after head

// Check
list_empty(&head)                    // Returns true if list is empty
list_is_last(&node, &head)           // Returns true if node is the last element
```

### 2.4 Iteration

```c
// Basic iteration (read-only):
struct list_head *pos;
list_for_each(pos, &my_list) {
    struct my_struct *item = list_entry(pos, struct my_struct, list);
    // use item
}

// Preferred: direct entry iteration (more readable):
struct my_struct *item;
list_for_each_entry(item, &my_list, list) {
    printk(KERN_INFO "item value: %d\n", item->value);
}

// Safe iteration (can delete while iterating):
struct my_struct *item, *tmp;
list_for_each_entry_safe(item, tmp, &my_list, list) {
    if (should_delete(item)) {
        list_del(&item->list);
        kfree(item);
    }
}

// Reverse iteration:
list_for_each_entry_reverse(item, &my_list, list) { ... }

// From a given position:
list_for_each_entry_continue(item, &my_list, list) { ... }
```

### 2.5 Circular List Trick

A `list_head` is circular — the last element's `next` points to the head, and the head's `prev` points to the last element. The head itself is a sentinel node (not an actual item). This allows O(1) `list_add_tail()` and checking if the list is empty.

```
HEAD ←→ item1 ←→ item2 ←→ item3
 ↑________________________________↓  (circular)
```

---

## 3. Singly-Linked List with Hash (`hlist_head`)

For hash tables, a doubly-linked list's `prev` pointer in the head wastes space. The kernel uses `hlist_head`/`hlist_node`:

```c
// include/linux/list.h
struct hlist_head {
    struct hlist_node *first;
};

struct hlist_node {
    struct hlist_node *next;
    struct hlist_node **pprev;  // pointer to pointer (saves memory vs full doubly-linked)
};
```

```c
// Declaration:
struct hlist_head bucket[HASH_SIZE];

// Initialize:
INIT_HLIST_HEAD(&bucket[i]);

// Add at head:
hlist_add_head(&node, &bucket[hash]);

// Iterate:
struct hlist_node *pos;
hlist_for_each(pos, &bucket[hash]) {
    struct my_struct *item = hlist_entry(pos, struct my_struct, node);
}

// Preferred:
struct my_struct *item;
hlist_for_each_entry(item, &bucket[hash], node) { ... }

// Safe (for deletion during iteration):
hlist_for_each_entry_safe(item, tmp, &bucket[hash], node) { ... }
```

The `hlist` family has RCU-safe variants used throughout networking:
- `hlist_for_each_entry_rcu()`
- `hlist_add_head_rcu()`

---

## 4. Red-Black Tree (`rbtree`)

Used where O(log n) ordered lookup/insertion is needed:
- Process scheduling (CFS run queue — ordered by `vruntime`)
- Memory management (VMAs ordered by start address)
- Timer management
- Page cache lookup

### 4.1 Structure

```c
// include/linux/rbtree.h
struct rb_node {
    unsigned long  __rb_parent_color;  // Parent pointer + color bit (LSB = color)
    struct rb_node *rb_right;
    struct rb_node *rb_left;
};

struct rb_root {
    struct rb_node *rb_node;  // Root of the tree
};
```

Like `list_head`, embed `rb_node` in your struct:

```c
struct interval {
    struct rb_node rb;   // Embedded node
    unsigned long start;
    unsigned long end;
    // ... your data ...
};
```

### 4.2 Using the Red-Black Tree

The kernel's rbtree doesn't provide automatic key comparison — you must implement your own insert/search:

```c
// Insertion (must maintain BST property yourself):
static void interval_insert(struct rb_root *root, struct interval *new_iv)
{
    struct rb_node **link = &root->rb_node;
    struct rb_node *parent = NULL;
    
    while (*link) {
        struct interval *this = rb_entry(*link, struct interval, rb);
        parent = *link;
        
        if (new_iv->start < this->start)
            link = &(*link)->rb_left;
        else if (new_iv->start > this->start)
            link = &(*link)->rb_right;
        else
            return;  // duplicate key — handle as needed
    }
    
    // Link the new node
    rb_link_node(&new_iv->rb, parent, link);
    // Re-balance (the kernel's rb_insert_color does this):
    rb_insert_color(&new_iv->rb, root);
}

// Search:
static struct interval *interval_find(struct rb_root *root, unsigned long start)
{
    struct rb_node *node = root->rb_node;
    
    while (node) {
        struct interval *iv = rb_entry(node, struct interval, rb);
        
        if (start < iv->start)
            node = node->rb_left;
        else if (start > iv->start)
            node = node->rb_right;
        else
            return iv;  // Found!
    }
    return NULL;
}

// Deletion:
rb_erase(&iv->rb, &my_rb_root);  // O(log n) delete + rebalance

// Traversal (in-order = sorted order):
struct rb_node *node;
for (node = rb_first(&my_rb_root); node; node = rb_next(node)) {
    struct interval *iv = rb_entry(node, struct interval, rb);
    // process iv
}
```

### 4.3 CFS Scheduler's Use

```c
// kernel/sched/fair.c
struct cfs_rq {
    struct rb_root_cached tasks_timeline;  // Red-black tree of tasks by vruntime
    // ...
};

// CFS always picks the leftmost node (min vruntime) to run next:
struct sched_entity *se = __pick_first_entity(cfs_rq);
// = rb_entry(cfs_rq->tasks_timeline.rb_leftmost, struct sched_entity, run_node)
```

`rb_root_cached` is an optimization that caches the leftmost (minimum) node, making `rb_first()` O(1) instead of O(log n). CFS needs this for the hot scheduling path.

---

## 5. Hash Tables

### 5.1 `DECLARE_HASHTABLE` — Kernel Hash Tables

```c
#include <linux/hashtable.h>

// Declare a hash table with 2^10 = 1024 buckets:
DECLARE_HASHTABLE(pid_table, 10);
// Equivalent to: struct hlist_head pid_table[1 << 10];

// Initialize:
hash_init(pid_table);

// Add an element (using its key):
struct my_entry {
    int pid;
    char name[16];
    struct hlist_node node;
};

struct my_entry *e = kzalloc(sizeof(*e), GFP_KERNEL);
e->pid = 1234;
hash_add(pid_table, &e->node, e->pid);  // Key = pid

// Lookup:
struct my_entry *found;
hash_for_each_possible(pid_table, found, node, 1234) {
    if (found->pid == 1234) {
        // Got it!
        break;
    }
}

// Delete:
hash_del(&e->node);

// Iterate ALL entries:
int bkt;
struct my_entry *item;
hash_for_each(pid_table, bkt, item, node) {
    printk("pid=%d name=%s\n", item->pid, item->name);
}
```

The hash function: `hash_add()` uses `hash_min(key, HASH_BITS(ht))` which applies a multiplicative hash (Knuth's algorithm).

### 5.2 The PID Hash Table (Kernel Example)

```c
// kernel/pid.c — real kernel PID lookup
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
    struct pid *pid;
    
    hlist_for_each_entry_rcu(pid,
            &ns->idr.idr_rt /* simplified */,
            tasks[PIDTYPE_PID]) {
        if (pid->numbers[ns->level].nr == nr)
            return pid;
    }
    return NULL;
}
```

---

## 6. XArray (Replacing Radix Trees)

The XArray (`include/linux/xarray.h`) replaced the old radix tree in Linux 4.20. It's a sparse array indexed by unsigned long, optimized for:
- Page cache (pages indexed by file offset in pages)
- Buffer cache
- IDR (integer-to-pointer mapping, e.g., file descriptor table)

### 6.1 Basic Usage

```c
#include <linux/xarray.h>

// Declare and initialize:
DEFINE_XARRAY(my_array);                    // Static
struct xarray my_xa;                        // Dynamic
xa_init(&my_xa);                            // Initialize dynamic

// Store (key = unsigned long, value = pointer):
void *old = xa_store(&my_xa, key, ptr, GFP_KERNEL);
if (xa_is_err(old)) {
    int err = xa_err(old);
    // handle error
}

// Load:
void *val = xa_load(&my_xa, key);
if (val) {
    // found
}

// Delete:
xa_erase(&my_xa, key);

// Iterate:
unsigned long index;
void *entry;
xa_for_each(&my_xa, index, entry) {
    // process entry at index
}

// Range iteration:
xa_for_each_range(&my_xa, index, entry, start, end) { ... }

// Destroy:
xa_destroy(&my_xa);   // Frees all stored entries' slots (NOT the pointed-to objects)
```

### 6.2 Page Cache Usage

```c
// mm/filemap.c — page cache lookup
struct page *find_get_page(struct address_space *mapping, pgoff_t offset)
{
    struct page *page;
    
    rcu_read_lock();
    page = xa_load(&mapping->i_pages, offset);  // O(log n) or O(1) amortized
    // ...
    rcu_read_unlock();
    
    return page;
}
```

---

## 7. Bitmaps

Bitmaps are used for CPU masks, IRQ masks, PID allocation, and more.

```c
#include <linux/bitmap.h>

// Declare:
DECLARE_BITMAP(my_bitmap, NR_CPUS);  // Static
unsigned long *bm = bitmap_zalloc(nr_bits, GFP_KERNEL);  // Dynamic

// Operations:
bitmap_zero(my_bitmap, NR_CPUS);         // Clear all
bitmap_fill(my_bitmap, NR_CPUS);         // Set all

set_bit(3, my_bitmap);                    // Set bit 3
clear_bit(3, my_bitmap);                  // Clear bit 3
test_bit(3, my_bitmap);                   // Test bit 3 (returns 0 or 1)
test_and_set_bit(3, my_bitmap);           // Atomic test+set, returns old value
test_and_clear_bit(3, my_bitmap);         // Atomic test+clear

// Find bits:
find_first_zero_bit(my_bitmap, NR_CPUS);  // First 0 bit position
find_first_bit(my_bitmap, NR_CPUS);       // First 1 bit position
find_next_zero_bit(my_bitmap, NR_CPUS, start);  // Next 0 from start
find_next_bit(my_bitmap, NR_CPUS, start);       // Next 1 from start

// Logical operations:
bitmap_and(dst, src1, src2, nbits);       // dst = src1 & src2
bitmap_or(dst, src1, src2, nbits);        // dst = src1 | src2
bitmap_andnot(dst, src1, src2, nbits);    // dst = src1 & ~src2

// Count:
bitmap_weight(my_bitmap, NR_CPUS);       // Number of set bits

// Iterate set bits:
unsigned int cpu;
for_each_set_bit(cpu, my_bitmap, NR_CPUS) {
    pr_info("cpu %d is set\n", cpu);
}
```

### 7.1 CPU Masks

CPU masks are bitmaps with special semantics:

```c
#include <linux/cpumask.h>

cpumask_t mask;
cpumask_t *pmask = cpu_online_mask;  // Which CPUs are online

// Operations:
cpumask_set_cpu(0, &mask);
cpumask_clear_cpu(0, &mask);
cpumask_test_cpu(0, &mask);

// Predefined masks:
cpu_online_mask       // Currently online CPUs
cpu_possible_mask     // CPUs that could ever be online
cpu_present_mask      // CPUs physically present
cpu_active_mask       // CPUs available for scheduling

// Iterate:
int cpu;
for_each_online_cpu(cpu) {
    // Do something on each online CPU
}
for_each_possible_cpu(cpu) { ... }
for_each_cpu(cpu, &mask) { ... }     // Iterate specific mask
```

---

## 8. IDR — Integer-to-Pointer Mapping

`idr` provides a mechanism to allocate unique IDs and map them to pointers. Used for: file descriptors, inode numbers, process IDs.

```c
#include <linux/idr.h>

// Declaration and init:
DEFINE_IDR(my_idr);           // Static
struct idr my_idr;            // Dynamic
idr_init(&my_idr);

// Allocate an ID and associate a pointer:
int id = idr_alloc(&my_idr, my_object, min_id, max_id, GFP_KERNEL);
// id >= 0 on success, negative errno on error

// Find by ID:
void *obj = idr_find(&my_idr, id);

// Remove:
idr_remove(&my_idr, id);

// Iterate:
int id;
void *obj;
idr_for_each_entry(&my_idr, obj, id) {
    printk("id=%d obj=%p\n", id, obj);
}

// Destroy:
idr_destroy(&my_idr);
```

The file descriptor table is essentially an XArray (newer kernels) or IDR:

```c
// fs/file.c
int __alloc_fd(struct files_struct *files, unsigned start, unsigned end, unsigned flags)
{
    // Find an unused FD number starting from 'start'
    fd = find_next_zero_bit(files->open_fds, end, start);
    // ...
    __set_open_fd(fd, fdt);  // Mark as allocated in bitmap
    // ...
}
```

---

## 9. Circular Buffer / Ring Buffer

Used for: `dmesg` ring buffer, networking ring buffers, perf ring buffer, `io_uring` rings.

### 9.1 The `kfifo` Interface

```c
#include <linux/kfifo.h>

// Static declaration (size must be power of 2):
DECLARE_KFIFO(my_fifo, unsigned int, 128);  // 128-entry fifo of unsigned ints
INIT_KFIFO(my_fifo);

// Dynamic allocation:
struct kfifo my_fifo;
kfifo_alloc(&my_fifo, PAGE_SIZE, GFP_KERNEL);

// Put data:
kfifo_put(&my_fifo, value);          // Put single item
kfifo_in(&my_fifo, buf, count);      // Put array of items

// Get data:
unsigned int val;
kfifo_get(&my_fifo, &val);           // Get single item
kfifo_out(&my_fifo, buf, count);     // Get array of items

// Peek (don't consume):
kfifo_peek(&my_fifo, &val);

// Status:
kfifo_is_empty(&my_fifo)
kfifo_is_full(&my_fifo)
kfifo_len(&my_fifo)                  // Number of items currently in fifo
kfifo_avail(&my_fifo)                // Space available

// Free:
kfifo_free(&my_fifo);
```

### 9.2 Lock-Free Ring Buffer Principle

The kernel's `kfifo` is safe for single-producer/single-consumer without locking, using the property that:
- The head index is only modified by the producer
- The tail index is only modified by the consumer
- With proper memory barriers, no locking is needed

```c
// Conceptual implementation:
struct kfifo {
    unsigned int in;   // head (producer writes here)
    unsigned int out;  // tail (consumer reads here)
    unsigned int mask; // size - 1 (size must be power of 2)
    void *data;
};

// Producer:
void kfifo_in(struct kfifo *fifo, const void *buf, unsigned int len) {
    unsigned int l = fifo->mask + 1 - (fifo->in - fifo->out);  // space
    memcpy(fifo->data + (fifo->in & fifo->mask), buf, len);
    smp_wmb();         // Write barrier: ensure data is visible before updating in
    fifo->in += len;
}

// Consumer:
unsigned int kfifo_out(struct kfifo *fifo, void *buf, unsigned int len) {
    unsigned int l = fifo->in - fifo->out;  // used space
    smp_rmb();         // Read barrier: ensure we read in before reading data
    memcpy(buf, fifo->data + (fifo->out & fifo->mask), len);
    fifo->out += len;
    return len;
}
```

---

## 10. Per-CPU Variables

Per-CPU variables avoid cache bouncing on SMP systems by giving each CPU its own copy of data.

```c
#include <linux/percpu.h>

// Declare:
DEFINE_PER_CPU(int, my_counter);               // Per-CPU int
DEFINE_PER_CPU(struct stats, per_cpu_stats);   // Per-CPU struct

// Access (must disable preemption to prevent moving to another CPU mid-access):
// get_cpu() disables preemption and returns current CPU number:
int cpu = get_cpu();
per_cpu(my_counter, cpu)++;
put_cpu();  // Re-enable preemption

// Short forms (safe only if you don't need the CPU number):
this_cpu_inc(my_counter);           // Preempt-safe increment
this_cpu_add(my_counter, 5);        // Add 5
this_cpu_read(my_counter);          // Read current CPU's value
this_cpu_write(my_counter, val);    // Write

// For read-modify-write atomics:
this_cpu_cmpxchg(my_counter, old, new);

// Read ANY CPU's value:
per_cpu(my_counter, cpu_id);

// Iterate over all CPUs:
int cpu;
for_each_possible_cpu(cpu) {
    printk("cpu %d: %d\n", cpu, per_cpu(my_counter, cpu));
}

// Sum across all CPUs:
int total = 0;
for_each_possible_cpu(cpu)
    total += per_cpu(my_counter, cpu);
```

Per-CPU variables are heavily used in:
- Statistics counters (avoid lock contention by counting per-CPU, summing on read)
- The scheduler's run queues (one per CPU)
- Memory allocator (per-CPU slab caches)
- Network packet counters

---

## 11. Seqlock for Read-Heavy Data

A seqlock allows fast parallel reads and exclusive writes. Perfect for time values that are updated frequently but read even more frequently:

```c
#include <linux/seqlock.h>

// Declaration:
seqlock_t my_seqlock;
seqlock_init(&my_seqlock);
// Or static: DEFINE_SEQLOCK(my_seqlock);

// Write path (exclusive):
write_seqlock(&my_seqlock);
my_data.x = new_x;
my_data.y = new_y;
write_sequnlock(&my_seqlock);

// Read path (retry if writer was active):
unsigned int seq;
do {
    seq = read_seqbegin(&my_seqlock);
    // Read data (may be inconsistent if writer ran concurrently!)
    x = my_data.x;
    y = my_data.y;
} while (read_seqretry(&my_seqlock, seq));
// After the loop, x and y are consistent

// How it works:
// write_seqlock: increments the sequence counter (makes it odd)
// write_sequnlock: increments again (makes it even)
// read_seqbegin: reads the counter (if odd, writer is active — loop)
// read_seqretry: reads again; if changed since begin, retry
```

The kernel uses seqlocks for timekeeping (`timekeeper_seqcount`), so reading `clock_gettime()` via vDSO is always consistent.

---

## 12. Radix Tree / Page Cache Lookup

The old radix tree (`lib/radix-tree.c`) is now replaced by XArray but understanding it helps reading older kernel versions:

```c
// include/linux/radix-tree.h (still present for compat)
struct radix_tree_root root;
INIT_RADIX_TREE(&root, GFP_KERNEL);

radix_tree_insert(&root, index, ptr);
void *found = radix_tree_lookup(&root, index);
radix_tree_delete(&root, index);
```

The page cache stores pages indexed by `(inode, page_offset)`. Before XArray:
```c
// mm/filemap.c (older kernels)
struct page *find_get_page(struct address_space *mapping, pgoff_t offset)
{
    return radix_tree_lookup(&mapping->page_tree, offset);
}
```

---

## 13. Priority Queues and Heaps

The kernel doesn't have a generic heap, but heaps appear implicitly:

```c
// Timer heap: kernel's timer implementation uses a per-CPU timer wheel
// (not a heap) for coarse timers, and an rbtree for hrtimers:

// kernel/time/hrtimer.c
struct hrtimer_clock_base {
    struct timerqueue_head  active;     // rbtree of pending hrtimers
    // ...
};

// Finding the next timer = leftmost rbtree node = O(1) with rb_root_cached
struct hrtimer *hrtimer_get_next_event(void)
{
    return timerqueue_getnext(&base->active);  // rb_leftmost
}
```

---

## 14. Common Patterns: Object Lifecycle with Lists

A typical kernel subsystem manages objects on multiple lists:

```c
struct myobj {
    struct list_head global_list;   // On global all-objects list
    struct list_head active_list;   // On "active objects" list
    struct hlist_node hash_node;    // In hash table for fast lookup
    struct rb_node rb_node;         // In rbtree for ordered access
    
    int id;
    int state;
    spinlock_t lock;               // Protects this object
    struct kref refcount;          // Reference counting (see below)
};

// Global lists/structures:
static LIST_HEAD(all_objects);
static LIST_HEAD(active_objects);
static DECLARE_HASHTABLE(object_hash, 8);   // 256 buckets
static struct rb_root object_tree = RB_ROOT;
static DEFINE_SPINLOCK(global_lock);

// Adding a new object:
struct myobj *obj = kzalloc(sizeof(*obj), GFP_KERNEL);
obj->id = allocate_id();
obj->state = STATE_IDLE;
spin_lock_init(&obj->lock);
kref_init(&obj->refcount);

spin_lock(&global_lock);
list_add_tail(&obj->global_list, &all_objects);
hash_add(object_hash, &obj->hash_node, obj->id);
myobj_insert_rb(&object_tree, obj);
spin_unlock(&global_lock);

// Lookup by ID:
static struct myobj *find_by_id(int id)
{
    struct myobj *obj;
    hash_for_each_possible(object_hash, obj, hash_node, id) {
        if (obj->id == id)
            return obj;
    }
    return NULL;
}
```

---

## 15. Reference Counting with `kref`

`kref` provides safe reference counting for kernel objects:

```c
#include <linux/kref.h>

struct myobj {
    struct kref refcount;
    // ...
};

// Initialization:
kref_init(&obj->refcount);         // Set refcount to 1

// Increment:
kref_get(&obj->refcount);          // +1

// Decrement and release if zero:
static void myobj_release(struct kref *ref)
{
    struct myobj *obj = container_of(ref, struct myobj, refcount);
    // Remove from lists, free resources:
    list_del(&obj->global_list);
    kfree(obj);
}

kref_put(&obj->refcount, myobj_release);  // -1, calls release if hits 0

// Example lifecycle:
struct myobj *obj = create_myobj();  // kref_init() → refcount = 1
kref_get(&obj->refcount);           // Someone takes a reference → 2
// ... use obj ...
kref_put(&obj->refcount, myobj_release);  // → 1
// ... use obj ...
kref_put(&obj->refcount, myobj_release);  // → 0, myobj_release() called, obj freed
```

---

## 16. Comparing Data Structure Performance

| Structure | Lookup | Insert | Delete | Ordered? | Space |
|-----------|--------|--------|--------|----------|-------|
| `list_head` | O(n) | O(1) | O(1)* | No | 2 ptrs |
| `hlist_head` | O(n/buckets) | O(1) | O(1)* | No | 1 ptr (head) |
| `rbtree` | O(log n) | O(log n) | O(log n) | Yes | 3 ptrs + color |
| `hashtable` | O(1) avg | O(1) | O(1)* | No | 1 ptr/bucket |
| `xarray` | O(log n) | O(log n) | O(log n) | Yes (sparse) | 64B min |
| Per-CPU var | O(1) | N/A | N/A | No | size × ncpus |

*O(1) given pointer to the node; O(n) to find the node first.

---

## 17. Real Kernel Code Examples

### 17.1 Process List (list_head)

```c
// Iterate over ALL processes in the system:
struct task_struct *task;
for_each_process(task) {  // Macro using list_for_each_entry
    printk(KERN_INFO "pid=%d name=%s\n", task->pid, task->comm);
}

// Macro definition:
#define for_each_process(p) \
    for (p = &init_task; (p = next_task(p)) != &init_task; )
```

### 17.2 VMA Lookup (rbtree)

```c
// mm/mmap.c — find VMA containing an address:
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
    struct rb_node *rb_node;
    struct vm_area_struct *vma;
    
    rb_node = mm->mm_rb.rb_node;  // Root of VMA rbtree
    while (rb_node) {
        vma = rb_entry(rb_node, struct vm_area_struct, vm_rb);
        if (vma->vm_end > addr) {
            if (vma->vm_start <= addr)
                return vma;  // Found!
            rb_node = rb_node->rb_left;
        } else {
            rb_node = rb_node->rb_right;
        }
    }
    return NULL;
}
```

### 17.3 Network Neighbor Table (hash + rcu)

```c
// net/core/neighbour.c — ARP table lookup:
struct neighbour *neigh_lookup(struct neigh_table *tbl,
                               const void *pkey, struct net_device *dev)
{
    struct neighbour *n;
    u32 hash = tbl->hash(pkey, dev, NULL);
    
    rcu_read_lock_bh();
    n = __neigh_lookup_noref(tbl, pkey, dev, hash);
    if (n)
        refcount_inc(&n->refcnt);  // Hold reference
    rcu_read_unlock_bh();
    
    return n;
}
```

---

## 18. Mental Model Checkpoint

After Day 7, you should be able to:

1. Use `list_head` to embed a linked list in your struct and iterate it with `list_for_each_entry`.
2. Implement insertion and search in a red-black tree (`rb_node`).
3. Use `DECLARE_HASHTABLE` and perform operations on it.
4. Explain when to use list vs rbtree vs hashtable (tradeoffs).
5. Use per-CPU variables correctly and explain why they avoid cache bouncing.
6. Use `kref` for safe reference counting.
7. Use `kfifo` for a lock-free ring buffer.
8. Read any kernel code using these structures and understand what it's doing.

---

## Key Source Files

```bash
include/linux/list.h        # list_head, hlist_head, all list operations
include/linux/rbtree.h      # rb_node, rb_root, insert/delete/iterate
include/linux/hashtable.h   # DECLARE_HASHTABLE, hash_add, hash_for_each
include/linux/xarray.h      # XArray: the modern sparse array
include/linux/percpu.h      # Per-CPU variables
include/linux/kfifo.h       # Ring buffer
include/linux/idr.h         # Integer-to-pointer maps
include/linux/kref.h        # Reference counting
include/linux/bitmap.h      # Bitmap operations
include/linux/cpumask.h     # CPU masks
lib/rbtree.c                # Red-black tree implementation
lib/xarray.c                # XArray implementation
```

---

## Summary

The kernel's data structures share a common philosophy: **intrusive design** — nodes embed into objects instead of wrapping them. This allows objects to be on multiple data structures simultaneously and eliminates separate allocations.

The most important structures in order of frequency:
1. `list_head` — used everywhere; O(n) lookup, O(1) insert/delete given pointer
2. `hlist_head` — hash buckets; same as list but memory-optimized for heads
3. `rb_node` — where O(log n) ordered access is needed (scheduler, VMAs, timers)
4. `xarray` / `hashtable` — for indexed/keyed lookup
5. Per-CPU variables — for scalable counters and per-core state

Tomorrow: kernel memory layout — understanding the virtual address space and how the kernel organizes its own memory.
