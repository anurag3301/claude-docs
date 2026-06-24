# Day 23 — Physical Memory Model

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand how the kernel tracks every physical page of RAM — `struct page`, memory zones, NUMA nodes, and the memory model itself.

---

## 1. The Fundamental Problem

The kernel must track every physical page of RAM simultaneously:
- Is this page free or in use?
- Who owns it (which process, which kernel subsystem)?
- What state is it in (dirty, locked, referenced)?
- Which NUMA node does it belong to?

On a machine with 64GB RAM: 64GB / 4KB = 16,777,216 physical pages to track.

The data structure that tracks a single physical page is `struct page`.

---

## 2. `struct page` — The Page Descriptor

`struct page` is one of the most carefully optimized structures in the kernel — it must be as small as possible because there's one per physical page.

```c
// include/linux/mm_types.h
// Current size: 64 bytes on x86_64 (exactly one cache line)
// For 64GB RAM: 64 bytes × 16M pages = 1GB overhead

struct page {
    unsigned long flags;        // Atomic bitfield — see PG_* below
    
    union {
        // For pages in the page cache (file-backed):
        struct {
            struct list_head lru;      // LRU list (active/inactive)
            struct address_space *mapping;  // Which file owns this page
            pgoff_t index;             // Offset within the file (in pages)
            unsigned long private;     // For buffer_heads or private data
        };
        
        // For slab/SLUB allocator pages:
        struct {
            union {
                struct list_head slab_list;  // List of full/partial slabs
                struct {
                    struct page *next;       // Next partial slab
                    int pages;               // Number of pages in slab
                    int pobjects;            // Approximate # of objects
                };
            };
            struct kmem_cache *slab_cache;  // Which cache this belongs to
            void *freelist;                 // First free object in slab
            union {
                unsigned long counters;
                struct {
                    unsigned inuse:16;      // # of used objects
                    unsigned objects:15;    // Total objects in slab
                    unsigned frozen:1;      // On per-CPU freelist?
                };
            };
        };
        
        // For compound pages (huge pages, THP):
        struct {
            unsigned long compound_head;    // Head page pointer (bit 0 = set means tail)
            unsigned char compound_dtor;    // Destructor type
            unsigned char compound_order;  // log2 of page count
            atomic_t compound_mapcount;    // How many PTEs map this compound page
        };
        
        // For page table pages:
        struct {
            unsigned long _pt_pad_1;
            pgtable_t pmd_huge_pte;
            unsigned long _pt_pad_2;
            union {
                struct mm_struct *pt_mm;    // Which mm owns this page table
                atomic_t pt_frag_refcount;
            };
            spinlock_t ptl;                 // Page table lock
        };
        
        // For unused pages (in free lists):
        struct {
            struct list_head buddy_list;   // Free list in buddy allocator
            unsigned long buddy_order;     // Order of the free block
        };
    };
    
    // Reference counting:
    atomic_t _refcount;    // How many references to this page
                            // 0 = free, >0 = in use
    
    // For mapped pages:
    atomic_t _mapcount;    // How many PTEs map this page
                            // -1 = not mapped, 0 = mapped once, N = N+1 times
    
#ifdef CONFIG_MEMCG
    unsigned long memcg_data;    // Memory cgroup accounting
#endif
};
```

### 2.1 Key Page Flags (`PG_*`)

```c
// include/linux/page-flags.h
// All accessed atomically via test_bit/set_bit/clear_bit on flags:

PG_locked       // Page is locked (I/O in progress)
PG_referenced   // Page has been recently accessed
PG_uptodate     // Page data is valid (read from disk completed)
PG_dirty        // Page has been modified (needs writeback to disk)
PG_lru          // Page is on an LRU list
PG_active       // Page is on the active LRU list
PG_slab         // Page is managed by slab allocator
PG_reserved     // Page cannot be freed (kernel or firmware use)
PG_private      // page->private has valid data (used by filesystems)
PG_writeback    // Page is being written to disk right now
PG_head         // This is the head page of a compound page
PG_mappedtodisk // Has blocks allocated on disk
PG_reclaim      // Page is being reclaimed
PG_swapbacked   // Page is backed by swap (anonymous)
PG_unevictable  // Page cannot be evicted (mlock'd, ramfs, etc.)
PG_mlocked      // Page is mlock()'d in memory
PG_hwpoison     // Hardware detected memory error on this page

// Accessor macros:
PageLocked(page)       // Test PG_locked
SetPageDirty(page)     // Set PG_dirty
ClearPageUptodate(page) // Clear PG_uptodate
TestSetPageLocked(page) // Atomic test-and-set
TestClearPageActive(page) // Atomic test-and-clear
```

