# Day 1 — Linux Kernel Architecture & Build System
**Environment:** 💻 PC / QEMU
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [What is the Linux Kernel?](#1-what-is-the-linux-kernel)
2. [Kernel Space vs User Space](#2-kernel-space-vs-user-space)
3. [System Calls — The Bridge](#3-system-calls--the-bridge)
4. [Major Kernel Subsystems](#4-major-kernel-subsystems)
5. [Where Do Device Drivers Live?](#5-where-do-device-drivers-live)
6. [Kernel Source Tree Layout](#6-kernel-source-tree-layout)
7. [Kernel Configuration System (Kconfig)](#7-kernel-configuration-system-kconfig)
8. [The Kernel Build System (Kbuild)](#8-the-kernel-build-system-kbuild)
9. [Hands-On Lab — Build & Boot in QEMU](#9-hands-on-lab--build--boot-in-qemu)
10. [Interview Questions](#10-interview-questions)

---

## 1. What is the Linux Kernel?

The Linux kernel is the core of the operating system. It is the single piece of software that has full, unrestricted control over all hardware on a machine. Every other program — your shell, your web browser, your database — runs at the pleasure of the kernel, using resources the kernel manages and protects.

The kernel is responsible for:

- **Process management** — creating, scheduling, and destroying processes and threads
- **Memory management** — virtual address spaces, page tables, physical memory allocation
- **Device management** — drivers that talk to hardware peripherals
- **File system management** — VFS abstraction over all storage backends
- **Networking** — socket layer, protocol stacks (TCP/IP), NIC drivers
- **Security** — access control, namespaces, cgroups, LSM hooks
- **Inter-process communication** — pipes, signals, shared memory, sockets

The Linux kernel is **monolithic** — all subsystems run in the same address space with the same privilege level. This is different from a **microkernel** (like L4 or QNX) where drivers run in user space and communicate via IPC. Monolithic kernels are faster because subsystems call each other directly via function pointers, not message passing. The trade-off is that a bad driver can corrupt the entire kernel.

Despite being monolithic, Linux supports **Loadable Kernel Modules (LKMs)** — compiled code that can be dynamically inserted into and removed from the running kernel without rebooting. This is how most drivers are shipped — as `.ko` (kernel object) files loaded with `insmod` or `modprobe`.

### Kernel Versions

```
v6.x.y
 │   │  └── patch level (bug fixes only)
 │   └────── minor version
 └────────── major version (rarely changes)
```

Since kernel 3.0, version numbers are not strictly semantic. Linus releases a new major.minor every ~10 weeks. Use `uname -r` to see your current kernel version.

---

## 2. Kernel Space vs User Space

This is the most fundamental concept in kernel development. Modern CPUs implement hardware privilege levels (called **rings** on x86, **exception levels** on ARM).

```
┌─────────────────────────────────────────────┐
│               User Space (Ring 3 / EL0)     │
│  Applications, libraries (glibc, etc.)      │
│  • Limited CPU instructions                 │
│  • Cannot access hardware directly          │
│  • Cannot access other process memory       │
│  • Virtual addresses only                   │
├─────────────────────────────────────────────┤
│         ↕  System Call Interface (trap)     │
├─────────────────────────────────────────────┤
│             Kernel Space (Ring 0 / EL1)     │
│  Kernel, drivers, kernel modules            │
│  • All CPU instructions available           │
│  • Direct hardware access (MMIO, I/O ports) │
│  • Access all physical memory               │
│  • Manage virtual memory of all processes   │
└─────────────────────────────────────────────┘
```

### Why Does This Separation Exist?

Without privilege separation, any program could:
- Read another program's memory (passwords, keys)
- Write directly to disk bypassing the filesystem
- Crash the entire machine with a bad pointer

The CPU enforces this separation in hardware. If a user-space program tries to execute a privileged instruction, the CPU raises a **fault** (General Protection Fault on x86, Data Abort on ARM), and the kernel terminates that process.

### What Happens on a Context Switch?

When a user-space program makes a system call:

1. CPU executes `syscall` (x86-64) or `svc` (ARM) instruction
2. CPU atomically switches to kernel privilege level and jumps to the syscall entry point
3. CPU saves user-space registers to the kernel stack
4. Kernel identifies the syscall number from `rax` (x86) or `x8` (ARM64)
5. Kernel dispatches to the appropriate handler (e.g., `sys_read`)
6. Handler executes, result placed in `rax`/`x0`
7. Kernel restores user registers and executes `sysret`/`eret`
8. CPU drops back to user privilege level, program continues

This entire round trip costs typically 100–300 nanoseconds on modern hardware. That is why minimizing syscalls matters for performance-critical code.

### Kernel Stack vs User Stack

Every process has **two stacks**:
- **User stack** — in user virtual address space, grows dynamically, default 8MB limit
- **Kernel stack** — small (4KB or 8KB depending on config), used when running kernel code on behalf of that process

The kernel stack being so small is critically important for driver developers. **Never allocate large arrays on the kernel stack.** Use `kmalloc()` instead.

---

## 3. System Calls — The Bridge

System calls are the API through which user space requests services from the kernel. Linux on x86-64 has ~350 system calls. Common ones:

| Syscall | Number (x86-64) | Purpose |
|---------|----------------|---------|
| `read`  | 0 | Read from file descriptor |
| `write` | 1 | Write to file descriptor |
| `open`  | 2 | Open a file/device |
| `close` | 3 | Close a file descriptor |
| `mmap`  | 9 | Map memory or device |
| `ioctl` | 16 | Device-specific control |
| `fork`  | 57 | Create a new process |
| `execve`| 59 | Execute a program |

You can see all syscalls a program makes with `strace`:

```bash
strace ls /tmp
```

### The VFS Layer

Device drivers are not accessed directly through named syscalls — they're accessed through the **Virtual File System (VFS)**. In Linux, almost everything is a file. When you `open("/dev/ttyS0", ...)`, you get back a file descriptor. `read()`, `write()`, `ioctl()`, `mmap()` all go through VFS which routes them to the appropriate driver's `file_operations` function pointers.

```
User: open("/dev/mydevice", O_RDWR)
       │
       ▼
   VFS layer
       │  looks up inode → finds it's a char device
       │  finds driver registered for this major/minor
       ▼
   Driver's file_operations.open()
```

This is why writing a character device driver means implementing `file_operations` callbacks — you're plugging into the VFS.

---

## 4. Major Kernel Subsystems

Understanding the map of the kernel helps you know where to look when reading source or debugging.

```
┌──────────────────────────────────────────────────────────────┐
│                    System Call Interface                      │
├──────────┬──────────┬──────────┬───────────┬────────────────┤
│ Process  │  Memory  │   VFS    │  Network  │   Security     │
│ Scheduler│  Manager │  Layer   │   Stack   │   (LSM/SELinux)│
├──────────┴──────────┴────┬─────┴─────┬─────┴────────────────┤
│     Architecture Layer   │           │                       │
│  (arch/arm64, arch/x86)  │  Device   │   Power Management   │
├──────────────────────────┤   Model   │   (PM Core)          │
│  ┌─────────────────────┐ │   (sysfs, │                      │
│  │    Device Drivers   │ │   kobject)│                      │
│  │  ┌──────┬──────────┐│ │           │                      │
│  │  │ Char │  Block   ││ │           │                      │
│  │  │Devs  │  Devs    ││ │           │                      │
│  │  ├──────┼──────────┤│ │           │                      │
│  │  │  Net │  USB     ││ │           │                      │
│  │  │Devs  │  I2C/SPI ││ │           │                      │
│  └──┴──────┴──────────┘│ │           │                      │
├──────────────────────────┴───────────┴──────────────────────┤
│               Hardware Abstraction (CPU, MMU, IRQ, DMA)      │
└──────────────────────────────────────────────────────────────┘
```

### Process Scheduler

The scheduler decides which task runs on which CPU and for how long. Linux uses **CFS (Completely Fair Scheduler)** for normal processes and a real-time scheduler for `SCHED_FIFO`/`SCHED_RR` tasks. Understanding scheduling matters for driver writers because:
- Drivers run in process context (schedulable) or interrupt context (not schedulable)
- You cannot call `sleep()` or any blocking function from interrupt context

### Memory Manager

Manages the mapping between virtual addresses (what programs see) and physical addresses (what RAM is). Key data structures:
- `struct page` — represents one 4KB physical page frame
- `struct mm_struct` — a process's entire virtual memory layout
- `struct vm_area_struct (VMA)` — one contiguous region in a process's virtual address space

### VFS (Virtual File System)

An abstraction layer that provides a uniform interface (`open`, `read`, `write`, `seek`, `mmap`) over all filesystems (ext4, btrfs, tmpfs, procfs, sysfs, devfs). Device drivers plug into VFS through `file_operations`, `inode_operations`, and `address_space_operations`.

---

## 5. Where Do Device Drivers Live?

Linux has three types of drivers:

### Character Drivers
Access devices as a stream of bytes. The simplest driver type.
- Examples: serial ports, keyboards, custom hardware
- Interface: `open`, `read`, `write`, `ioctl`, `mmap`
- Accessed via `/dev/ttyS0`, `/dev/video0`, `/dev/mydevice`

### Block Drivers
Access devices in fixed-size blocks (sectors). Must support random access.
- Examples: hard drives, SSDs, eMMC, RAM disks
- Interface: bio-based request queue, blk-mq
- Accessed via `/dev/sda`, `/dev/mmcblk0`

### Network Drivers
Don't use file descriptors at all — data goes through socket API.
- Examples: Ethernet NICs, Wi-Fi adapters, virtual tap devices
- Interface: `net_device` ops, NAPI polling
- Accessed via `eth0`, `wlan0`, etc.

### The Driver Binding Model

A driver doesn't "claim" a device at compile time. Instead:
1. Kernel discovers hardware (via ACPI, Device Tree, PCI enumeration, USB enumeration)
2. Kernel creates a `struct device` for each discovered hardware
3. Kernel tries to match this device to a registered `struct driver`
4. Matching uses IDs (PCI vendor/device ID, DT compatible string, USB VID/PID)
5. On match, kernel calls the driver's `probe()` function
6. Driver initializes hardware and registers with appropriate subsystem

```
Hardware Discovered        Driver Registered
       │                          │
       ▼                          ▼
  struct device    ←──match──  struct driver
       │
       └── calls driver's probe()
                   │
                   └── driver sets up, registers /dev entry
```

---

## 6. Kernel Source Tree Layout

Clone the kernel and navigate it — you will live here for the next 30 days.

```bash
git clone https://github.com/torvalds/linux.git --depth=1
```

```
linux/
├── arch/           # Architecture-specific code (x86, arm64, riscv, mips...)
│   ├── x86/
│   │   ├── kernel/ # x86 CPU init, interrupts, syscall entry
│   │   └── mm/     # x86 memory management, page tables
│   └── arm64/
├── drivers/        # ALL device drivers live here
│   ├── char/       # Character devices (tty, random, mem)
│   ├── block/      # Block devices
│   ├── net/        # Network drivers
│   ├── i2c/        # I2C bus drivers + client drivers
│   ├── spi/        # SPI bus drivers + client drivers
│   ├── gpio/       # GPIO controllers
│   ├── usb/        # USB host/device drivers
│   ├── pci/        # PCI bus support
│   ├── media/      # V4L2, DVB, camera drivers
│   ├── sound/      # ALSA, ASoC audio drivers
│   ├── input/      # Input devices (keyboard, mouse, touchscreen)
│   ├── dma/        # DMA engine drivers
│   └── power/      # Power management
├── fs/             # Filesystems (ext4, btrfs, proc, sys, dev)
│   ├── ext4/
│   └── proc/
├── include/        # Kernel headers
│   ├── linux/      # Core kernel headers (types.h, kernel.h, module.h...)
│   └── uapi/       # Headers exported to user space
├── kernel/         # Core kernel (scheduler, signals, timers, printk)
│   ├── sched/      # CFS scheduler
│   └── irq/        # Interrupt management
├── mm/             # Memory management (slab, vmalloc, mmap, oom)
├── net/            # Networking core (TCP/IP, sockets, netfilter)
├── init/           # Kernel init code (main.c — kernel entry point!)
├── lib/            # Generic kernel library functions
├── scripts/        # Build scripts, checkpatch.pl, kconfig tools
├── Documentation/  # Kernel documentation (ABI, devicetree bindings)
├── Kconfig         # Top-level Kconfig
└── Makefile        # Top-level Makefile
```

**Key files to bookmark:**
- `init/main.c` — `start_kernel()` is where the kernel begins after boot
- `kernel/sched/core.c` — the scheduler core
- `mm/slub.c` — the slab allocator (kmalloc backend)
- `include/linux/fs.h` — `file_operations`, `inode`, `file` structs
- `include/linux/module.h` — everything for writing kernel modules

### Finding Your Way Around

```bash
# Search for a function definition
grep -rn "struct file_operations" include/linux/fs.h

# Use cscope (much better for large codebases)
make cscope
cscope -d

# Use ctags
make tags
# Then in vim: Ctrl+] to jump to definition

# Elixir cross-reference (web-based, excellent)
# https://elixir.bootlin.com/linux/latest/source
```

---

## 7. Kernel Configuration System (Kconfig)

Before building the kernel, you configure it. The configuration system is called **Kconfig**.

### What is Kconfig?

Every `Kconfig` file in the source tree defines configuration symbols. For example, `drivers/i2c/Kconfig` defines `CONFIG_I2C`. Each symbol can be:
- `y` — built into the kernel image
- `m` — built as a loadable module (.ko file)
- `n` — not compiled at all

The result is a `.config` file in the kernel root — a plain text file with thousands of lines like:

```
CONFIG_I2C=y
CONFIG_I2C_BCM2835=m
CONFIG_SPI=y
# CONFIG_DEBUG_KERNEL is not set
```

### Configuration Interfaces

```bash
# Menu-based TUI (most common)
make menuconfig

# Graphical Qt interface
make xconfig

# Plain text editor friendly
make config

# Start from current running kernel's config
cp /boot/config-$(uname -r) .config
make olddefconfig  # fill in new options with defaults

# For cross-compilation (ARM64 RPi4 baseline)
make ARCH=arm64 defconfig
# Then customize with menuconfig
```

### Important Configuration Sections for Driver Developers

```
General Setup → 
  [*] Enable loadable module support    ← must have for .ko modules
  [*] Kernel .config support            ← embeds .config in kernel
  
Kernel hacking →
  [*] Kernel debugging                  ← enables many debug features
  [*] Magic SysRq key
  [*] Kernel address sanitizer (KASAN)  ← catches memory bugs
  [*] Undefined behavior sanitizer
  -*- Compile the kernel with debug info ← enables gdb debugging

Device Drivers →
  <*> I2C support
  <M> Raspberry Pi SenseHAT joystick
  ...
```

### How CONFIG_ Symbols Affect Compilation

In a driver's `Makefile`:
```makefile
obj-$(CONFIG_MY_DRIVER) += my_driver.o
```
- If `CONFIG_MY_DRIVER=y` → compiled into kernel
- If `CONFIG_MY_DRIVER=m` → compiled as `my_driver.ko`
- If `CONFIG_MY_DRIVER=n` → not compiled

In source code, you can use:
```c
#ifdef CONFIG_MY_DRIVER_DEBUG
    pr_debug("debug info: %d\n", value);
#endif
```

---

## 8. The Kernel Build System (Kbuild)

Kbuild is the kernel's custom build system built on top of GNU Make.

### Building the Kernel

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt install build-essential libncurses-dev bison flex \
    libssl-dev libelf-dev bc dwarves

# Configure
make menuconfig

# Build (use -j$(nproc) for parallel build)
make -j$(nproc)

# This produces:
# arch/x86/boot/bzImage  ← compressed kernel image
# vmlinux                ← uncompressed ELF (for debugging)
# System.map             ← symbol table

# Build modules separately
make modules -j$(nproc)

# Install modules to /lib/modules/$(kernel_version)/
sudo make modules_install

# Install kernel image + update bootloader
sudo make install
```

### Build Artifacts Explained

| File | Description |
|------|-------------|
| `vmlinux` | Uncompressed ELF kernel image. Used for debugging with gdb/crash |
| `arch/x86/boot/bzImage` | Compressed, bootable x86 kernel image |
| `arch/arm64/boot/Image` | Uncompressed ARM64 kernel image |
| `arch/arm64/boot/Image.gz` | Compressed ARM64 kernel image |
| `System.map` | Symbol address table. Used by the kernel to print symbol names in oops |
| `Module.symvers` | Exported kernel symbols and their CRC checksums |
| `*.ko` | Kernel object — a loadable module |

### Cross-Compilation for RPi4

When building on your x86 PC for the ARM64 RPi4:

```bash
# Install ARM64 toolchain
sudo apt install gcc-aarch64-linux-gnu

# Set ARCH and CROSS_COMPILE for every make invocation
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

# Configure for RPi4 (use RPi foundation's config as base)
make bcm2711_defconfig   # RPi4 SoC is BCM2711

# Customize if needed
make menuconfig

# Build
make -j$(nproc) Image modules dtbs

# Output files for RPi4:
# arch/arm64/boot/Image            ← kernel image
# arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb  ← device tree blob
# **/*.ko                          ← kernel modules
```

---

## 9. Hands-On Lab — Build & Boot in QEMU

### Goal
Build a minimal Linux kernel and a minimal root filesystem, then boot it in QEMU on your local PC. No real hardware needed today.

### Step 1 — Install QEMU

```bash
sudo apt install qemu-system-x86-64 qemu-system-arm
```

### Step 2 — Get the Kernel Source

```bash
git clone https://github.com/torvalds/linux.git --depth=1 ~/kernel
cd ~/kernel
```

### Step 3 — Configure for a Tiny x86-64 Kernel

```bash
# Start with a minimal configuration
make x86_64_defconfig

# Enable a few extras we'll need
make menuconfig
# Navigate to: General Setup → [*] Initial RAM filesystem and RAM disk support
# Navigate to: Device Drivers → Character devices → [*] Enable TTY
# Save and exit
```

### Step 4 — Build the Kernel

```bash
make -j$(nproc) 2>&1 | tee build.log
# Takes 5-20 minutes depending on your machine
# Watch for errors in build.log
ls -lh arch/x86/boot/bzImage   # Should see ~10MB file
```

### Step 5 — Build a Minimal Root Filesystem with BusyBox

Without a root filesystem, the kernel boots but panics immediately (Kernel panic - not syncing: VFS: Unable to mount root fs).

```bash
# Get BusyBox — single binary that provides shell + coreutils
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1

# Configure for static linking (no shared libs needed)
make defconfig
make menuconfig
# Settings → [*] Build BusyBox as a static binary (no shared libs)

make -j$(nproc)
make install   # installs to ./_install/
```

### Step 6 — Create the initramfs

```bash
cd ..
mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev,tmp}

# Copy BusyBox and its symlinks
cp -a busybox-1.36.1/_install/* initramfs/

# Create a minimal init script
cat > initramfs/init << 'EOF'
#!/bin/sh

# Mount essential filesystems
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

echo "========================================="
echo " Linux Kernel Driver Course - Day 1 Lab  "
echo " Minimal QEMU Boot - Kernel $(uname -r)  "
echo "========================================="

# Show kernel ring buffer
dmesg | head -30

echo ""
echo "Type 'dmesg | less' to see full boot log"
echo "Type 'cat /proc/modules' to see loaded modules (none yet)"
echo "Type 'cat /proc/cpuinfo' to see CPU info"
echo ""

# Drop to shell
exec /bin/sh
EOF
chmod +x initramfs/init

# Package as cpio archive
cd initramfs
find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz
cd ..
```

### Step 7 — Boot in QEMU

```bash
qemu-system-x86_64 \
    -kernel ~/kernel/arch/x86/boot/bzImage \
    -initrd initramfs.cpio.gz \
    -append "console=ttyS0 nokaslr" \
    -nographic \
    -m 256M \
    -smp 2
# Press Ctrl+A then X to exit QEMU
```

You should see the kernel booting and eventually drop into a shell. Try these commands inside QEMU:

```bash
# Inside QEMU shell
uname -r                    # kernel version
cat /proc/version           # more kernel info
cat /proc/cmdline           # kernel command line
cat /proc/meminfo           # memory stats
cat /proc/cpuinfo           # CPU info
ls /proc/                   # explore proc filesystem
cat /proc/interrupts        # hardware interrupts
dmesg | grep -i error       # look for errors
```

### Step 8 — Explore the Boot Process

```bash
# Still inside QEMU
# See the kernel's view of memory regions
cat /proc/iomem

# See I/O port map
cat /proc/ioports

# See loaded modules (should be empty since we built everything in)
cat /proc/modules

# See kernel symbols (if CONFIG_KALLSYMS=y)
head -20 /proc/kallsyms

# Exit QEMU
exit
# Then: Ctrl+A, X
```

### Step 9 — Explore the Source

Back on your PC, spend 15 minutes tracing the boot path:

```bash
cd ~/kernel

# Where does the kernel start?
grep -n "asmlinkage __visible void __init start_kernel" init/main.c

# Read the first ~100 lines of start_kernel()
# You'll see calls to: setup_arch(), mm_init(), sched_init(), etc.

# Find where device drivers are initialized
grep -n "driver_init\|do_initcalls" init/main.c

# Understand initcalls — how drivers register themselves at boot
grep -rn "module_init\|__initcall" include/linux/init.h | head -20
```

---

## 10. Interview Questions

These are real questions asked at companies like Qualcomm, NVIDIA, Intel, Apple, and automotive companies.

---

**Q1. What is the difference between kernel space and user space? What prevents user space from accessing kernel memory?**

**Answer:** User space and kernel space are privilege domains enforced by the CPU's Memory Management Unit (MMU) and privilege level hardware. On x86-64, user space runs at Ring 3 and kernel space at Ring 0. On ARM64, user space runs at EL0 and kernel space at EL1.

The MMU enforces separation through page table permissions — kernel page table entries are marked with supervisor-only access bits. If a user-space process tries to access a kernel virtual address, the MMU raises a page fault, the CPU jumps to the kernel's fault handler, and the kernel sends SIGSEGV to the offending process.

Additionally, since kernel 4.15, Linux uses **KPTI (Kernel Page Table Isolation)** on x86 — in user mode, the kernel's virtual address mappings are almost entirely absent from the active page table. This was introduced to mitigate the Spectre/Meltdown hardware vulnerabilities.

---

**Q2. What is a system call and what happens at the CPU level when one is invoked?**

**Answer:** A system call is a controlled entry point into the kernel that user-space programs use to request kernel services. The mechanism is hardware-enforced.

On x86-64: user-space places the syscall number in `rax` and arguments in `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9`, then executes the `syscall` instruction. The CPU atomically saves `rip`/`rflags`/`rsp` to per-CPU MSR-defined locations, loads the kernel stack pointer, switches to Ring 0, and jumps to the kernel's syscall entry point (`entry_SYSCALL_64` in `arch/x86/entry/entry_64.S`). The kernel dispatches via `sys_call_table[rax]`. On return, `sysret` restores the CPU state and drops back to Ring 3.

On ARM64: `svc #0` instruction triggers an exception to EL1 via the exception vector table. The syscall number is in `x8`.

---

**Q3. What is the difference between a monolithic kernel and a microkernel? What are the trade-offs?**

**Answer:**
- **Monolithic kernel** (Linux, BSD): All kernel subsystems (memory management, scheduling, drivers, filesystems) run in the same address space at the same privilege level. Communication is direct function calls — extremely fast. A bug in any subsystem can corrupt the entire kernel. Linux mitigates this with loadable modules, but they run in kernel space.
- **Microkernel** (L4, QNX, MINIX): Only a minimal core (IPC, basic scheduling, memory) runs in kernel space. Everything else (drivers, filesystems) runs as user-space servers. A crashing driver doesn't take down the kernel. Trade-off: performance overhead of IPC for every driver interaction.

Linux chose monolithic for performance, but adopted module loading for flexibility. Modern Linux also has mechanisms like FUSE (filesystem in userspace) and io_uring that push some work toward user space.

---

**Q4. Explain the Linux kernel's initcall mechanism. How does a driver register itself at boot time?**

**Answer:** The initcall mechanism is how kernel subsystems and built-in drivers register themselves during boot without the core kernel needing to explicitly call them. 

When a driver uses `module_init(my_driver_init)`, this macro (for a built-in driver, `CONFIG=y`) expands to place a function pointer into a special ELF section (`.initcall6.init` for `module_init` level). During `do_initcalls()` in `init/main.c`, the kernel iterates through all these sections in order and calls each function pointer.

There are 8 initcall levels (0 through 7 plus `rootfs`), allowing ordering:
- `pure_initcall` (level 0) — very early, before most subsystems
- `core_initcall` (level 1) — core subsystems
- `subsys_initcall` (level 4) — bus subsystems
- `device_initcall` (level 6) — drivers (this is what `module_init` maps to)
- `late_initcall` (level 7) — things that need everything else ready

For modules (`.ko` files), `module_init` marks the init function and it's called by `do_init_module()` when `insmod`/`modprobe` loads the module.

---

**Q5. What is `vmlinux` vs `bzImage` vs `Image`? When would you use each?**

**Answer:**
- **`vmlinux`**: The raw, uncompressed ELF binary of the kernel. Not bootable directly by most bootloaders. Used for kernel debugging with `gdb` or the `crash` utility (post-mortem analysis of kernel crash dumps). Contains all debug symbols if built with `CONFIG_DEBUG_INFO=y`.
- **`bzImage`** (x86): "Big zImage" — a self-decompressing x86 kernel image. The bootloader (GRUB, syslinux) loads this into memory; the decompressor stub runs and decompresses the kernel in place, then jumps to `startup_64`. This is what you put in `/boot/`.
- **`Image`** (ARM64): Uncompressed ARM64 kernel image. ARM64 bootloaders (U-Boot, EFI) load this directly. Can also be `Image.gz` (gzip compressed) or `Image.lz4`.

For driver development, `vmlinux` is important because crash dumps and KGDB need it to resolve symbol addresses back to function names.

---

**Q6. What does `make defconfig` do versus `make menuconfig`?**

**Answer:** `make defconfig` generates a `.config` file using the architecture's default configuration defined in `arch/<arch>/configs/<arch>_defconfig`. This is a sane baseline with common options enabled. For x86-64 it enables most common hardware; for ARM, it may be SoC-specific (e.g., `bcm2711_defconfig` for RPi4). The result is a working kernel that boots on most hardware for that arch.

`make menuconfig` opens a text-based interactive menu where you can browse and change any of the ~10,000+ configuration options. Changes are written to `.config`. You run `menuconfig` after `defconfig` to customize — enabling a new driver, turning on debug features like KASAN, or reducing kernel size.

`make olddefconfig` takes an existing `.config` (perhaps copied from a running system) and automatically fills in any new symbols added since it was generated, using their default values. This is the safe way to update a `.config` to a newer kernel version.

---

**Q7. What is the `System.map` file and why does the kernel need it?**

**Answer:** `System.map` is a text file mapping kernel symbol names to their virtual addresses, generated after each kernel build. Example:

```
ffffffff81234560 T sys_read
ffffffff81234700 t copy_from_user_nofault
ffffffff82000000 B _end
```

The kernel uses it for:
1. **Oops messages** — when the kernel crashes, it prints a stack trace. Without symbol information, you'd see raw addresses like `ffffffff81234560`. With `System.map`, the kernel can print `sys_read+0x40/0x80` — the function name plus offset and size.
2. **`/proc/kallsyms`** — provides runtime access to kernel symbols
3. **Debugging tools** — `gdb`, `crash`, `perf`, `ftrace` all use symbol information

If you boot a kernel with the wrong `System.map`, oops messages will show incorrect symbol names — confusing when debugging.

---

**Q8. How does the Linux kernel discover hardware? Name at least three discovery mechanisms.**

**Answer:**
1. **Device Tree (DT)**: Used on most ARM/ARM64 platforms. A binary blob (`.dtb`) passed by the bootloader to the kernel describing the hardware topology — what peripherals exist, their base addresses, IRQ numbers, clocks, etc. The kernel parses the DT and creates platform devices. This is the dominant mechanism on embedded Linux (RPi4, BeagleBone, i.MX, etc.).

2. **ACPI (Advanced Configuration and Power Interface)**: Used on x86 PCs and ARM64 servers. The BIOS/UEFI provides ACPI tables (stored in ROM) describing hardware. The kernel parses these tables to discover devices, power states, IRQ routing.

3. **PCI/PCIe enumeration**: The PCI bus controller supports reading device identity from config space (vendor ID, device ID, class code) by scanning bus/device/function combinations. The kernel's PCI subsystem enumerates all 256 buses × 32 devices × 8 functions and creates a `pci_dev` for each responding device.

4. **USB enumeration**: When a USB device is connected, the hub generates a port status change interrupt. The USB core resets the device, reads its device descriptor, then reads configuration descriptors, interface descriptors, and endpoint descriptors. This builds a complete picture of the device's capabilities.

5. **`/sys/bus/platform/devices`** (platform devices): On some simple hardware, the kernel itself hardcodes the existence of certain peripherals via board files or DT — these become "platform devices" that don't need active enumeration.

---

> **Next:** Day 2 — Cross-compiling your first kernel module and deploying it to RPi4.
