# Day 27 — Page Tables & TLB

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand the multi-level page table structure, how PTEs encode physical addresses and permissions, TLB operation and shootdowns, and huge page table entries.

---

## 1. Virtual-to-Physical Translation

Every memory access by a CPU core goes through address translation:

```
CPU generates virtual address (VA)
    │
    ▼
MMU (Memory Management Unit) in the CPU
    │
    ├── Check TLB (Translation Lookaside Buffer):
    │     HIT  → Physical Address (PA) → cache/memory
    │     MISS → Page Table Walk
    │               ↓
    │         Read page table entries from memory
    │               ↓
    │         Find physical page frame
    │               ↓
    │         Update TLB
    │               ↓
    │         Physical Address → cache/memory
    │
    └── If page not present → #PF (page fault exception) → kernel
```

The page table is a hardware data structure that the CPU reads automatically (on modern architectures with hardware page table walkers like x86_64). The kernel writes and maintains it; the CPU reads it.

---

## 2. x86_64 Four-Level Page Table

A 64-bit virtual address is split into fields that index four levels of tables:

```
Virtual Address (48 bits used):
  Bits [63:48] = sign extension (must match bit 47 for canonical addresses)
  Bits [47:39] = PGD index (9 bits) → 512 entries
  Bits [38:30] = PUD index (9 bits) → 512 entries
  Bits [29:21] = PMD index (9 bits) → 512 entries
  Bits [20:12] = PTE index (9 bits) → 512 entries
  Bits [11:0]  = Page offset (12 bits) → offset within 4KB page

9 + 9 + 9 + 9 + 12 = 48 bits → 256 TB addressable space
```

```
CR3 (control register)
    │
    ↓ [bits 47:39 of VA]
PGD (Page Global Directory)     4KB page, 512 × 8-byte entries
    │
    ↓ [bits 38:30 of VA]
PUD (Page Upper Directory)      4KB page, 512 × 8-byte entries
    │
    ↓ [bits 29:21 of VA]
PMD (Page Middle Directory)     4KB page, 512 × 8-byte entries
    │
    ↓ [bits 20:12 of VA]
PTE (Page Table Entry)          4KB page, 512 × 8-byte entries
    │
    ↓ [bits 11:0 of VA]
Physical Page + Offset → Physical Address
```

---

## 3. Page Table Entry (PTE) Format

On x86_64, each PTE is a 64-bit value encoding the physical address and flags:

```
64-bit PTE:
  Bits [63]    = XD (eXecute Disable) — if set, cannot execute from this page
  Bits [62:52] = Available for OS use (11 bits)
  Bits [51:12] = Physical Page Frame Number (PFN) — 40 bits = 1TB max PA
  Bits [11:9]  = Available for OS use (3 bits)
  Bit  [8]     = G (Global) — don't flush this entry on CR3 change (e.g., kernel pages)
  Bit  [7]     = PS (Page Size) — if set at PMD level: 2MB huge page
  Bit  [6]     = D (Dirty) — CPU sets this on write
  Bit  [5]     = A (Accessed) — CPU sets this on read or write
  Bit  [4]     = PCD (Page Cache Disable) — bypass cache (for MMIO)
  Bit  [3]     = PWT (Page Write Through) — write-through caching
  Bit  [2]     = U/S (User/Supervisor) — 0=kernel only, 1=user accessible
  Bit  [1]     = R/W (Read/Write) — 0=read-only, 1=writable
  Bit  [0]     = P (Present) — 0=not present (triggers page fault), 1=present
```

```c
// include/linux/pgtable.h — PTE manipulation:
typedef unsigned long pteval_t;
typedef struct { pteval_t pte; } pte_t;

// Constructing PTEs:
pte_t pte = pfn_pte(pfn, prot);   // PFN + protection → PTE

// Checking PTE flags:
pte_present(pte)    // Is bit[0] (P) set?
pte_write(pte)      // Is bit[1] (R/W) set?
pte_exec(pte)       // Is bit[63] (XD) clear?
pte_dirty(pte)      // Is bit[6] (D) set?
pte_young(pte)      // Is bit[5] (A) set? (recently accessed)
pte_none(pte)       // Is entire PTE zero? (no page mapped)

// Modifying PTEs:
pte_t writable = pte_mkwrite(pte);       // Set R/W bit
pte_t readonly = pte_wrprotect(pte);     // Clear R/W bit
pte_t dirty    = pte_mkdirty(pte);       // Set D bit
pte_t clean    = pte_mkclean(pte);       // Clear D bit
pte_t old      = pte_mkyoung(pte);       // Set A bit
pte_t aged     = pte_mkold(pte);         // Clear A bit
```

