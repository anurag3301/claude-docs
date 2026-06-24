# Day 30 — Swapping & Page Reclaim

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand the LRU-based reclaim algorithm, `kswapd`, anonymous vs file reclaim, swappiness, and swap device mechanics.

---

## 1. Why Reclaim Exists

The kernel keeps as much data in RAM as possible — the page cache uses "all available" memory. When new allocations need physical pages, the kernel **reclaims** (frees) existing pages.

Two kinds of reclaimable memory:
1. **File-backed pages** (page cache): backed by a file on disk — can be freed and re-read from disk on next access
2. **Anonymous pages** (heap, stack): no backing file — must be **swapped** to swap space before freeing

Key insight: swapping is not a failure — it's a feature. Inactive anonymous pages on a system with swap are better than getting an OOM kill.

---

## 2. LRU Lists — The Core Data Structure

The kernel maintains four LRU (Least Recently Used) lists per NUMA node:

```c
// include/linux/mm_types.h / include/linux/mmzone.h
enum lru_list {
    LRU_INACTIVE_ANON = LRU_BASE,          // Anonymous pages not recently used
    LRU_ACTIVE_ANON   = LRU_BASE + LRU_ACTIVE, // Anonymous pages recently used
    LRU_INACTIVE_FILE = LRU_BASE + LRU_FILE,   // File pages not recently used
    LRU_ACTIVE_FILE   = LRU_BASE + LRU_FILE + LRU_ACTIVE, // File pages recently used
    LRU_UNEVICTABLE,                        // Pages that cannot be evicted (mlock)
    NR_LRU_LISTS
};

struct lruvec {
    struct list_head lists[NR_LRU_LISTS];
    spinlock_t lru_lock;
    // Stats per list:
    unsigned long anon_cost;
    unsigned long file_cost;
    atomic_long_t nonresident_age;
    unsigned long refaults[ANON_AND_FILE];
    unsigned long flags;
    // For mglru:
    struct pglist_data *pgdat;
};
```

### 2.1 How Pages Move Between Lists

```
New file page:      → LRU_INACTIVE_FILE
New anon page:      → LRU_INACTIVE_ANON

On second access:   → LRU_ACTIVE_ANON/FILE (promoted)
  (when PTE A bit is found set)

Reclaim path:
  LRU_ACTIVE_*     → LRU_INACTIVE_* (demoted, "aged")
  LRU_INACTIVE_*   → freed (file) or swapped (anon)

mlock()'d pages:    → LRU_UNEVICTABLE (never reclaimed)
```

---

## 3. `kswapd` — Background Reclaim

`kswapd` is a per-node kernel thread that runs when free memory falls below watermarks:

```bash
# One kswapd per NUMA node:
ps aux | grep kswapd
# root    105  0.0  0.0  kswapd0   ← NUMA node 0
# root    106  0.0  0.0  kswapd1   ← NUMA node 1

# kswapd wakes when:
# - Free pages fall below zone->_watermark[WMARK_LOW]
# - kswapd sleeps when free pages reach zone->_watermark[WMARK_HIGH]
```

```c
// mm/vmscan.c
static int kswapd(void *p)
{
    pg_data_t *pgdat = (pg_data_t *)p;
    
    for (;;) {
        // Sleep until woken (by memory allocator when below WMARK_LOW):
        kswapd_wait_event(pgdat);
        
        // Balance all zones:
        kswapd_shrink_node(pgdat, &sc);
        // This calls shrink_node() which calls shrink_lruvec()
    }
}

static void shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)
{
    // Calculate how many pages to scan from each list:
    // Ratio: (swappiness / 200) for anon, rest for file
    get_scan_count(lruvec, sc, nr);
    
    // Shrink each list:
    shrink_list(LRU_INACTIVE_ANON, nr[LRU_INACTIVE_ANON], lruvec, sc);
    shrink_list(LRU_INACTIVE_FILE, nr[LRU_INACTIVE_FILE], lruvec, sc);
    
    // Promote active → inactive to refill inactive lists:
    shrink_active_list(nr_to_deactivate, lruvec, sc, LRU_ACTIVE_ANON);
    shrink_active_list(nr_to_deactivate, lruvec, sc, LRU_ACTIVE_FILE);
}
```

---

## 4. `swappiness` — Anon vs File Reclaim Balance

