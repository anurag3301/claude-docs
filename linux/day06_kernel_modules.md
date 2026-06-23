# Day 6 — Kernel Modules

> **Estimated read time:** 90–120 minutes  
> **Goal:** Write, compile, load, debug, and manage kernel modules. Understand the module infrastructure deeply.

---

## 1. What Kernel Modules Are

A Loadable Kernel Module (LKM) is an object file that can be inserted into a running kernel to extend its functionality — without rebooting. Modules run in kernel space with full kernel privileges.

Use cases:
- **Device drivers**: most hardware drivers ship as modules (`nvme.ko`, `e1000e.ko`)
- **Filesystems**: `ext4.ko`, `btrfs.ko`, `fuse.ko`
- **Network protocols**: `nf_conntrack.ko`, `ip_tables.ko`, `vxlan.ko`
- **Security modules**: `selinux.ko`, `apparmor.ko`
- **Debugging/profiling**: `kprobes`, test modules
- **Custom kernel extensions**: your own subsystems

```bash
# See all currently loaded modules:
lsmod

# Output format: Name  Size  UsedBy
# Module                  Size  Used by
# nvidia               38502400  0
# i915                 2830336  0
# e1000e                282624  0
# nf_conntrack          192512  4 xt_conntrack,nf_nat,nf_conntrack_netlink,xt_state

# Detailed module info:
modinfo e1000e
# filename:       /lib/modules/6.9.0/kernel/drivers/net/ethernet/intel/e1000e/e1000e.ko
# version:        3.8.4-NAPI
# description:    Intel(R) PRO/1000 Network Driver
# license:        GPL v2
# depends:        ptp
# parm:           copybreak:Maximum size of packet that is copied to a new buffer (uint)
```

---

## 2. Module Structure: The Minimum

```c
// hello.c — absolute minimum kernel module

#include <linux/module.h>    // MODULE_* macros, module_init/exit
#include <linux/kernel.h>    // printk, KERN_*

// Module metadata (modinfo reads these from the .ko ELF file)
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Hello World kernel module");
MODULE_VERSION("1.0");

// Called when: insmod hello.ko
static int __init hello_init(void)
{
    printk(KERN_INFO "hello: module loaded\n");
    return 0;   // Must return 0 for success; negative errno aborts load
}

// Called when: rmmod hello
static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello: module unloaded\n");
}

module_init(hello_init);   // Register init function
module_exit(hello_exit);   // Register cleanup function
```

### 2.1 Out-of-Tree Makefile

When building a module outside the kernel source tree:

```makefile
# Makefile — out-of-tree module build

# Name of the module (without .ko)
obj-m := hello.o

# If module needs multiple source files:
# mymodule-objs := file1.o file2.o
# obj-m := mymodule.o

# Kernel source directory (where headers are)
KERNEL_DIR ?= /lib/modules/$(shell uname -r)/build

# Default target: build
all:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) modules

# Clean
clean:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) clean

# Install to /lib/modules/$(uname -r)/extra/
install:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) modules_install
	depmod -a
```

```bash
make           # Builds hello.ko
sudo insmod hello.ko
dmesg | tail   # See: hello: module loaded
sudo rmmod hello
dmesg | tail   # See: hello: module unloaded
```

### 2.2 `MODULE_LICENSE` and GPL-Only Symbols

`MODULE_LICENSE("GPL")` is not just metadata. Many kernel symbols are only exported to GPL-licensed modules. If you use `MODULE_LICENSE("Proprietary")`, you'll get link errors trying to use GPL-exported functions, and loading will "taint" the kernel:

```bash
# Check kernel taint flags:
cat /proc/sys/kernel/tainted
# Bit 0 (1): Proprietary module loaded
# Bit 4 (16): Module staging/out-of-tree
# Bit 30 (1073741824): Live patch applied

# See human-readable taint:
cat /proc/sys/kernel/tainted | xargs -I{} sh -c 'printf "%d\n" {}' | \
    python3 -c "
import sys
flags = int(sys.stdin.read())
meanings = {0:'P',1:'F',2:'S',3:'R',4:'M',5:'B',6:'U',7:'D',8:'A',9:'W',10:'C',11:'I',12:'O',13:'E',14:'L',15:'K',16:'X',17:'T'}
print('Taint: ' + ''.join(meanings.get(i,'?') for i in range(18) if flags & (1<<i)))
"
```

