# Day 5 — System Calls: Tracing & Adding New Ones

> **Estimated read time:** 90–120 minutes  
> **Goal:** Master syscall observability tools, understand tracepoints, and go hands-on with a complete new syscall implementation.

---

## 1. Why Syscall Tracing Matters

In kernel engineering, you constantly need to answer:
- What syscalls is this process making? (debugging, security analysis)
- Which syscall is the bottleneck? (performance optimization)
- Is this library calling what I think it's calling? (ABI verification)
- What arguments is userspace passing? (bug investigation)
- Is a process exceeding expectations? (security auditing)

Five tools dominate this space: `strace`, `ltrace`, `perf`, `ftrace`, and `bpftrace`. Each has different overhead, granularity, and production-safety characteristics.

---

## 2. `strace` — Deep Dive

`strace` uses `ptrace(PTRACE_SYSCALL)` to intercept every syscall. High overhead (~10–100x slowdown), not for production — but invaluable for debugging.

### 2.1 Essential `strace` Patterns

```bash
# Basic: trace all syscalls of a command
strace ls /tmp

# Attach to a running process
strace -p $(pgrep nginx)

# Filter to specific syscalls only
strace -e trace=network          # All network-related
strace -e trace=file             # All file-related
strace -e trace=openat,read,write ls
strace -e trace='!mmap,mprotect' # Exclude noise

# Follow forks and threads
strace -f ls         # Follow all children (fork/clone)
strace -ff -o /tmp/strace ls  # Separate file per process

# Time each syscall
strace -T ls          # Show time spent in each call
strace -t ls          # Absolute timestamps
strace -r ls          # Relative timestamps

# Count and summarize
strace -c ls          # Summary table: count, time, errors
strace -C ls          # Count AND trace

# Show string arguments fully (default truncates at 32 chars)
strace -s 256 ls

# Trace system calls and signals
strace -e signal=SIGTERM ls
```

### 2.2 Reading `strace` Output

```
# Format: syscall(args...) = return_value [optional error]
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1975344, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f123abc0000
mmap(NULL, 2065480, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f123a8c4000
mprotect(0x7f123a8ec000, 1523712, PROT_NONE) = 0
mmap(0x7f123a8ec000, 1241088, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7f123a8ec000
...
close(3) = 0
write(1, "file1.txt\nfile2.txt\n", 20) = 20

# An error:
open("/nonexistent", O_RDONLY) = -1 ENOENT (No such file or directory)

# A blocked/slow syscall (with -T timing):
read(5, "", 4096) = 0 <1.234567>  ← waited 1.23 seconds
```

### 2.3 Common Use Cases

```bash
# What is nginx opening at startup?
strace -e trace=open,openat -f nginx 2>&1 | grep -v ENOENT

# Why is my program slow? (find the slow syscalls)
strace -c -f ./myprogram
# %time   seconds   usecs/call  calls   syscall
# 45.32   1.234567       12345    100   futex    ← lock contention!
# 23.11   0.630000          63  10000   read

# What is causing high syscall count?
strace -c -f ./myprogram 2>&1 | sort -k4 -rn | head -20

# Trace a process and log to file for analysis
strace -p 1234 -o /tmp/trace.log -f -T &
# Later: analyze
awk '{print $1}' /tmp/trace.log | sort | uniq -c | sort -rn | head

# Is this using epoll or select?
strace -e trace=epoll_create,epoll_ctl,epoll_wait,select,poll ./server

# Debug SECCOMP violations:
strace -e trace=seccomp ./program
```

### 2.4 strace Internals

```c
// How strace works (simplified):
// 1. Fork the target or attach with PTRACE_ATTACH
ptrace(PTRACE_ATTACH, pid, 0, 0);

// 2. Set options for syscall tracing and fork following
ptrace(PTRACE_SETOPTIONS, pid, 0,
    PTRACE_O_TRACESYSGOOD | PTRACE_O_TRACEFORK | PTRACE_O_TRACECLONE);

// 3. Enter the event loop
while (1) {
    // Allow process to run until next syscall entry or exit
    ptrace(PTRACE_SYSCALL, pid, 0, 0);
    waitpid(pid, &status, 0);
    
    // Check if this is a syscall stop
    if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
        // Read registers to get syscall info
        struct user_regs_struct regs;
        ptrace(PTRACE_GETREGS, pid, 0, &regs);
        
        if (entering_syscall) {
            print_syscall_entry(&regs);
        } else {
            print_syscall_exit(&regs);  // regs.rax = return value
        }
    }
}
```

