# Day 4 — System Calls: Mechanics

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand exactly how a `write()` call in your C program becomes kernel code execution — every step.

---

## 1. What a System Call Is

A system call is the **controlled gate** between user space and kernel space. It is the *only* legitimate way for a user-space program to request kernel services. There is no other way.

When your program calls `write(1, "hello\n", 6)`:
- It's not calling kernel code directly (that would be a privilege violation)
- It's not signaling anything
- It calls the C library's `write()` wrapper, which executes a special CPU instruction (`SYSCALL` on x86_64) that atomically:
  1. Switches the CPU from Ring 3 to Ring 0
  2. Switches the stack from the user stack to the kernel stack
  3. Jumps to a fixed kernel entry point
  4. Passes control to the kernel's syscall dispatcher

The kernel then executes the requested operation (kernel's `sys_write()`) and returns.

Linux on x86_64 has ~350+ system calls. You can see them all in:
```bash
ausyscall --dump              # using audit library
cat /usr/include/asm/unistd_64.h  # raw numbers
man 2 syscalls               # man page list
```

---

## 2. The x86_64 SYSCALL Mechanism

### 2.1 The `SYSCALL` Instruction

`SYSCALL` is the fast syscall instruction on x86_64 (Intel/AMD). It does the following **atomically**:

1. Saves `RIP` (return address) into `RCX`
2. Saves `RFLAGS` into `R11`
3. Loads `CS` selector from `IA32_STAR` MSR bits [47:32]
4. Loads `SS` selector from `IA32_STAR` MSR bits [47:32] + 8
5. Clears `RFLAGS` bits specified in `IA32_FMASK` MSR (disables interrupts: `IF` flag cleared)
6. Sets `RIP` to the value in `IA32_LSTAR` MSR — the syscall entry point

The kernel sets up these MSRs during `syscall_init()` (called from `cpu_init()`):

```c
// arch/x86/kernel/cpu/common.c
void syscall_init(void)
{
    wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
    wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
    wrmsrl(MSR_CSTAR, (unsigned long)entry_SYSCALL_compat);
    wrmsrl(MSR_SYSCALL_MASK, X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|...);
}
```

So `IA32_LSTAR` points to `entry_SYSCALL_64` — the kernel's syscall handler.

### 2.2 Calling Convention for Syscalls

Linux's x86_64 syscall ABI (NOT the same as the C ABI):

| Register | Purpose |
|----------|---------|
| `RAX` | Syscall number (input) / return value (output) |
| `RDI` | Argument 1 |
| `RSI` | Argument 2 |
| `RDX` | Argument 3 |
| `R10` | Argument 4 (NOT `RCX` — that's saved by `SYSCALL`) |
| `R8` | Argument 5 |
| `R9` | Argument 6 |
| `RCX` | Trashed (holds return address after SYSCALL) |
| `R11` | Trashed (holds saved RFLAGS) |

Maximum 6 arguments. If a syscall needs more (rare), it takes a pointer to a struct.

### 2.3 User-Space Side (What glibc Does)

```c
// How glibc implements write() — simplified
// (actual is in sysdeps/unix/sysv/linux/x86_64/syscall.S)
ssize_t write(int fd, const void *buf, size_t count)
{
    // In assembly:
    // mov rax, 1       (SYS_write = 1)
    // mov rdi, fd
    // mov rsi, buf
    // mov rdx, count
    // syscall
    // cmp rax, -4095   (check for error: -MAX_ERRNO to -1)
    // jae error_path   (set errno and return -1)
    // ret
    
    long retval;
    asm volatile(
        "syscall"
        : "=a"(retval)
        : "0"(1), "D"(fd), "S"(buf), "d"(count)
        : "rcx", "r11", "memory"
    );
    if (retval < 0) {
        errno = -retval;
        return -1;
    }
    return retval;
}
```

Error handling convention: syscalls return negative errno values on error (e.g., -ENOENT = -2). The C library converts these to `-1` return value + `errno = 2`.

---

## 3. The Syscall Entry Point: `entry_SYSCALL_64`

This is the actual kernel code that runs when a user calls `syscall`. It's in assembly:

```asm
/* arch/x86/entry/entry_64.S */
SYM_CODE_START(entry_SYSCALL_64)
    UNWIND_HINT_ENTRY
    ENDBR
    
    /* At this point:
     * - We're in Ring 0 (kernel mode)
     * - Stack pointer is still the USER's RSP (!)
     * - CS = kernel code segment
     * - RCX = user's return RIP (saved by SYSCALL)
     * - R11 = user's RFLAGS (saved by SYSCALL)
     * - RAX = syscall number
     * - RDI,RSI,RDX,R10,R8,R9 = syscall arguments
     */

    /* Switch from user stack to kernel stack.
     * The kernel stack address is stored in the CPU's
     * TSS (Task State Segment) .sp2 field, which the
     * kernel keeps updated on every context switch. */
    swapgs                          /* Swap GS: user GS ↔ kernel GS (per-CPU ptr) */
    movq    %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  /* Save user RSP */
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp             /* KPTI: switch page tables */
    movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp /* Load kernel stack */

    /* Build the pt_regs structure on the kernel stack.
     * pt_regs is the saved user-mode register state. */
    pushq   $__USER_DS              /* SS */
    pushq   PER_CPU_VAR(cpu_tss_rw + TSS_sp2)  /* RSP (user stack) */
    pushq   %r11                    /* RFLAGS */
    pushq   $__USER_CS              /* CS */
    pushq   %rcx                    /* RIP (return address to user) */
    pushq   %rax                    /* orig_rax (syscall number) */
    PUSH_AND_CLEAR_REGS             /* Push R15..R0 (all GPRs), clear with 0 */

    /* Now RSP points to a struct pt_regs on the kernel stack */
    movq    %rsp, %rdi              /* First arg: pt_regs pointer */

    /* Dispatch: call the C handler */
    call    do_syscall_64

    /* Return path: restore registers and return to user */
    ...
    SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi  /* KPTI: restore user page tables */
    swapgs
    sysretq                         /* Return to user space */
SYM_CODE_END(entry_SYSCALL_64)
```

### 3.1 The `pt_regs` Structure

The assembly above pushes all user registers onto the kernel stack, forming a `struct pt_regs`:

```c
// arch/x86/include/asm/ptrace.h
struct pt_regs {
    unsigned long r15;
    unsigned long r14;
    unsigned long r13;
    unsigned long r12;
    unsigned long rbp;
    unsigned long rbx;
    /* rest: pushed by entry_SYSCALL_64 */
    unsigned long r11;
    unsigned long r10;
    unsigned long r9;
    unsigned long r8;
    unsigned long rax;    /* syscall number / return value */
    unsigned long rcx;    /* user RIP (return address) */
    unsigned long rdx;
    unsigned long rsi;
    unsigned long rdi;
    unsigned long orig_rax;  /* original syscall number (for restart) */
    /* Frame from hardware: */
    unsigned long rip;
    unsigned long cs;
    unsigned long eflags;
    unsigned long rsp;    /* user stack pointer */
    unsigned long ss;
};
```

This is also what `ptrace()` sees when you attach a debugger. It's the complete user-mode CPU state.

---

## 4. `do_syscall_64` — The Dispatcher

```c
// arch/x86/entry/common.c
__visible noinstr void do_syscall_64(struct pt_regs *regs, int nr)
{
    add_random_kstack_offset();   // Mitigate stack info leaks
    nr = syscall_enter_from_user_mode(regs, nr);   // seccomp, ptrace hooks

    // Bounds check the syscall number
    if (likely(nr < NR_syscalls)) {
        // Look up the syscall in the table
        // sys_call_table[nr] is a function pointer to the actual handler
        nr = array_index_nospec(nr, NR_syscalls);  // spectre mitigation
        regs->ax = sys_call_table[nr](regs);
    } else {
        regs->ax = -ENOSYS;
    }
    
    syscall_exit_to_user_mode(regs);   // seccomp, ptrace, signal handling
}
```

### 4.1 The Syscall Table

`sys_call_table` is an array of function pointers, indexed by syscall number:

```c
// Generated from: arch/x86/entry/syscalls/syscall_64.tbl
// Via script: scripts/syscalls/syscalltbl.sh
// Into: arch/x86/include/generated/asm/syscalls_64.h

// arch/x86/entry/syscall_64.c
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &__x64_sys_ni_syscall,  // default: -ENOSYS
    [__NR_read]   = __x64_sys_read,        // 0
    [__NR_write]  = __x64_sys_write,       // 1
    [__NR_open]   = __x64_sys_open,        // 2
    [__NR_close]  = __x64_sys_close,       // 3
    [__NR_stat]   = __x64_sys_stat,        // 4
    // ...
};
```

The table lives in read-only memory. Writing to it (which a rootkit might try) would trigger a page fault or be blocked by write-protect mechanisms.

---

## 5. Defining a System Call: `SYSCALL_DEFINE`

All syscall implementations use the `SYSCALL_DEFINE` macro family:

```c
// include/linux/syscalls.h
#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
// etc up to SYSCALL_DEFINE6

#define SYSCALL_DEFINEx(x, sname, ...)          \
    SYSCALL_METADATA(sname, x, __VA_ARGS__)     \
    __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

Example — `write()` syscall:

```c
// fs/read_write.c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count)
{
    return ksys_write(fd, buf, count);
}

// Expands to something like:
asmlinkage long __x64_sys_write(const struct pt_regs *regs)
{
    // Extract arguments from pt_regs
    return __se_sys_write(
        regs->di,   // fd (arg 1)
        regs->si,   // buf (arg 2)  
        regs->dx    // count (arg 3)
    );
}

static long __se_sys_write(unsigned int fd, const char __user *buf, size_t count)
{
    // Type checking, then call:
    long ret = __do_sys_write(fd, buf, count);
    return ret;
}

static inline long __do_sys_write(unsigned int fd, const char __user *buf, size_t count)
{
    return ksys_write(fd, buf, count);
}
```

The multi-layer wrapping (`__x64_sys_`, `__se_sys_`, `__do_sys_`) exists for:
- Type safety (catches ABI mismatches)
- Syscall tracing infrastructure (BPF, ftrace)
- Compat layer (32-bit syscalls on 64-bit kernel)

### 5.1 The `__user` Annotation

Note `const char __user *buf`. The `__user` annotation tells:
- `sparse` (static analyzer): this pointer points to user space; don't dereference directly
- Developers: use `copy_from_user()` / `copy_to_user()` to access it

**NEVER dereference a `__user` pointer directly** in kernel code. The user might have:
- Passed a NULL pointer
- Passed a kernel address (which, with SMAP disabled, you could accidentally dereference)
- Passed an address to MMIO memory
- Modified the memory from another thread after you checked it (TOCTOU)

```c
// WRONG (and dangerous):
char ch = *buf;    // Direct dereference of user pointer!

// CORRECT:
char ch;
if (copy_from_user(&ch, buf, 1))
    return -EFAULT;  // User provided invalid pointer
```

---

## 6. `copy_from_user` / `copy_to_user`

These are the safe bridges between user and kernel memory:

```c
// include/linux/uaccess.h
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);

// Return value: number of bytes NOT copied (0 = success)
// Caller should check: if (copy_from_user(&val, uptr, sizeof(val))) return -EFAULT;
```

Internally, on x86_64:

```c
// arch/x86/lib/copy_user_64.S (simplified concept)
copy_from_user:
    // STAC instruction: temporarily enable SMAP access
    // (allows kernel to access user pages)
    stac
    
    // REP MOVSQ: fast bulk copy
    // With exception table entries so page faults are handled gracefully
    rep movsq   // exception if user address is invalid/unmapped
    
    // CLAC: re-enable SMAP protection
    clac
    
    // If exception occurred, fixup code returns # bytes not copied
```

The "exception table" (`__ex_table`) contains fixup entries: if the `rep movsq` causes a page fault at a specific IP, jump to the fixup code instead of killing the kernel. This is what makes `copy_from_user` safe — a bad user pointer causes a clean -EFAULT return, not a kernel panic.

```c
// Scalar shortcuts (avoid the rep overhead for small copies):
get_user(x, ptr)    // read one value from user space
put_user(x, ptr)    // write one value to user space
// These are like: if(get_user(x, ptr)) return -EFAULT;
```

---

## 7. The Complete `write()` Call Path

Let's trace `write(1, "hello\n", 6)` end-to-end:

```
User: write(1, "hello\n", 6)
  ↓
glibc write() wrapper:
    mov rax, 1   (SYS_write)
    mov rdi, 1   (fd)
    mov rsi, buf
    mov rdx, 6
    syscall
  ↓
CPU SYSCALL instruction:
    saves RIP → RCX, RFLAGS → R11
    switches CS/SS to kernel segments
    sets RIP = IA32_LSTAR (entry_SYSCALL_64)
  ↓
entry_SYSCALL_64:
    swapgs (get kernel per-CPU data)
    switch to kernel stack
    KPTI: switch CR3 to kernel page tables
    push all regs (build pt_regs)
    call do_syscall_64(pt_regs, rax=1)
  ↓
do_syscall_64():
    syscall_enter_from_user_mode()  ← seccomp/ptrace checks
    sys_call_table[1](regs)         ← calls __x64_sys_write
  ↓
__x64_sys_write(regs):
    extracts fd=1, buf=ptr, count=6
    calls ksys_write(1, ptr, 6)
  ↓
ksys_write():
    fdget_pos(fd) → gets file* for fd=1 (stdout)
    checks f_op->write or f_op->write_iter
    calls vfs_write(file, buf, count, &pos)
  ↓
vfs_write():
    check permissions (file->f_mode & FMODE_WRITE)
    rw_verify_area()
    __vfs_write(file, buf, count, pos)
  ↓
__vfs_write():
    file->f_op->write_iter (or write)
    ← for a tty/terminal, this is tty_write()
  ↓
tty_write():
    copy_from_user() to get the data
    tty_ldisc_ref() → line discipline
    ld->ops->write() → n_tty_write()
  ↓
n_tty_write():
    process_output_block() → sends to tty driver
  ↓
... tty driver → serial or pty → display
  ↓
Return path (back up the call chain):
    each function returns bytes written
  ↓
do_syscall_64():
    stores return value in regs->ax
    syscall_exit_to_user_mode()  ← signals, ptrace, context switch?
  ↓
entry_SYSCALL_64 return path:
    pop all regs from pt_regs
    KPTI: switch CR3 back to user page tables
    swapgs (restore user GS)
    sysretq  ← restores RIP=RCX, RFLAGS=R11, switches to Ring 3
  ↓
glibc returns to user code:
    checks RAX for error (< -4095)
    returns bytes written (or -1 + errno)
  ↓
User: write() returns 6
```

---

## 8. System Call Numbers

```bash
# x86_64 syscall numbers (/usr/include/asm/unistd_64.h):
#define __NR_read        0
#define __NR_write       1
#define __NR_open        2
#define __NR_close       3
#define __NR_stat        4
#define __NR_fstat       5
#define __NR_lstat       6
#define __NR_poll        7
#define __NR_lseek       8
#define __NR_mmap        9
#define __NR_mprotect   10
#define __NR_munmap     11
#define __NR_brk        12
#define __NR_rt_sigaction 13
...
#define __NR_clone       56
#define __NR_fork        57
#define __NR_execve      59
...
#define __NR_getpid      39
#define __NR_kill        62
...
#define __NR_socket     41
#define __NR_connect    42
#define __NR_accept     43
#define __NR_sendto     44
#define __NR_recvfrom   45
```

Syscall numbers are **architecture-specific** and **ABI-stable** (they never change for a given arch). That's why x86_64 has different numbers than ARM64, and why a number that exists on x86_64 might not exist on RISC-V.

---

## 9. vDSO — Zero-Cost Syscalls

Some syscalls are called so frequently (gettimeofday, clock_gettime) that even the `SYSCALL` instruction overhead matters. The **vDSO** (virtual Dynamic Shared Object) solves this.

### 9.1 How vDSO Works

The kernel maps a read-only page into every process's address space containing:
- A pre-compiled shared library (position-independent code)
- Shared data structures updated by the kernel (clock values, etc.)

User-space calls `clock_gettime()`, which glibc resolves to the vDSO version, which reads the shared memory directly **without trapping into the kernel**.

```bash
# vDSO in a process's memory:
cat /proc/self/maps | grep vdso
# 7fff12345000-7fff12346000 r-xp 00000000 00:00 0  [vdso]

# Disassemble the vDSO:
vdso_addr=$(cat /proc/self/maps | grep vdso | awk -F- '{print $1}')
dd if=/proc/self/mem bs=4096 count=1 skip=$((0x$vdso_addr / 4096)) \
   of=/tmp/vdso.so 2>/dev/null
objdump -d /tmp/vdso.so
```

### 9.2 vDSO Implementation

```c
// arch/x86/vdso/vgettimeofday.c
notrace int __cvdso_clock_gettime(clockid_t clock, struct __kernel_timespec *ts)
{
    // vdso_data is a pointer to a read-only page shared with the kernel
    // Kernel updates it via seqlock in timekeeping_update()
    const struct vdso_data *vd = __arch_get_vdso_data();
    
    return do_hres(vd, clock, ts);  // reads from shared page, no syscall!
}
```

The shared page (`vdso_data`) contains:
- Clock values
- Sequence number (for seqlock reads — detects if kernel updated mid-read)
- Clock parameters (multiplier, shift for TSC → nanoseconds conversion)

---

## 10. 32-bit Compatibility Syscalls

On a 64-bit kernel, 32-bit programs work via a separate syscall path:

```
32-bit process uses:
    int $0x80    (legacy) → entry_INT80_compat
    sysenter     (fast)   → entry_SYSENTER_compat
    
Both → __do_fast_syscall_32() → ia32_sys_call_table[]
```

The 32-bit syscall table (`ia32_sys_call_table`) maps legacy 32-bit numbers to compatibility wrapper functions that convert 32-bit types (u32 pointers, etc.) to 64-bit.

```c
// arch/x86/entry/syscall_32.c
asmlinkage const sys_call_ptr_t ia32_sys_call_table[] = {
    [0]  = __ia32_sys_restart_syscall,
    [1]  = __ia32_sys_exit,
    [2]  = __ia32_sys_fork,
    [3]  = __ia32_sys_read,
    [4]  = __ia32_sys_write,
    // ...
};
```

---

## 11. seccomp — Syscall Filtering

`seccomp` (Secure Computing) allows a process to restrict which syscalls it can make:

```c
// Mode 1: Only allow read, write, exit, sigreturn
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);

