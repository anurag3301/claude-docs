# Day 8 — Kernel Memory Layout

> **Estimated read time:** 90–120 minutes  
> **Kernel version referenced:** 6.x (x86_64 unless noted)  
> **Goal:** Understand exactly how the kernel organizes virtual and physical memory — zones, regions, allocator domains, and KASLR.

---

## 1. Two Kinds of Memory to Track

Always keep these separate in your head:

- **Physical memory**: actual DRAM chips. Addresses start at 0, go up to however much RAM the machine has. The CPU accesses this via the memory bus. The kernel needs to manage *which physical pages are in use* and *which are free*.
- **Virtual memory**: the 64-bit address space that software (including the kernel) actually uses. Every access goes through the MMU which translates virtual → physical. The kernel has its own virtual address space, distinct from each process's.

The kernel manages both simultaneously. It tracks physical pages with `struct page`. It maps those physical pages into virtual addresses for both itself and user processes.

---

## 2. Physical Address Space Layout (x86_64)

When the system boots, the firmware (UEFI/BIOS) provides a memory map (e820) describing which physical address ranges are usable RAM, reserved, MMIO, etc.

```
Physical Address Space (example 16GB system):
0x0000000000000000  ── RAM starts
0x000000000009F000  ── BIOS data area (reserved)
0x00000000000A0000  ── Legacy VGA/ROM area (reserved)
0x0000000000100000  ── RAM resumes (1MB mark)
      ...
0x00000000BFFF0000  ── ACPI tables (reserved by firmware)
0x00000000C0000000  ── PCI device BARs (MMIO) — not RAM!
0x00000000FFFFFFFF  ── End of 32-bit address space
0x0000000100000000  ── RAM above 4GB ("high memory" on older systems)
      ...
0x000000040FFFFFFF  ── End of 16GB RAM
0x000000FD00000000  ── NVMe/PCIe MMIO BARs (high MMIO)
0xFFFFFFFFFFFFFFFF  ── End of 64-bit physical address space
```

The kernel learns this from the e820 map and calls `e820__memory_setup()` in `setup_arch()`. Only "usable" ranges go to the buddy allocator.

---

## 3. Virtual Address Space Layout (x86_64, 4-level paging)

The 64-bit virtual address space is 2^64 bytes — but x86_64 only uses 48 bits of it (with 4-level paging), giving 256 TB. With 5-level paging (`CONFIG_X86_5LEVEL`), it's 57 bits = 128 PB.

The space is split exactly in half between user and kernel by the "canonical" hole:

```
Virtual Address Space (48-bit / 4-level paging):

0x0000000000000000 ┐
                   │  User space: 128 TB
                   │  Each process gets its own mapping here.
0x00007FFFFFFFFFFF ┘

    ~~~~ non-canonical hole (addresses 0x0000800000000000 to 0xFFFF7FFFFFFFFFFF) ~~~~
    Any access here causes instant #GP fault — used as a security feature.

0xFFFF800000000000 ┐
                   │  Kernel space: 128 TB
                   │  Same virtual layout in ALL processes (pre-KPTI).
0xFFFFFFFFFFFFFFFF ┘
```

The kernel's 128 TB is subdivided into well-defined regions. These are fixed at compile time but randomized within ranges by KASLR.

---

## 4. Kernel Virtual Address Map (x86_64 Detailed)

```
Kernel Virtual Address Space (128 TB):

0xFFFF800000000000 ── Guard hole (prevents overflow from user space)
      8 TB reserved

0xFFFF880000000000 ── Direct map of all physical memory
      64 TB max
      The ENTIRE physical RAM is mapped here linearly.
      Physical addr P → virtual addr 0xFFFF880000000000 + P
      Called: "direct mapping" or "physmap"
      Used by: kmalloc, kzalloc, DMA buffers

0xFFFFC80000000000 ── Guard hole

0xFFFFC90000000000 ── vmalloc/ioremap region
      32 TB
      Non-contiguous virtual memory allocations (vmalloc)
      MMIO device register mapping (ioremap)

0xFFFFE80000000000 ── Guard hole

0xFFFFEA0000000000 ── Virtual memory map (struct page array)
      1 TB
      The "mem_map": array of struct page, one per physical page.
      Physical page N → &mem_map[N]

0xFFFFEC0000000000 ── Guard hole

0xFFFFEC0000000000 ── KASAN shadow memory (if CONFIG_KASAN=y)
      16 TB (1 byte tracks 8 bytes of main memory)

0xFFFFFF0000000000 ── Guard hole

0xFFFFFF5000000000 ── %esp fixup stacks

0xFFFFFF7FF0000000 ── ── unused hole

0xFFFFFFFF80000000 ── Kernel text (vmlinux .text, .data, .bss)
      1 GB max (this is where the kernel binary lives)
      Physical: 1MB–~50MB (the kernel is ~20–30MB)

0xFFFFFFFFC0000000 ── Module text/data
      up to 1.5 GB
      All .ko files live here (within ±2GB of kernel text
      so direct CALL instructions work)

0xFFFFFFFFFFFFFFFF ── End
```