The performance overhead comes from:
1. Every syscall causes two `SIGTRAP` signals (entry + exit)
2. Each signal requires context switches between tracee ↔ kernel ↔ strace
3. `ptrace(PTRACE_GETREGS)` is itself a syscall

---

## 3. `ltrace` — Library Call Tracing

```bash
# Trace library calls (not syscalls):
ltrace ls

# Trace both library calls AND syscalls:
ltrace -S ls

# Filter to specific library functions:
ltrace -e malloc,free,open ./myprogram
```

ltrace uses breakpoints on PLT (Procedure Linkage Table) entries, not ptrace syscall mode.

---

## 4. `perf trace` — Low-Overhead Syscall Profiling

`perf trace` is a modern strace alternative using tracepoints + perf ring buffer — much lower overhead, suitable for short production profiling.

```bash
# System-wide syscall count (1 second):
sudo perf trace -a --duration 1000 2>&1 | head -30

# Trace specific process:
sudo perf trace -p $(pgrep redis-server)

# Count syscalls, show top offenders:
sudo perf trace -a -s sleep 5
#   calls    errors  syscall
#  100000         0  epoll_wait
#   50000         0  read
#   30000         0  write
#   20000     5000   futex   ← lots of errors = lock contention

# Trace with timing:
sudo perf trace --duration 100 ls  # Only first 100ms

# Filter by syscall:
sudo perf trace -e write ls
```

### 4.1 `perf stat` for Syscall Counting

```bash
# Count specific syscalls via tracepoints:
sudo perf stat -e 'syscalls:sys_enter_write,syscalls:sys_enter_read' \
    -a sleep 5

# Performance counters for syscall overhead:
sudo perf stat -e 'raw_syscalls:sys_enter,raw_syscalls:sys_exit' \
    -a -I 1000
```

---

## 5. Kernel Tracepoints

Tracepoints are static instrumentation points scattered throughout the kernel. Unlike `kprobes` (dynamic), they have minimal overhead when disabled.

### 5.1 Finding Available Tracepoints

```bash
# List all tracepoints:
sudo perf list tracepoint

# Syscall-specific tracepoints:
sudo perf list | grep syscalls
# syscalls:sys_enter_read              [Tracepoint event]
# syscalls:sys_exit_read               [Tracepoint event]
# syscalls:sys_enter_write             [Tracepoint event]
# (one pair for each syscall!)

# Via tracefs:
ls /sys/kernel/debug/tracing/events/syscalls/ | head -20

# See fields of a tracepoint:
cat /sys/kernel/debug/tracing/events/syscalls/sys_enter_write/format
# name: sys_enter_write
# ID: 641
# format:
#     field:unsigned short common_type;   offset:0; size:2; signed:0;
#     field:unsigned char common_flags;   offset:2; size:1; signed:0;
#     field:unsigned char common_preempt_count; offset:3; size:1; signed:0;
#     field:int common_pid;               offset:4; size:4; signed:1;
#     
#     field:int __syscall_nr;             offset:8; size:4; signed:1;
#     field:unsigned int fd;              offset:16; size:8; signed:0;
#     field:const char * buf;            offset:24; size:8; signed:0;
#     field:size_t count;               offset:32; size:8; signed:0;
```

### 5.2 Using Tracepoints via `tracefs`

```bash
# Enable a specific tracepoint:
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_write/enable

# Set a filter:
echo 'fd == 1' > /sys/kernel/debug/tracing/events/syscalls/sys_enter_write/filter

# Read the trace buffer:
cat /sys/kernel/debug/tracing/trace
# tracer: nop
#   TASK-PID   CPU#  |||  TIMESTAMP  FUNCTION
#       bash-1234  [000] d... 1234.567890: sys_enter_write: fd: 1, buf: 0x..., count: 6

# Live stream:
cat /sys/kernel/debug/tracing/trace_pipe

# Disable:
echo 0 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_write/enable
echo > /sys/kernel/debug/tracing/trace  # Clear buffer
```

