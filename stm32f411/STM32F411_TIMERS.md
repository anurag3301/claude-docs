# STM32F411 Timers — Complete Reference Guide

---

## Table of Contents

1. [Introduction to Timers](#1-introduction-to-timers)
2. [STM32F411 Timer Hardware Overview](#2-stm32f411-timer-hardware-overview)
3. [Timer Core Concepts](#3-timer-core-concepts)
4. [Timer Registers (Bare-Metal)](#4-timer-registers-bare-metal)
5. [Basic Timer — Periodic Interrupt (Bare-Metal)](#5-basic-timer--periodic-interrupt-bare-metal)
6. [HAL Timer Base Configuration](#6-hal-timer-base-configuration)
7. [Counting Modes](#7-counting-modes)
8. [Input Capture Mode](#8-input-capture-mode)
9. [Output Compare Mode](#9-output-compare-mode)
10. [One-Pulse Mode](#10-one-pulse-mode)
11. [Encoder Interface Mode](#11-encoder-interface-mode)
12. [Timer Master-Slave Synchronization](#12-timer-master-slave-synchronization)
13. [Timer-Triggered ADC Sampling](#13-timer-triggered-adc-sampling)
14. [Advanced Control Timer TIM1](#14-advanced-control-timer-tim1)
15. [Microsecond Delay Using a Free-Running Timer](#15-microsecond-delay-using-a-free-running-timer)
16. [Common Pitfalls and Debugging](#16-common-pitfalls-and-debugging)
17. [Interview Questions and Answers](#17-interview-questions-and-answers)

---

## 1. Introduction to Timers

Timers are one of the most versatile and critical peripherals in any microcontroller. At the hardware level a timer is a counter register that increments (or decrements) on every tick of its input clock. The surrounding logic — prescalers, auto-reload registers, capture/compare units, and synchronisation buses — turns that simple counter into the backbone of virtually every time-critical embedded task:

- **Precise delays and timebase** — generate a 1 ms system tick, replace HAL_Delay busy-loops
- **Periodic interrupts** — drive PID control loops, sensor polling, communication state machines
- **PWM output** — control brushed/brushless motors, servos, LED brightness, switching regulators
- **Input capture** — measure the period, frequency, or pulse width of external signals
- **Output compare** — generate precisely timed output transitions without CPU polling
- **Quadrature encoder interface** — track motor shaft position and speed
- **ADC triggering** — sample an analog signal at an exact, jitter-free rate
- **Waveform generation** — produce sine waves via DMA + timer trigger + DAC
- **Dead-time control** — safely drive complementary gate-drive signals in motor inverters

---

## 2. STM32F411 Timer Hardware Overview

| Timer | Type             | Bus  | Resolution | Channels | Complementary | Repetition | Main Use                |
|-------|------------------|------|------------|----------|---------------|------------|-------------------------|
| TIM1  | Advanced         | APB2 | 16-bit     | 4        | CH1N–CH3N     | Yes (RCR)  | Motor control, H-bridge |
| TIM2  | General-purpose  | APB1 | **32-bit** | 4        | No            | No         | Long periods, timestamps|
| TIM3  | General-purpose  | APB1 | 16-bit     | 4        | No            | No         | General use             |
| TIM4  | General-purpose  | APB1 | 16-bit     | 4        | No            | No         | General use             |
| TIM5  | General-purpose  | APB1 | **32-bit** | 4        | No            | No         | Long periods, timestamps|
| TIM9  | General-purpose  | APB2 | 16-bit     | 2        | No            | No         | Simple PWM/IC           |
| TIM10 | General-purpose  | APB2 | 16-bit     | 1        | No            | No         | Single-channel          |
| TIM11 | General-purpose  | APB2 | 16-bit     | 1        | No            | No         | Single-channel          |

> **Note:** STM32F411 does **not** have TIM6, TIM7, TIM8, TIM12–TIM14.

### Timer Clock Frequencies

All timer clocks on STM32F411 running at 96 MHz (typical max with external 8 MHz crystal) with default CubeMX prescalers:

| Bus  | APB Prescaler | APB Freq | Timer Multiplier | Timer Clock |
|------|---------------|----------|------------------|-------------|
| APB1 | /2            | 48 MHz   | ×2               | **96 MHz**  |
| APB2 | /1            | 96 MHz   | ×1               | **96 MHz**  |

When the APB prescaler equals 1, the timer clock equals the APB clock (no doubling). When it is >1 the timer clock is doubled.  
**Always check the clock tree in CubeMX** — the exact value depends on your PLL settings.

For an STM32F411 at exactly **84 MHz** (common with 8 MHz HSE, PLL N=336, P=4):
- APB2 = 84 MHz → TIM1, TIM9-11 clock = 84 MHz
- APB1 = 42 MHz → TIM2-5 clock = 84 MHz (doubled because APB1 prescaler = /2)

---

## 3. Timer Core Concepts

### 3.1 Prescaler (PSC)

The prescaler divides the timer input clock down to the **counter clock**:

```
f_counter = f_timer_clock / (PSC + 1)
```

PSC is a 16-bit register → maximum divide-by = 65536.

| PSC  | f_counter (84 MHz timer clock) | Resolution per tick |
|------|-------------------------------|---------------------|
| 0    | 84 MHz                        | ~11.9 ns            |
| 83   | 1 MHz                         | 1 µs                |
| 839  | 100 kHz                       | 10 µs               |
| 8399 | 10 kHz                        | 100 µs              |
| 83999| 1 kHz                         | 1 ms                |

### 3.2 Auto-Reload Register (ARR)

The counter counts from **0 to ARR** (up-counting). When it reaches ARR it generates an **Update Event (UEV)** and resets to 0. The timer period is:

```
T_period = (ARR + 1) × (PSC + 1) / f_timer_clock

f_update  = f_timer_clock / ((PSC + 1) × (ARR + 1))
```

**Design formula** — choose PSC and ARR for a desired interrupt frequency `f`:

```
(PSC + 1) × (ARR + 1) = f_timer_clock / f

Pick PSC to give a convenient counter resolution, then:
ARR = (f_timer_clock / ((PSC + 1) × f)) - 1
```

**Example:** 1 kHz ISR, 84 MHz timer clock  
Set PSC = 83 → f_counter = 1 MHz → ARR = 1 000 000 / 1000 − 1 = **999**

### 3.3 Update Event (UEV)

Generated when:
- Counter overflows (up) or underflows (down)
- The UG bit in EGR is written by software

The UEV:
1. Sets the **UIF** flag in SR (triggers interrupt if UIE is set)
2. Transfers shadow register values (PSC, ARR, CCRx) into active registers
3. Can be used as a trigger output (TRGO) for other timers or ADC

### 3.4 Shadow Registers / Preload

PSC, ARR, and each CCRx have a "shadow" (buffer) register. With preload enabled (ARPE=1 for ARR; OCxPE=1 for CCRx):
- Writes to these registers go into the shadow buffer
- Values transfer to active registers only on the next UEV
- Prevents partial updates mid-cycle (important for glitch-free PWM)

Without preload: changes take effect immediately — useful during init but dangerous at runtime.

### 3.5 Capture/Compare Channels

Each channel can independently be configured as:

| Function       | Hardware action                                              |
|----------------|-------------------------------------------------------------|
| Output Compare | Pin changes state when CNT == CCRx                          |
| PWM output     | Pin HIGH when CNT < CCRx (mode 1), or inverted (mode 2)    |
| Input Capture  | CNT value is copied to CCRx when edge detected on input pin |
| Encoder        | Two channels used together to decode quadrature signals      |

---

## 4. Timer Registers (Bare-Metal)

### TIMx_CR1

| Bit  | Name   | Description                                                          |
|------|--------|----------------------------------------------------------------------|
| 9:8  | CKD    | Clock division for input filter and dead-time: 00=×1, 01=×2, 10=×4 |
| 7    | ARPE   | Auto-reload preload enable (1 = shadow ARR)                          |
| 6:5  | CMS    | Center-aligned mode: 00=edge-aligned, 01/10/11=center modes 1/2/3   |
| 4    | DIR    | Direction: 0=up-counting, 1=down-counting                            |
| 3    | OPM    | One-pulse mode (stop after one cycle)                                |
| 2    | URS    | Update request source: 0=all UEV sources, 1=overflow/underflow only  |
| 1    | UDIS   | Update disable: 1=UEV inhibited (shadow regs freeze)                 |
| 0    | CEN    | Counter enable                                                        |

### TIMx_CR2

| Bit  | Name | Description                                                   |
|------|------|---------------------------------------------------------------|
| 6:4  | MMS  | Master mode select — controls TRGO trigger output             |
| 3    | CCDS | CC DMA request on UEV (1) or CC event (0)                     |

**MMS values:**

| MMS | TRGO output                              |
|-----|------------------------------------------|
| 000 | Reset (UG bit)                           |
| 001 | Counter Enable (CEN)                     |
| 010 | Update event (UEV) ← most useful for ADC |
| 011 | Compare pulse on CC1IF                   |
| 100 | OC1REF                                   |
| 101 | OC2REF                                   |
| 110 | OC3REF                                   |
| 111 | OC4REF                                   |

### TIMx_SMCR — Slave Mode Control

| Bit  | Name | Description                                                |
|------|------|------------------------------------------------------------|
| 6:4  | TS   | Trigger selection (which ITRx or TIxFPx drives the slave)  |
| 2:0  | SMS  | Slave mode: 000=disabled, 001=encoder1, 010=encoder2, 011=encoder3, 100=reset, 101=gated, 110=trigger, 111=external clock |

### TIMx_DIER — DMA/Interrupt Enable

| Bit  | Name  | Description                                    |
|------|-------|------------------------------------------------|
| 14   | TDE   | Trigger DMA enable                             |
| 9:5  | CCxDE | Capture/Compare x DMA enable (ch 4 down to 1) |
| 0    | UDE   | Update DMA enable                              |
| 6    | TIE   | Trigger interrupt enable                       |
| 4:1  | CCxIE | Capture/Compare x interrupt enable             |
| 0    | UIE   | Update interrupt enable                        |

### TIMx_SR — Status Register (write 0 to clear)

| Bit  | Name   | Description                                  |
|------|--------|----------------------------------------------|
| 12:9 | CCxOF  | Capture overcapture flag (ch 4 down to 1)    |
| 6    | TIF    | Trigger flag                                 |
| 4:1  | CCxIF  | Capture/Compare x flag                       |
| 0    | UIF    | Update interrupt flag                        |

### TIMx_EGR — Event Generation (write-only)

| Bit  | Name  | Description                          |
|------|-------|--------------------------------------|
| 6    | TG    | Generate trigger event               |
| 4:1  | CCxG  | Generate CC x event                  |
| 0    | UG    | Generate update event (force reload) |

### TIMx_CCMR1 / CCMR2 — Capture/Compare Mode Registers

CCMR1 controls CH1 (bits 7:0) and CH2 (bits 15:8).  
CCMR2 controls CH3 (bits 7:0) and CH4 (bits 15:8).

**Output mode (CCxS = 00):**

| Bits 6:4 | OCxM | Description              |
|----------|------|--------------------------|
| 000      | Frozen       | No output effect                |
| 001      | Active       | Set HIGH on match               |
| 010      | Inactive     | Set LOW on match                |
| 011      | Toggle       | Toggle on match                 |
| 100      | Force LOW    | Always LOW                      |
| 101      | Force HIGH   | Always HIGH                     |
| 110      | PWM mode 1   | HIGH while CNT < CCRx (up)     |
| 111      | PWM mode 2   | LOW while CNT < CCRx (up)      |

Bit 3 = OCxPE (preload enable), Bit 2 = OCxFE (fast enable).

**Input mode (CCxS ≠ 00):**

| CCxS | Input mapping              |
|------|----------------------------|
| 01   | ICx maps to TIx (direct)   |
| 10   | ICx maps to TIy (indirect) |
| 11   | ICx maps to TRC            |

Bits 7:4 = ICxF (input filter), Bits 3:2 = ICxPSC (input prescaler).

### TIMx_CCER — Capture/Compare Enable

| Bit  | Name   | Description                                |
|------|--------|--------------------------------------------|
| 4n+3 | CCxNP  | Complementary polarity (active HIGH/LOW)   |
| 4n+2 | CCxNE  | Complementary channel enable (TIM1 only)   |
| 4n+1 | CCxP   | Polarity: 0=active HIGH/rising, 1=active LOW/falling |
| 4n+0 | CCxE   | Channel enable                             |

### TIMx_PSC, TIMx_ARR, TIMx_CNT, TIMx_CCRx

All 16-bit (except TIM2/TIM5 where ARR, CNT, and CCRx are 32-bit).

---

## 5. Basic Timer — Periodic Interrupt (Bare-Metal)

### 1 kHz Periodic ISR (TIM2, 84 MHz timer clock)

```c
#include "stm32f4xx.h"

volatile uint32_t sysTickMs = 0;

void TIM2_Init_1kHz(void)
{
    /* 1. Enable TIM2 peripheral clock on APB1 */
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;

    /* 2. Reset control registers */
    TIM2->CR1  = 0;
    TIM2->SR   = 0;

    /* 3. Prescaler: 84 MHz / (83+1) = 1 MHz → 1 µs per tick */
    TIM2->PSC = 83;

    /* 4. Auto-reload: count 1000 ticks = 1 ms */
    TIM2->ARR = 999;

    /* 5. Enable auto-reload preload (ARPE=1) */
    TIM2->CR1 |= TIM_CR1_ARPE;

    /* 6. Enable Update interrupt */
    TIM2->DIER |= TIM_DIER_UIE;

    /* 7. Force load PSC and ARR into active registers */
    TIM2->EGR |= TIM_EGR_UG;
    TIM2->SR   = 0;   /* Clear UIF that UG just set */

    /* 8. Configure NVIC */
    NVIC_SetPriority(TIM2_IRQn, 2);
    NVIC_EnableIRQ(TIM2_IRQn);

    /* 9. Start counter */
    TIM2->CR1 |= TIM_CR1_CEN;
}

void TIM2_IRQHandler(void)
{
    if (TIM2->SR & TIM_SR_UIF)
    {
        TIM2->SR &= ~TIM_SR_UIF;   /* MUST clear the flag */
        sysTickMs++;
        /* 1 kHz task: run PID, update sensors, etc. */
    }
}

/* Non-blocking delay (uses sysTickMs) */
void Delay_ms(uint32_t ms)
{
    uint32_t start = sysTickMs;
    while ((sysTickMs - start) < ms);
}
```

> **Why not just use HAL_Delay()?**  
> HAL_Delay depends on SysTick which can be preempted or blocked. A dedicated timer ISR gives you a deterministic, independently-prioritised timebase.

---

## 6. HAL Timer Base Configuration

### CubeMX Settings

| Parameter              | Meaning                                                   | Typical Value    |
|------------------------|-----------------------------------------------------------|------------------|
| Prescaler              | PSC value (counter clock = timer clock / (PSC+1))         | 83 (→ 1 MHz)     |
| Counter Mode           | Up / Down / Center-Aligned 1/2/3                          | Up               |
| Counter Period         | ARR value                                                  | 999 (→ 1 kHz)    |
| Internal Clock Division| CKD: ×1 / ×2 / ×4 (for input filter, not counter clock)  | No Division      |
| Auto-reload Preload    | ARPE bit                                                   | Enable           |
| Trigger Output (TRGO)  | What event appears on TRGO (for ADC/other timer trigger)   | Update Event     |
| Repetition Counter     | RCR — TIM1 only; UEV every (RCR+1) overflows              | 0                |

### Generated Init Code

```c
TIM_HandleTypeDef htim2;

void MX_TIM2_Init(void)
{
    TIM_ClockConfigTypeDef  sClockSourceConfig = {0};
    TIM_MasterConfigTypeDef sMasterConfig      = {0};

    htim2.Instance               = TIM2;
    htim2.Init.Prescaler         = 83;               /* PSC   */
    htim2.Init.CounterMode       = TIM_COUNTERMODE_UP;
    htim2.Init.Period            = 999;              /* ARR   */
    htim2.Init.ClockDivision     = TIM_CLOCKDIVISION_DIV1;
    htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
    HAL_TIM_Base_Init(&htim2);

    sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
    HAL_TIM_ConfigClockSource(&htim2, &sClockSourceConfig);

    sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;
    sMasterConfig.MasterSlaveMode     = TIM_MASTERSLAVEMODE_DISABLE;
    HAL_TIM_MasterConfigSynchronization(&htim2, &sMasterConfig);
}

/* Start with interrupt */
HAL_TIM_Base_Start_IT(&htim2);

/* HAL callback (called from HAL_TIM_IRQHandler → TIM2_IRQHandler) */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        sysTickMs++;
    }
}
```

---

## 7. Counting Modes

### Up-Counting (default)
```
CNT:  0 → 1 → 2 → ... → ARR  (UEV, reset)  → 0 → ...
```
Output HIGH when CNT < CCRx (PWM mode 1).

### Down-Counting
```
CNT:  ARR → ARR-1 → ... → 1 → 0  (UEV, reload) → ARR → ...
```
Set `TIM2->CR1 |= TIM_CR1_DIR;` or `CounterMode = TIM_COUNTERMODE_DOWN`.

### Center-Aligned Mode 1 / 2 / 3
```
CNT:  0 → 1 → ... → ARR (UEV up) → ARR-1 → ... → 1 → 0 (UEV down) → ...
```
- Mode 1: UIF set only at down-count UEV
- Mode 2: UIF set only at up-count UEV
- Mode 3: UIF set at both

**Why center-aligned for motor control?**  
The PWM pulse is symmetric around the period midpoint → both current rising and falling edges are centred → easier current sampling at peak/trough, lower harmonic content, natural dead-time symmetry.

Center-aligned **halves the effective PWM frequency** at the same ARR value (one full up-down cycle = 2×ARR ticks).

---

## 8. Input Capture Mode

Input Capture **copies the current CNT value into CCRx** whenever a configured edge is detected on the input pin. This creates a precise timestamp of the external event.

### Applications

| Measurement        | Technique                                                   |
|--------------------|-------------------------------------------------------------|
| Signal period      | Capture two consecutive rising edges → ΔT = T2 − T1        |
| Signal frequency   | f = f_counter / ΔT                                          |
| Pulse width (HIGH) | Capture rising edge → T1; switch to falling → capture T2    |
| Duty cycle         | Pulse width HIGH / period                                   |
| RPM                | Count rising edges per second (tachometer)                  |

### Bare-Metal Input Capture (Pulse Width Measurement)

```c
/*
 * TIM3 CH1 on PA6 (AF2)
 * Measures HIGH pulse width of an external signal
 * Counter clock: 84 MHz / 84 = 1 MHz (1 µs resolution)
 */

volatile uint32_t risingEdgeTime  = 0;
volatile uint32_t fallingEdgeTime = 0;
volatile uint32_t pulseWidthUs    = 0;
volatile uint8_t  capState        = 0;   /* 0=waiting rising, 1=waiting falling */

void TIM3_IC_Init(void)
{
    /* Clock GPIO and TIM3 */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;

    /* PA6 → AF2 (TIM3_CH1), input, pull-up */
    GPIOA->MODER  &= ~(3U << 12);
    GPIOA->MODER  |=  (2U << 12);         /* Alternate function */
    GPIOA->PUPDR  &= ~(3U << 12);
    GPIOA->PUPDR  |=  (1U << 12);         /* Pull-up */
    GPIOA->AFR[0] &= ~(0xFU << 24);
    GPIOA->AFR[0] |=  (2U << 24);         /* AF2 = TIM3 */

    /* Timer base: 1 µs/tick */
    TIM3->PSC = 83;
    TIM3->ARR = 0xFFFF;                    /* Free-running 16-bit */
    TIM3->CR1 |= TIM_CR1_ARPE;

    /* CH1: input capture, IC1 mapped to TI1 (direct), no filter, no prescaler */
    TIM3->CCMR1 &= ~(TIM_CCMR1_CC1S | TIM_CCMR1_IC1F | TIM_CCMR1_IC1PSC);
    TIM3->CCMR1 |=  TIM_CCMR1_CC1S_0;    /* CC1S=01: IC1 on TI1 */

    /* Polarity: rising edge (CC1P=0, CC1NP=0) */
    TIM3->CCER &= ~(TIM_CCER_CC1P | TIM_CCER_CC1NP);

    /* Enable capture channel 1 */
    TIM3->CCER |= TIM_CCER_CC1E;

    /* Enable CC1 interrupt */
    TIM3->DIER |= TIM_DIER_CC1IE;

    TIM3->EGR |= TIM_EGR_UG;
    TIM3->SR   = 0;

    NVIC_SetPriority(TIM3_IRQn, 1);
    NVIC_EnableIRQ(TIM3_IRQn);

    TIM3->CR1 |= TIM_CR1_CEN;
}

void TIM3_IRQHandler(void)
{
    if (TIM3->SR & TIM_SR_CC1IF)
    {
        TIM3->SR &= ~TIM_SR_CC1IF;

        if (capState == 0)                          /* Rising edge */
        {
            risingEdgeTime = TIM3->CCR1;
            TIM3->CCER |= TIM_CCER_CC1P;           /* Switch to falling */
            capState = 1;
        }
        else                                        /* Falling edge */
        {
            fallingEdgeTime = TIM3->CCR1;
            if (fallingEdgeTime >= risingEdgeTime)
                pulseWidthUs = fallingEdgeTime - risingEdgeTime;
            else
                pulseWidthUs = (0xFFFF - risingEdgeTime) + fallingEdgeTime + 1;

            TIM3->CCER &= ~TIM_CCER_CC1P;          /* Switch back to rising */
            capState = 0;
        }
    }

    /* Clear overcapture flag if set */
    if (TIM3->SR & TIM_SR_CC1OF)
        TIM3->SR &= ~TIM_SR_CC1OF;
}
```

### Frequency Measurement (Period between two rising edges)

```c
volatile uint32_t lastCapture  = 0;
volatile float    freqHz       = 0.0f;

void TIM3_IRQHandler(void)
{
    if (TIM3->SR & TIM_SR_CC1IF)
    {
        TIM3->SR &= ~TIM_SR_CC1IF;
        uint32_t now    = TIM3->CCR1;
        uint32_t period = (now >= lastCapture) ?
                          (now - lastCapture) :
                          (0xFFFF - lastCapture + now + 1);
        lastCapture = now;

        if (period > 0)
            freqHz = 1e6f / (float)period;   /* counter is 1 MHz */
    }
}
```

### HAL Input Capture

```c
TIM_IC_InitTypeDef sConfigIC = {0};

sConfigIC.ICPolarity  = TIM_ICPOLARITY_RISING;
sConfigIC.ICSelection = TIM_ICSELECTION_DIRECTTI;
sConfigIC.ICPrescaler = TIM_ICPSC_DIV1;
sConfigIC.ICFilter    = 0;
HAL_TIM_IC_ConfigChannel(&htim3, &sConfigIC, TIM_CHANNEL_1);
HAL_TIM_IC_Start_IT(&htim3, TIM_CHANNEL_1);

void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM3 && htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
    {
        uint32_t val = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
        /* Process val */
    }
}
```

### Input Capture Input Filter (ICxF)

Filters glitches by requiring N consecutive samples at the same level before accepting an edge. Sampling clock = CK_INT / (2^CKD).

| ICxF | Valid edges require N samples | Useful for                     |
|------|-------------------------------|--------------------------------|
| 0000 | 1 (no filter)                 | Clean logic signals            |
| 0001 | 2 at fCK_INT/1                | Minor debounce                 |
| 0111 | 8 at fCK_INT/4                | Moderate noise rejection       |
| 1111 | 16 at fCK_INT/32              | Maximum filtering (slow signal)|

### Input Prescaler (ICxPSC)

Capture every Nth event instead of every one:

| ICxPSC | Capture on     |
|--------|----------------|
| 00     | Every edge     |
| 01     | Every 2nd edge |
| 10     | Every 4th edge |
| 11     | Every 8th edge |

Useful for very fast signals: instead of generating an interrupt per edge, capture every 8th and multiply in software.

---

## 9. Output Compare Mode

Output Compare fires a hardware action on the pin **the instant CNT == CCRx** without any software intervention.

### Available OC Modes

| OCxM | Name          | Pin action when CNT == CCRx                                       |
|------|---------------|-------------------------------------------------------------------|
| 000  | Frozen        | No action (interrupt only)                                        |
| 001  | Active        | Force pin HIGH                                                    |
| 010  | Inactive      | Force pin LOW                                                     |
| 011  | Toggle        | Toggle pin                                                        |
| 100  | Force LOW     | Pin always LOW regardless of CCRx                                 |
| 101  | Force HIGH    | Pin always HIGH regardless of CCRx                                |
| 110  | PWM mode 1    | HIGH when CNT < CCRx; LOW when CNT ≥ CCRx (up-count)            |
| 111  | PWM mode 2    | LOW when CNT < CCRx; HIGH when CNT ≥ CCRx                       |

### Toggle Mode — Square Wave Generation

```c
/*
 * TIM3 CH1 (PA6, AF2) — Toggle mode: square wave at 500 Hz
 * Timer: 84 MHz / (83+1) = 1 MHz counter
 * Toggle every 1000 µs → 1/(2×1000µs) = 500 Hz
 */
void TIM3_OC_Toggle_Init(void)
{
    /* GPIO: PA6 as AF2 push-pull */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    GPIOA->MODER  &= ~(3U << 12); GPIOA->MODER  |= (2U << 12);
    GPIOA->OTYPER &= ~(1U << 6);
    GPIOA->OSPEEDR|= (3U << 12);
    GPIOA->AFR[0] &= ~(0xFU << 24); GPIOA->AFR[0] |= (2U << 24);

    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;

    TIM3->PSC  = 83;        /* 1 MHz counter */
    TIM3->ARR  = 0xFFFF;    /* Free-running */
    TIM3->CCR1 = 1000;      /* Toggle at 1000 ticks after each toggle */

    /* CH1 Output Compare: Toggle mode */
    TIM3->CCMR1 &= ~TIM_CCMR1_OC1M;
    TIM3->CCMR1 |= (TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1M_0); /* 011 = toggle */

    /* Enable CH1 output, active HIGH */
    TIM3->CCER |= TIM_CCER_CC1E;

    /* Enable CC1 interrupt to update CCR1 for next toggle */
    TIM3->DIER |= TIM_DIER_CC1IE;
    NVIC_SetPriority(TIM3_IRQn, 2);
    NVIC_EnableIRQ(TIM3_IRQn);

    TIM3->EGR |= TIM_EGR_UG;
    TIM3->SR   = 0;
    TIM3->CR1 |= TIM_CR1_CEN;
}

void TIM3_IRQHandler(void)
{
    if (TIM3->SR & TIM_SR_CC1IF)
    {
        TIM3->SR &= ~TIM_SR_CC1IF;
        TIM3->CCR1 += 1000;    /* Schedule next toggle 1000 µs later */
    }
}
```

### HAL Output Compare

```c
TIM_OC_InitTypeDef sConfigOC = {0};

sConfigOC.OCMode     = TIM_OCMODE_TOGGLE;
sConfigOC.Pulse      = 1000;             /* CCRx initial value */
sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
HAL_TIM_OC_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1);
HAL_TIM_OC_Start_IT(&htim3, TIM_CHANNEL_1);

void HAL_TIM_OC_DelayElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM3 && htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
    {
        uint32_t next = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1) + 1000;
        __HAL_TIM_SET_COMPARE(htim, TIM_CHANNEL_1, next);
    }
}
```

---

## 10. One-Pulse Mode

One-Pulse Mode (OPM=1): the timer starts counting on a trigger, generates one output pulse, then stops automatically (CEN clears). No need for software to stop the timer.

```c
/*
 * TIM3 One-Pulse Mode
 * Trigger: rising edge on TI1 (PA6, TIM3_CH1)
 * Output: TIM3_CH2 (PA7) — pulse from CCR2 to ARR
 *
 * Delay  = CCR2 / f_counter (time from trigger to rising edge of pulse)
 * Width  = (ARR - CCR2) / f_counter
 */
void TIM3_OnePulse_Init(void)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;

    /* PA6: TIM3_CH1 input (trigger), PA7: TIM3_CH2 output */
    GPIOA->MODER  &= ~((3U<<12)|(3U<<14));
    GPIOA->MODER  |=  ((2U<<12)|(2U<<14));
    GPIOA->AFR[0] &= ~((0xFU<<24)|(0xFU<<28));
    GPIOA->AFR[0] |=  ((2U<<24)|(2U<<28));

    TIM3->PSC  = 83;       /* 1 MHz counter */
    TIM3->ARR  = 1999;     /* Total length: 2000 µs = 2 ms */
    TIM3->CCR2 = 199;      /* Delay: 200 µs before pulse starts */

    /* CH2: PWM mode 1 output (HIGH while CNT > CCR2, up to ARR) */
    TIM3->CCMR1 &= ~(TIM_CCMR1_OC2M | TIM_CCMR1_CC2S);
    TIM3->CCMR1 |=  (TIM_CCMR1_OC2M_2 | TIM_CCMR1_OC2M_1); /* 110 = PWM1 */
    TIM3->CCER  |=  TIM_CCER_CC2E;

    /* Slave mode: trigger on TI1FP1 (rising edge on TI1), trigger mode starts counter */
    TIM3->SMCR  =  (5U << TIM_SMCR_TS_Pos)   /* TS=101: TI1FP1 */
                 | (6U << TIM_SMCR_SMS_Pos);  /* SMS=110: trigger mode */

    /* One-pulse mode */
    TIM3->CR1 |= TIM_CR1_OPM;
    TIM3->CR1 |= TIM_CR1_CEN;   /* Armed — waits for trigger */
}
```

**HAL One-Pulse:**

```c
HAL_TIM_OnePulse_Init(&htim3, TIM_OPMODE_SINGLE);
HAL_TIM_OnePulse_Start(&htim3, TIM_CHANNEL_2);  /* Enable CH2 output */
```

---

## 11. Encoder Interface Mode

Two channels are used together to decode a **quadrature encoder** (also called an incremental encoder). The hardware:
- Counts UP when rotating in one direction
- Counts DOWN when rotating in the other
- Multiplies the resolution by 2 (edges on CH1 only) or 4 (edges on both CH1 and CH2)
- Automatically handles direction detection via phase relationship

### Quadrature Signal Explained

```
Forward (CW):      A: ‾|_|‾|_|‾     B: _|‾|_|‾|_
                   A leads B by 90° — counter counts UP

Reverse (CCW):     B: ‾|_|‾|_|‾     A: _|‾|_|‾|_
                   B leads A by 90° — counter counts DOWN
```

### Encoder Modes

| SMS   | Mode          | Description                                    |
|-------|---------------|------------------------------------------------|
| 001   | Encoder 1     | Count on TI1 edges only; TI2 sets direction    |
| 010   | Encoder 2     | Count on TI2 edges only; TI1 sets direction    |
| 011   | Encoder 3     | Count on both TI1 and TI2 edges (4× resolution)|

### Bare-Metal Encoder Interface

```c
/*
 * TIM3 Encoder mode 3 (4× resolution) on PA6 (CH1) and PA7 (CH2)
 * 1000 PPR encoder → 4000 counts/revolution
 */
void TIM3_Encoder_Init(void)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;

    /* PA6 (CH1), PA7 (CH2) — AF2, input with pull-up */
    GPIOA->MODER  &= ~((3U<<12)|(3U<<14));
    GPIOA->MODER  |=  ((2U<<12)|(2U<<14));   /* Alternate function */
    GPIOA->PUPDR  &= ~((3U<<12)|(3U<<14));
    GPIOA->PUPDR  |=  ((1U<<12)|(1U<<14));   /* Pull-up */
    GPIOA->AFR[0] &= ~((0xFU<<24)|(0xFU<<28));
    GPIOA->AFR[0] |=  ((2U<<24)|(2U<<28));

    /* No prescaler; full ARR for ±32768 range (signed 16-bit) */
    TIM3->PSC = 0;
    TIM3->ARR = 0xFFFF;

    /* CH1 and CH2: input direct (IC1 on TI1, IC2 on TI2) */
    TIM3->CCMR1 = TIM_CCMR1_CC1S_0 | TIM_CCMR1_CC2S_0;  /* CC1S=01, CC2S=01 */

    /* Input filter: 8 samples at fCK_INT/8 — reduces glitch sensitivity */
    TIM3->CCMR1 |= (0x8U << TIM_CCMR1_IC1F_Pos) | (0x8U << TIM_CCMR1_IC2F_Pos);

    /* Both channels active-HIGH (no polarity inversion) */
    TIM3->CCER &= ~(TIM_CCER_CC1P | TIM_CCER_CC1NP | TIM_CCER_CC2P | TIM_CCER_CC2NP);

    /* Enable CH1 and CH2 capture */
    TIM3->CCER |= TIM_CCER_CC1E | TIM_CCER_CC2E;

    /* Slave mode: Encoder mode 3 */
    TIM3->SMCR = TIM_SMCR_SMS_0 | TIM_SMCR_SMS_1;   /* SMS=011 */

    TIM3->EGR |= TIM_EGR_UG;
    TIM3->SR   = 0;
    TIM3->CR1 |= TIM_CR1_CEN;
}

/* Read signed encoder count (center = 0) */
int32_t Encoder_GetCount(void)
{
    return (int16_t)TIM3->CNT;   /* Cast to signed for -32768 to +32767 */
}

/* Reset counter to midpoint */
void Encoder_Reset(void)
{
    TIM3->CNT = 0;
}

/* Direction: read DIR bit after a count change */
uint8_t Encoder_GetDirection(void)
{
    return (TIM3->CR1 & TIM_CR1_DIR) ? 0 : 1;  /* 1=CW, 0=CCW */
}

/* Speed measurement: read count, compute delta per fixed interval */
int32_t lastCount = 0;
int32_t Encoder_GetSpeedCPS(void)   /* Counts Per Second */
{
    int32_t now    = Encoder_GetCount();
    int32_t delta  = now - lastCount;
    lastCount      = now;
    return delta * 1000;  /* multiply by call rate (1000 if called every 1 ms) */
}
```

### HAL Encoder Interface

```c
TIM_Encoder_InitTypeDef sConfig = {0};

sConfig.EncoderMode  = TIM_ENCODERMODE_TI12;     /* Both edges */
sConfig.IC1Polarity  = TIM_ICPOLARITY_RISING;
sConfig.IC1Selection = TIM_ICSELECTION_DIRECTTI;
sConfig.IC1Prescaler = TIM_ICPSC_DIV1;
sConfig.IC1Filter    = 8;                         /* Glitch filter */
sConfig.IC2Polarity  = TIM_ICPOLARITY_RISING;
sConfig.IC2Selection = TIM_ICSELECTION_DIRECTTI;
sConfig.IC2Prescaler = TIM_ICPSC_DIV1;
sConfig.IC2Filter    = 8;

HAL_TIM_Encoder_Init(&htim3, &sConfig);
HAL_TIM_Encoder_Start(&htim3, TIM_CHANNEL_ALL);

/* Reading count */
int32_t pos = (int16_t)__HAL_TIM_GET_COUNTER(&htim3);
```

---

## 12. Timer Master-Slave Synchronization

Multiple timers can be linked in a master-slave chain without CPU involvement:

```
TIM2 (master) --TRGO--> TIM3 (slave resets) --TRGO--> TIM4 (slave starts)
```

### Master Setup (TIM2)

```c
/* TIM2 master: output Update Event as TRGO */
TIM2->CR2 &= ~TIM_CR2_MMS;
TIM2->CR2 |=  TIM_CR2_MMS_1;   /* MMS=010: Update */
```

### Slave Setup (TIM3 — Reset mode, triggered by TIM2)

```c
/* TIM3 slave: reset on ITR1 (TIM2 TRGO) */
TIM3->SMCR = (0x1U << TIM_SMCR_TS_Pos)    /* TS=001: ITR1 = TIM2 TRGO */
           | (0x4U << TIM_SMCR_SMS_Pos);   /* SMS=100: Reset mode */
```

### ITR (Internal Trigger) Connections (STM32F411)

| Slave receives | ITR0 | ITR1 | ITR2 | ITR3 |
|----------------|------|------|------|------|
| TIM2           | TIM1 | TIM8*| TIM3 | TIM4 |
| TIM3           | TIM1 | TIM2 | TIM5 | TIM4 |
| TIM4           | TIM1 | TIM2 | TIM3 | TIM5 |
| TIM5           | TIM2 | TIM3 | TIM4 | TIM8*|
| TIM9           | TIM2 | TIM3 | TIM10| TIM11|

*TIM8 not present on STM32F411; these connections are device-dependent.*

---

## 13. Timer-Triggered ADC Sampling

Using a timer's TRGO output to trigger ADC conversions achieves **jitter-free, precisely-timed** analog sampling — far better than software-triggered ADC inside an ISR.

```c
/* TIM2 at 10 kHz triggers ADC1 */

/* --- TIM2 Master setup --- */
TIM2->PSC = 83;                    /* 1 MHz counter */
TIM2->ARR = 99;                    /* 1 MHz / 100 = 10 kHz */
TIM2->CR2 |= TIM_CR2_MMS_1;       /* TRGO = Update */
TIM2->CR1 |= TIM_CR1_ARPE | TIM_CR1_CEN;

/* --- ADC1: triggered by TIM2 TRGO --- */
ADC1->CR2 &= ~ADC_CR2_EXTEN;
ADC1->CR2 |=  ADC_CR2_EXTEN_0;    /* Rising edge trigger */

ADC1->CR2 &= ~ADC_CR2_EXTSEL;
ADC1->CR2 |=  (0x6U << ADC_CR2_EXTSEL_Pos); /* EXTSEL=0110: TIM2 TRGO */

ADC1->CR2 |= ADC_CR2_ADON;        /* Enable ADC */

/* ADC + DMA now samples at exactly 10 kHz, triggered by TIM2 */
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adcBuffer, BUFFER_SIZE);
```

---

## 14. Advanced Control Timer TIM1

TIM1 is the most feature-rich timer. Extra capabilities over general-purpose timers:

### Complementary Outputs

Each of CH1, CH2, CH3 has a complementary output (CH1N, CH2N, CH3N). These are used to drive the high-side and low-side gate signals of an H-bridge or three-phase inverter.

```
CH1  → High-side switch gate
CH1N → Low-side switch gate (inverted CH1 with dead-time)
```

### Dead-Time Insertion

Dead time prevents shoot-through when both high-side and low-side are ON simultaneously.

```c
TIM_BreakDeadTimeConfigTypeDef sBDT = {0};

sBDT.OffStateRunMode   = TIM_OSSR_ENABLE;
sBDT.OffStateIDLEMode  = TIM_OSSI_ENABLE;
sBDT.LockLevel         = TIM_LOCKLEVEL_OFF;
sBDT.DeadTime          = 50;      /* 50 ticks × (1/84MHz) × CKD = ~595 ns */
sBDT.BreakState        = TIM_BREAK_ENABLE;
sBDT.BreakPolarity     = TIM_BREAKPOLARITY_HIGH;
sBDT.AutomaticOutput   = TIM_AUTOMATICOUTPUT_ENABLE;

HAL_TIMEx_ConfigBreakDeadTime(&htim1, &sBDT);
```

**Dead-time calculation:**

The DTG (Dead-Time Generator) field in BDTR is 8 bits encoding dead time in multiple formats depending on bit 7:6:
- DTG[7:5]=0xx: DT = DTG × tDTS
- DTG[7:5]=10x: DT = (64 + DTG[5:0]) × 2 × tDTS
- DTG[7:5]=110: DT = (32 + DTG[4:0]) × 8 × tDTS
- DTG[7:5]=111: DT = (32 + DTG[4:0]) × 16 × tDTS

Where tDTS = 1/f_timer_clock (if CKD=00).

### Break Input

The BKIN pin acts as an emergency shutdown. When BKIN is asserted, all PWM outputs immediately go to their safe state (defined by OSSI/OSSR and the polarity bits). Useful for overcurrent protection.

```c
/* BKIN on PA6 — active HIGH */
sBDT.BreakState    = TIM_BREAK_ENABLE;
sBDT.BreakPolarity = TIM_BREAKPOLARITY_HIGH;
/* When BKIN=HIGH: all CHx and CHxN go to idle level defined by OSSI/OSSR */
```

### Main Output Enable (MOE)

**TIM1 PWM outputs are disabled by default.** Always call:
```c
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);          /* Sets MOE=1 */
HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);       /* Also enables CH1N */

/* Bare-metal: */
TIM1->BDTR |= TIM_BDTR_MOE;
```

### Repetition Counter (RCR)

Update event (and thus ADC trigger) occurs every **(RCR + 1)** timer overflows. In center-aligned PWM at 20 kHz with RCR=1:
- Timer overflows at 40 kHz (up and down)
- Update event (and ADC trigger) at 40/2 = 20 kHz
- This puts the ADC trigger at the exact peak of the triangular carrier (minimum current ripple)

---

## 15. Microsecond Delay Using a Free-Running Timer

```c
/*
 * TIM5 (32-bit) as a free-running µs counter
 * Timer clock = 84 MHz, PSC = 83 → 1 µs per tick
 * ARR = 0xFFFFFFFF → wraps after ~4295 seconds
 */
void TIM5_us_Init(void)
{
    RCC->APB1ENR |= RCC_APB1ENR_TIM5EN;
    TIM5->PSC  = 83;
    TIM5->ARR  = 0xFFFFFFFF;
    TIM5->EGR  |= TIM_EGR_UG;
    TIM5->SR    = 0;
    TIM5->CR1  |= TIM_CR1_CEN;
}

uint32_t micros(void)   { return TIM5->CNT; }

void delay_us(uint32_t us)
{
    uint32_t t = micros();
    while ((micros() - t) < us);
}

void delay_ms(uint32_t ms)
{
    delay_us(ms * 1000UL);
}

/* Stopwatch */
uint32_t t_start;
void Stopwatch_Start(void)  { t_start = micros(); }
uint32_t Stopwatch_Read(void){ return micros() - t_start; }
```

---

## 16. Common Pitfalls and Debugging

### 1. Forgetting the ×2 timer clock multiplier
On STM32F4, when APB prescaler ≠ 1, the timer clock is **2× the APB clock**. At 84 MHz system with APB1 = /2 = 42 MHz, TIM2-5 clock = 84 MHz. Always verify in CubeMX's clock tree view.

### 2. Not generating the initial update event (TIM_EGR_UG)
PSC and ARR are transferred from their shadow registers to active registers only on a UEV. Without `TIM->EGR |= TIM_EGR_UG`, the counter may run at the old (reset-state = 0) prescaler until the first natural overflow. Always force UG after configuring PSC and ARR, then clear SR.

### 3. Missing the flag clear in the ISR
If `TIM->SR &= ~TIM_SR_UIF` is not executed, the hardware immediately re-enters the ISR — effectively an infinite loop. Clear the flag at the top of the handler before any processing.

### 4. TIM1 PWM outputs stuck low
The Main Output Enable (BDTR.MOE) is **0 at reset**. Without setting it, CH1-4 and CH1N-CH3N produce no output. In HAL it is set by `HAL_TIM_PWM_Start()`. In bare-metal: `TIM1->BDTR |= TIM_BDTR_MOE;`.

### 5. ARR preload off — glitches when updating duty/period
With ARPE=0, writing a new ARR value takes effect immediately, possibly truncating the current count if the new ARR is less than CNT. Always enable ARPE (preload) and change values atomically through the shadow register.

### 6. Encoder count jitter / noise
If encoder lines are long or noisy, the counter may increment and decrement spuriously. Enable input filter (ICxF in CCMR1/2) — start at 0x8 (8 samples at fCK/8). Also ensure encoder ground is connected directly to MCU GND.

### 7. Encoder direction reversal
If encoder count goes the wrong direction for a given physical rotation, swap CH1 and CH2 connections OR invert the polarity of one channel: `TIM3->CCER |= TIM_CCER_CC1P;`.

### 8. One-pulse mode not re-arming
After one-pulse mode fires (CEN clears), you must re-enable CEN (`TIM->CR1 |= TIM_CR1_CEN`) to arm it for the next trigger. You can do this in the UIF ISR.

### 9. Input capture overflow (CCxOF flag)
If a new edge arrives before the previous CCRx value has been read, the old value is overwritten and the overcapture flag (CCxIF stays set and CCxOF is set). You lose one measurement. Handle this by increasing ISR priority, or use input capture DMA to read CCRx automatically.

---

## 17. Interview Questions and Answers

**Q1: Explain how PSC and ARR work together to set interrupt frequency.**

**A:** The timer input clock is first divided by (PSC + 1) to produce the counter clock. The counter increments at this rate from 0 to ARR. When it reaches ARR it resets and fires an Update Event — this is the interrupt. So the interrupt frequency = f_timer_clock / ((PSC + 1) × (ARR + 1)). PSC and ARR are both 16-bit registers giving a combined divide-by range of 1 to 65536 × 65536 ≈ 4.3 billion. The designer typically chooses PSC to produce a convenient counter resolution (e.g., 1 µs per tick with PSC=83 at 84 MHz timer clock), then calculates ARR from the desired period.

---

**Q2: What is the Update Event and why is it important beyond just the interrupt?**

**A:** The Update Event (UEV) is generated at every counter overflow/underflow and also by software writing TIM_EGR_UG. It serves three roles: (1) transfers shadow register values into active registers — PSC, ARR, and CCRx writes are buffered and only become active at the next UEV, preventing glitches during runtime changes; (2) sets the UIF flag and fires the interrupt if UIE is enabled; (3) generates the TRGO signal if MMS=010, which other timers or the ADC can use as a precise trigger. The UEV is therefore the synchronisation hub of the entire timer subsystem.

---

**Q3: What is the difference between Input Capture and Output Compare?**

**A:** Input Capture is a measurement function: on a detected input edge, the hardware copies the current CNT value into CCRx — giving you a timestamp of when the external event occurred. It is used to measure signal frequency, period, or pulse width. Output Compare is a generation function: the hardware compares CNT with CCRx continuously; when they match, it performs an action (toggle pin, set HIGH, set LOW) without software involvement. PWM is a special case of output compare running in modes 6/7 (PWM mode 1/2). Both modes share the CCRx register, but one reads it (input) and the other writes it (output).

---

**Q4: How does center-aligned PWM differ from edge-aligned? When would you choose each?**

**A:** Edge-aligned PWM (up-counting): counter goes from 0 to ARR, output goes HIGH at 0 and LOW when CNT == CCRx — the active edge is aligned to the counter reset. Center-aligned (symmetric) PWM: counter goes up to ARR then back down to 0, output is HIGH during both the up and down phases while CNT < CCRx — the active pulse is centred in the period. At the same ARR value, center-aligned has half the update frequency (period = 2×ARR ticks). Advantages of center-aligned for motor drives: the switching waveform is symmetric, harmonics are concentrated at even multiples of the PWM frequency (easier to filter), dead-time insertion is naturally symmetric, and the optimal ADC sampling moment (counter peak/trough) is unambiguous.

---

**Q5: Explain the encoder interface mode — how does hardware determine direction?**

**A:** A quadrature encoder produces two square waves A and B with a 90° phase difference. The STM32 encoder mode connects A to TI1 and B to TI2. In encoder mode 3 (TIM_ENCODERMODE_TI12, SMS=011), every edge on both TI1 and TI2 is counted. Direction detection is based on phase: the hardware uses the state of TI2 at the moment a TI1 rising edge occurs (and vice versa) to determine whether to increment or decrement CNT. If A leads B (forward rotation): rising edge of A arrives when B is LOW → count up. If B leads A (reverse): rising edge of A arrives when B is HIGH → count down. The DIR bit in CR1 reflects the current direction. The input filter (ICxF) should be set to reject noise — glitches on encoder lines cause spurious counts that accumulate over time.

---

**Q6: What is shadow register preload and why is it critical for PWM?**

**A:** The PSC, ARR, and CCRx registers each have a hardware shadow (buffer) register. When preload is enabled (ARPE=1 for ARR, OCxPE=1 for CCRx), writes go to the shadow register and transfer to the active register only at the next UEV — the start of the next PWM cycle. Without preload: if you write a new CCRx value mid-cycle (say CNT is at 500 and ARR=999, and you write CCRx=300), the compare fires immediately (since 500 > 300) in the same cycle, creating a spurious narrow pulse. With preload, the new value takes effect only at the next period boundary — the current cycle completes normally. This is essential for smooth, glitch-free PWM duty cycle transitions.

---

**Q7: How do you use a timer to trigger ADC at a precise sample rate?**

**A:** Configure the timer to generate a TRGO output on its Update Event (set MMS=010 in TIMx_CR2). Set the timer frequency to the desired ADC sample rate. Configure the ADC external trigger: set EXTEN to rising edge and EXTSEL to the appropriate timer's TRGO code (e.g., TIM2 TRGO = 0110 for ADC1 on STM32F411 — verify in the datasheet). Enable DMA on the ADC so conversion results are transferred to memory without software intervention. Now every timer UEV starts one ADC conversion, and DMA moves the result. The sample rate is determined entirely by the timer — no ISR jitter, no CPU involvement per sample. This is the correct approach for audio, vibration analysis, or any application requiring a precise sample rate.

---

**Q8: What is dead-time insertion and why is it necessary in motor control?**

**A:** In a half-bridge or full-bridge motor driver, the high-side (HS) and low-side (LS) power switches must never both be ON simultaneously — that would create a short-circuit (shoot-through) from the supply rail directly to ground. Real transistors have finite turn-OFF time (typically 100 ns–several µs for power MOSFETs). If the complementary switch turns ON before the previous one fully turns OFF, shoot-through current destroys the transistors instantly. Dead time is a gap inserted by hardware between turning OFF one switch and turning ON the other. TIM1's dead-time generator automatically delays the edge on CHxN relative to CHx (and vice versa) by a programmed duration (BDTR.DTG field). The dead time must be long enough to cover the worst-case transistor turn-OFF time including temperature derating.

---

**Q9: How do you measure an input signal frequency greater than the maximum counter frequency?**

**A:** Several approaches: (1) **Input prescaler** — use ICxPSC to count every Nth edge instead of every one. Capture two events separated by N cycles and multiply the measured period by N to get the actual period. (2) **Gate counting** — use a known gate time (e.g., 1 second) and count how many rising edges occur; the count equals the frequency. Gate-count mode: put timer A in input capture triggered mode, use timer B to generate the gate window. (3) **Reduce counter resolution** — increase PSC to reduce counter clock frequency. (4) **Divide the input** — use an external prescaler (e.g., 74HC590 counter IC) to divide the input frequency down to a measurable range.

---

**Q10: What is one-pulse mode and how does it work?**

**A:** One-pulse mode (OPM=1 in CR1) causes the timer to automatically stop (clear CEN) after completing one full count cycle. With an external trigger configured via slave mode (e.g., a rising edge on TI1 starts the counter), the timer: counts from 0, fires the output compare action at CCRx (e.g., start of pulse), continues to ARR (end of pulse), fires the UEV, and stops. No CPU intervention is needed to stop the timer — it self-disables. To generate another pulse, software re-enables CEN. Use cases include generating a precisely-timed single pulse (e.g., ultrasonic transducer drive, camera shutter trigger, LIDAR pulse), where you want exactly one burst and then the output to return to idle.

---

**Q11: How do you use the master-slave timer synchronisation for multi-phase PWM?**

**A:** Set TIM1 as master with TRGO = Update, then configure TIM2, TIM3, TIM4 as slaves with SMS=Reset mode triggered by TIM1's TRGO. Each slave timer resets its counter every time TIM1 overflows. All timers share the same counter period but each has its own CCRx for independent duty cycle. By adjusting the CCRx values of the slave timers relative to TIM1's CCRx, you can create phase-shifted PWM signals. This is used in interleaved DC-DC converters (reduce input current ripple), multi-phase motor control, and multi-channel waveform generation — all synchronised in hardware without ISR overhead.
