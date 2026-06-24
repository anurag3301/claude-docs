# Day 28 — Page Fault Handling

> **Estimated read time:** 90–120 minutes  
> **Goal:** Trace the complete page fault path from CPU exception through `handle_mm_fault()` — demand paging, COW faults, protection faults, and kernel fixups.

---

## 1. Types of Page Faults

A page fault (`#PF` exception, vector 14 on x86) fires whenever the CPU cannot complete a memory access using the current page tables. There are three broad categories:

**1. Valid fault (resolvable)**: The VMA exists, the access is permitted — just the page isn't in memory yet. The kernel resolves by loading the page.
- Demand paging: anonymous page not yet allocated
- File page not yet loaded from disk
- COW: write to a shared read-only page

**2. Protection fault (access violation)**: The VMA exists but the access violates permissions (e.g., writing to a read-only page).
- Could be a real bug (SIGSEGV) or a COW signal to copy

**3. Invalid fault (segfault)**: No VMA covers the faulting address — a true bug.
- Stack overflow
- Null pointer dereference
- Use-after-free

---

## 2. Hardware Fault Delivery (x86)

When the CPU encounters a bad address:

1. Saves `RIP` (faulting instruction), `CS`, `RFLAGS`, `RSP`, `SS` on the kernel stack
2. Pushes the **error code** (32-bit value encoding fault type)
3. Stores the **faulting virtual address** in `CR2`
4. Jumps to IDT vector 14 handler (`asm_exc_page_fault`)

```c
// Error code bits (x86_64):
// Bit 0 (P):  0 = not-present fault, 1 = protection fault
// Bit 1 (W):  0 = read fault, 1 = write fault
// Bit 2 (U):  0 = kernel fault, 1 = user-mode fault
// Bit 3 (R):  0 = normal fault, 1 = reserved bit violation
// Bit 4 (I):  0 = data fault, 1 = instruction fetch fault
// Bit 5 (PK): 0 = normal, 1 = protection key violation (MPK)
// Bit 6 (SS): 0 = normal, 1 = shadow stack access fault
// Bit 15 (SGX): 0 = normal, 1 = SGX-related fault

// These bits tell the kernel:
// P=0: page wasn't present → demand paging, swap-in
// P=1, W=1: wrote to read-only page → maybe COW, maybe SIGSEGV
// P=1, U=0: kernel wrote to user page with SMAP on → kernel bug
// I=1: tried to execute non-executable page → SIGSEGV (NX violation)
```

---

## 3. The Fault Handler Entry Chain

```
CPU: #PF → IDT[14] → asm_exc_page_fault
    │
    ▼  arch/x86/mm/fault.c
exc_page_fault(regs, error_code)
    │
    ├─ Is this a kernel-mode fault?
    │   YES → handle_kernel_page_fault() → check exception table / oops
    │
    └─ Is this a user-mode fault?
        YES → handle_user_page_fault()
                │
                ▼
          fault_address = read_cr2()  ← The faulting virtual address
                │
                ▼
          do_user_addr_fault(regs, error_code, fault_address)
```

---

## 4. `do_user_addr_fault()` — Main Fault Handler