// Mode 2: BPF filter (much more flexible)
struct sock_filter filter[] = {
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};
struct sock_fprog prog = { .len = ARRAY_SIZE(filter), .filter = filter };
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

In the kernel, seccomp is checked in `do_syscall_64()` via `syscall_enter_from_user_mode()` → `__secure_computing()`:

```c
// kernel/seccomp.c
int __secure_computing(const struct seccomp_data *sd)
{
    // Run the BPF filter program
    u32 action = bpf_prog_run(sfilter->prog, (void *)sd);
    
    switch (action & SECCOMP_RET_ACTION) {
    case SECCOMP_RET_ALLOW:   return 0;        // Allow the syscall
    case SECCOMP_RET_KILL:    do_exit(SIGSYS); // Kill the process
    case SECCOMP_RET_ERRNO:   return -EACCES;  // Return error
    case SECCOMP_RET_TRACE:   notify_tracer(); // Notify ptrace
    case SECCOMP_RET_TRAP:    force_sig_info(SIGSYS); break;
    }
}
```

Docker, Chrome sandbox, systemd services all use seccomp to limit attack surface.

---

## 12. ptrace and Syscall Tracing

`ptrace()` is the mechanism behind `strace`, `gdb`, and `ltrace`. It lets a tracer process intercept every syscall of the tracee.