```bash
# Current swappiness:
cat /proc/sys/vm/swappiness
# 60  ← Default: moderate tendency to swap

# Range: 0–200 (in newer kernels)
# 0   = Extremely avoid swapping anonymous pages
#       (still may swap if no file pages to reclaim)
# 60  = Default: moderate balance
# 100 = Equal treatment of anon and file pages
# 200 = Aggressively prefer swapping over file cache eviction

# For databases (like PostgreSQL, which manages its own cache):
echo 10 > /proc/sys/vm/swappiness  # Prefer to evict file cache over swapping

# For desktop (user experience: prefer keeping app data over file cache):
echo 10 > /proc/sys/vm/swappiness  # Also often set low on desktops

# Persist:
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

```c
// mm/vmscan.c — the swappiness calculation:
static void get_scan_count(struct lruvec *lruvec, struct scan_control *sc,
                            unsigned long *nr)
{
    unsigned long anon_cost, file_cost, total_cost;
    unsigned long ap, fp;  // anon proportion, file proportion
    
    // swappiness controls the ratio:
    // ap = swappiness (0-200)
    // fp = 200 - swappiness
    
    // Adjusted by actual cost:
    // - File pages: cost = 1 (free immediately if clean)
    // - Anon pages: cost = (swap latency) — higher if swap is slow
    
    unsigned long swappiness = mem_cgroup_swappiness(memcg);
    
    // Final scan counts:
    // nr[INACTIVE_ANON] = anon_cost * anon_pct
    // nr[INACTIVE_FILE] = file_cost * file_pct
}
```

---

## 5. Page Reclaim: `shrink_page_list()`

```c
// mm/vmscan.c
static unsigned long shrink_page_list(struct list_head *page_list,
                                       struct pglist_data *pgdat,
                                       struct scan_control *sc,
                                       struct reclaim_stat *stat, ...)
{
    struct page *page;
    
    list_for_each_entry_safe(page, next, page_list, lru) {
        
        // 1. Skip locked pages:
        if (PageLocked(page) && !trylock_page(page))
            goto keep;
        
        // 2. Anon page: must swap before freeing:
        if (PageAnon(page) && !PageSwapCache(page)) {
            if (!sc->may_swap)
                goto keep_locked;
            
            // Allocate swap slot:
            if (!add_to_swap(page)) {
                // No swap space: can't reclaim this page
                stat->nr_skipped_anon++;
                goto keep_locked;
            }
        }
        
        // 3. Dirty page: must write back before freeing:
        if (PageDirty(page)) {
            if (page_is_file_lru(page)) {
                // File page: schedule async writeback
                pageout(page, mapping);
                goto keep_locked;  // Will try again after writeback
            }
        }
        
        // 4. Referenced page: give it another chance:
        if (PageReferenced(page)) {
            ClearPageReferenced(page);
            goto keep;  // Move to active list
        }
        
        // 5. Still mapped in page tables:
        if (page_mapped(page)) {
            // Unmap from all PTEs (TLB flush):
            try_to_unmap(page, TTU_BATCH_FLUSH);
            // If still mapped: another process is using it → keep
            if (page_mapped(page))
                goto keep;
        }
        
        // 6. Page is clean and unmapped: free it!
        if (PageSwapCache(page)) {
            // Anon page in swap cache: actual write to swap already done
            free_swap_cache(page);
        }
        
        if (!mapping || !mapping->a_ops->is_partially_uptodate)
            SetPageUptodate(page);
        
        // Remove from page cache and free:
        __delete_from_page_cache(page, NULL);
        list_add(&page->lru, &free_pages);
        stat->nr_reclaimed++;
        continue;
        
keep_locked:
        unlock_page(page);
keep:
        list_add(&page->lru, &ret_pages);
    }
    
    mem_cgroup_uncharge_list(&free_pages);
    release_pages(&free_pages, ...);
    
    return stat->nr_reclaimed;
}
```

---

## 6. Reverse Mapping (`rmap`)

To unmap a page from all page tables, the kernel needs to find all PTEs pointing to a given physical page. This is **reverse mapping** (rmap):

### 6.1 File Pages: `i_mmap` Tree

```c
// address_space maintains a tree of VMAs mapped to this file:
// address_space->i_mmap: rb_root_cached of vm_area_structs

