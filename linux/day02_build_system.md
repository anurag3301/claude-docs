# Day 2 — Kernel Source Layout & Build System

> **Estimated read time:** 90–120 minutes  
> **Kernel version referenced:** 6.x  
> **Goal:** Navigate the kernel tree confidently, understand Kconfig/Kbuild deeply, and compile + boot a custom kernel.

---

## 1. Getting the Source

Never use a distro-patched kernel for learning internals. Always use upstream:

```bash
# Option 1: Git (full history, essential for kernel work)
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
# Warning: ~4GB repo. Use --depth=1 if you just want latest.

# Option 2: Tarball (faster, no history)
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.9.tar.xz
tar -xf linux-6.9.tar.xz

# Useful tree for stable releases:
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

The kernel.org Git tree has multiple branches:
- `master` / `main` — Linus's tree (latest development)
- `linux-6.9.y` — stable/LTS maintenance branch
- `next/master` — linux-next: integration tree where subsystem trees merge ahead of the next release

---

## 2. Source Tree Deep Dive

```
linux/
├── arch/                    ← Architecture-specific code
│   ├── x86/
│   │   ├── boot/           # Real-mode boot code, compressed image
│   │   ├── entry/          # Syscall entry/exit, exception handlers
│   │   │   ├── entry_64.S  # The actual syscall/interrupt entry in asm
│   │   │   └── syscalls/   # Syscall table generation
│   │   ├── include/asm/    # x86 arch headers
│   │   ├── kernel/         # CPU setup, SMP, APIC, context_switch
│   │   ├── kvm/            # KVM hypervisor code
│   │   ├── mm/             # x86 page table management
│   │   └── lib/            # Optimized string/copy routines (memcpy, etc.)
│   ├── arm64/              # AArch64
│   ├── riscv/              # RISC-V
│   └── ...                 # ~25 architectures total
│
├── block/                   ← Block I/O layer
│   ├── blk-core.c          # bio allocation, submission
│   ├── blk-mq.c            # Multi-queue block layer
│   └── elevator.c          # I/O scheduler framework
│
├── certs/                   ← Kernel signing certificates
│
├── crypto/                  ← Cryptographic API
│
├── drivers/                 ← ALL device drivers (~60% of kernel source)
│   ├── base/               # Device model core (kobject, driver core)
│   ├── block/              # Block device drivers (loop, ramdisk...)
│   ├── char/               # Character devices (random, tty, mem...)
│   ├── dma/                # DMA engine subsystem
│   ├── gpu/drm/            # DRM/KMS display subsystem
│   ├── hwmon/              # Hardware monitoring (sensors)
│   ├── i2c/                # I2C bus
│   ├── input/              # Input devices (keyboard, mouse)
│   ├── iommu/              # IOMMU drivers
│   ├── irqchip/            # Interrupt controller drivers
│   ├── md/                 # Software RAID
│   ├── net/                # Network interface drivers
│   │   ├── ethernet/
│   │   │   ├── intel/      # e1000e, igb, ixgbe, ice...
│   │   │   └── ...
│   │   └── wireless/
│   ├── nvme/               # NVMe storage
│   ├── pci/                # PCI/PCIe subsystem
│   ├── pinctrl/            # Pin controller (GPIO muxing)
│   ├── platform/           # Platform devices
│   ├── scsi/               # SCSI subsystem
│   ├── thermal/            # Thermal management
│   ├── usb/                # USB subsystem
│   └── virtio/             # Virtio paravirtual devices
│
├── fs/                      ← Filesystems
│   ├── ext4/               # ext4
│   ├── btrfs/              # btrfs
│   ├── xfs/                # XFS
│   ├── nfs/                # NFS client
│   ├── nfsd/               # NFS server
│   ├── proc/               # /proc virtual filesystem
│   ├── sysfs/              # /sys virtual filesystem
│   ├── tmpfs/              # tmpfs / ramfs
│   ├── overlayfs/          # Union mount (used by containers)
│   ├── fuse/               # FUSE userspace filesystem
│   ├── f2fs/               # Flash-Friendly FS
│   └── ...                 # 60+ filesystems
│
├── include/
│   ├── linux/              # Core kernel headers (sched.h, mm_types.h, fs.h...)
│   ├── uapi/linux/         # Headers exported to userspace (syscall numbers, ioctl defs)
│   ├── asm-generic/        # Generic arch implementations
│   └── net/                # Networking headers
│
├── init/
│   ├── main.c              # start_kernel() — DO READ THIS
│   ├── do_mounts.c         # Root filesystem mounting
│   └── initramfs.c         # Initial RAM filesystem
│
├── ipc/                     ← SysV IPC, POSIX message queues
│
├── kernel/                  ← Core kernel subsystems
│   ├── sched/              # Scheduler
│   │   ├── core.c          # schedule(), context_switch(), wake_up_process()
│   │   ├── fair.c          # CFS implementation
│   │   ├── rt.c            # Real-time scheduler
│   │   ├── deadline.c      # EDF deadline scheduler
│   │   └── smp.c           # SMP load balancing
│   ├── fork.c              # copy_process(), kernel_thread()
│   ├── exit.c              # do_exit(), release_task()
│   ├── signal.c            # Signal delivery
│   ├── sys.c               # Miscellaneous syscalls
│   ├── time/               # Timekeeping, hrtimers
│   ├── irq/                # IRQ management
│   ├── locking/            # Mutex, rwsem, spinlock implementations
│   ├── bpf/                # eBPF verifier, maps, programs
│   ├── cgroup/             # cgroup v1 and v2
│   └── trace/              # ftrace, tracepoints
│
├── lib/                     ← Kernel library (not architecture-specific)
│   ├── rbtree.c            # Red-black tree
│   ├── list_sort.c         # List sorting
│   ├── idr.c               # ID allocation (integer ↔ pointer maps)
│   ├── bitmap.c            # Bitmap operations
│   └── ...
│
├── mm/                      ← Memory management
│   ├── memory.c            # Page fault handling, do_anonymous_page()
│   ├── page_alloc.c        # Buddy allocator
│   ├── slub.c              # SLUB allocator
│   ├── vmalloc.c           # vmalloc
│   ├── mmap.c              # mmap() implementation
│   ├── swap.c              # Swap management
│   ├── vmscan.c            # Page reclaim (kswapd)
│   ├── oom_kill.c          # OOM killer
│   └── huge_memory.c       # Transparent Huge Pages
│
├── net/                     ← Network stack
│   ├── core/               # Socket layer, net_device, dev.c (packet Rx/Tx)
│   ├── ipv4/               # IPv4, TCP, UDP, ICMP
│   ├── ipv6/               # IPv6
│   ├── netfilter/          # Netfilter / iptables hooks
│   ├── sched/              # Traffic control (TC) qdiscs
│   └── ...
│
├── security/                ← LSM and security modules
│   ├── security.c          # LSM hook dispatch
│   ├── selinux/            # SELinux
│   ├── apparmor/           # AppArmor
│   └── ...
│
├── sound/                   ← ALSA audio subsystem
│
├── tools/                   ← Userspace tools in the kernel tree
│   ├── perf/               # perf tool source
│   ├── bpf/                # BPF development tools (bpftool, samples)
│   ├── testing/            # Testing infrastructure (kunit)
│   └── ...
│
├── usr/                     ← initramfs generation scripts
│
└── virt/
    └── kvm/                # KVM hypervisor (common parts; arch-specific in arch/)