```c
// Simplified ptrace flow:
// 1. Tracer attaches:
ptrace(PTRACE_ATTACH, pid, 0, 0);

// 2. Wait for tracee to stop at each syscall:
ptrace(PTRACE_SYSCALL, pid, 0, 0);  // Run until next syscall
waitpid(pid, &status, 0);

// 3. Read syscall number from registers:
struct user_regs_struct regs;
ptrace(PTRACE_GETREGS, pid, 0, &regs);
printf("syscall: %lld\n", regs.orig_rax);

// 4. Allow syscall to execute:
ptrace(PTRACE_SYSCALL, pid, 0, 0);
waitpid(pid, &status, 0);

// 5. Read return value:
ptrace(PTRACE_GETREGS, pid, 0, &regs);
printf("returned: %lld\n", regs.rax);
```

In the kernel, ptrace hooks are called in `syscall_enter_from_user_mode()` and `syscall_exit_to_user_mode()` via `tracehook_report_syscall_entry()` / `tracehook_report_syscall_exit()`.

---

## 13. Syscall Overhead Analysis

```bash
# Measure raw syscall overhead:
# Using getpid() (minimal work, good benchmark)
perf stat -e 'syscalls:sys_enter_getpid' ./getpid_benchmark

# On a modern x86_64:
# - Non-KPTI: ~100ns per syscall
# - With KPTI: ~200-400ns per syscall (TLB flush cost)
# - With vDSO: ~10ns (no trap at all)
```

