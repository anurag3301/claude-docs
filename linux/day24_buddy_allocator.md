# Day 24 — Buddy Allocator

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand the buddy system implementation — free lists by order, allocation and freeing algorithms, fragmentation, and GFP flags in depth.

---

## 1. The Problem the Buddy Allocator Solves

The kernel frequently needs physically **contiguous** pages — not just pages that are virtually contiguous:
- DMA operations: the DMA engine accesses physical addresses directly
- Huge pages (THP, HugeTLBFS): must be physically contiguous
- Some device drivers require contiguous buffers

The buddy allocator manages the physical page pool and satisfies requests for 1, 2, 4, 8, ... 1024 contiguous pages.

---

## 2. The Buddy System Concept

Pages are allocated in **powers of 2** (called orders). The allocator maintains free lists for each order from 0 to `MAX_ORDER` (10 on most systems):

```
Order 0: free lists of 1-page blocks    (4KB each)
Order 1: free lists of 2-page blocks    (8KB each)
Order 2: free lists of 4-page blocks   (16KB each)
...
Order 10: free lists of 1024-page blocks (4MB each)
```

The "buddy" property: every allocated block has a **buddy** — the block at the same level that was split from the same parent. The buddy's PFN is obtained by flipping bit N (where N = order) in the block's PFN:

```
Order-1 block at PFN 100:
  buddy PFN = 100 XOR (1 << 1) = 100 XOR 2 = 102

Order-2 block at PFN 100:
  buddy PFN = 100 XOR (1 << 2) = 100 XOR 4 = 104

Order-3 block at PFN 104:
  buddy PFN = 104 XOR (1 << 3) = 104 XOR 8 = 96
```

---

## 3. Allocation: `alloc_pages()`

### 3.1 The API

```c
// include/linux/gfp.h

// Allocate 2^order contiguous pages:
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);
struct page *alloc_pages_node(int nid, gfp_t gfp_mask, unsigned int order);

// Single page (order=0):
struct page *alloc_page(gfp_t gfp_mask);
unsigned long __get_free_page(gfp_t gfp_mask);   // Returns virtual address
unsigned long get_zeroed_page(gfp_t gfp_mask);   // Zero-filled

// Multiple pages:
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);

// Free (must use same order as allocation):
__free_pages(page, order);
free_pages(addr, order);
```

### 3.2 The Allocation Algorithm

```c
// mm/page_alloc.c (simplified)
struct page *__alloc_pages(gfp_t gfp_mask, unsigned int order,
                           int preferred_nid, nodemask_t *nodemask)
{
    struct page *page;
    
    // Fast path: try preferred zone with low watermark check:
    page = get_page_from_freelist(gfp_mask, order, ALLOC_WMARK_LOW, ...);
    if (likely(page))
        return page;
    
    // Slow path: memory pressure situation
    page = __alloc_pages_slowpath(gfp_mask, order, ...);
    return page;
}

static struct page *get_page_from_freelist(gfp_t gfp_mask, unsigned int order,
                                           unsigned int alloc_flags, ...)
{
    struct zone *zone;
    
    // Try each zone in the zonelist:
    for_each_zone_zonelist(zone, ...) {
        // Check watermarks:
        if (!zone_watermark_ok(zone, order, watermark, ...))
            continue;  // Below watermark — try next zone
        
        // Try to allocate from this zone:
        page = rmqueue(preferred_zone, zone, order, gfp_mask, ...);
        if (page)
            return page;
    }
    return NULL;
}
```

### 3.3 `rmqueue()` — The Core Allocation

```c
// mm/page_alloc.c
static __always_inline struct page *rmqueue(struct zone *preferred_zone,
                                             struct zone *zone,
                                             unsigned int order,
                                             gfp_t gfp_flags, ...)
{
    struct page *page;
    
    if (likely(order == 0)) {
        // Fast path: single page from per-CPU pageset (no zone lock!)
        page = rmqueue_pcplist(preferred_zone, zone, gfp_flags, ...);
        if (likely(page))
            return page;
    }
    
    // Multi-page or fallback: take zone lock
    spin_lock_irqsave(&zone->lock, flags);
    
    do {
        page = NULL;
        // Try migration type (MIGRATE_MOVABLE, MIGRATE_UNMOVABLE, MIGRATE_RECLAIMABLE):
        if (order > 0 && (alloc_flags & ALLOC_HARDER))
            page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
        
        if (!page)
            page = __rmqueue(zone, order, migratetype, alloc_flags);
    } while (!page && zone_check_alc_request_allowed(...));
    
    spin_unlock_irqrestore(&zone->lock, flags);
    return page;
}
```