---

## 3. `printk` — Kernel Logging

`printk()` is the kernel's only output mechanism. It writes to a ring buffer (`dmesg`) and optionally to the console.

```c
// Log levels (decreasing severity):
printk(KERN_EMERG   "system is unusable\n");         // 0 - "<0>"
printk(KERN_ALERT   "action must be taken now\n");   // 1 - "<1>"
printk(KERN_CRIT    "critical conditions\n");         // 2 - "<2>"
printk(KERN_ERR     "error conditions\n");            // 3 - "<3>"
printk(KERN_WARNING "warning conditions\n");          // 4 - "<4>"
printk(KERN_NOTICE  "normal but significant\n");      // 5 - "<5>"
printk(KERN_INFO    "informational\n");               // 6 - "<6>"
printk(KERN_DEBUG   "debug-level messages\n");        // 7 - "<7>"

// pr_* shortcuts (preferred in modern code):
pr_emerg("...\n");
pr_alert("...\n");
pr_crit("...\n");
pr_err("error: %d\n", err);       // Equivalent to printk(KERN_ERR "error: %d\n", err)
pr_warn("warning: %s\n", msg);
pr_notice("...\n");
pr_info("loaded version %s\n", VERSION);
pr_debug("debug value: %lx\n", val);  // Only compiled in if DEBUG is defined

// Device-specific wrappers (include device name automatically):
dev_err(dev, "initialization failed: %d\n", err);    // drivers
dev_warn(dev, "retrying...\n");
dev_info(dev, "probe successful\n");

// Network-specific:
netdev_err(netdev, "TX timeout\n");
netdev_info(netdev, "link is up at %d Mbps\n", speed);
```

```bash
# View kernel log:
dmesg -T                        # With timestamps
dmesg -w                        # Follow (like tail -f)
dmesg -l err,warn               # Only errors and warnings
journalctl -k                   # Systemd-based systems
journalctl -k -f                # Follow kernel log

# Set console log level (0-7, what appears on console):
echo 7 > /proc/sys/kernel/printk   # Show all messages on console
echo 3 > /proc/sys/kernel/printk   # Only errors and above

# Dynamic debug (enable pr_debug for specific modules/files):
echo 'module hello +p' > /sys/kernel/debug/dynamic_debug/control
# Now pr_debug() calls in hello.c will print!
```

### 3.1 Rate-Limited Logging

Never use bare `printk` in hot paths — you'll flood the log:

```c
// These fire at most once per second (per-CPU):
printk_ratelimited(KERN_ERR "error in hot path: %d\n", err);
pr_err_ratelimited("packet drop: %pI4\n", &addr);

// Fire only on the first occurrence:
printk_once(KERN_INFO "module: first packet processed\n");
pr_warn_once("deprecated API called\n");
```

---

## 4. Module Parameters

Modules can accept parameters at load time:

```c
#include <linux/module.h>
#include <linux/moduleparam.h>

// Simple integer parameter
static int timeout = 5;
module_param(timeout, int, 0644);
MODULE_PARM_DESC(timeout, "Timeout in seconds (default: 5)");

// String parameter
static char *name = "default";
module_param(name, charp, 0444);
MODULE_PARM_DESC(name, "Device name");

// Boolean
static bool verbose = false;
module_param(verbose, bool, 0644);
MODULE_PARM_DESC(verbose, "Enable verbose logging");

// Array of ints
static int ports[8];
static int nports;
module_param_array(ports, int, &nports, 0644);
MODULE_PARM_DESC(ports, "Port numbers to listen on");
```

```bash
# Set at load time:
sudo insmod mymodule.ko timeout=10 verbose=1
sudo insmod mymodule.ko ports=80,443,8080

# View current values (via sysfs):
cat /sys/module/mymodule/parameters/timeout
echo 20 > /sys/module/mymodule/parameters/timeout  # If 0644 (writable)

# Module parameter permissions:
# 0     = not exposed in sysfs
# 0444  = readable by all, not writable
# 0644  = readable/writable by root, readable by others
```

---

## 5. Exporting Symbols

For one module to use functions from another, the provider must export them:

```c
// In provider module (layer.c):
int my_function(int x)
{
    return x * 2;
}
EXPORT_SYMBOL(my_function);          // Available to ALL modules (even non-GPL)

int my_gpl_function(int x)
{
    return x * 3;
}
EXPORT_SYMBOL_GPL(my_gpl_function);  // Only available to GPL-licensed modules

// In consumer module (upper.c):
extern int my_function(int x);       // Just declare; resolved at load time

static int __init upper_init(void)
{
    int result = my_function(21);    // Works if layer.ko is loaded first
    pr_info("result: %d\n", result);
    return 0;
}
```

```bash
# See all exported symbols:
cat /proc/kallsyms | grep ' T '   # Text (code) symbols
cat /proc/kallsyms | grep my_function

# Module dependencies (from exported symbols):
# When upper.ko needs layer.ko:
sudo modprobe upper   # Automatically loads layer first (if dependency recorded in Module.symvers)

# Manual order:
sudo insmod layer.ko
sudo insmod upper.ko

# Check dependencies:
modinfo upper.ko | grep depends
# depends: layer
```

### 5.1 `Module.symvers`

When you compile a kernel, it generates `Module.symvers` at the top of the build tree. This file records all exported symbols and their CRC values. When compiling an out-of-tree module, Kbuild uses this to verify symbol compatibility:

```bash
cat Module.symvers | head -5
# 0x12345678	my_function	layer	EXPORT_SYMBOL
# 0xdeadbeef	some_kernel_func	vmlinux	EXPORT_SYMBOL_GPL
```

If the CRC doesn't match (e.g., function signature changed between kernel versions), loading fails with "disagrees about version of symbol."

---

## 6. Module Loading Process

### 6.1 What Happens During `insmod`

```
User calls: insmod mymodule.ko

1. Userspace: insmod reads .ko file into memory

2. Calls: finit_module(fd, params, 0) syscall
   OR:    init_module(buf, len, params)

3. Kernel: kernel/module/main.c :: load_module()
   a. Allocate memory for module metadata
   b. Copy module ELF from user space
   c. Verify ELF format (magic, sections)
   d. Check module license, version, etc.
   e. Find and verify exported symbol versions (CRCs)
   f. Allocate module memory regions:
      - Module core (text, data) in module_alloc() area
      - Init section (text, data) separately
   g. Relocate ELF: fix up symbol references
   h. Apply module parameters from command line
   i. Call security hooks: security_module_load()
   j. If signed: verify signature against trusted keys
   k. Add to module list (modules linked list)
   l. Call module's init function
```

### 6.2 Memory Regions for Modules

On x86_64, modules are loaded into a special memory region distinct from the main kernel:

```
Kernel address space:
ffffffffc0000000 - fffffffffffff000  ← Module text area
                                      (~1GB) vmalloc area for modules

# Why separate? Because:
# 1. call instructions from kernel to module must be within ±2GB
# 2. Module memory is separate from kmalloc/vmalloc for easier tracking
```

```bash
# See module memory addresses:
cat /proc/modules
# hello 16384 0 - Live 0xffffffffc0a00000

cat /sys/module/hello/sections/.text
# 0xffffffffc0a00000
```

### 6.3 ELF Relocation

A `.ko` file is a partially-linked ELF object. References to kernel symbols are unresolved. The loader applies relocations at load time:

```bash
# See relocations in a module:
readelf -r mymodule.ko | head -20
# Relocation section '.rela.text':
# Offset           Info     Type             Sym. Value  Sym. Name + Addend
# 000000000012 000d00000004 R_X86_64_PLT32   0000000000 printk - 4

# After loading, the kernel fills in printk's actual address.
```

---

## 7. `modprobe` vs `insmod`

```bash
# insmod: dumb loader — loads exactly the specified .ko
sudo insmod /path/to/module.ko

# modprobe: smart loader — handles dependencies
sudo modprobe e1000e         # Automatically loads all deps first
sudo modprobe -r e1000e      # Remove module and unused deps

# modprobe config:
cat /etc/modprobe.d/blacklist.conf  # Blacklisted modules
cat /etc/modprobe.conf              # Module options and aliases

# Add permanent module option:
echo 'options e1000e SmartPowerDownEnable=1' > /etc/modprobe.d/e1000e.conf

# Map device to module (aliases):
cat /lib/modules/$(uname -r)/modules.alias | grep e1000e
# alias pci:v00008086d00001533sv*sd*bc*sc*i* e1000e
```

### 7.1 `depmod` — Module Dependency Graph

