# Day 14 — Process Creation: `fork()`, `clone()`, `vfork()`, `execve()`

> **Estimated read time:** 90–120 minutes  
> **Goal:** Trace every step of process creation from the syscall to the first instruction of the child process, and understand how execve replaces an image.

---

## 1. The Three Syscalls, One Kernel Function

User space has three process creation calls:

```c
pid_t fork(void);           // Create a child process (full copy)
pid_t vfork(void);          // Create a child, parent blocks until exec/exit
int clone(fn, stack, flags, arg, ...); // Fine-grained control
int clone3(struct clone_args *args, size_t size); // Modern extensible form
```

All of them ultimately call the same kernel function: **`kernel_clone()`** (formerly `_do_fork()`), which calls **`copy_process()`**.

```c
// kernel/fork.c
SYSCALL_DEFINE0(fork)
{
    struct kernel_clone_args args = { .exit_signal = SIGCHLD };
    return kernel_clone(&args);
}

SYSCALL_DEFINE0(vfork)
{
    struct kernel_clone_args args = {
        .flags     = CLONE_VFORK | CLONE_VM,
        .exit_signal = SIGCHLD,
    };
    return kernel_clone(&args);
}

SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
                int __user *, parent_tidptr, unsigned long, tls,
                int __user *, child_tidptr)
{
    struct kernel_clone_args args = {
        .flags       = (lower_32_bits(clone_flags) & ~CSIGNAL),
        .pidfd       = parent_tidptr,
        .child_tid   = child_tidptr,
        .parent_tid  = parent_tidptr,
        .exit_signal = (lower_32_bits(clone_flags) & CSIGNAL),
        .stack       = newsp,
        .tls         = tls,
    };
    return kernel_clone(&args);
}
```

---

## 2. `clone()` Flags — The Key to Understanding Fork vs Thread

The `clone_flags` argument controls what the child shares with the parent:

```c
// include/uapi/linux/sched.h
#define CLONE_VM        0x00000100  // Share address space (threads)
#define CLONE_FS        0x00000200  // Share filesystem info (cwd, root, umask)
#define CLONE_FILES     0x00000400  // Share open file descriptors
#define CLONE_SIGHAND   0x00000800  // Share signal handlers
#define CLONE_PIDFD     0x00001000  // Return a pidfd (file descriptor for the PID)
#define CLONE_PTRACE    0x00002000  // Trace child as parent is traced
#define CLONE_VFORK     0x00004000  // Parent blocks until child calls exec/exit
#define CLONE_PARENT    0x00008000  // Child has same parent as caller
#define CLONE_THREAD    0x00010000  // Add to the same thread group (pthread)
#define CLONE_NEWNS     0x00020000  // Create new mount namespace
#define CLONE_SYSVSEM   0x00040000  // Share SysV semaphore undo list
#define CLONE_SETTLS    0x00080000  // Set TLS (thread-local storage)
#define CLONE_PARENT_SETTID 0x00100000  // Write TID to parent_tidptr
#define CLONE_CHILD_CLEARTID 0x00200000 // Clear child_tid when thread exits
#define CLONE_DETACHED  0x00400000  // Unused
#define CLONE_UNTRACED  0x00800000  // Don't auto-attach ptrace
#define CLONE_CHILD_SETTID  0x01000000 // Write TID to child_tidptr in child
#define CLONE_NEWCGROUP 0x02000000  // Create new cgroup namespace
#define CLONE_NEWUTS    0x04000000  // Create new UTS namespace
#define CLONE_NEWIPC    0x08000000  // Create new IPC namespace
#define CLONE_NEWUSER   0x10000000  // Create new user namespace
#define CLONE_NEWPID    0x20000000  // Create new PID namespace
#define CLONE_NEWNET    0x40000000  // Create new network namespace
#define CLONE_IO        0x80000000  // Clone I/O context
```

What each call passes:

| Call | `CLONE_VM` | `CLONE_FILES` | `CLONE_SIGHAND` | `CLONE_THREAD` |
|------|-----------|--------------|----------------|----------------|
| `fork()` | No | No | No | No |
| `vfork()` | Yes | Yes | No | No |
| `pthread_create()` (via `clone()`) | Yes | Yes | Yes | Yes |
| Container (`unshare`) | No | No | No | + NEWNS, NEWPID... |

