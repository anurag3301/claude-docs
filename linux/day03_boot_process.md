# Day 3 — Boot Process Deep Dive

> **Estimated read time:** 90–120 minutes  
> **Goal:** Trace the complete path from power-on to your shell prompt, at source-code level.

---

## 1. The Complete Boot Chain

```
Power On
    │
    ▼
CPU Reset Vector (0xFFFFFFF0)
    │
    ▼
BIOS / UEFI Firmware
    │  (POST, hardware init, finds boot device)
    ▼
Bootloader (GRUB2 / systemd-boot / EFISTUB)
    │  (loads kernel image + initramfs into RAM)
    ▼
Kernel Decompression  ← arch/x86/boot/compressed/
    │  (bzImage decompresses itself)
    ▼
arch/x86/boot/header.S  ← Real-mode setup code
    │
    ▼
arch/x86/kernel/head_64.S  ← 64-bit entry, early page tables
    │
    ▼
start_kernel()  ← init/main.c  ← THIS IS THE MAIN SUBJECT
    │
    ▼
kernel_init() thread  ← PID 1 kernel thread
    │
    ▼
/sbin/init (or systemd)  ← First user-space process
    │
    ▼
Login prompt / getty
```

---

## 2. Firmware Phase: BIOS vs UEFI

### 2.1 Legacy BIOS

Power-on → CPU starts at the reset vector (`0xFFFFFFF0` in real mode, which maps to the last 16 bytes of a 4GB address space, usually ROM). Executes firmware code.

BIOS firmware:
1. POST (Power-On Self-Test): tests RAM, finds CPUs, detects hardware
2. Reads the **MBR** (Master Boot Record — first 512 bytes of boot disk)
3. MBR contains GRUB's stage 1 bootloader (446 bytes)
4. Stage 1 loads stage 2 from a dedicated partition
5. Stage 2 presents the GRUB menu and loads the kernel

BIOS limitations: 16-bit real-mode entry, 1 MB addressable, MBR partition table limited to 2TB disks, no native driver model.

### 2.2 UEFI

UEFI (Unified Extensible Firmware Interface) is the modern replacement. Key differences:

- Starts in 32-bit protected mode (or 64-bit on most modern systems)
- Reads **GPT** partition tables (supports >2TB disks, >4 primary partitions)
- Has a full **ESP** (EFI System Partition) — FAT32 filesystem where bootloaders live as `.efi` executables
- Provides runtime services: `EFI_RUNTIME_SERVICES` (time, NVRAM variables, system reset)
- Provides boot-time services: memory map, GOP (graphics), PCI enumeration
- Supports **Secure Boot**: only signed bootloaders/kernels can run

UEFI boot flow for Linux:
1. UEFI firmware finds the ESP
2. Loads the bootloader EFI binary (e.g., `/EFI/ubuntu/grubx64.efi`)
3. GRUB or systemd-boot presents menu
4. Selected entry loads kernel + initramfs

### 2.3 EFISTUB

The Linux kernel can also be loaded *directly* as a UEFI binary without a separate bootloader (`CONFIG_EFI_STUB=y`). The kernel embeds a small EFI stub in `arch/x86/boot/compressed/eboot.c` that interfaces with UEFI services directly.

```
UEFI Firmware
    → loads arch/x86/boot/bzImage directly
    → EFI stub: sets up memory map, cmdline, initramfs
    → jumps to kernel entry point
```

### 2.4 What UEFI Hands to the Kernel

Before jumping to the kernel, the bootloader/UEFI stub passes:
- **Command line** (from GRUB config or `linux` command)
- **Memory map** (which physical ranges are usable, reserved, ACPI, etc.)
- **Initramfs location** in RAM
- **Framebuffer info** (for early console)
- **ACPI tables** location
- **SMBIOS** (system information)

All this is passed via the **Boot Protocol** — a documented interface in `Documentation/x86/boot.rst`. The bootloader fills in a `setup_header` struct at a fixed offset in the bzImage.

---

## 3. The bzImage Format

When you `make bzImage`, the output is not just the raw `vmlinux` binary. It's a composite:

```
bzImage layout:
┌─────────────────────────┐  ← offset 0
│  Real-mode setup code   │  (arch/x86/boot/setup.bin)
│  (512 bytes aligned)    │  Contains: setup_header with version info,
│                         │  boot parameters, early video setup
├─────────────────────────┤  ← offset 512 * setup_sects
│  Protected-mode kernel  │  (arch/x86/boot/compressed/vmlinux.bin)
│  (compressed with gzip/ │  Contains: decompressor + compressed vmlinux
│   xz/lzma/zstd)        │
└─────────────────────────┘
```

The bootloader:
1. Reads the setup header to find the protected-mode offset
2. Loads the real-mode setup into low memory (< 640KB)
3. Loads the protected-mode part into a higher address (typically 1MB+)
4. Jumps to the real-mode entry point OR directly to the protected-mode entry (UEFI)

---

## 4. Kernel Decompression (arch/x86/boot/compressed/)

The protected-mode portion runs the **decompressor**:

```c
// arch/x86/boot/compressed/head_64.S (simplified flow):
// 1. Enter 64-bit long mode (set up early page tables)
// 2. Call extract_kernel()

// arch/x86/boot/compressed/misc.c:
asmlinkage __visible void *extract_kernel(...)
{
    // Chooses output address (randomized if KASLR enabled)
    choose_random_location(input, input_size, &output, ...);
    
    // Decompresses vmlinux to the chosen location
    __decompress(input, input_size, NULL, NULL, output, ...);
    
    // Parses ELF headers, applies relocations
    parse_elf(output);
    handle_relocations(output, virt_addr);
    
    return output;  // pointer to decompressed vmlinux entry
}
```

If `CONFIG_RANDOMIZE_BASE=y` (KASLR), `choose_random_location()` picks a random physical address for the kernel. This requires relocations to be applied after decompression.

After decompression, a jump to the decompressed kernel's entry point:
`arch/x86/kernel/head_64.S :: startup_64`

---

## 5. Early Kernel Entry (arch/x86/kernel/head_64.S)

`startup_64` is the real kernel's entry point:

```asm
// arch/x86/kernel/head_64.S (conceptual flow):

startup_64:
    // We're in 64-bit mode, but with the decompressor's page tables
    // Set up the real kernel page tables (early_top_pgt)
    // Map: kernel text, data, BSS
    // Map: physmem map (for accessing all physical memory)

    // Initialize the stack (to init_thread_union.stack)
    
    // Clear BSS segment (zero-initialize uninitialized globals)
    
    // Call x86_64_start_kernel() (first C function!)
```

`x86_64_start_kernel()` in `arch/x86/kernel/head64.c`:

```c
void __init x86_64_start_kernel(char *real_mode_data)
{
    // Verify CPU supports needed features (CPUID checks)
    check_la57_support();
    
    // Build the initial page tables properly
    init_top_pgt();
    
    // Set up early exception handlers
    idt_setup_early_handler();
    
    // Copy boot parameters from real_mode_data
    copy_bootdata(__va(real_mode_data));
    
    // Jump to start_kernel()
    start_kernel();
}
```

---

## 6. `start_kernel()` — The Kernel's Main Function

`init/main.c :: start_kernel()` is arguably the most important function to read in the entire kernel. It initializes every major subsystem in a careful order. Here's the annotated real sequence:

```c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    char *command_line;
    char *after_dashes;

    // The very first thing: set up the stack canary for this CPU
    set_task_stack_end_magic(&init_task);
    
    // SMP CPU ID for boot CPU
    smp_setup_processor_id();
    
    // Debug object infrastructure
    debug_objects_early_init();
    
    // Initialize the NUMA topology
    init_vmlinux_build_id();
    
    // Register CPU-specific handlers
    cgroup_init_early();
    
    // Disable interrupts (we're not ready for them yet)
    local_irq_disable();
    early_boot_irqs_disabled = true;
    
    // Architecture-specific early setup
    // On x86: detect CPU type, set up ACPI tables, parse e820 memory map
    boot_cpu_init();
    page_address_init();
    
    pr_notice("%s", linux_banner);  // "Linux version 6.9..."
    
    // Parse kernel command line (e.g., "root=/dev/sda1 quiet splash")
    early_security_init();
    setup_arch(&command_line);   // arch-specific: memory layout, CPU features
    setup_boot_config();
    setup_command_line(command_line);
    
    // Initialize per-CPU areas
    setup_nr_cpu_ids();
    setup_per_cpu_areas();
    
    // Prepare the scheduler data structures (no actual scheduling yet)
    smp_prepare_boot_cpu();
    boot_cpu_hotplug_init();
    
    // Build the memory zone lists (DMA, NORMAL, HIGHMEM zones)
    build_all_zonelists(NULL);
    page_alloc_init();
    
    // Parse more of the command line
    parse_early_param();
    
    // Initialize the buddy allocator
    // After this point, kmalloc() works
    mm_init();
    
    // Initialize the slab allocators
    kmem_cache_init();
    
    // Set up the scheduler properly
    sched_init();
    
    // Enable the local timer interrupt
    init_timers();
    hrtimers_init();
    
    // Set up softirqs
    softirq_init();
    
    // Initialize the clocksource subsystem
    timekeeping_init();
    
    // Enable interrupts now that we're ready
    early_irq_init();
    init_IRQ();
    tick_init();
    rcu_init();
    
    // Initialize the virtual filesystem switch (but no filesystems yet)
    vfs_caches_init_early();
    
    // Sort exception table (for fixup handlers)
    sort_main_extable();
    
    // Trap/exception handler setup
    trap_init();
    
    // Memory management final setup
    mm_init();     // calls mem_init() among others
    
    // Initialize the rest of the scheduler
    sched_clock_init();
    calibrate_delay();
    
    // PID allocation
    pid_idr_init();
    anon_vma_init();
    
    // IPC subsystem
    ipc_init();
    
    // Initialize virtual filesystem caches
    vfs_caches_init();
    
    // Page writeback
    page_writeback_init();
    
    // Proc filesystem
    proc_root_init();
    
    // NUMA policy
    numa_policy_init();
    
    // Setup initial root filesystem (in initramfs)
    // Then fork PID 1
    arch_call_rest_init();  // → rest_init()
}
```

### 6.1 `setup_arch()` — Architecture Entry Point

`setup_arch()` (in `arch/x86/kernel/setup.c`) is one of the longest functions in the kernel. For x86, it:

1. Parses the **e820 memory map** from firmware — this tells the kernel which physical addresses are RAM, reserved, ACPI, etc.
2. Sets up **KASLR** slot if enabled
3. Initializes early **NUMA** topology
4. Copies command line from boot params
5. Sets up the **DMI** table (firmware strings)
6. Initializes the **ACPI** tables
7. Calls `paging_init()` to set up the permanent page tables for the kernel

---

## 7. `rest_init()` — Spawning PID 1 and PID 2

```c
// init/main.c
static noinline void __ref rest_init(void)
{
    struct task_struct *tsk;
    int pid;

    rcu_scheduler_starting();
    
    // Create PID 1: kernel_init thread
    // This will eventually exec() /sbin/init (systemd, etc.)
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    
    // Create PID 2: kthreadd
    // This is the kernel thread daemon — all other kernel threads
    // are created as children of kthreadd
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    
    // Start the scheduler — the boot CPU becomes the idle thread (PID 0)
    // PID 0 runs when no other thread is runnable
    schedule_preempt_disabled();
    
    // We are now the idle thread (swapper/0)
    cpu_startup_entry(CPUHP_ONLINE);  // → while(1) cpu_idle()
}
```

After this point:
- **PID 0** (`swapper/0`): the idle task, runs `hlt` instruction when no work
- **PID 1** (`kernel_init`): will mount filesystems and exec init
- **PID 2** (`kthreadd`): creates all other kernel threads on request

---

## 8. `kernel_init()` — The Path to User Space

