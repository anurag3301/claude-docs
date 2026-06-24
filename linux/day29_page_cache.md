# Day 29 — Page Cache & File-Backed Memory

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand the page cache — how file data is cached in RAM, `address_space`, readahead, dirty page tracking, and writeback.

---

## 1. What the Page Cache Is

The page cache is the kernel's unified cache for file data. When you `read()` a file, the kernel:
1. Reads pages from disk into RAM
2. Stores them in the page cache (indexed by file + offset)
3. Returns data to user space from this cache

On subsequent reads of the same data: served directly from the page cache — no disk access.

```bash
# See page cache size:
cat /proc/meminfo | grep -E 'Cached|Buffers'
# Buffers:           1234 kB  ← Block device buffer cache (small)
# Cached:          123456 kB  ← Page cache (files)
# SwapCached:           0 kB  ← Anonymous pages in swap that are still in RAM

# On a server with 32GB RAM and lots of files:
# MemTotal:  32768000 kB
# MemFree:    1234567 kB   ← Only 1.2GB "free"
# Cached:    28000000 kB   ← 28GB in page cache!
# MemAvailable: 29000000   ← 29GB available (cache is evictable)

# Drop the page cache (useful for benchmarking):
sudo echo 3 > /proc/sys/vm/drop_caches
# 1 = drop page cache
# 2 = drop dentries and inodes
# 3 = drop all
# WARNING: Forces cold cache — next accesses are slow!
```

---

## 2. `address_space` — The Cache Index

Each file (inode) has an `address_space` that maps file offsets to cached pages:

```c
// include/linux/fs.h
struct address_space {
    struct inode         *host;         // Owner (inode or swapper)
    struct xarray         i_pages;      // XArray mapping pgoff → struct page
    struct rw_semaphore   invalidate_lock; // Protect against cache invalidation
    gfp_t                 gfp_mask;     // GFP flags for page allocation
    atomic_t              i_mmap_writable; // Writable MAP_SHARED count
    struct rb_root_cached i_mmap;       // Tree of VMAs mapping this file
    struct list_head      i_private_list;
    spinlock_t            i_mmap_rwsem;
    unsigned long         nrpages;      // Number of pages cached
    unsigned long         writeback_index; // Next page to write back
    const struct address_space_operations *a_ops; // FS-specific operations
    unsigned long         flags;        // AS_EIO, AS_ENOSPC, etc.
    errseq_t              wb_err;       // Writeback error tracking
};

// The key operations table (FS implements these):
struct address_space_operations {
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*read_folio)(struct file *, struct folio *);   // Read one page from disk
    int (*writepages)(struct address_space *, struct writeback_control *);
    bool (*dirty_folio)(struct address_space *, struct folio *);
    void (*readahead)(struct readahead_control *);     // Bulk readahead
    int (*write_begin)(struct file *, struct address_space *, loff_t,
                       unsigned, struct page **, void **);
    int (*write_end)(struct file *, struct address_space *, loff_t,
                     unsigned, unsigned, struct page *, void *);
    sector_t (*bmap)(struct address_space *, sector_t);   // VFS → block mapping
    void (*invalidate_folio)(struct folio *, size_t offset, size_t len);
    bool (*release_folio)(struct folio *, gfp_t);
    void (*free_folio)(struct folio *);
    int (*direct_IO)(struct kiocb *, struct iov_iter *iter);
    int (*migrate_folio)(struct address_space *, struct folio *dst,
                         struct folio *src, enum migrate_mode);
    void (*swap_activate)(struct swap_info_struct *, struct file *, sector_t *);
    void (*swap_deactivate)(struct file *);
    int (*swap_rw)(struct kiocb *iocb, struct iov_iter *iter);
};
```

---

## 3. Page Cache Lookup and Insertion

```c
// mm/filemap.c

// Find a page in the cache:
struct page *find_get_page(struct address_space *mapping, pgoff_t offset)
{
    struct page *page;
    
    rcu_read_lock();
    page = xa_load(&mapping->i_pages, offset);  // XArray lookup: O(log n) or O(1)
    if (xa_is_value(page))                       // Shadow entry (formerly cached)
        page = NULL;
    if (page && !get_page_unless_zero(page))     // Safe: don't take zero-ref page
        page = NULL;
    rcu_read_unlock();
    
    return page;
}

// Insert a page into the cache:
int add_to_page_cache_lru(struct page *page, struct address_space *mapping,
                           pgoff_t offset, gfp_t gfp_mask)
{
    int ret;
    
    // Insert into XArray:
    xa_lock_irq(&mapping->i_pages);
    ret = __add_to_page_cache_locked(page, mapping, offset, gfp_mask, NULL);
    xa_unlock_irq(&mapping->i_pages);
    
    if (!ret) {
        // Add to LRU list:
        lru_cache_add(page);  // Adds to inactive LRU list
    }
    
    return ret;
}
```