```c
// arch/x86/mm/fault.c
static void __do_page_fault(struct pt_regs *regs, unsigned long error_code,
                            unsigned long address)
{
    struct vm_area_struct *vma;
    struct task_struct *tsk = current;
    struct mm_struct *mm = tsk->mm;
    vm_fault_t fault;
    
    // ─── Acquire mmap_lock (read mode) ─────────────────────────────
    // Can't use write lock here: would deadlock if fault occurs inside
    // mmap_write_lock() itself
    mmap_read_lock(mm);
    
    // ─── Find the VMA containing the faulting address ───────────────
    vma = find_vma(mm, address);
    
    if (unlikely(!vma)) {
        // No VMA at this address → definitely SIGSEGV
        goto bad_area;
    }
    
    if (likely(vma->vm_start <= address)) {
        // address is within vma → valid VMA
        goto good_area;
    }
    
    if (!(vma->vm_flags & VM_GROWSDOWN)) {
        // Address is before this VMA and VMA doesn't grow down
        goto bad_area;
    }
    
    // VMA grows down (stack): try to expand it
    if (expand_stack(vma, address)) {
        goto bad_area;  // Stack expansion failed (rlimit or OOM)
    }
    
good_area:
    // ─── Check permissions ─────────────────────────────────────────
    if (error_code & X86_PF_WRITE) {
        // Write fault
        if (!(vma->vm_flags & VM_WRITE))
            goto bad_area;  // VMA is not writable → SIGSEGV
    }
    if (error_code & X86_PF_INSTR) {
        // Instruction fetch fault (NX bit)
        if (!(vma->vm_flags & VM_EXEC))
            goto bad_area;  // VMA is not executable → SIGSEGV
        if (boot_cpu_has(X86_FEATURE_SMEP) && error_code & X86_PF_USER)
            goto bad_area;  // SMEP violation: kernel tried to exec user page
    }
    
    // ─── Handle the fault ──────────────────────────────────────────
    fault = handle_mm_fault(vma, address, fault_flags, regs);
    
    // handle_mm_fault() returns vm_fault_t:
    // VM_FAULT_MINOR: resolved without I/O (page was in page cache)
    // VM_FAULT_MAJOR: resolved with I/O (had to read from disk)
    // VM_FAULT_OOM:  out of memory
    // VM_FAULT_SIGBUS: bad hardware address
    // VM_FAULT_SIGSEGV: permissions issue (shouldn't happen here)
    
    mmap_read_unlock(mm);
    
    if (unlikely(fault & VM_FAULT_ERROR)) {
        // Fatal fault: send signal to process
        mm_fault_error(regs, error_code, address, fault);
    }
    return;
    
bad_area:
    mmap_read_unlock(mm);
    // Send SIGSEGV to process
    force_sig_fault(SIGSEGV, SEGV_MAPERR, (void __user *)address);
}
```

---

## 5. `handle_mm_fault()` — The Resolution Engine

```c
// mm/memory.c
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
                           unsigned int flags, struct pt_regs *regs)
{
    vm_fault_t ret;
    
    // Build vmf (vm_fault) structure — passed through all fault handlers:
    struct vm_fault vmf = {
        .vma        = vma,
        .address    = address & PAGE_MASK,
        .real_address = address,
        .flags      = flags,
        .pgoff      = linear_page_index(vma, address),
        .gfp_mask   = __get_fault_gfp_mask(vma),
    };
    
    // Handle huge pages (THP or hugetlb) separately:
    if (unlikely(is_vm_hugetlb_page(vma)))
        return hugetlb_fault(vma->vm_mm, vma, address, flags);
    
    // For regular (4KB) pages:
    return __handle_mm_fault(vma, &vmf);
}

static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
                                     struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    pgd_t *pgd;
    p4d_t *p4d;
    pud_t *pud;
    pmd_t *pmd;
    
    // Walk to the correct position in the page table,
    // allocating intermediate levels as needed:
    pgd = pgd_offset(vma->vm_mm, vmf->address);
    p4d = p4d_alloc(vma->vm_mm, pgd, vmf->address);
    pud = pud_alloc(vma->vm_mm, p4d, vmf->address);
    
    // Try PMD-level huge page first (THP):
    vmf->pmd = pmd_alloc(vma->vm_mm, pud, vmf->address);
    if (pmd_trans_unstable(vmf->pmd))
        return 0;  // PMD being split — retry
    
    // THP: if 2MB region and aligned, might upgrade to THP:
    if (pmd_none(*vmf->pmd) && hugepage_vma_check(vma, vma->vm_flags, ...)) {
        ret = create_huge_pmd(vmf);
        if (ret != VM_FAULT_FALLBACK)
            return ret;
    }
    
    // Regular PTE-level fault:
    return handle_pte_fault(vmf);
}
```

---

## 6. `handle_pte_fault()` — The Decision Tree