```c
// init/main.c
static int __ref kernel_init(void *unused)
{
    int ret;
    
    // Wait for kthreadd to be ready
    kernel_init_freeable();
    
    // kernel_init_freeable() does:
    //   - do_basic_setup() → driver_init() → all device drivers probe
    //   - do_initcalls()   → runs all __initcall functions in order
    //   - prepare_namespace() → mounts root filesystem
    
    // Free __init sections (recover memory from init code)
    free_initmem();
    
    // From this point we can exec user-space programs
    
    // Try to run /init from the initramfs
    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
        // ramdisk_execute_command = "/init" (from initramfs) or
        // can be overridden via "rdinit=" kernel parameter
    }

    // If that fails, try the compiled-in execute_command:
    if (execute_command) {
        ret = run_init_process(execute_command);
        // execute_command = "init=" kernel parameter
    }
    
    // Fallback: try common init locations
    if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init")  ||
        !try_to_run_init_process("/bin/init")  ||
        !try_to_run_init_process("/bin/sh"))
        panic("No working init found. Try passing init= option to kernel.");
    
    // We never return from run_init_process if successful
    // (exec replaces this kernel thread with the init process)
}
```

### 8.1 `do_initcalls()` — Driver Initialization

Inside `kernel_init_freeable()` → `do_basic_setup()`:

```c
// init/main.c
static void __init do_initcalls(void)
{
    int level;
    size_t len = saved_command_line_len + 1;
    char *command_line;
    
    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
        // Run all initcalls at this level
        do_initcall_level(level, command_line);
    }
}
```

The initcall table is built from ELF sections. Each `device_initcall(fn)` macro puts `fn` into `.initcall6.init` section. The linker script (`arch/x86/kernel/vmlinux.lds.S`) collects all these into a sorted array.

```bash
# You can see the actual initcall order in dmesg:
dmesg | grep "initcall\|calling"

# Or with kernel parameter: initcall_debug
# Boot with: initcall_debug in kernel command line
```

---

## 9. initramfs — The Initial RAM Filesystem

The initramfs is a compressed cpio archive loaded into RAM by the bootloader. The kernel extracts it to a tmpfs filesystem and uses it as the initial root filesystem.

### 9.1 Why initramfs Exists

The kernel needs drivers to access the root filesystem (e.g., NVMe driver for an NVMe SSD). But those drivers are compiled as modules stored *on* the root filesystem. Chicken-and-egg problem.

Solution: initramfs contains the minimum drivers needed to mount the real root, then hands off.

### 9.2 initramfs Content

```bash
# Unpack and examine an initramfs:
mkdir /tmp/initrd-examine && cd /tmp/initrd-examine
dd if=/boot/initrd.img-6.9.0 bs=512 skip=0 | zcat | cpio -idm

ls /tmp/initrd-examine/
# bin/   dev/   etc/   init   lib/   lib64/  run/   sbin/  sys/   tmp/   usr/   var/

cat init   # The /init script that runs in the initramfs
```

The `/init` script (usually from `dracut` or `initramfs-tools`):
1. Mounts `/proc`, `/sys`, `/dev`
2. Loads necessary modules (NVMe, AHCI, filesystem modules)
3. Scans for the real root partition
4. Mounts the real root at `/sysroot`
5. Does `switch_root /sysroot /sbin/init`

### 9.3 `switch_root`

