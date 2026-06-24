# Day 32 — Huge Pages & Transparent Huge Pages (THP)

> **Estimated read time:** 90 minutes  
> **Goal:** Understand hugetlbfs (static huge pages), Transparent Huge Pages, `khugepaged`, compaction, and the performance tradeoffs of each approach.

---

## 1. Why Huge Pages?

Every 4KB page mapped by a process requires one TLB entry. A process with 1GB of heap uses:
- 4KB pages: 262,144 TLB entries (impossible — TLB has ~1,000–4,000 entries)
- 2MB pages: 512 TLB entries (fits easily)
- 1GB pages: 1 TLB entry

TLB misses require a page table walk (3–4 memory accesses). A large working set with 4KB pages means constant TLB misses — a major performance bottleneck for databases, JVMs, and HPC workloads.

```bash
# Measure TLB miss impact:
sudo perf stat -e dTLB-load-misses,dTLB-loads ./myprogram
# dTLB-load-misses   12,345,678   # 2% miss rate → significant
# dTLB-loads        617,283,950

# With huge pages the same program:
# dTLB-load-misses       23,456   # 0.004% miss rate → negligible
```

---

## 2. HugeTLBFS — Static Huge Pages

The original huge page mechanism: pre-allocate huge pages at boot (or runtime), expose them through a filesystem.

### 2.1 Reserving Huge Pages

```bash
# Check available huge page sizes on this system:
ls /sys/kernel/mm/hugepages/
# hugepages-1048576kB/   ← 1GB pages
# hugepages-2048kB/      ← 2MB pages

# Reserve 2MB huge pages:
echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
# Reserves 512 × 2MB = 1GB of huge pages

# Reserve at boot (better: avoids fragmentation):
# In /etc/default/grub:
# GRUB_CMDLINE_LINUX="hugepages=512 hugepagesz=2M default_hugepagesz=2M"

# Reserve 1GB pages (must be at boot — kernel needs contiguous 1GB physical blocks):
# GRUB_CMDLINE_LINUX="hugepages=8 hugepagesz=1G default_hugepagesz=1G"

# Check reservation status:
cat /proc/meminfo | grep Huge
# AnonHugePages:    524288 kB  ← THP anonymous pages
# ShmemHugePages:        0 kB
# HugePages_Total:     512     ← Reserved huge pages
# HugePages_Free:      400     ← Available for allocation
# HugePages_Rsvd:      100     ← Reserved but not yet mapped
# HugePages_Surp:        0     ← Surplus (over nr_hugepages)
# Hugepagesize:       2048 kB  ← 2MB each
# Hugetlb:         1048576 kB  ← Total memory in huge pages = 512 × 2MB
```

### 2.2 Using HugeTLBFS Pages

```c
// Method 1: MAP_HUGETLB (anonymous huge pages):
void *addr = mmap(NULL, 2 * 1024 * 1024,   // Must be multiple of huge page size
                  PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                  -1, 0);
if (addr == MAP_FAILED) {
    // Check: ulimit -l (memlock limit) and /proc/meminfo HugePages_Free
    perror("mmap huge page failed");
}

// Method 2: hugetlbfs filesystem:
// Mount:
// mount -t hugetlbfs -o pagesize=2M hugetlbfs /mnt/huge

int fd = open("/mnt/huge/myfile", O_CREAT | O_RDWR, 0600);
ftruncate(fd, 2 * 1024 * 1024);  // 2MB
void *addr = mmap(NULL, 2*1024*1024, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

// Method 3: SysV shared memory with huge pages:
int shmid = shmget(IPC_PRIVATE, size,
                   SHM_HUGETLB | IPC_CREAT | SHM_R | SHM_W);
void *addr = shmat(shmid, NULL, 0);
```

### 2.3 Kernel Implementation of HugeTLBFS

