# Day 26 — Virtual Memory Areas

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand `mm_struct`, `vm_area_struct`, how `mmap()` creates VMAs, and the distinction between anonymous and file-backed mappings.

---

## 1. The Two Levels of Virtual Memory

Virtual memory has two conceptually distinct levels:

**Level 1 — VMAs (Virtual Memory Areas)**: High-level regions of the address space with uniform properties (permissions, backing store). Managed in `mm_struct`. Tells the kernel *what* is mapped *where*.

**Level 2 — Page Tables**: Low-level hardware structures that translate virtual addresses to physical pages. Only populated lazily (on first access). Tells the CPU *which physical page* backs each virtual address.

Today covers Level 1. Day 27 covers Level 2.

---

## 2. `mm_struct` — The Address Space Descriptor

```c
// include/linux/mm_types.h
struct mm_struct {
    // VMA tree: all VMAs in this address space
    struct {
        struct maple_tree mm_mt;   // Maple tree of VMAs (Linux 6.1+)
                                    // Earlier kernels used rb_root + linked list
    };
    
    // Page table root (hardware-level):
    pgd_t *pgd;                   // Page Global Directory (top-level page table)
    
    // Reference counts:
    atomic_t mm_users;            // How many processes use this mm (share it)
                                  // fork() without CLONE_VM: 1
                                  // pthread threads sharing mm: N
    atomic_t mm_count;            // Total references including lazy ones
                                  // kernel threads borrowing mm, etc.
    
    // Locks:
    struct rw_semaphore mmap_lock; // Protects VMA tree and page tables
                                   // Read: page fault handlers, mmap read
                                   // Write: mmap/munmap/mprotect/brk
    
    // Address space size:
    unsigned long task_size;      // Size of user address space
                                  // (= TASK_SIZE = 0x00007FFFFFFFFFFF on x86_64)
    
    // VMA statistics:
    unsigned long map_count;      // Number of VMAs
    
    // Special region addresses:
    unsigned long mmap_base;      // Start of mmap region (grows down from here)
    unsigned long total_vm;       // Total pages mapped (all VMAs)
    unsigned long locked_vm;      // Pages locked with mlock()
    unsigned long pinned_vm;      // Pages pinned by get_user_pages()
    unsigned long data_vm;        // Data segment pages (executable, file-backed)
    unsigned long exec_vm;        // Executable pages
    unsigned long stack_vm;       // Stack pages
    
    // Segment boundaries (populated by execve):
    unsigned long start_code;     // .text start
    unsigned long end_code;       // .text end
    unsigned long start_data;     // .data start
    unsigned long end_data;       // .data end
    unsigned long start_brk;      // Heap start (original)
    unsigned long brk;            // Current heap end (moved by brk())
    unsigned long start_stack;    // Stack start (bottom of stack VMA)
    
    // Arguments and environment:
    unsigned long arg_start;      // argv[] strings start
    unsigned long arg_end;        // argv[] strings end
    unsigned long env_start;      // environ[] strings start
    unsigned long env_end;        // environ[] strings end
    
    // CPU and TLB management:
    mm_cpumask_var_t cpu_vm_mask_var;  // CPUs that have TLB entries for this mm
    unsigned long context.id;          // ASID (Address Space ID) for PCID/ASID
    
    // NUMA:
    struct mempolicy *mempolicy;  // NUMA allocation policy
    
    // Fault tracking (for NUMA balancing):
    unsigned long numa_scan_offset;  // Where to start next NUMA scan
    
    // Executable:
    struct file __rcu *exe_file;  // The executable file (for /proc/<pid>/exe)
    
    // Core dump filtering:
    unsigned long flags;          // MMF_* flags
};
```