You can verify this on a running system:

```bash
# See kernel virtual memory layout:
sudo cat /proc/iomem        # Physical memory map
sudo cat /proc/vmallocinfo  # vmalloc allocations

# See kernel symbols with addresses:
sudo cat /proc/kallsyms | head -20
# ffffffff81000000 T startup_64        ← kernel text start
# ffffffff81000030 T secondary_startup_64
# ...
# ffffffff82800000 T __init_begin      ← init section start

# Module addresses:
sudo cat /proc/modules
# e1000e 282624 0 - Live 0xffffffffc0000000
#                              ↑ module area
```

---

## 5. The Direct Map (`__va` / `__pa`)

The most important mapping: **all physical RAM is mapped linearly starting at `PAGE_OFFSET`** (`0xFFFF880000000000` on x86_64).

```c
// include/asm/page.h (x86_64)
#define PAGE_OFFSET  0xFFFF880000000000UL

// Physical → Virtual (for direct-mapped memory):
#define __va(x)  ((void *)((unsigned long)(x) + PAGE_OFFSET))

// Virtual → Physical (for direct-mapped memory):
#define __pa(x)  ((unsigned long)(x) - PAGE_OFFSET)

// Examples:
void *kptr = __va(0x1000000);    // Physical 16MB → kernel virtual
unsigned long phys = __pa(ptr);  // Kernel virtual → physical

// Used in practice:
// kmalloc() returns a virtual address in the direct map
// page_to_phys(page) = __pa(page_address(page))
```

This direct map means the kernel can access ANY physical memory by simply adding `PAGE_OFFSET`. No need to set up page table entries for each access — they're all pre-mapped.

**`struct page` and the mem_map:**

```c
// For each physical page there is exactly one struct page.
// The mem_map[] array at 0xFFFFEA0000000000 contains them all.

// Given a physical page frame number (PFN = physical addr >> PAGE_SHIFT):
struct page *page = pfn_to_page(pfn);    // PFN → struct page*
unsigned long pfn = page_to_pfn(page);  // struct page* → PFN
void *kaddr = page_address(page);        // struct page* → kernel virtual addr
struct page *page = virt_to_page(kaddr); // kernel virtual → struct page*
unsigned long phys = page_to_phys(page); // struct page* → physical addr
```

`struct page` is 64 bytes per page (4KB), so it costs:
- 1 GB RAM → 256 KB of `struct page` overhead
- 1 TB RAM → 256 MB of `struct page` overhead

---

## 6. Memory Zones

Not all physical RAM is equivalent. Devices with 24-bit DMA addresses can only access the first 16MB. The kernel splits RAM into **zones**:

```c
// include/linux/mmzone.h
enum zone_type {
    ZONE_DMA,        // 0–16MB: for legacy ISA DMA devices
    ZONE_DMA32,      // 0–4GB: for 32-bit DMA devices (e.g., older PCIe)
    ZONE_NORMAL,     // Directly mapped in kernel, main allocation zone
    ZONE_HIGHMEM,    // 32-bit only: RAM above 896MB not permanently mapped (OBSOLETE on 64-bit)
    ZONE_MOVABLE,    // Movable pages for memory hotplug
    ZONE_DEVICE,     // Persistent memory (PMEM), GPU memory
    __MAX_NR_ZONES
};
```

On modern 64-bit systems, `ZONE_HIGHMEM` is effectively unused (all RAM is directly mapped). You mainly deal with `ZONE_DMA`, `ZONE_DMA32`, and `ZONE_NORMAL`.