```c
// mm/hugetlb.c
// Huge pages are pre-allocated into a pool:
struct hstate {
    int next_nid_to_alloc;
    int next_nid_to_free;
    unsigned int order;          // log2(pages per huge page) = 9 for 2MB
    unsigned int demote_order;   // Can demote to smaller huge page
    unsigned long mask;          // ~(huge_page_size - 1)
    unsigned long max_huge_pages; // Requested reservation
    unsigned long nr_huge_pages;  // Actually allocated
    unsigned long free_huge_pages; // Available
    unsigned long resv_huge_pages; // Reserved (alloc promised, not mapped yet)
    unsigned long surplus_huge_pages; // Over limit
    unsigned long nr_huge_pages_node[MAX_NUMNODES];
    unsigned long free_huge_pages_node[MAX_NUMNODES];
    unsigned long surplus_huge_pages_node[MAX_NUMNODES];
    struct list_head hugepage_activelist;  // In-use huge pages
    struct list_head hugepage_freelists[MAX_NUMNODES];  // Free pool
    // ...
};

// Huge page allocation:
struct page *alloc_huge_page(struct vm_area_struct *vma,
                              unsigned long addr, int avoid_reserve)
{
    struct hstate *h = hstate_vma(vma);
    struct page *page;
    
    // Take from free list (no buddy allocator involved):
    page = dequeue_huge_page_vma(h, vma, addr, avoid_reserve, ...);
    if (!page)
        return ERR_PTR(-ENOMEM);
    
    // Split the huge page into the PMD-level mapping:
    // (actually sets up the compound page structure)
    prep_new_huge_page(h, page, page_to_nid(page));
    
    return page;
}
```

### 2.4 NUMA and Huge Pages

```bash
# Per-NUMA-node huge page reservations:
cat /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
cat /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages

# Reserve on specific node:
echo 256 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 256 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```

---

## 3. Transparent Huge Pages (THP)

THP automatically promotes 4KB page clusters to 2MB huge pages — **without application changes**. The kernel handles this transparently.

### 3.1 THP Modes

```bash
# Global THP setting:
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
# ↑ Current mode (in brackets)

# "always":  THP for all eligible anonymous mappings (aggressive)
# "madvise": THP only where madvise(MADV_HUGEPAGE) is called
# "never":   THP disabled entirely

# Set mode:
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled

# "madvise" is the safest default for production:
# - Databases can opt out (they do their own page management)
# - Applications that benefit opt in with madvise(MADV_HUGEPAGE)
```

### 3.2 THP Defrag Policy

```bash
# Defrag: how aggressively should the kernel defragment memory to create THPs?
cat /sys/kernel/mm/transparent_hugepage/defrag
# always defer defer+madvise [madvise] never

# "always":       Block allocation until THP is available (latency spikes!)
# "defer":        Try in background, fall back to 4KB if not available
# "defer+madvise": Background for "always" regions, sync for madvise regions
# "madvise":      Synchronous defrag only for madvise(MADV_HUGEPAGE) regions
# "never":        Never defrag for THP

echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag

# Per-VMA control:
madvise(addr, len, MADV_HUGEPAGE);   // Enable THP for this region
madvise(addr, len, MADV_NOHUGEPAGE); // Disable THP for this region
```

### 3.3 THP Fault Path

```c
// mm/huge_memory.c
// On page fault, if conditions are right, allocate a 2MB page:

static vm_fault_t do_huge_pmd_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct page *page;
    
    // Check eligibility:
    if (!transhuge_vma_suitable(vma, vmf->address))
        return VM_FAULT_FALLBACK;  // Fall back to 4KB
    
    // Allocate 2MB compound page (order-9 from buddy allocator):
    page = alloc_hugepage_vma(GFP_TRANSHUGE, vma, vmf->address, HPAGE_PMD_ORDER);
    if (unlikely(!page)) {
        // Allocation failed: fall back to 4KB pages
        return VM_FAULT_FALLBACK;
    }
    
    // Zero the 2MB page:
    clear_huge_page(page, vmf->address, HPAGE_PMD_NR);  // HPAGE_PMD_NR = 512
    
    // Install the PMD entry (2MB mapping):
    pmd_t entry = mk_huge_pmd(page, vma->vm_page_prot);
    set_pmd_at(vma->vm_mm, vmf->address & HPAGE_PMD_MASK, vmf->pmd, entry);
    // This single PMD entry maps 512 pages at once!
    
    return 0;
}
```