```

---

## 3. Kconfig — The Configuration System

### 3.1 What Kconfig Is

Kconfig is a domain-specific language for describing the kernel's configuration options. Every feature, driver, and subsystem can be:
- **y** (yes): compiled directly into the kernel image
- **m** (module): compiled as a loadable kernel module (.ko file)
- **n** (no): not compiled at all

Options have types: `bool` (y/n), `tristate` (y/m/n), `int`, `hex`, `string`.

### 3.2 Kconfig Files

Every directory has a `Kconfig` file. They are included hierarchically from the top-level `Kconfig`.

```kconfig
# Example: drivers/net/ethernet/intel/Kconfig (simplified)

config E1000
    tristate "Intel(R) PRO/1000 Gigabit Ethernet support"
    depends on PCI
    select PHYLIB
    help
      This driver supports Intel(R) PRO/1000 gigabit ethernet family.
      ...

config E1000E
    tristate "Intel(R) PRO/1000 PCI-Express Gigabit Ethernet support"
    depends on PCI && (!SPARC32)
    depends on PTP_1588_CLOCK_OPTIONAL
    select CRC32
    help
      ...
```

Key Kconfig keywords:
- `depends on X`: only visible/selectable if X is y or m
- `select X`: forces X=y when this option is enabled
- `imply X`: softly suggests enabling X (user can override)
- `default y/m/n`: default value
- `if ... endif`: conditional blocks
- `source "path/Kconfig"`: include another Kconfig file
- `menuconfig`: creates a submenu in the UI
- `choice ... endchoice`: mutually exclusive options

### 3.3 Configuration Interfaces

```bash
# Text-based menu (most common)
make menuconfig

