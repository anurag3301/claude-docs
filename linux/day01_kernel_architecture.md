# Day 1 — Linux Kernel Architecture Overview

> **Estimated read time:** 90–120 minutes  
> **Kernel version referenced:** 6.x  
> **Goal:** Build a mental model of the entire system before diving into individual subsystems.

---

## 1. What a Kernel Actually Is

Most people learn that "the kernel manages hardware." That's true but useless. Let's be precise.

A kernel is a privileged program that runs continuously from boot until shutdown. It owns the CPU in the most fundamental sense — it decides which code runs, when, and with what access to physical resources. Everything else on the system — your shell, your database, your browser — runs only because the kernel allows it to, under rules the kernel enforces.

The Linux kernel specifically is:

- **Monolithic** — all core services (scheduler, memory manager, filesystems, network stack, drivers) run in a single address space with kernel privileges. They are compiled into one binary (`vmlinux`).
- **Modular** — despite being monolithic, it supports loadable kernel modules (LKMs) that can be inserted/removed at runtime without a reboot.
- **Preemptible** — (with `CONFIG_PREEMPT`) the kernel itself can be interrupted mid-execution to run something more urgent.
- **SMP-aware** — the kernel is designed from the ground up for multiple CPUs, with per-CPU data structures and fine-grained locking.
- **Portable** — the same kernel source tree targets x86_64, ARM64, RISC-V, MIPS, PowerPC, s390, and more. Architecture-specific code lives in `arch/`.

---

## 2. Kernel vs Microkernel — Why It Matters

The microkernel design (e.g., Mach, L4, QNX) runs only a tiny scheduling/IPC core in privileged mode, with filesystem, drivers, and network running as user-space servers. Linux chose monolithic for performance: syscalls into drivers don't require IPC through a message-passing layer. The cost is that a buggy driver crashes the kernel.

Hybrid kernels (Windows NT, macOS XNU) put some services in privileged mode and others as user-space components.

**Why you need to understand this:** Kernel engineering interviews frequently start here. Be ready to articulate the performance/reliability tradeoffs, not just name the categories.

---

## 3. Hardware Privilege Rings

x86 defines four privilege rings (0–3). Linux uses only two:

```
Ring 0 (CPL=0) — Kernel mode
  - Direct hardware access
  - All CPU instructions available (including privileged ones: HLT, LGDT, LIDT, IN/OUT...)
  - Runs: kernel code, interrupt handlers, device drivers

Ring 3 (CPL=3) — User mode
  - No direct hardware access
  - Cannot execute privileged instructions (raises #GP fault)
  - Runs: all user applications
```

ARM64 uses "Exception Levels" (EL0 = user, EL1 = kernel, EL2 = hypervisor, EL3 = secure firmware). RISC-V uses "privilege levels" (U, S, M modes).

The **protection mechanism** works because the CPU checks the Current Privilege Level (CPL) stored in the `CS` register's bottom two bits before executing any sensitive instruction. If a Ring 3 process tries `HLT`, the CPU raises a General Protection Fault (`#GP`) and the kernel's exception handler kills the process.

### Memory Separation

Beyond instruction access, Ring 0/3 enforce **memory separation** via page table permissions. Page table entries (PTEs) have a User/Supervisor bit:

- Supervisor pages (U/S=0): only Ring 0 can read/write
- User pages (U/S=1): Ring 3 can access, but only if the page is mapped in the process's page table

Modern mitigations complicate this:
- **SMAP** (Supervisor Mode Access Prevention): prevents kernel from *accidentally* reading user memory without explicitly using `copy_from_user()` / `copy_to_user()`.
- **SMEP** (Supervisor Mode Execution Prevention): prevents kernel from executing code at user-space addresses (stops ret2usr attacks).
- **KPTI** (Kernel Page Table Isolation): each process has *two* page table sets — a minimal one for user mode (no kernel mappings) and a full one for kernel mode. Mitigates Meltdown.

---

## 4. The Big Picture: Subsystems