```bash
# See zone sizes and allocation counts:
cat /proc/zoneinfo | head -60

# cat /proc/buddyinfo — free pages by order per zone:
# Node 0, zone      DMA      1  1  0  0  2  1  1  0  1  1  3
# Node 0, zone    DMA32      0  0  1 ...
# Node 0, zone   Normal   1234  567  89 ...
#                              ↑ column = order (0=4KB, 1=8KB, ... 10=4MB)
```

---

## 7. NUMA — Non-Uniform Memory Access

On multi-socket systems, each CPU socket has RAM "near" it (low latency) and "far" (high latency through the interconnect).

```
NUMA Node 0:              NUMA Node 1:
  CPU 0-15                  CPU 16-31
  RAM 0-64GB                RAM 64-128GB
       ↕ (fast)                  ↕ (fast)
  [interconnect: slower cross-node access]
```

```c
// include/linux/mmzone.h
struct pglist_data {          // One per NUMA node
    struct zone node_zones[MAX_NR_ZONES];
    int nr_zones;
    unsigned long node_start_pfn;   // First PFN of this node
    unsigned long node_present_pages;
    int node_id;
    // ...
};

// Access:
NODE_DATA(0)     // pglist_data for NUMA node 0
NODE_DATA(nid)   // pglist_data for node 'nid'
```

```bash
# See NUMA topology:
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
# node 0 size: 64477 MB
# node 1 cpus: 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
# node 1 size: 65536 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# See per-node memory stats:
numastat
numastat -m  # per-process memory placement
```

The allocator always tries to allocate from the local NUMA node first. `GFP_KERNEL` will fall back to remote nodes if local is full.

---

## 8. KASLR — Kernel Address Space Layout Randomization

`CONFIG_RANDOMIZE_BASE=y` randomizes the kernel's load address at boot time. This makes it harder for attackers to use hardcoded kernel addresses.

```bash
# Without KASLR (nokaslr boot param or old kernel):
cat /proc/kallsyms | grep ' T startup_64'
# ffffffff81000000 T startup_64   ← Always the same

# With KASLR:
cat /proc/kallsyms | grep ' T startup_64'
# ffffffff8a000000 T startup_64   ← Randomized each boot
# ffffffff93000000 T startup_64   ← Different on next boot
```

KASLR randomization:
- **Kernel text**: randomized within a ±1GB range (in 2MB steps)
- **Physical load address**: also randomized
- **Direct map**: offset randomized
- **vmalloc region**: base randomized
- **Stack addresses**: per-thread randomization (separate from KASLR)

```bash
# KASLR info in dmesg:
dmesg | grep -i kaslr
# KASLR enabled

# Disable for debugging (kernel command line):
nokaslr

# FKASLR — function-level KASLR (randomizes function order within kernel):
# CONFIG_FG_KASLR=y  (newer kernels)
```

**For kernel debugging**: always boot with `nokaslr` so symbol addresses in `dmesg` match `System.map` without adjustment.

---

## 9. The `vmalloc` Region