# Graphical (requires Qt or GTK)
make xconfig
make gconfig

# Minimalist text
make config     # asks every question sequentially (don't use this)

# Generate .config from current running kernel
make localmodconfig  # only enable modules currently loaded (great for VMs)
make localyesconfig  # same but compile modules as built-in

# Default config for the current architecture
make defconfig

# A known-minimal config for fast testing
make tinyconfig

# Update .config after changing kernel version (keeps existing choices)
make oldconfig
make olddefconfig  # same but uses defaults for new options without asking
```

The result of any config step is a `.config` file at the top of the source tree. This is what Kbuild reads.

### 3.4 `.config` File Format

```bash
# Auto-generated by Kconfig. Manual editing works but use the tools.
CONFIG_64BIT=y
CONFIG_X86_64=y
CONFIG_SMP=y
CONFIG_PREEMPT_VOLUNTARY=y
# CONFIG_PREEMPT is not set       ← this means n
CONFIG_HZ_250=y
CONFIG_HZ=250
CONFIG_E1000E=m                   ← module
CONFIG_DEBUG_KERNEL=y
```

After configuration, `include/generated/autoconf.h` is generated with `#define CONFIG_*` values that C code uses.

---

## 4. Kbuild — The Build System

### 4.1 How Kbuild Works

Kbuild is a set of Makefiles (the top-level `Makefile` + recursive `Makefile`s in each directory) that drive the build process.

Each directory has a `Makefile` that tells Kbuild what to build:

```makefile
# drivers/net/ethernet/intel/e1000e/Makefile
obj-$(CONFIG_E1000E) += e1000e.o
e1000e-objs := netdev.o ethtool.o hw.o mac.o manage.o nvm.o param.o \
               phy.o ich8lan.o 82571.o 82574.o pch_gbe.o
```

`obj-$(CONFIG_E1000E)` expands to either:
- `obj-y` if `CONFIG_E1000E=y` → compiled into vmlinux
- `obj-m` if `CONFIG_E1000E=m` → compiled as `e1000e.ko`
- nothing if `CONFIG_E1000E=n` → not built

### 4.2 Object Naming Conventions

```makefile
obj-y    += foo.o          # built into vmlinux
obj-m    += bar.o          # built as bar.ko module
lib-y    += baz.o          # compiled into kernel library (lib.a)

# Compound module (multiple source files → single .ko)
mydriver-y := file1.o file2.o file3.o
obj-m += mydriver.o

# Subdirectory
obj-y += subdir/           # recurse into subdir/Makefile
```

### 4.3 Build Targets

```bash
# Build everything (vmlinux + modules + compressed image)
make -j$(nproc)

# Just the kernel image (no modules)
make vmlinux

# Compressed bootable image (what goes in /boot)
make bzImage          # x86: arch/x86/boot/bzImage
make Image            # arm64: arch/arm64/boot/Image

# Just modules
make modules

# Install modules to /lib/modules/$(uname -r)/
sudo make modules_install

# Install kernel image + update bootloader
sudo make install

# Cross-compile for ARM64 (from x86 host):
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

### 4.4 Build Artifacts

```
vmlinux          ← Uncompressed ELF kernel binary (with debug symbols)
vmlinux.o        ← Linked but not stripped
System.map       ← Symbol address table (critical for debugging)
arch/x86/boot/bzImage  ← Bootable compressed kernel
.config          ← Current configuration
Module.symvers   ← Exported symbol versions (for module compatibility)
```

### 4.5 Compiler Flags

The kernel has its own set of compiler flags enforced by Kbuild:

```makefile
# Key flags (from Makefile, simplified):
KBUILD_CFLAGS = -Wall -Wundef -Wstrict-prototypes -fno-builtin \
                -ffreestanding -fno-stack-protector \
                -fno-common -std=gnu11

# No user-space C standard library — kernel is freestanding
KBUILD_CFLAGS += -ffreestanding

# Position-independent code for KASLR
KBUILD_CFLAGS += -fPIE
```

Notable: `-ffreestanding` means no `main()`, no libc headers, no standard startup. The kernel provides its own `<linux/...>` headers.

---

## 5. Building Your First Custom Kernel

### 5.1 Prerequisites

```bash
# Debian/Ubuntu:
sudo apt install build-essential libncurses-dev bison flex \
                 libssl-dev libelf-dev dwarves bc pahole