The kernel is not a single program — it's a collection of cooperating subsystems, each with its own maintainer(s), mailing list, and coding patterns. Here's the map:

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                               │
│  (glibc, systemd, bash, nginx, python, ...)                     │
└──────────────────────┬──────────────────────────────────────────┘
                       │  System Call Interface
┌──────────────────────▼──────────────────────────────────────────┐
│                      Linux Kernel                               │
│                                                                 │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────────────────┐  │
│  │   Process   │  │    Memory     │  │     VFS / Storage    │  │
│  │  Management │  │  Management   │  │  (ext4, btrfs, NFS)  │  │
│  │  Scheduler  │  │  (VMM, MM)    │  │  Block layer, bio    │  │
│  └─────────────┘  └───────────────┘  └──────────────────────┘  │
│                                                                 │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────────────────┐  │
│  │  Network    │  │    Security   │  │     IPC / Signals    │  │
│  │  Stack      │  │  (LSM, caps,  │  │  (pipes, shmem,      │  │
│  │  (TCP/IP)   │  │   SELinux)    │  │   futexes, signals)  │  │
│  └─────────────┘  └───────────────┘  └──────────────────────┘  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Device Driver Framework                    │    │
│  │   (char, block, network, USB, PCIe, DMA, GPIO, ...)    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Architecture-Specific Code (arch/)            │    │
│  │   (x86_64, arm64, riscv — context switch, page tables)  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                       Hardware                                  │
│  (CPU, RAM, NIC, storage, GPU, USB controllers, ...)            │
└─────────────────────────────────────────────────────────────────┘
```

Let's define each subsystem clearly.

### 4.1 Process Management
- Creates, schedules, and destroys processes and threads
- Key structures: `task_struct` (process descriptor), `mm_struct` (memory context), `files_struct` (open files)
- Key files: `kernel/sched/`, `kernel/fork.c`, `kernel/exit.c`
- The scheduler (CFS, real-time, deadline) lives in `kernel/sched/`

### 4.2 Memory Management (MM)
- Manages physical RAM, virtual address spaces, the page cache
- Key structures: `struct page`, `vm_area_struct`, `mm_struct`, `address_space`
- Implements: demand paging, Copy-on-Write, swap, OOM killing, huge pages
- Key files: `mm/`

### 4.3 VFS & Storage
- Virtual File System: a uniform API layer so user code doesn't care if it's reading from ext4, btrfs, proc, or a socket
- Key structures: `super_block`, `inode`, `dentry`, `file`
- Key files: `fs/`

### 4.4 Network Stack
- Full TCP/IP implementation inside the kernel
- Key structures: `sk_buff` (packet buffer), `socket`, `sock`, `net_device`
- Key files: `net/`

### 4.5 Device Driver Framework
- Abstracts hardware behind a consistent model
- `kobject`/`kset` form the device model backbone; `sysfs` exposes it to userspace
- Key files: `drivers/`

### 4.6 Security
- Linux Security Modules (LSM) provide hooks throughout the kernel
- SELinux, AppArmor, Smack all plug in via LSM
- Capabilities replace the old root/non-root binary
- Key files: `security/`

### 4.7 Arch-Specific Code
- Context switch, CPU initialization, page table formats, interrupt routing
- Key files: `arch/x86/`, `arch/arm64/`, etc.

---

## 5. Kernel Space vs User Space — The Deep Version

The split isn't just about privilege — it's about **separate address spaces** (mostly).

On a 64-bit x86 system with a 4-level page table:

```
Virtual Address Space (48-bit = 256 TB total)
────────────────────────────────────────────────
0x0000000000000000  ┐
        ...         │  User space (128 TB)