---

## 4. The `read()` System Call Through the Page Cache

```
sys_read(fd, buf, count)
    │
    ▼
vfs_read() → __vfs_read()
    │
    ▼
file->f_op->read_iter()   (e.g., ext4_file_read_iter or generic_file_read_iter)
    │
    ▼
generic_file_read_iter()
    │
    ▼
filemap_read()
    │
    ├─ For each page needed:
    │   find_get_page(mapping, page_offset)
    │   ├─ HIT: page in cache → copy to user buffer
    │   └─ MISS: 
    │       readahead: read surrounding pages too
    │       page_cache_read() → read_folio() → disk I/O
    │       wait_on_page_locked()  ← wait for I/O
    │       copy to user buffer
    │
    └─ copy_page_to_iter() → copy_to_user()
```

---

## 5. Readahead

Readahead pre-loads pages into the page cache before they're needed, hiding disk latency:

```c
// mm/readahead.c
void page_cache_sync_readahead(struct address_space *mapping,
                               struct file_ra_state *ra,
                               struct file *file,
                               pgoff_t offset, unsigned long req_size)
{
    // Calculate how many pages to read ahead:
    // Based on: ra->ra_pages (max), previous access pattern, request size
    unsigned long size = ra->ra_pages;
    
    // Trigger async readahead:
    ondemand_readahead(mapping, ra, file, offset, req_size);
}

// The readahead state per open file:
struct file_ra_state {
    pgoff_t start;           // Where current window starts
    unsigned int size;       // Window size (in pages)
    unsigned int async_size; // Async ahead size
    unsigned int ra_pages;   // Maximum readahead size
    unsigned int mmap_miss;  // Cache miss stat for mmap access
    loff_t prev_pos;         // Previous file position
};
```

```bash
# Configure readahead:
blockdev --getra /dev/sda    # Current readahead for block device (in 512B sectors)
blockdev --setra 4096 /dev/sda  # Set to 4096 × 512B = 2MB

# Per-file readahead via posix_fadvise():
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);  # Large readahead
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);      # Disable readahead
posix_fadvise(fd, offset, len, POSIX_FADV_WILLNEED); # Prefetch now

# madvise for mmap'd files:
madvise(addr, len, MADV_SEQUENTIAL);   # Enable readahead
madvise(addr, len, MADV_WILLNEED);     # Prefetch now (async)
```

---

## 6. Dirty Pages and Write-Back

When user space writes to a file-backed mmap or `write()` syscall modifies a cached page:

1. The page is marked **dirty** (`PG_dirty` flag set)
2. The page is moved to the **dirty list**
3. Eventually, the kernel writes dirty pages back to disk (**writeback**)

```c
// Marking a page dirty:
void set_page_dirty(struct page *page)
{
    struct address_space *mapping = page->mapping;
    
    // Set the PG_dirty flag:
    if (!TestSetPageDirty(page)) {
        // First time this page is dirtied:
        if (mapping)
            __set_page_dirty(page, mapping, 1);
        // Adds to mapping->dirty_pages list
        // Wakes up bdi_writeback if needed
    }
}

// For mmap writes: CPU sets PG_dirty via hardware (D bit in PTE)
// Kernel reads PTE's dirty bit in:
//   - page reclaim (before evicting a dirty page, must write back)
//   - writeback (collect dirty pages from the address_space)
```

---

## 7. Writeback — `pdflush` and `wb_writeback`

```bash
# Dirty page thresholds (in percentage of total memory):
cat /proc/sys/vm/dirty_ratio          # 20 = hard limit (20% of RAM)
cat /proc/sys/vm/dirty_background_ratio  # 10 = background writeback starts at 10%
cat /proc/sys/vm/dirty_expire_centisecs  # 3000 = pages are written after 30s
cat /proc/sys/vm/dirty_writeback_centisecs # 500 = writeback runs every 5s

# Absolute byte limits (take effect if smaller than ratio-based limits):
cat /proc/sys/vm/dirty_bytes          # 0 = not set
cat /proc/sys/vm/dirty_background_bytes  # 0 = not set

# Current dirty pages:
cat /proc/meminfo | grep -i dirty
# Dirty:             1234 kB  ← Currently dirty (awaiting writeback)
# Writeback:           56 kB  ← Currently being written to disk
```