# Fedora/RHEL:
sudo dnf install gcc make ncurses-devel bison flex \
                 openssl-devel elfutils-libelf-devel dwarves bc
```

### 5.2 Start from Your Running Config

```bash
cd linux/
# Copy running kernel config (most sane starting point)
cp /boot/config-$(uname -r) .config

# Update it for the new kernel version
make olddefconfig

# Customize (optional)
make menuconfig
```

### 5.3 Build

```bash
# Parallel build — use all cores
make -j$(nproc) 2>&1 | tee build.log

# Build will produce:
# arch/x86/boot/bzImage  (compressed kernel)
# vmlinux                (uncompressed, with symbols)
# .ko files under various subdirs
```

### 5.4 Install

```bash
# Install modules first (required before booting)
sudo make modules_install
# Creates: /lib/modules/6.9.0/

# Install kernel + update GRUB
sudo make install
# Copies bzImage to /boot/vmlinuz-6.9.0
# Copies System.map to /boot/System.map-6.9.0
# Runs update-grub or grub2-mkconfig automatically
```

### 5.5 Verify Boot

```bash
reboot
# Select new kernel in GRUB menu
uname -r    # should show 6.9.0
```

---

## 6. Kernel Headers and `#include` Conventions

Kernel code includes headers differently from user space:

```c
// User space:
#include <stdio.h>       // libc
#include <stdlib.h>

// Kernel:
#include <linux/kernel.h>    // printk, ARRAY_SIZE, min/max, etc.
#include <linux/module.h>    // MODULE_* macros, module_init/exit
#include <linux/sched.h>     // task_struct, current
#include <linux/mm.h>        // mm_struct, vmalloc, kmalloc
#include <linux/fs.h>        // file, inode, super_block
#include <linux/net.h>       // socket
#include <linux/skbuff.h>    // sk_buff
#include <linux/slab.h>      // kmalloc, kfree, kmem_cache_*
#include <linux/spinlock.h>  // spinlock_t, spin_lock/unlock
#include <linux/mutex.h>     // struct mutex
#include <linux/list.h>      // list_head, list_add, list_for_each
#include <linux/types.h>     // u8, u16, u32, u64, bool, ...
#include <linux/init.h>      // __init, __exit section markers
#include <asm/io.h>          // ioread32, iowrite32, ioremap
```

**`linux/` vs `asm/`**: `linux/` headers are portable across architectures; `asm/` headers are architecture-specific (resolved via `include/asm` symlink to `arch/x86/include/asm/` etc.).

**`uapi/` headers**: These are the subset of kernel headers safe to include from user space. They live in `include/uapi/linux/` and are installed to `/usr/include/linux/` by `make headers_install`.

---

## 7. Makefile Tricks Kernel Developers Use

### 7.1 Compiling a Single File

```bash
# Just compile one .c file (useful for testing syntax)
make drivers/net/ethernet/intel/e1000e/netdev.o

# Rebuild a single module
make M=drivers/net/ethernet/intel/e1000e/
```

### 7.2 Verbose Build

```bash
# See full compiler commands
make V=1

# Or just see commands without output
make V=2
```

### 7.3 `ccache` for Faster Rebuilds

```bash
# Install ccache, then:
make CC="ccache gcc" -j$(nproc)
# Subsequent builds with unchanged files are near-instant
```

### 7.4 Out-of-Tree Build

```bash
# Build kernel in a separate directory (keeps source clean)
mkdir /tmp/kernel-build
make O=/tmp/kernel-build menuconfig
make O=/tmp/kernel-build -j$(nproc)
```

### 7.5 `make` Targets You Should Know

```bash
make help                  # List all targets
make kernelversion         # Print kernel version
make kernelrelease         # Print full release string
make cscope/tags/ctags     # Generate code navigation database
make coccicheck            # Run Coccinelle semantic patches
make dt_bindings_check     # Validate device tree bindings
make kernel_lockdown_check # Check lockdown compatibility
make kunit                 # Run KUnit tests
```

---

## 8. Code Navigation — Essential for Kernel Work

The kernel is ~30 million lines of code. You need tools.

### 8.1 `cscope`

```bash
make cscope
# Generates cscope.out
cscope -d    # launch TUI
# Then: Ctrl+\ to find symbol references, definitions, callers
```