0x00007FFFFFFFFFFF  ┘  (process-specific mappings)

    [non-canonical gap — any access here is instant #GP]

0xFFFF800000000000  ┐
        ...         │  Kernel space (128 TB)
0xFFFFFFFFFFFFFFFF  ┘  (same for ALL processes)
```

**Critical insight:** Kernel virtual addresses are the *same* in every process's page table (pre-KPTI). When you do a context switch, the kernel address mappings don't change — only the user-space portion of the page table switches. This is why the kernel can always execute without reloading its own page tables, and why KPTI (which breaks this) has a performance cost.

After KPTI, each process has two page table roots (CR3 values):
1. **User-mode CR3**: maps only user pages + tiny kernel trampoline for syscall entry
2. **Kernel-mode CR3**: maps everything (kernel + current process's user pages)

Switching CR3 flushes the TLB (partially mitigated by PCID — Process Context IDentifiers).

---

## 6. How User Space Talks to the Kernel

There are exactly four ways:

### 6.1 System Calls (Primary Path)
Process executes `SYSCALL` instruction (x86_64) → CPU switches to Ring 0 → kernel dispatches via syscall table → returns via `SYSRET`.

```c
// User space (what the C library does under the hood):
// write(1, "hello\n", 6);
// compiles to (conceptually):
mov rax, 1        // syscall number for write
mov rdi, 1        // fd
mov rsi, buf      // buffer pointer
mov rdx, 6        // length
syscall           // trap into kernel
```

### 6.2 Signals (Kernel → Process)
Kernel delivers signals asynchronously. Process's execution is interrupted at a safe point, saved state, runs signal handler, restores state.

### 6.3 `/proc`, `/sys`, `/dev` (Virtual Filesystems)
These are kernel data structures exposed as files. Reading `/proc/meminfo` calls kernel functions that format memory stats as text. Writing to `/sys/class/net/eth0/mtu` calls a driver function to set the MTU.

### 6.4 Memory-Mapped Files + vDSO
`mmap()` lets the kernel map physical memory (or files) into a process's virtual space. The vDSO (virtual Dynamic Shared Object) is a small kernel-provided shared library mapped into every process that implements a handful of syscalls (`gettimeofday`, `clock_gettime`) entirely in user space using shared memory — zero trap overhead.

---

## 7. The Kernel's Execution Contexts

Understanding *where* kernel code runs is essential for writing correct kernel code (especially around locking).

### 7.1 Process Context
Code executing on behalf of a specific process (running a syscall, for example). You can:
- Sleep / block
- Access `current` (pointer to current `task_struct`)
- Be preempted (with preemptible kernel)
- Hold mutexes

### 7.2 Interrupt Context (Hardirq)
Code running inside a hardware interrupt handler. You **cannot**:
- Sleep
- Access user memory
- Take sleeping locks (mutexes, semaphores)

You **can** use spinlocks, atomic operations.

`in_interrupt()` returns true. The CPU is using the interrupt stack, not the process stack.

### 7.3 Softirq Context
Code running as a "software interrupt" (deferred from hardirq). Same rules as hardirq — no sleeping. Runs on the same CPU that raised the softirq. Examples: network packet processing (`NET_RX_SOFTIRQ`), timer callbacks.

### 7.4 Tasklet Context
A special case of softirq — they're serialized (only one instance runs at a time per tasklet). Also cannot sleep.

### 7.5 Workqueue Context
Deferred work that *can* sleep — runs in kernel thread context (`kworker/0:0`, etc.). Bridges the gap: if a driver needs to do something in interrupt context but needs to sleep, it defers to a workqueue.

```
Interrupt arrives
    → Hardirq handler (fast, cannot sleep)
        → Schedule softirq / tasklet (cannot sleep)
            → Queue work item (can sleep)
                → kworker thread picks it up
```

---

## 8. The `current` Macro

Throughout kernel code you'll see `current`. This is a macro that returns a pointer to the `task_struct` of the currently running process.

On x86_64, `current` is implemented by storing the `task_struct` pointer at a fixed offset from the per-CPU stack base, or in a per-CPU variable. The exact implementation is in `arch/x86/include/asm/current.h`:

```c
// Simplified — actual uses per-CPU storage
static __always_inline struct task_struct *get_current(void)
{
    return this_cpu_read_stable(pcpu_hot.current_task);
}
#define current get_current()
```

You'll use `current->pid`, `current->comm`, `current->mm`, `current->files`, etc. constantly.

---

## 9. Kernel Versioning

Linux version numbers: `MAJOR.MINOR.PATCH` (e.g., `6.9.3`)

- **MAJOR**: Historically 2 and 3 lasted decades. Linus bumped to 4.0 and later 5.0 for size/symbolism. Not a break point.
- **MINOR**: Released every ~10 weeks. Each cycle: 2 weeks merge window, then rc1 through rc7/rc8, then release.
- **PATCH/STABLE**: Bug and security fixes backported by the stable team (`kernel.org/pub/linux/kernel/v6.x/`).
- **LTS (Long Term Support)**: Select kernels maintained for 2–6 years (e.g., 5.15, 6.1, 6.6).

The `-rc` kernels (release candidates) are what kernel developers work against during the merge window.

Distribution kernels (RHEL, Ubuntu, Debian) take an upstream kernel and apply hundreds of out-of-tree patches — different from what you'd find at kernel.org.

---

## 10. How the Kernel Is Organized in Source

```
linux/
├── arch/           # Architecture-specific (x86, arm64, riscv...)
│   └── x86/
│       ├── kernel/ # CPU init, interrupts, context switch
│       ├── mm/     # x86 page table management
│       └── include/asm/
├── block/          # Block layer (bio, request_queue)
├── crypto/         # Crypto API
├── drivers/        # All device drivers (hundreds of subdirs)
│   ├── net/        # Network drivers
│   ├── nvme/       # NVMe storage
│   ├── gpu/drm/    # GPU/display
│   └── ...
├── fs/             # Filesystems (ext4, btrfs, nfs, proc, sysfs...)
├── include/        # Kernel headers (linux/, asm-generic/)
├── init/           # Kernel startup (main.c = start_kernel)
├── ipc/            # IPC (SysV shm, msg queues, semaphores)
├── kernel/         # Core kernel (sched/, fork.c, signal.c...)
├── lib/            # Kernel helper library (sort, rbtree, idr...)
├── mm/             # Memory management
├── net/            # Network stack (ipv4, ipv6, core, sched...)
├── security/       # LSM framework, SELinux, AppArmor
├── sound/          # ALSA audio subsystem
├── tools/          # Userspace tools (perf, bpf, testing)
└── virt/           # Virtualization (KVM lives in arch/ + virt/)
```

---

## 11. Key Kernel Concepts to Always Have in Mind

### 11.1 Everything Is Concurrent
The kernel runs on multiple CPUs simultaneously. Any data structure accessed from multiple CPUs must be protected — by locks, RCU, per-CPU variables, or atomic operations. There is no "global lock" — the Big Kernel Lock (BKL) was fully removed in kernel 3.0.

### 11.2 No Memory Protection for Kernel Code
A buffer overflow in kernel code corrupts kernel memory. There's no OS to catch you — **you are the OS**. KASAN (Kernel Address SANitizer) and KFENCE are compile-time/runtime tools that add some protection in debug builds, but production kernels run without guard rails.

### 11.3 Stack Is Tiny
The kernel stack per thread is only **8KB** (or 16KB on some configs) — not the megabytes user programs get. Never recurse deeply, never put large arrays on the kernel stack.

### 11.4 No libc
Kernel code cannot call `malloc()`, `printf()`, `pthread_mutex_lock()`, or any standard library function. It has its own:
- `kmalloc()` / `kfree()` instead of malloc/free
- `printk()` instead of printf
- `struct mutex`, `spinlock_t` instead of pthreads
- `copy_from_user()` / `copy_to_user()` to safely read/write user memory

### 11.5 Sleeping Is Context-Dependent
In interrupt context, sleeping is illegal — it deadlocks. In process context with a spinlock held, sleeping is illegal — it deadlocks (another CPU waiting on the spinlock can never release it). Always know your context before calling anything that might block.

### 11.6 CONFIG_ Controls Everything
The kernel is highly configurable. `CONFIG_PREEMPT`, `CONFIG_HZ`, `CONFIG_NR_CPUS`, `CONFIG_NUMA`, `CONFIG_DEBUG_LOCK_ALLOC` — thousands of compile-time options control what code gets included and how it behaves. When reading kernel source, always check what CONFIG option enables a given code path.

---

## 12. Monolithic Does Not Mean Unstructured

Despite being one binary, the kernel has clear layering enforced by convention (not always the compiler):

```
syscall interface
    │
    ▼
subsystem API (e.g., VFS operations)
    │
    ▼
generic implementation (e.g., generic_file_read_iter)
    │
    ▼
filesystem-specific ops (e.g., ext4_file_read_iter)
    │
    ▼
block layer (bio submission)
    │
    ▼
driver layer (NVMe, SATA...)
    │
    ▼
hardware
```

Each layer calls down through well-defined function pointer tables (ops structs). For example, `file->f_op->read_iter` is a function pointer set by whatever filesystem opened the file. This is how the kernel achieves polymorphism in C.

---

## 13. The Linux Kernel Community Model

Understanding this helps you as an engineer working with or contributing to the kernel.

- **Linus Torvalds** maintains the main tree (`torvalds/linux` on GitHub mirroring kernel.org)
- **Subsystem maintainers** maintain sub-trees (e.g., David Miller/Jakub Kicinski for networking, Andrew Morton for MM)
- **Changes go through mailing lists**, not GitHub PRs — `linux-kernel@vger.kernel.org` and subsystem-specific lists
- **Patch format**: `git format-patch` + `git send-email`, following a strict format
- **Review process**: patches are reviewed on the mailing list, `Reviewed-by:` / `Acked-by:` tags added
- **Merge window**: 2 weeks after each release where maintainers send pull requests to Linus
- **Stable fixes**: bug fixes backported to stable kernels by the stable team

Companies like Red Hat, Canonical, Google, Intel, and Meta all have engineers whose primary job is upstream kernel work.

---

## 14. What Makes Linux Different from Other Kernels

| Feature | Linux | Windows NT | macOS XNU | BSD |
|---------|-------|------------|-----------|-----|
| Architecture | Monolithic + modules | Hybrid | Hybrid (Mach + BSD) | Monolithic |
| Open source | Yes (GPLv2) | No | Partially (Darwin) | Yes (BSD license) |
| Real-time support | PREEMPT_RT patch | Limited | Limited | Limited |
| eBPF | Yes (powerful) | Limited | No | No |
| Module loading | Yes | Yes (signed only) | Yes (signed only) | Yes |
| Namespaces/cgroups | Yes (container substrate) | Limited (WSL2 uses Linux) | No | Partial (jails) |

---

## 15. Mental Model Checkpoint

Before moving to Day 2, you should be able to answer:

1. Why does Linux use a monolithic kernel? What's the tradeoff vs microkernel?
2. What is Ring 0 vs Ring 3 on x86? What prevents a Ring 3 process from accessing Ring 0 instructions?
3. What are SMEP, SMAP, and KPTI? What attacks do they mitigate?
4. Name the four ways user space communicates with the kernel.
5. What is "process context" vs "interrupt context"? Why does it matter for locking?
6. Why is `current` always valid in process context but meaningless in hardirq context?
7. What does KPTI do to the TLB? Why does it hurt performance?
8. The kernel stack is 8KB. Why is this a problem and how is it handled?

---

## Key Source Files to Read

```bash
# After you have a kernel source tree:
init/main.c              # start_kernel() — the kernel's main()
include/linux/sched.h    # task_struct definition
include/linux/mm_types.h # mm_struct, vm_area_struct, page
include/linux/fs.h       # file, inode, dentry, super_block
arch/x86/entry/entry_64.S # syscall entry point in assembly
kernel/fork.c            # copy_process(), the core of fork()
```

---

## Summary

The Linux kernel is a monolithic, modular, SMP-aware privileged program that:
- Enforces user/kernel separation via CPU privilege rings and page table permissions
- Exposes services to user space exclusively through system calls (plus signals, virtual filesystems, and shared memory for special cases)
- Is organized into cooperating subsystems (MM, scheduler, VFS, network, drivers, security) each with clear internal APIs
- Runs in multiple execution contexts (process, hardirq, softirq, workqueue) with different rules about what you can do
- Is configured at compile time via thousands of `CONFIG_*` options
- Has no memory protection for its own code — discipline and code review are the safety net

Tomorrow: we go into the source tree, understand Kconfig and Kbuild, and compile our first custom kernel.