### 5.3 Tracepoint Implementation in Kernel Source

```c
// How TRACE_EVENT macro works (simplified):
// In include/trace/events/syscalls.h:

TRACE_EVENT(sys_enter,
    TP_PROTO(struct pt_regs *regs, long id),
    TP_ARGS(regs, id),
    TP_STRUCT__entry(
        __field(long, id)
        __array(unsigned long, args, 6)
    ),
    TP_fast_assign(
        __entry->id   = id;
        syscall_get_arguments(current, regs, __entry->args);
    ),
    TP_printk("NR %ld (%lx, %lx, %lx, %lx, %lx, %lx)",
        __entry->id,
        __entry->args[0], __entry->args[1], __entry->args[2],
        __entry->args[3], __entry->args[4], __entry->args[5])
);

// Called from: arch/x86/entry/common.c
// trace_sys_enter(regs, regs->orig_ax);  ← When tracepoint enabled
// [NOP instruction]                       ← When tracepoint disabled
```

When disabled, tracepoints compile to a single `NOP` instruction — near-zero overhead.

---

## 6. `ftrace` — Kernel Function Tracer

`ftrace` is the kernel's built-in tracing infrastructure. More powerful than tracepoints for understanding what the kernel does during a syscall.

### 6.1 Function Tracing

```bash
# Set tracer to function mode:
cd /sys/kernel/debug/tracing
echo function > current_tracer

# Filter to specific functions (otherwise everything is traced — millions of lines):
echo 'sys_write' > set_ftrace_filter
echo 'vfs_write' >> set_ftrace_filter
echo 'ksys_write' >> set_ftrace_filter

# Enable tracing:
echo 1 > tracing_on

# Run your workload:
echo "test" > /dev/null

# Disable and read:
echo 0 > tracing_on
cat trace
# tracer: function
#          bash-1234  [001] ...  9876.543210: sys_write <-do_syscall_64
#          bash-1234  [001] ...  9876.543211: ksys_write <-__x64_sys_write
#          bash-1234  [001] ...  9876.543212: vfs_write <-ksys_write

# Function graph tracer (shows call depth and timing):
echo function_graph > current_tracer
echo 'ksys_write' > set_graph_function
cat trace
# tracer: function_graph
# CPU  DURATION                  FUNCTION CALLS
# |   |   |    |                 |   |   |   |
#  1)               |  ksys_write() {
#  1)               |    vfs_write() {
#  1)               |      rw_verify_area() {
#  1)   0.250 us    |        security_file_permission();
#  1)   1.100 us    |      }
#  1)               |      tty_write() {
#  1)  45.320 us    |      }
#  1)  48.100 us    |    }
#  1)  49.200 us    |  }
```

### 6.2 `trace-cmd` — ftrace Frontend

```bash
# Install:
sudo apt install trace-cmd

# Record syscall traces:
sudo trace-cmd record -e syscalls:sys_enter_write -e syscalls:sys_exit_write \
    ./myprogram

# Analyze recording:
trace-cmd report

# Record kernel function graph for a command:
sudo trace-cmd record -p function_graph -g ksys_write ls

# Report with ASCII art:
trace-cmd report -l | head -50
```

### 6.3 `kprobes` via ftrace — Dynamic Tracing

kprobes let you instrument *any* kernel function dynamically, not just tracepoints:

```bash
# Trace a non-tracepoint function using kprobes:
echo 'p:myprobe tcp_sendmsg sk=%di size=%dx' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable
cat /sys/kernel/debug/tracing/trace_pipe

# The format: p:NAME FUNCNAME ARG1=FORMAT ARG2=...
# Probe types: p (entry), r (return)
# Format strings: %di (rdi register), %dx (rdx), %ax (rax)...

# Return probe to capture return value:
echo 'r:myretprobe tcp_sendmsg retval=$retval' >> kprobe_events
```

---

## 7. `bpftrace` — Production-Safe Dynamic Tracing