```bash
# Regenerate module dependency files (after installing new modules):
sudo depmod -a

# This reads all .ko files and creates:
ls /lib/modules/$(uname -r)/modules.*
# modules.dep          - dependency map
# modules.dep.bin      - binary version
# modules.alias        - PCI/USB/etc ID → module name
# modules.alias.bin
# modules.symbols      - symbol → module map
# modules.order        - load order hints
# modules.builtin      - modules built into vmlinux
```

---

## 8. Module Reference Counting

Modules must not be unloaded while something is using them:

```c
// A module automatically can't be unloaded if:
// 1. Its refcount is nonzero
// 2. Other modules depend on it

// Manually increment/decrement refcount:
try_module_get(THIS_MODULE)   // Increment — prevents unload
module_put(THIS_MODULE)       // Decrement

// Common pattern in device drivers:
static int device_open(struct inode *inode, struct file *file)
{
    if (!try_module_get(THIS_MODULE))
        return -ENODEV;  // Module being removed
    // ... open device ...
    return 0;
}

static int device_release(struct inode *inode, struct file *file)
{
    module_put(THIS_MODULE);  // Allow unload again
    return 0;
}

// Check refcount:
lsmod | grep mymodule
# mymodule    16384  2     ← "2" = reference count
```

---

## 9. A Practical Module: Kernel Character Device

This is a fundamental pattern — exposing kernel functionality via a `/dev/` file:

```c
// mychardev.c — A complete character device module
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/slab.h>
#include <linux/mutex.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Example character device");

#define DEVICE_NAME "mychardev"
#define CLASS_NAME  "mychardev"
#define BUF_SIZE    4096

static dev_t dev_num;          // Major:minor number
static struct cdev my_cdev;    // Character device structure
static struct class *my_class; // Device class (/sys/class/...)
static struct device *my_dev;  // Device (/dev/mychardev)

// Internal buffer and protection:
static char *kernel_buf;
static size_t buf_pos;
static DEFINE_MUTEX(buf_mutex);

// --- File Operations ---

static int mychardev_open(struct inode *inode, struct file *file)
{
    pr_info("mychardev: opened\n");
    if (!try_module_get(THIS_MODULE))
        return -ENODEV;
    return 0;
}

static int mychardev_release(struct inode *inode, struct file *file)
{
    pr_info("mychardev: closed\n");
    module_put(THIS_MODULE);
    return 0;
}

static ssize_t mychardev_read(struct file *file, char __user *ubuf,
                               size_t len, loff_t *ppos)
{
    ssize_t bytes_read;
    
    if (mutex_lock_interruptible(&buf_mutex))
        return -EINTR;
    
    // Nothing to read
    if (buf_pos == 0) {
        mutex_unlock(&buf_mutex);
        return 0;  // EOF
    }
    
    // Limit read size
    len = min(len, buf_pos);
    
    if (copy_to_user(ubuf, kernel_buf, len)) {
        mutex_unlock(&buf_mutex);
        return -EFAULT;
    }
    
    // Shift remaining data to front (simple ring buffer alternative)
    memmove(kernel_buf, kernel_buf + len, buf_pos - len);
    buf_pos -= len;
    bytes_read = len;
    
    mutex_unlock(&buf_mutex);
    return bytes_read;
}

static ssize_t mychardev_write(struct file *file, const char __user *ubuf,
                                size_t len, loff_t *ppos)
{
    if (mutex_lock_interruptible(&buf_mutex))
        return -EINTR;
    
    // Check space
    if (len > BUF_SIZE - buf_pos) {
        mutex_unlock(&buf_mutex);
        return -ENOSPC;
    }
    
    if (copy_from_user(kernel_buf + buf_pos, ubuf, len)) {
        mutex_unlock(&buf_mutex);
        return -EFAULT;
    }
    
    buf_pos += len;
    mutex_unlock(&buf_mutex);
    return len;
}

static long mychardev_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    switch (cmd) {
    case 0x1234:  // Clear buffer
        mutex_lock(&buf_mutex);
        buf_pos = 0;
        mutex_unlock(&buf_mutex);
        return 0;
    default:
        return -ENOTTY;
    }
}

// The ops table: function pointers for each file operation
static const struct file_operations mychardev_fops = {
    .owner          = THIS_MODULE,
    .open           = mychardev_open,
    .release        = mychardev_release,
    .read           = mychardev_read,
    .write          = mychardev_write,
    .unlocked_ioctl = mychardev_ioctl,
    // Optional: .llseek, .poll, .mmap, .fasync, ...
};

// --- Init / Exit ---

static int __init mychardev_init(void)
{
    int err;
    
    // Allocate buffer
    kernel_buf = kzalloc(BUF_SIZE, GFP_KERNEL);
    if (!kernel_buf)
        return -ENOMEM;
    
    // Allocate a major:minor number pair
    err = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (err < 0) {
        pr_err("mychardev: failed to allocate device number\n");
        goto err_free_buf;
    }
    pr_info("mychardev: got major=%d minor=%d\n", MAJOR(dev_num), MINOR(dev_num));
    
    // Initialize and add the cdev structure
    cdev_init(&my_cdev, &mychardev_fops);
    my_cdev.owner = THIS_MODULE;
    err = cdev_add(&my_cdev, dev_num, 1);
    if (err < 0) {
        pr_err("mychardev: failed to add cdev\n");
        goto err_unreg_region;
    }
    
    // Create device class (appears in /sys/class/)
    my_class = class_create(CLASS_NAME);
    if (IS_ERR(my_class)) {
        err = PTR_ERR(my_class);
        goto err_del_cdev;
    }
    
    // Create device file (appears as /dev/mychardev, handled by udev)
    my_dev = device_create(my_class, NULL, dev_num, NULL, DEVICE_NAME);
    if (IS_ERR(my_dev)) {
        err = PTR_ERR(my_dev);
        goto err_destroy_class;
    }
    
    pr_info("mychardev: initialized. /dev/%s created\n", DEVICE_NAME);
    return 0;

err_destroy_class:
    class_destroy(my_class);
err_del_cdev:
    cdev_del(&my_cdev);
err_unreg_region:
    unregister_chrdev_region(dev_num, 1);
err_free_buf:
    kfree(kernel_buf);
    return err;
}

static void __exit mychardev_exit(void)
{
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    kfree(kernel_buf);
    pr_info("mychardev: removed\n");
}

module_init(mychardev_init);
module_exit(mychardev_exit);
```

