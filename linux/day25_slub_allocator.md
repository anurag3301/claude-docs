# Day 25 — Slab/SLUB Allocator

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand how `kmalloc` works, how SLUB organizes object caches, per-CPU freelists, slab pages, and memory debugging with SLUB.

---

## 1. Why a Slab Allocator?

The buddy allocator deals in pages (4KB minimum). But most kernel allocations are tiny:
- `struct task_struct`: ~7KB (close to 2 pages)  
- `struct file`: 232 bytes
- `struct sk_buff`: 240 bytes
- `struct dentry`: 192 bytes
- Typical driver allocation: 64–512 bytes

Without a slab allocator, allocating 240 bytes would waste most of a 4KB page. Worse: 240 bytes doesn't divide into 4KB evenly → internal fragmentation.

The slab allocator solves this by:
1. **Object caching**: maintaining pools of pre-allocated, same-size objects
2. **Slab pages**: packing multiple objects into a single page
3. **Per-CPU caches**: fast allocation without locks for the common case
4. **Constructor caches**: objects can be pre-initialized

---

## 2. Three Implementations: SLAB, SLUB, SLOB

```bash
# Modern kernels default to SLUB:
cat /boot/config-$(uname -r) | grep -E 'CONFIG_SLUB|CONFIG_SLAB'
# CONFIG_SLUB=y    ← SLUB: the default since Linux 2.6.23

# SLUB is simpler and faster than the original SLAB:
# - Fewer data structures
# - Per-CPU freelists instead of SLAB's cpu_cache
# - Better debugging facilities
# - Less metadata overhead

# SLOB: tiny allocator for embedded systems (rarely used)
# CONFIG_SLOB=y
```

---

## 3. SLUB Architecture Overview

```
kmem_cache (one per object type):
  name, object_size, align, flags
  │
  ├── kmem_cache_cpu (per-CPU, FAST PATH):
  │     freelist: pointer to next free object
  │     page: current active slab page
  │     partial: list of partially filled pages
  │
  └── kmem_cache_node (per-NUMA-node, SLOW PATH):
        partial: list of partially filled slab pages
        nr_partial: count of partial slabs
```

### 3.1 `struct kmem_cache`

```c
// include/linux/slub_def.h
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab; // Per-CPU freelists (fast path)
    
    slab_flags_t flags;        // SLAB_HWCACHE_ALIGN, SLAB_POISON, etc.
    unsigned long min_partial; // Minimum partial slabs to keep
    unsigned int size;         // Size of object including metadata
    unsigned int object_size;  // Real object size (no padding)
    unsigned int offset;       // Offset of free pointer within object
    struct kmem_cache_order_objects oo; // pages per slab + objects per slab
    
    struct kmem_cache_order_objects max; // Max order we try
    struct kmem_cache_order_objects min; // Min order we try
    
    gfp_t allocflags;         // GFP flags for slab page allocation
    int refcount;             // Reference count for cache reuse
    
    void (*ctor)(void *);     // Constructor (optional)
    
    unsigned int inuse;       // Offset of metadata after object
    unsigned int align;       // Required alignment
    
    const char *name;         // Name shown in /proc/slabinfo
    struct list_head list;    // All kmem_caches are linked here
    
    struct kmem_cache_node *node[MAX_NUMNODES]; // Per-node partial lists
};

struct kmem_cache_cpu {
    void **freelist;          // Pointer to first free object in slab
                              // (Each free object's first word points to next)
    unsigned long tid;        // Transaction ID (for ABA protection)
    struct slab *slab;        // Current slab page
    struct slab *partial;     // Partially used slabs (local list)
};
```

---

## 4. The Slab Page

A "slab" is a set of pages holding objects of the same size. In SLUB, the slab metadata is stored in the `struct slab` which overlays `struct page`:

```c
// mm/slab.h
struct slab {
    unsigned long __page_flags;  // Same as struct page flags
    
    struct kmem_cache *slab_cache;  // Which cache owns this slab
    
    union {
        struct {
            union {
                struct list_head slab_list;  // In partial/full list
                struct {
                    struct slab *next;
                    int slabs;  // Number of slabs left in list
                };
            };
            void *freelist;    // First free object in this slab
            union {
                unsigned long counters;
                struct {
                    unsigned inuse:16;      // Objects currently used
                    unsigned objects:15;    // Total objects
                    unsigned frozen:1;      // Frozen (in cpu cache)
                };
            };
        };
    };
};
```