`bpftrace` is the most powerful tool for syscall tracing in production. It runs eBPF programs in the kernel with near-zero overhead.

### 7.1 Installation

```bash
sudo apt install bpftrace   # or bpftrace snap
# Requires: kernel >= 4.9 with CONFIG_BPF=y
```

### 7.2 One-Liners for Syscall Analysis

```bash
# Count syscalls by name, system-wide:
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[ksym(args->id)] = count(); }'

# Count syscalls by process name:
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm, ksym(args->id)] = count(); }'

# Trace write() calls with fd, size, and process:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_write {
    printf("pid=%d comm=%s fd=%d size=%d\n", pid, comm, args->fd, args->count);
}'

# Measure write() latency (histogram):
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_write { @start[tid] = nsecs; }
tracepoint:syscalls:sys_exit_write  /@start[tid]/ {
    @latency_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}'

# Find which process is calling open() most:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_openat {
    @[comm] = count();
}
END { print(@); }'

# Trace slow syscalls (>1ms):
sudo bpftrace -e '
tracepoint:raw_syscalls:sys_enter { @ts[tid] = nsecs; }
tracepoint:raw_syscalls:sys_exit {
    if (@ts[tid]) {
        $delta = nsecs - @ts[tid];
        if ($delta > 1000000) {
            printf("slow syscall: %s pid=%d time=%dms\n",
                ksym(args->id), pid, $delta/1000000);
        }
        delete(@ts[tid]);
    }
}'

# Stack trace of what leads to read() calls:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_read {
    @[ustack] = count();
}
END { print(@); }'
```

### 7.3 bpftrace Scripts for Deeper Analysis

```bash
# File: syscall_top.bt
#!/usr/bin/env bpftrace

// Top 10 syscalls by total time, per process
tracepoint:raw_syscalls:sys_enter
{
    @entry[pid, tid, args->id] = nsecs;
}

tracepoint:raw_syscalls:sys_exit
/@entry[pid, tid, args->id]/
{
    @time_ns[comm, ksym(args->id)] = sum(nsecs - @entry[pid, tid, args->id]);
    delete(@entry[pid, tid, args->id]);
}

END
{
    print(@time_ns, 20);  // top 20
}
```

```bash
sudo bpftrace syscall_top.bt
# Wait 10 seconds, Ctrl+C
# Output:
# @time_ns[postgres, futex]: 45234567
# @time_ns[nginx, epoll_wait]: 34123456
# @time_ns[mysqld, read]: 12345678
```

---

## 8. Syscall Policies with Audit

The Linux Audit system provides tamper-evident syscall logging:

```bash
# Log all execve calls (process executions):
sudo auditctl -a always,exit -F arch=b64 -S execve -k exec_log

# Log opens of sensitive files:
sudo auditctl -w /etc/passwd -p rwar -k passwd_watch

# Log all syscalls by a specific user:
sudo auditctl -a always,exit -F uid=1000 -S all -k user1000

# View audit log:
sudo ausearch -k exec_log
sudo ausearch -f /etc/passwd
sudo aureport --syscall  # summary report

# auditd.conf — configure log destination:
cat /etc/audit/auditd.conf
```

---

## 9. Complete New Syscall: `sys_procinfo`

Let's build a real, useful syscall that returns per-process CPU and memory statistics. This demonstrates the full end-to-end.

### 9.1 Design

```c
// Interface we're implementing:
// int procinfo(pid_t pid, struct proc_stat_user *stat);
// Returns 0 on success, negative errno on error.

struct proc_stat_user {
    pid_t pid;
    pid_t ppid;
    unsigned long long utime;    // User time in nanoseconds
    unsigned long long stime;    // System time in nanoseconds
    unsigned long vm_rss_kb;     // Resident Set Size in KB
    unsigned long vm_virt_kb;    // Virtual memory in KB
    int nr_threads;
    char comm[16];               // Process name
};
```

### 9.2 Implementation