```bash
# View mm_struct fields for a process:
cat /proc/$$/status | grep -E 'Vm|threads'
# VmPeak:   123456 kB  ← Peak virtual memory
# VmSize:   100000 kB  ← Current virtual memory (total_vm in pages × 4)
# VmLck:        0 kB  ← locked_vm
# VmPin:        0 kB  ← pinned_vm
# VmHWM:    12345 kB  ← Peak RSS
# VmRSS:    10000 kB  ← Current RSS
# VmData:    8000 kB  ← data_vm
# VmStk:      132 kB  ← stack_vm
# VmExe:      100 kB  ← exec_vm
# VmLib:     3456 kB  ← Shared library pages
# VmPTE:       48 kB  ← Page table entries memory
# VmSwap:       0 kB  ← Pages in swap
```

---

## 3. `vm_area_struct` — One Region of the Address Space

```c
// include/linux/mm_types.h
struct vm_area_struct {
    // Address range of this VMA:
    unsigned long vm_start;     // First byte of this VMA's virtual address
    unsigned long vm_end;       // First byte BEYOND this VMA (exclusive)
    
    // The mm_struct this VMA belongs to:
    struct mm_struct *vm_mm;
    
    // Permissions and flags:
    pgprot_t vm_page_prot;      // Page table protection bits for this VMA
    unsigned long vm_flags;     // VM_READ, VM_WRITE, VM_EXEC, etc.
    
    // For file-backed VMAs:
    struct file *vm_file;       // The backing file (NULL for anonymous)
    unsigned long vm_pgoff;     // Offset into file (in pages)
    
    // For anonymous VMAs (no file):
    struct anon_vma *anon_vma;  // Chain for RMAP (reverse mapping)
    
    // VMA operations (function pointers):
    const struct vm_operations_struct *vm_ops;
    // vm_ops->fault() = called on page fault in this VMA
    // vm_ops->open() = called when VMA is duplicated (fork)
    // vm_ops->close() = called when VMA is freed
    // vm_ops->mmap() = called when device driver's mmap hook fires
    
    void *vm_private_data;      // For vm_ops use (e.g., device driver state)
    
    // For maple tree / rbtree navigation:
    struct {
        struct rb_node rb;      // In mm->mm_rb (older kernels)
        // OR in maple tree (6.1+)
    };
    struct list_head anon_vma_chain; // For RMAP
    
    // For huge pages:
    unsigned int vm_userfaultfd_ctx;
    struct vma_lock *vm_lock;   // Per-VMA lock (newer kernels)
};
```

### 3.1 VMA Flags

```c
// include/linux/mm.h
#define VM_READ         0x00000001  // Pages can be read
#define VM_WRITE        0x00000002  // Pages can be written
#define VM_EXEC         0x00000004  // Pages can be executed
#define VM_SHARED       0x00000008  // Pages are shared (MAP_SHARED)
                                    // Without this: MAP_PRIVATE (COW)

#define VM_MAYREAD      0x00000010  // read() is allowed (even if not mapped readable)
#define VM_MAYWRITE     0x00000020
#define VM_MAYEXEC      0x00000040
#define VM_MAYSHARE     0x00000080

#define VM_GROWSDOWN    0x00000100  // VMA grows downward (stack)
#define VM_GROWSUP      0x00000200  // VMA grows upward
#define VM_PFNMAP       0x00000400  // VMA maps physical pages (device driver)
#define VM_DENYWRITE    0x00000800  // File cannot be written while mapped exec
#define VM_LOCKED       0x00002000  // Pages locked in memory (mlock)
#define VM_IO           0x00004000  // Memory-mapped I/O (device registers)
#define VM_SEQ_READ     0x00008000  // Sequential access hint
#define VM_RAND_READ    0x00010000  // Random access hint
#define VM_DONTCOPY     0x00020000  // Don't copy VMA on fork
#define VM_DONTEXPAND   0x00040000  // Cannot expand with mremap
#define VM_ACCOUNT      0x00100000  // Account this VMA to rlimit
#define VM_NORESERVE    0x00200000  // Don't reserve swap space (overcommit)
#define VM_HUGETLB      0x00400000  // Huge TLB page VMA
#define VM_SYNC         0x00800000  // Writes to this VMA are synchronous
#define VM_ARCH_1       0x01000000  // Architecture-specific (PROT_BTI on ARM64)
#define VM_WIPEONFORK   0x02000000  // Clear pages on fork (for secrets)
#define VM_DONTDUMP     0x04000000  // Don't include in core dump
#define VM_SOFTDIRTY    0x08000000  // Soft-dirty tracking for checkpoint/restore
#define VM_MIXEDMAP     0x10000000  // Mixed pfnmap + regular pages
#define VM_HUGEPAGE     0x20000000  // THP hints enabled
#define VM_NOHUGEPAGE   0x40000000  // THP disabled for this VMA
#define VM_MERGEABLE    0x80000000  // KSM-eligible (Kernel Samepage Merging)
```