### 4.1 Free Object Linked List

Objects within a slab are linked using the objects themselves (intrusive free list):

```
Slab page (4KB), object size = 64 bytes → 64 objects:

[obj0 | next→] [obj1 | next→] [obj2 | NULL] ... [obj63]
  ↑
freelist → obj0 → obj1 → obj2 → NULL (end of free list)

After allocating obj0:
freelist → obj1 → obj2 → NULL

After allocating obj1:
freelist → obj2 → NULL

After allocating obj2 (slab full):
freelist = NULL → this slab is moved to "full" list
```

The "next" pointer is stored at `offset` bytes within the free object (computed at cache creation time based on alignment).

---

## 5. `kmalloc` — The Generic Interface

```c
// include/linux/slab.h
void *kmalloc(size_t size, gfp_t flags);
void *kzalloc(size_t size, gfp_t flags);  // kmalloc + zero fill
void *kmalloc_array(size_t n, size_t size, gfp_t flags);  // n*size, overflow check
void *kcalloc(size_t n, size_t size, gfp_t flags);        // kmalloc_array + zero
void kfree(const void *objp);

void *krealloc(const void *p, size_t new_size, gfp_t flags);
size_t ksize(const void *objp);  // Get actual allocation size (may be > requested)
```

`kmalloc` uses a set of generic caches sized at powers of 2:

```bash
# The kmalloc caches:
cat /proc/slabinfo | grep 'kmalloc-'
# Name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
# kmalloc-8              12345     16384          8         512           1
# kmalloc-16              5678      8192         16         256           1
# kmalloc-32              3456      4096         32         128           1
# kmalloc-64              1234      2048         64          64           1
# kmalloc-128              567      1024        128          32           1
# kmalloc-256              234       512        256          16           1
# kmalloc-512               89       256        512           8           1
# kmalloc-1024              45       128       1024           4           1
# kmalloc-2048              23        64       2048           2           1
# kmalloc-4096              12        32       4096           1           1
# kmalloc-8192               6        16       8192           2           2  ← 2 pages/slab

# kmalloc selects the smallest cache >= requested size
# kmalloc(200, GFP_KERNEL) → uses kmalloc-256 cache (wastes 56 bytes)
```

```c
// How kmalloc selects the right cache:
// mm/slab_common.c
void *kmalloc(size_t size, gfp_t flags)
{
    if (unlikely(size > KMALLOC_MAX_CACHE_SIZE)) {
        // Large allocation: go directly to buddy allocator
        return kmalloc_large(size, flags);
    }
    
    // Select cache by size:
    unsigned int index = kmalloc_index(size);  // fls(size - 1)
    struct kmem_cache *cache = kmalloc_caches[flags_to_allocator][index];
    
    return kmem_cache_alloc(cache, flags);
}

// KMALLOC_MAX_CACHE_SIZE = 8KB (2 pages)
// Larger than 8KB → alloc_pages() directly
```

---

## 6. SLUB Fast Path: Per-CPU Freelist

```c
// mm/slub.c
static __always_inline void *slab_alloc_node(struct kmem_cache *s, ...)
{
    void *object;
    struct kmem_cache_cpu *c;
    
    // Disable preemption (critical: we're accessing per-CPU data):
    c = raw_cpu_ptr(s->cpu_slab);
    
    // FAST PATH: take from per-CPU freelist (no lock needed!):
    object = c->freelist;
    if (likely(object && slab_on_cpu(c->slab))) {
        // Advance freelist pointer (next free object):
        c->freelist = get_freepointer_safe(s, object);
        c->tid = next_tid(c->tid);  // Update transaction ID
        
        preempt_enable();
        
        // Zero-fill if requested:
        if (unlikely(slab_want_init_on_alloc(gfpflags, s)))
            memset(object, 0, s->object_size);
        
        return object;  // Done! No locks taken.
    }
    
    // SLOW PATH: per-CPU freelist empty or wrong CPU:
    return __slab_alloc(s, gfpflags, node, addr, c, object);
}
```