// try_to_unmap for file-backed page:
static bool try_to_unmap_one(struct page *page, struct vm_area_struct *vma, ...)
{
    // Walk i_mmap tree to find all VMAs:
    // For each VMA: calculate virtual address, invalidate PTE
    
    pte_t *pte = page_check_address(page, mm, address, ...);
    if (pte) {
        pte_t pteval = ptep_clear_flush(vma, address, pte);
        // ptep_clear_flush: atomically clear PTE and flush TLB
    }
}
```

### 6.2 Anonymous Pages: `anon_vma`

```c
// Anonymous pages use a different structure (no file → no i_mmap):
struct anon_vma {
    struct anon_vma *root;       // Root of tree
    struct rw_semaphore rwsem;
    atomic_t refcount;
    struct rb_root rb_root;      // Tree of avc structs
};

struct anon_vma_chain {
    struct vm_area_struct *vma;  // The VMA
    struct anon_vma *anon_vma;   // The anon_vma
    struct list_head same_vma;   // In vma->anon_vma_chain
    struct rb_node rb;           // In anon_vma->rb_root
};

// On fork: parent and child share anon_vma (COW pages need rmap from both)
// Each page's page->anon_vma → anon_vma → find all VMAs via rb_root
```

---

## 7. Swap Space

### 7.1 Swap Types

```bash
# See swap usage:
swapon --show
# NAME       TYPE  SIZE  USED PRIO
# /swapfile  file  8G    0B    -2
# /dev/sdb1  partition 16G  2G  -3

# Enable swap:
mkswap /dev/sdb1          # Format as swap
swapon /dev/sdb1          # Enable
swapon -p 10 /dev/sdb1   # Enable with priority 10 (higher = preferred)

# Create a swapfile:
fallocate -l 8G /swapfile   # or: dd if=/dev/zero of=/swapfile bs=1M count=8192
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# See swap statistics:
cat /proc/swaps
vmstat -s | grep swap
```

### 7.2 Swap Entry in PTE

When a page is swapped out, its PTE is replaced with a **swap entry**:

```c
// include/linux/swapops.h
// Swap PTE format (when P bit = 0):
// Bits [0]   = 0 (page not present)
// Bits [4:1] = swap type (0-31, which swap device)
// Bits [63:5] = swap offset (page number in swap space)

// Encoding/decoding:
swp_entry_t entry = make_swp_entry(type, offset);
pte_t pte = swp_entry_to_pte(entry);

swp_entry_t entry = pte_to_swp_entry(pte);
unsigned int type   = swp_type(entry);
pgoff_t offset      = swp_offset(entry);
```

### 7.3 Swap Out Process

```c
// mm/swap_state.c + mm/vmscan.c

// 1. Select page to swap (from inactive anon LRU list)

// 2. Allocate a swap slot:
swp_entry_t entry;
get_swap_page(page, &entry);  // Find free slot in swap space

// 3. Write page to swap:
swap_writepage(page, &wbc);
// → submit_bio() → block device → swap partition/file
// Page is marked PG_writeback during this

// 4. After write completes:
// - Page stays in "swap cache" (page cache backed by swap, not file)
// - PTE updated to swap entry
// - Page freed when last PTE is replaced

// For efficiency: multiple pages are written together (swap cluster)
#define SWAPFILE_CLUSTER 256  // Write 256 pages at once to minimize seeks
```

---

## 8. Swap In Process

When a process accesses a swapped-out page:

```c
// mm/memory.c → handle_pte_fault → do_swap_page():

vm_fault_t do_swap_page(struct vm_fault *vmf)
{
    swp_entry_t entry = pte_to_swp_entry(vmf->orig_pte);
    
    // 1. Check swap cache (might still be in memory):
    struct page *page = lookup_swap_cache(entry, vma, vmf->address);
    
    if (!page) {
        // 2. Read from swap device:
        page = swapin_readahead(entry, GFP_HIGHUSER_MOVABLE, vmf);
        // swapin_readahead: read this page + surrounding pages (swap readahead)
        // Returns page in swap cache
    }
    
    // 3. Wait for I/O:
    wait_on_page_locked(page);
    
    // 4. Remove from swap cache (page back in anonymous use):
    delete_from_swap_cache(page_folio(page));
    
    // 5. Install PTE (now points to physical page, not swap entry):
    pte_t pte = mk_pte(page, vma->vm_page_prot);
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, pte);
    
    // 6. Free the swap slot:
    put_swap_page(page, entry);
    
    return VM_FAULT_MAJOR;  // Major fault: did disk I/O
}
```

---

## 9. Multi-Generational LRU (MGLRU)

Linux 6.1+ includes an optional redesigned reclaim algorithm called **Multi-Generational LRU** (CONFIG_LRU_GEN):

```bash
# Enable/disable MGLRU at runtime:
cat /sys/kernel/mm/lru_gen/enabled
# 0x0003   ← Bitmask: bit0=MGLRU on, bit1=use PTE hints