### 3.4 `__rmqueue_smallest()` — Splitting Blocks

```c
// mm/page_alloc.c
static __always_inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
                                int migratetype)
{
    unsigned int current_order;
    struct free_area *area;
    struct page *page;
    
    // Search from the requested order up to MAX_ORDER:
    for (current_order = order; current_order <= MAX_ORDER; current_order++) {
        area = &zone->free_area[current_order];
        
        // Check if there's a block at this order:
        page = get_page_from_free_area(area, migratetype);
        if (!page)
            continue;
        
        // Found! Remove it from the free list:
        del_page_from_free_list(page, zone, current_order);
        
        // Expand: if block is larger than requested, split and return remainder:
        expand(zone, page, order, current_order, migratetype);
        // expand() puts the "buddy" halves back into lower free lists
        
        return page;
    }
    return NULL;  // Out of memory
}

// Example: requesting order-2 (4 pages), find order-4 (16 pages):
// expand(zone, page, order=2, current_order=4, ...):
//   Put page[8..15] into order-3 free list  (8 pages)
//   Put page[4..7] into order-2 free list   (4 pages = 1 order-2 block)
//   Return page[0..3] to caller             (the 4 pages requested)
static void expand(struct zone *zone, struct page *page,
                   int low, int high, int migratetype)
{
    unsigned long size = 1 << high;
    
    while (high > low) {
        high--;
        size >>= 1;
        // The "buddy" of what we're keeping:
        struct page *buddy = page + size;  // Upper half
        // Add buddy to the appropriate free list:
        add_to_free_list(buddy, zone, high, migratetype);
        set_buddy_order(buddy, high);
    }
}
```

---

## 4. Freeing: `__free_pages()`

```c
// mm/page_alloc.c
void __free_pages(struct page *page, unsigned int order)
{
    if (order == 0)
        free_unref_page(page, order);  // Return to per-CPU pageset
    else
        __free_pages_ok(page, order, FPI_NONE);
}

static void __free_pages_core(struct page *page, unsigned int order)
{
    // Merge buddies: try to combine this block with its free buddy
    // Walk up the order tree merging as long as the buddy is free:
    while (order < MAX_ORDER) {
        // Calculate buddy PFN:
        unsigned long buddy_pfn = pfn ^ (1 << order);
        struct page *buddy = page + (buddy_pfn - pfn);
        
        // Is the buddy free at this order?
        if (!page_is_buddy(page, buddy, order))
            break;  // Buddy not free — stop merging
        
        // Merge: remove buddy from free list:
        del_page_from_free_list(buddy, zone, order);
        
        // The merged block starts at the lower of the two PFNs:
        page = (pfn < buddy_pfn) ? page : buddy;
        pfn = page_to_pfn(page);
        
        order++;  // Now working with a higher-order block
    }
    
    // Add the (possibly merged) block to the free list:
    add_to_free_list(page, zone, order, migratetype);
    set_buddy_order(page, order);
}
```

---

## 5. Migration Types

To reduce fragmentation, pages are categorized by their "movability":

```c
// include/linux/mmzone.h
enum migratetype {
    MIGRATE_UNMOVABLE,   // Pages that cannot be moved (kernel data, etc.)
    MIGRATE_MOVABLE,     // Pages that can be migrated (anonymous, page cache)
    MIGRATE_RECLAIMABLE, // Pages that can be reclaimed (filesystem caches)
    MIGRATE_PCPTYPES,    // (= 3, number of types in per-CPU lists)
    MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,  // Emergency reserve
    MIGRATE_CMA,         // CMA (contiguous memory allocator) pages
    MIGRATE_ISOLATE,     // Pages being isolated for migration/offlining
    MIGRATE_TYPES        // Total count
};
```