```c
// mm/memory.c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry;
    
    // Check if PTE page itself exists:
    if (!vmf->pmd || pmd_none(*vmf->pmd)) {
        vmf->pte = NULL;
        vmf->ptl = NULL;
    } else {
        // Lock and read PTE:
        vmf->pte = pte_offset_map_lock(vmf->vma->vm_mm, vmf->pmd,
                                        vmf->address, &vmf->ptl);
        entry = *vmf->pte;
    }
    
    if (!vmf->pte) {
        // ─── Case 1: No PTE yet ──────────────────────────────────────
        if (vma_is_anonymous(vmf->vma))
            return do_anonymous_page(vmf);     // Anonymous demand paging
        else
            return do_fault(vmf);              // File-backed demand paging
    }
    
    if (!pte_present(entry)) {
        // ─── Case 2: PTE exists but page not present ─────────────────
        if (pte_none(entry)) {
            // Completely empty PTE (should not happen if vmf->pte != NULL)
            return do_anonymous_page(vmf);
        }
        if (pte_file(entry))
            return do_nonlinear_fault(vmf, entry);
        // Otherwise: swapped out page:
        return do_swap_page(vmf);              // Swap-in from disk
    }
    
    if (pte_protnone(entry) && vma_is_accessible(vmf->vma))
        return do_numa_page(vmf);              // NUMA page migration hint
    
    // ─── Case 3: PTE present but protection fault ────────────────────
    // (P=1 in error_code: page IS present but permissions wrong)
    ptl = pte_lockptr(vmf->vma->vm_mm, vmf->pmd);
    spin_lock(ptl);
    
    if (vmf->flags & FAULT_FLAG_WRITE) {
        if (!pte_write(entry))
            return do_wp_page(vmf);            // COW write-protect fault
        entry = pte_mkdirty(entry);
    }
    
    entry = pte_mkyoung(entry);  // Mark page as recently accessed
    set_pte_at(vmf->vma->vm_mm, vmf->address, vmf->pte, entry);
    
    spin_unlock(ptl);
    return 0;
}
```

---

## 7. Demand Paging: Anonymous Pages

```c
// mm/memory.c
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct page *page;
    pte_t entry;
    
    // Zero page optimization for reads:
    // When first reading an anonymous page, map the zero page
    // (a single shared physical page of zeros) instead of allocating
    if (!(vmf->flags & FAULT_FLAG_WRITE)) {
        // Map the zero page read-only:
        // Multiple processes share this single physical page of zeros
        entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),
                                       vma->vm_page_prot));
        vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd,
                                        vmf->address, &vmf->ptl);
        if (!pte_none(*vmf->pte))
            goto unlock;
        set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
        return 0;
    }
    
    // Write access: allocate a real zero page:
    if (unlikely(anon_vma_prepare(vma)))
        return VM_FAULT_OOM;
    
    page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
    if (!page)
        return VM_FAULT_OOM;
    
    // Build PTE:
    entry = mk_pte(page, vma->vm_page_prot);
    entry = pte_sw_mkyoung(entry);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry));
    
    // Install PTE:
    vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd,
                                    vmf->address, &vmf->ptl);
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
    
    // Set up reverse mapping (for page reclaim):
    page_add_new_anon_rmap(page, vma, vmf->address);
    lru_cache_add_inactive_or_unevictable(page, vma);
    
    pte_unmap_unlock(vmf->pte, vmf->ptl);
    return 0;
}
```

---

## 8. Demand Paging: File-Backed Pages