The fast path is just:
1. Read `c->freelist` (pointer to first free object)
2. Update `c->freelist` to point to next object
3. Return the object

No locks, no atomics — pure pointer manipulation.

### 6.1 Slow Path: `__slab_alloc()`

```c
// When per-CPU freelist is empty:
static void *__slab_alloc(struct kmem_cache *s, gfp_t gfpflags, int node, ...)
{
    struct kmem_cache_cpu *c = this_cpu_ptr(s->cpu_slab);
    struct slab *slab;
    
    // 1. Try partial slab from per-CPU partial list:
    slab = c->partial;
    if (slab) {
        // Move to active slab, replenish freelist:
        goto redo;
    }
    
    // 2. Try partial slab from node list (slower, needs lock):
    slab = get_partial(s, node, ...);
    if (slab)
        goto redo;
    
    // 3. Allocate new slab from buddy allocator:
    slab = new_slab(s, gfpflags, node);
    if (slab) {
        c->slab = slab;
        c->freelist = slab->freelist;
        slab->freelist = NULL;
        slab->frozen = 1;  // Frozen = owned by this CPU's cache
        goto redo;
    }
    
    // 4. Out of memory:
    return NULL;
    
redo:
    // Take an object from c->freelist:
    object = c->freelist;
    c->freelist = get_freepointer(s, object);
    return object;
}
```

---

## 7. `kfree` Path

```c
// mm/slub.c
void kfree(const void *x)
{
    struct slab *slab;
    void *object = (void *)x;
    
    // Find the slab this object belongs to (via page flags):
    slab = virt_to_slab(x);
    if (unlikely(!slab)) {
        // Not a slab allocation: could be from buddy allocator (large kmalloc)
        free_large_kmalloc(virt_to_folio(x), object);
        return;
    }
    
    kmem_cache_free(slab->slab_cache, object);
}

void kmem_cache_free(struct kmem_cache *s, void *x)
{
    struct kmem_cache_cpu *c;
    struct slab *slab;
    
    slab = virt_to_slab(x);  // Get the slab page
    
    // Fast path: return to per-CPU freelist:
    c = raw_cpu_ptr(s->cpu_slab);
    if (likely(c->slab == slab && !c->slab->frozen)) {
        // Put object at head of per-CPU freelist:
        set_freepointer(s, x, c->freelist);
        c->freelist = x;
        c->tid = next_tid(c->tid);
        return;
    }
    
    // Slow path: cross-CPU free or slab management needed:
    __slab_free(s, slab, x, ...);
}
```

---

## 8. Creating Custom Caches

For frequently allocated objects, create a dedicated cache:

```c
// Initialization (module init or subsystem init):
struct kmem_cache *my_cache;

my_cache = kmem_cache_create(
    "my_object",             // Name in /proc/slabinfo
    sizeof(struct my_obj),   // Object size
    __alignof__(struct my_obj), // Alignment (0 = default)
    SLAB_HWCACHE_ALIGN |     // Align to cache line
    SLAB_RECLAIM_ACCOUNT |   // Account for reclaim statistics
    SLAB_MEM_SPREAD,         // Spread across NUMA nodes
    NULL                     // Constructor (NULL = no init)
);
if (!my_cache)
    return -ENOMEM;

// Allocate:
struct my_obj *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
if (!obj) {
    err = -ENOMEM;
    goto out;
}

// Free:
kmem_cache_free(my_cache, obj);

// Destroy cache (all objects must be freed first!):
kmem_cache_destroy(my_cache);
```

### 8.1 `SLAB_*` Flags

```c
SLAB_HWCACHE_ALIGN     // Align objects to CPU cache line size (64 bytes)
                       // Prevents false sharing between adjacent objects

SLAB_RECLAIM_ACCOUNT   // Account these pages to reclaimable memory category
                       // Affects /proc/meminfo SReclaimable

SLAB_PANIC             // Panic if cache creation fails (critical caches only)

SLAB_ACCOUNT           // Charge to memcg

SLAB_MEM_SPREAD        // Spread pages across NUMA nodes

SLAB_NOLEAKTRACE       // Don't track this cache in kmemleak

// DEBUG flags (only in debug builds):
SLAB_POISON            // Fill freed objects with 0x6B (POISON_FREE)
                       // Fill allocated objects with 0x5A (POISON_INUSE)
                       // Detects use-after-free

SLAB_RED_ZONE          // Add red zones before/after object
                       // Detects out-of-bounds writes

SLAB_STORE_USER        // Store the last allocator's address
                       // (who called kmem_cache_alloc)

SLAB_TRACE             // Trace all allocations from this cache
```