```c
// mm/page-writeback.c — The writeback mechanism:

// Per-block-device writeback thread (bdi_writeback):
// Created by: alloc_bdi_with_capabilities()
// Thread: wb_workfn() (runs when woken)
// Process: bdi_writeback for each backing device (BDI)

static long wb_writeback(struct bdi_writeback *wb,
                          struct wb_writeback_work *work)
{
    long nr_pages = work->nr_pages;  // How many pages to write
    
    while (nr_pages > 0 && !list_empty(&wb->b_io)) {
        // Take next inode from writeback queue:
        struct inode *inode = wb_inode(wb->b_io.prev);
        
        // Write dirty pages for this inode:
        writeback_single_inode(inode, wb, work);
        nr_pages -= wrote;
    }
    return wrote;
}

// Triggered by:
// 1. Background: periodic timer (dirty_writeback_centisecs)
// 2. Threshold: dirty pages exceed dirty_background_ratio
// 3. Direct: dirty pages exceed dirty_ratio (direct reclaim does writeback)
// 4. fsync(): explicit flush of one file
// 5. sync(): flush all dirty data
```

---

## 8. `fsync()`, `msync()`, and Write Ordering

```c
// Force immediate writeback of one file:
int fsync(int fd);
// Waits for all dirty pages of this file to reach disk

// Force writeback AND metadata (inode, directory entries):
int fdatasync(int fd);
// Like fsync but no metadata (faster)

// Force writeback of a mmap'd region:
int msync(void *addr, size_t length, int flags);
// MS_SYNC: wait for completion
// MS_ASYNC: just mark dirty, let background writeback handle it
// MS_INVALIDATE: invalidate cached data from disk

// Kernel fsync path:
// sys_fsync → do_fsync → vfs_fsync → file->f_op->fsync()
// e.g., ext4_sync_file():
//   1. Write all dirty pages for this inode
//   2. Write journal/log (ext4 journal commit)
//   3. Issue FLUSH or FUA command to storage
//   4. Wait for completion
```

---

## 9. Page Cache Eviction

When memory is low, the kernel evicts (reclaims) page cache pages. Only clean pages can be evicted directly; dirty pages must be written back first.

```c
// mm/vmscan.c — page reclaim:
static unsigned long shrink_page_list(struct list_head *page_list,
                                       struct pglist_data *pgdat,
                                       struct scan_control *sc, ...)
{
    struct page *page;
    
    list_for_each_entry_safe(page, next, page_list, lru) {
        // Is this page dirty?
        if (PageDirty(page)) {
            if (page_is_file_lru(page)) {
                // File-backed dirty: schedule writeback, skip for now
                SetPageReclaim(page);
                writeback_page(page);  // Async writeback
                continue;
            }
        }
        
        // Can we free this page?
        if (PageWriteback(page)) {
            // Wait for ongoing writeback:
            wait_on_page_writeback(page);
        }
        
        // Remove from page tables (TLB flush may be needed):
        if (!remove_mapping(mapping, page))
            continue;
        
        // Page is clean and unmapped: free it!
        free_page = page;
        list_add(&page->lru, &free_pages);
    }
    
    release_pages(free_pages);  // Return to buddy allocator
    return freed_count;
}
```

---

## 10. Folios — The Modern Page Cache Unit

Linux 5.16+ introduced **folios** as the abstraction for page cache entries. A folio is a power-of-2 contiguous set of pages treated as one unit:

```c
// include/linux/mm_types.h
// A folio is essentially a struct page with extra metadata for multi-page folios
struct folio {
    union {
        struct page page;
        // ...
    };
    // For large folios (order > 0):
    // - Represent 2^order pages as one cache unit
    // - Single dirty bit, single lock, single refcount for all pages
};

// API:
struct folio *filemap_alloc_folio(gfp_t gfp, unsigned int order);
void folio_put(struct folio *folio);
bool folio_trylock(struct folio *folio);
void folio_unlock(struct folio *folio);
bool folio_test_dirty(const struct folio *folio);
void folio_mark_dirty(struct folio *folio);

// Large folios improve throughput for sequential I/O by:
// - Reducing metadata overhead (one folio vs 512 pages for 2MB)
// - Reducing TLB pressure (large folio maps contiguous pages)
// - Reducing locking overhead (one lock per folio)
```

---

## 11. Direct I/O — Bypassing the Page Cache

For databases and applications that do their own caching:

```c
// Open with O_DIRECT:
int fd = open("database.db", O_RDWR | O_DIRECT);

// Constraints:
// - User buffer must be aligned to block size (usually 512 or 4096 bytes)
// - File offset must be aligned to block size
// - Transfer size must be multiple of block size

// Kernel path: O_DIRECT bypasses page cache entirely:
// generic_file_read_iter → kiocb->ki_flags & IOCB_DIRECT → direct_IO()
// → submit_bio() directly to block layer

// Advantages:
// - No double buffering (kernel cache + application cache)
// - Predictable latency (no page cache eviction surprises)
// - Good for write-once, read-once data (don't pollute cache)

// Disadvantages:
// - No readahead
// - No write coalescing
// - Higher latency for small reads (no cache warm-up effect)
```

