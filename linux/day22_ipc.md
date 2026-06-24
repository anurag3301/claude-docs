# Day 22 — IPC Mechanisms

> **Estimated read time:** 90–120 minutes  
> **Goal:** Understand the kernel implementation of pipes, FIFO, SysV IPC (message queues, shared memory, semaphores), POSIX IPC, and how to choose between them.

---

## 1. IPC Taxonomy

```
Inter-Process Communication (IPC) in Linux:

Byte streams:
  ├── Pipe (unnamed, related processes only)
  ├── FIFO (named, unrelated processes)
  └── Unix domain socket (SOCK_STREAM or SOCK_DGRAM)

Messages:
  ├── SysV message queues
  ├── POSIX message queues (mqueue)
  └── Unix domain socket (SOCK_DGRAM)

Shared memory:
  ├── SysV shared memory (shmget/shmat)
  ├── POSIX shared memory (shm_open/mmap)
  ├── Anonymous mmap (PROT_SHARED between forked processes)
  └── memfd_create (anonymous file + mmap)

Synchronization:
  ├── SysV semaphores (semget/semop)
  ├── POSIX semaphores (sem_open or sem_init)
  ├── Futexes (userspace-based, see Day 16)
  └── Eventfd (Linux-specific)

Event notification:
  ├── Signals (limited payload)
  ├── eventfd (integer counter, readable via epoll)
  ├── signalfd (signals as file events)
  └── Pipe (byte stream)
```

---

## 2. Pipes — Kernel Implementation

### 2.1 Anonymous Pipes

```c
// Syscall:
int pipefd[2];
pipe2(pipefd, O_CLOEXEC | O_NONBLOCK);
// pipefd[0] = read end
// pipefd[1] = write end
```

```c
// fs/pipe.c
SYSCALL_DEFINE2(pipe2, int __user *, fildes, int, flags)
{
    struct file *files[2];
    int fd[2];
    
    // Create the pipe inode:
    struct pipe_inode_info *pipe = alloc_pipe_info();
    // Allocates the pipe ring buffer (16 pages by default = 65536 bytes)
    // pipe->bufs[] = array of pipe_buffer structs
    // pipe->head = write head, pipe->tail = read tail
    
    // Create two file descriptors:
    create_pipe_files(files, 0);
    // files[0]: O_RDONLY, f_op = pipefifo_fops
    // files[1]: O_WRONLY, f_op = pipefifo_fops
    
    fd[0] = get_unused_fd_flags(flags);
    fd[1] = get_unused_fd_flags(flags);
    
    fd_install(fd[0], files[0]);
    fd_install(fd[1], files[1]);
    
    copy_to_user(fildes, fd, sizeof(fd));
    return 0;
}
```

### 2.2 Pipe Internal Structure

```c
// include/linux/pipe_fs_i.h
struct pipe_inode_info {
    struct mutex mutex;         // Protects pipe state
    wait_queue_head_t rd_wait;  // Readers wait here
    wait_queue_head_t wr_wait;  // Writers wait here
    unsigned int head;          // Write head index
    unsigned int tail;          // Read tail index
    unsigned int max_usage;     // Maximum ring size
    unsigned int ring_size;     // Current ring buffer size (power of 2)
    bool readers;               // Any readers?
    bool writers;               // Any writers?
    unsigned int r_counter;     // # of file structs open for reading
    unsigned int w_counter;     // # of file structs open for writing
    struct pipe_buffer *bufs;   // Array of pipe_buffer structs
    struct user_struct *user;   // User who created the pipe (for rlimit)
};

struct pipe_buffer {
    struct page *page;          // The page containing data
    unsigned int offset;        // Data starts at this offset in the page
    unsigned int len;           // Length of data in this buffer
    const struct pipe_buf_operations *ops;
    unsigned int flags;         // PIPE_BUF_FLAG_*
    unsigned long private;
};
```

The pipe is a **ring buffer of pages**. Up to 16 pages (64KB) by default can be buffered.

### 2.3 Pipe Write Path