---

## 3. `copy_process()` — The Core

`copy_process()` in `kernel/fork.c` is one of the most important functions in the kernel. Here's its annotated structure:

```c
static __latent_entropy struct task_struct *copy_process(
    struct pid *pid,
    int trace,
    int node,
    struct kernel_clone_args *args)
{
    unsigned long clone_flags = args->flags;
    int retval;
    struct task_struct *p;
    
    // ─── Validation ─────────────────────────────────────────────────
    // Cannot have CLONE_SIGHAND without CLONE_VM:
    if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
        return ERR_PTR(-EINVAL);
    
    // Cannot have CLONE_THREAD without CLONE_SIGHAND:
    if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
        return ERR_PTR(-EINVAL);
    
    // ─── Check Resource Limits ──────────────────────────────────────
    retval = security_task_create(clone_flags);  // LSM hooks
    
    // ─── Allocate task_struct ───────────────────────────────────────
    p = dup_task_struct(current, node);
    // This:
    // 1. Allocates a new task_struct (from task_struct slab cache)
    // 2. Copies the ENTIRE current task_struct into it
    // 3. Allocates a new kernel stack (16KB via vmalloc if VMAP_STACK)
    // 4. Sets up the new stack canary
    
    // ─── Copy or Share Subsystem State ─────────────────────────────
    retval = copy_creds(p, clone_flags);  // Credentials
    
    // Initialize scheduling for new task:
    retval = sched_fork(clone_flags, p);  // Set p->state = TASK_NEW
    
    // Copy/share each subsystem:
    retval = copy_files(clone_flags, p, args->no_files);
    // CLONE_FILES set: p->files = current->files (shared, refcount++)
    // CLONE_FILES not: p->files = dup_fd(current->files) (full copy of fd table)
    
    retval = copy_fs(clone_flags, p);
    // CLONE_FS set:    p->fs = current->fs (shared)
    // CLONE_FS not:    p->fs = copy_fs_struct(current->fs) (copy with same cwd/root)
    
    retval = copy_sighand(clone_flags, p);
    // CLONE_SIGHAND: p->sighand = current->sighand (shared)
    // Otherwise:     p->sighand = copy_sighand(current->sighand) (copy handlers)
    
    retval = copy_signal(clone_flags, p);
    // CLONE_THREAD: p->signal = current->signal (same thread group)
    // Otherwise:    alloc new signal_struct (new process group)
    
    retval = copy_mm(clone_flags, p);
    // CLONE_VM: p->mm = current->mm (shared — thread)
    // Otherwise: p->mm = dup_mm(current) — Copy-on-Write fork
    
    retval = copy_namespaces(clone_flags, p);
    // CLONE_NEW*: create new namespace(s)
    // Otherwise: share parent's namespaces (refcount++)
    
    retval = copy_io(clone_flags, p);
    // CLONE_IO: share I/O context
    
    retval = copy_thread(p, args);
    // Architecture-specific: set up the new stack so the child
    // starts executing at the right place with the right registers
    
    // ─── Assign PID ─────────────────────────────────────────────────
    if (pid != &init_struct_pid) {
        pid = alloc_pid(p->nsproxy->pid_ns_for_children, ...);
    }
    p->pid = pid_nr(pid);
    p->tgid = CLONE_THREAD ? current->tgid : p->pid;
    
    // ─── Link into the system ────────────────────────────────────────
    // Attach to parent:
    INIT_LIST_HEAD(&p->children);
    INIT_LIST_HEAD(&p->sibling);
    
    write_lock_irq(&tasklist_lock);
    // Link child to parent:
    if (CLONE_THREAD) {
        p->group_leader = current->group_leader;
        list_add_tail_rcu(&p->thread_node, &p->signal->thread_head);
    } else {
        p->group_leader = p;
        list_add_tail(&p->sibling, &p->real_parent->children);
    }
    // Add to global process list:
    list_add_tail_rcu(&p->tasks, &init_task.tasks);
    write_unlock_irq(&tasklist_lock);
    
    // Wake up the child (it starts in TASK_NEW state):
    // (Actually done in kernel_clone() after copy_process returns)
    
    return p;
}
```