---

## 9. SLUB Debugging Features

### 9.1 Poison Values

```c
// When SLAB_POISON is set (or SLUB_DEBUG_ON=y):
// After free: object filled with 0x6B6B6B6B...
// Before alloc: verified to still be 0x6B (use-after-free detection)
// After alloc: filled with 0x5A5A5A5A... until zeroed by user

// Poison values defined in include/linux/poison.h:
#define POISON_INUSE     0x5a  // Object in use (freshly allocated)
#define POISON_FREE      0x6b  // Object is free
#define POISON_END       0xa5  // End of object marker
```

```bash
# Enable SLUB debugging for all caches:
# Boot with: slub_debug=FPZU
# F = sanity checks
# P = poisoning
# Z = red zones
# U = user tracking (store allocator address)

# Enable for specific cache only:
# Boot with: slub_debug=P,dentry    (poison only for dentry cache)

# Enable at runtime (if CONFIG_SLUB_DEBUG=y):
echo FPZU > /sys/kernel/slab/my_cache/flags
```

### 9.2 Red Zones

```
With SLAB_RED_ZONE:

[red zone (8 bytes)] [OBJECT (n bytes)] [red zone (8 bytes)]
 0xBB...                                 0xBB...

If object writes beyond its bounds → red zone is overwritten
→ Next alloc or free detects the corruption
```

### 9.3 `slabinfo` and `slabtop`

```bash
# Static view of all slab caches:
cat /proc/slabinfo
# Name     <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> ...
# dentry         89012      92160        192          21            1
# inode_cache    45678      51840        608           6            1
# sock_inode_cache 1234      2048        704           5            1
# filp            5678       8192        232          17            1
# task_struct      234        240       7168            1            2
# mm_struct        156        160       1088            7            1

# Interactive slab monitor (sorted by memory usage):
sudo slabtop
# Active / Total Objects (% used)    : 1234567 / 1456789 (84.7%)
# Active / Total Slabs (% used)      :   56789 /   60000 (94.6%)
# Active / Total Caches (% used)     :     234 /     300 (78.0%)
# Active / Total Size (% used)       :  456.78M / 523.45M (87.2%)
# Minimum / Average / Maximum Object : 0.01K / 0.42K / 8.00K
#  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
# 89012  87654  98%   0.19K   4238       21    16952K dentry
# 45678  44321  97%   0.59K   7613        6    30452K inode_cache
```

---

## 10. `kmem_cache_alloc_trace` — Allocation Tracing

```bash
# Enable per-cache allocation tracing:
echo 1 > /sys/kernel/slab/dentry/trace

# Now: every alloc/free from the dentry cache is logged via ftrace
cat /sys/kernel/debug/tracing/trace_pipe | grep "dentry"

# Disable:
echo 0 > /sys/kernel/slab/dentry/trace
```

---

## 11. Memory Accounting

```bash
# Total slab memory:
cat /proc/meminfo | grep Slab
# Slab:            2345678 kB    ← Total slab memory

# Reclaimable vs unreclaimable:
cat /proc/meminfo | grep -E 'SReclaimable|SUnreclaim'
# SReclaimable:    1234567 kB   ← Can be reclaimed under memory pressure
# SUnreclaim:       456789 kB   ← Kernel data that can't be reclaimed

# Per-cache statistics:
cat /sys/kernel/slab/dentry/alloc_calls    # Where dentry objects are allocated from
cat /sys/kernel/slab/dentry/total_objects
cat /sys/kernel/slab/dentry/objects_partial

# Memory usage by cache:
for cache in /sys/kernel/slab/*/; do
    name=$(basename $cache)
    obj_size=$(cat $cache/object_size 2>/dev/null || echo 0)
    objects=$(cat $cache/total_objects 2>/dev/null || echo 0)
    total_kb=$((objects * obj_size / 1024))
    if [ $total_kb -gt 0 ]; then
        echo "$total_kb KB    $name"
    fi
done | sort -rn | head -20
```