```bash
# Decode page flags for a specific physical page (via /proc/kpageflags):
# Physical address 0x100000 → page frame number (PFN) = 0x100000 / 0x1000 = 256
python3 << 'EOF'
import struct
PFN = 256
with open('/proc/kpageflags', 'rb') as f:
    f.seek(PFN * 8)
    flags = struct.unpack('Q', f.read(8))[0]
names = ['LOCKED','ERROR','REFERENCED','UPTODATE','DIRTY','LRU','ACTIVE',
         'SLAB','WRITEBACK','RECLAIM','BUDDY','MMAP','ANON','SWAPCACHE',
         'SWAPBACKED','COMPOUND_HEAD','COMPOUND_TAIL','HUGE','UNEVICTABLE',
         'HWPOISON','NOPAGE','KSM','THP','OFFLINE','ZERO_PAGE','IDLE','PGTABLE']
print(f'PFN {PFN} flags: {[names[i] for i in range(len(names)) if flags & (1<<i)]}')
EOF
```

### 2.2 `_refcount` vs `_mapcount`

```c
// _refcount: general reference counter
// >0 = someone holds a reference to this page
// Incremented by: get_page(), page cache lookup, slab, DMA
// Decremented by: put_page()
// When hits 0: page is returned to allocator

// _mapcount: how many page table entries point to this page
// -1 = not mapped in any page table
//  0 = mapped in exactly one page table (one PTE points here)
//  N = mapped in N+1 page tables (shared/COW pages)

// Example:
// After fork(): both parent and child have PTEs pointing to the same page
// _mapcount = 1 (two PTEs)
// After COW fault in parent: new page allocated, _mapcount drops to 0 for old page

// Accessing:
int refcount = page_ref_count(page);   // _refcount
int mapcount = page_mapcount(page);    // _mapcount (= _mapcount + 1)
```

---

## 3. Memory Zones

Not all RAM is equal. The kernel divides physical memory into **zones** based on hardware capabilities:

```c
// include/linux/mmzone.h
enum zone_type {
#ifdef CONFIG_ZONE_DMA
    ZONE_DMA,        // Low memory: 0–16MB
                     // For ISA DMA devices limited to 24-bit addresses
#endif
#ifdef CONFIG_ZONE_DMA32
    ZONE_DMA32,      // 0–4GB
                     // For 32-bit DMA devices (PCIe DMA without 64-bit support)
#endif
    ZONE_NORMAL,     // Directly mapped in kernel address space
                     // The main zone on 64-bit systems
#ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,    // 32-bit only: >896MB that isn't permanently mapped
                     // Essentially unused on 64-bit kernels
#endif
    ZONE_MOVABLE,    // Pages that can be migrated (for memory hotplug)
    ZONE_DEVICE,     // Persistent memory (NVDIMMs), GPU memory
    __MAX_NR_ZONES
};
```

### 3.1 Zone Structure