### 8.2 `ctags` / `etags`

```bash
make tags     # ctags format (vim)
make TAGS     # etags format (emacs)

# In vim:
# Ctrl+]  = jump to definition
# Ctrl+t  = jump back
```

### 8.3 `clangd` / `compile_commands.json`

```bash
# Generate compilation database for clangd/ccls IDE integration
make CC=clang bear -- make -j$(nproc)
# Or use:
scripts/clang-tools/gen_compile_commands.py
# Creates compile_commands.json at root
# Point VSCode/CLion/Neovim to it for full IntelliSense
```

### 8.4 Elixir Cross-Reference (Online)

`https://elixir.bootlin.com/linux/latest/` — browser-based kernel source navigator. Best for quick lookups without a local checkout.

### 8.5 `grep` / `ripgrep` Patterns

```bash
# Find all uses of a function
rg 'kmalloc\b' mm/

# Find all CONFIG_NET references
rg 'CONFIG_NET' Kconfig --type kconfig

# Find all EXPORT_SYMBOL for a function
rg 'EXPORT_SYMBOL.*my_func'

# Find where a struct is defined
rg 'struct task_struct {' include/
```

---

## 9. `__init`, `__exit`, and Section Annotations

```c
// Functions marked __init are placed in a special ELF section
// (.init.text). After boot, this section is freed.
static int __init my_driver_init(void) { ... }

// Similarly for data:
static int __initdata my_boot_param = 42;

// __exit marks cleanup code only used when module is removed
// (or not needed if built-in)
static void __exit my_driver_exit(void) { ... }
```

This is how the kernel reclaims ~1–2MB of memory after initialization code is no longer needed. You'll see `Freeing unused kernel image (text/rodata gap) memory` in `dmesg` during boot.

---

## 10. `MODULE_*` Macros and Kernel Module Boilerplate

Even if you're not writing a module yet, understanding this helps because it appears everywhere:

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");              // Required for GPL-only symbols
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Example module");
MODULE_VERSION("1.0");

static int __init my_init(void)
{
    printk(KERN_INFO "my_module: loaded\n");
    return 0;  // 0 = success, negative errno = failure
}

static void __exit my_exit(void)
{
    printk(KERN_INFO "my_module: unloaded\n");
}

module_init(my_init);   // registers init function
module_exit(my_exit);   // registers cleanup function
```

`module_init()` and `module_exit()` are macros that, for built-in code, wire into the kernel's `initcall` mechanism. For modules, they set the entry/exit points loaded by `insmod`.

---

## 11. The `initcall` System

When you see `core_initcall()`, `device_initcall()`, `late_initcall()` in the kernel source — these are the built-in equivalent of `module_init()`. They define *when* during boot a subsystem initializes:

```c
// Ordered from earliest to latest:
early_initcall(fn)      // Very early (memory allocators)
core_initcall(fn)       // Core subsystems
postcore_initcall(fn)
arch_initcall(fn)       // Architecture-specific
subsys_initcall(fn)     // Subsystems (PCI, USB buses...)
fs_initcall(fn)         // Filesystems
rootfs_initcall(fn)     // Root filesystem
device_initcall(fn)     // Most drivers (= module_init for built-ins)
late_initcall(fn)       // Things that need everything else ready
```

`start_kernel()` in `init/main.c` calls `do_initcalls()` which runs all these in order, populating the kernel with functionality from a bare minimum to fully operational.

---

## 12. Kernel Version Checks in Code

```c
#include <linux/version.h>

// Check kernel version at compile time
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 0, 0)
    // Use new API
#else
    // Fall back to old API
#endif

// KERNEL_VERSION(a,b,c) expands to (a << 16) + (b << 8) + c
// LINUX_VERSION_CODE is the same format for the current kernel
```

This matters when writing drivers that need to support multiple kernel versions (common in out-of-tree drivers).

---

## 13. `sparse` — The Kernel's Static Analyzer

```bash
# Install sparse
sudo apt install sparse

# Build with sparse checking:
make C=1   # Check only files being recompiled
make C=2   # Check all files