---

## 4. Copy-on-Write Memory (`copy_mm`)

When `fork()` is called without `CLONE_VM`, the child gets a **copy-on-write (COW)** copy of the parent's address space. This is the most important optimization in process creation.

### 4.1 How COW Works

Instead of physically copying 100MB of parent memory, the kernel:

1. Creates a new `mm_struct` for the child
2. Copies the VMA tree (virtual memory area descriptors) — cheap, just metadata
3. Marks **all** PTEs in both parent and child as **read-only**
4. Points the child's PTEs to the **same physical pages** as the parent

```
Parent PTEs:        Child PTEs:
page A → [RO]       page A → [RO]  ← same physical page!
page B → [RO]       page B → [RO]
page C → [RO]       page C → [RO]
```

5. When either process **writes** to a page → page fault
6. Kernel handles `do_wp_page()` (write-protected page fault):
   - Allocates a new physical page
   - Copies the content of the shared page to the new page
   - Updates the faulting process's PTE to point to the new page (writable)
   - Restores the other process's PTE to writable (if it was the only sharer)

```
After parent writes to page A:
Parent PTEs:        Child PTEs:
page A' → [RW]      page A  → [RW]  ← now separate pages
page B  → [RW]      page B  → [RW]  ← still shared (not written)
page C  → [RW]      page C  → [RW]
```

### 4.2 COW Implementation in Code

```c
// mm/memory.c
static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct page *old_page = vmf->page;
    
    // If we're the only user of this page (no other COW copies):
    if (can_reuse_swap_page(old_page)) {
        // Just make the PTE writable again — no copy needed
        entry = pte_mkwrite(pte_mkdirty(vmf->orig_pte), vmf->vma);
        set_pte_at(vmf->vma->vm_mm, vmf->address, vmf->pte, entry);
        return VM_FAULT_WRITE;
    }
    
    // Multiple users: must copy
    new_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vmf->vma, vmf->address);
    copy_user_highpage(new_page, old_page, vmf->address, vmf->vma);
    
    // Install the new page in child's page table:
    entry = mk_pte(new_page, vma->vm_page_prot);
    entry = pte_mkwrite(pte_mkdirty(entry), vmf->vma);
    set_pte_at(mm, vmf->address, vmf->pte, entry);
    
    // Drop reference to old shared page:
    put_page(old_page);
    
    return VM_FAULT_WRITE;
}
```

### 4.3 Fork Performance

```bash
# Time a fork-heavy workload:
time for i in $(seq 1 10000); do true; done
# Each 'true' is a fork + exec

# strace to see what happens:
strace /bin/bash -c 'true' 2>&1 | head -20
# execve("/bin/bash", ...) = 0     ← shell starts
# clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|SIGCHLD) = 12345  ← fork
# wait4(12345, ...)               ← wait for child
```

---

## 5. `copy_thread()` — The Architecture-Specific Part

After `copy_process()` creates and sets up the new `task_struct`, it calls `copy_thread()` (arch-specific) to set up the **kernel stack of the child** so it can be scheduled.

```c
// arch/x86/kernel/process.c
int copy_thread(struct task_struct *p, const struct kernel_clone_args *args)
{
    unsigned long clone_flags = args->flags;
    unsigned long sp = args->stack;   // Child's user stack (for threads)
    
    struct pt_regs *childregs;
    
    // The child's kernel stack will look like we just did a syscall.
    // Get pointer to where pt_regs should go (top of kernel stack minus size):
    childregs = task_pt_regs(p);
    
    // Copy parent's registers to child:
    *childregs = *current_pt_regs();
    
    // The child returns 0 from fork():
    childregs->ax = 0;   // RAX = return value = 0 in the child
    
    // If a new stack was specified (threads), set RSP:
    if (sp)
        childregs->sp = sp;
    
    // Set up child's kernel stack so schedule() will return to:
    // ret_from_fork → user space (via childregs)
    p->thread.sp = (unsigned long)childregs;  // Kernel SP points to pt_regs
    p->thread.sp0 = (unsigned long)(childregs + 1);
    
    // Child resumes from: ret_from_fork
    p->thread.ip = (unsigned long)ret_from_fork;
    
    // TLS setup for threads:
    if (clone_flags & CLONE_SETTLS)
        p->thread.fsbase = args->tls;
    
    return 0;
}
```

