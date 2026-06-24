# Day 5 — Kernel Memory Management: kmalloc, vmalloc, Slab & GFP Flags
**Environment:** 💻 QEMU/PC
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [Physical vs Virtual Memory — The Full Picture](#1-physical-vs-virtual-memory--the-full-picture)
2. [The Kernel's Virtual Address Space (ARM64 & x86-64)](#2-the-kernels-virtual-address-space-arm64--x86-64)
3. [struct page — The Basic Unit](#3-struct-page--the-basic-unit)
4. [The Page Allocator (Buddy System)](#4-the-page-allocator-buddy-system)
5. [The Slab Allocator](#5-the-slab-allocator)
6. [kmalloc — What Drivers Use Most](#6-kmalloc--what-drivers-use-most)
7. [vmalloc — Large Non-Contiguous Allocations](#7-vmalloc--large-non-contiguous-allocations)
8. [GFP Flags — Controlling Allocation Behavior](#8-gfp-flags--controlling-allocation-behavior)
9. [Memory Pools — Pre-allocated Reserves](#9-memory-pools--pre-allocated-reserves)
10. [Common Memory Bugs in Kernel Code](#10-common-memory-bugs-in-kernel-code)
11. [Hands-On Lab — Memory Demo Module in QEMU](#11-hands-on-lab--memory-demo-module-in-qemu)
12. [Interview Questions](#12-interview-questions)

---

## 1. Physical vs Virtual Memory — The Full Picture

Understanding memory in the kernel requires holding two maps in your head simultaneously:

### Physical Memory Layout (RAM)

Physical memory is the actual DRAM chips. On a system with 4GB RAM, physical addresses run from `0x0000_0000` to `0xFFFF_FFFF`. But this map isn't just RAM — it also contains:

```
Physical Address Space (example 4GB ARM64 system):

0x0000_0000 ┌───────────────────┐
            │  Reserved/Boot    │  (BIOS/firmware vectors)
0x0000_8000 ├───────────────────┤
            │  Kernel Image     │  (text, data, bss)
            ├───────────────────┤
            │  initrd           │  (initial ramdisk)
            ├───────────────────┤
            │  Free RAM         │  (managed by page allocator)
            │                   │
            │       ...         │
0xFC00_0000 ├───────────────────┤
            │  MMIO Regions     │  (device registers — NOT RAM!)
            │  e.g. GPIO, UART  │
            │  e.g. PCIe        │
0xFFFF_FFFF └───────────────────┘
```

Device registers appear in the physical address space as MMIO (Memory-Mapped I/O). The CPU reads/writes these addresses to communicate with hardware, but there's no RAM there.

### Virtual Memory

Every process has its own virtual address space. The kernel has its own virtual address space too. The MMU translates virtual addresses to physical addresses via page tables.

```
Virtual Address Space (ARM64, 48-bit addressing):

User Process Virtual Space:
0x0000_0000_0000_0000 ┌───────────────────┐
                      │  Program Text      │
                      │  Program Data      │
                      │  Heap              │
                      │  Stack             │
                      │  Shared Libraries  │
0x0000_FFFF_FFFF_FFFF └───────────────────┘
                         [huge gap — invalid addresses]
Kernel Virtual Space:
0xFFFF_0000_0000_0000 ┌───────────────────┐
                      │  vmalloc area     │  vmalloc()
                      ├───────────────────┤
                      │  Direct Map       │  Linear mapping of all RAM
                      │  (physaddr + base)│  kmalloc() memory lives here
                      ├───────────────────┤
                      │  Kernel Image     │  kernel text/data
                      ├───────────────────┤
                      │  Modules          │  .ko files loaded here
0xFFFF_FFFF_FFFF_FFFF └───────────────────┘
```

### The Direct Map (Kernel Linear Mapping)

The most important concept for driver writers: **the kernel maps all physical RAM at a fixed virtual offset**. On ARM64 with `CONFIG_ARM64_VA_BITS=48`, all physical RAM is accessible at `virtual_address = physical_address + PAGE_OFFSET` where `PAGE_OFFSET = 0xFFFF_0000_0000_0000` (approximately).

This means:
- Physical address `0x4000_0000` (1GB) → kernel virtual `0xFFFF_0000_4000_0000`
- `kmalloc()` returns addresses in this direct map
- `virt_to_phys(ptr)` converts a kernel virtual address to its physical address
- `phys_to_virt(phys)` converts physical to kernel virtual

**Why this matters for drivers:** DMA operations need physical addresses. When you do `kmalloc()` and give the memory to a DMA controller, you need `virt_to_phys()` to convert your kernel pointer to a physical address the hardware can use.

---

## 2. The Kernel's Virtual Address Space (ARM64 & x86-64)

### ARM64 (48-bit virtual addresses, typical for RPi4)

```
0xFFFF_FFFF_FFFF_FFFF ─── End of address space
        │  Fixed mappings, kmap, etc.
0xFFFF_FF80_0000_0000 ─── vmemmap (struct page array)
        │
0xFFFF_FF00_0000_0000 ─── vmalloc / ioremap area
        │
0xFFFF_C000_0000_0000 ─── Linear map of all physical memory (direct map)
        │  (all RAM accessible here: physaddr + PAGE_OFFSET)
0xFFFF_8000_0000_0000 ─── Kernel image, modules
```

### x86-64 (48-bit virtual addresses)

```
0xFFFF_FFFF_FFFF_FFFF ─── End
        │  Fixmaps
0xFFFF_FFFF_8000_0000 ─── Kernel text (loaded at ~16MB physical)
        │
0xFFFF_E000_0000_0000 ─── vmalloc/ioremap area
        │
0xFFFF_8800_0000_0000 ─── Direct map (all RAM)
        │
0xFFFF_8000_0000_0000 ─── Guard hole
0x0000_7FFF_FFFF_FFFF ─── End of user space
```

### Key Zones

The kernel divides physical memory into zones based on DMA reachability:

| Zone | x86-64 | Purpose |
|------|--------|---------|
| `ZONE_DMA` | 0-16MB | Legacy ISA DMA (24-bit address) |
| `ZONE_DMA32` | 0-4GB | 32-bit DMA devices |
| `ZONE_NORMAL` | >4GB | Normal kernel allocations |
| `ZONE_HIGHMEM` | (32-bit only) | Memory not directly mapped |
| `ZONE_MOVABLE` | All | For memory hotplug |

On modern 64-bit systems you primarily deal with `ZONE_DMA32` and `ZONE_NORMAL`. GFP flags (covered later) let you specify which zone to allocate from.

---

## 3. struct page — The Basic Unit

Every physical page frame (4KB on most architectures) is represented by a `struct page` in the kernel. For a 4GB system, that's 1,048,576 `struct page` objects. The kernel stores them in the **vmemmap** area.

```c
// include/linux/mm_types.h (simplified)
struct page {
    unsigned long flags;        // PG_locked, PG_dirty, PG_uptodate, etc.
    
    union {
        struct {
            // For pages in the page allocator (free or allocated)
            struct list_head lru;   // LRU list (or free list)
            unsigned long private;
        };
        struct {
            // For pages used as slab objects
            struct kmem_cache *slab_cache;
            void *freelist;
        };
        // ... more unions for other page types
    };
    
    union {
        atomic_t _mapcount;  // number of page table entries pointing here
        unsigned int page_type;
    };
    
    atomic_t _refcount;     // reference count — don't free if > 0
    
    // ... architecture-specific fields
};
```

`struct page` is exactly 64 bytes on most platforms. For 4GB of RAM with 4KB pages, the vmemmap takes: `1M pages × 64 bytes = 64MB`. This is the memory overhead of just tracking all physical pages.

### Page Flags

```c
// Common page flags (include/linux/page-flags.h)
PG_locked      // page is locked (I/O in progress)
PG_referenced  // page was recently accessed
PG_dirty       // page content modified, not yet written to disk
PG_uptodate    // page content is valid
PG_lru         // page is on an LRU list
PG_active      // page is on the active LRU list
PG_slab        // page is used by the slab allocator
PG_reserved    // page is reserved by BIOS/firmware
```

---

## 4. The Page Allocator (Buddy System)

The **buddy system** is the lowest-level physical memory allocator. It allocates and frees memory in **power-of-2 page blocks** (1, 2, 4, 8, 16... pages).

### How It Works

The allocator maintains free lists for each order (block size):
```
Order 0: blocks of 1 page  (4KB)
Order 1: blocks of 2 pages (8KB)
Order 2: blocks of 4 pages (16KB)
...
Order 10: blocks of 1024 pages (4MB)  ← MAX_ORDER
```

Allocation of 8KB (order 1):
1. Check free list for order 1 — if available, return it
2. If not, take a block from order 2, split it → two order-1 blocks ("buddies")
3. Return one, put the other in order-1 free list

Freeing 8KB (order 1):
1. Check if the "buddy" (adjacent block of same size) is also free
2. If yes: merge them into an order-2 block, check for order-2 buddy, repeat
3. This "bubbles up" until no buddy is free or max order reached

The page allocator API:
```c
#include <linux/gfp.h>

// Allocate 2^order contiguous pages
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);

// Allocate a single page
struct page *alloc_page(gfp_t gfp_mask);

// Allocate and return virtual address (order pages)
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);

// Allocate a single page, return virtual address
unsigned long get_zeroed_page(gfp_t gfp_mask);

// Free pages
void __free_pages(struct page *page, unsigned int order);
void free_pages(unsigned long addr, unsigned int order);
```

Drivers rarely call the page allocator directly. They use `kmalloc()` (which wraps the slab allocator) or `alloc_pages()` for large DMA buffers.

### Checking Memory Stats

```bash
# On RPi4 / QEMU
cat /proc/buddyinfo
# Node 0, zone   Normal   263 233 116  55  28  16  12  10   3   2  10
# ^ Each number = count of free blocks at that order
# Order:          0   1    2   3   4   5   6   7   8   9  10

cat /proc/meminfo
# MemTotal:        4032196 kB
# MemFree:         3245632 kB
# MemAvailable:    3891200 kB
# Buffers:           18368 kB
# Cached:           611560 kB
# SlabReclaimable:   51392 kB
# SUnreclaim:        18688 kB
```

---

## 5. The Slab Allocator

The buddy system is efficient for page-sized allocations but terrible for small objects (a 64-byte `struct` would waste 4KB using a full page). The **slab allocator** solves this.

### Concept

The slab allocator pre-allocates **caches** of fixed-size objects. When you `kmalloc(64, ...)`, it gives you an object from the 64-byte slab cache. When you `kfree()`, the object goes back to the cache — not returned to the buddy system immediately. This makes allocation/deallocation very fast (often just pointer manipulation) and reduces fragmentation.

Linux currently uses **SLUB** (the Simple/Unqueued allocator) by default, which is a simplified, more scalable version of the original SLAB. There's also SLOB for embedded systems.

### Slab Cache Sizes

The kernel pre-creates caches for common sizes (powers of 2):
- 8, 16, 32, 64, 96, 128, 192, 256, 512, 1024, 2048, 4096, 8192... bytes

When you call `kmalloc(100, ...)`, it rounds up to 128 bytes and uses the 128-byte cache.

```bash
# See all slab caches and their sizes
sudo cat /proc/slabinfo | head -30

# Or with slabtop
sudo slabtop

# More detail
sudo cat /sys/kernel/slab/kmalloc-64/object_size
```

### Custom Slab Caches

If your driver allocates many instances of the same structure, create a dedicated cache:

```c
#include <linux/slab.h>

// Create a custom slab cache
struct kmem_cache *my_cache;

my_cache = kmem_cache_create(
    "mydriver_obj",        // name (appears in /proc/slabinfo)
    sizeof(struct myobj),  // object size
    0,                     // alignment (0 = default)
    SLAB_HWCACHE_ALIGN,    // flags: align to CPU cache line
    NULL                   // constructor (called on each new object)
);
if (!my_cache)
    return -ENOMEM;

// Allocate from custom cache
struct myobj *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
if (!obj) { /* handle error */ }

// Free back to cache
kmem_cache_free(my_cache, obj);

// Destroy the cache (in module exit — only after freeing ALL objects!)
kmem_cache_destroy(my_cache);
```

Benefits of custom caches:
- Objects appear in `/proc/slabinfo` with your name (debugging)
- Can specify alignment (useful for DMA)
- Can specify a constructor that initializes objects when allocated

---

## 6. kmalloc — What Drivers Use Most

`kmalloc()` is the primary kernel memory allocator. It allocates **physically contiguous** memory from the slab allocator (for sizes ≤ typically 4MB) or the page allocator directly (for larger sizes).

```c
#include <linux/slab.h>

void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *objp);

// Zeroed version (like calloc)
void *kzalloc(size_t size, gfp_t flags);   // always prefer this

// Reallocate (like realloc)
void *krealloc(const void *p, size_t new_size, gfp_t flags);

// Allocate array (size * count), checks for overflow
void *kmalloc_array(size_t n, size_t size, gfp_t flags);
void *kcalloc(size_t n, size_t size, gfp_t flags);  // zeroed array
```

### kmalloc Size Limits

```
Size           → Result
1 - 192        → 192-byte slab cache object
193 - 256      → 256-byte slab cache object
...
4097 - 8192    → 8192-byte slab cache object (2 pages)
8193 - 16384   → 16384-byte object (4 pages)
...
4MB (max)      → Direct page allocator call
> 4MB          → Fails — use vmalloc() instead
```

### Physical Contiguity Requirement

`kmalloc()` returns **physically contiguous** memory. This is critical for DMA — many DMA controllers can only access contiguous physical memory. If you need to DMA a 4KB buffer, `kmalloc(4096, GFP_KERNEL)` guarantees it's one contiguous block.

### kfree Safety

```c
// kfree(NULL) is always safe — checks for NULL internally
kfree(NULL);    // no-op, no crash

// NEVER double-free — causes slab corruption
kfree(ptr);
kfree(ptr);     // CRASH or silent corruption

// Good pattern: NULL after free
kfree(ptr);
ptr = NULL;     // prevents accidental reuse

// NEVER free stack memory or static data
char buf[128];
kfree(buf);     // CRASH — buf is on the stack, not in slab
```

---

## 7. vmalloc — Large Non-Contiguous Allocations

`vmalloc()` allocates **virtually contiguous** but potentially **physically non-contiguous** memory.

```c
#include <linux/vmalloc.h>

void *vmalloc(unsigned long size);
void vfree(const void *addr);

void *vzalloc(unsigned long size);  // zeroed
```

### kmalloc vs vmalloc Comparison

| Feature | `kmalloc` | `vmalloc` |
|---------|-----------|-----------|
| Physical contiguity | Yes (guaranteed) | No (scattered pages) |
| Virtually contiguous | Yes | Yes |
| DMA-safe | Yes | NO — don't use for DMA |
| Max size | ~4MB | Virtually unlimited |
| Performance | Fast (slab) | Slower (must set up PTEs) |
| Best for | Small/medium allocations, DMA buffers | Large software buffers (firmware loading, etc.) |
| Overhead | Minimal | Page table entries for each page |

### When to Use vmalloc

```c
// Loading a firmware image (large, no DMA needed)
void *fw_buf = vmalloc(firmware_size);
if (!fw_buf) return -ENOMEM;
memcpy(fw_buf, firmware_data, firmware_size);
// ... process firmware ...
vfree(fw_buf);

// Large lookup table
u32 *table = vzalloc(256 * 1024 * sizeof(u32));  // 1MB table
```

### Converting vmalloc Addresses to Physical

You CAN get physical addresses of vmalloc memory, but they're NOT contiguous:
```c
void *vaddr = vmalloc(PAGE_SIZE * 4);  // 4 pages, possibly scattered
unsigned long phys0 = vmalloc_to_pfn(vaddr) << PAGE_SHIFT;
unsigned long phys1 = vmalloc_to_pfn(vaddr + PAGE_SIZE) << PAGE_SHIFT;
// phys0 and phys1 might be completely different physical pages
```

This is why vmalloc memory can't be used for simple DMA operations.

---

## 8. GFP Flags — Controlling Allocation Behavior

GFP stands for "Get Free Pages." These flags control how the allocator behaves when memory is scarce.

### Primary GFP Flags

```c
GFP_KERNEL     // Standard allocation — can sleep, can swap, can reclaim
               // Use in: process context (most common in drivers)
               // Will wait for memory to become available if needed

GFP_ATOMIC     // Atomic allocation — CANNOT sleep
               // Use in: interrupt handlers, spinlock-held sections, atomic context
               // Will fail immediately if memory not available
               // Uses emergency memory reserves

GFP_USER       // Allocation for user-space pages
               // Like GFP_KERNEL but lower priority

GFP_NOWAIT     // Like GFP_ATOMIC but without emergency reserves
               // Less impact on system but more likely to fail

GFP_NOIO       // Cannot initiate I/O to reclaim memory
               // Use in block driver paths to prevent deadlocks

GFP_NOFS       // Cannot call into filesystem to reclaim memory
               // Use in filesystem driver code paths

GFP_HIGHUSER   // For user pages, prefer ZONE_HIGHMEM (32-bit only)
GFP_DMA        // Must come from ZONE_DMA (<16MB) — legacy 24-bit DMA
GFP_DMA32      // Must come from ZONE_DMA32 (<4GB) — for 32-bit DMA devices
```

### Modifier Flags (combine with primary)

```c
__GFP_ZERO        // Zero the memory before returning
__GFP_NOWARN      // Don't print warning if allocation fails
__GFP_RETRY_MAYFAIL // Try hard, but allow failure after many retries
__GFP_NOFAIL      // MUST succeed — loop until memory available
__GFP_HIGH        // Use emergency memory reserves
__GFP_IO          // Allow I/O operations for reclaim
__GFP_FS          // Allow filesystem operations for reclaim
__GFP_RECLAIM     // Allow both I/O and FS reclaim
__GFP_DIRECT_RECLAIM // Allow direct reclaim (caller blocks while pages freed)
__GFP_KSWAPD_RECLAIM // Allow kswapd to run for reclaim
```

### The Golden Rule for GFP Selection

```
Are you in interrupt context or holding a spinlock?
  YES → Use GFP_ATOMIC
  NO  → Use GFP_KERNEL

Are you in a filesystem/block I/O path?
  YES in FS code   → Use GFP_NOFS
  YES in block code → Use GFP_NOIO

Do you need memory for DMA with a 32-bit device?
  YES → Add __GFP_DMA32 or use GFP_DMA32

Do you need it zeroed?
  YES → Use kzalloc() instead of kmalloc() + memset()
```

### Checking Your Context

```c
#include <linux/preempt.h>
#include <linux/interrupt.h>

// Check if we're in interrupt context (can't sleep)
if (in_interrupt()) {
    // Must use GFP_ATOMIC
}

// Check if we're in atomic context (spinlock held, etc.)
if (in_atomic()) {
    // Must use GFP_ATOMIC
}

// Check if we're in a softirq
if (in_softirq()) {
    // Must use GFP_ATOMIC
}

// Check if we CAN sleep
if (in_task()) {
    // Can use GFP_KERNEL
}
```

---

## 9. Memory Pools — Pre-allocated Reserves

`mempool` provides a guaranteed minimum pool of memory. Even if the system is under extreme memory pressure, the pool can satisfy allocations.

```c
#include <linux/mempool.h>

// Create a pool with min_nr pre-allocated objects
mempool_t *mempool_create_kmalloc_pool(int min_nr, size_t size);
mempool_t *mempool_create_slab_pool(int min_nr, struct kmem_cache *kc);
mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn,
                          mempool_free_t *free_fn, void *pool_data);

// Allocate from pool (may use reserve if normal alloc fails)
void *mempool_alloc(mempool_t *pool, gfp_t gfp_mask);

// Free back to pool
void mempool_free(void *element, mempool_t *pool);

// Destroy pool
void mempool_destroy(mempool_t *pool);
```

### When to Use mempool

Mempools are for **critical paths** where allocation failure would be unacceptable:
- Block device drivers — can't fail I/O completions
- Network drivers — can't drop critical packets
- Error recovery paths

```c
// Example: block driver with guaranteed memory for I/O completions
struct my_driver {
    mempool_t *io_pool;
};

// In init:
dev->io_pool = mempool_create_kmalloc_pool(
    16,               // keep 16 objects reserved
    sizeof(struct my_io_ctx)
);

// In I/O path:
struct my_io_ctx *ctx = mempool_alloc(dev->io_pool, GFP_NOIO);
// ctx is guaranteed non-NULL (will block if necessary to get it)
```

---

## 10. Common Memory Bugs in Kernel Code

These bugs will be tested in interviews and will be your most frustrating debugging sessions.

### 1. Stack Overflow — Allocating Too Much on Kernel Stack

```c
// WRONG — 4KB allocation on 4KB-8KB kernel stack is a disaster
static int my_func(void)
{
    char buffer[4096];      // potentially overflows kernel stack!
    u32 lookup[1024];       // 4KB — also too large
    // ...
}

// RIGHT — use kmalloc for large buffers
static int my_func(void)
{
    char *buffer = kmalloc(4096, GFP_KERNEL);
    if (!buffer) return -ENOMEM;
    // ...
    kfree(buffer);
    return 0;
}
```

**Rule of thumb:** Keep kernel stack usage under 256 bytes per function. Check with `CONFIG_FRAME_WARN=256` and `checkstack.pl`.

### 2. Memory Leak

```c
// WRONG — buffer leaked if second allocation fails
int init(void)
{
    u8 *buf1 = kmalloc(256, GFP_KERNEL);
    u8 *buf2 = kmalloc(256, GFP_KERNEL);  // if this fails, buf1 is leaked!
    if (!buf2) return -ENOMEM;
    // ...
}

// RIGHT — free on every error path
int init(void)
{
    u8 *buf1 = kmalloc(256, GFP_KERNEL);
    if (!buf1) return -ENOMEM;
    
    u8 *buf2 = kmalloc(256, GFP_KERNEL);
    if (!buf2) {
        kfree(buf1);  // clean up
        return -ENOMEM;
    }
    // ...
}
```

**Detection:** Enable `CONFIG_KASAN` or use `kmemleak` (`CONFIG_DEBUG_KMEMLEAK`):
```bash
cat /sys/kernel/debug/kmemleak  # shows leaked objects
echo scan > /sys/kernel/debug/kmemleak  # trigger scan
```

### 3. Use After Free

```c
// WRONG
struct myobj *obj = kmalloc(sizeof(*obj), GFP_KERNEL);
kfree(obj);
obj->field = 5;  // use after free — crash or silent corruption

// RIGHT — set to NULL after free
kfree(obj);
obj = NULL;
// Now any accidental access → null pointer dereference (catchable)
```

**Detection:** `CONFIG_KASAN` catches this immediately with exact stack trace.

### 4. Double Free

```c
// WRONG
kfree(ptr);
kfree(ptr);  // corrupts slab cache metadata → eventual crash

// RIGHT — use kfree_and_null or set NULL
kfree(ptr);
ptr = NULL;
kfree(ptr);  // kfree(NULL) is always safe
```

### 5. Wrong GFP Flag in Atomic Context

```c
// WRONG — GFP_KERNEL in interrupt handler → BUG: sleeping function called
// from invalid context
static irqreturn_t my_irq(int irq, void *data)
{
    void *buf = kmalloc(64, GFP_KERNEL);  // BUG! Can't sleep in IRQ handler
    // ...
}

// RIGHT
static irqreturn_t my_irq(int irq, void *data)
{
    void *buf = kmalloc(64, GFP_ATOMIC);  // won't sleep
    if (!buf) return IRQ_HANDLED;  // handle failure gracefully
    // ...
}
```

**Detection:** `CONFIG_DEBUG_ATOMIC_SLEEP` will immediately print a warning and stack trace.

---

## 11. Hands-On Lab — Memory Demo Module in QEMU

### Step 1 — Write the Demo Module

```c
// mem_demo.c — demonstrates various kernel memory allocation APIs

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/slab.h>
#include <linux/vmalloc.h>
#include <linux/mm.h>

#define SMALL_SIZE   64
#define MEDIUM_SIZE  4096
#define LARGE_SIZE   (1024 * 1024)   // 1MB

static void *kmalloc_small;
static void *kmalloc_medium;
static void *vmalloc_large;
static struct page *raw_page;
static struct kmem_cache *my_cache;

struct myobj {
    u32 id;
    char name[28];
};

static int __init mem_demo_init(void)
{
    struct myobj *obj;
    
    pr_info("=== Kernel Memory Demo ===\n");
    
    // ---- kmalloc: small allocation ----
    kmalloc_small = kmalloc(SMALL_SIZE, GFP_KERNEL);
    if (!kmalloc_small) {
        pr_err("kmalloc small failed\n");
        return -ENOMEM;
    }
    pr_info("kmalloc(%d): virt=%px phys=%pa\n",
            SMALL_SIZE,
            kmalloc_small,
            &(phys_addr_t){virt_to_phys(kmalloc_small)});
    
    // kzalloc: zeroed allocation
    kmalloc_medium = kzalloc(MEDIUM_SIZE, GFP_KERNEL);
    if (!kmalloc_medium) {
        pr_err("kzalloc medium failed\n");
        goto err_medium;
    }
    pr_info("kzalloc(%d): virt=%px phys=%pa\n",
            MEDIUM_SIZE,
            kmalloc_medium,
            &(phys_addr_t){virt_to_phys(kmalloc_medium)});
    
    // Verify it's zeroed
    {
        u8 *p = kmalloc_medium;
        int i, nonzero = 0;
        for (i = 0; i < MEDIUM_SIZE; i++)
            if (p[i]) nonzero++;
        pr_info("kzalloc zero check: %s\n",
                nonzero ? "FAILED" : "PASSED (all zeros)");
    }
    
    // ---- vmalloc: large allocation ----
    vmalloc_large = vmalloc(LARGE_SIZE);
    if (!vmalloc_large) {
        pr_err("vmalloc large failed\n");
        goto err_large;
    }
    pr_info("vmalloc(%d KB): virt=%px\n",
            LARGE_SIZE/1024, vmalloc_large);
    
    // Show that vmalloc pages are NOT physically contiguous
    {
        unsigned long va = (unsigned long)vmalloc_large;
        unsigned long phys0 = PFN_PHYS(vmalloc_to_pfn((void *)va));
        unsigned long phys1 = PFN_PHYS(vmalloc_to_pfn((void *)(va + PAGE_SIZE)));
        pr_info("vmalloc page 0 phys: %lx\n", phys0);
        pr_info("vmalloc page 1 phys: %lx\n", phys1);
        pr_info("Pages contiguous: %s\n",
                (phys1 == phys0 + PAGE_SIZE) ? "YES (lucky)" : "NO (as expected)");
    }
    
    // ---- Raw page allocation ----
    raw_page = alloc_page(GFP_KERNEL);
    if (!raw_page) {
        pr_err("alloc_page failed\n");
        goto err_page;
    }
    pr_info("alloc_page: struct page=%px, pfn=%lu, virt=%px\n",
            raw_page,
            page_to_pfn(raw_page),
            page_address(raw_page));
    
    // ---- Custom slab cache ----
    my_cache = kmem_cache_create("myobj_cache",
                                  sizeof(struct myobj),
                                  0,
                                  SLAB_HWCACHE_ALIGN | SLAB_PANIC,
                                  NULL);
    pr_info("Custom cache 'myobj_cache' created, object size=%zu\n",
            sizeof(struct myobj));
    
    obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
    if (obj) {
        obj->id = 42;
        strscpy(obj->name, "test_object", sizeof(obj->name));
        pr_info("Cache alloc: id=%u name=%s\n", obj->id, obj->name);
        kmem_cache_free(my_cache, obj);
        pr_info("Cache free: OK\n");
    }
    
    // ---- Show /proc/meminfo snapshot ----
    pr_info("Check: cat /proc/meminfo | grep -E 'MemFree|Slab'\n");
    pr_info("Check: cat /proc/buddyinfo\n");
    pr_info("Check: sudo cat /sys/kernel/slab/myobj_cache/object_size\n");
    
    return 0;

err_page:
    vfree(vmalloc_large);
    vmalloc_large = NULL;
err_large:
    kfree(kmalloc_medium);
    kmalloc_medium = NULL;
err_medium:
    kfree(kmalloc_small);
    kmalloc_small = NULL;
    return -ENOMEM;
}

static void __exit mem_demo_exit(void)
{
    pr_info("=== Cleaning up memory demo ===\n");
    
    if (my_cache) {
        kmem_cache_destroy(my_cache);
        pr_info("Custom cache destroyed\n");
    }
    if (raw_page) {
        __free_page(raw_page);
        pr_info("Raw page freed\n");
    }
    if (vmalloc_large) {
        vfree(vmalloc_large);
        pr_info("vmalloc freed\n");
    }
    if (kmalloc_medium) {
        kfree(kmalloc_medium);
        pr_info("kzalloc medium freed\n");
    }
    if (kmalloc_small) {
        kfree(kmalloc_small);
        pr_info("kmalloc small freed\n");
    }
}

module_init(mem_demo_init);
module_exit(mem_demo_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Day 5: Kernel memory allocation demo");
```

### Step 2 — Build and Load in QEMU

```bash
# Build for x86 (native or cross-compile)
make ARCH=x86 KDIR=/lib/modules/$(uname -r)/build

# Load in QEMU (or natively if on x86)
sudo insmod mem_demo.ko
dmesg | tail -30

# Explore /proc and /sys
cat /proc/meminfo | grep -E 'MemFree|Slab'
cat /proc/buddyinfo
sudo cat /sys/kernel/slab/myobj_cache/object_size 2>/dev/null || \
    sudo cat /sys/kernel/slab/kmalloc-64/object_size

# Unload
sudo rmmod mem_demo
dmesg | tail -5
```

### Step 3 — Enable KASAN and Test Memory Bugs

```bash
# In QEMU kernel config, enable:
# CONFIG_KASAN=y
# CONFIG_KASAN_INLINE=y

# Write a module that deliberately does use-after-free:
cat > uaf_demo.c << 'EOF'
#include <linux/module.h>
#include <linux/slab.h>

static int __init uaf_init(void)
{
    char *p = kmalloc(64, GFP_KERNEL);
    kfree(p);
    p[0] = 'X';  // use after free — KASAN should catch this
    return 0;
}
module_init(uaf_init);
MODULE_LICENSE("GPL");
EOF

# Build and load — KASAN will print a detailed report
sudo insmod uaf_demo.ko
dmesg | grep -A 30 "BUG: KASAN"
```

---

## 12. Interview Questions

---

**Q1. What is the difference between `kmalloc` and `vmalloc`? When would you use each in a driver?**

**Answer:** `kmalloc()` allocates physically **and** virtually contiguous memory from the kernel's slab allocator. It's fast, uses a small amount of memory overhead, and is safe for DMA because the physical address is linear. Size is limited to approximately 4MB. Use it for: small/medium structures, DMA buffers, per-device state.

`vmalloc()` allocates virtually contiguous but potentially physically scattered memory. It allocates full pages and sets up page table entries mapping the virtual range. It's slower (due to PTE setup) and cannot be used for DMA (physical pages are scattered). But it can allocate much larger blocks. Use it for: loading firmware images, large lookup tables, large software buffers where DMA is not needed.

The critical distinction: if hardware needs to DMA into your buffer, you must use `kmalloc()` (or `dma_alloc_coherent()` for large DMA buffers). If you just need a large kernel-side buffer for software use, `vmalloc()` is appropriate.

---

**Q2. Why is `GFP_ATOMIC` needed in interrupt handlers? What does it actually do differently than `GFP_KERNEL`?**

**Answer:** `GFP_KERNEL` allows the allocator to sleep if memory is tight — it may call the VM subsystem to reclaim pages (by writing dirty pages to disk, swapping out pages, etc.). This can take milliseconds. In an interrupt handler, sleeping is absolutely forbidden: the handler runs with interrupts disabled (or at a hardware interrupt level), and `schedule()` cannot be called from interrupt context. Attempting `GFP_KERNEL` in an interrupt handler will trigger: "BUG: sleeping function called from invalid context."

`GFP_ATOMIC` means: allocate immediately or fail. It doesn't call back into the VM subsystem. It draws from emergency reserves (a small pool of pages pre-reserved for atomic contexts). If even those are exhausted, it returns `NULL` immediately.

Practical difference: `GFP_KERNEL` has a much higher success rate under memory pressure; `GFP_ATOMIC` may fail under pressure. For interrupt handlers, you should pre-allocate in your driver's initialization (with `GFP_KERNEL`) and use mempool in the interrupt path, rather than allocating with `GFP_ATOMIC` where possible.

---

**Q3. What is the slab allocator and what problem does it solve over the page allocator?**

**Answer:** The buddy (page) allocator works in units of 4KB pages. If you need a 64-byte `struct`, allocating a full page wastes 4032 bytes. The slab allocator solves this by:

1. **Object caching**: Pre-allocates slabs (groups of pages) and divides them into fixed-size objects. A 64-byte slab cache holds 64 objects per 4KB page, wasting nothing.

2. **Fast allocation/deallocation**: Getting an object from a slab cache is often just a pointer dereference (the free list head). No expensive buddy system traversal needed.

3. **Cache coloring**: Offsets objects so that frequently used objects from different allocations don't compete for the same CPU cache lines.

4. **Per-CPU caches**: SLUB maintains per-CPU free lists to avoid locks in the common case.

5. **Constructor support**: Custom caches can have a constructor function called once when new objects are carved from slabs, so objects are always in a known initial state.

The result: `kmalloc(64, GFP_KERNEL)` returns in nanoseconds on a warm cache. The same allocation with the page allocator would take much longer and waste 63x more memory.

---

**Q4. What is `virt_to_phys()` and when would you use it in a driver?**

**Answer:** `virt_to_phys(addr)` converts a kernel virtual address (in the kernel's direct linear map) to its corresponding physical address. It's essentially: `phys = virt - PAGE_OFFSET`.

Use cases in drivers:
1. **DMA setup**: When you `kmalloc()` a buffer and need to program a DMA controller with its physical address. The DMA controller uses physical addresses (or bus addresses), not virtual addresses.
   ```c
   void *buf = kmalloc(4096, GFP_KERNEL);
   dma_addr_t phys = virt_to_phys(buf);  // program into DMA controller
   ```
   Note: In modern code, use the DMA API (`dma_map_single()`) instead — it handles IOMMU translations and cache coherency automatically.

2. **Debugging**: Printing physical addresses for comparison with hardware documentation or `cat /proc/iomem`.

**Limitations**: Only valid for addresses in the kernel linear map (kmalloc, alloc_page, etc.). It does NOT work for vmalloc addresses (which need `vmalloc_to_pfn()`), or user-space addresses.

---

**Q5. What is KASAN and how does it detect use-after-free bugs?**

**Answer:** KASAN (Kernel Address Sanitizer) is a dynamic memory error detector (`CONFIG_KASAN=y`). It catches:
- Use-after-free
- Out-of-bounds accesses (heap and stack)
- Use-after-scope
- Global variable out-of-bounds

**How it works (Shadow Memory approach)**: KASAN dedicates 1/8th of the kernel's virtual address space as "shadow memory." Each byte in shadow memory corresponds to 8 bytes of real memory and encodes the state (accessible, freed, padding, etc.).

For every memory access, the compiler (via KASAN instrumentation) inserts a check:
```c
// For an 8-byte access at addr:
char *shadow = (char *)((unsigned long)addr >> 3) + KASAN_SHADOW_OFFSET;
if (*shadow != 0) {
    kasan_report(addr, size, is_write, __builtin_return_address(0));
}
```

When `kfree()` is called, KASAN poisons the shadow memory for that allocation (marks it as freed). Any subsequent access hits the shadow check, triggers `kasan_report()`, and prints a detailed report with:
- The exact address accessed
- Stack trace of the access
- Stack trace of when the memory was freed
- Stack trace of when it was allocated

This makes use-after-free bugs trivially findable — bugs that would otherwise only manifest as random crashes hours later.

---

**Q6. Explain the buddy system allocator. What is the "order" of an allocation?**

**Answer:** The buddy system allocates physical memory in power-of-2 page blocks. The "order" is the exponent: order 0 = 1 page (4KB), order 1 = 2 pages (8KB), order n = 2^n pages.

The allocator maintains free lists for each order. When allocating order `n`:
1. Check the order-n free list
2. If empty, split an order-(n+1) block into two order-n "buddies"
3. Return one buddy, put the other on the order-n free list
4. Repeat recursively upward if needed

When freeing an order-n block:
1. Check if the buddy (the adjacent block of the same size, at address XOR `(n * PAGE_SIZE)`) is also free
2. If yes: merge → form an order-(n+1) block, check that block's buddy
3. Repeat until no buddy is free or max order reached
4. Add the final block to the appropriate free list

The name "buddy" comes from the bit-flip relationship: a buddy of a block at address `A` with order `n` is at address `A XOR (PAGE_SIZE << n)`. Two blocks are buddies if and only if their addresses differ only in bit `n`.

Fragmentation is limited because only same-sized blocks merge. `/proc/buddyinfo` shows the distribution of free blocks at each order for each memory zone.

---

> **Next:** Day 6 — Kernel Synchronization: spinlocks, mutexes, RCU, and atomic operations. Intentionally triggering race conditions on RPi4.