```c
// fs/pipe.c
static ssize_t pipe_write(struct kiocb *iocb, struct iov_iter *from)
{
    struct pipe_inode_info *pipe = iocb->ki_filp->private_data;
    
    mutex_lock(&pipe->mutex);
    
    for (;;) {
        // Is the ring full?
        unsigned int head = pipe->head;
        unsigned int tail = pipe->tail;
        unsigned int mask = pipe->ring_size - 1;
        
        if (!pipe_full(head, tail, pipe->max_usage)) {
            // Space available: copy data into a page
            struct pipe_buffer *buf = &pipe->bufs[head & mask];
            
            if (!buf->page) {
                // Allocate a new page:
                buf->page = alloc_page(GFP_HIGHUSER | __GFP_ACCOUNT);
                buf->offset = 0;
                buf->len = 0;
            }
            
            // Copy from user to the page:
            copied = copy_page_from_iter(buf->page, buf->offset + buf->len,
                                          from);
            buf->len += copied;
            
            if (head == tail + 1)  // Was empty: wake readers
                wake_up_interruptible_sync_poll(&pipe->rd_wait, ...);
            
            pipe->head++;
        } else {
            // Ring full: wait for reader to consume
            pipe_wait_writable(pipe);  // Sleeps until space available
        }
    }
    
    mutex_unlock(&pipe->mutex);
    return total;
}
```

### 2.4 PIPE_BUF Atomicity

POSIX guarantees that writes of up to `PIPE_BUF` bytes (4096 on Linux) are **atomic** — they either complete entirely or block, never partially written interleaved with other writers:

```c
// If write_size <= PIPE_BUF: atomic (either all or wait)
// If write_size >  PIPE_BUF: may be interleaved with other writers

#define PIPE_BUF 4096

// The kernel enforces this in pipe_write():
if (chars <= pipe_space_for_user(head - tail - 1, pipe)) {
    // Atomic write: enough space for all data
} else {
    // Non-atomic path: may block partially
}
```

### 2.5 Pipe Capacity and `F_SETPIPE_SZ`

```bash
# Default pipe capacity:
cat /proc/sys/fs/pipe-max-size   # Max: usually 1MB
# Default capacity per pipe: 65536 bytes (16 pages)

# Change pipe capacity (up to limit, requires CAP_SYS_RESOURCE for large):
int fd = pipefd[1];
fcntl(fd, F_SETPIPE_SZ, 1048576);  // 1MB pipe
fcntl(fd, F_GETPIPE_SZ);           // Get current capacity

# Splice: zero-copy data transfer through a pipe:
splice(src_fd, NULL, pipefd[1], NULL, len, SPLICE_F_MOVE);  // src → pipe
splice(pipefd[0], NULL, dst_fd, NULL, len, SPLICE_F_MOVE);  // pipe → dst
# No userspace copying — data stays in kernel pages
```

---

## 3. FIFOs (Named Pipes)

```bash
# Create:
mkfifo /tmp/myfifo
# OR:
mknod /tmp/myfifo p

# Use (shell):
cat /tmp/myfifo &      # Reader opens and waits
echo "hello" > /tmp/myfifo  # Writer sends and reader receives
```

```c
// Kernel: FIFO is just a pipe backed by an inode in the filesystem
// fs/inode.c: inode->i_fop = &pipefifo_fops
// Opening a FIFO blocks until both reader and writer are present
// (unless O_NONBLOCK is set)

// open() on FIFO:
// O_RDONLY: blocks until a writer opens it
// O_WRONLY: blocks until a reader opens it
// O_RDONLY|O_NONBLOCK: returns immediately, even without writer
// O_WRONLY|O_NONBLOCK: returns ENXIO if no reader
```

---

## 4. SysV IPC — The Old Standard

SysV IPC (System V Inter-Process Communication) is an older API, still widely used by databases (Oracle, PostgreSQL use shared memory). It has a key-based namespace.

### 4.1 IPC Keys

```c
// Key: unique 32-bit identifier for IPC resources
// Generated from a path + project ID:
key_t key = ftok("/some/path", 'A');
// OR: use IPC_PRIVATE for a private resource (only accessible to descendants)
```

### 4.2 SysV Shared Memory (`shmget`/`shmat`)