`vmalloc` allocates virtually contiguous memory (but physically non-contiguous). Used for:
- Large driver buffers (where physical contiguity isn't required)
- Module loading (`.ko` files)
- `ioremap` (mapping device MMIO into kernel virtual space)
- `vmap` (mapping an array of pages into contiguous virtual space)

```c
#include <linux/vmalloc.h>

// Allocate virtually contiguous, physically non-contiguous:
void *vptr = vmalloc(size);     // Can sleep, returns kernel virtual addr
void *vptr = vzalloc(size);     // Same but zero-filled
vfree(vptr);

// ioremap: map device MMIO registers into kernel virtual space:
void __iomem *base = ioremap(phys_addr, size);
// Then access registers:
u32 val = readl(base + REG_OFFSET);    // 32-bit MMIO read
writel(0x1, base + REG_CONTROL);        // 32-bit MMIO write
// Unmap:
iounmap(base);
```

```bash
# See all vmalloc allocations:
cat /proc/vmallocinfo

# Output format:
# 0xffffXXX-0xffffYYY  size caller
# 0xffffc90000000000-0xffffc90000005000   20480 vmalloc+0x18/...
# 0xffffc90000005000-0xffffc90000007000    8192 __module_alloc+...
# 0xffffXXXXXXXXXXXX-...                  ...  ioremap+... phys=0xfea00000 ...
```

**`kmalloc` vs `vmalloc` — when to use which:**

| Property | `kmalloc` | `vmalloc` |
|----------|-----------|-----------|
| Physical contiguity | Guaranteed | Not guaranteed |
| Virtual contiguity | Yes | Yes |
| Speed | Fast (slab cache) | Slower (page table setup) |
| Max size | ~4MB (order 10 = 1024 pages) | Limited by vmalloc region (~32TB) |
| DMA safe | Yes (with GFP_DMA) | No — need `virt_to_page` + `dma_map_page` per page |
| TLB pressure | Lower | Higher (each page needs own PTE) |
| Use case | Normal kernel allocs | Large allocs, modules, MMIO |

---

## 10. Physical Memory Allocator: Buddy System

The **buddy allocator** manages free physical pages. It's the lowest-level allocator — everything else (`kmalloc`, `vmalloc`) sits on top of it.

### 10.1 Buddy Concept

Pages are allocated in powers of 2. Each order has a free list:
- Order 0: 1 page = 4KB
- Order 1: 2 pages = 8KB
- Order 2: 4 pages = 16KB
- ...
- Order 10: 1024 pages = 4MB

When you request 3 pages:
1. No exact match for 3 pages, so look at order 2 (4 pages)
2. Split order-2 block: give 4 pages (allocate) + keep the extra as order-1 "buddy"
3. The buddy stays in the free list

When you free 4 pages:
1. Check if the "buddy" (the other half of the pair) is also free
2. If yes, merge them back into an order-2 block
3. Recurse: check if the merged block's buddy is free, keep merging

```bash
# See buddy free lists:
cat /proc/buddyinfo
# Node 0, zone   Normal  1234 567 89 12 3 1 0 0 1 0 2
# Column index = order (0 to 10)
# Value = number of free blocks at that order in that zone

# Low order fragmentation is normal; you care about order-10 (4MB) blocks
# being available for large contiguous allocations.
```

### 10.2 Page Allocation API

```c
// include/linux/gfp.h, include/linux/alloc_types.h

// Allocate 2^order contiguous pages:
struct page *page = alloc_pages(GFP_KERNEL, order);
// Returns struct page* (not a virtual address!)

// Get virtual address:
void *kaddr = page_address(page);
// Or:
void *kaddr = kmap(page);   // For highmem (32-bit only)

// Free:
__free_pages(page, order);

// Higher-level (returns virtual address directly):
unsigned long addr = __get_free_pages(GFP_KERNEL, order);
free_pages(addr, order);

// Single page shortcuts:
struct page *page = alloc_page(GFP_KERNEL);   // order=0
__free_page(page);
unsigned long addr = get_zeroed_page(GFP_KERNEL);
free_page(addr);
```

### 10.3 GFP Flags Reference

```c
// Primary context flags (must specify one):
GFP_KERNEL    // Normal allocation: can sleep, can reclaim, can swap
GFP_ATOMIC    // Cannot sleep: interrupt context, spinlock held
GFP_USER      // Allocation for user process
GFP_HIGHUSER  // User allocation, can use ZONE_HIGHMEM
GFP_DMA       // Must be from ZONE_DMA (legacy 24-bit DMA)
GFP_DMA32     // Must be from ZONE_DMA32 (32-bit DMA)
GFP_NOWAIT    // No waiting, no reclaim (but not as strict as GFP_ATOMIC)

// Modifiers (combine with |):
__GFP_ZERO         // Zero-fill the allocated memory
__GFP_NOWARN       // Don't print OOM warning on failure
__GFP_RETRY_MAYFAIL // Retry but return NULL if persistent failure
__GFP_NOFAIL       // MUST succeed — loop forever (dangerous, rarely correct)
__GFP_COMP         // Create compound page (for huge pages, THP)
__GFP_MOVABLE      // Page can be migrated (for memory defrag)
__GFP_ACCOUNT      // Charge to cgroup memory controller
```

---

## 11. The Slab Allocator: `kmalloc`

The buddy allocator is great for pages, but most kernel allocations are much smaller (a `task_struct` is ~8KB, a `sk_buff` is ~240 bytes). Allocating a whole page for a 240-byte object wastes 3850 bytes.

The **slab allocator** (specifically SLUB in modern kernels) sits between `kmalloc` callers and the buddy allocator, managing caches of fixed-size objects.

```c
#include <linux/slab.h>

// Generic size-based allocation (like malloc for the kernel):
void *ptr = kmalloc(size, GFP_KERNEL);    // Allocate 'size' bytes
void *ptr = kzalloc(size, GFP_KERNEL);   // Same + zero-fill (PREFERRED)
void *ptr = kmalloc_array(n, size, GFP_KERNEL);  // n×size, checks overflow
kfree(ptr);                               // Free

// Resizing:
ptr = krealloc(ptr, new_size, GFP_KERNEL);

// Get actual allocated size (may be larger than requested):
ksize(ptr);
```

Under the hood, `kmalloc` uses per-size slab caches:
- `kmalloc-8`, `kmalloc-16`, `kmalloc-32`, ..., `kmalloc-4096`, `kmalloc-8192`
- Each is a cache of pre-allocated objects of that size
- Objects come from pages acquired from the buddy allocator

```bash
# See slab caches and their usage:
cat /proc/slabinfo | head -20
# Name        Active  Num Objects  ObjSize  Pages
# task_struct    234      234       7168      432
# mm_struct      150      150       1088       50
# filp           850      850        256       54

# More details:
sudo slabtop   # Interactive slab usage monitor
sudo cat /sys/kernel/slab/kmalloc-256/alloc_fastpath  # Per-cache stats
```

### 11.1 Named Slab Caches

For frequently allocated objects, create your own slab cache:

```c
// Create a cache (typically in module init):
struct kmem_cache *my_cache;
my_cache = kmem_cache_create(
    "my_object",           // Name (appears in /proc/slabinfo)
    sizeof(struct my_obj), // Object size
    0,                     // Alignment (0 = default)
    SLAB_HWCACHE_ALIGN,    // Flags: align to cache line
    NULL                   // Constructor (optional, called on new objects)
);

// Allocate an object from the cache:
struct my_obj *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
if (!obj) return -ENOMEM;

// Free back to cache:
kmem_cache_free(my_cache, obj);

// Destroy the cache (all objects must be freed first!):
kmem_cache_destroy(my_cache);
```

Using named caches instead of `kmalloc` has advantages:
- Objects show up with your name in `/proc/slabinfo` (debugging)
- Can use a constructor to pre-initialize fields (rare)
- Slightly more cache-efficient (objects packed tightly at exact size)

---

## 12. Memory Models

The kernel has three memory models (configured at compile time):

### 12.1 Flat Memory Model (`CONFIG_FLATMEM`)
Single linear `mem_map[]` array for all physical pages. Simple but wastes memory if physical address space is sparse. Used for systems without NUMA and without holes in physical memory.

### 12.2 Sparse Memory Model (`CONFIG_SPARSEMEM`) — Default
Physical memory is divided into "sections" of 128MB each. Only sections that exist have `struct page` arrays allocated. Supports:
- Memory hotplug (adding/removing RAM at runtime)
- NUMA (discontiguous physical address spaces per node)
- Large physical address spaces with holes

```c
// Converting PFN to struct page in sparse model:
// Uses a two-level table: section → mem_map portion
struct page *pfn_to_page(unsigned long pfn)
{
    unsigned long section = pfn >> PFN_SECTION_SHIFT;
    return __pfn_to_page(pfn);  // Looks up in sparse section table
}
```

```bash
# See memory sections:
cat /sys/devices/system/memory/memory*/state
# online       ← This 128MB section is present and online
# offline      ← Hotplug: section is offline (can be removed)
```

---

## 13. Memory Layout for a Running Process

When you look at a process's virtual address space:

```bash
cat /proc/self/maps
# Address           Perms Offset  Dev   Inode  Path
# 55a234560000-55a234580000 r--p 00000000 fd:01 1234 /bin/bash
# 55a234580000-55a2345f0000 r-xp 00020000 fd:01 1234 /bin/bash  ← .text
# 55a2345f0000-55a234610000 r--p 00090000 fd:01 1234 /bin/bash  ← .rodata
# 55a234810000-55a234830000 r--p 000af000 fd:01 1234 /bin/bash  ← .data (COW)
# 55a234830000-55a234840000 rw-p 000cf000 fd:01 1234 /bin/bash  ← .bss
# 55a235800000-55a23600f000 rw-p 00000000 00:00 0              [heap]
# 7f8a12000000-7f8a12001000 rw-p 00000000 00:00 0              
# 7f8a12001000-7f8a13000000 ---p 00000000 00:00 0              
# ...
# 7ffcf8000000-7ffcf8022000 rw-p 00000000 00:00 0              [stack]
# 7ffcf8fff000-7ffcf9000000 r--p 00000000 00:00 0              [vvar]
# 7ffcf9000000-7ffcf9001000 r-xp 00000000 00:00 0              [vdso]
# ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0      [vsyscall]
```

Each line is a VMA (`vm_area_struct`):
- **Text**: `r-xp` — readable, executable, private (copy-on-write), file-backed
- **Data**: `rw-p` — readable, writable, private, file-backed (COW on fork)
- **Heap**: `rw-p` — readable, writable, private, anonymous (not file-backed)
- **Stack**: `rw-p` — readable, writable, private, anonymous, grows downward
- **vvar**: read-only page with kernel-shared data (for vDSO clocks)
- **vdso**: `r-xp` — the vDSO shared library

---

## 14. Kernel Memory vs User Memory: Key Rules

```c
// Kernel can access its own virtual addresses directly:
struct task_struct *t = current;    // Always valid
printk("%d\n", t->pid);            // Direct dereference — fine

// Kernel CANNOT access user virtual addresses directly:
// (SMAP enforces this on modern CPUs)
char *user_ptr = (char *)regs->rsi;  // User-space pointer
*user_ptr = 'x';                     // CRASH! SMAP violation (or bad data with SMAP disabled)

// Must use the safe API:
char ch;
get_user(ch, user_ptr);     // Safe single-char read
put_user('x', user_ptr);    // Safe single-char write
copy_from_user(kbuf, user_ptr, len);  // Safe bulk read
copy_to_user(user_ptr, kbuf, len);    // Safe bulk write
```

---

## 15. Special Memory Mappings

### 15.1 `fixmap` — Fixed Virtual Addresses

Some kernel addresses must be at a fixed, known virtual address regardless of KASLR:

```c
// arch/x86/include/asm/fixmap.h
enum fixed_addresses {
    VSYSCALL_PAGE,       // Legacy vsyscall page
    FIX_APIC_BASE,       // Local APIC registers
    FIX_IO_APIC_BASE_0,  // I/O APIC registers
    // ...
};

// Get the virtual address for a fixed mapping:
unsigned long addr = fix_to_virt(FIX_APIC_BASE);
void *apic_base = (void *)addr;
```

### 15.2 `kmap` and `kmap_atomic` (32-bit Legacy)

On 32-bit kernels, `ZONE_HIGHMEM` pages couldn't be directly addressed. `kmap()` created temporary mappings. On 64-bit kernels, all RAM is directly mapped, so `kmap()` is essentially a no-op:

```c
// 64-bit: no-op, just returns page_address(page)
void *kaddr = kmap(page);
// ...use kaddr...
kunmap(page);

// kmap_atomic: for use in atomic context (no sleeping):
void *kaddr = kmap_atomic(page);
// ... MUST NOT sleep here ...
kunmap_atomic(kaddr);
```

### 15.3 `memremap` — Persistent Memory

For mapping persistent memory (PMEM, NVDIMMs) that requires special handling:

```c
void *addr = memremap(phys_addr, size, MEMREMAP_WB);  // Write-back cached
void *addr = memremap(phys_addr, size, MEMREMAP_WT);  // Write-through
void *addr = memremap(phys_addr, size, MEMREMAP_WC);  // Write-combining (for framebuffers)
memunmap(addr);
```

---

## 16. Diagnosing Memory Issues

```bash
# Overall memory usage:
free -h
cat /proc/meminfo

# Key /proc/meminfo fields:
# MemTotal:     Total usable RAM
# MemFree:      Completely unused
# MemAvailable: Estimate of memory available for new allocations
#               (includes reclaimable caches — much more useful than MemFree)
# Buffers:      Raw block device cache
# Cached:       Page cache (file data)
# SwapCached:   Swapped pages still in swap and RAM
# Slab:         Kernel slab caches total
# SReclaimable: Slab memory that can be reclaimed (caches)
# SUnreclaim:   Slab memory that cannot be reclaimed

# Per-process memory:
cat /proc/$(pgrep myprocess)/status
# VmPeak: Peak virtual memory size
# VmSize: Current virtual memory size
# VmRSS:  Resident Set Size (physical pages currently used)
# VmSwap: Pages currently swapped out

# Detailed mapping stats:
cat /proc/$(pgrep myprocess)/smaps | grep -A 12 "heap"

# Kernel memory breakdown:
sudo cat /proc/meminfo | grep -E "Slab|Kernel|Vmalloc|KStack"
# Slab:           234567 kB
# KernelStack:      8192 kB  ← 8KB per thread × (threads)
# VmallocUsed:    123456 kB

# SLUB statistics:
sudo cat /sys/kernel/slab/task_struct/objects   # How many task_structs exist
sudo slabtop -o                                  # One-shot slab usage

# Memory fragmentation:
cat /proc/buddyinfo    # Available pages by order per zone
```

---

## 17. Memory Pressure and the OOM Killer

```bash
# Simulate memory pressure:
stress --vm 1 --vm-bytes 90%  # Consume 90% of RAM

# Watch for OOM events:
dmesg -w | grep -i "oom\|kill"
# Out of memory: Kill process 1234 (myprogram) score 987 or sacrifice child
# Killed process 1234 (myprogram) total-vm:2048000kB, anon-rss:1900000kB

# OOM score for a process (0-1000, higher = more likely to be killed):
cat /proc/$(pgrep myprocess)/oom_score
cat /proc/$(pgrep myprocess)/oom_adj       # Legacy (-17 to 15)
cat /proc/$(pgrep myprocess)/oom_score_adj # Current (-1000 to 1000)

# Protect a process from OOM killer:
echo -1000 > /proc/$(pgrep myprocess)/oom_score_adj  # Effectively immune
# Make a process first target:
echo 1000 > /proc/$(pgrep myprocess)/oom_score_adj
```

---

## 18. Mental Model Checkpoint

After Day 8, you should be able to:

1. Draw the x86_64 virtual address space showing user space, the canonical hole, and kernel space regions.
2. Explain the direct map: what physical address corresponds to kernel virtual `0xFFFF880001000000`?
3. What is `__va()` / `__pa()` and when would you use each?
4. What are memory zones and why do `ZONE_DMA` and `ZONE_DMA32` exist?
5. What does KASLR randomize? Why do you disable it for debugging?
6. When would you use `vmalloc` instead of `kmalloc`? What's the tradeoff?
7. What is the buddy allocator and what does `order` mean?
8. What does `GFP_ATOMIC` tell the allocator and when must you use it?
9. What is the difference between `MemFree` and `MemAvailable` in `/proc/meminfo`?
10. What is a NUMA node and how does it affect allocation decisions?

---

## Key Source Files

```bash
arch/x86/include/asm/page_types.h   # PAGE_OFFSET, physical/virtual conversion
arch/x86/include/asm/pgtable_64.h  # Virtual address layout constants
mm/page_alloc.c                      # Buddy allocator: alloc_pages, free_pages
mm/slub.c                            # SLUB allocator: kmalloc, kmem_cache_*
mm/vmalloc.c                         # vmalloc, ioremap implementation
include/linux/mmzone.h               # zone_type, pglist_data, NUMA structures
include/linux/gfp.h                  # GFP flags definitions
arch/x86/mm/init_64.c               # Direct map setup, mem_map initialization
Documentation/x86/x86_64/mm.rst     # OFFICIAL: x86_64 memory map documentation
```

---

## Summary

The kernel's memory layout on x86_64 is a carefully organized 128TB virtual address space:

- **Direct map** (64TB): all physical RAM linearly mapped for fast kernel access via `__va()`/`__pa()`
- **vmalloc region** (32TB): virtually contiguous but physically scattered allocations and MMIO mappings
- **mem_map** (1TB): the `struct page` array — one 64-byte struct per 4KB physical page
- **Kernel text** (1GB): the kernel binary itself, randomized by KASLR
- **Module region** (1.5GB): where `.ko` files live, within ±2GB of kernel text

Physical memory is managed through a two-layer system: the **buddy allocator** handles raw page allocation (in powers-of-2 orders), and the **SLUB allocator** handles sub-page allocations by maintaining per-size caches.

Tomorrow: interrupt handling hardware path — how IRQs travel from device to handler.