echo Y > /sys/kernel/mm/lru_gen/enabled  # Enable all features
echo N > /sys/kernel/mm/lru_gen/enabled  # Disable (use classic LRU)

# MGLRU advantages:
# - Pages grouped into "generations" instead of just active/inactive
# - 4+ generations provide better age tracking
# - Can better distinguish "just accessed" from "frequently accessed"
# - Claimed ~4-8% better performance on some workloads
# - Better memory pressure handling on mobile/desktop
```

---

## 10. `zswap` — Compressed Swap in RAM

`zswap` compresses pages before swapping, keeping them in RAM:

```bash
# Enable zswap:
echo 1 > /sys/module/zswap/parameters/enabled
echo 20 > /sys/module/zswap/parameters/max_pool_percent  # Use up to 20% RAM
echo lz4 > /sys/module/zswap/parameters/compressor      # Fast compression

# Check zswap stats:
cat /sys/kernel/debug/zswap/stored_pages     # Pages currently in zswap
cat /sys/kernel/debug/zswap/pool_total_size  # Compressed size
cat /sys/kernel/debug/zswap/written_back_pages  # Pages pushed to real swap

# zswap works as a front-end to real swap:
# Page evicted → try zswap (compress, store in RAM pool)
# If pool full → push oldest to real swap
# On swap-in → decompress from zswap (fast!) or read from disk (slow)

# zram: swap device backed by compressed RAM (no disk at all):
modprobe zram
echo lz4 > /sys/block/zram0/comp_algorithm
echo 8G > /sys/block/zram0/disksize
mkswap /dev/zram0
swapon -p 100 /dev/zram0  # Highest priority (use first)
```

---

## 11. Direct Reclaim vs `kswapd`

```
Memory allocation path:
  __alloc_pages()
      │
      ├─ Fast path: zone has free pages → alloc
      │
      └─ Slow path: zone below WMARK_LOW:
            │
            ├─ Wake kswapd (async reclaim in background)
            │
            └─ Try again (WMARK_MIN threshold):
                  │
                  ├─ Success → return page
                  │
                  └─ Still below WMARK_MIN:
                        Direct reclaim (caller itself reclaims)
                        Blocks the allocation until pages are freed
                        → Can trigger OOM if fails repeatedly
```

Direct reclaim is costly — the allocating task stalls while the kernel scans LRU lists, writes back dirty pages, and swaps out anonymous pages.

---

## 12. `/proc/meminfo` Deep Dive

```bash
cat /proc/meminfo
# MemTotal:     32GB   ← Physical RAM
# MemFree:       1GB   ← Completely unused
# MemAvailable: 28GB   ← Estimate of available memory (includes reclaimable cache)
# Buffers:      100MB  ← Temporary storage for raw disk blocks
# Cached:       24GB   ← Page cache (file data)
# SwapCached:    0MB   ← Anonymous pages in both RAM and swap
# Active:       16GB   ← Recently used pages (active LRU list)
# Inactive:      8GB   ← Less recently used (candidates for reclaim)
# Active(anon):  6GB   ← Active anonymous pages
# Inactive(anon): 1GB  ← Inactive anonymous pages (next to be swapped)
# Active(file): 10GB   ← Active file cache
# Inactive(file): 7GB  ← Inactive file cache (next to be evicted)
# Unevictable:  512MB  ← Pinned/mlock'd pages (not on LRU)
# Mlocked:      512MB  ← mlock'd pages
# SwapTotal:     16GB  ← Total swap space
# SwapFree:      14GB  ← Free swap space
# Dirty:         10MB  ← Pages modified, awaiting writeback
# Writeback:      2MB  ← Pages being written to disk now
# AnonPages:      7GB  ← Total anonymous pages in RAM
# Mapped:         2GB  ← File pages that are mmap'd by processes
# Shmem:         256MB ← Pages in tmpfs/shared memory
# Slab:          500MB ← Kernel slab caches
# SReclaimable:  400MB ← Reclaimable slab (dentries, inodes)
# SUnreclaim:    100MB ← Non-reclaimable slab
# KernelStack:    24MB ← Kernel stacks (8-16KB × nr_threads)
# PageTables:     48MB ← Page table memory
# NFS_Unstable:    0MB ← NFS pages sent but not yet committed to server
# AnonHugePages: 1GB   ← Anonymous transparent huge pages
```

---

## 13. Memory Pressure Monitoring

```bash
# PSI (Pressure Stall Information) — Linux 4.20+:
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
# "some": at least one task is stalled on memory
# "full": all tasks are stalled on memory (serious!)
# avg10/60/300: 10-second, 1-minute, 5-minute moving averages