The overhead breakdown:
- `SYSCALL` instruction: ~20 cycles
- `swapgs`: ~10 cycles
- KPTI CR3 switch + TLB flush: ~100-300 cycles (worst case)
- Kernel stack switch: ~5 cycles
- `do_syscall_64()` dispatch: ~20 cycles
- Return path: similar

This is why `io_uring` batches I/O operations — instead of 1 syscall per operation, you submit a batch with 1 syscall. This is also why high-frequency trading systems use kernel bypass (DPDK, RDMA) to eliminate syscalls entirely for network I/O.

---

## 14. Adding a New System Call (Complete Example)

**Note:** Adding syscalls to the upstream kernel is discouraged (use netlink, ioctl, or proc/sys instead). But for learning:

### Step 1: Add to syscall table

```bash
# arch/x86/entry/syscalls/syscall_64.tbl
# Add at end (find next available number, e.g., 463):
463    common  mysyscall              sys_mysyscall
```

### Step 2: Declare in header

```c
// include/linux/syscalls.h — add near similar syscalls:
asmlinkage long sys_mysyscall(int value, char __user *buf, size_t len);
```

### Step 3: Implement

```c
// kernel/mysyscall.c (new file)
#include <linux/syscalls.h>
#include <linux/uaccess.h>
#include <linux/kernel.h>

SYSCALL_DEFINE3(mysyscall, int, value, char __user *, buf, size_t, len)
{
    char kbuf[256];
    int ret;
    
    // Validate arguments
    if (len > sizeof(kbuf))
        return -EINVAL;
    
    if (value < 0 || value > 100)
        return -EINVAL;
    
    // Safely copy from user space
    if (copy_from_user(kbuf, buf, len))
        return -EFAULT;
    
    // Null-terminate for safety
    kbuf[len - 1] = '\0';
    
    pr_info("mysyscall: value=%d, buf=%s\n", value, kbuf);
    
    // Return something meaningful
    return value * 2;
}
```

