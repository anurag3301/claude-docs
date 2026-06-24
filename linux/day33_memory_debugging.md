# Day 33 — Memory Debugging

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand kmemleak (memory leak detection), KASAN (out-of-bounds and use-after-free detection), SLUB debug features, page_owner tracking, and vmalloc guard pages.

---

## 1. The Memory Bug Taxonomy

Kernel memory bugs fall into four categories:

| Bug Type | What Happens | Tools That Catch It |
|----------|-------------|---------------------|
| Use-after-free | Access freed memory | KASAN, SLUB poison |
| Out-of-bounds | Write past allocation end | KASAN, SLUB red zones |
| Memory leak | Allocate but never free | kmemleak |
| Double-free | Free same allocation twice | KASAN, page_owner, SLUB |
| Uninitialized use | Read before write | KMSAN |
| Page-use-after-free | Use physical page after freeing | page_poison, page_owner |

---

## 2. KASAN — Kernel Address Sanitizer

KASAN instruments every memory access at compile time and tracks which memory is valid. It catches use-after-free and out-of-bounds in near-realtime.

### 2.1 How KASAN Works

```bash
# Enable in kernel config:
# CONFIG_KASAN=y
# CONFIG_KASAN_GENERIC=y     ← Software-based (2-8× slowdown, 2× memory)
# CONFIG_KASAN_HW_TAGS=y     ← Hardware-based on ARM MTE (minimal overhead)
# CONFIG_KASAN_INLINE=y      ← Inline checks (faster than outline)

# Check if running:
cat /proc/sys/kernel/kasan_enabled  # 1 if active
```

KASAN adds a "shadow memory" region: for every 8 bytes of kernel memory, 1 byte of shadow tracks validity. Shadow byte meanings:
- `0x00`: all 8 bytes valid
- `0x01`–`0x07`: first N bytes valid, rest are partial allocation
- `0xFx`: poisoned (freed/out-of-bounds)
- `0xFA`: KMALLOC_REDZONE (red zone after allocation)
- `0xFB`: KMALLOC_FREE (freed kmalloc object)
- `0xFC`: SLAB_REDZONE (red zone, slab)
- `0xFD`: SLUB_FREE
- `0xFE`: STACK_LEFT_REDZONE / STACK_RIGHT_REDZONE

```c
// What KASAN-instrumented code looks like:
// Original: val = *ptr;
// Instrumented (by compiler):
// __asan_load8(ptr);  // Check shadow before load
// val = *ptr;

// The shadow check:
void __asan_load8(unsigned long addr)
{
    // Compute shadow address: shadow = (addr >> 3) + KASAN_SHADOW_OFFSET
    unsigned long shadow_addr = (addr >> 3) + kasan_shadow_offset;
    signed char shadow_val = *(signed char *)shadow_addr;
    
    if (unlikely(shadow_val)) {
        // Memory is poisoned or partially valid:
        if (shadow_val < 0 || shadow_val > (addr & 7))
            kasan_report(addr, 8, false, ...);
    }
}
```

### 2.2 KASAN Report Format

```
==================================================================
BUG: KASAN: slab-out-of-bounds in copy_from_user_nofault+0x5c/0x80
Write of size 1 at addr ffff888102b4e400 by task systemd/1

CPU: 0 PID: 1 Comm: systemd
Hardware name: QEMU Standard PC
Call Trace:
 dump_stack_lvl+0x45/0x5c
 print_address_description.constprop.0+0x28/0x1c0
 kasan_report+0x11c/0x120
 check_bytes_and_report+0xf5/0x110
 copy_from_user_nofault+0x5c/0x80
 ...

Allocated by task 1234:
 kmalloc+0x... 
 my_driver_probe+0x... 

The buggy address belongs to the object at ffff888102b4e380
 which belongs to the cache kmalloc-128 of size 128
The buggy address is located 128 bytes to the right of
 128-byte region [ffff888102b4e380, ffff888102b4e400)

Memory state around the buggy address:
 ffff888102b4e300: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
 ffff888102b4e380: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
>ffff888102b4e400: fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc fc
                   ^  ← fc = SLAB_REDZONE (out-of-bounds detected here)
==================================================================
```

### 2.3 KASAN Use-After-Free Example

```c
// This code has a use-after-free bug:
char *buf = kmalloc(64, GFP_KERNEL);
kfree(buf);
buf[0] = 'X';  // BUG: use-after-free

// With KASAN:
// kfree() poisons the shadow of buf with 0xFB (KMALLOC_FREE)
// buf[0] = 'X' → __asan_store1(buf) → shadow = 0xFB → KASAN_REPORT
// Reports: "BUG: KASAN: use-after-free in ..."
```