```c
// include/linux/mmzone.h
struct zone {
    // Watermarks: thresholds for reclaim
    unsigned long _watermark[NR_WMARK];
    // WMARK_MIN: emergency threshold (start direct reclaim)
    // WMARK_LOW: background reclaim (wake kswapd)
    // WMARK_HIGH: zone is balanced (stop reclaim)
    
    unsigned long watermark_boost;  // Boost watermarks after fragmentation
    
    long lowmem_reserve[MAX_NR_ZONES];  // Reserve for lower zones
    
    int node;              // NUMA node this zone is on
    
    struct pglist_data *zone_pgdat;  // Pointer to parent NUMA node
    
    struct per_cpu_pages __percpu *per_cpu_pageset; // Per-CPU page cache
    
    unsigned long zone_start_pfn;   // First page frame in this zone
    
    // Buddy allocator free lists:
    struct free_area free_area[MAX_ORDER + 1];  // MAX_ORDER = 10
    // free_area[0] = list of free single pages
    // free_area[1] = list of free 2-page blocks
    // free_area[10] = list of free 1024-page (4MB) blocks
    
    unsigned long flags;   // ZONE_BOOSTED_WATERMARK, etc.
    
    spinlock_t lock;       // Protects free_area lists
    
    // Statistics (per-zone counters):
    atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
    // NR_FREE_PAGES, NR_ZONE_LRU_BASE, NR_ZONE_ACTIVE_ANON, etc.
    
    const char *name;     // "DMA", "DMA32", "Normal", "Movable"
};
```

```bash
# Inspect zones:
cat /proc/zoneinfo
# Node 0, zone      DMA
#   pages free     3992
#         min      8
#         low      10
#         high     12
#         ...
#   nr_free_pages  3992
#   nr_zone_inactive_anon  0
#   nr_zone_active_anon    0
#   nr_zone_inactive_file  0
#   nr_zone_active_file    0
#
# Node 0, zone    DMA32
# Node 0, zone   Normal    ← This is the big one

# Quick zone summary:
cat /proc/buddyinfo
# Node 0, zone      DMA      1  1  0  0  0  0  0  0  0  0  0
# Node 0, zone    DMA32      4  3  2  1  1  1  0  0  1  0  0
# Node 0, zone   Normal   1234  567  89  ...
# Column N = number of free blocks of order N
```

---

## 4. NUMA Node: `pglist_data`

On NUMA systems, each memory node has a `pglist_data`:

```c
// include/linux/mmzone.h
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];  // Zones in this node
    struct zonelist node_zonelists[MAX_ZONELISTS];  // Fallback zone ordering
    int nr_zones;                          // Number of zones in this node
    
    unsigned long node_start_pfn;          // First PFN in this node
    unsigned long node_present_pages;      // Total usable pages
    unsigned long node_spanned_pages;      // Total pages including holes
    int node_id;                           // NUMA node number
    
    wait_queue_head_t kswapd_wait;         // kswapd waits here
    wait_queue_head_t pfmemalloc_wait;     // Emergency alloc wait queue
    struct task_struct *kswapd;            // Per-node kswapd thread
    int kswapd_order;                      // Target order for kswapd
    
    // LRU lists for this node:
    struct lruvec __lruvec;               // Active/inactive page lists
    
    unsigned long flags;                  // N_POSSIBLE, N_ONLINE, etc.
    
    struct per_cpu_nodestat __percpu *per_cpu_nodestats;
    atomic_long_t vm_stat[NR_VM_NODE_STAT_ITEMS];
    
} pg_data_t;

// Access:
NODE_DATA(nid)        // pg_data_t* for NUMA node nid
NODE_DATA(0)          // Node 0's pglist_data
```

```bash
# NUMA topology:
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 32752 MB
# node 0 free: 25431 MB
# node 1 cpus: 8 9 10 11 12 13 14 15
# node 1 size: 32768 MB
# node 1 free: 30123 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# Per-node memory stats:
numastat
# node0           node1
# numa_hit         1234567         987654
# numa_miss          1234             567    ← memory allocated on wrong node
# interleave_hit       12              8
# local_node       1234567         987654
# other_node          1234            567
```

---

## 5. The `mem_map` Array — Physical Memory Map

The physical memory map is an array of `struct page`, one entry per physical page frame:

```c
// On flat memory model:
// extern struct page *mem_map;  // Base of the mem_map array
// struct page *page = mem_map + pfn;  // Page N is at mem_map[N]

// On sparse memory model (modern):
struct page *pfn_to_page(unsigned long pfn)
{
    // Uses a two-level section table:
    unsigned long section_nr = pfn_to_section_nr(pfn);
    struct mem_section *ms = __pfn_to_section(pfn);
    return ms->section_mem_map + (pfn & ~SECTION_MASK);
}
```

```bash
# The mem_map lives in the kernel address space at:
# 0xFFFFEA0000000000 (see Day 8)
# Size = NR_PAGES × sizeof(struct page) = NR_PAGES × 64 bytes

# Actual size on your system:
python3 -c "
import os
meminfo = open('/proc/meminfo').read()
total_kb = int([l for l in meminfo.split('\n') if 'MemTotal' in l][0].split()[1])
pages = total_kb // 4
overhead_mb = pages * 64 / 1024 / 1024
print(f'Total pages: {pages:,}')
print(f'mem_map overhead: {overhead_mb:.1f} MB')
"
```

---

## 6. PFN — Page Frame Number

The PFN (Page Frame Number) is the fundamental way to identify a physical page:

```c
// PFN = physical address >> PAGE_SHIFT
// PAGE_SHIFT = 12 on all architectures (4KB pages)

// Conversions:
unsigned long pfn = phys_addr >> PAGE_SHIFT;     // physical → PFN
phys_addr_t pa = (phys_addr_t)pfn << PAGE_SHIFT; // PFN → physical

// Between pfn and struct page:
struct page *page = pfn_to_page(pfn);           // PFN → page descriptor
unsigned long pfn = page_to_pfn(page);          // page descriptor → PFN

// Between pfn and kernel virtual address:
void *kaddr = phys_to_virt(pfn << PAGE_SHIFT);  // PFN → kernel virt
void *kaddr = page_address(page);               // page → kernel virt

// Between kernel virtual and physical:
phys_addr_t pa = virt_to_phys(kaddr);           // kernel virt → phys
void *kaddr = phys_to_virt(pa);                 // phys → kernel virt

// These only work for directly-mapped kernel addresses (not vmalloc)!
```

---

## 7. Memory Model Implementations

### 7.1 Flat Memory (`CONFIG_FLATMEM`)

Single contiguous `mem_map[]` array. Simple but wastes memory if there are holes in physical address space (PCI holes, etc.):

```c
// mem_map is a simple array:
struct page *mem_map;  // Allocated at boot
// page N = mem_map[N]
```

### 7.2 Sparse Memory (`CONFIG_SPARSEMEM`)

Physical memory is divided into 128MB sections. Only existing sections have `struct page` arrays:

```c
#define SECTION_SIZE_BITS 27    // 128MB sections
#define SECTION_SHIFT     27
#define PAGES_PER_SECTION (1UL << (SECTION_SHIFT - PAGE_SHIFT))  // 32768 pages

struct mem_section {
    unsigned long section_mem_map;  // Base of struct page array for this section
    struct mem_section_usage *usage;
};

// Global section table:
extern struct mem_section mem_section[NR_MEM_SECTIONS][SECTIONS_PER_ROOT];
```

Sparse memory supports:
- **Memory hotplug**: add/remove 128MB sections at runtime
- **NUMA**: different nodes have different physical addresses
- **Large physical address spaces with holes**: no memory wasted for non-existent sections

### 7.3 VMEMMAP (`CONFIG_SPARSEMEM_VMEMMAP`)

The modern approach: `struct page` arrays for all sections are mapped into a contiguous virtual address range (`VMEMMAP_START`). This gives flat-memory-like O(1) `pfn_to_page()` performance while supporting sparse physical memory:

```c
// pfn_to_page is just pointer arithmetic:
#define pfn_to_page(pfn)  (vmemmap + (pfn))
// vmemmap = VMEMMAP_START virtual address
// This works because vmemmap is virtually contiguous even if physical is sparse
```

---

## 8. Page Reference Counting