---

## 4. `khugepaged` — Background THP Promotion

`khugepaged` is a kernel thread that scans process address spaces and promotes existing 4KB page clusters to 2MB THP pages:

```bash
# khugepaged is always running when THP is enabled:
ps aux | grep khugepaged
# root      42  0.0  0.0  [khugepaged]

# Configure how aggressively it scans:
cat /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
# 10000 (10 seconds between scans)
echo 1000 > /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs

cat /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan
# 4096 (pages to scan per pass)
echo 8192 > /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan

# Monitor khugepaged activity:
cat /sys/kernel/mm/transparent_hugepage/khugepaged/pages_collapsed
# 1234  ← Number of 4KB clusters promoted to 2MB THP
cat /sys/kernel/mm/transparent_hugepage/khugepaged/full_scans
# 567   ← Complete scans of all VMAs
```

### 4.1 Promotion Algorithm

```c
// mm/khugepaged.c
static int khugepaged_scan_mm_slot(unsigned int pages, int *result,
                                    struct mm_slot *mm_slot)
{
    struct mm_struct *mm = mm_slot->slot.mm;
    
    // Walk all VMAs in this mm:
    for_each_vma(vmi, vma) {
        // Skip VMAs that aren't eligible for THP:
        if (!hugepage_vma_check(vma, vma->vm_flags, false, false, true))
            continue;
        
        // Scan 2MB-aligned windows within this VMA:
        for (address = ...; address < vma->vm_end; address += HPAGE_PMD_SIZE) {
            // Is this 2MB window fully backed by 4KB pages?
            if (!khugepaged_scan_pmd(mm, vma, address, ...))
                continue;
            
            // Collapse: allocate a 2MB page, copy all 512 4KB pages:
            collapse_huge_page(mm, address, ...);
            // collapse_huge_page():
            //   1. alloc_pages(GFP_TRANSHUGE, HPAGE_PMD_ORDER)  ← 2MB alloc
            //   2. For each of the 512 pages: copy_user_highpage()
            //   3. Replace 512 PTEs with one PMD entry
            //   4. Free 512 old 4KB pages
        }
    }
}
```

---

## 5. Memory Compaction for Huge Pages

The buddy allocator needs 512 contiguous free 4KB pages to allocate one 2MB THP. Memory compaction migrates movable pages to create contiguous free regions.

```c
// mm/compaction.c
static int compact_zone(struct compact_control *cc, ...)
{
    // Two cursors: free_pfn (from top) and migrate_pfn (from bottom)
    // Scan backward from top: find free pages
    // Scan forward from bottom: find movable pages to migrate
    // Migrate pages from bottom to free space at top
    // → Creates contiguous free region in the middle
    
    while (cc->migrate_pfn < cc->free_pfn) {
        // Isolate movable pages near migrate_pfn:
        isolate_migratepages_block(cc, low_pfn, block_end_pfn, ...);
        
        // Migrate them to free area near free_pfn:
        migrate_pages(&cc->migratepages, compaction_alloc, ...);
        // migrate_pages: copies page content + updates reverse mappings + PTEs
    }
}
```

```bash
# Monitor compaction:
cat /proc/vmstat | grep compact
# compact_migrate_scanned  123456   ← Pages scanned for migration
# compact_free_scanned     234567   ← Free pages found
# compact_isolated         12345    ← Pages isolated for migration
# compact_stall            1234     ← Times allocator stalled for compaction
# compact_fail              567     ← Failed compaction attempts
# compact_success          1000     ← Successful compaction attempts

# High compact_stall with compact_fail > compact_success means:
# System can't create 2MB contiguous regions → THP unlikely to work
# Solution: reserve huge pages at boot, or lower THP defrag to "defer"
```

---

## 6. THP for File-Backed Mappings

Linux 6.1+ added THP support for file-backed mappings (not just anonymous):