```bash
# Test it:
make
sudo insmod mychardev.ko
ls -la /dev/mychardev       # Created by udev
echo "hello kernel" > /dev/mychardev
cat /dev/mychardev          # Reads back "hello kernel"
sudo rmmod mychardev
```

---

## 10. Error Handling Patterns in Modules

The kernel uses goto-based cleanup — the only clean way to handle multiple failure points without resource leaks:

```c
static int __init mymodule_init(void)
{
    int ret;
    
    ret = allocate_resource_a();
    if (ret) goto fail_a;
    
    ret = allocate_resource_b();
    if (ret) goto fail_b;
    
    ret = allocate_resource_c();
    if (ret) goto fail_c;
    
    return 0;   // Success
    
// Cleanup in REVERSE order of allocation:
fail_c:
    free_resource_b();
fail_b:
    free_resource_a();
fail_a:
    return ret;
}
```

Never use nested if-else or early returns that skip cleanup — you'll leak resources.

### 10.1 Kernel Error Pointer Convention

Some functions return either a valid pointer or an error code embedded in a pointer:

```c
struct myobj *ptr = get_or_error();
if (IS_ERR(ptr)) {
    int err = PTR_ERR(ptr);   // Extract the errno
    pr_err("failed: %d\n", err);
    return err;
}
// Use ptr normally...
```

```c
// How ERR_PTR works:
// Kernel memory is never mapped to the last 4KB of address space
// (0xffffffffffffff00 to 0xffffffffffffffff are "error" values)
// So these are unambiguous:

#define IS_ERR(ptr) (((unsigned long)(ptr)) >= (unsigned long)-MAX_ERRNO)
#define PTR_ERR(ptr) ((long)(ptr))
#define ERR_PTR(err) ((void *)(long)(err))

// Common usage:
static struct class *my_class;
my_class = class_create("myclass");
if (IS_ERR(my_class))
    return PTR_ERR(my_class);
```

---

## 11. `kzalloc` and Memory Allocation in Modules

