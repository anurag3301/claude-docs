# STM32F411 EXTI & NVIC — Complete Reference Guide

---

## Table of Contents

1. [Introduction to Interrupts on the STM32F411](#1-introduction-to-interrupts-on-the-stm32f411)
2. [NVIC — Nested Vectored Interrupt Controller Overview](#2-nvic--nested-vectored-interrupt-controller-overview)
3. [The Vector Table](#3-the-vector-table)
4. [Interrupt Priority — Pre-emption and Sub-priority](#4-interrupt-priority--pre-emption-and-sub-priority)
5. [NVIC Registers (Bare-Metal)](#5-nvic-registers-bare-metal)
6. [Bare-Metal NVIC Configuration](#6-bare-metal-nvic-configuration)
7. [CMSIS NVIC Functions](#7-cmsis-nvic-functions)
8. [HAL NVIC Configuration](#8-hal-nvic-configuration)
9. [EXTI — External Interrupt/Event Controller Overview](#9-exti--external-interruptevent-controller-overview)
10. [EXTI Line Mapping — SYSCFG and GPIO Ports](#10-exti-line-mapping--syscfg-and-gpio-ports)
11. [EXTI Registers (Bare-Metal)](#11-exti-registers-bare-metal)
12. [Bare-Metal EXTI Configuration — GPIO Interrupt](#12-bare-metal-exti-configuration--gpio-interrupt)
13. [HAL EXTI Configuration](#13-hal-exti-configuration)
14. [EXTI Interrupt vs Event Mode](#14-exti-interrupt-vs-event-mode)
15. [Software-Triggered EXTI](#15-software-triggered-exti)
16. [Button Debouncing with EXTI](#16-button-debouncing-with-exti)
17. [EXTI and Low-Power Mode Wakeup](#17-exti-and-low-power-mode-wakeup)
18. [Interrupt Latency and Tail-Chaining](#18-interrupt-latency-and-tail-chaining)
19. [Common Pitfalls and Debugging](#19-common-pitfalls-and-debugging)
20. [Interview Questions and Answers](#20-interview-questions-and-answers)

---

## 1. Introduction to Interrupts on the STM32F411

An interrupt is a hardware signal that causes the CPU to suspend its current execution path and immediately jump to a dedicated handler function, allowing the system to respond to time-sensitive events (a UART byte arriving, a timer overflowing, a button press, a DMA transfer completing) without the CPU having to continuously poll for them. Two components work together on the STM32F411 to make this possible:

- **NVIC (Nested Vectored Interrupt Controller)**: part of the Cortex-M4 CORE itself (not an STM32-specific peripheral) — manages interrupt priority, enabling/disabling, pending status, and the actual vectoring (jumping) to the correct handler.
- **EXTI (EXTernal Interrupt/event controller)**: an STM32-specific PERIPHERAL that detects edge/level transitions on GPIO pins (and a handful of internal signals) and asserts an interrupt request line INTO the NVIC.

Understanding the division of labor here is genuinely important: EXTI is the "detector" — it watches a signal and decides WHEN something interesting happened. NVIC is the "router/scheduler" — it decides WHETHER and WHEN the CPU should actually respond to that (or any other) interrupt request, based on enable state and priority.

```
GPIO pin transition ──► EXTI (edge/level detection, line masking) ──► NVIC (priority,
                                                                         enable, vectoring)
                                                                              │
                                                                              ▼
                                                                     CPU jumps to ISR
```

---

## 2. NVIC — Nested Vectored Interrupt Controller Overview

The NVIC is a standard ARM Cortex-M architectural component — its register layout and core behavior are IDENTICAL across virtually all Cortex-M3/M4/M7 microcontrollers from any vendor, not just STM32. This is a deliberate ARM design choice enabling portable interrupt-handling code across the entire Cortex-M ecosystem.

### Key NVIC Capabilities

| Capability          | Description                                                       |
|------------------------|------------------------------------------------------------------|
| **Nesting**             | A higher-priority interrupt can preempt (interrupt) a currently-executing lower-priority interrupt handler|
| **Vectoring**            | Each interrupt source has its own dedicated handler address in the vector table — no software dispatch/polling needed to determine which interrupt fired|
| **Priority levels**       | STM32F411 implements 4 priority bits (16 levels: 0–15, 0=highest priority)|
| **Tail-chaining**         | Back-to-back pending interrupts are serviced without the full exception-return/exception-entry overhead between them (Section 18)|
| **Late-arriving**          | A higher-priority interrupt arriving DURING another interrupt's entry sequence can take priority before the first handler even begins|

### STM32F411 Interrupt Sources

The STM32F411 implements **85 maskable interrupt sources** (IRQ0 through IRQ84 in the vector table, varying slightly by exact part/package), covering every peripheral's interrupt-capable events: UART RX/TX, timer updates/captures, DMA stream completion, ADC conversion complete, I2C events/errors, EXTI lines, USB events, and more.

---

## 3. The Vector Table

The vector table is an array of function pointers (addresses), located at the START of Flash memory (address 0x0800_0000, or wherever remapped via `VTOR`), that the Cortex-M4 hardware reads directly to determine WHERE to jump when a given exception/interrupt occurs — no software lookup or dispatch table search is needed; the hardware itself indexes directly into this table.

```c
/* Conceptual structure (actual definition is in the startup .s/.c file) */
typedef void (*IRQHandler)(void);

IRQHandler vectorTable[] __attribute__((section(".isr_vector"))) = {
    (IRQHandler)&_estack,        /* Initial stack pointer (not a handler — special case) */
    Reset_Handler,                /* IRQ -15 (reset) */
    NMI_Handler,                   /* IRQ -14 */
    HardFault_Handler,              /* IRQ -13 */
    MemManage_Handler,               /* IRQ -12 */
    BusFault_Handler,                 /* IRQ -11 */
    UsageFault_Handler,                /* IRQ -10 */
    0, 0, 0, 0,                         /* Reserved */
    SVC_Handler,                         /* IRQ -5 */
    DebugMon_Handler,                     /* IRQ -4 */
    0,                                     /* Reserved */
    PendSV_Handler,                        /* IRQ -2 */
    SysTick_Handler,                        /* IRQ -1 */
    /* --- STM32-specific peripheral interrupts start here, IRQ0+ --- */
    WWDG_IRQHandler,                         /* IRQ0 */
    PVD_IRQHandler,                           /* IRQ1 */
    TAMP_STAMP_IRQHandler,                     /* IRQ2 */
    RTC_WKUP_IRQHandler,                        /* IRQ3 */
    FLASH_IRQHandler,                            /* IRQ4 */
    RCC_IRQHandler,                               /* IRQ5 */
    EXTI0_IRQHandler,                              /* IRQ6 */
    EXTI1_IRQHandler,                               /* IRQ7 */
    EXTI2_IRQHandler,                                /* IRQ8 */
    EXTI3_IRQHandler,                                 /* IRQ9 */
    EXTI4_IRQHandler,                                  /* IRQ10 */
    DMA1_Stream0_IRQHandler,                            /* IRQ11 */
    /* ... continues through IRQ84 ... */
};
```

### VTOR — Vector Table Offset Register

```c
/* Relocate the vector table — essential for bootloaders jumping to an
 * application at a non-zero Flash offset */
SCB->VTOR = 0x08010000;   /* Application starts at 64KB offset */
```

This is the EXACT mechanism referenced in the USB document's bootloader discussion — without correctly updating `VTOR` after jumping to an application located somewhere other than the chip's default boot address, the CPU would continue using the BOOTLOADER's vector table for all subsequent interrupts, jumping to the wrong (bootloader's own) handler addresses when the APPLICATION expects its own handlers to run.

### Negative vs Positive IRQ Numbers

Negative IRQ numbers (-15 through -1) are **architecturally-defined Cortex-M system exceptions** (Reset, NMI, HardFault, SysTick, etc.) — identical across every Cortex-M chip from every vendor. Positive IRQ numbers (0 and up) are **vendor/device-specific peripheral interrupts** — their exact assignment (which IRQ number corresponds to which peripheral) is defined by ST specifically for the STM32F411, documented in the device's reference manual/datasheet interrupt mapping table, and will differ on a different STM32 family member or a different vendor's Cortex-M chip entirely.

---

## 4. Interrupt Priority — Pre-emption and Sub-priority

The STM32F411 implements **4 priority bits** in each interrupt's priority register field, giving 16 distinct priority levels (0=highest urgency, 15=lowest). These 4 bits can be SPLIT between two conceptually different priority dimensions, controlled by the **Priority Grouping** configuration:

### Preemption Priority vs Sub-priority

- **Preemption priority**: determines whether THIS interrupt can INTERRUPT (preempt) an already-executing LOWER preemption-priority interrupt handler.
- **Sub-priority**: ONLY matters when two pending interrupts have the SAME preemption priority and the NVIC must decide which one to service FIRST (it does NOT allow preemption between same-preemption-priority interrupts — a lower-sub-priority interrupt cannot interrupt an already-running higher-preemption-priority-but-equal-or-lower-sub-priority handler).

### Priority Grouping Configuration

```c
/* NVIC_PriorityGroupConfig — splits the 4 available priority bits between
 * preemption and sub-priority */

NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);
/* Group 4 = 4 bits preemption, 0 bits sub-priority (16 preemption levels,
 * no sub-priority distinction) — the MOST COMMON configuration in practice,
 * since most applications don't need the sub-priority distinction at all */

/* Other groupings split the 4 bits differently: */
/* PRIORITYGROUP_0: 0 bits preempt, 4 bits sub  (no preemption at all between
 *                   any two interrupts — rarely useful)                    */
/* PRIORITYGROUP_1: 1 bit preempt (2 levels), 3 bits sub (8 levels)         */
/* PRIORITYGROUP_2: 2 bits preempt (4 levels), 2 bits sub (4 levels)        */
/* PRIORITYGROUP_3: 3 bits preempt (8 levels), 1 bit sub (2 levels)         */
/* PRIORITYGROUP_4: 4 bits preempt (16 levels), 0 bits sub                   */
```

```c
/* Setting an individual interrupt's priority (with grouping = 4, this is
 * purely a preemption priority, 0-15, no sub-priority component) */
NVIC_SetPriority(EXTI0_IRQn, 5);

/* HAL equivalent, explicit preemption + sub priority */
HAL_NVIC_SetPriority(EXTI0_IRQn, 5, 0);
```

### Why Priority Grouping Matters in Practice

Choosing PRIORITYGROUP_4 (all bits to preemption, none to sub-priority) is the right default for the large majority of applications — it gives the maximum number of distinct PREEMPTION levels (the dimension that actually matters for real-time behavior: can interrupt A interrupt currently-running interrupt B), and avoids the genuinely confusing situation of two interrupts sharing a preemption level needing a tie-breaker sub-priority decision, which is rarely a meaningful distinction applications actually need to make deliberately.

---

## 5. NVIC Registers (Bare-Metal)

The NVIC's registers live in the Cortex-M4's System Control Space (SCS), part of the Private Peripheral Bus — NOT part of the normal AHB/APB bus matrix (covered in the Bus Architecture document).

| Register Group | Address Base | Purpose                                            |
|--------------------|--------------|------------------------------------------------|
| `NVIC_ISER[0..2]`  | 0xE000E100   | Interrupt Set-Enable Registers (writing 1 enables an IRQ; 3 registers cover up to 96 IRQs)|
| `NVIC_ICER[0..2]`  | 0xE000E180   | Interrupt Clear-Enable Registers (writing 1 disables an IRQ)|
| `NVIC_ISPR[0..2]`  | 0xE000E200   | Interrupt Set-Pending Registers (writing 1 manually pends an IRQ — software-triggerable)|
| `NVIC_ICPR[0..2]`  | 0xE000E280   | Interrupt Clear-Pending Registers (writing 1 clears a pending IRQ)|
| `NVIC_IABR[0..2]`  | 0xE000E300   | Interrupt Active Bit Registers (read-only — is this IRQ currently being serviced)|
| `NVIC_IPR[0..20]`  | 0xE000E400   | Interrupt Priority Registers (8 bits per IRQ, only the top 4 bits implemented on STM32F4)|

### Why Multiple Array Elements (ISER[0], ISER[1], ISER[2])

With 85 possible peripheral interrupts but only 32 bits per register, the IRQ enable/pending/active state is spread across multiple consecutive 32-bit registers — IRQ0-31 in element [0], IRQ32-63 in element [1], IRQ64-95 (covering up through IRQ84) in element [2]. CMSIS functions (Section 7) handle this indexing automatically; bare-metal code accessing these registers directly must compute the correct array index and bit position manually.

```c
/* Manually enabling IRQ40 (e.g., some peripheral whose IRQn = 40) without
 * CMSIS helper functions — demonstrates the underlying indexing */
uint8_t irqn = 40;
NVIC->ISER[irqn / 32] = (1U << (irqn % 32));
/* irqn/32 = 1 (second register), irqn%32 = 8 (bit 8 within that register) */
```

---

## 6. Bare-Metal NVIC Configuration

```c
/*
 * Direct register manipulation, without CMSIS helper functions —
 * shown for understanding; CMSIS functions (Section 7) are strongly
 * preferred in real code for correctness and readability
 */

void NVIC_EnableIRQ_BareMetal(uint8_t irqn)
{
    NVIC->ISER[irqn >> 5] = (1U << (irqn & 0x1F));
}

void NVIC_DisableIRQ_BareMetal(uint8_t irqn)
{
    NVIC->ICER[irqn >> 5] = (1U << (irqn & 0x1F));
}

void NVIC_SetPriority_BareMetal(uint8_t irqn, uint8_t priority)
{
    /* STM32F4 implements only the top 4 bits of each 8-bit priority field */
    NVIC->IP[irqn] = (priority << 4) & 0xF0;
}

/* Example usage: enable and configure EXTI0 interrupt */
NVIC_SetPriority_BareMetal(EXTI0_IRQn, 5);
NVIC_EnableIRQ_BareMetal(EXTI0_IRQn);
```

---

## 7. CMSIS NVIC Functions

CMSIS (Cortex Microcontroller Software Interface Standard) provides a vendor-neutral, ARM-defined C API for NVIC access — this is what virtually all real-world code (bare-metal AND HAL-based) actually uses, rather than hand-rolled register manipulation.

```c
/* Core CMSIS NVIC functions (declared in core_cm4.h) */

void NVIC_SetPriorityGrouping(uint32_t PriorityGroup);
uint32_t NVIC_GetPriorityGrouping(void);

void NVIC_EnableIRQ(IRQn_Type IRQn);
void NVIC_DisableIRQ(IRQn_Type IRQn);

void NVIC_SetPendingIRQ(IRQn_Type IRQn);
void NVIC_ClearPendingIRQ(IRQn_Type IRQn);
uint32_t NVIC_GetPendingIRQ(IRQn_Type IRQn);

uint32_t NVIC_GetActive(IRQn_Type IRQn);

void NVIC_SetPriority(IRQn_Type IRQn, uint32_t priority);
uint32_t NVIC_GetPriority(IRQn_Type IRQn);

void NVIC_SystemReset(void);   /* Triggers a full system reset via AIRCR */
```

```c
/* Typical bare-metal initialization sequence using CMSIS functions */
void EXTI0_Interrupt_Setup(void)
{
    NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);
    NVIC_SetPriority(EXTI0_IRQn, 5);
    NVIC_EnableIRQ(EXTI0_IRQn);
}
```

### IRQn_Type Enum

```c
/* Defined per-device in the CMSIS device header (stm32f411xe.h or similar) —
 * gives meaningful, type-checked names instead of raw integers */
typedef enum {
    NonMaskableInt_IRQn  = -14,
    HardFault_IRQn       = -13,
    /* ... */
    SysTick_IRQn         = -1,
    WWDG_IRQn             = 0,
    PVD_IRQn               = 1,
    TAMP_STAMP_IRQn        = 2,
    RTC_WKUP_IRQn           = 3,
    FLASH_IRQn               = 4,
    RCC_IRQn                  = 5,
    EXTI0_IRQn                 = 6,
    EXTI1_IRQn                  = 7,
    EXTI2_IRQn                   = 8,
    EXTI3_IRQn                    = 9,
    EXTI4_IRQn                     = 10,
    DMA1_Stream0_IRQn                = 11,
    /* ... continues through the full STM32F411 IRQ list ... */
} IRQn_Type;
```

---

## 8. HAL NVIC Configuration

```c
/* HAL_NVIC_SetPriority — sets BOTH preemption and sub-priority explicitly */
HAL_NVIC_SetPriority(EXTI0_IRQn, 5, 0);   /* preempt=5, sub=0 */

HAL_NVIC_EnableIRQ(EXTI0_IRQn);
HAL_NVIC_DisableIRQ(EXTI0_IRQn);

/* HAL also wraps priority grouping (typically set ONCE in HAL_Init(),
 * defaulting to NVIC_PRIORITYGROUP_4 — rarely needs to be touched again) */
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);

/* Note: HAL_Init() ALREADY calls HAL_NVIC_SetPriorityGrouping internally
 * with the default grouping — explicitly calling it again is usually
 * unnecessary unless deliberately changing from the default */
```

### CubeMX NVIC Configuration

In STM32CubeIDE/CubeMX, each peripheral's interrupt is enabled in the "NVIC Settings" tab of that peripheral's configuration pane, with a Preemption Priority and Sub Priority slider directly exposed in the GUI — CubeMX generates the corresponding `HAL_NVIC_SetPriority()`/`HAL_NVIC_EnableIRQ()` calls automatically inside the relevant `MX_<Peripheral>_Init()` function.

---

## 9. EXTI — External Interrupt/Event Controller Overview

EXTI is a dedicated STM32 peripheral that monitors up to **23 lines** for rising-edge, falling-edge, or both-edge transitions, and can generate either an INTERRUPT request (routed to the NVIC) or an EVENT (routed directly to the power-management/wakeup logic, without necessarily involving the CPU/NVIC at all — Section 14).

### EXTI Line Assignment

| EXTI Line | Source                                                            |
|-------------|------------------------------------------------------------------|
| EXTI0–15    | GPIO pins — EXTI line N corresponds to PIN NUMBER N (e.g., EXTI5 monitors whichever GPIO port's pin 5 is currently selected via SYSCFG — see Section 10)|
| EXTI16      | PVD (Programmable Voltage Detector) output                          |
| EXTI17      | RTC Alarm event                                                       |
| EXTI18      | USB OTG FS Wakeup event                                                |
| EXTI19      | (Reserved/device-specific on F411)                                       |
| EXTI21      | RTC Tamper and TimeStamp events                                           |
| EXTI22      | RTC Wakeup event                                                            |

### Critical Concept — EXTI Lines 0-15 Are Shared Across All GPIO Ports

This is THE single most important and most commonly misunderstood aspect of EXTI: **EXTI line N corresponds to PIN NUMBER N across ALL GPIO ports, but only ONE PORT'S pin N can be connected to that EXTI line at any given time.** For example, EXTI Line 5 can be connected to PA5, PB5, PC5, PD5, or PE5 — but NEVER more than one of these simultaneously. Selecting WHICH port's pin N feeds EXTI line N is done via the SYSCFG peripheral (Section 10).

```
EXTI Line 0  ◄── (PA0 OR PB0 OR PC0 OR PD0 OR PE0 OR PH0) — only ONE at a time
EXTI Line 1  ◄── (PA1 OR PB1 OR PC1 OR PD1 OR PE1 OR PH1) — only ONE at a time
...
EXTI Line 15 ◄── (PA15 OR PB15 OR PC15 OR PD15 OR PE15) — only ONE at a time
```

A direct, important consequence: **you cannot use EXTI interrupts on both PA5 AND PB5 simultaneously** — they share EXTI line 5, and only one can be selected as the source at a time. If your application genuinely needs interrupt-driven inputs on two different ports' SAME pin number, you must either choose different pin numbers (e.g., use PA5 and PB6 instead of PA5 and PB5) or use a different mechanism (e.g., polling, or a different peripheral's interrupt) for one of them.

---

## 10. EXTI Line Mapping — SYSCFG and GPIO Ports

```c
/* SYSCFG_EXTICR1-4 registers — each holds the port-selection for 4 EXTI
 * lines (4 bits per line, since up to ~9 possible ports could theoretically
 * be selected, though F411 only populates GPIOA-E and GPIOH) */

/* EXTICR1: EXTI0, EXTI1, EXTI2, EXTI3 */
/* EXTICR2: EXTI4, EXTI5, EXTI6, EXTI7 */
/* EXTICR3: EXTI8, EXTI9, EXTI10, EXTI11 */
/* EXTICR4: EXTI12, EXTI13, EXTI14, EXTI15 */

#define EXTI_PORT_A  0x0
#define EXTI_PORT_B  0x1
#define EXTI_PORT_C  0x2
#define EXTI_PORT_D  0x3
#define EXTI_PORT_E  0x4
#define EXTI_PORT_H  0x7

/* Example: route PC13 (common "user button" pin on many STM32 boards)
 * to EXTI line 13 */
RCC->APB2ENR |= RCC_APB2ENR_SYSCFGEN;   /* SYSCFG clock MUST be enabled first */

SYSCFG->EXTICR[3] &= ~SYSCFG_EXTICR4_EXTI13;   /* Clear EXTI13 port selection bits */
SYSCFG->EXTICR[3] |= (EXTI_PORT_C << SYSCFG_EXTICR4_EXTI13_Pos);  /* Select Port C */
```

> **Critical, easily-forgotten step:** `RCC->APB2ENR`'s `SYSCFGEN` bit MUST be set before writing to ANY `SYSCFG` register, including `EXTICR`. Forgetting this is an extremely common bug — the EXTICR write silently has no effect (same symptom class as forgetting any other peripheral's RCC clock enable, covered in the Bus Architecture document), and the EXTI line ends up still pointing at whatever port it defaulted to (typically Port A) instead of the intended port.

---

## 11. EXTI Registers (Bare-Metal)

| Register   | Purpose                                                              |
|--------------|--------------------------------------------------------------------|
| `EXTI_IMR`  | Interrupt Mask Register — 1 = this line's interrupt request is enabled (unmasked)|
| `EXTI_EMR`  | Event Mask Register — 1 = this line's EVENT (not interrupt) is enabled            |
| `EXTI_RTSR` | Rising Trigger Selection Register — 1 = trigger on rising edge                      |
| `EXTI_FTSR` | Falling Trigger Selection Register — 1 = trigger on falling edge                     |
| `EXTI_SWIER`| Software Interrupt Event Register — writing 1 manually pends that line's interrupt/event (Section 15)|
| `EXTI_PR`   | Pending Register — 1 = this line's interrupt is pending; **write 1 to CLEAR** (not 0 — a common point of confusion, covered in pitfalls)|

```c
/* Configure EXTI13 for falling-edge interrupt (typical "active-low button
 * press" detection — button pulls the pin LOW when pressed) */

EXTI->IMR  |= (1U << 13);    /* Unmask: enable interrupt on this line */
EXTI->RTSR &= ~(1U << 13);   /* Disable rising-edge trigger */
EXTI->FTSR |= (1U << 13);    /* Enable falling-edge trigger */

/* Both-edge trigger (rising AND falling) */
EXTI->RTSR |= (1U << 13);
EXTI->FTSR |= (1U << 13);
```

---

## 12. Bare-Metal EXTI Configuration — GPIO Interrupt

```c
/*
 * Complete example: PC13 button, active-low (pressed = LOW), falling-edge
 * interrupt, with internal pull-up enabled (typical "user button to GND"
 * wiring found on many STM32 development boards)
 */

void PC13_Button_EXTI_Init(void)
{
    /* 1. Enable GPIOC and SYSCFG clocks */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOCEN;
    RCC->APB2ENR |= RCC_APB2ENR_SYSCFGEN;

    /* 2. Configure PC13 as input with pull-up */
    GPIOC->MODER &= ~(3U << (13 * 2));    /* 00 = input mode */
    GPIOC->PUPDR &= ~(3U << (13 * 2));
    GPIOC->PUPDR |=  (1U << (13 * 2));    /* 01 = pull-up */

    /* 3. Route EXTI line 13 to Port C */
    SYSCFG->EXTICR[3] &= ~SYSCFG_EXTICR4_EXTI13;
    SYSCFG->EXTICR[3] |= (EXTI_PORT_C << SYSCFG_EXTICR4_EXTI13_Pos);

    /* 4. Configure EXTI13: falling-edge trigger, unmasked */
    EXTI->FTSR |= (1U << 13);
    EXTI->RTSR &= ~(1U << 13);
    EXTI->IMR  |= (1U << 13);

    /* 5. Configure and enable NVIC for EXTI15_10 (lines 10-15 share one IRQ!) */
    NVIC_SetPriority(EXTI15_10_IRQn, 5);
    NVIC_EnableIRQ(EXTI15_10_IRQn);
}

/* ISR — note the shared IRQ name for lines 10-15 */
void EXTI15_10_IRQHandler(void)
{
    if (EXTI->PR & (1U << 13))   /* Confirm THIS specific line caused it */
    {
        EXTI->PR = (1U << 13);   /* Clear pending flag — WRITE 1 TO CLEAR */

        /* Button press handling logic here */
        ButtonPressed_Flag = 1;
    }
}
```

### EXTI Interrupt-to-NVIC-IRQ Grouping

A subtlety worth calling out explicitly: not every EXTI line has its OWN dedicated NVIC IRQ number — lines 0 through 4 each get a dedicated IRQ (`EXTI0_IRQn` through `EXTI4_IRQn`), but lines 5-9 SHARE one IRQ (`EXTI9_5_IRQn`), and lines 10-15 SHARE another (`EXTI15_10_IRQn`). This means a single ISR (`EXTI9_5_IRQHandler` or `EXTI15_10_IRQHandler`) may need to check MULTIPLE possible source lines via the `EXTI_PR` register to determine which specific line(s) actually triggered it, exactly as shown in the example above checking bit 13 specifically.

```c
/* A more complete EXTI15_10 handler checking multiple possible lines */
void EXTI15_10_IRQHandler(void)
{
    if (EXTI->PR & (1U << 13))
    {
        EXTI->PR = (1U << 13);
        HandleButton();
    }
    if (EXTI->PR & (1U << 14))
    {
        EXTI->PR = (1U << 14);
        HandleOtherSignal();
    }
}
```

---

## 13. HAL EXTI Configuration

```c
/* CubeMX GPIO configuration: set the pin's mode to "GPIO_EXTI13" (or
 * similar) instead of plain Input — CubeMX automatically handles SYSCFG
 * clock enable and EXTICR routing */

/* Generated/typical HAL GPIO init for an EXTI-triggered pin */
GPIO_InitTypeDef GPIO_InitStruct = {0};
GPIO_InitStruct.Pin  = GPIO_PIN_13;
GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;   /* Interrupt, falling edge */
GPIO_InitStruct.Pull = GPIO_PULLUP;
HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

HAL_NVIC_SetPriority(EXTI15_10_IRQn, 5, 0);
HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);

/* GPIO_MODE options covering all EXTI trigger combinations: */
/* GPIO_MODE_IT_RISING   — interrupt, rising edge   */
/* GPIO_MODE_IT_FALLING  — interrupt, falling edge   */
/* GPIO_MODE_IT_RISING_FALLING — interrupt, both edges */
/* GPIO_MODE_EVT_RISING/_FALLING/_RISING_FALLING — EVENT mode equivalents (Section 14) */
```

### HAL's Generic EXTI ISR Pattern

```c
/* The actual vector table entry, common across ALL HAL projects using EXTI */
void EXTI15_10_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
    /* HAL_GPIO_EXTI_IRQHandler internally: checks EXTI_PR for this specific
     * pin's line, clears it (write-1-to-clear), then calls the weak
     * callback below if it was indeed set */
}

/* User implements this weak callback — HAL handles all the EXTI_PR
 * checking/clearing machinery internally, so application code never
 * touches EXTI registers directly when using HAL */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_13)
    {
        ButtonPressed_Flag = 1;
    }
}
```

---

## 14. EXTI Interrupt vs Event Mode

EXTI lines can independently trigger an INTERRUPT (via `EXTI_IMR`, routed to the NVIC, ultimately invoking a C interrupt handler function) and/or an EVENT (via `EXTI_EMR`, routed directly to internal pulse-generation logic that can WAKE the CPU from a low-power mode WITHOUT actually entering an interrupt handler at all).

| Mode           | Routed to       | Wakes CPU from sleep? | Invokes an ISR?  |
|-------------------|------------------|------------------------|---------------------|
| Interrupt (IMR)   | NVIC              | Yes (if NVIC IRQ enabled)| Yes                  |
| Event (EMR)         | Internal wakeup pulse logic only| Yes (for WFE-based sleep)| **No** — no ISR runs at all|

### Why Event Mode Exists

Event mode is specifically useful for waking the CPU from a `WFE` (Wait For Event)-based low-power sleep WITHOUT the overhead of a full interrupt entry/exit (stacking registers, jumping to a handler, returning) — the CPU simply resumes execution at the instruction immediately following the `WFE` instruction, with no ISR invoked, no register stacking, and no NVIC interaction at all. This is a genuinely lower-latency, lower-overhead wakeup mechanism for situations where the application doesn't need to RUN any specific handler code in response to the triggering signal — it just needs the CPU to wake up and continue its main-loop polling from wherever it left off.

```c
/* Configure EXTI line for EVENT (not interrupt) mode */
EXTI->EMR |= (1U << 0);     /* Enable event on line 0 */
EXTI->IMR &= ~(1U << 0);    /* Ensure interrupt is NOT also enabled (or do both, if desired) */
EXTI->RTSR |= (1U << 0);

/* CPU sleep using WFE, woken by the EXTI event without entering any ISR */
__WFE();
/* Execution resumes HERE directly after the triggering edge, no ISR ran */
```

---

## 15. Software-Triggered EXTI

```c
/* EXTI_SWIER lets software manually pend an EXTI line's interrupt/event,
 * exactly as if the real hardware edge had occurred — useful for testing
 * ISR logic without needing to physically trigger the real signal, or for
 * deliberately triggering a software-initiated "interrupt-like" event */

EXTI->SWIER |= (1U << 0);
/* If EXTI_IMR's bit 0 is also set, this immediately pends EXTI0_IRQn in
 * the NVIC exactly as a real rising/falling edge on the routed GPIO pin
 * would have */
```

```c
/* HAL equivalent */
HAL_EXTI_GenerateSWInterrupt(&hexti0);
```

---

## 16. Button Debouncing with EXTI

Mechanical buttons physically "bounce" — producing multiple rapid, spurious edge transitions over a few milliseconds when pressed or released, rather than one clean transition. An EXTI interrupt configured naively will fire MULTIPLE times for what the user perceives as a single button press, unless debouncing is implemented.

### Software Debouncing — Timer-Based

```c
/*
 * Common pattern: EXTI ISR sets a flag / starts a timer; the ACTUAL
 * button-press logic only runs after confirming the signal has been
 * STABLE for a debounce period (typically 10-50 ms for mechanical buttons)
 */

volatile uint8_t debounceInProgress = 0;

void EXTI15_10_IRQHandler(void)
{
    if (EXTI->PR & (1U << 13))
    {
        EXTI->PR = (1U << 13);

        if (!debounceInProgress)
        {
            debounceInProgress = 1;
            /* Disable this EXTI line temporarily to ignore bounce-induced
             * spurious re-triggers during the debounce window */
            EXTI->IMR &= ~(1U << 13);

            /* Start a one-shot timer for ~20ms (TIM config not shown —
             * see the Timers document for one-pulse mode setup) */
            TIM6->CNT = 0;
            TIM6->CR1 |= TIM_CR1_CEN;
        }
    }
}

/* Timer ISR fires once, 20ms after the original edge */
void TIM6_DAC_IRQHandler(void)
{
    if (TIM6->SR & TIM_SR_UIF)
    {
        TIM6->SR &= ~TIM_SR_UIF;
        TIM6->CR1 &= ~TIM_CR1_CEN;

        /* Re-check the pin's CURRENT, settled state directly via GPIO IDR
         * (not via the EXTI flag, which may have already been consumed) */
        if (!(GPIOC->IDR & GPIO_PIN_13))   /* Still LOW = genuinely pressed */
        {
            ButtonPressed_Flag = 1;
        }

        debounceInProgress = 0;
        EXTI->PR  = (1U << 13);     /* Clear any bounce-induced pending flag */
        EXTI->IMR |= (1U << 13);    /* Re-enable the EXTI line */
    }
}
```

### Hardware Debouncing — RC Filter

A simple RC low-pass filter (typically ~1-10 kΩ resistor + 100nF capacitor) on the button signal physically smooths out the bounce's high-frequency transitions before they ever reach the GPIO pin, often eliminating the need for software debouncing logic entirely — a common, low-cost hardware-side mitigation, especially valuable when the EXTI line genuinely needs to respond promptly without the added software complexity/latency of a debounce-timer state machine.

---

## 17. EXTI and Low-Power Mode Wakeup

(Cross-referenced from the Clock document's low-power modes coverage — EXTI is the PRIMARY wakeup mechanism for STOP mode specifically.)

```c
/*
 * EXTI lines remain ACTIVE and can wake the CPU even during STOP mode,
 * because EXTI's edge-detection logic itself does not require the main
 * system clocks (HCLK/PCLK) to be running — it operates off a separate,
 * always-available clock domain specifically so it can serve as a
 * wakeup source even when the rest of the chip is deeply clock-gated.
 */

/* Configure PA0 as a wakeup source from STOP mode */
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
RCC->APB2ENR |= RCC_APB2ENR_SYSCFGEN;

GPIOA->MODER &= ~(3U << 0);   /* Input mode */

SYSCFG->EXTICR[0] &= ~SYSCFG_EXTICR1_EXTI0;
SYSCFG->EXTICR[0] |= (EXTI_PORT_A << SYSCFG_EXTICR1_EXTI0_Pos);

EXTI->RTSR |= (1U << 0);
EXTI->IMR  |= (1U << 0);

NVIC_SetPriority(EXTI0_IRQn, 5);
NVIC_EnableIRQ(EXTI0_IRQn);

/* Enter STOP mode */
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);

/* Execution resumes HERE after EXTI0 wakes the CPU from STOP — but
 * remember (per the Clock document): SYSCLK is now running on HSI at
 * 16 MHz, NOT whatever high-speed PLL configuration was active before
 * STOP — SystemClock_Config() must be called again to restore full speed */
SystemClock_Config();
```

This EXTI-as-wakeup-source pattern is the standard mechanism for "wake on button press" or "wake on external sensor interrupt" designs in battery-powered STM32F411 applications spending most of their time in STOP mode for power savings.

---

## 18. Interrupt Latency and Tail-Chaining

### Basic Interrupt Latency

On the Cortex-M4, interrupt latency (from the triggering event to the first instruction of the ISR executing) is **fixed and deterministic** — typically 12 cycles for the exception entry sequence (automatic register stacking) under ideal conditions (no bus wait states, no higher-priority interrupt already in progress), making Cortex-M interrupt response highly predictable compared to many other architectures.

### Tail-Chaining

When a SECOND interrupt is already PENDING at the moment the FIRST interrupt's handler is about to RETURN, the Cortex-M4 hardware optimizes this "back-to-back" scenario by skipping the full exception-RETURN-then-exception-ENTRY overhead (which would otherwise involve unstacking registers just to immediately re-stack them again) — instead directly transitioning to the second handler with much lower overhead (typically 6 cycles instead of the full ~12+12 cycle entry+exit sequence).

```
Without tail-chaining: ISR1 runs → full exception return (~12 cyc, unstack)
                        → full exception entry (~12 cyc, re-stack) → ISR2 runs

With tail-chaining:     ISR1 runs → tail-chain transition (~6 cyc) → ISR2 runs
```

This is an entirely hardware-automatic optimization — no software configuration is needed to benefit from it; it simply happens whenever the NVIC detects this exact back-to-back pending-interrupt scenario at exception-return time.

---

## 19. Common Pitfalls and Debugging

### 1. Forgetting to enable the SYSCFG clock before writing EXTICR
The single most common EXTI-specific bug — `RCC->APB2ENR |= RCC_APB2ENR_SYSCFGEN` must be set BEFORE any `SYSCFG->EXTICR[]` write, or the write silently has no effect (same general failure class as any other missing-RCC-enable bug, covered in the Bus Architecture document).

### 2. Clearing EXTI_PR by writing 0 instead of 1
`EXTI_PR` is "write 1 to clear" — writing `EXTI->PR &= ~(1U << N)` (which writes a 0 to bit N while preserving other bits) does NOT clear the flag and is actually a no-op for that bit (since writing 0 to an EXTI_PR bit has no effect per the hardware design) — the correct pattern is `EXTI->PR = (1U << N)` (writing exactly a 1 to the specific bit, which clears it, while the 0s elsewhere in the write have no effect on other bits' pending status). Getting this backwards is an extremely common bug, often resulting in the ISR firing exactly once and then NEVER firing again (because the stale pending flag, never properly cleared, keeps the IRQ perpetually re-pending or simply masks subsequent genuine triggers depending on exact symptom manifestation).

### 3. Assuming EXTI lines 0-15 are independent across different GPIO ports
Covered in depth in Section 9 — a very common conceptual misunderstanding for those new to STM32's EXTI architecture, leading to confusing "why can't I get both PA3 and PC3 to interrupt independently" debugging sessions. They share EXTI line 3 and can never both be active simultaneously.

### 4. Not checking the correct line within a shared IRQ handler
Lines 5-9 share `EXTI9_5_IRQHandler`; lines 10-15 share `EXTI15_10_IRQHandler`. Forgetting to check WHICH specific line within that shared handler actually fired (Section 12's example) means a handler written assuming only one specific line is in use will malfunction the moment a second EXTI line sharing that same IRQ is also configured and fires.

### 5. Priority grouping mismatch between an interrupt-heavy bare-metal project and a mixed HAL/bare-metal codebase
If some code explicitly calls `NVIC_SetPriorityGrouping()` with a DIFFERENT grouping value than what `HAL_Init()` already configured, the meaning of subsequently-set priority VALUES can shift unexpectedly (a priority value that meant "preemption level 5" under one grouping can mean something entirely different under another) — generally best to set priority grouping exactly ONCE, early in startup, and never change it again for the remainder of program execution.

### 6. Forgetting that ISR-set flags need `volatile`
A flag variable (like `ButtonPressed_Flag` in this document's examples) written inside an ISR and read inside the main loop MUST be declared `volatile`, or the compiler's optimizer may legally cache the main loop's read in a register, never re-reading the genuinely-updated memory value the ISR wrote — a classic, easy-to-overlook bug that often only manifests at higher optimization levels (`-O2`/`-O3`), passing silently at `-O0` during initial development and testing.

### 7. Re-entrant EXTI configuration race conditions
Reconfiguring an EXTI line's trigger edge or enable state from WITHIN that same line's own ISR (or from a different, concurrently-running ISR) without appropriate care can race against the EXTI hardware's own internal state machine — generally safest to do EXTI reconfiguration from a clearly-sequenced point (main loop, or a lower-priority deferred-processing context), not directly inside the time-critical ISR itself, unless the specific reconfiguration pattern has been carefully verified safe.

---

## 20. Interview Questions and Answers

**Q1: Explain the division of labor between EXTI and NVIC — why does the STM32F411 need both, rather than just one unified interrupt controller?**

**A:** EXTI and NVIC operate at fundamentally different levels of the interrupt-handling stack and serve different purposes. EXTI is a peripheral-level signal CONDITIONING and DETECTION mechanism specific to STM32 (and similar to equivalent peripherals on other vendors' chips) — its job is purely to watch GPIO pins (and a handful of internal signals like RTC alarm, PVD, USB wakeup) for edge or level transitions and translate "a transition happened on this specific pin/line" into a single, generic interrupt-REQUEST signal that it asserts toward the NVIC. NVIC, by contrast, is a standard ARM Cortex-M CORE component (architecturally identical across virtually every Cortex-M3/M4/M7 chip from any vendor) whose job is to manage the much broader, chip-wide concern of: given potentially MANY simultaneously-pending interrupt requests from MANY different sources (not just EXTI — also UART, timers, DMA, ADC, and dozens of other peripherals), which one should the CPU actually respond to right now, in what priority order, with what vectoring (which handler address to jump to). EXTI doesn't know or care about priority relative to OTHER, non-EXTI interrupt sources — it just raises its single request line into the NVIC; NVIC is what actually arbitrates across the ENTIRE chip's interrupt landscape and decides what the CPU does next. Separating these concerns lets ARM define a single, portable, vendor-neutral NVIC architecture that works identically regardless of what specific peripheral-level interrupt-generating mechanisms (EXTI included) a given vendor's chip happens to implement underneath it.

---

**Q2: A developer wants interrupt-driven inputs from a button on PA5 and a separate sensor interrupt on PB5. Why won't this work as written, and what are the two main ways to fix it?**

**A:** This won't work because EXTI line 5 is a SHARED resource across all GPIO ports' pin-5 — PA5, PB5, PC5, PD5, and PE5 all compete for the SAME EXTI line 5, with the SYSCFG_EXTICR registers determining which ONE of these ports' pin 5 is actually connected to that line at any given time. Configuring the SYSCFG routing for "Port A" on EXTI line 5 means PB5's transitions are simply invisible to the EXTI/interrupt system entirely — selecting a NEW port for line 5 doesn't ADD a second source, it REPLACES which single source feeds that line. There are two standard fixes: (1) **Choose different pin NUMBERS for the two signals** — for example, move the sensor interrupt to PB6 instead of PB5; since EXTI line 6 is an entirely separate, independent line from EXTI line 5, PA5 (on line 5) and PB6 (on line 6) can both be independently configured and will both generate genuinely independent interrupts with no resource conflict at all. This is by far the most common, simplest fix when pin assignment flexibility exists. (2) **If the pin number genuinely cannot be changed** (fixed by an existing PCB layout, for example), one of the two signals must use a DIFFERENT interrupt mechanism entirely — for example, if the "sensor" signal is actually a peripheral with its own dedicated interrupt capability (a timer input capture, a UART RX interrupt, etc.) rather than a generic GPIO transition, that peripheral's own interrupt source could be used instead of EXTI for that particular signal, freeing up the shared EXTI line conflict.

---

**Q3: What is the difference between EXTI interrupt mode and EXTI event mode, and when would event mode specifically be preferred?**

**A:** Interrupt mode (configured via `EXTI_IMR`) routes a triggering edge through to the NVIC, which — if that interrupt is enabled and has sufficient priority — causes the CPU to perform a full exception entry (automatically stacking registers, jumping to the corresponding ISR's address from the vector table) and execute whatever C handler function is registered for that interrupt, before eventually returning to wherever normal execution was interrupted. Event mode (configured via `EXTI_EMR`) is architecturally different and lighter-weight: it routes the same triggering edge to internal pulse-generation logic that can wake the CPU from a `WFE` (Wait-For-Event)-based sleep state WITHOUT invoking any ISR at all — no register stacking, no vector table lookup, no C handler function runs; the CPU simply resumes execution at the instruction immediately following the `WFE` instruction that put it to sleep. Event mode is specifically preferred when the application's actual need is simply "wake the CPU up and let it continue its main-loop logic from where it left off" with NO requirement to run any SPECIFIC response code tied to that particular triggering signal — for example, a low-power main-loop-polling application that periodically sleeps via `WFE` and just needs to wake up on ANY of several possible trigger conditions to re-evaluate its main loop state, without needing distinct, signal-specific interrupt handler logic for each possible wakeup cause. Using event mode instead of interrupt mode in this scenario avoids the (admittedly modest, but non-zero) overhead of a full interrupt entry/exit sequence for a wakeup that doesn't actually need any interrupt-specific handling.

---

**Q4: Explain preemption priority versus sub-priority in the NVIC, and why most real applications choose PRIORITYGROUP_4 (all bits to preemption, none to sub-priority).**

**A:** These two priority dimensions answer genuinely different questions. Preemption priority answers: "if interrupt A is currently executing and interrupt B becomes pending, should B be allowed to INTERRUPT A's currently-running handler?" — this is determined PURELY by comparing preemption priorities; a higher preemption priority CAN interrupt a lower one's already-running handler. Sub-priority only matters in the narrower scenario where TWO OR MORE interrupts with the EXACT SAME preemption priority are simultaneously PENDING at the same moment, and the NVIC needs a tie-breaker to decide which one to service FIRST — critically, sub-priority does NOT enable preemption between same-preemption-priority interrupts; a lower-sub-priority interrupt cannot interrupt an already-running same-preemption-priority handler just because of its sub-priority value, it can only win the "which one starts first" race when both are simultaneously waiting to begin. Most real applications choose PRIORITYGROUP_4 (dedicating all 4 available priority bits to preemption priority, 0 bits to sub-priority) because the preemption dimension is almost always the one applications actually care about and need fine-grained control over (ensuring a truly time-critical interrupt, like a motor-control PWM update, can always preempt a less time-critical one, like a UART receive handler) — the sub-priority tie-breaking scenario (two same-priority interrupts pending at the EXACT same instant) is comparatively rare and, when it does occur, the specific tie-breaking order rarely has meaningful real-world consequence for most applications, making the extra granularity not worth sacrificing preemption-priority resolution for.

---

**Q5: A button's EXTI interrupt fires correctly exactly once, then never fires again on subsequent presses, despite the hardware signal genuinely toggling correctly each time. What is the most likely bug, and why does it produce this exact symptom?**

**A:** This is the classic "EXTI_PR cleared incorrectly" bug. EXTI_PR (the Pending Register) uses a write-1-to-clear convention — to clear a pending flag for line N, you must WRITE A 1 to bit N specifically (`EXTI->PR = (1U << N)`), NOT write a 0 to it. A common, easy mistake is instead writing something like `EXTI->PR &= ~(1U << N)` — which actually writes a 0 to bit N (while preserving other bits via the read-modify-write `&=` pattern) — but per the EXTI_PR register's actual hardware behavior, writing a 0 to a pending bit has NO EFFECT on that bit's state at all; only writing a 1 clears it. The result: after the FIRST button press correctly sets the pending flag and fires the ISR, the buggy "clear" code fails to actually clear it (it was a no-op), leaving the pending flag still set. Depending on the exact NVIC/EXTI interaction at this point, this typically manifests as: the IRQ either appears to have been "handled" once (since the ISR DID run once for the genuine first edge) but then the NVIC's pending state for that IRQ, tied to the still-set EXTI_PR bit, prevents the interrupt from being recognized as a NEW, distinct pending event on subsequent genuine button presses — the hardware sees the flag as "already pending/already serviced" rather than recognizing each new edge as a fresh, distinct trigggering event requiring a fresh ISR invocation. The fix is straightforward once correctly diagnosed: change the clearing code to `EXTI->PR = (1U << N)` (a plain assignment writing exactly a 1 to that bit, not a read-modify-write AND-with-complement pattern), which correctly clears the flag and allows subsequent genuine edges to be correctly detected and serviced as new, independent interrupt events.