---

## 4. Kernel Page Table Structures

```c
// include/linux/pgtable.h
// All four levels:
typedef struct { pgdval_t pgd; } pgd_t;   // PGD entry
typedef struct { pudval_t pud; } pud_t;   // PUD entry
typedef struct { pmdval_t pmd; } pmd_t;   // PMD entry
typedef struct { pteval_t pte; } pte_t;   // PTE entry

// Extracting the next level:
pgd_t *pgd = pgd_offset(mm, address);      // mm->pgd[VA[47:39]]
pud_t *pud = pud_offset(p4d, address);     // pgd[VA[38:30]] (skips P4D on 4-level)
pmd_t *pmd = pmd_offset(pud, address);     // pud[VA[29:21]]
pte_t *pte = pte_offset_map(pmd, address); // pmd[VA[20:12]]

// Getting PFN from each level:
unsigned long pfn = pte_pfn(*pte);   // From PTE
unsigned long pfn = pmd_pfn(*pmd);   // From PMD (if 2MB huge page: PS bit set)
unsigned long pfn = pud_pfn(*pud);   // From PUD (if 1GB huge page: PS bit set)
```

### 4.1 Walking the Page Table

```c
// mm/memory.c — follow_pte (simplified):
int follow_pte(struct vm_area_struct *vma, unsigned long address,
               pte_t **ptepp, spinlock_t **ptlp)
{
    struct mm_struct *mm = vma->vm_mm;
    pgd_t *pgd;
    p4d_t *p4d;
    pud_t *pud;
    pmd_t *pmd;
    pte_t *pte;
    
    pgd = pgd_offset(mm, address);
    if (pgd_none(*pgd) || pgd_bad(*pgd))
        goto out;
    
    p4d = p4d_offset(pgd, address);
    if (p4d_none(*p4d) || p4d_bad(*p4d))
        goto out;
    
    pud = pud_offset(p4d, address);
    if (pud_none(*pud) || pud_bad(*pud))
        goto out;
    
    pmd = pmd_offset(pud, address);
    if (pmd_none(*pmd))
        goto out;
    
    if (pmd_trans_huge(*pmd)) {
        // 2MB huge page at PMD level
        // No PTE level needed
    }
    
    pte = pte_offset_map_lock(mm, pmd, address, ptlp);
    if (!pte_present(*pte))
        goto unlock;
    
    *ptepp = pte;
    return 0;  // Success
    
unlock:
    pte_unmap_unlock(pte, *ptlp);
out:
    return -EINVAL;
}
```

---

## 5. `pgd_alloc()` and Page Table Allocation

Page table pages are allocated from the buddy allocator (order 0 = single 4KB page):

```c
// arch/x86/mm/pgtable.c
pgd_t *pgd_alloc(struct mm_struct *mm)
{
    pgd_t *pgd;
    
    // Allocate the top-level page directory:
    pgd = _pgd_alloc();  // alloc_pages(GFP_KERNEL, 0) → 1 page
    if (!pgd)
        return NULL;
    
    // Copy kernel mappings into new PGD:
    // (The kernel is mapped in the upper half of every process's address space)
    pgd_ctor(mm, pgd);
    
    return pgd;
}

// Lower levels are allocated on demand (during page faults):
static int __pte_alloc(struct mm_struct *mm, pmd_t *pmd)
{
    pgtable_t new = pte_alloc_one(mm);  // alloc_page(GFP_KERNEL | __GFP_ZERO)
    if (!new)
        return -ENOMEM;
    
    spin_lock(&mm->page_table_lock);
    if (!pmd_present(*pmd)) {
        pmd_populate(mm, pmd, new);  // Install new PTE page
        new = NULL;
    }
    spin_unlock(&mm->page_table_lock);
    
    if (new)
        pte_free(mm, new);  // Someone else installed a PTE page first
    return 0;
}
```

---

## 6. Setting Page Table Entries