```c
// mm/filemap.c
vm_fault_t filemap_fault(struct vm_fault *vmf)
{
    struct file *file = vmf->vma->vm_file;
    struct address_space *mapping = file->f_mapping;
    struct inode *inode = mapping->host;
    pgoff_t offset = vmf->pgoff;
    struct page *page;
    vm_fault_t ret = 0;
    
    // 1. Look up page in the page cache:
    page = find_get_page(mapping, offset);
    
    if (!page) {
        // 2. Page not in cache — read from disk:
        // Readahead: pre-read surrounding pages too
        page_cache_sync_readahead(mapping, &file->f_ra, file, offset, ...);
        
        // Try again (readahead may have populated it):
        page = find_get_page(mapping, offset);
        if (unlikely(!page)) {
            // Still not there: allocate and read:
            page = __page_cache_alloc(mapping_gfp_mask(mapping));
            err = add_to_page_cache_lru(page, mapping, offset, GFP_KERNEL);
            
            // Read the page from disk:
            error = mapping->a_ops->readpage(file, page);
            // readpage() submits I/O and may block (major fault)
            ret |= VM_FAULT_MAJOR;
        }
    }
    
    // 3. Wait for I/O to complete:
    if (PageLocked(page))
        wait_on_page_locked(page);
    
    // 4. Install PTE:
    vmf->page = page;
    return ret | VM_FAULT_LOCKED;
}
```

---

## 9. COW Fault: `do_wp_page()`

When a process writes to a read-only page (protection fault with P=1):

```c
// mm/memory.c
static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct page *old_page = vmf->page;
    
    // Check: is this the only mapping of this page?
    if (PageAnon(old_page)) {
        // Anonymous page:
        if (page_mapcount(old_page) == 1 && !PageAnonExclusive(old_page)) {
            // We're the only one mapping it:
            // Just make the PTE writable (no copy needed!)
            wp_page_reuse(vmf);
            return 0;
        }
        // Multiple mappings (shared COW):
        return wp_page_copy(vmf);
    }
    
    if (!PageKsm(old_page) && page_count(old_page) == 1) {
        // Only one process has this page: just make writable
        wp_page_reuse(vmf);
        return 0;
    }
    
    // Must copy:
    return wp_page_copy(vmf);
}

static vm_fault_t wp_page_copy(struct vm_fault *vmf)
{
    struct page *old_page = vmf->page;
    struct page *new_page;
    
    // Allocate a new page:
    new_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vma, vmf->address);
    if (!new_page)
        return VM_FAULT_OOM;
    
    // Copy the old page's contents:
    copy_user_highpage(new_page, old_page, vmf->address, vma);
    // copy_user_highpage: uses optimized memcpy, handles highmem
    
    // Install the new page in the page table (writable):
    pte_t entry = mk_pte(new_page, vma->vm_page_prot);
    entry = pte_mkwrite(pte_mkdirty(entry));
    
    pte_unmap_unlock(vmf->pte, vmf->ptl);
    vmf->ptl = NULL;
    
    vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd,
                                    vmf->address, &vmf->ptl);
    if (likely(pte_same(*vmf->pte, vmf->orig_pte))) {
        set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
        
        // Update reverse mappings:
        page_remove_rmap(old_page, vma, false);
        page_add_new_anon_rmap(new_page, vma, vmf->address);
        
        put_page(old_page);  // Release old page reference
    }
    
    pte_unmap_unlock(vmf->pte, vmf->ptl);
    return 0;
}
```

---

## 10. Swap Fault: `do_swap_page()`

When a page has been swapped to disk (PTE present=0, non-null):

```c
// mm/memory.c
vm_fault_t do_swap_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    swp_entry_t entry;
    struct page *page;
    
    // The PTE contains a "swap entry" (not a physical address):
    // Bits encode: swap type (which swap device) + swap offset
    entry = pte_to_swp_entry(vmf->orig_pte);
    
    // Look in swap cache first (maybe it's still in memory):
    page = lookup_swap_cache(entry, vma, vmf->address);
    if (!page) {
        // Not in cache: read from swap device:
        page = swapin_readahead(entry, GFP_HIGHUSER_MOVABLE,
                                 vmf);  // Reads from disk (MAJOR fault)
        if (!page)
            return VM_FAULT_OOM;
    }
    
    // Wait for I/O to complete:
    wait_on_page_locked(page);
    
    // Check if page is still valid (wasn't reused):
    if (PageError(page) || !PageUptodate(page)) {
        put_page(page);
        return VM_FAULT_SIGBUS;
    }
    
    // Delete swap entry (page is being brought back to memory):
    delete_from_swap_cache(page);
    
    // Install new PTE:
    pte_t pte = mk_pte(page, vma->vm_page_prot);
    if ((vmf->flags & FAULT_FLAG_WRITE) && reuse_swap_page(page, NULL))
        pte = pte_mkwrite(pte_mkdirty(pte));
    
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, pte);
    
    // Set up reverse mapping:
    if (!PageAnonExclusive(page))
        page_add_anon_rmap(page, vma, vmf->address, false);
    
    return VM_FAULT_MAJOR;  // Had to do I/O
}
```