### 2.4 KMSAN — Uninitialized Memory Sanitizer

```bash
# CONFIG_KMSAN=y (Linux 5.18+)
# Similar to KASAN but tracks uninitialized memory:

char *buf = kmalloc(64, GFP_KERNEL);
// buf contents are uninitialized (KMSAN marks them "uninit")
if (buf[0] == 'X') { ... }  // BUG: using uninitialized value
// KMSAN reports: "BUG: KMSAN: uninit-value in ..."
```

---

## 3. kmemleak — Memory Leak Detection

`kmemleak` scans all kernel memory for allocated objects that are no longer referenced. It's based on a mark-and-sweep garbage collector concept.

### 3.1 How kmemleak Works

```c
// mm/kmemleak.c
// Every kmalloc/kmem_cache_alloc is tracked:
// kernel/kmemleak.c hooks into the slab allocator:

static void kmemleak_alloc_recursive(const void *ptr, size_t size, ...)
{
    // Create a "kmemleak_object" for this allocation:
    struct kmemleak_object *object;
    object = mem_pool_alloc(...);
    object->pointer = ptr;
    object->size    = size;
    object->count   = 0;   // Will be set to reference count
    // Add to radix tree indexed by pointer:
    radix_tree_insert(&object_tree, (unsigned long)ptr, object);
}

// Scanning: kmemleak_scan():
// 1. Mark all objects as "gray" (referenced count = 0)
// 2. Scan all kernel memory regions for pointer values:
//    - Kernel BSS/data sections
//    - Per-CPU memory
//    - Stack of all tasks
//    - All currently allocated objects themselves
// 3. For each pointer found: if it points into a tracked object → mark object "referenced"
// 4. Objects still "unreferenced" after scan → leaked!
```

### 3.2 Using kmemleak

```bash
# Enable at boot:
# CONFIG_DEBUG_KMEMLEAK=y
# kernel command line: kmemleak=on  (or off to save overhead)

# Check if enabled:
ls /sys/kernel/debug/kmemleak

# Run a scan manually:
echo scan > /sys/kernel/debug/kmemleak

# View detected leaks:
cat /sys/kernel/debug/kmemleak
# unreferenced object 0xffff888123456789 (size 128):
#   comm "insmod", pid 1234, jiffies 4294769536 (age 50.000s)
#   hex dump (first 32 bytes):
#     68 65 6c 6c 6f 00 00 00 00 00 00 00 00 00 00 00  hello...........
#   backtrace:
#     [<0000000012345678>] kmalloc+0x...
#     [<0000000087654321>] my_module_init+0x...

# Clear reported leaks (doesn't free them):
echo clear > /sys/kernel/debug/kmemleak

# Disable scanning:
echo off > /sys/kernel/debug/kmemleak
```

### 3.3 kmemleak in Driver Code

```c
// Sometimes pointers are stored in non-obvious places (hardware registers,
// private data embedded in other structures). kmemleak needs hints:

// Tell kmemleak this pointer is actually referenced (don't report as leak):
kmemleak_not_leak(ptr);         // Suppress for this allocation
kmemleak_ignore(ptr);           // Suppress permanently

// Tell kmemleak about a container: if container is referenced,
// ptr at offset is also referenced:
kmemleak_offset_of(ptr, struct my_type, field);

// Register an external pointer (in MMIO-mapped registers):
kmemleak_no_scan(ptr);  // Don't scan this region for pointers

// For objects freed by custom paths (not kfree):
kmemleak_free(ptr);  // Manually tell kmemleak this was freed
```

---

## 4. SLUB Debug Features

### 4.1 Poison Detection

When `CONFIG_SLUB_DEBUG=y` and `SLAB_POISON` flag is set:

```c
// After kfree():
// Object filled with 0x6B ('k' pattern)

// Before kmalloc() returns:
// Object verified to still be all-0x6B (use-after-free detection)
// Then overwritten with 0x5A ('Z' pattern)

// When corruption detected:
// ============================================================
// BUG kmalloc-128: Redzone overwritten
// -------------------------------------------------------
// INFO: 0xffff888102b4e380-0xffff888102b4e3ff @offset=0
// First byte 0x5a instead of 0x6b
//   Object at 0xffff888102b4e380, in cache kmalloc-128
//   Allocated in task/pid 1234/kmalloc+0x...
//   Freed in task/pid 5678/kfree+0x...
// ============================================================
```

### 4.2 Enabling Per-Cache or Global SLUB Debug