```c
// Create shared memory segment:
key_t key = ftok("/var/db/mydb", 1);
int shmid = shmget(key,
                   size,         // Size in bytes
                   IPC_CREAT | 0666);  // Create if not exists, permissions

// Attach to address space:
void *addr = shmat(shmid, NULL, 0);   // NULL = kernel chooses address
// addr is now mapped; multiple processes can attach the same shmid

// Access:
struct my_data *data = (struct my_data *)addr;
data->counter++;

// Detach (unmap from address space):
shmdt(addr);

// Destroy (when no longer needed):
shmctl(shmid, IPC_RMID, NULL);
```

```bash
# List SysV shared memory segments:
ipcs -m
# ------ Shared Memory Segments --------
# key        shmid      owner      perms      bytes      nattch     status
# 0x00005678 1234       postgres   600        134217728  3               ← 128MB
# 0x00001234 5678       oracle     660        1073741824 1               ← 1GB

# Remove a segment:
ipcrm -m 1234   # By shmid
ipcrm -M 0x5678 # By key

# Show limits:
ipcs -lm
# ------ Shared Memory Limits --------
# max number of segments = 4096
# max seg size (kbytes) = 18014398509481983
# max total shared memory (kbytes) = 18014398509481983
# min seg size (bytes) = 1
```

### 4.3 Kernel Implementation of Shared Memory

```c
// ipc/shm.c
SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
    struct ipc_namespace *ns = current->nsproxy->ipc_ns;
    
    // Find or create the segment:
    return ipcget(ns, &shm_ids(ns), &shm_ops, &params);
}

// shmat: maps the segment into process address space
SYSCALL_DEFINE3(shmat, int, shmid, char __user *, shmaddr, int, shmflg)
{
    struct shmid_kernel *shp;
    unsigned long addr;
    
    // Find the segment:
    shp = shm_obtain_object_check(ns, shmid);
    
    // Map it:
    // do_mmap() with the shmem file as backing
    addr = do_mmap(shp->shm_file, requested_addr, shp->shm_segsz,
                   prot, MAP_SHARED | MAP_FIXED_NOREPLACE, 0);
    
    // Track the attachment:
    shp->shm_nattch++;
    
    return addr;
}
```

Internally, SysV shared memory uses `tmpfs` — a segment is really a file in an internal tmpfs filesystem, mmap'd into process address spaces.

---

## 5. SysV Message Queues

```c
// Create or open a message queue:
int msqid = msgget(key, IPC_CREAT | 0666);

// Send a message:
struct msgbuf {
    long mtype;     // Message type (must be > 0)
    char mtext[1024]; // Message data
} msg;
msg.mtype = 1;
snprintf(msg.mtext, sizeof(msg.mtext), "hello");
msgsnd(msqid, &msg, strlen(msg.mtext) + 1, 0);  // 0 = block if queue full

// Receive:
msgrcv(msqid, &msg, sizeof(msg.mtext),
       0,     // 0 = any message type; N = only type N; -N = type <= N
       0);    // 0 = block if empty

// Remove:
msgctl(msqid, IPC_RMID, NULL);
```

```c
// kernel implementation: ipc/msg.c
// Message queues are linked lists of struct msg_msg:
struct msg_msg {
    struct list_head m_list;   // Link in queue
    long m_type;               // Message type
    size_t m_ts;               // Message size
    struct msg_msgseg *next;   // For messages > PAGE_SIZE
    void *security;            // LSM security
    /* Data follows inline */
};

// SYSCALL_DEFINE4(msgsnd):
// - Allocates msg_msg in kernel memory
// - Copies user data into it (copy_from_user)
// - Appends to queue (list_add_tail)
// - Wakes up any waiting receivers

// SYSCALL_DEFINE5(msgrcv):
// - Searches queue for matching message type
// - Copies to user (copy_to_user)
// - Removes from queue
// - Wakes up any waiting senders
```

---

## 6. SysV Semaphores

SysV semaphores are *arrays* of semaphores, not single values:

```c
// Create a semaphore set with 2 semaphores:
int semid = semget(key, 2, IPC_CREAT | 0666);

// Initialize values:
unsigned short init_vals[2] = {1, 1};  // Both start at 1
semctl(semid, 0, SETALL, init_vals);

// Atomic operations on semaphores:
struct sembuf ops[2];
ops[0].sem_num = 0;    // Semaphore 0
ops[0].sem_op = -1;    // Decrement (acquire)
ops[0].sem_flg = SEM_UNDO;  // Undo if process dies

ops[1].sem_num = 1;    // Semaphore 1
ops[1].sem_op = -1;    // Decrement (acquire)
ops[1].sem_flg = SEM_UNDO;

semop(semid, ops, 2);  // Atomic: both or neither

// Release:
ops[0].sem_op = +1;  // Increment (release)
ops[1].sem_op = +1;
semop(semid, ops, 2);

// Remove:
semctl(semid, 0, IPC_RMID);
```

`SEM_UNDO` is important: if the process dies while holding a semaphore, the kernel automatically reverses the operation — prevents deadlock from crashed processes.

---

## 7. POSIX Shared Memory

Cleaner API than SysV:

```c
#include <sys/mman.h>
#include <fcntl.h>

// Create (visible as /dev/shm/<name> on Linux):
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, size);  // Set size

// Map:
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);  // Can close fd; mapping persists

// In another process:
int fd = shm_open("/myshm", O_RDWR, 0);
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);

// Destroy:
shm_unlink("/myshm");

// munmap each process's mapping:
munmap(addr, size);
```

```bash
# POSIX shm lives in /dev/shm/:
ls -lh /dev/shm/
# -rw------- 1 postgres postgres 128M ... PostgreSQL shared buffer

# It's backed by tmpfs:
mount | grep shm
# tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=8388608k)
```

---

## 8. `memfd_create` — Anonymous Shared Memory

The modern way to create anonymous shared memory:

```c
#include <sys/memfd.h>

// Create an anonymous file in RAM:
int fd = memfd_create("myshm", MFD_CLOEXEC | MFD_ALLOW_SEALING);
ftruncate(fd, size);

// Map it:
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// Pass fd to another process via Unix socket (SCM_RIGHTS):
// ... (sendmsg with ancillary data containing fd)
// Other process maps it too → shared memory

// Seal it (prevent further modifications):
fcntl(fd, F_ADD_SEALS, F_SEAL_WRITE | F_SEAL_GROW | F_SEAL_SHRINK);
// Now the memory is read-only for everyone, even the creator
// Useful for sharing immutable data (e.g., font files in GUI frameworks)
```

`memfd_create` is used by: modern GUI toolkits (GTK, Qt) for texture sharing, container runtimes for OCI layers, `bpf_obj_pin`.

---

## 9. POSIX Message Queues (`mqueue`)

```c
#include <mqueue.h>

// Create:
struct mq_attr attr = {
    .mq_flags   = 0,
    .mq_maxmsg  = 10,     // Max messages in queue
    .mq_msgsize = 1024,   // Max message size
    .mq_curmsgs = 0,
};
mqd_t mqd = mq_open("/mymq", O_CREAT | O_RDWR, 0666, &attr);

// Send (with priority):
mq_send(mqd, "hello", 5, 0);  // Priority 0

// Receive (highest priority first):
char buf[1024];
unsigned int prio;
mq_receive(mqd, buf, sizeof(buf), &prio);

// Non-blocking:
mq_send(mqd, msg, len, 0) with O_NONBLOCK on mqd

// Async notification when message arrives:
struct sigevent sev = {
    .sigev_notify = SIGEV_SIGNAL,
    .sigev_signo  = SIGUSR1,
};
mq_notify(mqd, &sev);  // Get SIGUSR1 when message arrives

// Close and unlink:
mq_close(mqd);
mq_unlink("/mymq");
```

```bash
# POSIX mqueues are in /dev/mqueue/:
ls /dev/mqueue/

# System limits:
cat /proc/sys/fs/mqueue/msg_max    # Max messages per queue
cat /proc/sys/fs/mqueue/msgsize_max # Max message size
cat /proc/sys/fs/mqueue/queues_max # Max queues per user
```

---

## 10. Unix Domain Sockets for IPC

The most powerful and flexible IPC mechanism:

```c
// Create a pair of connected sockets (like pipe but bidirectional):
int fds[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, fds);
// fds[0] and fds[1] are connected

// Named socket (for unrelated processes):
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strncpy(addr.sun_path, "/tmp/myserver.sock", sizeof(addr.sun_path));

// Server:
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
listen(server_fd, 10);
int client_fd = accept(server_fd, NULL, NULL);

// Client:
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
connect(sock, (struct sockaddr *)&addr, sizeof(addr));

// KEY FEATURE: File descriptor passing (SCM_RIGHTS):
// Send an fd to another process:
struct msghdr msg = {};
struct iovec iov = { .iov_base = "!", .iov_len = 1 };
char cmsgbuf[CMSG_SPACE(sizeof(int))];
struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type  = SCM_RIGHTS;
cmsg->cmsg_len   = CMSG_LEN(sizeof(int));
*(int *)CMSG_DATA(cmsg) = fd_to_send;
msg.msg_control    = cmsgbuf;
msg.msg_controllen = sizeof(cmsgbuf);
msg.msg_iov        = &iov;
msg.msg_iovlen     = 1;
sendmsg(sock, &msg, 0);
```

Unix sockets are used by: X11/Wayland, D-Bus, systemd, Docker daemon, SSH agent, PulseAudio.

---

## 11. `eventfd` — Lightweight Notification

```c
#include <sys/eventfd.h>

// Create:
int efd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);

// Signal (add to counter):
uint64_t val = 1;
write(efd, &val, sizeof(val));  // Increments counter by val

// Wait (read resets to 0):
uint64_t count;
read(efd, &count, sizeof(count));  // Blocks until counter > 0
// count = how many times write was called (if EFD_SEMAPHORE: always 1)

// Use with epoll:
struct epoll_event ev = { .events = EPOLLIN, .data.fd = efd };
epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &ev);
// Readable when counter > 0

// Semaphore mode (EFD_SEMAPHORE):
// read() decrements by 1 each time (not reset to 0)
int efd = eventfd(0, EFD_SEMAPHORE);
```

`eventfd` is used by: `io_uring` for completion notification, `timerfd`, KVM for virtio, epoll-based event loops.

---

## 12. Choosing the Right IPC Mechanism

```
What you need:                       Use:
─────────────────────────────────────────────────────────────
One-way byte stream (parent↔child)   pipe(2)
One-way byte stream (unrelated)      FIFO or Unix socket
Bidirectional byte stream            Unix socket (socketpair)
Pass file descriptors                Unix socket + SCM_RIGHTS
Share large data buffer              POSIX shm or memfd_create
Fast inter-thread sync               Futex (pthread_mutex)
Fast inter-process counter           Atomic in shared memory
Message passing (small)              POSIX mqueue or Unix SOCK_DGRAM
Message passing (large)              Unix SOCK_STREAM
Inherited by children                Anonymous pipe or mmap(MAP_SHARED)
Named (any process can connect)      FIFO, Unix socket, or POSIX shm
Legacy application compatibility     SysV IPC
Notify without data                  eventfd
```

---

## 13. IPC in the Kernel: Namespace Isolation

All SysV IPC and POSIX IPC is namespace-scoped:

```c
// Each process has an IPC namespace:
struct ipc_namespace {
    struct idr       mq_idr;     // POSIX message queue IDs
    unsigned int     msg_ctlmax; // Max message size
    unsigned int     msg_ctlmnb; // Max queue size
    unsigned int     msg_ctlmni; // Max queues
    
    struct ipc_ids   ids[3];     // SysV: [0]=msg, [1]=sem, [2]=shm
    
    unsigned int     shm_ctlmax; // Max shared memory segment size
    unsigned int     shm_ctlall; // Max total shared memory
    unsigned int     shm_ctlmni; // Max segments
    // ...
};
```

```bash
# Create new IPC namespace (container):
unshare --ipc bash
# Inside: ipcs shows empty IPC resources
# Outside: original IPC resources still visible

# Docker uses separate IPC namespace per container by default
# --ipc=host: share host IPC namespace with container
```

---

## 14. Performance Comparison