Each `free_area[order]` has separate lists for each migration type:

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];
    unsigned long    nr_free;  // Total free pages at this order
};
```

**Why this matters**: When the kernel allocates an unmovable page, it takes from `MIGRATE_UNMOVABLE` lists. When pages are movable (process heap, page cache), they go to `MIGRATE_MOVABLE`. This prevents unmovable allocations from fragmenting the movable pool.

---

## 6. GFP Flags in Depth

GFP (Get Free Page) flags control how and where the allocator works:

```c
// Primary zone modifiers (where to allocate from):
#define __GFP_DMA      ((__force gfp_t)___GFP_DMA)      // ZONE_DMA only
#define __GFP_HIGHMEM  ((__force gfp_t)___GFP_HIGHMEM)  // Allow ZONE_HIGHMEM
#define __GFP_DMA32    ((__force gfp_t)___GFP_DMA32)    // ZONE_DMA32 only
#define __GFP_MOVABLE  ((__force gfp_t)___GFP_MOVABLE)  // ZONE_MOVABLE preferred

// Reclaim modifiers (what can be done to get memory):
#define __GFP_RECLAIMABLE ((__force gfp_t)___GFP_RECLAIMABLE) // Pages are reclaimable
#define __GFP_WRITE      ((__force gfp_t)___GFP_WRITE)   // Hint: will be written
#define __GFP_HARDWALL   ((__force gfp_t)___GFP_HARDWALL) // Strict NUMA policy
#define __GFP_THISNODE   ((__force gfp_t)___GFP_THISNODE) // Must be from this node
#define __GFP_ACCOUNT    ((__force gfp_t)___GFP_ACCOUNT)  // Charge to cgroup
#define __GFP_RECLAIM    ((__force gfp_t)___GFP_RECLAIM)  // Allow page reclaim
#define __GFP_DIRECT_RECLAIM (__force gfp_t)___GFP_DIRECT_RECLAIM) // Allow direct reclaim
#define __GFP_KSWAPD_RECLAIM ((__force gfp_t)___GFP_KSWAPD_RECLAIM) // Allow waking kswapd
#define __GFP_IO         ((__force gfp_t)___GFP_IO)    // Allow disk I/O
#define __GFP_FS         ((__force gfp_t)___GFP_FS)    // Allow filesystem operations
#define __GFP_RETRY_MAYFAIL ((__force gfp_t)___GFP_RETRY_MAYFAIL) // Retry but may fail
#define __GFP_NOFAIL     ((__force gfp_t)___GFP_NOFAIL)  // Must not fail
#define __GFP_NORETRY    ((__force gfp_t)___GFP_NORETRY) // Don't retry
#define __GFP_MEMALLOC   ((__force gfp_t)___GFP_MEMALLOC) // Use memory reserves
#define __GFP_NOMEMALLOC ((__force gfp_t)___GFP_NOMEMALLOC) // Don't use reserves
#define __GFP_NOWARN     ((__force gfp_t)___GFP_NOWARN) // Suppress OOM warning
#define __GFP_COMP       ((__force gfp_t)___GFP_COMP)   // Compound page (THP/hugetlb)
#define __GFP_ZERO       ((__force gfp_t)___GFP_ZERO)   // Zero-fill

// Composite flags (what you actually use):
#define GFP_KERNEL   (__GFP_RECLAIM | __GFP_IO | __GFP_FS)
// Can sleep, can do I/O, can do filesystem ops, can reclaim pages

#define GFP_ATOMIC   (__GFP_HIGH | __GFP_ATOMIC | __GFP_KSWAPD_RECLAIM)
// Cannot sleep; uses memory reserves; for IRQ context

#define GFP_NOWAIT   (__GFP_KSWAPD_RECLAIM)
// Cannot sleep but doesn't use reserves; may fail

#define GFP_DMA      (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_DMA)

#define GFP_USER     (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
// For user-space pages; respects NUMA policy; can do everything

#define GFP_HIGHUSER (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL | __GFP_HIGHMEM)
// Like GFP_USER but can use ZONE_HIGHMEM (32-bit)

#define GFP_HIGHUSER_MOVABLE (__GFP_RECLAIM | __GFP_IO | __GFP_FS | \
                              __GFP_HARDWALL | __GFP_HIGHMEM | __GFP_MOVABLE)