---

## 12. `mmap()` vs `read()` Performance

```c
// mmap approach:
// 1. mmap(file) → create VMA, no pages loaded
// 2. Access data → page fault → kernel loads from disk
// 3. Subsequent access → TLB hit or page cache hit (no syscall)

// read() approach:
// 1. read() → kernel reads from disk → copy_to_user()
// 2. Subsequent read() → kernel cache hit → copy_to_user() (still syscall)

// When mmap wins:
// - Many small reads to same data (no syscall per access)
// - Random access (let the OS handle demand loading)
// - Shared read-only data between processes

// When read() wins:
// - Sequential streaming (readahead more effective with read())
// - O_DIRECT (mmap can't bypass cache)
// - Need to control buffer size precisely
```

---

## 13. Page Cache Statistics

```bash
# Detailed page cache view:
# (install linux-tools-$(uname -r) for fincore, vmtouch)
vmtouch -v /path/to/largefile
# [OOOOOOOOOOOO.........]   60.0% (12345/20000 pages in core)

# Which pages of a file are in cache:
fincore /path/to/file

# Cache usage by process:
sudo bash -c "cat /proc/*/smaps" | awk '/^Cached:/{sum+=$2} END{print sum/1024 "MB"}'

# perf: trace page cache hits/misses:
sudo perf stat -e 'block:block_rq_issue,filemap:mm_filemap_add_to_page_cache' -a sleep 5
# block_rq_issue: actual disk requests (misses)
# mm_filemap_add_to_page_cache: page cache fills (first access per page)

# bpftrace: trace cache misses for a specific file:
sudo bpftrace -e '
kprobe:__do_page_cache_readahead {
    printf("readahead: %s offset=%d\n",
        str(((struct address_space*)arg0)->host->i_sb->s_id),
        arg2);
}'
```

---

## 14. tmpfs — Memory as Filesystem

`tmpfs` (and `ramfs`) stores files entirely in the page cache:

```bash
# Mount tmpfs:
mount -t tmpfs -o size=1G tmpfs /tmp
df -h /tmp
# tmpfs           1.0G  0 1.0G   0% /tmp

# Unlike ramfs:
# - Has a size limit
# - Pages can be swapped to disk (unlike ramfs which pins in RAM)
# - /dev/shm, /run are tmpfs

# tmpfs internals:
# Backed by inode whose address_space lives entirely in RAM (no backing store)
# writepage() for tmpfs: writes to swap (if configured)
# Page cache == filesystem data for tmpfs
```

---

## 15. Mental Model Checkpoint

After Day 29, you should be able to:

1. What is the page cache and what data structure indexes it?
2. Trace a `read()` from a file, showing when the page cache is checked.
3. What is readahead and what state tracks the readahead window?
4. Explain dirty page tracking: what marks a page dirty, what triggers writeback?
5. What are the dirty page thresholds and what happens when they're exceeded?
6. What is `fsync()` at the kernel level? What must ext4 do to satisfy it?
7. When would you use `O_DIRECT` and what alignment constraints does it impose?
8. What is a folio and why was it introduced?

---

## Key Source Files

```bash
mm/filemap.c              # filemap_fault(), filemap_read(), find_get_page()
mm/readahead.c            # Readahead algorithm
mm/page-writeback.c       # Dirty page tracking, writeback thresholds, balance_dirty_pages
mm/vmscan.c               # Page cache eviction in shrink_page_list()
include/linux/fs.h        # struct address_space, address_space_operations
fs/ext4/inode.c           # ext4's a_ops: readpage, writepage, write_begin/end
mm/truncate.c             # invalidate_mapping_pages(), truncate_inode_pages()
Documentation/mm/page_cache.rst  # Official page cache documentation
```

---

## Summary

The page cache is a unified RAM buffer for all file I/O. Its core structure is the `address_space`, which uses an XArray to map `(inode, offset)` pairs to physical pages.

Key flows:
- **Read**: check XArray → if miss, allocate page, submit I/O, wait, copy to user
- **Write (write syscall)**: write to page cache → mark dirty → background writeback flushes to disk
- **Write (mmap)**: CPU sets PTE D bit → kernel collects dirty pages via reverse mapping → writeback
- **Eviction**: LRU-based reclaim, dirty pages must be written back first, clean pages freed immediately

Writeback is governed by three controls: `dirty_background_ratio` (start background writeback), `dirty_ratio` (block writers in direct reclaim), and `dirty_expire_centisecs` (maximum page age before writeback).

Tomorrow: swap and page reclaim — how the kernel decides which pages to evict and where they go.
