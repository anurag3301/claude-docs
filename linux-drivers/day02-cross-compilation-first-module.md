# Day 2 — Cross-Compilation & Your First Kernel Module on RPi4
**Environment:** 🔵 RPi4 + 💻 PC
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [Cross-Compilation Fundamentals](#1-cross-compilation-fundamentals)
2. [Setting Up the RPi4 Toolchain](#2-setting-up-the-rpi4-toolchain)
3. [Anatomy of a Kernel Module](#3-anatomy-of-a-kernel-module)
4. [The Module Makefile in Detail](#4-the-module-makefile-in-detail)
5. [printk — Kernel Logging](#5-printk--kernel-logging)
6. [Module Parameters](#6-module-parameters)
7. [Module Metadata & Licensing](#7-module-metadata--licensing)
8. [Loading, Unloading & Inspecting Modules](#8-loading-unloading--inspecting-modules)
9. [Hands-On Lab — Build & Deploy to RPi4](#9-hands-on-lab--build--deploy-to-rpi4)
10. [Interview Questions](#10-interview-questions)

---

## 1. Cross-Compilation Fundamentals

Cross-compilation means building code on one architecture (the **host**) to run on a different architecture (the **target**).

```
Host: x86-64 PC (your dev machine)
         │
         │ cross-compiler (aarch64-linux-gnu-gcc)
         ▼
Target: ARM64 Raspberry Pi 4B+
```

### Why Cross-Compile?

Your RPi4 can compile kernel modules natively, but:
- It's much slower (Cortex-A72 vs your desktop CPU)
- You don't want a full build environment on your target
- In professional embedded development, the target often has no compiler at all (memory constraints)
- The target may not even have a display or keyboard

### Toolchain Components

A cross-compilation toolchain consists of:

| Component | Native Name | Cross Name (ARM64) |
|-----------|------------|-------------------|
| C compiler | `gcc` | `aarch64-linux-gnu-gcc` |
| C++ compiler | `g++` | `aarch64-linux-gnu-g++` |
| Linker | `ld` | `aarch64-linux-gnu-ld` |
| Assembler | `as` | `aarch64-linux-gnu-as` |
| Object dumper | `objdump` | `aarch64-linux-gnu-objdump` |
| Strip tool | `strip` | `aarch64-linux-gnu-strip` |
| Symbol tool | `nm` | `aarch64-linux-gnu-nm` |

The prefix `aarch64-linux-gnu-` is the **target triplet**:
- `aarch64` — target CPU architecture (ARM 64-bit)
- `linux` — target OS kernel
- `gnu` — target C library (glibc)

### The ARCH and CROSS_COMPILE Variables

The kernel Makefile checks these two environment variables:
- `ARCH` — target architecture. Maps to `arch/` subdirectory. Examples: `x86`, `arm64`, `arm`, `riscv`, `mips`
- `CROSS_COMPILE` — prefix for all toolchain binaries

```bash
# Every kernel make command for RPi4 needs these:
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- <target>

# Or export them once in your shell session:
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
# Now you can just run: make <target>
```

### Kernel Module Versioning — A Critical Concept

A kernel module is tightly coupled to the exact kernel it was built against. You **cannot** take a `.ko` built for kernel `6.1.21-v8+` and load it into kernel `6.6.28-v8+`. The kernel enforces this through:

1. **`vermagic` string** — embedded in every `.ko`. Encodes kernel version, SMP/preempt configuration. `insmod` rejects modules with mismatched vermagic.
2. **Symbol CRC checksums** — in `Module.symvers`. Each exported kernel symbol has a CRC. If the kernel API changed (function signature changed), the CRC changes, and the module load fails with `disagrees about version of symbol`.

This means: **you must build your module against the exact kernel headers of the kernel running on your target**.

---

## 2. Setting Up the RPi4 Toolchain

### On Your PC (Ubuntu/Debian)

```bash
# Install cross-compiler
sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# Verify
aarch64-linux-gnu-gcc --version
# aarch64-linux-gnu-gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0

# Install additional build tools
sudo apt install build-essential bc libssl-dev libelf-dev \
    flex bison libncurses-dev dwarves rsync

# Install deployment tools
sudo apt install sshpass  # for passwordless scp in scripts
```

### Getting the RPi Kernel Source

For RPi4, use the Raspberry Pi Foundation's fork which has all RPi-specific patches:

```bash
git clone --depth=1 https://github.com/raspberrypi/linux ~/rpi-kernel
cd ~/rpi-kernel
```

### Getting the Correct Kernel Config for Your RPi4

You need the `.config` that matches the kernel **currently running** on your RPi4. There are two ways:

**Method 1: Copy from RPi4 over SSH (preferred)**
```bash
# First, check kernel version on RPi4
ssh pi@<rpi-ip> uname -r
# e.g., 6.6.28-v8+

# Copy the config
scp pi@<rpi-ip>:/proc/config.gz ./
gunzip config.gz
mv config ~/rpi-kernel/.config
```

**Method 2: Use RPi Foundation's defconfig**
```bash
cd ~/rpi-kernel
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
```

### Prepare Kernel Headers for Module Building

```bash
cd ~/rpi-kernel

# This generates all the headers and module infrastructure needed
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules_prepare

# This creates:
# - include/generated/autoconf.h
# - include/generated/utsrelease.h
# - Module.symvers
# - scripts/ (build helpers)
```

---

## 3. Anatomy of a Kernel Module

A kernel module is an ELF shared object that can be dynamically linked into the running kernel. Let's dissect one completely.

```c
// hello.c — the simplest possible kernel module

#include <linux/init.h>      // __init, __exit macros
#include <linux/module.h>    // MODULE_LICENSE, module_init, module_exit
#include <linux/kernel.h>    // KERN_INFO, printk

// Module init function — called when module is loaded
static int __init hello_init(void)
{
    printk(KERN_INFO "hello: module loaded on CPU %d\n", smp_processor_id());
    return 0;  // non-zero = load failure, module is unloaded
}

// Module exit function — called when module is unloaded
static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello: module unloaded\n");
    // No return value — exit cannot fail
}

// Register init and exit functions
module_init(hello_init);
module_exit(hello_exit);

// Required metadata
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name <you@example.com>");
MODULE_DESCRIPTION("Day 2 hello world kernel module");
MODULE_VERSION("0.1");
```

### Line-by-Line Breakdown

#### Headers

```c
#include <linux/init.h>
```
Provides `__init` and `__exit` section markers. These tell the linker to place these functions in special ELF sections (`.init.text` and `.exit.text`). For built-in drivers (`CONFIG=y`), the kernel **frees** the `.init.text` section after boot — those functions are only needed once. For modules, the section markers are ignored at runtime but useful for documentation.

```c
#include <linux/module.h>
```
The most important header for any module. Provides:
- `module_init()` / `module_exit()` macros
- `MODULE_LICENSE()` / `MODULE_AUTHOR()` etc.
- `EXPORT_SYMBOL()` / `EXPORT_SYMBOL_GPL()`
- `THIS_MODULE` pointer

```c
#include <linux/kernel.h>
```
Core kernel utilities:
- `printk()` — kernel logging
- `KERN_INFO`, `KERN_ERR`, `KERN_DEBUG` — log level macros
- `pr_info()`, `pr_err()`, `pr_debug()` — formatted wrappers
- `BUG()`, `BUG_ON()`, `WARN()`, `WARN_ON()` — assertion helpers

#### The `__init` and `__exit` Macros

```c
static int __init hello_init(void)
```

`__init` expands to `__attribute__((__section__(".init.text")))` — it places the function in the `.init.text` ELF section.

`static` is important: your init/exit functions should always be `static`. They're not exported; they're only referenced by `module_init()`. Making them non-static would pollute the global kernel namespace.

#### Return Value of init

```c
return 0;   // success
return -ENOMEM;  // failure — module won't load, kernel reports error
return -ENODEV;  // failure — device not found
```

The convention is to return 0 on success and a **negative errno** on failure. The list of error codes is in `include/uapi/asm-generic/errno-base.h` and `include/uapi/asm-generic/errno.h`. Common ones:

| Code | Value | Meaning |
|------|-------|---------|
| `ENOMEM` | 12 | Out of memory |
| `ENODEV` | 19 | No such device |
| `EINVAL` | 22 | Invalid argument |
| `EBUSY` | 16 | Device or resource busy |
| `EPERM` | 1 | Operation not permitted |
| `EIO` | 5 | I/O error |
| `ETIMEDOUT` | 110 | Connection timed out |

#### `module_init()` and `module_exit()`

```c
module_init(hello_init);
module_exit(hello_exit);
```

These macros do different things depending on context:
- For modules (`.ko`): They define special symbols that the module loader looks for: `init_module` and `cleanup_module`. The module loader calls these when loading/unloading.
- For built-in (compiled into kernel): `module_init` uses `__initcall()` to register the function in the initcall table at the `device_initcall` level.

---

## 4. The Module Makefile in Detail

The Makefile for an out-of-tree module (built separately from the kernel source) has a specific structure:

```makefile
# Makefile for hello kernel module

# The object file(s) that make up the module
# obj-m means "build this as a module"
obj-m += hello.o

# If your module has multiple source files:
# obj-m += mydriver.o
# mydriver-objs := core.o utils.o hw_access.o

# Path to the kernel source/build directory
# For native compilation on the target:
# KDIR := /lib/modules/$(shell uname -r)/build

# For cross-compilation from PC targeting RPi4:
KDIR := $(HOME)/rpi-kernel

# Architecture and cross-compiler
ARCH := arm64
CROSS_COMPILE := aarch64-linux-gnu-

# The M= variable tells the kernel build system where our module source is
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean

# Optional: copy to RPi4 after building
deploy:
	scp hello.ko pi@$(RPI_IP):/home/pi/
```

### How `make -C $(KDIR) M=$(PWD) modules` Works

1. `make -C $(KDIR)` — changes into the kernel source directory and invokes its Makefile
2. `M=$(PWD)` — tells the kernel Makefile "there's an external module in this directory"
3. `modules` — the target to build
4. The kernel Makefile then reads `$(PWD)/Makefile` and processes `obj-m += hello.o`
5. It compiles `hello.c` using the kernel's exact build flags and links it as `hello.ko`

### Build Output Files

After `make`, you get several files:

```
hello.c          — your source
hello.o          — compiled object
hello.mod.c      — auto-generated module metadata source
hello.mod.o      — compiled module metadata
hello.ko         — final loadable kernel module
Module.symvers   — symbol version info
modules.order    — order of modules built
.hello.ko.cmd    — exact compiler command used (useful for debugging)
```

The `.ko` file is what you copy to the target.

### Inspecting a `.ko` File

```bash
# See the vermagic string (must match target kernel)
modinfo hello.ko

# Example output:
# filename:       /path/to/hello.ko
# version:        0.1
# description:    Day 2 hello world kernel module
# author:         Your Name <you@example.com>
# license:        GPL
# vermagic:       6.6.28-v8+ SMP preempt mod_unload modversions aarch64

# See ELF sections
aarch64-linux-gnu-objdump -h hello.ko

# See exported symbols (none in this module)
aarch64-linux-gnu-nm hello.ko

# See all symbols including local ones
aarch64-linux-gnu-nm -a hello.ko
```

---

## 5. printk — Kernel Logging

`printk()` is the kernel's equivalent of `printf()`. It's the most fundamental debugging tool you have.

### Log Levels

```c
printk(KERN_EMERG   "system is unusable\n");        // level 0
printk(KERN_ALERT   "action must be taken now\n");   // level 1
printk(KERN_CRIT    "critical condition\n");          // level 2
printk(KERN_ERR     "error condition\n");             // level 3
printk(KERN_WARNING "warning condition\n");           // level 4
printk(KERN_NOTICE  "normal but significant\n");      // level 5
printk(KERN_INFO    "informational\n");               // level 6
printk(KERN_DEBUG   "debug-level message\n");         // level 7
```

### Shorthand Macros (Preferred)

```c
// These are equivalent to printk(KERN_XXX ...) but cleaner
pr_emerg("system is unusable\n");
pr_alert("action must be taken now\n");
pr_crit("critical condition\n");
pr_err("error: value is %d\n", val);     // use for errors
pr_warn("warning: threshold exceeded\n"); // warnings
pr_notice("port %d registered\n", port);
pr_info("module loaded successfully\n");  // general info
pr_debug("entering function %s\n", __func__); // debug (compiled out if !DEBUG)
```

### Device-Context Logging (Even Better for Drivers)

When your module is associated with a device, use device-context logging — it automatically prefixes the device name:

```c
dev_err(dev, "failed to read register\n");
// Prints: "mydriver mydevice: failed to read register"

dev_warn(dev, "temperature high: %d°C\n", temp);
dev_info(dev, "device initialized successfully\n");
dev_dbg(dev, "register 0x%02x = 0x%08x\n", reg, val);
```

### How printk Works Internally

`printk()` is non-blocking and can be called from **any context** — interrupt handlers, atomic sections, early boot. It writes to an in-kernel ring buffer (the kernel log buffer, default 512KB, configurable via `CONFIG_LOG_BUF_SHIFT`). The ring buffer is what `dmesg` reads from `/dev/kmsg`.

```bash
# Read the kernel ring buffer
dmesg

# Follow in real-time (like tail -f)
dmesg -w
# or
tail -f /var/log/kern.log

# Filter by log level (show only warnings and above)
dmesg -l warn,err,crit,alert,emerg

# Clear the ring buffer
sudo dmesg -c

# Show timestamps (relative to boot)
dmesg -T    # human-readable time
dmesg -t    # raw seconds since boot
```

### Console Log Level

The console (terminal) only shows messages at or above the **console log level**. Messages below go to the ring buffer but not the screen.

```bash
# Check current log levels
cat /proc/sys/kernel/printk
# 4  4  1  7
# │  │  │  └── default log level for new printk callers
# │  │  └───── minimum console log level
# │  └──────── default console log level
# └─────────── current console log level

# Show KERN_DEBUG messages on console (set level to 8 = show all)
echo 8 > /proc/sys/kernel/printk

# Or with sysctl
sysctl -w kernel.printk="8 4 1 7"
```

### Rate Limiting

```c
// Only print once per second even if called thousands of times
printk_ratelimited(KERN_WARNING "interrupt flood detected\n");

// Or use the pr_ variant
pr_warn_ratelimited("too many errors: %d\n", count);
```

### Dynamic Debug

```c
pr_debug("this is compiled in but disabled at runtime\n");
```

`pr_debug` uses the **dynamic debug** infrastructure. Messages are compiled in but disabled by default. Enable them per-module, per-file, or per-line at runtime:

```bash
# Enable all debug messages in your module
echo "module hello +p" > /sys/kernel/debug/dynamic_debug/control

# Enable debug for a specific file
echo "file hello.c +p" > /sys/kernel/debug/dynamic_debug/control

# Enable a specific line
echo "file hello.c line 25 +p" > /sys/kernel/debug/dynamic_debug/control

# Disable
echo "module hello -p" > /sys/kernel/debug/dynamic_debug/control
```

---

## 6. Module Parameters

Modules can accept parameters at load time, similar to program arguments.

```c
#include <linux/module.h>
#include <linux/moduleparam.h>

// Declare parameters with default values
static int gpio_pin = 17;           // default GPIO 17
static char *device_name = "mydev"; // default name
static bool debug_mode = false;     // default off
static int thresholds[4] = {10, 20, 30, 40};  // array
static int nthresholds = 4;         // array element count

// Register parameters
// module_param(name, type, permissions)
// Permissions: S_IRUGO = read by all; S_IWUSR = write by root
module_param(gpio_pin, int, S_IRUGO | S_IWUSR);
MODULE_PARM_DESC(gpio_pin, "GPIO pin number to use (default: 17)");

module_param(device_name, charp, S_IRUGO);
MODULE_PARM_DESC(device_name, "Name for the device (default: mydev)");

module_param(debug_mode, bool, S_IRUGO | S_IWUSR);
MODULE_PARM_DESC(debug_mode, "Enable debug output (default: false)");

module_param_array(thresholds, int, &nthresholds, S_IRUGO);
MODULE_PARM_DESC(thresholds, "Array of threshold values");
```

### Supported Parameter Types

| Type | C Type | Description |
|------|--------|-------------|
| `bool` | `bool` | true/false or 1/0 |
| `invbool` | `bool` | inverted bool |
| `charp` | `char *` | string (pointer to char) |
| `int` | `int` | signed integer |
| `uint` | `unsigned int` | unsigned integer |
| `long` | `long` | signed long |
| `ulong` | `unsigned long` | unsigned long |
| `short` | `short` | signed short |
| `ushort` | `unsigned short` | unsigned short |
| `byte` | `unsigned char` | byte |

### Loading with Parameters

```bash
# Pass parameters at load time
sudo insmod hello.ko gpio_pin=22 debug_mode=1

# With modprobe (searches /lib/modules/)
sudo modprobe hello gpio_pin=22 debug_mode=1

# Pass arrays
sudo insmod hello.ko thresholds=5,15,25,35

# Persistent parameters in /etc/modprobe.d/
echo "options hello gpio_pin=22 debug_mode=1" | \
    sudo tee /etc/modprobe.d/hello.conf
```

### Reading/Writing Parameters at Runtime

Parameters with appropriate permissions appear in `/sys/module/<name>/parameters/`:

```bash
# Read current value
cat /sys/module/hello/parameters/gpio_pin

# Change at runtime (if S_IWUSR permission set)
echo 18 | sudo tee /sys/module/hello/parameters/gpio_pin
```

---

## 7. Module Metadata & Licensing

### MODULE_LICENSE — Critical

```c
MODULE_LICENSE("GPL");           // GNU General Public License v2
MODULE_LICENSE("GPL v2");        // explicit GPL v2
MODULE_LICENSE("GPL and additional rights");
MODULE_LICENSE("Dual BSD/GPL");  // dual licensed
MODULE_LICENSE("Dual MIT/GPL");
MODULE_LICENSE("Proprietary");   // marks module as tainted
```

**Why it matters:** The Linux kernel exports two classes of symbols:
- Regular symbols via `EXPORT_SYMBOL()` — available to all modules
- GPL-only symbols via `EXPORT_SYMBOL_GPL()` — only available to GPL-licensed modules

Many important kernel APIs are `EXPORT_SYMBOL_GPL()` — including the I2C, SPI, regmap, DMA engine, V4L2, and many other frameworks. If your module doesn't have `MODULE_LICENSE("GPL")`, it **cannot use these APIs**. The kernel will refuse to load it.

Additionally, loading a non-GPL module **taints** the kernel. Kernel developers will refuse to help debug a tainted kernel. Many distributions reject bug reports from tainted kernels.

```bash
# Check if kernel is tainted
cat /proc/sys/kernel/tainted
# 0 = clean kernel
# Non-zero = tainted (look up taint flags in kernel docs)
```

### MODULE_AUTHOR, MODULE_DESCRIPTION, MODULE_VERSION

```c
MODULE_AUTHOR("Alice Smith <alice@example.com>");
MODULE_DESCRIPTION("Driver for the XYZ1234 temperature sensor");
MODULE_VERSION("1.2.3");
MODULE_ALIAS("platform:xyz1234");   // module alias for autoloading
MODULE_DEVICE_TABLE(i2c, xyz_id_table);  // for udev autoloading
```

These are embedded as ELF section data in the `.ko` file. Readable with `modinfo`.

---

## 8. Loading, Unloading & Inspecting Modules

### insmod vs modprobe

| Feature | `insmod` | `modprobe` |
|---------|----------|------------|
| Input | Takes a file path (`./hello.ko`) | Takes a module name (`hello`) |
| Dependencies | Doesn't handle deps | Automatically loads dependencies |
| Search path | None — you specify path | Searches `/lib/modules/$(uname -r)/` |
| Config | Doesn't read modprobe.conf | Reads `/etc/modprobe.d/*.conf` |
| Use case | Development/testing | Production, scripting |

```bash
# Load a module (specify full path)
sudo insmod ./hello.ko

# Load with parameters
sudo insmod ./hello.ko gpio_pin=22

# Unload a module
sudo rmmod hello
# or
sudo rmmod ./hello.ko

# Load using modprobe (module must be in /lib/modules/)
sudo modprobe hello

# Unload with modprobe (also removes now-unused dependencies)
sudo modprobe -r hello

# Force remove even if busy (DANGEROUS — can cause instability)
sudo rmmod -f hello
```

### Inspecting Loaded Modules

```bash
# List all loaded modules
lsmod
# Output: Module    Size  Used by
#         hello     16384  0

# Detailed info about a loaded module
cat /proc/modules
# hello 16384 0 - Live 0xffff800000a00000

# Or using sysfs
ls /sys/module/hello/
# holders  initstate  parameters  refcnt  sections  srcversion  taint  uevent

# Module's init state
cat /sys/module/hello/initstate
# live

# Reference count (how many things are using this module)
cat /sys/module/hello/refcnt

# Module information (from .ko file or loaded module)
modinfo hello.ko       # from file
modinfo hello          # from loaded module in /lib/modules/
```

### The Module Reference Count

The `Used by` column in `lsmod` shows the **reference count**. If it's non-zero, `rmmod` will fail with "Module is in use". This prevents unloading a module that other modules or open file descriptors depend on.

```c
// In your driver code, you manage references:
// When a device is opened, reference count goes up:
if (!try_module_get(THIS_MODULE)) {
    return -ENODEV;
}

// When device is closed, decrement:
module_put(THIS_MODULE);
```

For character drivers, the kernel handles this automatically when you register with `cdev_add()`.

### Handling Load Failures

```c
static int __init hello_init(void)
{
    int ret;
    
    ret = some_resource_setup();
    if (ret < 0) {
        pr_err("hello: failed to set up resource: %d\n", ret);
        return ret;  // Module load fails, kernel reports error to insmod
    }
    
    ret = another_setup();
    if (ret < 0) {
        pr_err("hello: second setup failed: %d\n", ret);
        some_resource_teardown();  // Clean up what we already set up!
        return ret;
    }
    
    pr_info("hello: initialized successfully\n");
    return 0;
}
```

This pattern — clean up in reverse order on error — is called **error-path cleanup** and is one of the most common sources of bugs in kernel drivers. Every resource acquired must be freed on every error path.

---

## 9. Hands-On Lab — Build & Deploy to RPi4

### Prerequisites
- RPi4 running Raspberry Pi OS (64-bit recommended)
- PC with Ubuntu/Debian
- SSH access to RPi4
- Both on the same network

### Step 0 — Verify Your RPi4 Kernel Version

```bash
# On RPi4
uname -r
# Example: 6.6.28-v8+
```

Keep this version string — you need it.

### Step 1 — Set Up Kernel Source on PC

```bash
# On PC
git clone --depth=1 --branch rpi-6.6.y \
    https://github.com/raspberrypi/linux ~/rpi-kernel
cd ~/rpi-kernel

# Get the config from your RPi4
scp pi@<RPI_IP>:/proc/config.gz ./
gunzip config.gz && mv config .config

# Prepare the kernel build environment
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules_prepare
```

### Step 2 — Write the Module

```bash
mkdir ~/day2-module
cd ~/day2-module
```

```c
// hello_rpi.c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/utsname.h>   // for utsname()

static char *student_name = "student";
module_param(student_name, charp, S_IRUGO);
MODULE_PARM_DESC(student_name, "Your name (shown in log)");

static int __init hello_rpi_init(void)
{
    pr_info("======================================\n");
    pr_info("hello_rpi: Hello from %s!\n", student_name);
    pr_info("hello_rpi: Running on kernel: %s\n",
            utsname()->release);
    pr_info("hello_rpi: Loaded on CPU: %d\n",
            smp_processor_id());
    pr_info("======================================\n");
    return 0;
}

static void __exit hello_rpi_exit(void)
{
    pr_info("hello_rpi: Goodbye from %s! Module unloaded.\n",
            student_name);
}

module_init(hello_rpi_init);
module_exit(hello_rpi_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("You <you@example.com>");
MODULE_DESCRIPTION("Day 2: Hello RPi4 kernel module");
MODULE_VERSION("0.1");
```

### Step 3 — Write the Makefile

```makefile
# Makefile
ARCH := arm64
CROSS_COMPILE := aarch64-linux-gnu-
KDIR := $(HOME)/rpi-kernel

obj-m += hello_rpi.o

PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean

deploy: all
	scp hello_rpi.ko pi@$(RPI_IP):/home/pi/
	@echo "Deployed to RPi4. Run: sudo insmod hello_rpi.ko"
```

### Step 4 — Build

```bash
cd ~/day2-module
make
# Expected output:
# make -C /home/user/rpi-kernel M=/home/user/day2-module ... modules
# CC [M]  /home/user/day2-module/hello_rpi.o
# MODPOST /home/user/day2-module/Module.symvers
# CC [M]  /home/user/day2-module/hello_rpi.mod.o
# LD [M]  /home/user/day2-module/hello_rpi.ko
# SIGN    /home/user/day2-module/hello_rpi.ko

ls -lh hello_rpi.ko
modinfo hello_rpi.ko
```

### Step 5 — Deploy to RPi4

```bash
# On PC
RPI_IP=<your-rpi-ip>
scp hello_rpi.ko pi@${RPI_IP}:/home/pi/
```

### Step 6 — Load and Test on RPi4

```bash
# On RPi4 (via SSH)

# Check current kernel version
uname -r

# Look at kernel log before loading
sudo dmesg -c > /dev/null  # clear old messages

# Load the module
sudo insmod ~/hello_rpi.ko
# Or with parameter:
sudo insmod ~/hello_rpi.ko student_name="Alice"

# Check the kernel log
dmesg | tail -10
# You should see your printk messages!

# Verify it's loaded
lsmod | grep hello_rpi

# Check module parameters
cat /sys/module/hello_rpi/parameters/student_name

# Unload
sudo rmmod hello_rpi
dmesg | tail -5
# You should see the exit message
```

### Step 7 — Explore Module Details

```bash
# On RPi4
# See where the module was loaded in kernel memory
cat /proc/modules | grep hello_rpi

# See module sections (text, data, bss addresses)
ls /sys/module/hello_rpi/sections/
cat /sys/module/hello_rpi/sections/.text

# Try to force a vermagic mismatch (educational)
# On PC, deliberately change kernel version string in source:
# (this will demonstrate the protection mechanism)
```

### Step 8 — Automate with a Script

```bash
# On PC, create deploy_and_test.sh
cat > deploy_and_test.sh << 'EOF'
#!/bin/bash
RPI_IP=${1:-raspberrypi.local}
MODULE=hello_rpi

echo "Building..."
make clean && make

echo "Deploying to ${RPI_IP}..."
scp ${MODULE}.ko pi@${RPI_IP}:/tmp/

echo "Loading on RPi4..."
ssh pi@${RPI_IP} "sudo rmmod ${MODULE} 2>/dev/null; \
    sudo insmod /tmp/${MODULE}.ko student_name='CI_Test' && \
    dmesg | tail -10 && \
    sudo rmmod ${MODULE} && \
    dmesg | tail -5"
EOF
chmod +x deploy_and_test.sh
./deploy_and_test.sh 192.168.1.xxx
```

---

## 10. Interview Questions

---

**Q1. What is the difference between `insmod` and `modprobe`? Which should you use in production and why?**

**Answer:** `insmod` is a low-level tool that loads a kernel module directly from a file path. It does not resolve dependencies — if your module requires another module to be loaded first, `insmod` will fail. `modprobe` is a higher-level tool that reads the module database in `/lib/modules/$(uname -r)/` (built by `depmod`), automatically resolves and loads dependencies in the correct order, and reads module options from `/etc/modprobe.d/*.conf`.

In production, always use `modprobe`. In development, `insmod` is fine for quick iteration since you're directly referencing your built `.ko` file. For proper packaging and deployment, `modprobe` is essential because drivers often depend on bus infrastructure (e.g., your I2C sensor driver depends on the I2C core being loaded first).

---

**Q2. Why can't you just copy a `.ko` file from one kernel version to another?**

**Answer:** The kernel enforces ABI compatibility through two mechanisms:

1. **vermagic string**: Every `.ko` embeds a `vermagic` string containing the exact kernel version, SMP status, preempt configuration, and architecture. When loading, `insmod`/`modprobe` compares this against the running kernel's vermagic. A mismatch causes an immediate load rejection with "Invalid module format."

2. **Symbol version CRCs**: When `CONFIG_MODVERSIONS=y` (common in distribution kernels), each exported kernel symbol has a CRC computed from its signature (name, argument types, return type). These are stored in `Module.symvers`. When a module is built, it records the CRC of each symbol it uses. At load time, the kernel compares these CRCs against the running kernel's symbol CRCs. If a kernel function's signature changed between versions, the CRC differs and the module load fails.

This strict checking prevents loading a module built against kernel 6.1 into a kernel 6.6 where function signatures may have changed, which would cause memory corruption or crashes.

---

**Q3. What does `MODULE_LICENSE("GPL")` actually do at a technical level?**

**Answer:** `MODULE_LICENSE("GPL")` does three things:

1. **Embeds the license string** in the `.ko` file's `.modinfo` ELF section, readable via `modinfo`. This is metadata.

2. **Controls access to GPL-only symbols**: The kernel marks many of its exported functions with `EXPORT_SYMBOL_GPL()` instead of `EXPORT_SYMBOL()`. When a module attempts to use a GPL-only symbol, the module loader checks the module's license at load time. Non-GPL modules get an `-ENOPROTOOPT` error and the load fails.

3. **Taint flag**: Loading a non-GPL module sets a taint flag on the kernel (`TAINT_PROPRIETARY_MODULE`), visible in `/proc/sys/kernel/tainted`. This signals that the kernel is running unreviewed closed-source code. The taint flag makes kernel developers ignore bug reports — they cannot audit closed-source drivers.

Functionally, `MODULE_LICENSE("GPL")` is what unlocks `EXPORT_SYMBOL_GPL()` APIs like the regmap framework, many I2C/SPI driver APIs, the DMA engine API, V4L2 subdevice ops, and numerous others that drivers commonly need.

---

**Q4. What happens when module_init() returns a non-zero value?**

**Answer:** The kernel's module loader (`kernel/module/main.c`) calls the init function. If it returns a non-zero value (a negative errno), the loader:

1. Prints a message to the kernel log: `hello: init_module failed -12` (for -ENOMEM)
2. Calls the module's `exit` function — **no**, actually it does NOT call exit if init fails
3. Frees any resources the module loader itself allocated
4. Removes the module from the module list
5. Returns the error code to `insmod`/`modprobe`, which prints an error to the user

The important implication: **if your init function fails partway through, you must clean up whatever you allocated before the failure yourself**. The exit function will NOT be called. This is why error handling in init functions uses the reverse-cleanup pattern (goto labels or reverse-order teardown).

```c
static int __init my_init(void)
{
    int ret;
    
    ret = alloc_resource_a();
    if (ret) return ret;
    
    ret = alloc_resource_b();
    if (ret) {
        free_resource_a();  // must clean up A manually
        return ret;
    }
    return 0;
}
```

---

**Q5. Explain the `__init` and `__exit` section attributes. What is their purpose?**

**Answer:** `__init` is defined as `__attribute__((__section__(".init.text")))` and `__exit` as `__attribute__((__section__(".exit.text")))`. They instruct the linker to place the decorated function in specific ELF sections rather than the normal `.text` section.

For **built-in drivers** (compiled with `CONFIG=y`):
- After the kernel finishes booting and all `__init` functions have been called, the kernel calls `free_initmem()` which unmaps and frees all memory occupied by `.init.text`, `.init.data`, etc. This recovers memory — you'll see the "Freeing unused kernel image (initmem) memory: X KB" message in dmesg. `__exit` decorated functions are simply discarded entirely for built-in drivers (there's no way to unload a built-in).

For **loadable modules** (`.ko` files):
- The `__init`/`__exit` attributes have no memory-saving effect since modules can be loaded and unloaded. They serve as documentation conventions and some tools use them for analysis.

The practical implication: functions marked `__init` will disappear from kernel memory after boot (for built-in code). Don't call them after the init phase — the kernel will panic with a page fault.

---

**Q6. What is the `vermagic` string and what does it contain?**

**Answer:** `vermagic` is a string embedded in every `.ko` file (in the `.modinfo` section) that uniquely identifies the kernel build the module was compiled for. On a typical ARM64 system it looks like:

```
6.6.28-v8+ SMP preempt mod_unload modversions aarch64
```

Breaking it down:
- `6.6.28-v8+` — exact kernel version string (`uname -r`)
- `SMP` — compiled for symmetric multiprocessing (multiple CPUs)
- `preempt` — preemptible kernel compiled in (`CONFIG_PREEMPT=y`)
- `mod_unload` — module unloading supported (`CONFIG_MODULE_UNLOAD=y`)
- `modversions` — symbol CRC versioning enabled (`CONFIG_MODVERSIONS=y`)
- `aarch64` — target CPU architecture

The running kernel stores its own vermagic and compares it against the module's. Even a minor version difference (6.6.28 vs 6.6.29) or a different config (preempt vs no preempt) will cause a mismatch. This is why you must build modules against the exact running kernel's headers.

---

**Q7. How does the kernel ensure a module is unloaded safely when it has open file descriptors?**

**Answer:** The kernel uses a **module reference count** mechanism. Every module has a reference counter accessible via `THIS_MODULE->refcnt`. The count is managed by:

1. `try_module_get(THIS_MODULE)` — atomically increments refcount; returns false if the module is being unloaded
2. `module_put(THIS_MODULE)` — decrements refcount

For character drivers, when a user-space process `open()`s the device file, the VFS calls `try_module_get()` on the module that owns the `file_operations`. When `close()` is called, `module_put()` is called.

When `rmmod` is called, the kernel checks if the module's refcount is zero. If non-zero, `rmmod` fails with `ERROR: Module hello is in use` and the module stays loaded. Only when all references are released (all file descriptors closed, all dependent modules unloaded) does the refcount reach zero, allowing safe removal.

`rmmod -f` (force) can bypass this check — it's dangerous because the module's memory is freed even though something still holds a reference to it, leading to use-after-free bugs and kernel panics.

---

> **Next:** Day 3 — Writing a real character device driver that controls the RPi4's ACT LED via `/dev/myled`.