---

## 11. Kernel Fault Handling: The Exception Table

When the kernel itself causes a page fault (e.g., in `copy_from_user()`), it's not a bug — it's expected. The kernel uses an **exception table** (fixup table) to handle this gracefully:

```c
// arch/x86/lib/copy_user_64.S
// The copy_from_user implementation has exception table entries:
ENTRY(copy_user_generic_unrolled)
    // ...
1:  movq (%rsi), %r8       // Read from user pointer
    // Exception table entry: if fault at address 1:, jump to 10:
    
10: // Fixup code: set return value = bytes not copied
    movl %edx, %eax
    ret

// The exception table (in .init.ex_table section):
// .quad 1b        // Instruction address that might fault
// .quad 10b       // Fixup address to jump to

// arch/x86/mm/fault.c
static bool fixup_exception(struct pt_regs *regs, int trapnr, ...)
{
    // Search exception table for the faulting RIP:
    const struct exception_table_entry *e;
    e = search_exception_tables(regs->ip);
    if (!e)
        return false;  // Not in exception table: real kernel bug → panic
    
    // Found: jump to fixup code:
    regs->ip = e->fixup;  // This makes the faulting instruction "return"
    return true;           // to the fixup code instead
}
```

This is why `copy_from_user()` can safely handle an invalid user pointer — a page fault jumps to the fixup code which returns `-EFAULT` to the caller.

---

## 12. Fault Statistics

```bash
# Page fault counts:
cat /proc/vmstat | grep -E 'pgfault|pgmajfault|pswpin|pswpout'
# pgfault          1234567   ← Total page faults (minor = demand paging)
# pgmajfault          1234   ← Major faults (required disk I/O)

# Per-process fault counts:
cat /proc/$$/stat
# Field 10 (minflt): minor fault count
# Field 11 (cminflt): children's minor faults
# Field 12 (majflt): major fault count
# Field 13 (cmajflt): children's major faults

# Better: via getrusage()
# ru_minflt = minor faults
# ru_majflt = major faults (page-ins from disk)

# bpftrace: trace page faults:
sudo bpftrace -e '
tracepoint:exceptions:page_fault_user {
    @faults[comm] = count();
}'

# ftrace: trace major faults:
sudo bpftrace -e '
kretprobe:filemap_fault {
    if (retval & 4) {   // VM_FAULT_MAJOR = 4
        @major[comm] = count();
    }
}'
```

---

## 13. Stack Growth on Fault

The stack VMA has `VM_GROWSDOWN` set. When a fault occurs just below the current stack:

```c
// mm/mmap.c
int expand_stack(struct vm_area_struct *vma, unsigned long address)
{
    // Check: address must be within one stack guard page of vma->vm_start
    if (unlikely(address + PAGE_SIZE < vma->vm_start))
        return -ENOMEM;  // Too far — not a stack access, real fault
    
    // Check rlimit:
    if (vma->vm_end - address > rlimit(RLIMIT_STACK))
        return -ENOMEM;
    
    // Extend the VMA downward:
    vma->vm_start = address & PAGE_MASK;
    vma->vm_pgoff -= (vma->vm_start - old_start) >> PAGE_SHIFT;
    
    return 0;
}
```

```bash
# Increase stack size limit:
ulimit -s unlimited    # No stack limit (dangerous: can exhaust memory)
ulimit -s 65536        # 64MB stack

# Default: 8MB (8192KB)
ulimit -s
# 8192

# Kernel stack guard gap:
cat /proc/sys/vm/stack_guard_gap
# 256 pages = 1MB gap between stack and other mappings
```