```c
// File: kernel/procinfo.c
#include <linux/syscalls.h>
#include <linux/sched.h>
#include <linux/mm.h>
#include <linux/pid.h>
#include <linux/uaccess.h>
#include <linux/sched/cputime.h>
#include <linux/rcupdate.h>

struct proc_stat_user {
    pid_t pid;
    pid_t ppid;
    unsigned long long utime;
    unsigned long long stime;
    unsigned long vm_rss_kb;
    unsigned long vm_virt_kb;
    int nr_threads;
    char comm[16];
};

SYSCALL_DEFINE2(procinfo, pid_t, pid,
                struct proc_stat_user __user *, ustat)
{
    struct proc_stat_user kstat = {};
    struct task_struct *task;
    u64 utime, stime;
    
    // Validate output pointer before doing any work
    if (!ustat || !access_ok(ustat, sizeof(*ustat)))
        return -EFAULT;
    
    // Look up the task by PID. rcu_read_lock protects task_struct
    // from being freed while we read it.
    rcu_read_lock();
    
    task = pid_task(find_vpid(pid), PIDTYPE_PID);
    if (!task) {
        rcu_read_unlock();
        return -ESRCH;  // No such process
    }
    
    // Check permission: can we read info about this process?
    // A full implementation would check ptrace permission:
    // if (!ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS)) { ... }
    
    // Read task info (under rcu_read_lock, we can read most fields):
    kstat.pid = task_tgid_vnr(task);   // Thread-group ID (PID in userspace)
    kstat.ppid = task_ppid_nr(task);
    kstat.nr_threads = get_nr_threads(task);
    
    memcpy(kstat.comm, task->comm, sizeof(kstat.comm));
    kstat.comm[sizeof(kstat.comm) - 1] = '\0';
    
    // CPU time (requires holding task_lock for consistency):
    task_lock(task);
    thread_group_cputime_adjusted(task, &utime, &stime);
    task_unlock(task);
    
    kstat.utime = ktime_to_ns(utime);
    kstat.stime = ktime_to_ns(stime);
    
    // Memory info (requires mmap_lock for mm_struct):
    if (task->mm) {
        // Try to get a reference to mm so it stays alive
        struct mm_struct *mm = get_task_mm(task);
        if (mm) {
            rcu_read_unlock();  // Can drop rcu while holding mm ref
            
            mmap_read_lock(mm);
            kstat.vm_virt_kb = mm->total_vm << (PAGE_SHIFT - 10);
            kstat.vm_rss_kb  = get_mm_rss(mm) << (PAGE_SHIFT - 10);
            mmap_read_unlock(mm);
            
            mmput(mm);
            
            // Copy result to user space
            if (copy_to_user(ustat, &kstat, sizeof(kstat)))
                return -EFAULT;
            return 0;
        }
    }
    
    rcu_read_unlock();
    
    // Process has no mm (kernel thread) or already exiting
    if (copy_to_user(ustat, &kstat, sizeof(kstat)))
        return -EFAULT;
    return 0;
}
```

### 9.3 Register the Syscall

```bash
# 1. Add to syscall table:
echo "463    common  procinfo   sys_procinfo" >> arch/x86/entry/syscalls/syscall_64.tbl

# 2. Declare in syscalls.h:
# asmlinkage long sys_procinfo(pid_t pid, struct proc_stat_user __user *stat);

# 3. Add to kernel/Makefile:
# obj-y += procinfo.o

# 4. Rebuild:
make -j$(nproc)
```

### 9.4 User-Space Test Program

```c
// test_procinfo.c
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

#define __NR_procinfo 463

struct proc_stat_user {
    int pid;
    int ppid;
    unsigned long long utime;
    unsigned long long stime;
    unsigned long vm_rss_kb;
    unsigned long vm_virt_kb;
    int nr_threads;
    char comm[16];
};

int main(int argc, char *argv[])
{
    pid_t target = argc > 1 ? atoi(argv[1]) : getpid();
    struct proc_stat_user stat = {};
    
    long ret = syscall(__NR_procinfo, target, &stat);
    if (ret < 0) {
        fprintf(stderr, "procinfo(%d): %s\n", target, strerror(errno));
        return 1;
    }
    
    printf("PID:      %d\n", stat.pid);
    printf("PPID:     %d\n", stat.ppid);
    printf("Name:     %s\n", stat.comm);
    printf("Threads:  %d\n", stat.nr_threads);
    printf("RSS:      %lu KB\n", stat.vm_rss_kb);
    printf("VSZ:      %lu KB\n", stat.vm_virt_kb);
    printf("Utime:    %llu ns\n", stat.utime);
    printf("Stime:    %llu ns\n", stat.stime);
    
    return 0;
}
```