```bash
# Enable for all caches at boot:
# kernel command line: slub_debug=FPZU
# F = sanity checks (meta-data integrity)
# P = poisoning (0x6B/0x5A fill)
# Z = red zones (boundary detection)
# U = user tracking (store allocator/freer call stack)

# Enable for specific cache at boot:
# slub_debug=P,dentry         ← Only dentry cache
# slub_debug=FP,kmalloc-128   ← kmalloc-128 only

# Enable at runtime:
echo FPZU > /sys/kernel/slab/kmalloc-128/flags
# or:
echo 1 > /sys/kernel/slab/kmalloc-128/sanity_checks
echo 1 > /sys/kernel/slab/kmalloc-128/trace

# Check current flags:
cat /sys/kernel/slab/kmalloc-128/flags
# POISON|RED_ZONE|STORE_USER
```

### 4.3 Red Zone Overflow Detection

```c
// With SLAB_RED_ZONE, the slab layout becomes:
// [PRE_RED_ZONE(8 bytes=0xBB...)] [OBJECT(N bytes)] [POST_RED_ZONE(8 bytes=0xBB...)]
//
// Example overflow:
char *buf = kmalloc(128, GFP_KERNEL);
memset(buf, 'A', 140);  // BUG: writes 12 bytes into red zone

// On next alloc or explicit check:
// BUG kmalloc-128: Right Redzone overwritten
// at offset 128 size 8 allocated at: ...
```

### 4.4 `validate_slab_cache()`

```bash
# Validate all objects in a slab cache:
echo validate > /sys/kernel/slab/kmalloc-128/validate
# Checks every object's red zones and poison patterns
# Detects corruption even before next alloc/free triggers it

# Validate all caches:
for cache in /sys/kernel/slab/*/; do
    echo validate > $cache/validate 2>/dev/null
done
# Then check dmesg for any BUG reports
```

---

## 5. page_owner — Physical Page Allocation Tracking

`page_owner` tracks who allocated each physical page (not slab objects, but raw pages from the buddy allocator).

```bash
# Enable at boot:
# CONFIG_PAGE_OWNER=y
# kernel command line: page_owner=on

# After boot, find what holds a suspicious PFN:
echo 1 > /proc/sys/kernel/page_owner  # Enable (if not enabled at boot)
cat /sys/kernel/debug/page_owner | head -100

# Output format:
# Page allocated via order 0, mask 0xcc0(GFP_KERNEL), pid 1234, ts 1234567890ns, free_ts 0ns
# PFN 0x123456 type Movable
# Page last allocated via order 0, gfp_mask 0xcc0, pid 1234 (init), ts 1234567890
# get_page_from_freelist+0x...
# __alloc_pages+0x...
# alloc_pages+0x...
# kmem_cache_alloc+0x...
# my_function+0x...
```

### 5.1 Detecting Page Leaks

```bash
# Find pages allocated but never freed:
# 1. Boot with page_owner=on
# 2. Do your workload
# 3. Compare before/after:

# Snapshot 1:
cat /sys/kernel/debug/page_owner | grep "PFN" | sort > /tmp/pages_before.txt

# Do suspected leaky operation

# Snapshot 2:
cat /sys/kernel/debug/page_owner | grep "PFN" | sort > /tmp/pages_after.txt

# Find new pages:
comm -13 /tmp/pages_before.txt /tmp/pages_after.txt | head -20
```

### 5.2 `page_poison` — Page-Level Use-After-Free

```bash
# Enable page poisoning:
# CONFIG_PAGE_POISONING=y
# kernel command line: page_poison=on
# OR: page_poison=1 (for some kernel versions)

# Effect:
# When page is freed → filled with 0xAA (PAGE_POISON_FREE)
# When page is allocated → verified to still be 0xAA
# If not: page was written after being freed → BUG()

# Alternative via debug:
echo 1 > /sys/kernel/mm/page_poison
```

---

## 6. vmalloc Guards

`vmalloc()` allocates virtually contiguous memory with guard pages to detect out-of-bounds access:

```c
// vmalloc layout:
// [GUARD_PAGE | alloc_page1 | alloc_page2 | ... | alloc_pageN | GUARD_PAGE]
//  (no PTE)    4KB real     4KB real              4KB real     (no PTE)
//
// If code writes before or after the allocation:
// → Hardware page fault (guard page has no PTE → #PF exception)
// → Kernel catches the fault → BUG/WARN

// vmalloc guard pages are unmapped (PTE not present)
// vmalloc memory layout: VMALLOC_START to VMALLOC_END
// Each vmalloc() region separated by at least one guard page
```