---

## 4. Reading VMAs: `/proc/<pid>/maps` and `smaps`

```bash
# Basic VMA listing:
cat /proc/$$/maps
# addr-range      perms  offset   dev   inode   pathname
# 55b8a3456000-55b8a3470000 r--p 00000000 fd:01 1234567 /usr/bin/bash
# 55b8a3470000-55b8a34f0000 r-xp 0001a000 fd:01 1234567 /usr/bin/bash
# 55b8a34f0000-55b8a3520000 r--p 0009a000 fd:01 1234567 /usr/bin/bash
# 55b8a3520000-55b8a3524000 r--p 000c9000 fd:01 1234567 /usr/bin/bash ← .data
# 55b8a3524000-55b8a352d000 rw-p 000cd000 fd:01 1234567 /usr/bin/bash ← .bss
# 55b8a4000000-55b8a4200000 rw-p 00000000 00:00 0                     [heap]
# 7f1234000000-7f1234021000 rw-p 00000000 00:00 0                     ← anon mmap
# 7f1234021000-7f1235000000 ---p 00000000 00:00 0                     ← guard
# 7f1235000000-7f1235200000 r--p 00000000 fd:01 7654321 /lib/x86_64-linux-gnu/libc.so.6
# 7f1235200000-7f1236800000 r-xp 00200000 fd:01 7654321 /lib/.../libc.so.6
# 7ffd12345000-7ffd12366000 rw-p 00000000 00:00 0                     [stack]
# 7ffd123ff000-7ffd12400000 r--p 00000000 00:00 0                     [vvar]
# 7ffd12400000-7ffd12401000 r-xp 00000000 00:00 0                     [vdso]

# Detailed per-VMA statistics:
cat /proc/$$/smaps | head -50
# 55b8a3456000-55b8a3470000 r--p 00000000 fd:01 1234567 /usr/bin/bash
# Size:                 88 kB   ← VMA size
# KernelPageSize:        4 kB
# MMUPageSize:           4 kB
# Rss:                  88 kB   ← Currently in RAM
# Pss:                  44 kB   ← Proportional share (shared pages / sharers)
# Shared_Clean:         88 kB   ← Read-only shared pages (clean)
# Shared_Dirty:          0 kB
# Private_Clean:         0 kB
# Private_Dirty:         0 kB   ← Pages modified in this process (COW'd)
# Referenced:           88 kB
# Anonymous:             0 kB   ← Anonymous pages in this VMA
# LazyFree:              0 kB
# AnonHugePages:         0 kB
# ShmemPmdMapped:        0 kB
# FilePmdMapped:         0 kB
# Shared_Hugetlb:        0 kB
# Private_Hugetlb:       0 kB
# Swap:                  0 kB
# SwapPss:               0 kB
# Locked:                0 kB
# THPeligible:           0
# VmFlags: rd mr mw me
```

---

## 5. `mmap()` System Call Internals

```c
// User space:
void *addr = mmap(NULL,        // Let kernel choose address
                  length,      // Bytes to map
                  PROT_READ | PROT_WRITE,  // Protection
                  MAP_PRIVATE | MAP_ANONYMOUS,  // Flags
                  -1,          // fd = -1 for anonymous
                  0);          // offset (ignored for anonymous)
```