```bash
# File THP (experimental, kernel 6.1+):
cat /sys/kernel/mm/transparent_hugepage/enabled_file
# [never] madvise always

# Enable file THP (use with caution — increases memory usage):
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled_file

# madvise a file mapping to use THP:
int fd = open("/path/to/largefile", O_RDONLY);
void *addr = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
madvise(addr, size, MADV_HUGEPAGE);  // Request file THP
```

---

## 7. HugePage for Specific Applications

### 7.1 PostgreSQL

```bash
# postgresql.conf:
huge_pages = try   # Use huge pages if available, fall back if not
# huge_pages = on  # Require huge pages (fail if not available)

# Required: shared memory must be mappable with huge pages:
# Server must have enough huge pages for shared_buffers:
# shared_buffers = 8GB → need 8GB / 2MB = 4096 huge pages

echo 4096 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

### 7.2 Java / JVM

```bash
# JVM options to use huge pages:
java -XX:+UseLargePages \
     -XX:LargePageSizeInBytes=2m \
     -Xmx8g \
     MyApp

# Or: use THP with madvise (JVM calls madvise on heap regions):
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
java -XX:+UseTransparentHugePages -Xmx8g MyApp
```

### 7.3 Redis

```bash
# Redis recommends DISABLING THP for latency reasons:
# THP causes latency spikes when khugepaged consolidates pages
# during production operations (copy-on-write on fork for persistence)

echo never > /sys/kernel/mm/transparent_hugepage/enabled
# Redis startup warning: "you have Transparent Huge Pages (THP) support enabled
# in your kernel. This will create latency and memory usage issues with Redis."

# The problem:
# Redis forks for RDB/AOF saves
# Parent modifies data → COW on 2MB THP = copy entire 2MB at once
# vs 4KB page: copy only 4KB when modified
# Result: memory usage 2× + latency spikes
```

---

## 8. THP Statistics

```bash
# Per-process THP usage:
cat /proc/$$/smaps | grep -i 'anon\|huge'
# AnonHugePages:    4194304 kB  ← 4MB of THP in this VMA
# FilePmdMapped:    2097152 kB  ← 2MB of file THP in this VMA

# System-wide THP:
cat /proc/meminfo | grep -i huge
# AnonHugePages:   2097152 kB   ← 1GB of anonymous THP
# ShmemHugePages:        0 kB
# ShmemPmdMapped:        0 kB
# FileHugePages:         0 kB
# FilePmdMapped:         0 kB
# HugePages_Total:       0      ← Static huge pages (hugetlbfs)
# HugePages_Free:        0
# Hugepagesize:       2048 kB

# vmstat for THP events:
cat /proc/vmstat | grep thp
# thp_fault_alloc          1234  ← THP allocated on fault
# thp_fault_fallback       2345  ← THP fault fell back to 4KB
# thp_fault_fallback_charge 12   ← Fell back due to memcg charge
# thp_collapse_alloc        567  ← khugepaged collapsed to THP
# thp_collapse_alloc_failed  23  ← khugepaged collapse failed
# thp_file_alloc             89  ← File THP allocated
# thp_file_mapped            45  ← File THP mapped
# thp_split_page            678  ← THP split back to 4KB pages
# thp_split_page_failed      12  ← THP split failed
# thp_deferred_split_page   234  ← Deferred THP split (partial unmap)
# thp_split_pmd             678  ← PMD split (THP unmapped partially)
# thp_scan_exceed_none_pte   34  ← khugepaged scan hit non-THP area
# thp_zero_page_alloc         5  ← THP zero page allocated
```

---

## 9. 1GB Pages (PUD-Level)

```bash
# Reserve 1GB pages (kernel command line only — can't reserve after boot):
# GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=4"

# 1GB pages need 30-bit aligned physical addresses (1GB-aligned)
# Best reserved at boot before memory fragmentation

# Use (same API as 2MB, just larger):
void *addr = mmap(NULL, 1024*1024*1024,
                  PROT_READ|PROT_WRITE,
                  MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB|
                  (30 << MAP_HUGE_SHIFT),  // Specify 1GB (2^30)
                  -1, 0);