The key trick: the child's kernel stack is pre-set so when the scheduler runs the child for the first time, it "returns" through `ret_from_fork` → `ret_from_fork_asm` → exits syscall to user space with `RAX = 0`.

```asm
/* arch/x86/entry/entry_64.S */
SYM_CODE_START(ret_from_fork_asm)
    /* Child enters here on first schedule */
    movq    %rax, %rdi          /* pt_regs pointer */
    call    ret_from_fork       /* C function: run schedule hooks, TIF flags */
    jmp     swapgs_restore_regs_and_return_to_usermode
SYM_CODE_END(ret_from_fork_asm)
```

---

## 6. `kernel_clone()` — Waking the Child

```c
// kernel/fork.c
pid_t kernel_clone(struct kernel_clone_args *args)
{
    struct task_struct *p;
    pid_t nr;
    
    // Create the new task_struct:
    p = copy_process(NULL, 0, NUMA_NO_NODE, args);
    if (IS_ERR(p))
        return PTR_ERR(p);
    
    // Notify ptrace if being traced:
    trace_sched_process_fork(current, p);
    
    // Get the PID (from child's namespace):
    nr = pid_vnr(get_task_pid(p, PIDTYPE_PID));
    
    // CLONE_VFORK: parent must wait for child to exec/exit
    if (args->flags & CLONE_VFORK) {
        p->vfork_done = &vfork;
        init_completion(&vfork);
    }
    
    // Wake up the child — make it runnable:
    wake_up_new_task(p);  // Sets state TASK_RUNNING, adds to run queue
    
    // If CLONE_VFORK: parent blocks here until child signals:
    if (args->flags & CLONE_VFORK) {
        wait_for_vfork_done(p, &vfork);
    }
    
    // Return child PID to parent:
    return nr;
}
```

---

## 7. After `fork()` — What Happens on Each Side

```
Parent: kernel_clone() returns child PID
    │
    └── returns to user space from syscall
        write(1, "parent\n", 7) ...

Child: wake_up_new_task() adds it to a CPU's run queue
    │
    └── scheduler picks it up
        → ret_from_fork_asm
        → ret_from_fork (runs syscall exit hooks)
        → exits syscall with RAX=0
        → user space: fork() returned 0

Both processes now run concurrently.
```

---

## 8. `vfork()` — Synchronous Fork

`vfork()` is an optimization for the common `fork()` + `exec()` pattern. The key difference:

- Parent's address space is **shared** (not COW-copied)
- Parent is **blocked** until the child calls `exec()` or `_exit()`
- Child runs first, on the parent's address space
- Child MUST NOT write to variables the parent uses, return from the calling function, or call most library functions

```c
// Usage pattern — the ONLY correct use of vfork:
pid_t pid = vfork();  // CLONE_VM | CLONE_VFORK
if (pid == 0) {
    // Child: DO NOT touch parent's memory!
    // Only allowed: exec*() or _exit()
    execl("/bin/ls", "ls", NULL);   // Replaces address space → releases vfork
    _exit(127);   // If exec fails: use _exit, NOT exit() (which flushes stdio)
} else {
    // Parent: was blocked, now resumed because child exec'd/exited
    wait(NULL);
}
```

vfork is rarely used directly today — but it's used internally by shells for performance.

---

## 9. `execve()` — Replacing the Process Image

`exec()` replaces the current process's memory image with a new program. The PID remains the same; everything else changes.

### 9.1 Syscall Entry

```c
// fs/exec.c
SYSCALL_DEFINE3(execve,
    const char __user *, filename,
    const char __user *const __user *, argv,
    const char __user *const __user *, envp)
{
    return do_execve(getname(filename), argv, envp);
}
```

### 9.2 `do_execve()` and `bprm`