```c
// Getting and releasing page references:
get_page(page);     // Increment _refcount (atomic)
put_page(page);     // Decrement; if hits 0: call page destructor

// Safe version that won't increment a page with 0 refs:
bool success = get_page_unless_zero(page);  // Returns false if already 0

// Direct refcount manipulation:
page_ref_inc(page);             // ++_refcount
page_ref_dec(page);             // --_refcount  
int ref = page_ref_count(page); // Read _refcount
page_ref_add(page, nr);         // += nr

// Check if this is the last reference:
bool last = put_page_testzero(page);  // Decrements; returns true if now 0
```

---

## 9. Zone Lists and Fallback

When the preferred zone is out of memory, the allocator falls back to other zones via **zone lists**:

```c
// include/linux/mmzone.h
struct zonelist {
    struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};

// Fallback order on a 64-bit single-node system:
// GFP_KERNEL → Normal → DMA32 → DMA
//
// On NUMA:
// GFP_KERNEL, node=0 → Normal(node0) → Normal(node1) → DMA32(node0) → ...

// Watermark-based zone selection:
// Each zone has three watermarks (min, low, high):
// - If free_pages < WMARK_MIN: in critical zone, only emergency allocs
// - If free_pages < WMARK_LOW: wake kswapd (background reclaim)
// - If free_pages >= WMARK_HIGH: zone is healthy
```

```bash
# View watermarks:
cat /proc/zoneinfo | grep -A4 "Node 0, zone   Normal"
# pages free     123456
#       min      1234
#       low      1543
#       high     1852

# Adjust watermarks:
# vm.min_free_kbytes controls WMARK_MIN; others derived from it
cat /proc/sys/vm/min_free_kbytes
# 67584 (67MB on a large system)
echo 131072 > /proc/sys/vm/min_free_kbytes  # Raise to 128MB
```

---

## 10. Per-CPU Page Sets

To avoid lock contention on the zone's `free_area` lists for single-page allocations, each CPU has a cache of pre-allocated pages:

```c
// include/linux/mmzone.h
struct per_cpu_pages {
    spinlock_t lock;
    int count;           // Number of pages in this per-CPU cache
    int high;            // High watermark: drain to zone when count > high
    int batch;           // Batch size for refill/drain
    
    /* Lists of pages, one per migration type: */
    struct list_head lists[NR_PCP_LISTS];
};
```

```c
// Allocating a single page:
// Fast path: take from per-CPU cache (no zone lock needed!)
// Slow path: refill per-CPU cache from zone (takes zone->lock)

// Freeing a single page:
// Fast path: return to per-CPU cache
// Slow path: drain per-CPU cache to zone when count > high

// This eliminates lock contention for the most common case (single-page allocs)
```

---

## 11. Observing Physical Memory

```bash
# Overall memory layout:
cat /proc/meminfo              # High-level summary
cat /proc/iomem               # Physical address space map (RAM + MMIO)

# Detailed per-page info:
# /proc/kpagemap: maps virtual pages to physical PFNs
# /proc/kpagecount: reference count per PFN
# /proc/kpageflags: flags per PFN (PG_dirty, PG_uptodate, etc.)

# Example: decode pages of current process
python3 << 'EOF'
import struct, os
pid = os.getpid()
maps = open(f'/proc/{pid}/maps').readlines()
pagemap = open(f'/proc/{pid}/pagemap', 'rb')
kpageflags = open('/proc/kpageflags', 'rb')

for line in maps[:3]:  # First 3 VMAs
    parts = line.split()
    start, end = [int(x, 16) for x in parts[0].split('-')]
    print(f"\nVMA: {parts[0]} {parts[1]}")
    
    # Read pagemap entries for first page
    page_start = start >> 12
    pagemap.seek(page_start * 8)
    entry = struct.unpack('Q', pagemap.read(8))[0]
    
    if entry & (1 << 63):  # Page present
        pfn = entry & 0x7FFFFFFFFFFFFF
        kpageflags.seek(pfn * 8)
        flags = struct.unpack('Q', kpageflags.read(8))[0]
        print(f"  PFN: {pfn:#x}, flags: {flags:#x}")
EOF

# Show physical memory zones and their sizes:
dmesg | grep -E "^.*Zone ranges:|^.*DMA|^.*Normal|^.*Movable" | head -10
# Zone ranges:
#   DMA      [mem 0x0000000000001000-0x0000000000ffffff]
#   DMA32    [mem 0x0000000001000000-0x00000000ffffffff]
#   Normal   [mem 0x0000000100000000-0x000000083fffffff]
#   Movable  empty
```

