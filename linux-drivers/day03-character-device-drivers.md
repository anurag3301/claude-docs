# Day 3 — Character Device Drivers: Control RPi4 ACT LED via `/dev/myled`
**Environment:** 🔵 RPi4
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [Character Devices — The Big Picture](#1-character-devices--the-big-picture)
2. [Major and Minor Numbers](#2-major-and-minor-numbers)
3. [The `file_operations` Structure](#3-the-file_operations-structure)
4. [The `cdev` Structure](#4-the-cdev-structure)
5. [Device File Creation — Static vs Dynamic](#5-device-file-creation--static-vs-dynamic)
6. [The `file` and `inode` Structures](#6-the-file-and-inode-structures)
7. [User-Kernel Data Transfer](#7-user-kernel-data-transfer)
8. [RPi4 GPIO Architecture](#8-rpi4-gpio-architecture)
9. [Complete LED Driver Implementation](#9-complete-led-driver-implementation)
10. [Hands-On Lab](#10-hands-on-lab)
11. [Interview Questions](#11-interview-questions)

---

## 1. Character Devices — The Big Picture

A character device is the simplest and most common driver type. It presents a device to user space as a **file** in `/dev/` that can be opened, read, written, and closed — just like a regular file, but the data flows to/from hardware.

```
User Space Program
    │
    │  open("/dev/myled", O_WRONLY)  → returns fd
    │  write(fd, "1", 1)             → turns LED on
    │  write(fd, "0", 1)             → turns LED off
    │  close(fd)
    │
    ▼  [system call boundary — CPU privilege level change]
    │
VFS Layer (fs/char_dev.c)
    │  Looks up inode for /dev/myled
    │  Finds major:minor = 240:0
    │  Looks up driver registered for major 240
    │  Calls driver's file_operations->open()
    │
    ▼
Your Kernel Driver
    │  open()   → allocate state, grab GPIO
    │  write()  → parse '0'/'1', set GPIO high/low
    │  read()   → return current LED state
    │  release()→ release GPIO
    │
    ▼
GPIO Hardware (BCM2711 GPIO controller on RPi4)
```

The key insight: **everything between the user-space `write()` call and the hardware pin being set high is your driver's job**.

---

## 2. Major and Minor Numbers

Every character (and block) device is identified by two numbers:

### Major Number
Identifies **which driver** handles this device. The kernel uses it to look up the registered driver. Examples:
- Major 1: `mem` (memory devices — `/dev/null`, `/dev/zero`, `/dev/random`)
- Major 4: `tty` (TTY devices)
- Major 5: `tty` alternates (`/dev/tty`, `/dev/console`)
- Major 188: USB serial converters
- Major 240-254: Reserved for local/experimental use (use these in development)

### Minor Number
Identifies **which specific instance** of that device within the driver. The driver interprets this itself. Examples:
- `/dev/ttyS0` → major=4, minor=64 (first serial port)
- `/dev/ttyS1` → major=4, minor=65 (second serial port)
- `/dev/sda` → major=8, minor=0 (first SCSI disk)
- `/dev/sda1` → major=8, minor=1 (first partition of first disk)

### `dev_t` — The Device Number Type

```c
#include <linux/types.h>
#include <linux/kdev_t.h>

dev_t dev;  // 32-bit value encoding major+minor

// Pack major + minor into dev_t
dev = MKDEV(major, minor);

// Extract from dev_t
unsigned int major = MAJOR(dev);
unsigned int minor = MINOR(dev);
```

On Linux: upper 12 bits = major (0-4095), lower 20 bits = minor (0-1048575).

### Registering a Character Device Number

**Static allocation** (you pick the major number):
```c
int register_chrdev_region(dev_t first, unsigned count, const char *name);
// first = MKDEV(240, 0) — starting device number
// count = number of minors to reserve
// name = appears in /proc/devices
```

**Dynamic allocation** (kernel picks an available major — preferred):
```c
int alloc_chrdev_region(dev_t *dev, unsigned baseminor,
                        unsigned count, const char *name);
// dev — output: kernel fills in the allocated dev_t
// baseminor — usually 0
// count — number of minors needed
// name — appears in /proc/devices
```

**Why dynamic is preferred:** Static allocation requires you to know which major numbers are free. In development, 240+ is usually available. But in upstream kernel code, you must use dynamic allocation to avoid conflicts with other drivers. Dynamic allocation means you don't need a globally unique number.

**Unregistering:**
```c
void unregister_chrdev_region(dev_t first, unsigned count);
```

---

## 3. The `file_operations` Structure

`struct file_operations` is a table of function pointers — the driver's interface to the VFS. Defined in `include/linux/fs.h`.

```c
struct file_operations {
    struct module *owner;       // THIS_MODULE — for reference counting
    
    // Position/seek operations
    loff_t (*llseek)(struct file *, loff_t, int);
    
    // Read operations
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    
    // Vectored I/O
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    
    // I/O multiplexing — for select()/poll()/epoll()
    __poll_t (*poll)(struct file *, struct poll_table_struct *);
    
    // Device-specific commands
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    
    // Memory mapping
    int (*mmap)(struct file *, struct vm_area_struct *);
    
    // Open/close
    int (*open)(struct inode *, struct file *);
    int (*flush)(struct file *, fl_owner_t id);
    int (*release)(struct inode *, struct file *);
    
    // fsync/fasync
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);
    
    // ... many more
};
```

### Unimplemented Operations

You don't implement every operation. Set unimplemented fields to `NULL` — the VFS handles null pointers gracefully:
- `NULL` for `llseek` → default seek behavior (or `ESPIPE` if the device doesn't support seeking)
- `NULL` for `read` → returns `-EINVAL` to user space
- `NULL` for `write` → returns `-EINVAL` to user space
- `NULL` for `poll` → device is always readable/writable (select returns immediately)
- `NULL` for `release` → no cleanup needed on close

### Designated Initializer Syntax

Always use C99 designated initializers for `file_operations`:

```c
static const struct file_operations myled_fops = {
    .owner   = THIS_MODULE,
    .open    = myled_open,
    .release = myled_release,
    .read    = myled_read,
    .write   = myled_write,
};
```

This is safe against kernel API additions — if new fields are added to `file_operations`, your struct compiles fine with them initialized to `NULL`.

---

## 4. The `cdev` Structure

`struct cdev` is the kernel's representation of a character device. Defined in `include/linux/cdev.h`.

```c
struct cdev {
    struct kobject kobj;           // embedded kobject for reference counting
    struct module *owner;          // THIS_MODULE
    const struct file_operations *ops;  // your file_operations
    struct list_head list;         // linked into kernel's cdev list
    dev_t dev;                     // device number (major + minor)
    unsigned int count;            // number of minor numbers
};
```

### Full cdev Lifecycle

```c
// 1. Allocate a cdev (dynamic — preferred)
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &myled_fops;
my_cdev->owner = THIS_MODULE;

// OR: embed cdev in your device struct (even better)
struct myled_dev {
    struct cdev cdev;      // embedded — no separate allocation
    int gpio_pin;
    bool state;
    // ... your device state
};

// Initialize the embedded cdev:
cdev_init(&myled.cdev, &myled_fops);
myled.cdev.owner = THIS_MODULE;

// 2. Register cdev with the kernel
// This makes the device accessible — do this LAST in probe/init
int ret = cdev_add(&myled.cdev, dev_num, 1);
if (ret < 0) {
    // handle error
}

// 3. Unregister (do this FIRST in cleanup/remove)
cdev_del(&myled.cdev);
```

**Critical ordering rule:** `cdev_add()` must be the **last** step in initialization (after all other resources are set up), and `cdev_del()` must be the **first** step in cleanup. Once `cdev_add()` is called, user space can open the device immediately — your driver must be fully ready.

---

## 5. Device File Creation — Static vs Dynamic

### Static — Manual `mknod`

```bash
# Create device file manually (educational, not for production)
sudo mknod /dev/myled c 240 0
# c = character device
# 240 = major number
# 0 = minor number

# Set permissions
sudo chmod 666 /dev/myled
```

### Dynamic — Using `class` and `device_create`

Production drivers create device files automatically using the device class mechanism, which triggers **udev** to create the `/dev/` entry:

```c
#include <linux/device.h>
#include <linux/device/class.h>

// Global state
static struct class *myled_class;
static struct device *myled_device;

// In init function:
// 1. Create a device class (appears in /sys/class/)
myled_class = class_create("myled");
if (IS_ERR(myled_class)) {
    ret = PTR_ERR(myled_class);
    goto err_class;
}

// 2. Create the device (triggers udev to create /dev/myled)
myled_device = device_create(myled_class, NULL, dev_num,
                             NULL, "myled");
if (IS_ERR(myled_device)) {
    ret = PTR_ERR(myled_device);
    goto err_device;
}

// In exit function (reverse order):
device_destroy(myled_class, dev_num);
class_destroy(myled_class);
```

This creates:
- `/sys/class/myled/myled/` — sysfs entry
- `/dev/myled` — device file (created by udev)

### IS_ERR / PTR_ERR / ERR_PTR Pattern

Many kernel functions return pointers that can encode errors. Instead of returning `NULL` on error (which loses the error code), they encode the negative errno into the pointer using the top of the address space (which is always unmapped):

```c
// Function returns ERR_PTR on failure
myled_class = class_create("myled");

// Check for error
if (IS_ERR(myled_class)) {
    // Extract the error code
    int err = PTR_ERR(myled_class);
    pr_err("class_create failed: %d\n", err);
    return err;
}

// To create an error pointer yourself:
return ERR_PTR(-ENOMEM);
```

Values from `-1` to `-4095` (as pointers, near `0xfffff...`) are recognized as error pointers. `IS_ERR()` checks if the pointer is in this error range.

---

## 6. The `file` and `inode` Structures

When a user calls `open("/dev/myled", ...)`, the kernel creates two structures:

### `struct inode` — The File's Identity

`inode` represents the **file's permanent identity** in the filesystem. For device files, the key fields are:

```c
struct inode {
    umode_t          i_mode;    // file type + permissions (e.g., S_IFCHR | 0666)
    dev_t            i_rdev;    // device number (major + minor) for device files
    struct cdev     *i_cdev;    // pointer to our cdev (set by kernel on open)
    // ... many more fields
};
```

In your `open()` callback:
```c
static int myled_open(struct inode *inode, struct file *filp)
{
    // Get your device struct from the embedded cdev
    struct myled_dev *dev = container_of(inode->i_cdev,
                                         struct myled_dev, cdev);
    // Store it in filp->private_data for other callbacks to use
    filp->private_data = dev;
    return 0;
}
```

### `struct file` — One Open Instance

`file` represents **one open file descriptor**. Multiple processes can open the same device, each getting their own `struct file`.

```c
struct file {
    struct path      f_path;      // path to the file
    struct inode    *f_inode;     // the inode (permanent identity)
    const struct file_operations *f_op;  // our file_operations
    
    unsigned int     f_flags;     // flags passed to open(): O_RDONLY, O_NONBLOCK...
    fmode_t          f_mode;      // FMODE_READ, FMODE_WRITE
    loff_t           f_pos;       // current file position (for seek)
    
    void            *private_data; // YOUR data — use this to pass state between ops
    
    // ... more fields
};
```

`private_data` is the critical field. Your `open()` sets it to point to your device's state structure. Then `read()`, `write()`, `release()` all receive the same `struct file *` and can retrieve the state via `filp->private_data`.

### The `container_of` Macro

This is one of the most important macros in the Linux kernel:

```c
container_of(ptr, type, member)
```

Given a pointer to a **member** of a struct, it returns a pointer to the **containing struct**. The magic is pure C arithmetic:

```c
struct myled_dev {
    int gpio_pin;
    struct cdev cdev;   // member at some offset within myled_dev
    bool state;
};

// In open():
struct myled_dev *dev = container_of(inode->i_cdev, struct myled_dev, cdev);
//                                   ^ptr to member  ^type of container  ^member name
```

Internally it's: `(struct myled_dev *)((char *)ptr - offsetof(struct myled_dev, cdev))`

This pattern avoids separate allocations and is used throughout the kernel.

---

## 7. User-Kernel Data Transfer

**You cannot directly dereference user-space pointers in kernel code.** This is a fundamental rule.

### Why Not?

1. The user-space pointer is a virtual address in the **user's address space**. It may not be mapped in kernel context.
2. The user may pass a bad pointer (NULL, garbage) to crash the kernel.
3. The page may not be present (swapped out) — dereferencing would fault.
4. Security: without checking, a user could pass a kernel address and read kernel memory.

### Safe Transfer Functions

```c
#include <linux/uaccess.h>

// Copy FROM user space TO kernel space
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
// Returns: number of bytes NOT copied (0 = success, non-zero = fault occurred)

// Copy FROM kernel space TO user space
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
// Returns: number of bytes NOT copied (0 = success)

// Optimized single-value transfer (for simple types)
int get_user(x, ptr);   // like: x = *ptr (from user space)
int put_user(x, ptr);   // like: *ptr = x (to user space)
```

The `__user` annotation is a sparse checker hint — it marks the pointer as belonging to user space. The kernel compiles fine without it, but `sparse` (a kernel static analysis tool) will warn if you try to use a `__user` pointer without proper access functions.

### Example: write() callback

```c
static ssize_t myled_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *f_pos)
{
    struct myled_dev *dev = filp->private_data;
    char kbuf[4];  // small buffer on kernel stack (fine for small amounts)
    
    if (count > sizeof(kbuf) - 1)
        count = sizeof(kbuf) - 1;
    
    // Safe copy from user space
    if (copy_from_user(kbuf, buf, count))
        return -EFAULT;  // bad user pointer
    
    kbuf[count] = '\0';
    
    // Parse and act
    if (kbuf[0] == '1') {
        gpio_set_value(dev->gpio_pin, 1);
        dev->state = true;
    } else if (kbuf[0] == '0') {
        gpio_set_value(dev->gpio_pin, 0);
        dev->state = false;
    } else {
        return -EINVAL;
    }
    
    return count;  // return bytes written — important!
}
```

### Example: read() callback

```c
static ssize_t myled_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct myled_dev *dev = filp->private_data;
    char state_str[3];  // "0\n" or "1\n"
    int len;
    
    // Implement simple EOF for sequential reads
    if (*f_pos > 0)
        return 0;  // EOF — reader got the data already
    
    len = snprintf(state_str, sizeof(state_str), "%d\n", dev->state ? 1 : 0);
    
    if (count < len)
        return -EINVAL;
    
    // Safe copy to user space
    if (copy_to_user(buf, state_str, len))
        return -EFAULT;
    
    *f_pos += len;  // advance position
    return len;     // return bytes read
}
```

---

## 8. RPi4 GPIO Architecture

Before writing the LED driver, understand how RPi4's GPIO system works.

### BCM2711 GPIO Overview

The RPi4 uses Broadcom's BCM2711 SoC. It has 58 GPIO lines (GPIO0–GPIO57). Each GPIO is controlled by registers in the BCM2711's GPIO peripheral block, memory-mapped at physical address `0xFE200000`.

### The ACT LED

The RPi4's green ACT LED (activity LED) is connected to GPIO42 (or GPIO47 on some board revisions). However, the kernel's LED subsystem already manages this LED. We'll access it via the kernel's LED API rather than raw GPIO manipulation, which is the proper kernel way.

Alternatively, we'll use a user-accessible GPIO (e.g., GPIO17 on pin 11 of the 40-pin header) and wire an external LED with a resistor.

### GPIO APIs in the Kernel

There are **two** GPIO APIs in the Linux kernel:

**Legacy API (integer-based, being phased out):**
```c
#include <linux/gpio.h>
gpio_request(gpio_num, "label");
gpio_direction_output(gpio_num, 0);
gpio_set_value(gpio_num, 1);
gpio_free(gpio_num);
```

**Modern descriptor-based API (preferred):**
```c
#include <linux/gpio/consumer.h>
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id, enum gpiod_flags flags);
gpiod_set_value(desc, value);
gpiod_put(desc);
```

For our driver, we'll use the legacy integer API since we're not going through Device Tree yet (that's Day 9). We'll upgrade to the descriptor API on Day 13.

### The Kernel LED Class

The kernel has a built-in LED framework that creates `/sys/class/leds/` entries. The ACT LED appears here:

```bash
# On RPi4
ls /sys/class/leds/
# ACT  PWR  (or similar names depending on RPi OS version)

# Control ACT LED directly (no driver needed)
echo 0 | sudo tee /sys/class/leds/ACT/brightness  # LED off
echo 1 | sudo tee /sys/class/leds/ACT/brightness  # LED on

# Control trigger (heartbeat, mmc, none, etc.)
echo none | sudo tee /sys/class/leds/ACT/trigger
echo heartbeat | sudo tee /sys/class/leds/ACT/trigger
```

For our driver, we'll take over an external GPIO-connected LED on GPIO17.

---

## 9. Complete LED Driver Implementation

Here is a complete, production-quality character device driver for an LED on GPIO17:

```c
// myled.c — Character device driver for LED on RPi4 GPIO17

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>         // file_operations, alloc_chrdev_region
#include <linux/cdev.h>       // cdev_init, cdev_add
#include <linux/device.h>     // class_create, device_create
#include <linux/gpio.h>       // gpio_request, gpio_set_value
#include <linux/uaccess.h>    // copy_from_user, copy_to_user
#include <linux/slab.h>       // kzalloc, kfree (not needed here but good habit)
#include <linux/string.h>     // strncmp

#define DRIVER_NAME  "myled"
#define DEVICE_NAME  "myled"
#define CLASS_NAME   "myled"

// GPIO pin connected to LED (with 220Ω resistor to GND)
// RPi4 GPIO17 = physical pin 11 on 40-pin header
#define LED_GPIO_PIN  17

// Device state — one instance
struct myled_dev {
    struct cdev cdev;     // embedded cdev
    bool led_state;       // current LED state
    int gpio_pin;         // GPIO pin number
};

// Global variables for device number, class, device
static dev_t myled_dev_num;
static struct myled_dev *myled;
static struct class *myled_class;
static struct device *myled_device_node;

// ============================================================
// File Operations
// ============================================================

static int myled_open(struct inode *inode, struct file *filp)
{
    struct myled_dev *dev;
    
    // Get our device struct from the embedded cdev using container_of
    dev = container_of(inode->i_cdev, struct myled_dev, cdev);
    
    // Store in private_data so read/write/release can access it
    filp->private_data = dev;
    
    pr_info("myled: device opened (pid=%d, comm=%s)\n",
            current->pid, current->comm);
    return 0;
}

static int myled_release(struct inode *inode, struct file *filp)
{
    pr_info("myled: device closed\n");
    return 0;
    // Don't free filp->private_data — it points to our global device struct
    // which persists across open/close cycles
}

static ssize_t myled_write(struct file *filp, const char __user *ubuf,
                            size_t count, loff_t *f_pos)
{
    struct myled_dev *dev = filp->private_data;
    char kbuf[16];
    ssize_t to_copy;
    
    // Clamp count to our buffer size
    to_copy = min(count, sizeof(kbuf) - 1);
    
    // Safe copy from user space
    if (copy_from_user(kbuf, ubuf, to_copy)) {
        pr_err("myled: copy_from_user failed\n");
        return -EFAULT;
    }
    kbuf[to_copy] = '\0';
    
    // Strip trailing newline (for echo "1" > /dev/myled)
    if (to_copy > 0 && kbuf[to_copy - 1] == '\n')
        kbuf[to_copy - 1] = '\0';
    
    // Parse command
    if (strcmp(kbuf, "1") == 0 || strcmp(kbuf, "on") == 0) {
        gpio_set_value(dev->gpio_pin, 1);
        dev->led_state = true;
        pr_info("myled: LED ON\n");
    } else if (strcmp(kbuf, "0") == 0 || strcmp(kbuf, "off") == 0) {
        gpio_set_value(dev->gpio_pin, 0);
        dev->led_state = false;
        pr_info("myled: LED OFF\n");
    } else if (strcmp(kbuf, "toggle") == 0) {
        dev->led_state = !dev->led_state;
        gpio_set_value(dev->gpio_pin, dev->led_state ? 1 : 0);
        pr_info("myled: LED toggled -> %s\n",
                dev->led_state ? "ON" : "OFF");
    } else {
        pr_warn("myled: unknown command '%s'. Use: 0, 1, on, off, toggle\n",
                kbuf);
        return -EINVAL;
    }
    
    return count;  // must return bytes consumed
}

static ssize_t myled_read(struct file *filp, char __user *ubuf,
                           size_t count, loff_t *f_pos)
{
    struct myled_dev *dev = filp->private_data;
    char state_buf[32];
    int len;
    
    // EOF handling — for tools like cat which call read until it returns 0
    if (*f_pos > 0)
        return 0;  // EOF
    
    len = snprintf(state_buf, sizeof(state_buf),
                   "LED state: %s (GPIO%d)\n",
                   dev->led_state ? "ON" : "OFF",
                   dev->gpio_pin);
    
    if (count < len)
        return -EINVAL;
    
    if (copy_to_user(ubuf, state_buf, len)) {
        pr_err("myled: copy_to_user failed\n");
        return -EFAULT;
    }
    
    *f_pos += len;  // advance position
    return len;
}

// ============================================================
// File operations table
// ============================================================
static const struct file_operations myled_fops = {
    .owner   = THIS_MODULE,
    .open    = myled_open,
    .release = myled_release,
    .read    = myled_read,
    .write   = myled_write,
};

// ============================================================
// Module Init
// ============================================================
static int __init myled_init(void)
{
    int ret;
    
    pr_info("myled: initializing LED driver\n");
    
    // ---- Step 1: Allocate device state ----
    myled = kzalloc(sizeof(*myled), GFP_KERNEL);
    if (!myled) {
        pr_err("myled: failed to allocate device state\n");
        return -ENOMEM;
    }
    myled->gpio_pin = LED_GPIO_PIN;
    myled->led_state = false;
    
    // ---- Step 2: Request GPIO ----
    ret = gpio_request(myled->gpio_pin, "myled_gpio");
    if (ret) {
        pr_err("myled: failed to request GPIO %d: %d\n",
               myled->gpio_pin, ret);
        goto err_gpio_request;
    }
    
    // Set GPIO as output, initially low (LED off)
    ret = gpio_direction_output(myled->gpio_pin, 0);
    if (ret) {
        pr_err("myled: failed to set GPIO %d direction: %d\n",
               myled->gpio_pin, ret);
        goto err_gpio_direction;
    }
    
    // ---- Step 3: Allocate device number ----
    ret = alloc_chrdev_region(&myled_dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        pr_err("myled: alloc_chrdev_region failed: %d\n", ret);
        goto err_chrdev_region;
    }
    pr_info("myled: allocated major=%d, minor=%d\n",
            MAJOR(myled_dev_num), MINOR(myled_dev_num));
    
    // ---- Step 4: Initialize and register cdev ----
    cdev_init(&myled->cdev, &myled_fops);
    myled->cdev.owner = THIS_MODULE;
    
    ret = cdev_add(&myled->cdev, myled_dev_num, 1);
    if (ret < 0) {
        pr_err("myled: cdev_add failed: %d\n", ret);
        goto err_cdev_add;
    }
    
    // ---- Step 5: Create device class ----
    myled_class = class_create(CLASS_NAME);
    if (IS_ERR(myled_class)) {
        ret = PTR_ERR(myled_class);
        pr_err("myled: class_create failed: %d\n", ret);
        goto err_class;
    }
    
    // ---- Step 6: Create device node (triggers udev → /dev/myled) ----
    myled_device_node = device_create(myled_class, NULL,
                                       myled_dev_num, NULL,
                                       DEVICE_NAME);
    if (IS_ERR(myled_device_node)) {
        ret = PTR_ERR(myled_device_node);
        pr_err("myled: device_create failed: %d\n", ret);
        goto err_device;
    }
    
    pr_info("myled: driver initialized. /dev/%s ready\n", DEVICE_NAME);
    pr_info("myled: echo '1' > /dev/myled  to turn LED ON\n");
    pr_info("myled: echo '0' > /dev/myled  to turn LED OFF\n");
    return 0;
    
    // Error handling — unwind in reverse order
err_device:
    class_destroy(myled_class);
err_class:
    cdev_del(&myled->cdev);
err_cdev_add:
    unregister_chrdev_region(myled_dev_num, 1);
err_chrdev_region:
err_gpio_direction:
    gpio_free(myled->gpio_pin);
err_gpio_request:
    kfree(myled);
    return ret;
}

// ============================================================
// Module Exit
// ============================================================
static void __exit myled_exit(void)
{
    pr_info("myled: removing driver\n");
    
    // Turn LED off on unload
    gpio_set_value(myled->gpio_pin, 0);
    
    // Reverse order of init — cdev_del first!
    device_destroy(myled_class, myled_dev_num);
    class_destroy(myled_class);
    cdev_del(&myled->cdev);
    unregister_chrdev_region(myled_dev_num, 1);
    gpio_free(myled->gpio_pin);
    kfree(myled);
    
    pr_info("myled: driver removed\n");
}

module_init(myled_init);
module_exit(myled_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux Driver Course");
MODULE_DESCRIPTION("RPi4 LED character device driver");
MODULE_VERSION("1.0");
```

---

## 10. Hands-On Lab

### Hardware Setup

```
RPi4 Pin 11 (GPIO17) ──┤220Ω├── LED Anode (+)
RPi4 Pin 9  (GND)    ────────── LED Cathode (-)
```

The 220Ω resistor limits current to ~15mA (safe for GPIO and LED).

### Step 1 — Write and Build

Save the driver above as `myled.c`. Create the Makefile:

```makefile
ARCH := arm64
CROSS_COMPILE := aarch64-linux-gnu-
KDIR := $(HOME)/rpi-kernel

obj-m += myled.o
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean
```

```bash
make
scp myled.ko pi@<RPI_IP>:/home/pi/
```

### Step 2 — Load and Test

```bash
# On RPi4
sudo insmod myled.ko
dmesg | tail -10  # Check for initialization messages

# Verify /dev/myled was created
ls -l /dev/myled
# crw------- 1 root root 240, 0 Jun 1 12:00 /dev/myled

# Fix permissions for testing
sudo chmod 666 /dev/myled

# Turn LED ON
echo "1" > /dev/myled

# Turn LED OFF
echo "0" > /dev/myled

# Read state
cat /dev/myled

# Toggle
echo "toggle" > /dev/myled

# Use with bash loop — blink!
while true; do
    echo "1" > /dev/myled; sleep 0.5
    echo "0" > /dev/myled; sleep 0.5
done
```

### Step 3 — Write a Userspace Control Program

```c
// led_control.c — userspace program to control myled driver
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define DEVICE "/dev/myled"

int main(int argc, char *argv[])
{
    int fd;
    char buf[64];
    ssize_t n;
    
    if (argc < 2) {
        fprintf(stderr, "Usage: %s [0|1|on|off|toggle|status]\n", argv[0]);
        return 1;
    }
    
    fd = open(DEVICE, O_RDWR);
    if (fd < 0) {
        perror("open " DEVICE);
        return 1;
    }
    
    if (strcmp(argv[1], "status") == 0) {
        n = read(fd, buf, sizeof(buf) - 1);
        if (n < 0) { perror("read"); close(fd); return 1; }
        buf[n] = '\0';
        printf("%s", buf);
    } else {
        n = write(fd, argv[1], strlen(argv[1]));
        if (n < 0) { perror("write"); close(fd); return 1; }
        printf("Command '%s' sent successfully\n", argv[1]);
    }
    
    close(fd);
    return 0;
}
```

```bash
# Compile natively on RPi4
gcc -o led_control led_control.c
./led_control 1
./led_control 0
./led_control toggle
./led_control status
```

### Step 4 — Inspect the Driver's Kernel Structures

```bash
# On RPi4 with driver loaded
# Check /proc/devices for our major number
cat /proc/devices | grep myled

# Check sysfs
ls /sys/class/myled/
ls /sys/class/myled/myled/

# Check GPIO in sysfs
ls /sys/class/gpio/
cat /sys/class/gpio/gpio17/direction  # should be "out"
cat /sys/class/gpio/gpio17/value      # 0 or 1

# Check cdev in /proc/modules
cat /proc/modules | grep myled
```

### Step 5 — Trigger Errors Intentionally

```bash
# Try to send invalid command
echo "blah" > /dev/myled
# Should return -EINVAL — check dmesg

# Open from two processes simultaneously
cat /dev/myled &
echo "1" > /dev/myled
# Both should work — char drivers allow multiple opens by default

# Test bad permissions
sudo chmod 000 /dev/myled
echo "1" > /dev/myled
# Should fail with "Permission denied"
sudo chmod 666 /dev/myled
```

---

## 11. Interview Questions

---

**Q1. Explain the complete journey of `write(fd, "1", 1)` on a `/dev/myled` device from user space to hardware.**

**Answer:** 
1. User process calls `write(fd, "1", 1)` — this is a C library wrapper that invokes the `write` syscall (syscall #1 on x86-64)
2. CPU executes `syscall` instruction, drops to Ring 0, jumps to `entry_SYSCALL_64`
3. Kernel's `sys_write()` is called with the fd, buffer pointer, and count
4. `sys_write` → `ksys_write()` → `vfs_write()` — VFS layer
5. VFS looks up the `struct file` for this fd from the current process's `files_struct`
6. VFS calls `file->f_op->write()` — which is our driver's `myled_write()`
7. `myled_write()` calls `copy_from_user()` to safely copy "1" from user's virtual address space into a kernel buffer
8. Driver compares the buffer, calls `gpio_set_value(17, 1)` 
9. `gpio_set_value` → BCM2711 GPIO driver → writes to GPIO output set register at MMIO address `0xFE20001C`
10. BCM2711 hardware sets GPIO17 pin high → LED illuminates
11. `myled_write` returns `count` (1)
12. VFS propagates return value to `sys_write`
13. `sys_write` returns 1 to user space via `rax` register
14. `sysret` instruction, CPU back to Ring 3
15. C library `write()` returns 1 to your program

---

**Q2. What is `container_of` and why is it used in kernel drivers?**

**Answer:** `container_of(ptr, type, member)` is a macro that returns a pointer to a containing structure given a pointer to one of its members. It computes: `(type *)((char *)ptr - offsetof(type, member))`.

It's used because the kernel's object model embeds generic structures inside driver-specific structures. For example:
- `struct cdev` is embedded inside our `struct myled_dev`
- When the kernel calls our `open()`, it passes `inode->i_cdev` — a pointer to the `struct cdev`
- We need to get back to our `myled_dev` to access our state (gpio_pin, led_state, etc.)
- `container_of(inode->i_cdev, struct myled_dev, cdev)` does exactly this

This pattern avoids separate dynamic allocations for the embedded structure and is used extensively: `container_of` with `list_head` for linked lists, with `kobject` in the device model, with `work_struct` in workqueues, etc.

---

**Q3. Why must you never dereference a `__user` pointer directly in kernel code?**

**Answer:** Four reasons:

1. **Address space mismatch**: User-space virtual addresses are valid in the process's MMU page tables but not necessarily in kernel context. The kernel runs with a different active page table on some architectures (e.g., with KPTI enabled, user mappings are entirely absent in kernel mode).

2. **Security**: A user process could pass a kernel virtual address instead of a user address. Direct dereference would allow reading/writing kernel memory — a privilege escalation vulnerability. `copy_from_user` validates the address is in user space.

3. **Fault handling**: User pages can be swapped out. Accessing a swapped page from kernel context with interrupts disabled or while holding a spinlock would be catastrophic. `copy_from_user` uses a special fault handler that returns `-EFAULT` if the page isn't accessible.

4. **SMAP/SMEP**: Modern x86 CPUs enforce **SMAP (Supervisor Mode Access Prevention)** which causes a fault if kernel code accesses user pages without explicitly disabling SMAP first. `copy_from_user` properly disables SMAP via `stac`/`clac` instructions around the access.

---

**Q4. What is the difference between `alloc_chrdev_region` and `register_chrdev_region`?**

**Answer:** Both register a range of character device numbers, but differ in who chooses the major number:

- `register_chrdev_region(dev, count, name)` — you provide the `dev_t` (major + starting minor). You must know in advance which major number to use. Risk: conflicts with other drivers using the same major.

- `alloc_chrdev_region(&dev, baseminor, count, name)` — kernel scans its internal `chrdevs[]` table and dynamically assigns an available major number, which it writes to `*dev`. No conflicts possible.

The dynamic version is always preferred for new drivers. The only reason to use static allocation is for compatibility with very old device files that were created with a specific major number hardcoded (e.g., device files in `/dev/` that were created before the driver was loaded and have a hardcoded major number).

---

**Q5. What happens if you call `cdev_add()` before fully initializing the device? What is the correct initialization order?**

**Answer:** `cdev_add()` makes the device immediately accessible to user space — the kernel can start routing `open()` calls to your driver from that instant. If `cdev_add()` is called before your data structures, GPIO, interrupts, or DMA are set up, a user process could `open()` the device and call `read()`/`write()` before initialization is complete, causing null pointer dereferences, use of uninitialized data, or hardware access before hardware is ready.

Correct initialization order:
1. Allocate and initialize all data structures
2. Initialize hardware (GPIO, register maps, etc.)
3. Allocate device number (`alloc_chrdev_region`)
4. Set up interrupt handlers
5. Set up DMA if needed
6. Initialize `cdev` with `cdev_init()`
7. **Last: `cdev_add()`** — now user space can access the device

Cleanup order is the reverse: `cdev_del()` first, then hardware teardown, then free resources.

---

**Q6. How do you support multiple instances of the same device with a single driver?**

**Answer:** Use the minor number to identify each instance. Allocate multiple minor numbers with `alloc_chrdev_region(..., count)` where `count` is the number of instances. Create one `cdev` per instance, each registered with `cdev_add` for its minor number.

```c
#define MAX_DEVICES 4

struct myled_dev devices[MAX_DEVICES];
dev_t base_dev;

// In init:
alloc_chrdev_region(&base_dev, 0, MAX_DEVICES, "myled");

for (int i = 0; i < MAX_DEVICES; i++) {
    devices[i].gpio_pin = gpio_pins[i];
    cdev_init(&devices[i].cdev, &myled_fops);
    cdev_add(&devices[i].cdev, base_dev + i, 1);
    device_create(myled_class, NULL, base_dev + i, NULL, "myled%d", i);
}
```

This creates `/dev/myled0`, `/dev/myled1`, etc. In `open()`, use `iminor(inode)` to get the minor number and index into your devices array:

```c
static int myled_open(struct inode *inode, struct file *filp)
{
    unsigned int minor = iminor(inode);
    filp->private_data = &devices[minor];
    return 0;
}
```

---

> **Next:** Day 4 — Extending the char driver with ioctl, poll/select, and blocking I/O with wait queues.