### Step 4: Add to Makefile

```makefile
# kernel/Makefile — add:
obj-y += mysyscall.o
```

### Step 5: Test from user space

```c
// test_mysyscall.c
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>
#include <string.h>

#define __NR_mysyscall 463

int main(void)
{
    char buf[] = "hello kernel";
    long ret = syscall(__NR_mysyscall, 21, buf, strlen(buf) + 1);
    printf("returned: %ld\n", ret);  // Should print 42
    return 0;
}
```

---

## 15. Syscall Auditing and Tracing Tools

```bash
# strace: trace all syscalls of a process
strace ls /tmp
strace -p 1234              # attach to existing process
strace -e trace=read,write ls  # filter syscalls
strace -c ls                # count and summarize

# strace output format:
# open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
# fstat(3, {st_mode=S_IFREG|0644, st_size=12345, ...}) = 0
# mmap(NULL, 12345, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f...
# close(3) = 0

# ftrace: kernel-side syscall tracing
echo 'sys_write' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace

# perf: count syscalls system-wide
sudo perf stat -e 'syscalls:sys_enter_*' sleep 5

# bpftrace: custom syscall tracing
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_write { 
    printf("pid=%d fd=%d count=%d\n", pid, args->fd, args->count); 
}'
```

---

## 16. Slow vs Fast Paths