```c
// Kernel implementation: mm/mmap.c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, unsigned long, off)
{
    return ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
}

unsigned long ksys_mmap_pgoff(unsigned long addr, unsigned long len,
                              unsigned long prot, unsigned long flags,
                              unsigned long fd, unsigned long pgoff)
{
    // Validate arguments, convert fd to struct file if file-backed
    
    return vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
}

unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
                            unsigned long len, unsigned long prot,
                            unsigned long flag, unsigned long pgoff)
{
    unsigned long ret;
    
    // Acquire mmap_lock for writing (exclusive VMA modification):
    mmap_write_lock(mm);
    
    ret = do_mmap(file, addr, len, prot, flag, 0, pgoff, &populate);
    
    mmap_write_unlock(mm);
    
    // Pre-fault pages if MAP_POPULATE or MAP_LOCKED:
    if (populate)
        mm_populate(ret, populate);
    
    return ret;
}
```

### 5.1 `do_mmap()` — Creating the VMA

```c
// mm/mmap.c
unsigned long do_mmap(struct file *file, unsigned long addr,
                      unsigned long len, unsigned long prot,
                      unsigned long flags, vm_flags_t vm_flags,
                      unsigned long pgoff, unsigned long *populate,
                      struct list_head *uf)
{
    struct mm_struct *mm = current->mm;
    
    // 1. Validate arguments:
    if (!len) return -EINVAL;
    len = PAGE_ALIGN(len);
    
    // 2. Security check (mmap is a major security boundary):
    security_mmap_addr(addr);
    
    // 3. Find address range to map into:
    addr = get_unmapped_area(file, addr, len, pgoff, flags);
    // get_unmapped_area finds a gap in the VMA tree of size >= len
    // respects hints, alignment requirements, ASLR
    
    // 4. Compute vm_flags:
    vm_flags |= calc_vm_prot_bits(prot, 0) | calc_vm_flag_bits(flags);
    
    // 5. Call mmap_region() to create the VMA:
    addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);
    
    return addr;
}

unsigned long mmap_region(struct file *file, unsigned long addr,
                          unsigned long len, vm_flags_t vm_flags,
                          unsigned long pgoff, struct list_head *uf)
{
    // 1. Try to merge with adjacent VMAs (if compatible):
    vma = vma_merge(mm, prev, addr, addr+len, vm_flags, ...);
    if (vma) goto out;
    
    // 2. Allocate new VMA:
    vma = vm_area_alloc(mm);  // From vm_area_struct slab cache
    
    // 3. Initialize VMA fields:
    vma->vm_start = addr;
    vma->vm_end   = addr + len;
    vma->vm_flags = vm_flags;
    vma->vm_pgoff = pgoff;
    vma->vm_page_prot = vm_get_page_prot(vm_flags);
    
    if (file) {
        // File-backed mapping:
        vma->vm_file = get_file(file);  // Take reference to file
        // Call the file's mmap handler (sets vm_ops):
        file->f_op->mmap(file, vma);
        // For regular files: generic_file_mmap() sets vm_ops = &generic_file_vm_ops
        // For device files: driver's mmap() sets vm_ops for device-specific faults
    } else {
        // Anonymous mapping:
        vma->vm_ops = NULL;
        // Anonymous VMAs get an anon_vma only on first page fault
    }
    
    // 4. Insert into mm's VMA tree:
    vma_mas_store(vma, &mas);  // Maple tree insertion (Linux 6.1+)
    // OR: vma_link(mm, vma, ...) on older kernels
    
    mm->map_count++;
    
out:
    return addr;
}
```

---

## 6. Anonymous vs File-Backed Mappings

### 6.1 Anonymous Mappings

No backing file — data lives only in RAM (and swap when swapped out).

```c
// Created by:
mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
// OR:
brk(new_end)   // Extending the heap

// Properties:
// vm_file = NULL
// No page cache backing
// On page fault: alloc_page() + zero fill
// On fork: COW (copy-on-write)
// On swap: goes to swap partition/file

// Anonymous VMA fields:
vma->vm_file = NULL;
vma->anon_vma = NULL;  // Set on first fault: kmem_cache_alloc(anon_vma_cachep)
```