---

## 14. `madvise()` and Fault Behavior

`madvise()` hints to the kernel about access patterns, affecting fault behavior:

```c
// Prefault all pages now (avoid future faults):
madvise(addr, len, MADV_POPULATE_READ);   // Pre-read all pages
madvise(addr, len, MADV_POPULATE_WRITE);  // Pre-write (COW if needed)

// Free pages now, re-fault on access:
madvise(addr, len, MADV_FREE);      // Pages can be freed (lazy)
madvise(addr, len, MADV_DONTNEED);  // Advise: don't need pages, free now

// Access pattern hints:
madvise(addr, len, MADV_SEQUENTIAL); // Expect sequential access → readahead
madvise(addr, len, MADV_RANDOM);     // Expect random access → no readahead
madvise(addr, len, MADV_NORMAL);     // Default behavior

// Huge page hints:
madvise(addr, len, MADV_HUGEPAGE);   // Try to use THP for this region
madvise(addr, len, MADV_NOHUGEPAGE); // Don't use THP here
```

---

## 15. Debugging Page Faults

```bash
# Count page faults for a command:
/usr/bin/time -v ls 2>&1 | grep "Major page faults\|Minor page faults"
# Minor (reclaiming a frame): 12345
# Major (requiring I/O): 2

# perf: sample on page fault events:
sudo perf record -e 'exceptions:page_fault_user' -ag sleep 5
sudo perf report

# perf top: see which addresses fault most:
sudo perf top -e page-faults

# bpftrace: print fault addresses:
sudo bpftrace -e '
tracepoint:exceptions:page_fault_user {
    printf("pid=%d addr=0x%lx error=%d\n",
        pid, args->address, args->error_code);
}' | head -50

# kprobes: trace COW faults:
sudo bpftrace -e '
kprobe:wp_page_copy {
    @cow_faults[comm] = count();
}'
```

---

## 16. Mental Model Checkpoint

After Day 28, you should be able to:

1. Draw the complete fault handler call chain from CPU exception to PTE installation.
2. What are the three types of page faults and how does the handler distinguish them?
3. Trace a fault on an anonymous page that was never written: what path does it take?
4. Trace a COW fault (write to shared read-only page): what does `do_wp_page()` do?
5. What is the exception table and how does it make `copy_from_user()` safe?
6. When does a stack grow on fault? What limits this?
7. What is a major fault vs minor fault? How do you count them?
8. What does `madvise(MADV_DONTNEED)` do to a region's page table entries?

---

## Key Source Files

```bash
arch/x86/mm/fault.c         # exc_page_fault(), do_user_addr_fault()
mm/memory.c                  # handle_mm_fault(), do_anonymous_page(), do_wp_page()
mm/filemap.c                 # filemap_fault() — file-backed page loading
mm/swap.c / mm/swapfile.c   # do_swap_page() — swap-in
arch/x86/lib/copy_user_64.S # Exception table entries for user copy
arch/x86/mm/extable.c       # fixup_exception() — exception table lookup
mm/mmap.c                    # expand_stack()
```

---

## Summary

Page faults are the core mechanism enabling lazy virtual memory. The complete path:

1. CPU: `#PF` exception → `CR2` = faulting address, error code on stack
2. `do_user_addr_fault()`: find VMA, check permissions
3. `handle_mm_fault()` → `handle_pte_fault()`: determine fault type
4. **No PTE**: anonymous → `do_anonymous_page()` (alloc zero page), file → `filemap_fault()` (page cache lookup/read)
5. **PTE present=0**: swapped → `do_swap_page()` (disk read)
6. **Protection fault**: write to RO → `do_wp_page()` (COW copy or reuse)
7. Install PTE, update reverse mapping, return to user space

The exception table ensures kernel code faulting on user pointers (in `copy_from_user`) is handled gracefully rather than panicking.

Tomorrow: the page cache — how file data is cached in RAM and how write-back works.