---

## 10. System Call Alternatives

The kernel maintainers actively discourage adding new syscalls because:
1. Syscall numbers are ABI-stable forever — you can't remove them
2. They add to the attack surface
3. There are usually better alternatives

### 10.1 When to Use What Instead

| Need | Better Alternative |
|------|-------------------|
| Pass config to kernel | `sysctl` via `/proc/sys/` |
| Get kernel info | Read from `/proc/` or `/sys/` |
| Device-specific ops | `ioctl()` on a device file |
| New protocol-like interface | Netlink socket |
| Notify user of events | Netlink, `inotify`, `fanotify` |
| Share large data | `mmap()` shared memory |
| High-frequency small ops | batch them, or use `io_uring` |

### 10.2 `ioctl` as a Syscall Alternative

```c
// Kernel side: define a new ioctl command
// Using IOC macro: _IO, _IOR, _IOW, _IOWR
// Type (8-bit magic), number (8-bit), size (14-bit for data)

#define MYDEV_MAGIC 'M'
#define MYDEV_GET_INFO  _IOR(MYDEV_MAGIC, 1, struct mydev_info)
#define MYDEV_SET_MODE  _IOW(MYDEV_MAGIC, 2, int)
#define MYDEV_RESET     _IO(MYDEV_MAGIC,  3)

// In file_operations:
static long mydev_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct mydev_info kinfo;
    int mode;
    
    switch (cmd) {
    case MYDEV_GET_INFO:
        kinfo.version = 1;
        kinfo.features = 0x42;
        if (copy_to_user((void __user *)arg, &kinfo, sizeof(kinfo)))
            return -EFAULT;
        return 0;
        
    case MYDEV_SET_MODE:
        if (get_user(mode, (int __user *)arg))
            return -EFAULT;
        dev->mode = mode;
        return 0;
        
    case MYDEV_RESET:
        return mydev_do_reset(dev);
        
    default:
        return -ENOTTY;  // Not a tty, standard "unknown ioctl" error
    }
}
```

### 10.3 `netlink` for Complex Interfaces

```c
// Netlink provides a socket-based kernel↔userspace messaging system
// Used by: iproute2 (ip, ss, tc), NetworkManager, nftables...

// Kernel side:
static struct nla_policy my_policy[MY_ATTR_MAX + 1] = {
    [MY_ATTR_VALUE] = { .type = NLA_U32 },
    [MY_ATTR_NAME]  = { .type = NLA_STRING, .len = 64 },
};

static int my_cmd_handler(struct sk_buff *skb, struct nlmsghdr *nlh,
                          struct netlink_ext_ack *extack)
{
    struct nlattr *tb[MY_ATTR_MAX + 1];
    int err;
    
    err = nlmsg_parse(nlh, 0, tb, MY_ATTR_MAX, my_policy, extack);
    if (err < 0) return err;
    
    if (tb[MY_ATTR_VALUE]) {
        u32 val = nla_get_u32(tb[MY_ATTR_VALUE]);
        // ... process val ...
    }
    
    return 0;
}
```

---

## 11. Syscall Performance Benchmarking

```c
// benchmark_syscalls.c — measure raw syscall overhead
#include <time.h>
#include <unistd.h>
#include <stdio.h>

#define ITERATIONS 10000000UL

int main(void)
{
    struct timespec start, end;
    unsigned long i;
    
    // Warm up
    for (i = 0; i < 1000; i++)
        getpid();
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    for (i = 0; i < ITERATIONS; i++)
        getpid();   // Minimal-work syscall
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    unsigned long ns = (end.tv_sec - start.tv_sec) * 1000000000UL
                     + (end.tv_nsec - start.tv_nsec);
    
    printf("Total:    %lu ns\n", ns);
    printf("Per call: %.1f ns\n", (double)ns / ITERATIONS);
    // Typical: 50-200ns depending on KPTI, CPU model
    
    // Now benchmark vDSO:
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (i = 0; i < ITERATIONS; i++) {
        struct timespec ts;
        clock_gettime(CLOCK_REALTIME, &ts);  // Uses vDSO
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    ns = (end.tv_sec - start.tv_sec) * 1000000000UL
       + (end.tv_nsec - start.tv_nsec);
    printf("vDSO:     %.1f ns\n", (double)ns / ITERATIONS);
    // Typical: 5-20ns (no kernel trap)
    
    return 0;
}
```