# Application-level memory pressure notification:
# Use pollfd with /proc/pressure/memory for notifications
int fd = open("/proc/pressure/memory", O_RDWR);
write(fd, "some 5000 1000000", 18);  // Alert if 5ms stall in 1s window
poll(&pfd, 1, -1);  // Wait for notification

# vmstat: real-time memory stats:
vmstat 1
# procs  ------memory------  ---swap-- -----io---- -system-- ------cpu-----
#  r  b    swpd   free   buff  cache    si   so    bi    bo   in   cs us sy id wa
#  1  0  123456 234567 12345 2345678    0    0    23    45  500 1234  5  2 90  3
# si = swap-in pages/sec
# so = swap-out pages/sec
# bi = blocks in/sec (disk reads including swap-in)
# bo = blocks out/sec (disk writes including swap-out)
```

---

## 14. `cgroup` Memory Controller

cgroups allow limiting and monitoring memory per group of processes:

```bash
# Create a cgroup with memory limit:
mkdir /sys/fs/cgroup/myapp
echo "2147483648" > /sys/fs/cgroup/myapp/memory.max   # 2GB limit
echo "1073741824" > /sys/fs/cgroup/myapp/memory.swap.max  # 1GB swap limit

# Add a process:
echo $PID > /sys/fs/cgroup/myapp/cgroup.procs

# When the limit is hit:
# - Kernel reclaims within the cgroup first
# - If not enough: OOM kill within cgroup

# Stats:
cat /sys/fs/cgroup/myapp/memory.current   # Current usage
cat /sys/fs/cgroup/myapp/memory.stat      # Detailed breakdown

# Swap behavior per cgroup:
echo 0 > /sys/fs/cgroup/myapp/memory.swap.max  # Disable swap for this group
```

---

## 15. Mental Model Checkpoint

After Day 30, you should be able to:

1. Name the four LRU lists and what pages live on each.
2. When does `kswapd` wake up and when does it go back to sleep?
3. What does `swappiness=0` do vs `swappiness=60` vs `swappiness=100`?
4. Trace the swap-out path for an anonymous page from LRU to disk.
5. What is a swap entry in a PTE? What information does it encode?
6. What is reverse mapping? Why does it differ for anonymous vs file-backed pages?
7. What is `zswap` and how does it work differently from regular swap?
8. What is PSI (Pressure Stall Information) and what does the "full" category indicate?

---

## Key Source Files

```bash
mm/vmscan.c              # kswapd, shrink_page_list(), shrink_lruvec()
mm/swap.c                # LRU list management, add_to_swap()
mm/swap_state.c          # Swap cache operations
mm/swapfile.c            # Swap partition/file management
mm/rmap.c                # Reverse mapping: try_to_unmap()
mm/memory.c              # do_swap_page() — swap-in
mm/page-writeback.c      # Dirty page tracking and writeback thresholds
include/linux/swap.h     # swp_entry_t, swap type definitions
Documentation/mm/page_reclaim.rst  # Official reclaim documentation
```

---

## Summary

Page reclaim is the kernel's memory management mechanism that frees physical pages when needed. It works on two LRU hierarchies (anonymous + file) × two activity levels (active + inactive):

- **File pages**: evicted by dropping from cache (no I/O if clean, writeback if dirty)
- **Anonymous pages**: must be swapped out to swap space before eviction

`kswapd` provides background reclaim when free memory falls below `WMARK_LOW`. Direct reclaim blocks the allocating task when memory is critically low. The `swappiness` knob balances between these two types of reclaim.

Tomorrow: the OOM killer — when all reclaim fails and the kernel must terminate a process.