```c
#include <linux/slab.h>

// Allocate memory in init:
void *buf = kmalloc(size, GFP_KERNEL);   // Can sleep, for normal use
void *buf = kmalloc(size, GFP_ATOMIC);   // Cannot sleep (interrupt context)
void *buf = kzalloc(size, GFP_KERNEL);   // kmalloc + zero fill (preferred)
void *buf = vmalloc(size);               // Large allocations (not physically contiguous)

// Free:
kfree(buf);     // For kmalloc/kzalloc
vfree(buf);     // For vmalloc

// Always check for NULL:
buf = kzalloc(size, GFP_KERNEL);
if (!buf)
    return -ENOMEM;
```

GFP (Get Free Page) flags:
- `GFP_KERNEL`: normal allocation, can sleep, can reclaim. Use in process context.
- `GFP_ATOMIC`: cannot sleep. Use in interrupt context or with spinlock held.
- `GFP_DMA`: allocate from DMA zone (low memory, required for 24-bit DMA devices).
- `GFP_USER`: allocate for user-space pages.
- `__GFP_ZERO`: zero the allocated memory.
- `__GFP_NOWARN`: don't print warning if allocation fails.

---

## 12. Module Signing

With Secure Boot + kernel lockdown mode, unsigned modules cannot be loaded:

```bash
# Check if module loading is restricted:
cat /sys/kernel/security/lockdown
# integrity  ← modules must be signed
# confidentiality ← even stricter

# Generate signing key (RSA 4096-bit):
openssl req -new -nodes -utf8 -sha256 -days 3650 -batch \
    -x509 -config ./certs/x509.genkey \
    -outform PEM -out signing_key.pem \
    -keyout signing_key.pem

# Sign the module:
scripts/sign-file sha256 signing_key.pem signing_key.pem mymodule.ko

# Verify signature:
modinfo mymodule.ko | grep sig

# For distribution: enroll key in MOK (Machine Owner Key) database:
sudo mokutil --import signing_cert.pem
# Reboot, enroll in firmware menu
```

---

## 13. Debugging Modules

### 13.1 KASAN (Kernel Address Sanitizer)

```bash
# Build kernel with KASAN:
scripts/config --enable KASAN
scripts/config --set-val KASAN_MODE inline   # or outline
make -j$(nproc)

# KASAN catches:
# - Use after free
# - Out-of-bounds access (stack, heap, global)
# - Use of uninitialized memory

# When your module has a bug, you'll see in dmesg:
# ==================================================================
# BUG: KASAN: slab-out-of-bounds in mychardev_write+0x45/0x90
# Write of size 1 at addr ffff888012345678 by task bash/1234
# ...
# Allocated by task 1234:
#  kzalloc+0x...
#  mychardev_init+0x...
```

### 13.2 `PROVE_LOCKING` (lockdep)

```bash
# Enable lockdep to catch locking bugs:
scripts/config --enable PROVE_LOCKING
scripts/config --enable DEBUG_LOCKDEP
make -j$(nproc)

# Lockdep catches:
# - Lock ordering violations (potential deadlock)
# - Taking a sleeping lock in interrupt context
# - Lock held across a function that might sleep
# - Recursive locking

# Error example in dmesg:
# WARNING: possible circular locking dependency detected
# bash/1234 is trying to acquire lock:
# ...
# but task is already holding lock:
# ...
```

### 13.3 `checkpatch.pl` for Style

```bash
# Before submitting any patch, run:
scripts/checkpatch.pl --strict mymodule.c
scripts/checkpatch.pl --strict 0001-add-mymodule.patch

# Common errors:
# ERROR: do not use kmalloc(sizeof(*type)) use kmalloc_array()
# WARNING: please write a paragraph that describes the config symbol fully
# ERROR: code indent should use tabs where possible
# WARNING: line over 80 characters
```

---

## 14. The `init` and `exit` Section Lifecycle

```c
// __init: placed in .init.text section
static int __init mymodule_init(void) { ... }

// After ALL init functions run, this entire section is freed:
// dmesg shows: "Freeing unused kernel image (initmem) memory: 1234K"

// __exit: placed in .exit.text section
// For built-in code (not modules): this section is discarded at compile time
// because built-in code can never be "removed"
static void __exit mymodule_exit(void) { ... }

// __initdata: initialized data only needed during init
static int __initdata init_counter = 0;

// Combining:
static struct platform_driver __refdata my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = { .name = "mydevice" },
};
```