---

## 12. Syscall Restart Mechanism

Some syscalls can be interrupted by a signal and automatically restarted. You'll see this when signals interrupt `read()` or `sleep()`:

```c
// Kernel side — marking a syscall restartable:
// ERESTARTSYS = restart if handler doesn't set SA_RESTART
// ERESTARTNOINTR = always restart
// ERESTARTNOHAND = never restart

long sys_nanosleep(...) {
    // ...
    ret = do_nanosleep(t, HRTIMER_MODE_REL);
    if (ret == -ERESTART_RESTARTBLOCK) {
        // Set up restart_block so we can retry without full syscall
        current->restart_block.fn = hrtimer_nanosleep_restart;
        return -ERESTART_RESTARTBLOCK;
    }
    return ret;
}

// In syscall exit path (arch/x86/entry/common.c):
if (regs->ax == -ERESTARTSYS || regs->ax == -ERESTARTNOINTR ||
    regs->ax == -ERESTARTNOHAND || regs->ax == -ERESTART_RESTARTBLOCK) {
    // If no signal pending, restart the syscall
    // Adjust RIP to point back to SYSCALL instruction
    regs->ip -= 2;    // SYSCALL is a 2-byte instruction
    regs->ax = regs->orig_ax;  // Restore syscall number
}
```

User space doesn't see `ERESTARTSYS` — it either sees the restarted call complete, or `-EINTR` if restart isn't possible.

---

## 13. `ausyscall` — Quick Syscall Reference

```bash
# Map name to number:
ausyscall write      # 1
ausyscall openat     # 257

# Map number to name:
ausyscall 9          # mmap

# List all (current architecture):
ausyscall --dump
```

---

## 14. Mental Model Checkpoint

After Day 5, you should be able to:

1. Use `strace` effectively — filter by syscall type, follow forks, measure timing.
2. Read `strace` output and understand the argument/return format.
3. Use `bpftrace` to count syscalls, measure latency, and capture stack traces.
4. Enable a tracepoint via `tracefs` and filter its output.
5. Explain why `perf trace` has lower overhead than `strace`.
6. Write a complete syscall implementation with proper argument validation, `__user` handling, and locking.
7. Explain what `ERESTARTSYS` is and when it's used.
8. Choose between ioctl, netlink, sysctl, and a new syscall for a given use case.

---

## Key Source Files

```bash
arch/x86/entry/common.c              # do_syscall_64, syscall_exit handling
arch/x86/entry/syscalls/syscall_64.tbl  # Syscall table definition
include/trace/events/syscalls.h       # Syscall tracepoint definitions
kernel/trace/                         # ftrace, tracepoint infrastructure
kernel/seccomp.c                      # seccomp filter execution
include/uapi/linux/audit.h            # Audit syscall definitions
tools/perf/                           # perf tool source
```

---

## Summary

Syscall tracing is a layered stack:
- `strace` → ptrace-based, high overhead, full visibility, not for production
- `perf trace` → tracepoint-based, low overhead, suitable for production profiling
- `bpftrace` → eBPF-based, near-zero overhead, fully programmable
- `ftrace` → built-in, function-level granularity, captures kernel internals

Adding a syscall requires touching the table (`syscall_64.tbl`), declaring the prototype (`syscalls.h`), implementing with `SYSCALL_DEFINEn()`, and carefully validating all user-supplied arguments using `copy_from_user()`, `get_user()`, `access_ok()`, and proper locking.

Tomorrow: loadable kernel modules — the mechanism that lets you extend the kernel at runtime.
