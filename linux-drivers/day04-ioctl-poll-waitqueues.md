# Day 4 — ioctl, poll/select & Blocking I/O with Wait Queues
**Environment:** 🔵 RPi4
**Estimated Reading + Lab Time:** ~2 hours

---

## Table of Contents
1. [The Need for ioctl](#1-the-need-for-ioctl)
2. [Designing ioctl Commands — `_IOC` Macros](#2-designing-ioctl-commands--_ioc-macros)
3. [Implementing unlocked_ioctl](#3-implementing-unlocked_ioctl)
4. [Blocking I/O and Wait Queues](#4-blocking-io-and-wait-queues)
5. [poll() / select() / epoll() Support](#5-poll--select--epoll-support)
6. [Putting It Together — Extended LED + Button Driver](#6-putting-it-together--extended-led--button-driver)
7. [Hands-On Lab](#7-hands-on-lab)
8. [Interview Questions](#8-interview-questions)

---

## 1. The Need for ioctl

`read()` and `write()` transfer data. But some device operations don't fit that model:

- Get/set baud rate on a serial port
- Set gain on an audio ADC
- Query hardware revision
- Enable/disable specific interrupts
- Reset the device
- Set LED blink pattern

These are **control operations** — configuration, not data transfer. `ioctl()` (I/O Control) is the standard mechanism for device-specific commands that don't fit read/write.

```c
// User space
int ioctl(int fd, unsigned long request, ...);

// The third argument is optional and device-specific:
// - integer value
// - pointer to a struct (for complex data)
// - nothing (command has no argument)
```

### Why Not Just Use write()?

You could encode commands in write data (like our Day 3 driver parsing "on"/"off"). But:
1. No type safety — strings are fragile
2. Can't return data easily (would need a separate read())
3. No standard ABI between kernel and user space
4. ioctl has defined semantics for both input and output in one call

---

## 2. Designing ioctl Commands — `_IOC` Macros

ioctl command numbers are 32-bit integers, structured as fields:

```
Bit 31-30: direction  (00=none, 10=read, 01=write, 11=read+write)
Bit 29-16: size       (sizeof the argument type, 0-16383 bytes)
Bit 15-8:  type       (magic number — identifies your driver)
Bit 7-0:   number     (sequential command number within this driver)
```

The kernel provides macros to construct these:

```c
#include <linux/ioctl.h>

// Command with no argument
_IO(type, nr)

// Command that reads data FROM the kernel (kernel writes to user)
_IOR(type, nr, data_type)

// Command that writes data TO the kernel (kernel reads from user)
_IOW(type, nr, data_type)

// Command that does both
_IOWR(type, nr, data_type)
```

### Choosing a Magic Number (type)

The magic number should be unique to your driver to avoid collisions. The file `Documentation/userspace-api/ioctl/ioctl-number.rst` in the kernel source tracks allocated numbers. For development, use numbers in the 0x80-0xFF range that aren't allocated.

### Defining Your ioctl Commands

Create a header shared between your driver and userspace application:

```c
// myled_ioctl.h — shared between kernel driver and userspace

#ifndef MYLED_IOCTL_H
#define MYLED_IOCTL_H

#include <linux/ioctl.h>

#define MYLED_IOC_MAGIC  0xE1    // Choose a unique magic number

// Data structures for complex ioctls
struct myled_blink_cfg {
    unsigned int on_ms;    // LED on duration in milliseconds
    unsigned int off_ms;   // LED off duration in milliseconds
    unsigned int count;    // Number of blinks (0 = infinite)
};

struct myled_status {
    int  led_state;        // Current LED state (0 or 1)
    int  gpio_pin;         // GPIO pin number
    long toggle_count;     // How many times LED was toggled
};

// Define ioctl commands
// Simple commands — no data transfer
#define MYLED_IOC_LED_ON    _IO(MYLED_IOC_MAGIC,  0)
#define MYLED_IOC_LED_OFF   _IO(MYLED_IOC_MAGIC,  1)
#define MYLED_IOC_LED_TOGGLE _IO(MYLED_IOC_MAGIC, 2)
#define MYLED_IOC_RESET     _IO(MYLED_IOC_MAGIC,  3)

// Read status FROM kernel (kernel → user)
#define MYLED_IOC_GET_STATUS _IOR(MYLED_IOC_MAGIC, 4, struct myled_status)

// Write blink config TO kernel (user → kernel)
#define MYLED_IOC_SET_BLINK  _IOW(MYLED_IOC_MAGIC, 5, struct myled_blink_cfg)

// Read back current blink config
#define MYLED_IOC_GET_BLINK  _IOR(MYLED_IOC_MAGIC, 6, struct myled_blink_cfg)

// Max ioctl number for bounds checking
#define MYLED_IOC_MAXNR  6

#endif /* MYLED_IOCTL_H */
```

---

## 3. Implementing unlocked_ioctl

```c
static long myled_ioctl(struct file *filp, unsigned int cmd,
                         unsigned long arg)
{
    struct myled_dev *dev = filp->private_data;
    struct myled_status status;
    struct myled_blink_cfg blink_cfg;
    int ret = 0;

    // ---- Step 1: Validate the command ----

    // Check magic number matches our driver
    if (_IOC_TYPE(cmd) != MYLED_IOC_MAGIC)
        return -ENOTTY;  // "Not a typewriter" — wrong magic

    // Check command number is in range
    if (_IOC_NR(cmd) > MYLED_IOC_MAXNR)
        return -ENOTTY;

    // Check user-space pointer accessibility for _IOR/_IOW commands
    if (_IOC_DIR(cmd) & _IOC_READ) {
        // Kernel will write to user's buffer — check it's writable
        if (!access_ok((void __user *)arg, _IOC_SIZE(cmd)))
            return -EFAULT;
    }
    if (_IOC_DIR(cmd) & _IOC_WRITE) {
        // Kernel will read from user's buffer — check it's readable
        if (!access_ok((void __user *)arg, _IOC_SIZE(cmd)))
            return -EFAULT;
    }

    // ---- Step 2: Handle commands ----
    switch (cmd) {
    case MYLED_IOC_LED_ON:
        gpio_set_value(dev->gpio_pin, 1);
        dev->led_state = true;
        dev->toggle_count++;
        pr_info("myled: ioctl LED ON\n");
        break;

    case MYLED_IOC_LED_OFF:
        gpio_set_value(dev->gpio_pin, 0);
        dev->led_state = false;
        dev->toggle_count++;
        pr_info("myled: ioctl LED OFF\n");
        break;

    case MYLED_IOC_LED_TOGGLE:
        dev->led_state = !dev->led_state;
        gpio_set_value(dev->gpio_pin, dev->led_state ? 1 : 0);
        dev->toggle_count++;
        pr_info("myled: ioctl TOGGLE -> %s\n",
                dev->led_state ? "ON" : "OFF");
        break;

    case MYLED_IOC_RESET:
        gpio_set_value(dev->gpio_pin, 0);
        dev->led_state = false;
        dev->toggle_count = 0;
        memset(&dev->blink_cfg, 0, sizeof(dev->blink_cfg));
        pr_info("myled: ioctl RESET\n");
        break;

    case MYLED_IOC_GET_STATUS:
        // Fill status struct in kernel space
        status.led_state    = dev->led_state ? 1 : 0;
        status.gpio_pin     = dev->gpio_pin;
        status.toggle_count = dev->toggle_count;

        // Copy to user space
        if (copy_to_user((struct myled_status __user *)arg,
                         &status, sizeof(status)))
            return -EFAULT;
        break;

    case MYLED_IOC_SET_BLINK:
        // Copy blink config from user space
        if (copy_from_user(&blink_cfg,
                           (struct myled_blink_cfg __user *)arg,
                           sizeof(blink_cfg)))
            return -EFAULT;

        // Validate
        if (blink_cfg.on_ms == 0 || blink_cfg.off_ms == 0)
            return -EINVAL;
        if (blink_cfg.on_ms > 10000 || blink_cfg.off_ms > 10000)
            return -EINVAL;

        dev->blink_cfg = blink_cfg;
        pr_info("myled: blink config set: on=%ums off=%ums count=%u\n",
                blink_cfg.on_ms, blink_cfg.off_ms, blink_cfg.count);
        break;

    case MYLED_IOC_GET_BLINK:
        if (copy_to_user((struct myled_blink_cfg __user *)arg,
                         &dev->blink_cfg, sizeof(dev->blink_cfg)))
            return -EFAULT;
        break;

    default:
        return -ENOTTY;
    }

    return ret;
}
```

And add it to `file_operations`:

```c
static const struct file_operations myled_fops = {
    .owner          = THIS_MODULE,
    .open           = myled_open,
    .release        = myled_release,
    .read           = myled_read,
    .write          = myled_write,
    .unlocked_ioctl = myled_ioctl,
};
```

### Why `unlocked_ioctl` and not `ioctl`?

The old `ioctl` field required the **BKL (Big Kernel Lock)** to be held. The BKL was removed in kernel 2.6.36. `unlocked_ioctl` runs without any global lock — the driver is responsible for its own synchronization. There's also `compat_ioctl` for handling 32-bit userspace on a 64-bit kernel (where pointer sizes differ).

### Userspace Usage

```c
#include <sys/ioctl.h>
#include "myled_ioctl.h"

int fd = open("/dev/myled", O_RDWR);

// Simple command — no argument
ioctl(fd, MYLED_IOC_LED_ON);

// Get status
struct myled_status st;
ioctl(fd, MYLED_IOC_GET_STATUS, &st);
printf("LED: %s, pin: %d, toggles: %ld\n",
       st.led_state ? "ON" : "OFF", st.gpio_pin, st.toggle_count);

// Set blink config
struct myled_blink_cfg cfg = { .on_ms=500, .off_ms=500, .count=10 };
ioctl(fd, MYLED_IOC_SET_BLINK, &cfg);
```

---

## 4. Blocking I/O and Wait Queues

Most device reads/writes should **block** when data isn't available, rather than returning an error or spinning. This is how real devices work — `read()` on `/dev/ttyS0` blocks until serial data arrives.

### The Problem Without Blocking

```c
// Bad — busy wait burns 100% CPU
while (gpio_get_value(button_pin) == 0)
    ;  // spin forever waiting for button press

// Bad — busy polling from user space
while (1) {
    int state;
    read(fd, &state, sizeof(state));
    if (state == 1) break;
    usleep(1000);  // still wasteful
}
```

### Wait Queues — The Kernel's Blocking Mechanism

A **wait queue** is a list of sleeping processes waiting for a condition to become true. When the condition is met (e.g., button pressed, data arrived), a wakeup call unblocks all waiting processes.

```c
#include <linux/wait.h>

// Declare a wait queue head (static)
static DECLARE_WAIT_QUEUE_HEAD(button_wq);

// Or dynamic initialization:
wait_queue_head_t button_wq;
init_waitqueue_head(&button_wq);
```

### Sleeping in a Wait Queue

```c
// Wait until condition is true (interruptible — wakes on signals too)
wait_event_interruptible(queue, condition);
// Returns 0 if condition became true
// Returns -ERESTARTSYS if interrupted by a signal (like Ctrl+C)

// Wait with timeout (in jiffies — use msecs_to_jiffies() to convert)
wait_event_interruptible_timeout(queue, condition, timeout);
// Returns positive if condition met, 0 if timed out, negative if signal

// Non-interruptible (can't be killed with Ctrl+C — use sparingly)
wait_event(queue, condition);
```

### Waking Up Waiters

```c
// Wake all processes waiting on this queue
wake_up_interruptible(&button_wq);

// Wake all (even non-interruptible waiters)
wake_up(&button_wq);

// Wake only one waiter
wake_up_interruptible_sync(&button_wq);
```

### Complete Button + Wait Queue Example

```c
// In device struct:
struct myled_dev {
    struct cdev cdev;
    int led_gpio;
    int button_gpio;
    int button_irq;
    volatile bool button_pressed;  // condition for wait queue
    wait_queue_head_t button_wq;   // wait queue
    spinlock_t lock;               // protect button_pressed
    // ...
};

// IRQ handler — called in interrupt context when button is pressed
static irqreturn_t button_irq_handler(int irq, void *data)
{
    struct myled_dev *dev = data;
    unsigned long flags;
    
    spin_lock_irqsave(&dev->lock, flags);
    dev->button_pressed = true;
    spin_unlock_irqrestore(&dev->lock, flags);
    
    // Wake up anyone sleeping in read()
    wake_up_interruptible(&dev->button_wq);
    
    return IRQ_HANDLED;
}

// read() — blocks until button is pressed
static ssize_t myled_read(struct file *filp, char __user *ubuf,
                           size_t count, loff_t *f_pos)
{
    struct myled_dev *dev = filp->private_data;
    char state_str[4];
    int len;
    int ret;
    unsigned long flags;
    
    // Non-blocking mode — return immediately if no data
    if (filp->f_flags & O_NONBLOCK) {
        if (!dev->button_pressed)
            return -EAGAIN;
    } else {
        // Blocking mode — sleep until button is pressed
        ret = wait_event_interruptible(dev->button_wq,
                                       dev->button_pressed);
        if (ret == -ERESTARTSYS)
            return -EINTR;  // Signal received (e.g., Ctrl+C)
    }
    
    // Consume the event
    spin_lock_irqsave(&dev->lock, flags);
    dev->button_pressed = false;
    spin_unlock_irqrestore(&dev->lock, flags);
    
    len = snprintf(state_str, sizeof(state_str), "1\n");
    if (copy_to_user(ubuf, state_str, len))
        return -EFAULT;
    
    return len;
}
```

### `wait_event_interruptible` Internals

Under the hood, `wait_event_interruptible(wq, cond)` expands to roughly:

```c
for (;;) {
    // Add current task to wait queue
    prepare_to_wait(&wq, &wait, TASK_INTERRUPTIBLE);
    
    if (cond)   // Check condition
        break;
    
    if (signal_pending(current)) {  // Check for signals
        ret = -ERESTARTSYS;
        break;
    }
    
    schedule();  // Give up CPU — task goes to sleep
}
finish_wait(&wq, &wait);
```

`schedule()` is the key — it voluntarily yields the CPU. The task's state is set to `TASK_INTERRUPTIBLE` (can be woken by signals or explicit wakeup). When `wake_up_interruptible()` is called, it sets the task's state back to `TASK_RUNNING` and places it on the run queue. Next time the scheduler runs, the task resumes.

---

## 5. poll() / select() / epoll() Support

`poll()` and `select()` allow monitoring multiple file descriptors simultaneously. Your driver supports this by implementing the `poll` `file_operation`.

```c
#include <linux/poll.h>

static __poll_t myled_poll(struct file *filp, struct poll_table_struct *pt)
{
    struct myled_dev *dev = filp->private_data;
    __poll_t mask = 0;
    
    // Register this wait queue with the poll mechanism
    // When wake_up_interruptible(&dev->button_wq) is called,
    // poll() will re-evaluate all monitored fds
    poll_wait(filp, &dev->button_wq, pt);
    
    // Return a bitmask of events that are currently ready
    
    // EPOLLOUT | EPOLLWRNORM — device is always writable (LED always accepts writes)
    mask |= EPOLLOUT | EPOLLWRNORM;
    
    // EPOLLIN | EPOLLRDNORM — readable only if button was pressed
    if (dev->button_pressed)
        mask |= EPOLLIN | EPOLLRDNORM;
    
    return mask;
}
```

Add to `file_operations`:
```c
.poll = myled_poll,
```

### poll() Event Flags

| Flag | Meaning |
|------|---------|
| `EPOLLIN` | Data available for reading |
| `EPOLLRDNORM` | Normal data ready |
| `EPOLLOUT` | Space available for writing |
| `EPOLLWRNORM` | Normal write space |
| `EPOLLERR` | Error condition |
| `EPOLLHUP` | Hangup / device disconnected |
| `EPOLLPRI` | Urgent data available |

### User Space poll() Usage

```c
#include <poll.h>

struct pollfd fds[1];
fds[0].fd = open("/dev/myled", O_RDWR);
fds[0].events = POLLIN;  // Wait for button press

printf("Waiting for button press...\n");

int ret = poll(fds, 1, 5000);  // 5 second timeout
if (ret == 0) {
    printf("Timeout — no button press\n");
} else if (ret > 0) {
    if (fds[0].revents & POLLIN) {
        char buf[4];
        read(fds[0].fd, buf, sizeof(buf));
        printf("Button pressed!\n");
    }
}
```

### epoll for Multiple Devices

```c
// epoll is more efficient for many file descriptors
int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = led_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, led_fd, &ev);

struct epoll_event events[10];
int nfds = epoll_wait(epfd, events, 10, -1);  // wait indefinitely

for (int i = 0; i < nfds; i++) {
    if (events[i].data.fd == led_fd && events[i].events & EPOLLIN) {
        // Handle button press
    }
}
```

---

## 6. Putting It Together — Extended LED + Button Driver

Here is the full combined driver with LED control (write/ioctl) and button reading (blocking read + poll):

```c
// myled_btn.c — LED + button driver with ioctl, blocking read, poll

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/gpio.h>
#include <linux/interrupt.h>
#include <linux/uaccess.h>
#include <linux/wait.h>
#include <linux/poll.h>
#include <linux/spinlock.h>
#include <linux/slab.h>
#include "myled_ioctl.h"

#define DRIVER_NAME   "myled_btn"
#define LED_GPIO       17    // GPIO17 = pin 11
#define BUTTON_GPIO    27    // GPIO27 = pin 13 (wire button with pull-up)

struct myled_btn_dev {
    struct cdev           cdev;
    int                   led_gpio;
    int                   button_gpio;
    int                   button_irq;
    bool                  led_state;
    volatile bool         button_pressed;
    long                  toggle_count;
    struct myled_blink_cfg blink_cfg;
    wait_queue_head_t     button_wq;
    spinlock_t            lock;
};

static dev_t dev_num;
static struct myled_btn_dev *mydev;
static struct class *mydev_class;

// ---- IRQ Handler ----

static irqreturn_t button_handler(int irq, void *data)
{
    struct myled_btn_dev *dev = data;
    unsigned long flags;
    
    spin_lock_irqsave(&dev->lock, flags);
    dev->button_pressed = true;
    spin_unlock_irqrestore(&dev->lock, flags);
    
    wake_up_interruptible(&dev->button_wq);
    return IRQ_HANDLED;
}

// ---- file_operations ----

static int mydev_open(struct inode *inode, struct file *filp)
{
    filp->private_data = container_of(inode->i_cdev,
                                       struct myled_btn_dev, cdev);
    return 0;
}

static int mydev_release(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t mydev_read(struct file *filp, char __user *ubuf,
                           size_t count, loff_t *f_pos)
{
    struct myled_btn_dev *dev = filp->private_data;
    char buf[4];
    int len, ret;
    unsigned long flags;

    if (filp->f_flags & O_NONBLOCK) {
        if (!dev->button_pressed)
            return -EAGAIN;
    } else {
        ret = wait_event_interruptible(dev->button_wq,
                                       dev->button_pressed);
        if (ret)
            return -EINTR;
    }

    spin_lock_irqsave(&dev->lock, flags);
    dev->button_pressed = false;
    spin_unlock_irqrestore(&dev->lock, flags);

    len = snprintf(buf, sizeof(buf), "1\n");
    if (copy_to_user(ubuf, buf, len))
        return -EFAULT;

    return len;
}

static ssize_t mydev_write(struct file *filp, const char __user *ubuf,
                            size_t count, loff_t *f_pos)
{
    struct myled_btn_dev *dev = filp->private_data;
    char kbuf[16];
    ssize_t n = min(count, sizeof(kbuf) - 1);

    if (copy_from_user(kbuf, ubuf, n))
        return -EFAULT;
    kbuf[n] = '\0';

    if (kbuf[n-1] == '\n') kbuf[n-1] = '\0';

    if (!strcmp(kbuf, "1") || !strcmp(kbuf, "on")) {
        gpio_set_value(dev->led_gpio, 1);
        dev->led_state = true;
    } else if (!strcmp(kbuf, "0") || !strcmp(kbuf, "off")) {
        gpio_set_value(dev->led_gpio, 0);
        dev->led_state = false;
    } else if (!strcmp(kbuf, "toggle")) {
        dev->led_state = !dev->led_state;
        gpio_set_value(dev->led_gpio, dev->led_state);
        dev->toggle_count++;
    } else {
        return -EINVAL;
    }

    return count;
}

static __poll_t mydev_poll(struct file *filp, struct poll_table_struct *pt)
{
    struct myled_btn_dev *dev = filp->private_data;
    __poll_t mask = EPOLLOUT | EPOLLWRNORM;  // always writable

    poll_wait(filp, &dev->button_wq, pt);

    if (dev->button_pressed)
        mask |= EPOLLIN | EPOLLRDNORM;

    return mask;
}

static long mydev_ioctl(struct file *filp, unsigned int cmd,
                         unsigned long arg)
{
    struct myled_btn_dev *dev = filp->private_data;
    struct myled_status st;
    struct myled_blink_cfg cfg;

    if (_IOC_TYPE(cmd) != MYLED_IOC_MAGIC) return -ENOTTY;
    if (_IOC_NR(cmd) > MYLED_IOC_MAXNR)   return -ENOTTY;

    switch (cmd) {
    case MYLED_IOC_LED_ON:
        gpio_set_value(dev->led_gpio, 1);
        dev->led_state = true; dev->toggle_count++;
        break;
    case MYLED_IOC_LED_OFF:
        gpio_set_value(dev->led_gpio, 0);
        dev->led_state = false; dev->toggle_count++;
        break;
    case MYLED_IOC_LED_TOGGLE:
        dev->led_state = !dev->led_state;
        gpio_set_value(dev->led_gpio, dev->led_state);
        dev->toggle_count++;
        break;
    case MYLED_IOC_RESET:
        gpio_set_value(dev->led_gpio, 0);
        dev->led_state = false;
        dev->toggle_count = 0;
        break;
    case MYLED_IOC_GET_STATUS:
        st.led_state    = dev->led_state ? 1 : 0;
        st.gpio_pin     = dev->led_gpio;
        st.toggle_count = dev->toggle_count;
        if (copy_to_user((void __user *)arg, &st, sizeof(st)))
            return -EFAULT;
        break;
    case MYLED_IOC_SET_BLINK:
        if (copy_from_user(&cfg, (void __user *)arg, sizeof(cfg)))
            return -EFAULT;
        if (!cfg.on_ms || !cfg.off_ms) return -EINVAL;
        dev->blink_cfg = cfg;
        break;
    case MYLED_IOC_GET_BLINK:
        if (copy_to_user((void __user *)arg,
                         &dev->blink_cfg, sizeof(dev->blink_cfg)))
            return -EFAULT;
        break;
    default:
        return -ENOTTY;
    }
    return 0;
}

static const struct file_operations mydev_fops = {
    .owner          = THIS_MODULE,
    .open           = mydev_open,
    .release        = mydev_release,
    .read           = mydev_read,
    .write          = mydev_write,
    .poll           = mydev_poll,
    .unlocked_ioctl = mydev_ioctl,
};

// ---- Init / Exit ----

static int __init mydev_init(void)
{
    int ret;

    mydev = kzalloc(sizeof(*mydev), GFP_KERNEL);
    if (!mydev) return -ENOMEM;

    mydev->led_gpio    = LED_GPIO;
    mydev->button_gpio = BUTTON_GPIO;
    spin_lock_init(&mydev->lock);
    init_waitqueue_head(&mydev->button_wq);

    // Request GPIOs
    ret = gpio_request(mydev->led_gpio, "myled");
    if (ret) goto err_led_gpio;
    gpio_direction_output(mydev->led_gpio, 0);

    ret = gpio_request(mydev->button_gpio, "mybutton");
    if (ret) goto err_btn_gpio;
    gpio_direction_input(mydev->button_gpio);

    // Get IRQ for button GPIO (falling edge = button press with pull-up)
    mydev->button_irq = gpio_to_irq(mydev->button_gpio);
    ret = request_irq(mydev->button_irq, button_handler,
                      IRQF_TRIGGER_FALLING, "mybutton_irq", mydev);
    if (ret) goto err_irq;

    // Register char device
    ret = alloc_chrdev_region(&dev_num, 0, 1, DRIVER_NAME);
    if (ret) goto err_chrdev;

    cdev_init(&mydev->cdev, &mydev_fops);
    mydev->cdev.owner = THIS_MODULE;
    ret = cdev_add(&mydev->cdev, dev_num, 1);
    if (ret) goto err_cdev;

    mydev_class = class_create(DRIVER_NAME);
    if (IS_ERR(mydev_class)) {
        ret = PTR_ERR(mydev_class); goto err_class;
    }

    if (IS_ERR(device_create(mydev_class, NULL, dev_num, NULL, DRIVER_NAME))) {
        ret = -ENODEV; goto err_device;
    }

    pr_info("myled_btn: ready on /dev/%s\n", DRIVER_NAME);
    return 0;

err_device: class_destroy(mydev_class);
err_class:  cdev_del(&mydev->cdev);
err_cdev:   unregister_chrdev_region(dev_num, 1);
err_chrdev: free_irq(mydev->button_irq, mydev);
err_irq:    gpio_free(mydev->button_gpio);
err_btn_gpio: gpio_free(mydev->led_gpio);
err_led_gpio: kfree(mydev);
    return ret;
}

static void __exit mydev_exit(void)
{
    device_destroy(mydev_class, dev_num);
    class_destroy(mydev_class);
    cdev_del(&mydev->cdev);
    unregister_chrdev_region(dev_num, 1);
    free_irq(mydev->button_irq, mydev);
    gpio_free(mydev->button_gpio);
    gpio_set_value(mydev->led_gpio, 0);
    gpio_free(mydev->led_gpio);
    kfree(mydev);
    pr_info("myled_btn: removed\n");
}

module_init(mydev_init);
module_exit(mydev_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("LED+Button driver with ioctl, blocking read, poll");
```

---

## 7. Hands-On Lab

### Hardware Setup

```
RPi4 Pin 11 (GPIO17) ──┤220Ω├── LED(+) → LED(-) → GND (Pin 9)
RPi4 Pin 13 (GPIO27) ──────────── Button Pin 1
RPi4 Pin 1  (3.3V)   ──┤10kΩ├── Button Pin 1   (pull-up)
Button Pin 2 ─────────────────── GND (Pin 14)
```

### Task 1 — Test Basic ioctl

```c
// test_ioctl.c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include "myled_ioctl.h"

int main(void)
{
    int fd = open("/dev/myled_btn", O_RDWR);
    struct myled_status st;

    ioctl(fd, MYLED_IOC_LED_ON);  sleep(1);
    ioctl(fd, MYLED_IOC_LED_OFF); sleep(1);

    ioctl(fd, MYLED_IOC_GET_STATUS, &st);
    printf("LED: %d, GPIO: %d, Toggles: %ld\n",
           st.led_state, st.gpio_pin, st.toggle_count);

    close(fd);
    return 0;
}
```

### Task 2 — Test Blocking Read

```bash
# On RPi4
# Terminal 1: blocking read (waits for button press)
cat /dev/myled_btn

# Terminal 2: press the physical button
# OR simulate by manually setting GPIO27 low briefly:
echo 27 > /sys/class/gpio/export
echo "in" > /sys/class/gpio/gpio27/direction
# Terminal 1 should unblock and print "1"
```

### Task 3 — Test poll() from C

```c
// test_poll.c
#include <stdio.h>
#include <fcntl.h>
#include <poll.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/dev/myled_btn", O_RDWR | O_NONBLOCK);
    struct pollfd pfd = { .fd = fd, .events = POLLIN };

    printf("Waiting for button press (5 second timeout)...\n");

    int ret = poll(&pfd, 1, 5000);
    if (ret == 0)        printf("Timeout!\n");
    else if (ret < 0)    perror("poll");
    else {
        char buf[4];
        read(fd, buf, sizeof(buf));
        printf("Button pressed!\n");
    }

    close(fd);
    return 0;
}
```

### Task 4 — Test O_NONBLOCK

```bash
# O_NONBLOCK: read returns -EAGAIN if no data
python3 -c "
import os, errno
fd = os.open('/dev/myled_btn', os.O_RDWR | os.O_NONBLOCK)
try:
    data = os.read(fd, 4)
    print(f'Got: {data}')
except BlockingIOError:
    print('No data yet (EAGAIN) — non-blocking works correctly')
os.close(fd)
"
```

---

## 8. Interview Questions

---

**Q1. Why is `_ENOTTY` returned for unknown ioctl commands rather than `_EINVAL`?**

**Answer:** `ENOTTY` (literally "Not a typewriter") is the POSIX-mandated error code for an ioctl that is not supported by the device or file descriptor. It signals "this file descriptor does not understand this ioctl command," as opposed to `EINVAL` which means "this command is understood but you passed a bad argument." The distinction matters for portability — applications can use `ENOTTY` to probe whether a feature is supported, then fall back gracefully. `EINVAL` would falsely suggest the command is known but the argument is wrong, confusing the caller.

---

**Q2. Explain the `_IOC_TYPE`, `_IOC_NR`, `_IOC_SIZE`, and `_IOC_DIR` macros.**

**Answer:** These macros decode the 32-bit ioctl command number:
- `_IOC_TYPE(cmd)` — extracts bits 15-8 (the magic number). Used to verify the command is intended for your driver.
- `_IOC_NR(cmd)` — extracts bits 7-0 (the sequence number). Used for bounds checking against `MAX_NR`.
- `_IOC_SIZE(cmd)` — extracts bits 29-16 (size of the data argument). Used with `access_ok()` to verify user buffer is large enough.
- `_IOC_DIR(cmd)` — extracts bits 31-30 (direction: `_IOC_NONE`, `_IOC_READ`, `_IOC_WRITE`, or both). Used to determine whether to call `access_ok()` for read, write, or both.

The direction from `_IOC_DIR` is from the **kernel's perspective**: `_IOC_READ` means the kernel reads from user space (i.e., user writes to kernel), `_IOC_WRITE` means the kernel writes to user space (kernel → user read).

---

**Q3. What is the difference between `wait_event_interruptible` and `wait_event`? When would you use each?**

**Answer:** Both put the current task to sleep until a condition becomes true, but they differ in how they handle signals:

- `wait_event_interruptible(wq, cond)` sets the task state to `TASK_INTERRUPTIBLE`. It will wake up if either: (a) `wake_up_interruptible()` is called, or (b) a signal is delivered (Ctrl+C, SIGTERM, etc.). If woken by a signal, returns `-ERESTARTSYS`. The caller should propagate this as `-EINTR` or `-ERESTARTSYS`. This is what most drivers should use — it keeps the system responsive to signals and prevents unkillable processes.

- `wait_event(wq, cond)` sets state to `TASK_UNINTERRUPTIBLE`. The task cannot be woken by signals — it will sleep until the condition is true or the machine shuts down. Use only when you have a very short, guaranteed wait (e.g., waiting for hardware to complete a reset operation that takes 1ms). Abuse of this leads to unkillable processes and "D state" (uninterruptible sleep) visible in `top`.

---

**Q4. How does `poll_wait()` work internally? What is it actually registering?**

**Answer:** `poll_wait(filp, wq, pt)` adds the wait queue head `wq` to a **poll table** `pt`. The poll table is a kernel-internal structure that the `select()`/`poll()`/`epoll()` machinery maintains. When `poll()` is called on a file descriptor, the kernel calls the driver's `.poll` function with an empty poll table; the driver calls `poll_wait()` which registers the wait queue. The `poll()`/`select()` machinery then goes to sleep waiting for any of the registered wait queues to be woken up.

When your driver calls `wake_up_interruptible(&button_wq)` (e.g., from an interrupt handler), all tasks sleeping in `poll()/select()` that registered that wait queue are woken up. They re-call the `.poll` function (this time with a NULL poll table to avoid re-registering), which returns the current event mask. If the mask has the events the caller is interested in, `poll()` returns to user space.

Importantly, `poll_wait()` does NOT block — it just registers the wait queue. The sleeping happens in the `select()`/`poll()` infrastructure, not in your driver.

---

**Q5. What is the significance of the `O_NONBLOCK` flag for your driver? How should read() behave with and without it?**

**Answer:** `O_NONBLOCK` (also `O_NDELAY`) is set in `filp->f_flags` when a file is opened with `open(..., O_NONBLOCK)` or set later with `fcntl(fd, F_SETFL, O_NONBLOCK)`. 

**Without `O_NONBLOCK` (default blocking mode):** `read()` should sleep until data is available. Use `wait_event_interruptible()`. The process is suspended, consuming no CPU. This is the correct behavior for most device reads.

**With `O_NONBLOCK`:** `read()` should return immediately. If data is available, return it. If no data is available, return `-EAGAIN` (same as `EWOULDBLOCK`). This allows event-loop patterns (combined with `poll()`/`epoll()`) where a single thread manages many file descriptors without blocking on any.

Always check `filp->f_flags & O_NONBLOCK` in your `read()` before sleeping. Failing to do so breaks programs that rely on non-blocking I/O for multiplexing (nginx, Node.js event loops, etc.).

---

**Q6. What happens to a process sleeping in `wait_event_interruptible()` when the module is unloaded?**

**Answer:** This is one of the most dangerous situations in kernel driver development. If a process is sleeping in `wait_event_interruptible()` waiting on your driver's wait queue, and you then `rmmod` the driver, the kernel will:

1. Try to unload the module — this requires the module's reference count to be zero
2. If the reference count is non-zero (the sleeping process holds the device open), `rmmod` fails
3. If somehow the module is force-removed (`rmmod -f`), the sleeping process's stack references code that no longer exists — kernel panic

The correct solution is: in your driver's `exit()` or `remove()` function, **signal all waiters to wake up** before unregistering anything:

```c
static void __exit mydev_exit(void)
{
    // Signal waiters to exit
    // We typically do this by setting a "device dying" flag
    dev->dying = true;
    wake_up_interruptible_all(&dev->button_wq);
    
    // Now it's safe to clean up
    // (processes woken by dying flag will get -EIO and close the fd)
    device_destroy(...);
    // ...
}

// In read():
if (dev->dying)
    return -EIO;  // device is going away
```

This is why many professional drivers use a `kref` or completion to synchronize module teardown with open file descriptors.

---

> **Next:** Day 5 — Kernel Memory Management: kmalloc, vmalloc, the slab allocator, and GFP flags.