```c
// Setting a PTE atomically:
static void set_pte_at(struct mm_struct *mm, unsigned long addr,
                       pte_t *ptep, pte_t pte)
{
    // On x86_64: simple atomic write (64-bit write is atomic)
    *ptep = pte;
    // If other CPUs might be reading this entry: need TLB flush after
}

// Example: page fault resolution — installing a new physical page:
// mm/memory.c → do_anonymous_page()
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct page *page;
    pte_t entry;
    
    // Allocate a physical page:
    page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
    if (!page)
        return VM_FAULT_OOM;
    
    // Build the PTE:
    entry = mk_pte(page, vma->vm_page_prot);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry));  // Writable + dirty
    
    // Install in page table:
    set_pte_at(mm, vmf->address, vmf->pte, entry);
    
    // Update statistics:
    update_mmu_cache(vma, vmf->address, vmf->pte);
    
    return 0;
}
```

---

## 7. TLB — Translation Lookaside Buffer

The TLB is a cache in the CPU that stores recent virtual→physical translations. Without it, every memory access would require 4 page table reads.

### 7.1 TLB Structure

Modern x86 CPUs have multiple TLBs:
- **L1 ITLB**: 4KB pages only, for instruction fetches (~64 entries per core)
- **L1 DTLB**: 4KB and 2MB pages, for data accesses (~64 entries per core)
- **L2 TLB (STLB)**: shared, larger (~1536 entries per core)
- **Huge page TLBs**: separate smaller TLBs for 2MB and 1GB pages

A TLB **miss** is expensive: 3-4 memory accesses to walk 4 levels of page tables.

### 7.2 TLB Flush Operations

```c
// include/linux/tlbflush.h

// Flush entire TLB (all entries) on current CPU:
local_flush_tlb();
// On x86: writes CR3 with same value (MOV CR3, CR3) → TLB flush

// Flush a specific address on current CPU:
local_flush_tlb_one_user(addr);
// On x86: INVLPG addr

// Flush current mm's TLB across all CPUs:
flush_tlb_mm(mm);
// Sends TLB flush IPIs to all CPUs that have this mm's entries

// Flush a range of addresses:
flush_tlb_range(vma, start, end);

// Flush a single page:
flush_tlb_page(vma, addr);
```

### 7.3 TLB Shootdown

When a kernel on CPU 0 changes a page table entry that another CPU (CPU 1) might have cached in its TLB, CPU 0 must **flush CPU 1's TLB** via an IPI:

```c
// mm/tlb.c
static void do_flush_tlb_all(void *info)
{
    local_flush_tlb();  // Flush this CPU's TLB
}

void flush_tlb_all(void)
{
    on_each_cpu(do_flush_tlb_all, NULL, 1);  // Send IPI to all CPUs, wait
}

// More targeted: flush only CPUs that have this mm loaded:
void flush_tlb_mm_range(struct mm_struct *mm, unsigned long start,
                        unsigned long end, unsigned int stride_shift, bool freed_tables)
{
    // Find CPUs with this mm:
    cpumask_t *cpumask = mm_cpumask(mm);
    
    // Batch the flushes:
    arch_tlbbatch_flush(batch);
    // Sends IPI to each CPU in cpumask
    // Each CPU runs: flush_tlb_func() → invlpg for each address in range
}
```

TLB shootdowns are expensive (IPI + remote TLB flush). This is why:
- `munmap()` on a shared mapping is slow (must notify all CPUs)
- Thread migration is expensive (destination CPU must flush its TLB)
- KPTI makes every syscall expensive (two CR3 switches per syscall = 2 full TLB flushes)

### 7.4 PCID — Process Context IDs

To avoid flushing the entire TLB on every context switch, modern x86 CPUs support **PCID** (Process Context IDs):

```c
// Each CR3 load can carry a 12-bit PCID (0-4095)
// When context switching: change CR3 to new mm's physical address + PCID
// If NOFLUSH bit is set: don't flush TLB entries with this PCID

// Benefit: switch from process A to B and back:
// Without PCID: full TLB flush twice → process A must re-fill TLB
// With PCID:    A's entries stay in TLB with tag 0
//               B's entries go in with tag 1
//               Switching back to A: no flush needed!

// Linux implementation:
// arch/x86/mm/tlb.c
// Each mm gets a "PCID" assigned from a pool
// Context switch: write new CR3 with mm->pgd + PCID, set NOFLUSH bit
```