```c
// The binary program parameters structure:
struct linux_binprm {
    struct vm_area_struct *vma;     // VMA for the interpreter page
    unsigned long vma_pages;
    struct mm_struct *mm;           // New mm (being built)
    unsigned long p;                // Current top of stack
    unsigned int called_exec_mmap:1; // mm installed?
    struct file *file;              // The executable file
    struct cred *cred;              // New credentials
    int unsafe;                     // Is exec unsafe?
    unsigned int per_clear;         // Personality flags to clear
    int argc, envc;                 // Argument and environment counts
    const char *filename;           // Filename for /proc/<pid>/exe
    const char *interp;             // Interpreter name (for scripts)
    const char *fdpath;             // Path of fd being exec'd
    unsigned int interp_flags;
    int execfd;                     // fd for scripts (#! interpreters)
    unsigned long loader;           // Loader address
    unsigned long exec;             // Entry point
    struct rlimit rlim_stack;       // Stack rlimit
    char buf[BINPRM_BUF_SIZE];     // First 256 bytes of binary (for format detection)
};
```

### 9.3 The `execve()` Sequence

```c
// fs/exec.c — simplified flow
int do_execve(struct filename *filename,
              const char __user *const __user *argv,
              const char __user *const __user *envp)
{
    struct linux_binprm *bprm;
    
    // Allocate bprm:
    bprm = alloc_bprm(fd, filename);
    
    // Read first 256 bytes to identify the binary format:
    kernel_read(bprm->file, bprm->buf, BINPRM_BUF_SIZE, &pos);
    
    // Count argv and envp:
    bprm->argc = count(argv, MAX_ARG_STRINGS);
    bprm->envc = count(envp, MAX_ARG_STRINGS);
    
    // Point of no return: 
    retval = exec_binprm(bprm);
    // After this, if it fails, the process is killed (no recovery)
    
    // If successful: we're now running the new binary
    // This function never returns on success
}
```

### 9.4 Binary Format Detection

```c
// fs/binfmt_elf.c, fs/binfmt_script.c, etc.
// Each format registers a handler:
static struct linux_binfmt elf_format = {
    .module        = THIS_MODULE,
    .load_binary   = load_elf_binary,
    .load_shlib    = load_elf_library,
    .core_dump     = elf_core_dump,
    .min_coredump  = ELF_EXEC_PAGESIZE,
};

// fs/exec.c: search_binary_handler() tries each format:
int search_binary_handler(struct linux_binprm *bprm)
{
    list_for_each_entry(fmt, &formats, lh) {
        retval = fmt->load_binary(bprm);
        if (retval != -ENOEXEC) break;  // Found a handler
    }
    return retval;
}

// Format detection by magic bytes:
// ELF:    \x7fELF
// Script: #!
// Java:   PK\x03\x04 (JAR file via binfmt_misc)
// etc.
```

### 9.5 `load_elf_binary()` — Loading an ELF

```c
// fs/binfmt_elf.c (simplified flow)
static int load_elf_binary(struct linux_binprm *bprm)
{
    // 1. Validate ELF header
    // 2. Load program headers (PT_LOAD segments)
    // 3. If there's a PT_INTERP (dynamic linker path, e.g., /lib64/ld-linux-x86-64.so.2):
    //    - Load the interpreter into the address space
    //    - Entry point becomes the interpreter, not the ELF itself
    
    // 4. Call exec_mmap() — this is the point of no return:
    retval = exec_mmap(bprm->mm);
    // This replaces current->mm with the new mm (no going back)
    
    // 5. Set up the stack with argv, envp, auxv
    retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP), ...);
    
    // 6. Map the binary's PT_LOAD segments:
    for each PT_LOAD: {
        elf_map(bprm->file, load_bias + vaddr, elf_ppnt, elf_prot, elf_type);
    }
    
    // 7. Set entry point:
    // (ld.so address if interpreter exists, else ELF e_entry)
    start_thread(regs, elf_entry, bprm->p);
    // Sets RIP = elf_entry, RSP = bprm->p (top of stack with args)
    
    // 8. When this function returns, the syscall return path goes
    //    to the new entry point in the new address space
}
```

### 9.6 What `exec()` Changes vs Keeps

**Changed:**
- Address space (`mm_struct`) — entirely new
- Open files (except those with `FD_CLOEXEC` / `O_CLOEXEC` set)
- Signal handlers (reset to `SIG_DFL`)
- Signal alternate stack
- PID's `comm` (executable name)
- Memory maps, heap, stack