# Check if CPU supports 1GB pages:
grep pdpe1gb /proc/cpuinfo
# If listed: 1GB TLB entries supported
```

---

## 10. THP Splitting

THP pages can be split back into 4KB pages when needed:

```c
// When THP must be split:
// 1. munmap() covering only part of a THP → split the THP
// 2. mprotect() changing permissions on part of a THP
// 3. migrate_page() for NUMA balancing (can't migrate 2MB atomically)
// 4. Reclaim: THP can be split to reclaim individual 4KB pages

// mm/huge_memory.c
int split_huge_pmd(struct vm_area_struct *vma, pmd_t *pmd, unsigned long address)
{
    // Allocate 512 PTE entries to replace the single PMD entry:
    pgtable_t pgtable = pgtable_trans_huge_withdraw(mm, pmd);
    
    // Fill in all 512 PTEs:
    for (i = 0; i < HPAGE_PMD_NR; i++) {
        struct page *page = pmd_page(*pmd) + i;
        pte_t pte = mk_pte(page, vma->vm_page_prot);
        set_pte_at(mm, address + i*PAGE_SIZE, &pte_table[i], pte);
    }
    
    // Replace PMD entry with pointer to PTE table:
    pmd_populate(mm, pmd, pgtable);
    
    // TLB flush for the entire 2MB range:
    flush_tlb_range(vma, address, address + HPAGE_PMD_SIZE);
}
```

---

## 11. Choosing the Right Huge Page Strategy

| Scenario | Recommendation |
|----------|---------------|
| Database (PostgreSQL, Oracle) | Static hugetlbfs (`MAP_HUGETLB`) |
| Java heap | THP with `madvise` or `-XX:+UseLargePages` |
| Redis/Memcached | Disable THP (`echo never`) |
| HPC/MPI | Static huge pages, pin to NUMA node |
| General web server | THP with `madvise` (default) |
| Container workloads | THP with `defer+madvise` (avoid compaction stalls) |
| Memory-sensitive | `never` (predictable memory usage) |

---

## 12. Mental Model Checkpoint

After Day 32, you should be able to:

1. Explain why huge pages reduce TLB pressure — what's the math?
2. What is the difference between hugetlbfs and THP? Who controls each?
3. What does `khugepaged` do and how does it "collapse" 4KB pages?
4. Why does Redis recommend disabling THP? What's the COW problem?
5. What is memory compaction and when is it triggered for THP?
6. What does `madvise(addr, len, MADV_HUGEPAGE)` do?
7. When would you use 1GB pages instead of 2MB pages?
8. What does `thp_split_page` in vmstat indicate?

---

## Key Source Files

```bash
mm/huge_memory.c           # THP fault handling, THP splitting
mm/khugepaged.c            # khugepaged: background THP promotion
mm/hugetlb.c               # Static huge pages (hugetlbfs pool)
mm/compaction.c            # Memory compaction for THP
arch/x86/include/asm/pgtable.h  # Huge page PTE/PMD/PUD formats
include/linux/huge_mm.h    # THP definitions, hugepage_vma_check()
Documentation/mm/transhuge.rst   # Official THP documentation
Documentation/admin-guide/mm/hugetlbpage.rst  # HugeTLBFS documentation
```

---

## Summary

Huge pages solve the TLB pressure problem by mapping 2MB (or 1GB) contiguous physical regions with a single TLB entry instead of 512 (or 262,144).

Two mechanisms:
- **HugeTLBFS**: pre-allocated, explicit, requires application changes but guarantees availability — preferred by databases
- **THP**: automatic, transparent to applications, but has pitfalls (COW amplification on fork, compaction stalls, memory waste for small allocations)

The key THP knob is the `defrag` policy: `always` maximizes THP usage at the cost of compaction latency; `defer+madvise` is the production sweet spot; applications like Redis should opt out entirely with `MADV_NOHUGEPAGE` or `echo never`.

Tomorrow: memory debugging tools — kmemleak, KASAN, SLUB debug, and page_owner.