### 6.2 File-Backed Mappings

Data backed by a file — reads come from the page cache, writes go back to the file.

```c
// Created by:
int fd = open("/path/to/file", O_RDONLY);
void *addr = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
// MAP_PRIVATE: COW — writes create private copies (not written back to file)
// MAP_SHARED: changes written back to file, shared with other mappers

// Properties:
// vm_file = pointer to struct file
// vm_pgoff = offset in file (in pages)
// Pages come from page cache (address_space of the file)
// MAP_SHARED writes modify the page cache (dirty pages are written back)
// MAP_PRIVATE writes create COW copies (anonymous pages shadow the file)
```

### 6.3 The Mapping Decision

```
Page fault in VMA:
  if (vma->vm_ops && vma->vm_ops->fault):
      file-backed fault → call vm_ops->fault()
      → filemap_fault() → find/read page from page cache
  else:
      anonymous fault → do_anonymous_page()
      → alloc_page_vma() → zero-fill → install in page table
```

---

## 7. VMA Merging

When two adjacent VMAs have the same permissions, flags, and backing, the kernel merges them:

```c
// mm/mmap.c
static struct vm_area_struct *vma_merge(struct mm_struct *mm,
                                         struct vm_area_struct *prev, ...)
{
    // Check: can we extend prev to cover new range?
    // Conditions:
    // 1. prev->vm_end == addr (adjacent)
    // 2. Same vm_flags
    // 3. Same vm_file and consecutive vm_pgoff
    // 4. Same anon_vma
    // 5. vm_ops allows merging
    
    // Also check if new range can be merged with 'next' VMA
    
    // Merging reduces mm->map_count and saves vm_area_struct allocations
}
```

```bash
# See that multiple mmap calls get merged:
python3 << 'EOF'
import mmap, os

# Map same file 3 times consecutively:
f = open('/proc/self/exe', 'rb')
m1 = mmap.mmap(f.fileno(), 4096, offset=0)      # offset 0
m2 = mmap.mmap(f.fileno(), 4096, offset=0x1000)  # offset 1 page
m3 = mmap.mmap(f.fileno(), 4096, offset=0x2000)  # offset 2 pages

# If addresses are consecutive, they get merged into one VMA
maps = open('/proc/self/maps').readlines()
for m in maps:
    if '/proc/self/exe' in m or 'exe' in m:
        print(m.strip())
EOF
```

---

## 8. `munmap()` — Removing VMAs

```c
// mm/mmap.c
SYSCALL_DEFINE2(munmap, unsigned long, addr, size_t, len)
{
    return __vm_munmap(addr, len, true);
}

int __vm_munmap(unsigned long start, size_t len, bool downgrade)
{
    struct mm_struct *mm = current->mm;
    
    mmap_write_lock(mm);
    
    // Find all VMAs overlapping [start, start+len):
    // For each overlapping VMA:
    //   1. If VMA entirely within range: remove it
    //   2. If VMA partially overlaps: split it, remove the overlapping part
    
    do_mas_munmap(&mas, mm, start, start + len, &uf, downgrade);
    
    mmap_write_unlock(mm);
    
    // Unmap from TLB (flush TLB entries for this range):
    // done inside do_mas_munmap via unmap_region → tlb_finish_mmu
    
    return 0;
}
```

---

## 9. `mprotect()` — Changing VMA Protection

```c
// Change permissions on a mapped region:
mprotect(addr, len, PROT_READ);         // Make read-only
mprotect(addr, len, PROT_READ | PROT_WRITE);  // Make writable

// Kernel:
SYSCALL_DEFINE3(mprotect, unsigned long, start, size_t, len, unsigned long, prot)
{
    // 1. Find VMA(s) overlapping [start, start+len)
    // 2. For each VMA: update vm_flags and vm_page_prot
    // 3. If VMA partially overlaps: split the VMA
    // 4. Update page table entries for affected pages
    // 5. Flush TLB for affected range
}
```