**Preserved:**
- PID and TGID
- Parent-child relationships
- Open files without `FD_CLOEXEC`
- File descriptor table (minus closed-on-exec FDs)
- Current working directory
- Signal mask
- Process group and session
- Real UID/GID (effective may change for setuid binaries)
- Pending signals

---

## 10. setuid Executables and Credential Changes

When a setuid binary is exec'd, `exec_mmap()` triggers credential update:

```c
// fs/exec.c
int begin_new_exec(struct linux_binprm *bprm)
{
    // ...
    // Check if the binary is setuid:
    if (bprm->file->f_path.dentry->d_inode->i_mode & S_ISUID) {
        bprm->per_clear |= PER_CLEAR_ON_SETID;
        // Set effective UID to file owner's UID:
        bprm->cred->euid = bprm->file->f_path.dentry->d_inode->i_uid;
    }
    if (bprm->file->f_path.dentry->d_inode->i_mode & S_ISGID) {
        bprm->cred->egid = bprm->file->f_path.dentry->d_inode->i_gid;
    }
    // ...
}
```

This is how `sudo` and `su` work — they're setuid-root binaries.

---

## 11. `posix_spawn()` vs `fork()` + `exec()`

`posix_spawn()` (glibc) uses `vfork()` + `exec()` internally for efficiency:

```c
// glibc implementation (conceptual):
int posix_spawn(pid_t *pid, const char *path,
                const posix_spawn_file_actions_t *file_actions,
                const posix_spawnattr_t *attrp,
                char *const argv[], char *const envp[])
{
    // Uses vfork (CLONE_VM | CLONE_VFORK) in new glibc
    // Or clone with CLONE_VM in even newer versions
    // Child only calls exec or _exit
    // Parent waits for vfork completion
}
```

Modern glibc uses `clone()` with specific flags to avoid even `vfork`'s constraints.

---

## 12. `fork()` in the Shell

Understanding how the shell uses fork shows the real-world pattern:

```bash
# Shell executes: ls | grep foo

# 1. Shell forks child 1:
pid1 = fork()
if pid1 == 0:    # Child 1
    # Set up pipe: stdout → write end of pipe
    dup2(pipe_fd[1], STDOUT_FILENO)
    close(pipe_fd[0])
    execl("/bin/ls", "ls", NULL)

# 2. Shell forks child 2:
pid2 = fork()
if pid2 == 0:    # Child 2
    # Set up pipe: stdin ← read end of pipe
    dup2(pipe_fd[0], STDIN_FILENO)
    close(pipe_fd[1])
    execl("/bin/grep", "grep", "foo", NULL)

# 3. Parent closes pipe ends and waits:
close(pipe_fd[0])
close(pipe_fd[1])
waitpid(pid1, NULL, 0)
waitpid(pid2, NULL, 0)
```

---

## 13. The `pidfd` API — Modern Process Handles

Linux 5.2 introduced `pidfd` — file descriptors that reference a process, immune to PID reuse races:

```c
// Create a pidfd when forking:
struct clone_args args = {
    .flags    = CLONE_PIDFD,
    .pidfd    = (uint64_t)&child_pidfd,
    .exit_signal = SIGCHLD,
};
pid_t pid = syscall(SYS_clone3, &args, sizeof(args));
// child_pidfd is now a file descriptor referring to the child

// Send a signal via pidfd (safe from PID reuse):
syscall(SYS_pidfd_send_signal, child_pidfd, SIGTERM, NULL, 0);

// Wait via pidfd:
struct pollfd pfd = { .fd = child_pidfd, .events = POLLIN };
poll(&pfd, 1, -1);  // Blocks until child exits

// Get task info via pidfd:
int proc_fd = syscall(SYS_pidfd_open, getpid(), 0);
```

---

## 14. Kernel Thread Creation

Kernel threads are created differently — they don't have a user-space address space:

```c
// kernel/kthread.c
struct task_struct *kthread_create(int (*threadfn)(void *data),
                                   void *data,
                                   const char namefmt[], ...);

struct task_struct *kthread_create_on_node(int (*threadfn)(void *),
                                           void *data,
                                           int node,
                                           const char namefmt[], ...);

// Create and immediately run:
struct task_struct *kthread_run(int (*threadfn)(void *data),
                                void *data,
                                const char namefmt[], ...);

// Stop a kthread (sets kthread_should_stop() to true, wakes it, waits):
int kthread_stop(struct task_struct *k);

// kthread checks this to know when to exit:
bool kthread_should_stop(void);

// Example: a simple kernel thread:
static int my_kthread_fn(void *data)
{
    while (!kthread_should_stop()) {
        do_periodic_work();
        msleep(1000);  // Can sleep!
    }
    return 0;
}

// In module init:
struct task_struct *task;
task = kthread_run(my_kthread_fn, NULL, "my_kthread");
if (IS_ERR(task)) return PTR_ERR(task);

// In module exit:
kthread_stop(task);
```

Kernel threads have `task->mm == NULL` (use `active_mm` borrowed from last user task). They appear in `ps` with brackets: `[kworker/0:1]`.

---

## 15. Process Limits

```bash
# Resource limits (ulimit):
ulimit -a
# max user processes (-u) = 63363   ← per-user process limit

# System-wide limits:
cat /proc/sys/kernel/pid_max      # Max PID number
cat /proc/sys/kernel/threads-max  # Max threads system-wide
cat /proc/$$/limits               # Per-process limits

# Kernel enforcement in copy_process():
# - RLIMIT_NPROC: max processes per user
# - RLIMIT_MEMLOCK: max locked memory
# - Namespace PID limit: pid_ns->pid_max
```

---

## 16. Tracing `fork()` with bpftrace

```bash
# Trace all forks with parent/child info:
sudo bpftrace -e '
tracepoint:sched:sched_process_fork {
    printf("fork: parent=%s(%d) → child=%s(%d)\n",
        args->parent_comm, args->parent_pid,
        args->child_comm, args->child_pid);
}'

# Trace execve calls (what programs are being run):
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_execve {
    printf("exec: pid=%d comm=%s file=%s\n",
        pid, comm, str(args->filename));
}'

# Measure fork latency:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_clone { @start[tid] = nsecs; }
tracepoint:syscalls:sys_exit_clone  /@start[tid]/ {
    @fork_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}'
```

---

## 17. Mental Model Checkpoint

After Day 14, you should be able to:

1. Explain what `CLONE_VM`, `CLONE_FILES`, `CLONE_THREAD` each do.
2. Draw the `copy_process()` flow and name each subsystem copy step.
3. Explain Copy-on-Write: what's shared, what's separate, what triggers a copy?
4. What does `copy_thread()` do and why does the child return 0 from `fork()`?
5. What is the "point of no return" in `execve()` and why does it matter?
6. Trace an ELF load: from `execve()` to the first instruction of `main()`.
7. What happens to open file descriptors across `exec()`? What does `FD_CLOEXEC` do?
8. How does a kernel thread differ from a user process in terms of `mm`?
9. What is a `pidfd` and what problem does it solve?
10. How does `vfork()` differ from `fork()` and what are its restrictions?

---

## Key Source Files

```bash
kernel/fork.c               # copy_process(), kernel_clone(), dup_mm()
arch/x86/kernel/process.c  # copy_thread() x86 implementation
fs/exec.c                   # do_execve(), exec_binprm(), begin_new_exec()
fs/binfmt_elf.c             # load_elf_binary()
fs/binfmt_script.c          # load_script() — shebang (#!) handling
mm/memory.c                 # do_wp_page() — COW fault handler
kernel/kthread.c            # kthread_create(), kthread_run(), kthread_stop()
include/linux/sched/task.h  # for_each_process, for_each_process_thread
include/uapi/linux/sched.h  # CLONE_* flags
```

---

## Summary

Process creation in Linux is a masterclass in efficient sharing:

- `fork()` = `copy_process()` with no sharing flags → child gets COW copy of memory, separate file table, separate signal handlers, new PID
- `pthread_create()` = `clone()` with `CLONE_VM | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD` → all the important things are shared, only the stack and TID differ
- `exec()` = replaces the current process image via ELF loading, keeping the PID

The COW optimization means `fork()` followed immediately by `exec()` (the shell pattern) copies almost no memory — the child replaces its address space before touching the parent's pages. Only file descriptors and a few kernel structures require actual duplication.

Tomorrow: process states and lifecycle — zombie processes, orphans, wait(), and what happens during process exit.