// For movable user pages (page cache, anonymous pages)
// Most page cache allocations use this
```

### 6.1 GFP Flag Decision Tree

```
Can the allocator sleep (block)?
  YES → GFP_KERNEL (normal kernel allocs)
        GFP_USER   (for user-space pages)
  NO  → Are you in hardirq/atomic context?
        YES → GFP_ATOMIC (uses reserves, never fails silently)
        NO  → GFP_NOWAIT (no sleep, no reserves)

Do you need DMA-capable memory?
  YES → Add __GFP_DMA or __GFP_DMA32

Do you need it to succeed no matter what?
  YES → Add __GFP_NOFAIL (loops forever — use VERY sparingly)
  
Do you want no OOM warning on failure?
  YES → Add __GFP_NOWARN
```

---

## 7. The Slowpath: What Happens When Memory Is Low

```c
// mm/page_alloc.c
static page *__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order, ...)
{
    // 1. Wake kswapd to reclaim pages in the background:
    wake_all_kswapds(order, gfp_mask, ac);
    
    // 2. Try again with lower watermarks:
    page = get_page_from_freelist(gfp_mask, order, ALLOC_WMARK_MIN | ..., ac);
    if (page)
        return page;
    
    // 3. If __GFP_DIRECT_RECLAIM: do synchronous memory reclaim:
    if (gfp_mask & __GFP_DIRECT_RECLAIM) {
        page = __alloc_pages_direct_reclaim(gfp_mask, order, ...);
        if (page)
            return page;
    }
    
    // 4. Try compaction (move movable pages to consolidate free space):
    if (can_compact) {
        page = __alloc_pages_direct_compact(gfp_mask, order, ...);
        if (page)
            return page;
    }
    
    // 5. OOM killer (if __GFP_FS | __GFP_IO and retries exhausted):
    if (should_oom_kill) {
        page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
        if (page)
            return page;
    }
    
    // 6. Truly out of memory:
    if (!(gfp_mask & __GFP_NOFAIL))
        return NULL;
    
    // __GFP_NOFAIL: keep retrying:
    goto retry;
}
```

---

## 8. Buddy Allocator Statistics

```bash
# Free pages by order per zone:
cat /proc/buddyinfo
# Node 0, zone      DMA      1  1  0  0  2  1  1  0  1  1  3
# Node 0, zone    DMA32      4  3  2  1  1  1  0  1  1  1  0
# Node 0, zone   Normal    156 43  27  19  12   7   3   1   0   1   1

# Column interpretation:
# Col 0 = # of free 4KB blocks (order 0)
# Col 1 = # of free 8KB blocks (order 1)
# ...
# Col 10 = # of free 4MB blocks (order 10)

# Fragmentation: many small blocks, few large blocks = fragmented

# vmstat: allocation counters:
cat /proc/vmstat | grep -E "pgalloc|pgfree|pgscan"
# pgalloc_dma          0        ← Pages allocated from DMA zone
# pgalloc_dma32     1234        ← Pages from DMA32
# pgalloc_normal  1234567       ← Pages from Normal zone (the big one)
# pgfree         1234560        ← Pages freed
# pgscan_kswapd   12345         ← Pages scanned by kswapd
# pgscan_direct    1234         ← Pages scanned by direct reclaim

# Buddy allocator performance:
cat /proc/vmstat | grep -E "pgallocdma|order"
# (not available directly — use perf or tracing)
```

---

## 9. `alloc_pages` vs `vmalloc` Revisited

```c
// When to use alloc_pages:
// - DMA buffers (need physical contiguity)
// - Huge page allocations
// - Low-level drivers
// - Single-page allocations (most common case)

// When NOT to use alloc_pages directly:
// - General kernel allocations → use kmalloc (builds on slab which uses alloc_pages)
// - Large buffers where contiguity doesn't matter → use vmalloc

// Size constraints:
// alloc_pages(GFP_KERNEL, order):
//   order=0: 4KB
//   order=1: 8KB
//   order=2: 16KB
//   ...
//   order=10: 4MB (MAX_ORDER on most systems)
// Can't allocate >4MB physically contiguous with buddy allocator!

// For very large DMA allocations:
// Use CMA (Contiguous Memory Allocator): reserves a region at boot
// dma_alloc_contiguous() → uses CMA
```

---

## 10. `CMA` — Contiguous Memory Allocator

```c
// For drivers needing large physically contiguous DMA buffers:
// Reserves memory at boot that looks "free" but is available for CMA