Lowering permissions (write → read-only) is fast: just update PTEs and flush TLB.

Raising permissions (read-only → execute) requires security checks (W^X enforcement: can't have both writable and executable).

---

## 10. `brk()` — Heap Expansion

```c
// Growing or shrinking the heap:
void *sbrk(intptr_t increment);    // libc wrapper
// OR directly:
void *new_brk = brk(current_brk + size);

// Kernel:
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
    struct mm_struct *mm = current->mm;
    unsigned long min_brk = mm->start_brk;  // Can't go below original heap start
    unsigned long old_brk = mm->brk;
    
    if (brk < min_brk) return old_brk;  // Can't shrink below start
    
    if (brk > old_brk) {
        // Growing: extend the heap VMA or create a new one
        // Checks: rlimit(RLIMIT_DATA), vm_committed_pages
        do_brk_flags(old_brk, brk - old_brk, 0, ...);
        mm->brk = brk;
    } else if (brk < old_brk) {
        // Shrinking: munmap the excess
        do_mas_munmap(&mas, mm, brk, old_brk, ...);
        mm->brk = brk;
    }
    
    return mm->brk;
}
```

---

## 11. `MAP_FIXED`, ASLR, and Address Hints

```c
// Address space layout randomization (ASLR):
// The kernel randomizes where regions land:
cat /proc/sys/kernel/randomize_va_space
# 0 = disabled
# 1 = randomize mmap, stack, vdso
# 2 = also randomize heap (brk)

// mmap hint address:
// mmap(hint, ...) → kernel tries hint, may use different address
// mmap(NULL, ...) → kernel chooses freely (ASLR applies)
// mmap(addr, ..., MAP_FIXED, ...) → MUST use exactly addr (dangerous!)
// mmap(addr, ..., MAP_FIXED_NOREPLACE, ...) → use addr or fail with EEXIST

// ASLR implementation in get_unmapped_area():
// Adds random offset to mmap_base each time
// mmap_base = randomize(TASK_MMAP_BASE)  on exec
```

---

## 12. Device Memory Mapping (`MAP_FIXED` + `vm_ops`)

Device drivers implement `file->f_op->mmap()` to allow user space to map device registers or memory:

```c
// In a driver:
static int my_device_mmap(struct file *file, struct vm_area_struct *vma)
{
    unsigned long phys_addr = MY_DEVICE_MEM_BASE;
    unsigned long size = vma->vm_end - vma->vm_start;
    
    // Mark VMA as I/O memory:
    vma->vm_flags |= VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP;
    vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
    
    // Map physical pages directly into user page table:
    if (remap_pfn_range(vma,
                        vma->vm_start,          // Virtual address
                        phys_addr >> PAGE_SHIFT, // PFN of device memory
                        size,
                        vma->vm_page_prot))
        return -EAGAIN;
    
    return 0;
}
```

---

## 13. `copy_vma()` on Fork

When `fork()` is called without `CLONE_VM`, all VMAs are copied:

```c
// kernel/fork.c → dup_mm() → dup_mmap()
static int dup_mmap(struct mm_struct *mm, struct mm_struct *oldmm)
{
    struct vm_area_struct *mpnt, *tmp;
    
    // For each VMA in the parent:
    for_each_vma(old_vmas_iter, mpnt) {
        // Skip VMAs marked with VM_DONTCOPY:
        if (mpnt->vm_flags & VM_DONTCOPY)
            continue;
        
        // Allocate new VMA descriptor:
        tmp = vm_area_dup(mpnt);
        
        // File-backed: take reference to file:
        if (tmp->vm_file)
            get_file(tmp->vm_file);
        
        // Call vm_ops->open if defined:
        if (tmp->vm_ops && tmp->vm_ops->open)
            tmp->vm_ops->open(tmp);
        
        // Insert into child mm's VMA tree:
        vma_mas_store(tmp, &mas);
        mm->map_count++;
    }
    
    // Copy page tables (COW protection):
    copy_page_range(mm, oldmm, mpnt);
    // This marks all writable pages as read-only in both parent and child
    // (COW setup — actual page copying deferred to write faults)
    
    return 0;
}
```

---

## 14. The Maple Tree (Linux 6.1+)

The VMA lookup structure was changed from a red-black tree + linked list to a **Maple Tree** in Linux 6.1:

```c
// Maple tree: a B-tree variant optimized for VMA ranges
// Key property: can store intervals (range → VMA) natively
// vs red-black tree: stored VMAs by start address, needed list for ordering

// Benefits of Maple Tree:
// - O(log n) range queries (find VMA at address)
// - RCU-safe reads (lock-free page fault VMA lookup!)
// - Better cache locality (B-tree internal nodes hold multiple keys)
// - Simpler code than rb_tree + list combo

// Page fault VMA lookup (hot path):
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
    // Maple tree range query: find VMA containing addr
    return mt_find(&mm->mm_mt, &addr, ULONG_MAX);
    // O(log n), RCU-safe (no mmap_lock needed for read!)
}
```

---

## 15. VMA Statistics and Limits

```bash
# Number of VMAs per process:
cat /proc/$$/status | grep VmPTE
# VmPTE:     48 kB  ← Memory used for page tables (proportional to VMA count × pages)

cat /proc/$$/maps | wc -l  # Count VMAs

# System-wide VMA limit (protects against /proc/sys/vm/max_map_count exhaustion):
cat /proc/sys/vm/max_map_count
# 65530  ← Default; some apps (Java, Android) need more

# Increase for applications with many mmap calls:
echo 262144 > /proc/sys/vm/max_map_count

# ElasticSearch / Java typically needs this:
sysctl vm.max_map_count=262144
```

---

## 16. Mental Model Checkpoint

After Day 26, you should be able to:

1. Explain the relationship between `mm_struct`, `vm_area_struct`, and page tables.
2. What fields distinguish an anonymous VMA from a file-backed VMA?
3. Trace `mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` through `do_mmap()`.
4. What is `mmap_lock` and when is it taken in write mode vs read mode?
5. Explain VMA merging — when two adjacent VMAs become one.
6. What does `MAP_PRIVATE` mean for a file-backed mapping? What happens when you write to it?
7. What is ASLR and how does it work with `get_unmapped_area()`?
8. Why did Linux 6.1 switch from red-black tree to Maple Tree for VMAs?

---

## Key Source Files

```bash
include/linux/mm_types.h    # struct mm_struct, vm_area_struct
mm/mmap.c                   # do_mmap(), mmap_region(), munmap(), brk()
mm/vma.c                    # VMA utilities (Linux 6.1+)
mm/internal.h               # Internal MM functions
mm/vmacache.c               # VMA cache for fast lookups
include/linux/mm.h          # VM_* flags, mm_struct helpers
arch/x86/mm/mmap.c          # x86 address space layout, ASLR
lib/maple_tree.c            # Maple tree implementation
Documentation/mm/vma.rst    # Official VMA documentation
```

---

## Summary

Virtual memory areas (VMAs) are the kernel's high-level view of a process's address space. Each VMA represents a contiguous region with uniform properties: same permissions, same backing (anonymous or file), same flags.

Key operations:
- **`mmap()`** creates VMAs via `do_mmap()` → `mmap_region()`
- **`munmap()`** removes VMAs, updating the maple tree and flushing TLBs
- **`mprotect()`** changes VMA permissions and updates page table entries
- **`brk()`** adjusts the heap VMA (anonymous mapping)
- **Fork** duplicates all VMAs and sets up COW protection in page tables

VMAs don't directly contain page table entries — they describe *what should be mapped* and page faults populate the actual page tables lazily.

Tomorrow: page tables — the hardware structures that implement virtual-to-physical translation.