`switch_root` is a crucial step:
1. Moves all mounts from initramfs root to the real root (`/sysroot`)
2. Changes the process's root via `chroot`
3. Frees the initramfs memory (since it was tmpfs, it's just freed)
4. Execs the real `/sbin/init`

In kernel source, this is `init/do_mounts.c :: prepare_namespace()` → `mount_root()`.

### 9.4 Building initramfs

```bash
# Ubuntu/Debian:
update-initramfs -u -k 6.9.0    # Update for specific version
update-initramfs -u              # Update for running kernel

# RHEL/Fedora (uses dracut):
dracut --force /boot/initramfs-6.9.0.img 6.9.0

# Minimal initramfs with busybox (for kernel development/testing):
# Use buildroot or manually:
cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
exec /bin/sh
EOF
chmod +x init
find . | cpio -H newc -o | gzip > initramfs.cpio.gz
```

---

## 10. The e820 Memory Map

The e820 map (named after the BIOS interrupt `0xe820`) tells the kernel the physical memory layout. Modern UEFI exposes this via `EFI_MEMORY_MAP`.

```bash
# View e820 map (from boot messages):
dmesg | grep -i e820

# Example output:
# BIOS-e820: [mem 0x0000000000000000-0x000000000009efff] usable
# BIOS-e820: [mem 0x000000000009f000-0x000000000009ffff] reserved  ← BIOS data area
# BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved  ← ROM
# BIOS-e820: [mem 0x0000000000100000-0x00000000bffeffff] usable    ← Main RAM
# BIOS-e820: [mem 0x00000000bfff0000-0x00000000bfffffff] ACPI data
# BIOS-e820: [mem 0x00000000fd000000-0x00000000ffffffff] reserved  ← MMIO (PCI BARs)
# BIOS-e820: [mem 0x0000000100000000-0x000000023fffffff] usable    ← RAM above 4GB
```

The kernel processes this in `setup_arch()` via `e820__memory_setup()`. Non-RAM regions become `ioremap` territory for drivers. The buddy allocator only manages "usable" regions.

---

## 11. ACPI — Advanced Configuration and Power Interface

ACPI is the firmware interface for power management and hardware enumeration. The kernel's ACPI subsystem (`drivers/acpi/`) interprets ACPI tables to:

- Discover CPUs and their topology
- Manage power states (S0=running, S3=suspend-to-RAM, S4=hibernate, S5=soft off)
- Configure interrupt routing (MADT table = APIC configuration)
- Enumerate PCIe devices

```bash
# View ACPI tables:
sudo acpidump | acpixtract -a  # Extract to files
sudo cat /sys/firmware/acpi/tables/DSDT | iasl -d -  # Disassemble

# DSDT (Differentiated System Description Table) is AML bytecode
# that describes hardware. The kernel has an AML interpreter.

# See ACPI devices:
cat /proc/acpi/wakeup
ls /sys/bus/acpi/devices/
```

ACPI AML (ACPI Machine Language) is a bytecode language the firmware uses to describe hardware. The kernel's AML interpreter in `drivers/acpi/acpica/` (shared code from the ACPICA project) runs this code to configure devices.

---

## 12. SMP Boot — Bringing Up Secondary CPUs

The boot CPU (BSP — Bootstrap Processor) starts alone. Secondary CPUs (APs — Application Processors) are parked in a low-power state.

The SMP boot sequence after the BSP is up:

```c
// kernel/smp.c and arch/x86/kernel/smpboot.c

// BSP sends INIT IPI (inter-processor interrupt) to each AP
// Then sends two STARTUP IPIs with the address of:
// arch/x86/realmode/rm/trampoline_64.S

// Each AP starts in real mode at the trampoline:
// 1. Switches to protected mode
// 2. Switches to 64-bit long mode
// 3. Sets up its own GDT, IDT
// 4. Calls start_secondary()
```

`start_secondary()` (`arch/x86/kernel/smpboot.c`):

```c
static void notrace start_secondary(void *unused)
{
    cpu_init();                // Set up this CPU's GDT, IDT, TSS
    x86_cpuinit.early_percpu_clock_init();
    preempt_disable();
    smp_callin();              // Signal BSP that we're up
    
    cpu_startup_entry(CPUHP_AP_ONLINE_IDLE);  // Become idle thread
}
```

```bash
# See CPU bring-up messages:
dmesg | grep -E "smpboot|CPU.*up|Brought up"
# smpboot: Booting Node 0 Processor 1 APIC 0x1
# CPU1: thread -1, cpu 1, socket 0, mpidr 0x0 -- Set online
```

---

## 13. The Kernel Command Line

The kernel command line is the primary way to configure kernel behavior at boot. Passed by the bootloader.

```bash
# View current command line:
cat /proc/cmdline
# root=UUID=xxx ro quiet splash

# Common parameters:
root=/dev/sda1        # Root device
root=UUID=xxx         # Root by UUID (preferred)
ro / rw               # Mount root read-only or read-write initially
quiet                 # Suppress most boot messages
splash                # Show graphical splash screen
nomodeset             # Disable KMS (fall back to VGA mode)
init=/bin/bash        # Override init process (emergency access)
single / 1            # Boot into single-user mode
emergency             # Boot to emergency shell in initramfs
mem=4G                # Limit usable RAM
maxcpus=2             # Limit CPU count
nokaslr               # Disable KASLR (for debugging)
noapic                # Disable APIC (old hardware)
acpi=off              # Disable ACPI
console=ttyS0,115200  # Serial console
debug                 # Enable kernel debug messages
loglevel=7            # Max log level (0=emergency, 7=debug)
initcall_debug        # Print each initcall as it runs
```

Kernel subsystems register their command line parameters via:

```c
// Example: setting up "quiet" parameter
static int __init quiet_kernel(char *str)
{
    console_loglevel = CONSOLE_LOGLEVEL_QUIET;
    return 0;
}
early_param("quiet", quiet_kernel);   // Parsed very early in setup_arch
// OR:
__setup("root=", root_dev_setup);     // Parsed later in parse_args
```

---

## 14. dmesg — Reading the Boot Log

```bash
dmesg                              # All messages
dmesg -T                           # With timestamps
dmesg -l err,crit,alert,emerg      # Only errors
dmesg | grep -i "fail\|error\|warn"
dmesg --follow                     # Live stream (like tail -f)

# Persistent boot log (systemd):
journalctl -b -0                   # This boot
journalctl -b -1                   # Previous boot
journalctl -b --no-pager | grep "Kernel command"
```

Key dmesg phases to recognize:

```
[    0.000000] Linux version 6.9.0 (gcc version 13.2.0)
[    0.000000] BIOS-provided physical RAM map:          ← e820
[    0.000000] ACPI: RSDP 0x00000000000F04B0 000024    ← ACPI tables
[    0.000000] NUMA: ...
[    0.012345] Calibrating delay loop...                ← BogoMIPS
[    0.023456] PID hash table entries: 4096
[    0.034567] Mount-cache hash table entries: 512
[    0.045678] Initializing cgroup subsys cpuset        ← cgroups init
[    0.056789] NET: Registered PF_INET6 protocol family ← network
[    1.234567] PCI: Probing PCI hardware                ← driver probe
[    1.345678] nvme nvme0: pci function 0000:00:1d.0    ← NVMe found
[    2.456789] EXT4-fs (nvme0n1p2): mounted filesystem  ← root mounted
[    2.567890] Run /init as init process                ← initramfs /init
[    3.678901] systemd[1]: ...                          ← PID 1 running
```

---

## 15. Bootloader Deep Dive: GRUB2

### 15.1 GRUB2 Configuration

```bash
# Main config generated by update-grub:
cat /boot/grub/grub.cfg    # DO NOT EDIT DIRECTLY

# User customization:
cat /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
# GRUB_TIMEOUT=5

# Custom entries:
ls /etc/grub.d/

# Apply changes:
sudo update-grub          # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # Fedora/RHEL
```

### 15.2 GRUB2 Boot Entry Format

```
menuentry 'Ubuntu, with Linux 6.9.0' {
    recordfail
    load_video
    gfxmode $linux_gfx_mode
    insmod gzio
    insmod part_gpt
    insmod ext2
    
    set root='hd0,gpt2'           # Boot partition
    
    linux   /boot/vmlinuz-6.9.0 root=UUID=xxx ro quiet splash
    initrd  /boot/initrd.img-6.9.0
}
```

### 15.3 GRUB Rescue Shell

```bash
# If GRUB fails to find config, drops to grub rescue> prompt
# From there, manually boot:
grub rescue> ls                    # List devices
grub rescue> ls (hd0,gpt2)/       # List partition contents
grub rescue> set prefix=(hd0,gpt2)/boot/grub
grub rescue> set root=(hd0,gpt2)
grub rescue> insmod normal
grub rescue> normal                # Load normal GRUB
```

---

## 16. EFI Variables and Boot Entries

```bash
# View EFI boot entries:
efibootmgr -v

# Output:
# BootCurrent: 0003
# Timeout: 1 seconds
# BootOrder: 0003,0000,0001
# Boot0000* ubuntu    HD(1,GPT,...)/File(\EFI\ubuntu\shimx64.efi)
# Boot0003* Linux     HD(1,GPT,...)/File(\EFI\BOOT\BOOTX64.EFI)

# Add a new boot entry:
sudo efibootmgr --create --disk /dev/sda --part 1 \
    --label "My Kernel 6.9" \
    --loader '\EFI\ubuntu\vmlinuz-6.9.0' \
    --unicode 'root=UUID=xxx ro'
```

EFI variables are stored in NVRAM on the motherboard. On Linux, they're accessible via `/sys/firmware/efi/efivars/`.

---

## 17. Secure Boot and Kernel Signing

```bash
# Check if Secure Boot is enabled:
mokutil --sb-state
# or:
cat /sys/firmware/efi/efivars/SecureBoot-*/

# Kernel modules must be signed to load with Secure Boot + lockdown mode
# Key generation:
openssl req -new -x509 -newkey rsa:2048 -keyout signing_key.pem \
    -out signing_cert.pem -days 3650 -subj "/CN=My Kernel Module/"

# Sign a module:
/usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 \
    signing_key.pem signing_cert.pem my_module.ko

# Enroll the certificate so Secure Boot accepts it:
sudo mokutil --import signing_cert.pem
```

---

## 18. Tracing the Boot with systemd

For the userspace portion of boot:

```bash
# Analyze boot time breakdown:
systemd-analyze
# Startup finished in 1.2s (firmware) + 2.3s (loader) + 4.1s (kernel) + 8.7s (userspace)

# Which services took longest:
systemd-analyze blame

# Visual waterfall chart:
systemd-analyze plot > boot.svg
```

---

## 19. Mental Model Checkpoint

After Day 3, you should be able to:

1. Trace the complete boot path from CPU reset to your shell, naming specific files and functions.
2. Explain the difference between BIOS and UEFI boot. What is the ESP?
3. What is the bzImage format? What does the decompressor do?
4. What does `start_kernel()` do in terms of subsystem initialization order?
5. Why does the kernel need initramfs? What is `switch_root`?
6. What are initcalls and what order do they run in?
7. What is the e820 memory map and who provides it?
8. Explain the SMP boot process — how do secondary CPUs start?
9. What happens after `kernel_init()` calls `free_initmem()`?

---

## Key Source Files to Read

```bash
init/main.c                        # start_kernel(), rest_init(), kernel_init()
arch/x86/boot/header.S            # Boot sector, setup header
arch/x86/boot/compressed/misc.c   # extract_kernel() decompressor
arch/x86/kernel/head_64.S        # startup_64, first 64-bit code
arch/x86/kernel/setup.c          # setup_arch() — MASSIVE, read carefully
init/do_mounts.c                   # Root filesystem mounting
usr/gen_init_cpio.c               # initramfs generation
Documentation/x86/boot.rst        # Boot protocol specification
```

---

## Summary

The boot process has three major phases:

1. **Firmware phase** (BIOS/UEFI): hardware discovery, boot device selection, bootloader loading. The firmware hands the kernel a memory map, command line, and ACPI tables.

2. **Kernel startup phase** (`startup_64` → `start_kernel()`): progressive subsystem initialization in strict dependency order. The system goes from a bare CPU with no memory allocator to a fully running kernel with drivers, filesystems, and a scheduler.

3. **User space handoff** (`kernel_init()` → PID 1): initramfs execution, root filesystem mounting, module loading, and finally `exec()` of `/sbin/init` (systemd).

The key insight is that `start_kernel()` is *ordered* — each subsystem that comes up enables the next. You cannot initialize the slab allocator before the buddy allocator, and you cannot run drivers before the slab allocator. Reading `start_kernel()` from top to bottom is one of the best ways to understand kernel subsystem dependencies.

Tomorrow: system calls — the mechanism that is at the heart of every interaction between user space and kernel.