```bash
# Benchmark: measure IPC throughput:
# pipe:
dd if=/dev/zero bs=1M count=1000 | dd of=/dev/null  # pipe throughput

# /dev/zero → pipe → /dev/null:
# Typical: 5-20 GB/s on modern hardware (zero-copy with splice)

# Shared memory: effectively memory bandwidth
# Typical: 30-50 GB/s (limited by cache/DRAM)

# Unix socket:
# Typical: 1-10 GB/s (copying involved)

# SysV message queue:
# Typical: 100k-1M messages/second

# Relative performance (single-core, message passing):
# shared memory (mmap):  ~500ns round-trip  (with futex synchronization)
# pipe:                  ~1-5µs round-trip
# Unix socket:           ~2-10µs round-trip
# SysV msgq:             ~10-50µs round-trip
# TCP localhost:          ~50-100µs round-trip
```

---

## 15. Observing IPC

```bash
# SysV IPC overview:
ipcs -a                    # All SysV IPC resources
ipcs -lm                   # Shared memory limits
ipcs -ls                   # Semaphore limits
ipcs -lq                   # Message queue limits

# Shared memory usage:
cat /proc/sysvipc/shm
# key     shmid perms   size  cpid   lpid  nattch  uid   gid   cuid  cgid ...

# POSIX shm:
ls -lh /dev/shm/           # Named POSIX shm objects

# Pipe fd pairs (as seen in /proc):
ls -la /proc/$$/fd | grep pipe
# lrwxrwxrwx 1 user user 64 ... 0 -> pipe:[12345]
# lrwxrwxrwx 1 user user 64 ... 1 -> pipe:[12345]  ← same inode = connected

# eventfd and other:
ls -la /proc/$$/fd | grep -E 'eventfd|anon_inode'

# bpftrace: trace pipe writes:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_write {
    @[comm] = count();
}' 
```

---

## 16. Phase 2 Summary

Congratulations — all 10 Phase 2 guides are complete:

| Day | Topic | Key Takeaway |
|-----|-------|-------------|
| 13 | `task_struct` | Process descriptor — identity, state, mm, files, signals, creds |
| 14 | Process creation | `fork()` → `copy_process()`, COW, `exec()` → ELF loading |
| 15 | Process lifecycle | Zombie, orphan, reparenting, `do_exit()`, `wait()` |
| 16 | Threads | `CLONE_*` flags, TID vs TGID, futexes, TLS |
| 17 | CFS scheduler | vruntime, rbtree run queue, context switch, preemption |
| 18 | RT scheduling | SCHED_FIFO/RR/DEADLINE, PI mutexes, PREEMPT_RT |
| 19 | Kernel preemption | `preempt_count`, `TIF_NEED_RESCHED`, CONFIG_PREEMPT |
| 20 | SMP balancing | Per-CPU run queues, domains, load_balance, NUMA |
| 21 | Signals | Delivery path, signal frames, `rt_sigreturn`, SA_* flags |
| 22 | IPC mechanisms | Pipes, SysV IPC, POSIX shm, memfd, eventfd |

**Phase 3 begins**: Memory Management — the buddy allocator in depth, virtual memory, page tables, demand paging, swap, and OOM.

---

## Mental Model Checkpoint

After Day 22, you should be able to:

1. Draw the kernel data structures for a pipe (ring buffer of pages, two file descriptors).
2. What guarantees does `PIPE_BUF` provide?
3. How is SysV shared memory implemented internally? What filesystem backs it?
4. What is the difference between `shm_open()` and `shmget()`?
5. When would you use `memfd_create` instead of POSIX shm?
6. What does `SCM_RIGHTS` allow?
7. Choose the right IPC for: a) passing a 4KB message between unrelated processes, b) sharing a 100MB buffer between 5 processes, c) notifying a process from an interrupt context.
8. How are SysV IPC resources isolated between containers?

---

## Key Source Files

```bash
fs/pipe.c                    # Pipe implementation
ipc/shm.c                    # SysV shared memory
ipc/msg.c                    # SysV message queues
ipc/sem.c                    # SysV semaphores
ipc/mqueue.c                 # POSIX message queues
mm/shmem.c                   # tmpfs (backing store for shm)
fs/eventfd.c                 # eventfd implementation
net/unix/af_unix.c           # Unix domain sockets
include/linux/ipc_namespace.h # IPC namespace structure
```