For modules (not built-in), `__init` and `__exit` are effectively no-ops — the module `.ko` is loaded/unloaded entirely. The `__init`/`__exit` matters for built-in code where the init section can be freed after boot.

---

## 15. Module Versioning and Compatibility

```bash
# Check kernel version a module was built for:
modinfo mymodule.ko | grep vermagic
# vermagic: 6.9.0 SMP preempt mod_unload modversions

# The vermagic string must match the running kernel EXACTLY
# (or close enough based on ABI compatibility rules)
# A module built for 6.9.0 cannot be loaded on 6.8.0

# Module symbol CRC checking (CONFIG_MODVERSIONS=y):
# Each exported symbol has a CRC generated from its prototype
# If the signature changed between builds, loading fails:
# ERROR: disagrees about version of symbol 'my_func'

# Override version check (DANGEROUS, for development only):
sudo insmod --force mymodule.ko
```

---

## 16. Autoloading Modules

The kernel automatically loads modules when needed:

```bash
# When kernel needs a module (e.g., when a PCI device is found):
# 1. Kernel calls request_module("pci:v00008086d00001533...")
# 2. This runs /sbin/modprobe with that alias
# 3. modprobe checks modules.alias, finds "e1000e"
# 4. Loads e1000e.ko

# Manual autoload test:
sudo modprobe --resolve-alias "pci:v00008086d00001533sv*sd*bc*sc*i*"
# e1000e

# Add module to autoload at boot:
echo "mymodule" > /etc/modules-load.d/mymodule.conf

# Blacklist a module (prevent autoload):
echo "blacklist nouveau" > /etc/modprobe.d/nouveau-blacklist.conf
```

---

## 17. `/proc/modules` and `/sys/module/`

```bash
# /proc/modules: live module list
cat /proc/modules
# hello 16384 0 - Live 0xffffffffc0a00000

# Format: name size refcount deps state address

# /sys/module/: per-module sysfs directory
ls /sys/module/hello/
# coresize  initsize  initstate  parameters  refcnt  sections  srcversion  taint

cat /sys/module/hello/initstate    # live = running, going = being removed
cat /sys/module/hello/refcnt       # reference count
cat /sys/module/hello/srcversion   # checksum of source

# Module sections (code/data addresses for debugging):
ls /sys/module/hello/sections/
# .text  .data  .bss  .init.text  .init.data

cat /sys/module/hello/sections/.text
# 0xffffffffc0a00000  ← Where module code lives in kernel memory
```

---

## 18. Mental Model Checkpoint

After Day 6, you should be able to:

1. Write a complete kernel module from scratch with proper error handling.
2. Explain `module_init()`/`module_exit()` and the `initcall` mechanism.
3. Use `EXPORT_SYMBOL` and `EXPORT_SYMBOL_GPL` correctly. What's the difference?
4. Use `printk` with appropriate log levels; use `pr_*` wrappers; use dynamic debug.
5. Add module parameters and expose them via sysfs.
6. Implement a complete character device with proper file operations.
7. Explain what happens when `insmod` is called — ELF loading, relocation, init call.
8. Handle errors with goto-based cleanup.
9. Understand module signing and why it exists.
10. Use KASAN and lockdep to debug module problems.

---

## Key Source Files

```bash
include/linux/module.h              # All module macros
include/linux/moduleparam.h         # module_param* macros
include/linux/fs.h                  # file_operations, cdev
include/linux/slab.h                # kmalloc, kzalloc, kfree
kernel/module/main.c                # load_module(), core loader
kernel/module/sysfs.c               # /sys/module/ implementation
lib/kobject.c                       # Base for all kernel objects
samples/                            # Official kernel module examples
Documentation/kbuild/modules.rst    # Official module build docs
```

---

## Summary

Kernel modules are ELF objects loaded into the running kernel. Key points:

- `module_init()` / `module_exit()` register the init/cleanup functions
- `EXPORT_SYMBOL` / `EXPORT_SYMBOL_GPL` export functions to other modules
- `module_param()` exposes tunable parameters via sysfs
- The loader (`insmod` → `finit_module()` syscall) handles ELF relocation, symbol resolution, and calls init
- `modprobe` adds dependency management on top of raw `insmod`
- Reference counting (`try_module_get`/`module_put`) prevents removal while in use
- Proper error handling uses goto-based cleanup in reverse allocation order

Tomorrow: the kernel's data structures — the tools used throughout all subsystems.