---

## 8. Huge Pages in Page Tables

x86_64 supports three page sizes. Larger pages mean fewer TLB entries needed.

### 8.1 4KB (Standard)

Full 4-level walk: PGD → PUD → PMD → PTE → 4KB page.

### 8.2 2MB (PMD-level huge page)

Only 3 levels: PGD → PUD → PMD (with PS bit set) → 2MB page.

```c
// A huge PMD entry:
// bit[7] (PS) = 1 → this is a leaf entry
// bits[51:21] = physical address of 2MB-aligned frame (not a PTE page!)

// Check if PMD is a huge page:
pmd_trans_huge(*pmd)   // pmd & PSE bit

// Create a huge PMD entry:
pmd_t entry = mk_huge_pmd(page, prot);  // Physical 2MB page → huge PMD
set_pmd_at(mm, address, pmd, entry);
```

### 8.3 1GB (PUD-level huge page)

Only 2 levels: PGD → PUD (with PS bit) → 1GB page.

```c
// Used for very large static mappings (kernel direct map uses 1GB pages)
// Rarely used for user-space applications
```

---

## 9. Page Table Memory Overhead

```bash
# Page table memory for a process:
cat /proc/$$/status | grep VmPTE
# VmPTE:     48 kB  ← 12 page table pages = 48KB

# For a process with 1GB of address space:
# L1 PTE pages: 1GB / (512 × 4KB) = 512 pages = 512 × 4KB = 2MB
# L2 PMD pages: 1GB / (512 × 2MB)  = 1 page  = 4KB
# L3 PUD pages: 1 page = 4KB
# Total: ~2MB of page tables for 1GB of mapped space
# But: with 2MB huge pages, L1 PTE pages not needed → just 8KB overhead!

# System-wide page table memory:
cat /proc/meminfo | grep PageTables
# PageTables:     123456 kB  ← Total page table memory for all processes

# Analyze page table overhead for a process:
cat /proc/$$/smaps | grep VmPTE
```

---

## 10. Kernel Page Table vs User Page Table

The kernel address space (upper 128TB on x86_64) is mapped identically in all processes:

```c
// When the kernel creates its page tables at boot (arch/x86/mm/init_64.c):
// kernel_init() → paging_init() → map_direct_map()

// Direct map: map all physical RAM at PAGE_OFFSET
// Uses 1GB PUD entries where possible (for speed)

// init_top_pgt: the initial kernel PGD
// All new process PGDs copy kernel entries from init_top_pgt:
static void pgd_ctor(struct mm_struct *mm, pgd_t *pgd)
{
    // Copy kernel PGD entries to user process:
    memcpy(pgd + KERNEL_PGD_BOUNDARY,
           init_top_pgt + KERNEL_PGD_BOUNDARY,
           KERNEL_PGD_PTRS * sizeof(pgd_t));
}
```

With KPTI: user-mode page table (stored in separate pgd at mm->pgd) has only minimal kernel entries (trampoline for syscall entry). Full kernel mapping is in a separate "kernel pgd" switched on every kernel entry.

---

## 11. 5-Level Page Tables (x86_64 with LA57)

For systems with >256TB RAM or needing >128TB per-process address space, Linux supports 5-level paging:

```bash
cat /boot/config-$(uname -r) | grep CONFIG_X86_5LEVEL
# CONFIG_X86_5LEVEL=y  ← If enabled

# With 5 levels: 57-bit virtual addresses
# Adds P4D (Page 4-level Directory) between PGD and PUD
# User space: 0 - 64PB
# Kernel space: 64PB - 128PB

# Check if 5-level paging is active:
dmesg | grep "5-level paging"
```

---

## 12. Page Table Locks

The page tables are protected by two locking mechanisms:

```c
// 1. PTL (Page Table Lock) — per-PTE-page spinlock:
// Each PTE page has a spinlock embedded in the associated struct page
// Acquired with pte_offset_map_lock(), released with pte_unmap_unlock()

spinlock_t *ptl;
pte_t *pte = pte_offset_map_lock(mm, pmd, address, &ptl);
// ... safely modify PTE ...
pte_unmap_unlock(pte, ptl);

// 2. mmap_lock — protects the VMA tree:
// mmap_read_lock(mm): allows reading VMAs and page tables
// mmap_write_lock(mm): exclusive for VMA tree modification
// Page fault handlers: take mmap_read_lock (shared)
// mmap/munmap/mprotect: take mmap_write_lock (exclusive)
```