---

## 12. Memory Hot-Plug

Linux supports adding/removing RAM at runtime (used in cloud VMs, mainframe LPARs):

```bash
# Check if memory hotplug is supported:
ls /sys/devices/system/memory/

# Each memory block (128MB):
ls /sys/devices/system/memory/memory0/
# online  phys_device  phys_index  removable  state  uevent  valid_zones

# Take a memory block offline (remove from use):
echo offline > /sys/devices/system/memory/memory100/state

# Bring it back online:
echo online > /sys/devices/system/memory/memory100/state

# The kernel uses ZONE_MOVABLE for movable memory:
# Pages in ZONE_MOVABLE can be migrated, so blocks can be freed
echo online_movable > /sys/devices/system/memory/memory100/state
```

---

## 13. Memory Compaction

Over time, physical memory becomes fragmented — free pages are scattered rather than contiguous. This prevents large (high-order) allocations. **Compaction** migrates movable pages to consolidate free space:

```bash
# Trigger compaction manually:
echo 1 > /proc/sys/vm/compact_memory

# Compaction runs automatically when:
# - High-order allocation fails
# - kcompactd daemon runs periodically

# Monitor compaction:
cat /proc/vmstat | grep compact
# compact_migrate_scanned  123456  ← Pages scanned for migration
# compact_free_scanned     234567  ← Free pages found
# compact_isolated         12345   ← Pages isolated for migration
# compact_stall            1234    ← Direct compaction waited (bad!)
# compact_success          1000    ← Successful compaction runs

# See fragmentation index (0=low fragmentation, 1000=max fragmentation):
cat /sys/kernel/debug/extfrag/extfrag_index
```

---

## 14. Mental Model Checkpoint

After Day 23, you should be able to:

1. What is `struct page` and how large is it? Why does that size matter?
2. Name the major fields of `struct page` and which usage context each belongs to.
3. What is the difference between `_refcount` and `_mapcount`?
4. Name the four standard memory zones and what constraint defines each.
5. What is a PFN? Convert physical address 0x40000000 to a PFN.
6. What is the difference between `FLATMEM` and `SPARSEMEM_VMEMMAP`?
7. What are zone watermarks and what do they trigger?
8. What are per-CPU page sets and why do they exist?

---

## Key Source Files

```bash
include/linux/mm_types.h        # struct page (THE most important data structure)
include/linux/mmzone.h          # struct zone, pglist_data, free_area, watermarks
include/linux/page-flags.h      # PG_* flags and accessor macros
mm/page_alloc.c                 # Zone initialization, watermarks, per_cpu_pages
mm/sparse.c                     # Sparse memory model implementation
include/linux/memory.h          # Memory hotplug infrastructure
mm/compaction.c                 # Memory compaction
Documentation/mm/physical_memory.rst  # Official documentation
```

---

## Summary

Physical memory management starts with a single fundamental fact: every physical page has exactly one `struct page` descriptor tracking its state. These descriptors are organized into:

- **Memory zones** (`ZONE_DMA`, `ZONE_DMA32`, `ZONE_NORMAL`, `ZONE_MOVABLE`): hardware-capability-based groupings
- **NUMA nodes** (`pglist_data`): topology-based groupings for locality
- **`mem_map` / vmemmap**: the global array of `struct page`, addressed by PFN

Zone watermarks govern when the system reclaims memory: `WMARK_LOW` wakes `kswapd`, `WMARK_MIN` triggers direct reclaim. Per-CPU page caches avoid lock contention for single-page allocations.

Tomorrow: the buddy allocator — how the kernel allocates and frees physically contiguous pages.