---

## 12. SLUB Cache Merging

SLUB automatically merges caches with compatible properties (same size, same flags) to reduce the total number of caches:

```bash
# See which caches are aliases (merged):
cat /proc/slabinfo | awk '{print $1}' | sort | grep -v '#' | head -30

# List of all actual caches vs aliases:
ls /sys/kernel/slab/ | wc -l  # Total cache directories
ls -la /sys/kernel/slab/ | grep '^l' | wc -l  # Aliases (symlinks)

# Disable merging (for debugging):
# Boot with: slab_nomerge
# Or at cache creation: use SLAB_NO_MERGE flag
```

---

## 13. `kmem_cache` for Real Kernel Subsystems

```bash
# VFS caches:
cat /proc/slabinfo | grep -E 'dentry|inode|file'
# dentry           89012 inode_cache  45678 filp  5678

# Network:
cat /proc/slabinfo | grep -E 'sock|skb|net'
# sock_inode_cache 1234 skbuff_head_cache 5678 net_namespace 12

# Task management:
cat /proc/slabinfo | grep -E 'task|mm_struct|signal'
# task_struct 234 mm_struct 156 signal_cache 89 sighand_cache 78

# Anonymous memory:
cat /proc/slabinfo | grep -E 'anon|vm_area'
# anon_vma 12345 anon_vma_chain 23456 vm_area_struct 34567
```

---

## 14. `ksize()` — Finding Actual Allocation Size

```c
// Useful for validating allocations:
void *ptr = kmalloc(200, GFP_KERNEL);
size_t actual = ksize(ptr);
// actual = 256 (kmalloc-256 cache)
// You can safely use 256 bytes even though you requested 200

// This is how krealloc decides whether to reallocate:
void *krealloc(const void *p, size_t new_size, gfp_t flags)
{
    if (unlikely(!new_size)) {
        kfree(p);
        return ZERO_SIZE_PTR;
    }
    
    if (ksize(p) >= new_size)
        return (void *)p;  // Already big enough!
    
    void *ret = kmalloc_track_caller(new_size, flags);
    if (ret && p) {
        memcpy(ret, p, ksize(p));
        kfree(p);
    }
    return ret;
}
```

---

## 15. Mental Model Checkpoint

After Day 25, you should be able to:

1. Explain the three levels of the SLUB hierarchy: `kmem_cache`, `kmem_cache_cpu`, `kmem_cache_node`.
2. Trace a `kmalloc(200, GFP_KERNEL)` call: which cache, what's the fast path?
3. How are free objects linked within a slab page?
4. What does `SLAB_HWCACHE_ALIGN` do and why does it matter for performance?
5. What happens when the per-CPU freelist is empty?
6. How does SLUB detect use-after-free? What poison value is used?
7. When would you create a custom `kmem_cache` instead of using `kmalloc`?
8. What is `ksize()` and when is it useful?

---

## Key Source Files

```bash
mm/slub.c                    # SLUB implementation (3000+ lines, the core)
include/linux/slab.h         # kmalloc, kmem_cache_* API
include/linux/slub_def.h     # struct kmem_cache, kmem_cache_cpu
mm/slab_common.c             # Common slab code (shared between SLAB/SLUB/SLOB)
mm/slab.h                    # Internal slab definitions
include/linux/poison.h       # Poison byte values
Documentation/mm/slub.rst    # Official SLUB documentation
```

---

## Summary

SLUB organizes object allocation into three tiers:

1. **Per-CPU freelist** (fast path, no locks): pointer chasing through free objects
2. **Per-CPU partial list** (medium path, no lock): partially filled slabs local to CPU
3. **Per-node partial list** (slow path, needs lock): shared partially filled slabs

`kmalloc` maps size requests to power-of-2 caches (kmalloc-8 through kmalloc-8192). Custom caches via `kmem_cache_create` enable named allocations with tunable alignment and debugging.

SLUB's design principle: make the common case (small allocation, memory available, same CPU) a pure pointer operation with zero locking. Everything else is a slow path.

Tomorrow: virtual memory areas — how `mmap()` works and how VMAs represent a process's address space.