---

## 13. Inspecting Page Tables

```bash
# Show page table entries for a process (via /proc/<pid>/pagemap):
python3 << 'EOF'
import struct
pid = __import__('os').getpid()
# /proc/self/pagemap: 8 bytes per page, indexed by virtual page number
# Bit 63: page present
# Bits 54-0: PFN (if present) or swap entry (if not)
with open(f'/proc/{pid}/maps') as f, \
     open(f'/proc/{pid}/pagemap', 'rb') as pm:
    for line in f.readlines()[:5]:
        start, end = [int(x, 16) for x in line.split()[0].split('-')]
        vpn = start >> 12
        pm.seek(vpn * 8)
        entry = struct.unpack('Q', pm.read(8))[0]
        present = bool(entry >> 63)
        pfn = entry & 0x7fffffffffffff
        print(f'{line.split()[0]}: present={present}, pfn={pfn:#x}')
EOF

# Kernel page table dump (requires CONFIG_PTDUMP_DEBUGFS):
cat /sys/kernel/debug/page_tables/kernel   # Kernel page table
cat /sys/kernel/debug/page_tables/current_user  # Current process's page table
```

---

## 14. TLB Performance Optimization

```bash
# perf: measure TLB misses:
sudo perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses \
    ./myprogram

# Typical ratios:
# dTLB-load-misses: < 1% = good, > 5% = TLB pressure, consider huge pages
# iTLB-load-misses: < 0.1% = good

# Reduce TLB pressure:
# 1. Use huge pages (2MB vs 4KB → 512× fewer TLB entries needed)
# 2. Keep working set in contiguous memory
# 3. Use madvise(MADV_HUGEPAGE) on regions > 2MB

# Measure TLB shootdown overhead:
cat /proc/vmstat | grep tlb
# nr_tlb_remote_flush  12345   ← TLB flush IPIs sent to other CPUs
# nr_tlb_remote_flush_received  45678  ← TLBflush IPIs received

# High nr_tlb_remote_flush means lots of munmap/mprotect on shared mappings
```

---

## 15. Mental Model Checkpoint

After Day 27, you should be able to:

1. Draw the four-level page table hierarchy and show how a 48-bit virtual address is decoded.
2. Draw a PTE and identify which bits mean: present, writable, user-accessible, dirty, accessed, XD.
3. What is a TLB and why is TLB miss expensive?
4. Explain a TLB shootdown: when is it needed and what IPIs does it involve?
5. What is PCID and how does it avoid full TLB flushes on context switch?
6. How does a 2MB huge page affect the page table structure? What bit enables it?
7. Why do all processes have the same kernel-space page table entries?
8. What is the PTL (page table lock) and when is it taken vs `mmap_lock`?

---

## Key Source Files

```bash
arch/x86/include/asm/pgtable.h    # PTE format, pte_present, pte_write, etc.
arch/x86/include/asm/pgtable_64.h # 64-bit page table types
mm/memory.c                        # set_pte_at, page table installation
arch/x86/mm/tlb.c                  # TLB flush, TLB shootdown, PCID
arch/x86/mm/pgtable.c             # pgd_alloc, pte_alloc_one
arch/x86/mm/init_64.c             # Kernel page table setup
include/linux/pgtable.h            # Generic page table API
Documentation/mm/page_tables.rst   # Official page table documentation
```

---

## Summary

Page tables are the hardware-visible data structures mapping virtual to physical addresses. On x86_64, four levels (PGD → PUD → PMD → PTE) decode a 48-bit address. Each level is a 4KB page of 512 8-byte entries.

Key design points:
- **Lazy population**: page table entries are created only on page fault, not at mmap time
- **TLB caches translations**: 1-4ns lookup vs 50-100ns page table walk; TLB misses kill performance
- **TLB shootdowns**: changing a shared mapping requires IPIs to all CPUs holding that mm — an expensive but necessary operation
- **Huge pages** (2MB at PMD, 1GB at PUD) reduce TLB pressure 512× or 262144× respectively
- **PCID** avoids full TLB flushes on context switch by tagging entries with process IDs

Tomorrow: page fault handling — the complete path from hardware exception to mapping resolution.