// Reserve CMA region (kernel command line):
// cma=128M        ← 128MB reserved for CMA
// cma=128M@0x400000000  ← At specific physical address

// Allocator API:
struct page *cma_alloc(struct cma *cma, size_t count,
                       unsigned int align, bool no_warn);
bool cma_release(struct cma *cma, const struct page *pages, unsigned int count);

// DMA API (preferred for drivers):
void *dma_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t flag);
// Internally uses CMA when size > PAGE_SIZE
// Returns virtual address; *dma_handle = DMA-accessible physical address
```

---

## 11. High-Order Allocation Failures

High-order allocations (order ≥ 3) frequently fail on loaded systems. This is the primary cause of `vmalloc` existing:

```bash
# Check if high-order allocations are failing:
cat /proc/vmstat | grep -E "allocstall|drop_|direct_compact"
# allocstall_normal    1234  ← Direct reclaim stalls (waiting for memory)
# compact_stall        567   ← Compaction stalls

# Check fragmentation:
cat /sys/kernel/debug/extfrag/unusable_index
# Shows what fraction of free memory is unusable for each order
# 0.000 = perfect (all usable for this order)
# 1.000 = completely unusable (all free pages are too fragmented)

# Trigger proactive compaction:
echo 1 > /proc/sys/vm/compact_memory

# Or configure automatic compaction:
cat /proc/sys/vm/compaction_proactiveness  # 0 = off, 100 = aggressive
echo 20 > /proc/sys/vm/compaction_proactiveness
```

---

## 12. Tracing the Buddy Allocator

```bash
# ftrace: trace all page allocations:
cd /sys/kernel/debug/tracing
echo 'mm:mm_page_alloc' > set_event
echo 1 > tracing_on
sleep 1
cat trace | head -30
# mm_page_alloc: page=0xffff... pfn=0x12345 order=0 migratetype=0 gfp_flags=0xcc0

# bpftrace: trace high-order allocations:
sudo bpftrace -e '
tracepoint:mm:mm_page_alloc {
    if (args->order >= 3) {
        printf("high-order alloc: order=%d gfp=0x%x comm=%s\n",
            args->order, args->gfp_flags, comm);
    }
}'

# perf: count page allocations per process:
sudo perf stat -e 'kmem:mm_page_alloc' -ag sleep 5
```

---

## 13. Mental Model Checkpoint

After Day 24, you should be able to:

1. Explain the buddy system: what is an "order" and what is a "buddy"?
2. Trace a request for 3 pages (order=1 won't work — need order=2): what splitting happens?
3. Trace freeing an order-2 block: when does merging happen and when does it stop?
4. What is a migration type and why does it reduce fragmentation?
5. What does `GFP_KERNEL` allow the allocator to do that `GFP_ATOMIC` doesn't?
6. What happens in the slowpath when `get_page_from_freelist` returns NULL?
7. Why can't you allocate more than 4MB physically contiguous with the buddy allocator?
8. What is CMA and when do you need it?

---

## Key Source Files

```bash
mm/page_alloc.c                   # ENTIRE buddy allocator: alloc_pages, __free_pages
mm/compaction.c                   # Memory compaction
include/linux/gfp.h               # All GFP_* flags
include/linux/gfp_types.h         # GFP flag bit definitions
include/linux/mmzone.h            # struct free_area, zone watermarks
mm/internal.h                     # Internal allocator helpers
mm/cma.c                          # Contiguous Memory Allocator
Documentation/mm/page_migration.rst  # Page migration and compaction
```

---

## Summary

The buddy allocator is elegantly simple: maintain free lists at each power-of-2 size. Allocating requires splitting larger blocks (top-down); freeing triggers merging with buddies (bottom-up). The XOR trick makes buddy computation O(1).

Key design decisions:
- **Per-CPU pagesets**: eliminate zone lock for single-page allocs (99% of cases)
- **Migration types**: separate movable from unmovable to reduce fragmentation  
- **Watermarks**: three thresholds (min/low/high) trigger progressively aggressive reclaim
- **Slowpath**: wakes kswapd → direct reclaim → compaction → OOM killer

Tomorrow: the SLUB allocator — how `kmalloc` works on top of the buddy system.
