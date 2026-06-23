# Day 9 — Interrupt Handling: Hardware Path

> **Estimated read time:** 90–120 minutes  
> **Goal:** Trace an interrupt from the device pin through the APIC, CPU exception handling, and into the kernel's IRQ subsystem.

---

## 1. Why Interrupts Exist

The alternative to interrupts is **polling**: the CPU loops checking whether a device has data ready. For an NVMe drive completing a request in 50µs, polling wastes enormous CPU cycles. With interrupts:

1. CPU submits request, moves on to other work
2. Device signals completion by asserting an interrupt line
3. CPU suspends current work, runs the interrupt handler, resumes

This is fundamentally more efficient but requires the CPU and OS to handle asynchronous events correctly and quickly.

The kernel's interrupt handling has two major phases:
- **Hardware path** (today): the electrical signal → CPU exception → kernel entry → IRQ identification → handler call
- **Software path** (Day 10): deferred work via softirqs, tasklets, and workqueues

---

## 2. The Hardware Side: How a Device Signals an Interrupt

### 2.1 Legacy ISA Interrupts (IRQ Lines)

In legacy systems, devices have dedicated interrupt lines (IRQ0–IRQ15) wired to the **Programmable Interrupt Controller** (PIC, Intel 8259A). The NIC on IRQ10 asserts a voltage on that wire; the PIC delivers it to the CPU.

Problems with legacy PICs:
- Only 15 usable IRQ lines (IRQ2 is used for PIC cascading)
- Fixed priority ordering
- No SMP support — all interrupts go to one CPU