Some syscalls are in the "fast path" — the common case is optimized:

```c
// fs/read_write.c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
    struct fd f = fdget_pos(fd);  // Fast: looks up file in current->files->fdt->fd[]
    
    if (!f.file)
        return -EBADF;
    
    // ...
}
```

`fdget_pos()` is a very fast path — it's just an array lookup in `files_struct::fdt::fd[fd]`. No locks needed for normal cases due to RCU.

The **slow path** involves things like:
- Checking rlimits
- Security hooks (`security_file_permission()`)
- File locking (mandatory locking, POSIX locks)
- Mandatory Access Control (SELinux, AppArmor)
- Audit logging

---

## 17. Mental Model Checkpoint

After Day 4, you should be able to:

1. Explain exactly what the `SYSCALL` instruction does to CPU state.
2. What are the MSRs involved in syscall dispatch?
3. Draw the full call chain from `write(1, buf, n)` in user space to the kernel function.
4. What is `pt_regs` and why does the kernel build it on the stack?
5. Why is `__user` annotation important? What happens if you skip it?
6. What does `copy_from_user` do that a raw dereference doesn't?
7. What is the vDSO? Name two syscalls it optimizes.
8. What is seccomp and how does the kernel check it on every syscall?
9. What is KPTI's performance impact on syscalls?
10. What does `SYSCALL_DEFINE3` expand to and why is there a multi-layer wrapper?