```bash
# View vmalloc allocations:
cat /proc/vmallocinfo
# 0xffffb73800000000-0xffffb73800005000   20480 start_kernel+0x... pages=4 vmalloc
# 0xffffb73800006000-0xffffb73800009000   12288 __alloc_percpu+0x... pages=2 vmap
# 0xffffb73800010000-0xffffb73800015000   20480 my_driver_init+0x... pages=4 vmalloc

# Total vmalloc usage:
cat /proc/meminfo | grep VmallocUsed
# VmallocUsed:     123456 kB
```

---

## 7. lockdep for Memory-Related Locks

Memory-related code has complex locking (mmap_lock, PTL, zone->lock). `lockdep` validates lock ordering:

```bash
# CONFIG_PROVE_LOCKING=y (includes lockdep)
# At runtime lockdep builds a dependency graph and reports cycles:

# ======================================================
# WARNING: possible circular locking dependency detected
# 6.9.0-rc1 #1
# ------------------------------------------------------
# my_thread/1234 is trying to acquire lock:
# ffffffff81234567 (mmap_lock){....}
# but task is already holding lock:
# ffffffff87654321 (&zone->lock){....}
# 
# which lock already depends on the new lock.
# ...
# Possible unsafe locking scenario:
#   CPU0                    CPU1
#   ----                    ----
#   lock(&zone->lock);
#                           lock(mmap_lock);
#                           lock(&zone->lock);
#   lock(mmap_lock);
# ======================================================

# The rule for memory management locks (outermost to innermost):
# mmap_lock → page_table_lock → zone->lock → lru_lock
```

---

## 8. `CONFIG_DEBUG_PAGEALLOC` — Page Allocation Debugging

```bash
# CONFIG_DEBUG_PAGEALLOC=y
# When a page is freed to the buddy allocator:
# → Its PTE is removed (page becomes inaccessible)
# → Any access to the freed page → hardware page fault → oops
#
# Detects: use-after-free of full pages
# Cost: significant overhead (must remap pages on re-allocation)

# Also: DEBUG_PAGEALLOC fills freed pages with a pattern:
# (similar to page_poison)
```

---

## 9. `PROVE_RCU` and Memory Ordering Bugs

```bash
# CONFIG_PROVE_RCU=y
# Checks that RCU-protected data is not accessed outside RCU read-side critical sections:

// BUG: accessing RCU pointer without rcu_read_lock:
struct foo *p = rcu_dereference(global_foo);  // BUG if not in RCU CS
// lockdep prints:
// WARNING: suspicious RCU usage
// include/linux/rcupdate.h:670 suspicious rcu_dereference_check() usage!
```

---

## 10. Complete Debugging Workflow

### Scenario: Kernel Module Has Memory Bug

```bash
# Step 1: Enable comprehensive debugging at boot:
# CONFIG_KASAN=y
# CONFIG_DEBUG_KMEMLEAK=y
# CONFIG_SLUB_DEBUG=y
# CONFIG_PAGE_OWNER=y
# kernel command line: page_owner=on kmemleak=on slub_debug=FPZU

# Step 2: Load the module and run workload:
insmod my_buggy_module.ko
./stress_test

# Step 3: Check for immediate KASAN reports:
dmesg | grep -A 30 "BUG: KASAN"

# Step 4: Check kmemleak:
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak

# Step 5: Validate slab caches:
for cache in /sys/kernel/slab/*/; do
    name=$(basename $cache)
    echo "validate" > $cache/validate 2>/dev/null
done
dmesg | grep -i "SLUB\|slab\|BUG"

# Step 6: Remove module and scan again:
rmmod my_buggy_module
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
# If new entries appear → module leaked memory on unload
```

---

## 11. `memtest` — Physical Memory Testing

```bash
# At boot: test physical RAM for hardware errors:
# kernel command line: memtest=4  (run 4 test passes)

# Or use userspace memtest86+:
# Install and run from USB, scans all RAM independently of OS
```

---

## 12. `WARN_ON_ONCE` vs `BUG_ON` vs `VM_BUG_ON`

```c
// Different severity levels in kernel memory code:

// BUG(): Unrecoverable — prints oops, kills the kernel
BUG();
BUG_ON(condition);  // BUG() if condition is true

// WARN(): Recoverable — prints stack trace but continues
WARN(condition, "message %d", val);
WARN_ON(condition);        // Once per call site
WARN_ON_ONCE(condition);   // Only warns once even if hit repeatedly

// VM_BUG_ON(): Memory subsystem specific BUG_ON
// Only active with CONFIG_DEBUG_VM
VM_BUG_ON(!PageLocked(page));       // Disabled in production
VM_BUG_ON_PAGE(!page_count(page), page);  // Same but prints page info

// pr_warn_ratelimited(): For high-frequency events
pr_warn_ratelimited("suspicious PTE at %lx\n", addr);
// Prints at most 10 times per 5 seconds (prevents log spam)
```