# sparse catches:
# - Mixing user/kernel pointers (__user annotations)
# - Locking imbalances (using acquire/release annotations)
# - Endianness bugs (__be32, __le32 vs u32)
# - Missing RCU protection
```

The `__user`, `__iomem`, `__percpu`, `__rcu` annotations are for sparse. They help catch bugs like passing a user-space pointer directly to a function expecting a kernel pointer.

---

## 14. Key Makefile Variables

```makefile
# In kernel Makefiles:
$(srctree)    # top of source tree (= $(CURDIR) unless O= is used)
$(objtree)    # top of build tree (= srctree unless O= is used)
$(src)        # current source directory
$(obj)        # current object directory

# Compiler:
$(CC)         # C compiler (gcc or clang)
$(LD)         # Linker
$(AR)         # Archiver
$(NM)         # Symbol listing
$(OBJCOPY)    # Object copy/strip
```

---

## 15. Practical: Setting Up a Kernel Dev VM

The best way to develop and test kernel code without risking your host:

### Option A: QEMU + KVM (Recommended)

```bash
# Install QEMU
sudo apt install qemu-system-x86 qemu-kvm

# Build kernel with debug config
cp /boot/config-$(uname -r) .config
scripts/config --enable DEBUG_KERNEL
scripts/config --enable DEBUG_INFO
scripts/config --enable GDB_SCRIPTS
scripts/config --enable KGDB
scripts/config --enable KASAN
make olddefconfig
make -j$(nproc)

# Create a minimal root filesystem
# (use buildroot or a debootstrap minimal install)

# Boot in QEMU:
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd /path/to/initramfs.cpio.gz \
    -append "console=ttyS0 root=/dev/sda nokaslr" \
    -drive file=/path/to/rootfs.img,format=raw \
    -m 2G \
    -smp 2 \
    -nographic \
    -enable-kvm
```

### Option B: `virtme-ng` (Easier)

```bash
pip install virtme-ng
# Boot your compiled kernel directly:
virtme-ng --kdir . --mods auto
```

### Option C: Using `syzkaller` Infrastructure

For fuzzing-focused kernel work, `syzkaller` provides ready-made VM images.

---

## 16. `scripts/` — Essential Tools in the Kernel Tree

```
scripts/
├── checkpatch.pl        # Code style checker — run before any submission
├── clang-tools/         # compile_commands.json generator
├── coccinelle/          # Semantic patches (automated refactoring)
├── decodecode           # Decode oops disassembly
├── faddr2line           # Convert function+offset to source line
├── get_maintainer.pl    # Who to CC on a patch
├── gdb/                 # GDB scripts for kernel debugging
├── kconfig/             # Kconfig parser and UI
├── mod/                 # Module signing and version tools
└── recordmcount.c       # ftrace support
```

```bash
# Check your code before submitting:
scripts/checkpatch.pl --strict --file mydriver.c

# Find maintainers for net/ipv4/tcp.c:
scripts/get_maintainer.pl net/ipv4/tcp.c

# Decode an oops offset:
scripts/faddr2line vmlinux kernel_init+0x57
```

---

## 17. Mental Model Checkpoint

Before Day 3, you should be able to:

1. Navigate the kernel source tree and know which directory contains what.
2. Explain `obj-y` vs `obj-m` and how Kbuild decides what to compile.
3. Configure a kernel using `make menuconfig` and `make localmodconfig`.
4. Understand what `depends on`, `select`, and `default` do in Kconfig.
5. Compile a kernel from source, install it, and boot it.
6. Understand what `__init` and `__exit` do to binary size.
7. Know the difference between `linux/` and `uapi/linux/` headers.
8. Set up a QEMU-based kernel development environment.

---

## Key Files to Read

```bash
Makefile                          # Top-level build system — read the header comments
scripts/Makefile.build            # Core of the recursive build
init/Kconfig                      # Top-level kernel features
arch/x86/Kconfig                  # x86-specific options
include/linux/init.h              # __init, module_init, initcall macros
include/linux/module.h            # MODULE_* macros
init/main.c                       # start_kernel() and do_initcalls()
scripts/checkpatch.pl             # Read to understand kernel code style rules
```

---

## Summary

The kernel's build system is two interlinked pieces:
- **Kconfig**: a declarative language for feature selection, producing `.config`
- **Kbuild**: a recursive Makefile system that reads `.config` and compiles the selected code

Every directory has both a `Kconfig` (describing options) and a `Makefile` (describing what to build when options are enabled). `obj-$(CONFIG_X)` is the fundamental pattern. Understanding this fully means you can add a new driver, subsystem, or configuration option from scratch.

Tomorrow: the boot process in depth — from power-on through UEFI, bootloader, kernel decompression, `start_kernel()`, and the first user-space process.