---

## Key Source Files

```bash
arch/x86/entry/entry_64.S      # Assembly syscall entry (MUST READ)
arch/x86/entry/common.c        # do_syscall_64()
arch/x86/entry/syscalls/syscall_64.tbl  # Syscall number→name table
include/linux/syscalls.h       # SYSCALL_DEFINE* macros
arch/x86/include/asm/syscall.h # Syscall ABI helpers
kernel/seccomp.c               # seccomp implementation
arch/x86/vdso/                 # vDSO implementation
lib/usercopy.c                 # copy_from/to_user implementation
```

---

## Summary

System calls are the fundamental boundary crossing between user and kernel space. The mechanism on x86_64:

1. Registers carry the syscall number and arguments
2. `SYSCALL` instruction atomically switches privilege level and jumps to `entry_SYSCALL_64`
3. The entry code saves all registers as `pt_regs`, switches stacks, and switches page tables (KPTI)
4. `do_syscall_64()` dispatches to the right handler via `sys_call_table[nr]`
5. The handler validates arguments, uses `copy_from_user()` to safely access user memory, does the work, and returns
6. The return path restores everything and uses `SYSRET` to go back to user mode

The key invariant: **the kernel never trusts anything from user space**. Pointer validation, bounds checking, and type-safe argument extraction via `SYSCALL_DEFINE` macros are all mechanisms enforcing this.

Tomorrow: syscall tracing with `strace`, `bpftrace`, and `ftrace`, plus writing your first complete kernel module that adds a syscall.