Almost entirely obsolete on modern hardware but still present in emulators (QEMU's legacy devices).

### 2.2 APIC — Advanced Programmable Interrupt Controller

Modern x86 systems use the **APIC** system:

```
Device → I/O APIC → Local APIC (per CPU) → CPU
```

**I/O APIC** (`IOAPIC`):
- Receives interrupt signals from devices
- Has a redirection table: maps each input to a CPU, vector, and delivery mode
- Usually at physical address `0xFEC00000` (MMIO)
- Typical systems have 1–4 I/O APICs with 24 inputs each

**Local APIC** (`LAPIC`):
- One per CPU (at physical address `0xFEE00000` for `xAPIC`, MSR for `x2APIC`)
- Receives interrupts from I/O APIC and delivers to its CPU
- Also generates: timer interrupts, IPI (inter-processor interrupts), LINT0/LINT1 (NMI)

### 2.3 MSI — Message Signaled Interrupts

Modern PCIe devices use MSI/MSI-X instead of dedicated interrupt lines:

- Device writes a specific 32-bit value to a specific physical address
- The CPU/APIC treats this memory write as an interrupt signal
- MSI supports up to 32 vectors per device; MSI-X supports up to 2048

```c
// Requesting MSI-X in a PCIe driver:
int nvec = pci_msix_vec_count(pdev);    // How many vectors device supports
int err = pci_alloc_irq_vectors(pdev, 1, nvec, PCI_IRQ_MSIX);
// Now each vector has its own Linux IRQ number via pci_irq_vector(pdev, i)
```

MSI-X is used by: NVMe (one vector per queue), high-performance NICs (one vector per RX/TX queue pair), modern GPUs.

---

## 3. Interrupt Vectors

The CPU handles interrupts through **vectors** — a number from 0 to 255 identifying the interrupt type.

```
Vector numbers (x86_64):
0–31:    CPU exceptions (hardwired by Intel/AMD spec)
         0: #DE Divide Error
         1: #DB Debug
         2: NMI (Non-Maskable Interrupt)
         3: #BP Breakpoint
         4: #OF Overflow
         6: #UD Invalid Opcode
         8: #DF Double Fault
        13: #GP General Protection Fault
        14: #PF Page Fault
        ...

32–255:  Available for OS use (device interrupts, software interrupts)
         32–127:  Device interrupts (mapped by I/O APIC/MSI)
         128:     Old x86 Linux syscall vector (int 0x80)
         200-255: IPI vectors (inter-processor interrupts)
```

---

## 4. The IDT — Interrupt Descriptor Table

The CPU knows where to jump for each vector via the **IDT** (Interrupt Descriptor Table), a 256-entry array of **gate descriptors**.

```c
// arch/x86/include/asm/desc_defs.h
struct gate_struct {
    u16 offset_low;     // Bits 0-15 of handler address
    u16 segment;        // Code segment selector (kernel CS = __KERNEL_CS)
    struct idt_bits bits;  // Type, DPL, present bit
    u16 offset_middle;  // Bits 16-31 of handler address
    u32 offset_high;    // Bits 32-63 of handler address
    u32 reserved;
} __attribute__((packed));
```

The IDT lives at a fixed address in kernel memory. The CPU's `IDTR` register points to it.

```bash
# See IDT register (from crash or gdb):
# (gdb) monitor info registers  → shows IDTR base and limit
# In QEMU: info registers

# Kernel IDT setup:
sudo cat /proc/kallsyms | grep idt_table
# ffffffff82234000 D idt_table
```

Interrupt setup chain:

```c
// arch/x86/kernel/idt.c
void __init idt_setup_apic_and_irq_gates(void)
{
    // For each hardware interrupt vector 32-255:
    // Install the assembly trampoline interrupt_entry (or specific handlers)
    for (i = FIRST_EXTERNAL_VECTOR; i < FIRST_SYSTEM_VECTOR; i++) {
        set_intr_gate(i, irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR));
    }
}

// Exception gates (vectors 0-31):
void __init idt_setup_early_traps(void)
{
    set_intr_gate(X86_TRAP_DE, asm_exc_divide_error);
    set_intr_gate(X86_TRAP_PF, asm_exc_page_fault);
    set_intr_gate(X86_TRAP_GP, asm_exc_general_protection);
    // ...
}
```

---

## 5. When an Interrupt Fires: CPU State Changes

When a device triggers an interrupt (via I/O APIC → Local APIC), the CPU:

1. **Finishes the current instruction** (most interrupts are synchronous-ish at instruction boundaries; NMI is truly asynchronous)
2. **Checks IF flag** (interrupt flag in RFLAGS): if interrupts are disabled (`cli`), the interrupt is held pending in the APIC; if enabled, proceed
3. **Switches to the interrupt stack**: loads the `RSP` from the TSS's `IST` (Interrupt Stack Table) or regular kernel stack
4. **Pushes exception frame** on the new stack:

```
Old RSP    ← saved user/kernel stack pointer
SS         ← saved stack segment
RFLAGS     ← saved flags (including IF=1 that we're disabling)
CS         ← saved code segment
RIP        ← return address (instruction to resume after handler)
[Error code] ← only for some exceptions (#PF, #GP, etc.)
```

5. **Clears IF** (disables further interrupts — we're now in "hardirq context")
6. **Sets CS** to kernel code segment (Ring 0)
7. **Jumps to IDT handler** for this vector

This is all done in hardware in a single atomic operation — no kernel code runs between the interrupt firing and the CPU pushing the frame and jumping to the handler.

---

## 6. The Interrupt Entry Assembly

The assembly entry point for hardware interrupts is `irq_entries_start` in `arch/x86/entry/entry_64.S`:

```asm
/* arch/x86/entry/entry_64.S */

/* This is a trampoline — one for each IRQ vector 32-255.
 * They all share the body after pushing the vector number. */
SYM_CODE_START(irq_entries_start)
    vector = FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
        UNWIND_HINT_IRET_REGS
        pushq   $(~vector+0x80)   /* Push ~vector (negated for error detection) */
        jmp     common_interrupt
        .align  8
        vector = vector + 1
    .endr
SYM_CODE_END(irq_entries_start)

SYM_CODE_START_LOCAL(common_interrupt)
    SAVE_AND_SWITCH_TO_KERNEL_CR3   /* KPTI: switch page tables */
    
    /* At this point: hardware pushed SS,RSP,RFLAGS,CS,RIP onto stack.
     * We also pushed vector number. Save all general registers: */
    PUSH_AND_CLEAR_REGS             /* Push R15..RBX,RBP,RAX etc. → pt_regs */
    
    movq    %rsp, %rdi              /* pt_regs pointer as first argument */
    /* Get vector number (we pushed it as ~vector, decode it): */
    call    common_interrupt_return /* Don't call directly; use C wrapper */
    
    /* Actually calls: */
    /* __common_interrupt(regs, vector) in arch/x86/kernel/irq.c */
SYM_CODE_END(common_interrupt)
```

The C entry:

```c
// arch/x86/kernel/irq.c
__visible noinstr void __common_interrupt(struct pt_regs *regs, int vector)
{
    RCU_LOCKDEP_WARN(!rcu_is_watching(), "RCU not watching on irq entry");
    
    irq_enter_rcu();    // Marks we're in hardirq context
    
    handle_irq(irq_to_desc(vector), regs);  // Dispatch to IRQ handler
    
    irq_exit_rcu();     // Processes pending softirqs, marks hardirq exit
}
```

---

## 7. IRQ Subsystem: `irq_desc` and the IRQ Hierarchy

The kernel maintains one `struct irq_desc` per Linux IRQ number:

```c
// include/linux/irqdesc.h
struct irq_desc {
    struct irq_data         irq_data;     // Hardware IRQ data (chip, domain)
    irq_flow_handler_t      handle_irq;   // Top-level handler
    struct irqaction       *action;        // Linked list of ISRs (device handlers)
    unsigned int            status_use_accessors; // IRQ flags
    unsigned int            core_internal_state__do_not_mess_with_it;
    unsigned int            depth;         // Nesting depth (enabled/disabled count)
    unsigned int            irq_count;     // Statistics
    unsigned long           last_unhandled;
    unsigned int            irqs_unhandled;
    spinlock_t              lock;
    // ...
};
```

**Linux IRQ numbers vs hardware vectors**: Linux assigns its own IRQ numbers (independent of hardware vectors). The kernel maintains a mapping between the two:

```c
// The vector → irq_desc mapping:
struct irq_desc *irq_to_desc(unsigned int irq);  // irq → desc
unsigned int irq_find_mapping(struct irq_domain *domain, irq_hw_number_t hwirq);
```

### 7.1 IRQ Domains

Modern systems have hierarchical interrupt controllers (APIC → IOAPIC, or GIC → MSI controller on ARM). IRQ domains model this hierarchy:

```c
// include/linux/irqdomain.h
struct irq_domain {
    struct list_head link;
    const char *name;
    const struct irq_domain_ops *ops;
    struct irq_domain *parent;          // Parent domain (for hierarchical IRQ)
    void *host_data;
    unsigned int hwirq_max;
    // ...
};

// Mapping HW IRQ → Linux IRQ:
unsigned int virq = irq_create_mapping(domain, hwirq);
```

---

## 8. `request_irq` — Registering an Interrupt Handler

This is how drivers claim an IRQ:

```c
// include/linux/interrupt.h
int request_irq(unsigned int irq,           // Linux IRQ number
                irq_handler_t handler,       // Handler function
                unsigned long flags,         // IRQF_* flags
                const char *name,            // Name in /proc/interrupts
                void *dev_id);              // Cookie to identify this handler

// Modern alternative (prefers threaded IRQ):
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,        // Primary handler (atomic)
                         irq_handler_t thread_fn,      // Thread handler (can sleep)
                         unsigned long flags,
                         const char *name,
                         void *dev_id);

// Free it:
void free_irq(unsigned int irq, void *dev_id);
```

Important `IRQF_` flags:

```c
IRQF_SHARED       // Multiple devices share this IRQ line (legacy, calls all handlers)
IRQF_DISABLED     // (Obsolete) Run with interrupts disabled
IRQF_TRIGGER_RISING  // Edge-triggered on rising edge
IRQF_TRIGGER_FALLING // Edge-triggered on falling edge
IRQF_TRIGGER_HIGH    // Level-triggered, high level
IRQF_TRIGGER_LOW     // Level-triggered, low level
IRQF_ONESHOT      // IRQ line re-enabled only after thread_fn completes
                  // (for level-triggered interrupts with threaded handlers)
IRQF_NO_AUTOEN    // Don't enable the IRQ automatically after request
IRQF_PERCPU       // Per-CPU interrupt (local timer, etc.)
```

---

## 9. Writing an IRQ Handler

```c
// Handler signature — must be fast, cannot sleep:
irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;  // Cast the cookie back
    
    // 1. Verify this is our device (critical for IRQF_SHARED):
    u32 status = readl(dev->base + IRQ_STATUS_REG);
    if (!(status & OUR_IRQ_BIT))
        return IRQ_NONE;  // Not our interrupt
    
    // 2. Acknowledge the interrupt at the device (clear the IRQ source):
    // This prevents the device from re-asserting the same interrupt immediately.
    writel(status, dev->base + IRQ_CLEAR_REG);
    
    // 3. Do minimal work — just read what happened, queue for later:
    dev->last_status = status;
    dev->bytes_received = readl(dev->base + RX_COUNT_REG);
    
    // 4. Schedule deferred work (can't sleep here!):
    tasklet_schedule(&dev->tasklet);
    // OR:
    schedule_work(&dev->work);
    // OR just: wake_up(&dev->wait_queue);
    
    return IRQ_HANDLED;
}

// Requesting the IRQ in probe():
int err = request_irq(pdev->irq, my_irq_handler,
                      IRQF_SHARED, "my_driver", dev);
if (err) {
    dev_err(&pdev->dev, "Failed to request IRQ %d: %d\n", pdev->irq, err);
    return err;
}

// In driver remove/exit:
free_irq(pdev->irq, dev);
```

### 9.1 Return Values

```c
IRQ_NONE       // This interrupt wasn't from our device (for shared IRQs)
IRQ_HANDLED    // Successfully handled
IRQ_WAKE_THREAD // Wake the threaded handler (request_threaded_irq)
```

---

## 10. The IRQ Handler Call Chain

```
Hardware assertion
    │
    ▼
I/O APIC redirects to Local APIC
    │
    ▼
CPU: hardware pushes exception frame, disables IF, jumps to IDT entry
    │
    ▼
irq_entries_start (assembly trampoline)
    │  saves all registers (pt_regs), switches to interrupt stack
    ▼
__common_interrupt(pt_regs, vector)
    │  calls irq_enter_rcu()  ← marks hardirq start
    ▼
handle_irq(irq_desc)
    │  flow handler: handle_level_irq() or handle_edge_irq() or handle_fasteoi_irq()
    ▼
handle_irq_event(irq_desc)
    │
    ▼
handle_irq_event_percpu(irq_desc, irqaction)
    │  walks irqaction list (all handlers for this IRQ):
    │  for each action: res = action->handler(irq, action->dev_id)
    ▼
my_irq_handler(irq, dev_id)  ← YOUR CODE
    │  returns IRQ_HANDLED or IRQ_NONE
    ▼
(returns up the chain)
irq_exit_rcu()  ← checks for pending softirqs
    │  if softirqs pending: invoke_softirq()
    ▼
Return from interrupt (iret / sysretq)
    │  restores all registers, re-enables interrupts
    ▼
Resumes previously running code
```

### 10.1 Flow Handlers

The `handle_irq` call dispatches through a **flow handler** that implements the interrupt type:

- `handle_level_irq()`: Level-triggered (line stays asserted until acknowledged). Masks the IRQ line before calling handlers, unmasks after. Used for PIC.
- `handle_edge_irq()`: Edge-triggered (brief pulse). Doesn't mask. Used for APIC with edge trigger.
- `handle_fasteoi_irq()`: Modern APIC path. Sends EOI (End Of Interrupt) to APIC after all handlers run. Most common.
- `handle_percpu_irq()`: For per-CPU interrupts (local timer, IPI).
- `handle_bad_irq()`: No handler registered — spurious interrupt.

```c
// kernel/irq/chip.c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);
    
    if (!irq_may_run(desc))
        goto out;
    
    // Call all registered handlers:
    handle_irq_event(desc);
    
out:
    cond_unmask_eoi_irq(desc, chip);  // Send EOI to APIC
    raw_spin_unlock(&desc->lock);
}
```

---

## 11. EOI — End of Interrupt

After handling an interrupt, the handler must tell the APIC it's done. The APIC won't deliver another interrupt of the same or lower priority until it receives an EOI:

```c
// The APIC EOI register:
// Write any value to LAPIC EOI register (offset 0xB0 from LAPIC base)
// This tells the local APIC: "I'm done handling the interrupt"
// The APIC then notifies the I/O APIC to re-arm the interrupt line

// In kernel: done automatically by the flow handler (handle_fasteoi_irq)
// via chip->irq_eoi(irq_data)
```

Getting EOI right is crucial:
- Too early: interrupt can re-enter before you're done (race condition)
- Too late: the device can't send another interrupt (missed events)
- Missing: the system stops receiving that interrupt entirely (IRQ disabled state)

---

## 12. Interrupt Stacks

On x86_64, each CPU has multiple stacks for interrupt handling:

```c
// arch/x86/include/asm/processor.h
// The TSS (Task State Segment) defines stack pointers for exceptions:
struct tss_struct {
    struct x86_hw_tss  x86_tss;
    // ...
};

// arch/x86/include/asm/page_64_types.h
// Stack sizes:
#define THREAD_SIZE    16384  // 16KB per-thread kernel stack (was 8KB, doubled)
#define IRQ_STACK_SIZE 16384  // 16KB per-CPU interrupt stack
```

Stack switching for interrupts:

1. **Thread kernel stack** (16KB): normal syscall processing, process context
2. **IRQ stack** (16KB per-CPU): used for hardware interrupt handling  
   Defined in `irq_stack_ptr` per-CPU variable
3. **IST stacks** (4KB each, 7 total): special stacks for specific exceptions
   - IST1: `#DF` Double Fault (can't use regular stack — it might be corrupt)
   - IST2: `#NMI` Non-Maskable Interrupt
   - IST3: `#DB` Debug
   - IST4: Machine Check Exception

```bash
# See kernel stacks in use:
sudo cat /proc/$(pgrep myprocess)/wchan  # What kernel function the process is blocked in
sudo cat /proc/$(pgrep myprocess)/stack  # Full kernel stack trace
# [<0>] futex_wait_queue+0x56/0xa0
# [<0>] futex_wait+0x106/0x1f0
# [<0>] do_futex+0x1b0/0x1190
# [<0>] __x64_sys_futex+0x155/0x1c0
# [<0>] do_syscall_64+0x5c/0xf0
# [<0>] entry_SYSCALL_64_after_hwframe+0x6e/0xd8
```

---

## 13. Interrupt Affinity — Which CPU Handles Which IRQ

By default, interrupts can be handled by any CPU. This can cause cache thrashing. The kernel and administrators can pin IRQs to specific CPUs:

```bash
# See current IRQ → CPU affinity:
cat /proc/interrupts
#           CPU0       CPU1       CPU2       CPU3
#  24:          0          0   1234567          0  PCI-MSI 524288-edge nvme0q1
#  25:          0          0          0   7654321  PCI-MSI 524289-edge nvme0q2
#  26:   9876543          0          0          0  PCI-MSI 524290-edge nvme0q3

# See IRQ affinity (which CPUs can handle IRQ 24):
cat /proc/irq/24/smp_affinity
# 00000004  ← bitmask: bit 2 = CPU2

cat /proc/irq/24/smp_affinity_list
# 2          ← Human-readable: CPU2

# Pin IRQ 24 to CPU0 and CPU1:
echo 3 > /proc/irq/24/smp_affinity          # bitmask: 0b11 = CPU0+CPU1
echo 0-1 > /proc/irq/24/smp_affinity_list   # Equivalent, list format

# Automate with irqbalance:
sudo apt install irqbalance
# irqbalance daemon distributes interrupts based on CPU load
```

### 13.1 NAPI and Interrupt Mitigation in Network Drivers

High-speed NICs can generate hundreds of thousands of interrupts per second. Too many interrupts kill performance (interrupt processing overhead > actual work). Solution: **interrupt coalescing** + **NAPI** (Day 9 preview of Day 46):

```c
// NIC driver: don't enable interrupts immediately after handling
// Instead, poll until no more packets, then re-enable:
irqreturn_t nic_rx_irq_handler(int irq, void *dev_id)
{
    struct nic_ring *ring = dev_id;
    
    // Disable IRQ for this queue (don't re-arm the interrupt):
    napi_schedule(&ring->napi);  // Schedule NAPI poll
    
    return IRQ_HANDLED;
}

// NAPI poll function (runs as softirq):
int nic_poll(struct napi_struct *napi, int budget)
{
    int received = 0;
    
    while (received < budget && packets_available()) {
        process_packet(ring);
        received++;
    }
    
    if (received < budget) {
        // No more packets — re-enable IRQ:
        napi_complete(napi);
        enable_irq(ring->irq);
    }
    
    return received;
}
```

---

## 14. IPI — Inter-Processor Interrupts

CPUs send each other IPIs for coordination:

```c
// Common IPI uses:
// TLB shootdown: when one CPU changes page tables, others must flush their TLBs
smp_call_function_many(mask, flush_tlb_func, &info, 1);
// Sends an IPI to all CPUs in 'mask', waits for completion

// Send work to another CPU:
smp_call_function_single(cpu, my_func, my_arg, 0);  // Non-blocking

// Reschedule another CPU (when a higher-priority task is queued for it):
smp_send_reschedule(cpu);
// CPU receives IPI → runs scheduler → picks up new task
```

IPI vectors (x86_64):

```c
// arch/x86/include/asm/irq_vectors.h
#define RESCHEDULE_VECTOR           0xfc  // Wake up the scheduler
#define CALL_FUNCTION_VECTOR        0xfb  // smp_call_function
#define CALL_FUNCTION_SINGLE_VECTOR 0xfa  // smp_call_function_single
#define IRQ_WORK_VECTOR             0xf9  // irq_work_queue (deferred in-hardirq work)
#define REBOOT_VECTOR               0xf8  // System reboot
```

---

## 15. Non-Maskable Interrupts (NMI)

NMIs bypass the CPU's interrupt enable/disable (IF flag). They always fire.

Uses:
- Hardware watchdog (NMI watchdog: detects CPU lockup)
- Machine Check Exceptions (MCE: hardware error reporting)
- Perf NMI (PMU overflow)
- kdump / crash dump triggers

```c
// Register an NMI handler:
int register_nmi_handler(unsigned int type, nmi_handler_t handler,
                         unsigned long flags, const char *name);
// type: NMI_LOCAL, NMI_UNKNOWN, NMI_SERR, NMI_IO_CHECK

// Example: perf NMI handler
static int perf_event_nmi_handler(unsigned int cmd, struct pt_regs *regs)
{
    // cmd: NMI_HANDLED or NMI_DONE
    // Process PMU overflow events
    return perf_sample_event_took_too_long() ? NMI_HANDLED : NMI_DONE;
}
```

```bash
# NMI watchdog: detects CPU hard lockup (stuck for >10 seconds):
cat /proc/sys/kernel/nmi_watchdog
# 1  ← enabled

# If a CPU is stuck (infinite loop with interrupts disabled):
# NMI fires (cannot be masked) → kernel prints lockup trace → panic (if configured)
```

---

## 16. Interrupt Context Rules (Critical!)

When executing in interrupt context (`in_interrupt()` returns true):

**CANNOT do:**
```c
// Sleep or block:
msleep(100);            // FORBIDDEN
schedule();             // FORBIDDEN
wait_event(...);        // FORBIDDEN
kmalloc(size, GFP_KERNEL);  // FORBIDDEN (may sleep in reclaim)

// Take sleeping locks:
mutex_lock(&m);         // FORBIDDEN (may sleep)
down(&sem);             // FORBIDDEN (may sleep)

// Access user memory:
copy_from_user(...);    // FORBIDDEN (may page fault → sleep)
get_user(...);          // FORBIDDEN
```

**CAN do:**
```c
// Take spinlocks:
spin_lock(&lock);       // OK (but be careful about lock ordering)
spin_lock_irqsave(&lock, flags);  // Safer

// Atomic operations:
atomic_inc(&counter);   // OK

// Schedule deferred work:
tasklet_schedule(&t);   // OK
schedule_work(&w);      // OK (just queues it, doesn't block)
wake_up(&wq);           // OK (just marks, doesn't block)

// Non-sleeping allocation:
kmalloc(size, GFP_ATOMIC);  // OK (uses atomic reserve, may fail)

// Fast I/O:
readl(iobase + REG);    // OK — MMIO register reads
writel(val, iobase);    // OK
```

**Why no sleeping?** The interrupt handler preempts whatever was running. If the handler sleeps, it blocks the entire CPU — whatever was running can't resume. This causes system-wide stalls. With SMP this is less catastrophic (other CPUs continue), but it's still wrong because:
- The preempted task might hold locks the interrupt handler needs (deadlock)
- Sleeping requires scheduler, which requires the CPU is in schedulable state

---

## 17. `irq_enter` / `irq_exit` — Accounting

```c
// kernel/softirq.c
void irq_enter_rcu(void)
{
    // Increment interrupt nesting count (preempt_count)
    // This is what in_interrupt() checks
    __irq_enter_raw();  // preempt_count += HARDIRQ_OFFSET
    
    // Tell RCU we're in an interrupt:
    rcu_irq_enter();
}

void irq_exit_rcu(void)
{
    // Decrement interrupt nesting:
    preempt_count_sub(HARDIRQ_OFFSET);
    
    // Check if softirqs are pending:
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();  // Run pending softirqs
    
    rcu_irq_exit();
}
```

The `preempt_count` register per-CPU variable encodes multiple counts:

```c
// include/linux/preempt.h
// preempt_count bits:
// [0:7]   = preemption disable count (spin_lock increments this)
// [8:15]  = softirq nesting count
// [16:19] = hardirq nesting count
// [20]    = NMI count
// [21]    = Idle count

#define in_interrupt()   (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK | NMI_MASK))
#define in_irq()         (preempt_count() & HARDIRQ_MASK)
#define in_softirq()     (preempt_count() & SOFTIRQ_MASK)
#define in_nmi()         (preempt_count() & NMI_MASK)
#define in_serving_softirq() (preempt_count() & SOFTIRQ_OFFSET)
```

---

## 18. `/proc/interrupts` — Reading Interrupt Statistics

```bash
cat /proc/interrupts
#            CPU0       CPU1       CPU2       CPU3
#   0:         17          0          0          0  IO-APIC    2-edge  timer
#   1:          2          0          0          0  IO-APIC    1-edge  i8042
#   8:          1          0          0          0  IO-APIC    8-edge  rtc0
#   9:          0          0          0          0  IO-APIC    9-fasteoi  acpi
#  16:          0          0          0          0  IO-APIC   16-fasteoi  i801_smbus
#  24:          0          0   2834756          0  PCI-MSI 524288-edge  nvme0q1
#  25:          0          0          0   1928374  PCI-MSI 524289-edge  nvme0q2
# ...
# LOC:    4567890    4123456    3987654    4234567  Local timer interrupts
# CAL:        345        234        189        267  Function call interrupts
# TLB:    1234567     987654     876543     765432  TLB shootdowns
# RES:      12345      23456      34567      45678  Rescheduling interrupts
# ERR:          0                                   Error interrupts
# MIS:          0                                   Missed interrupts

# Fields:
# - IRQ number
# - Count per CPU
# - Controller type
# - Trigger mode
# - Device name
```

Key things to look for:
- **LOC** (Local timer): should be ~HZ×seconds×CPUs (e.g., 250 Hz × 60s × 4 CPUs = 60,000)
- **TLB**: high count means lots of mmap/munmap/page table changes
- **RES**: high count means lots of cross-CPU scheduling (workload is moving between CPUs)
- Unbalanced IRQ counts: one CPU handling all interrupts (configure `irqbalance`)

---

## 19. Spurious Interrupts and IRQ Debugging

```bash
# Spurious interrupts (no handler claimed them):
cat /proc/interrupts | grep ERR

# Enable IRQ debug output:
echo 1 > /proc/sys/kernel/irqaffinity

# ftrace IRQ events:
cd /sys/kernel/debug/tracing
echo 1 > events/irq/irq_handler_entry/enable
echo 1 > events/irq/irq_handler_exit/enable
cat trace_pipe | grep "irq=24"  # Watch IRQ 24 events

# perf: count IRQ events:
sudo perf stat -e 'irq:irq_handler_entry' -a sleep 1
# irq:irq_handler_entry    12345      # 12345 interrupt entries per second

# bpftrace: measure interrupt handler latency:
sudo bpftrace -e '
tracepoint:irq:irq_handler_entry { @ts[args->irq] = nsecs; }
tracepoint:irq:irq_handler_exit  /@ts[args->irq]/ {
    @lat[args->irq] = hist(nsecs - @ts[args->irq]);
    delete(@ts[args->irq]);
}'
```

---

## 20. Mental Model Checkpoint

After Day 9, you should be able to:

1. Trace an interrupt from device assertion through I/O APIC → Local APIC → CPU exception frame.
2. Explain the difference between legacy PIC, APIC, and MSI/MSI-X.
3. What is the IDT and what is an interrupt vector?
4. What does the CPU push on the stack when an interrupt fires?
5. Write a correct IRQ handler: what must you do, what are you forbidden from doing?
6. What is EOI and why is it important?
7. What is the difference between edge-triggered and level-triggered IRQs?
8. What stacks does the kernel use for interrupt handling?
9. How do you pin an IRQ to a specific CPU?
10. Interpret the output of `/proc/interrupts`.

---

## Key Source Files

```bash
arch/x86/entry/entry_64.S         # irq_entries_start, common_interrupt
arch/x86/kernel/irq.c             # __common_interrupt, handle_irq
arch/x86/kernel/idt.c             # IDT setup and gate descriptors
kernel/irq/handle.c               # handle_irq_event, irq handler dispatch
kernel/irq/chip.c                 # Flow handlers: handle_fasteoi_irq, handle_edge_irq
kernel/irq/manage.c               # request_irq, free_irq, IRQ affinity
include/linux/interrupt.h         # IRQF_* flags, irqreturn_t, request_irq prototypes
include/linux/irqdesc.h           # struct irq_desc
arch/x86/include/asm/irq_vectors.h  # Vector numbers, IPI vectors
Documentation/core-api/irq/      # Official IRQ documentation
```

---

## Summary

Hardware interrupts follow a precise path: device signals interrupt → I/O APIC routes to Local APIC → CPU saves state (pushes exception frame) and jumps to IDT handler → assembly trampoline saves all registers → `__common_interrupt()` calls `handle_irq()` → flow handler calls your `irqaction::handler` → handler returns → `irq_exit_rcu()` processes pending softirqs → CPU restores state and resumes.

Key invariants in hardirq context:
- No sleeping (no `mutex_lock`, no `kmalloc(GFP_KERNEL)`, no `schedule()`)
- Can take spinlocks but not sleeping locks
- Can use `GFP_ATOMIC` allocations (no reclaim, may fail)
- Must return quickly (microseconds, not milliseconds)
- Must acknowledge the interrupt at the device to prevent re-entry

The `preempt_count` variable tracks context (hardirq, softirq, preempt-disabled) and gates what operations are legal.

Tomorrow: the software path — softirqs, tasklets, workqueues, and how the kernel safely defers heavier work from interrupt context.