---

## 13. `KCSAN` — Kernel Concurrency Sanitizer

Detects data races in kernel code:

```bash
# CONFIG_KCSAN=y
# Instruments every memory access with instrumentation that checks for races:
# If two CPUs access the same memory location and at least one writes:
# KCSAN reports a data race.

# ====================================================
# BUG: KCSAN: data-race in my_func / other_func
# write to 0xffff88810b1b1234 of 4 bytes by task 123 on cpu 0:
#  my_func+0x20/0x40
#  ...
# read to 0xffff88810b1b1234 of 4 bytes by task 456 on cpu 1:
#  other_func+0x30/0x60
#  ...
# ====================================================
```

---

## 14. Tracing Memory Allocations

```bash
# kmem_cache tracing via ftrace:
cd /sys/kernel/debug/tracing
echo 'kmem:kmalloc kmem:kfree' > set_event
echo 1 > tracing_on
sleep 1
cat trace | grep -v '^#' | head -30
# kmalloc:        call_site=0xffffffff81234567 ptr=0xffff88811234 bytes_req=128 bytes_alloc=128 gfp_flags=0xcc0
# kfree:          call_site=0xffffffff87654321 ptr=0xffff88811234

# Find who's allocating the most:
cat trace | grep kmalloc | awk '{print $6}' | sort | uniq -c | sort -rn | head -10

# bpftrace: trace large kmalloc calls:
sudo bpftrace -e '
tracepoint:kmem:kmalloc {
    if (args->bytes_req > 4096) {
        printf("large kmalloc: %d bytes by %s\n", args->bytes_req, comm);
        printf("  site: %lx\n", args->call_site);
    }
}'
```

---

## 15. Phase 3 Summary

All 11 Phase 3 guides complete:

| Day | Topic | Key Takeaway |
|-----|-------|-------------|
| 23 | Physical memory model | `struct page`, zones, NUMA, PFN, vmemmap |
| 24 | Buddy allocator | Orders, splitting/merging, GFP flags, slowpath |
| 25 | SLUB allocator | Per-CPU freelist, slab pages, kmalloc caches, SLUB_DEBUG |
| 26 | VMAs | `mm_struct`, `vm_area_struct`, `mmap()`, anonymous vs file |
| 27 | Page tables | 4-level PT, PTE format, TLB, shootdown, PCID, huge PTEs |
| 28 | Page faults | Demand paging, COW, swap-in, exception table |
| 29 | Page cache | `address_space`, XArray, readahead, dirty tracking, writeback |
| 30 | Swap & reclaim | LRU lists, kswapd, swappiness, rmap, swap entries, zswap |
| 31 | OOM killer | `oom_badness()`, `oom_score_adj`, cgroup OOM, panic_on_oom |
| 32 | Huge pages | HugeTLBFS, THP, khugepaged, compaction, per-app strategy |
| 33 | Memory debugging | KASAN, kmemleak, SLUB debug, page_owner, vmalloc guards |

---

## Mental Model Checkpoint

After Day 33, you should be able to:

1. Explain how KASAN detects use-after-free using shadow memory.
2. What algorithm does kmemleak use and what does it scan?
3. How do SLUB poison values (0x6B, 0x5A) detect use-after-free?
4. What does `page_owner` track that kmemleak doesn't?
5. What does a vmalloc guard page do and what triggers it?
6. When would you use KASAN vs kmemleak vs SLUB_DEBUG for a kernel bug?
7. What is `VM_BUG_ON()` vs `BUG_ON()` — when is each compiled in?
8. How do you detect memory leaks in a loadable kernel module?

---

## Key Source Files

```bash
mm/kasan/                   # KASAN implementation (shadow memory, reporting)
mm/kmemleak.c               # kmemleak: allocation tracking, scanning, reporting
mm/slub.c                   # SLUB poison, red zones, user tracking
mm/page_owner.c             # page_owner: per-page allocation tracking
mm/vmalloc.c                # vmalloc guard pages, vmap_area management
include/linux/poison.h      # Poison byte values (0x6B, 0x5A, 0xAA, 0xBB...)
include/linux/kasan.h       # KASAN API
Documentation/dev-tools/kasan.rst      # KASAN documentation
Documentation/dev-tools/kmemleak.rst   # kmemleak documentation
```
