# STM32F411 PWM — Complete Reference Guide

---

## Table of Contents

1. [Introduction to PWM](#1-introduction-to-pwm)
2. [STM32F411 PWM Capable Pins and Timers](#2-stm32f411-pwm-capable-pins-and-timers)
3. [PWM Core Concepts](#3-pwm-core-concepts)
4. [PWM Registers (Bare-Metal)](#4-pwm-registers-bare-metal)
5. [Bare-Metal PWM Configuration](#5-bare-metal-pwm-configuration)
6. [HAL PWM Configuration (STM32CubeIDE/CubeMX)](#6-hal-pwm-configuration-stm32cubeidecubemx)
7. [Changing Duty Cycle at Runtime](#7-changing-duty-cycle-at-runtime)
8. [Multiple PWM Channels](#8-multiple-pwm-channels)
9. [Center-Aligned PWM](#9-center-aligned-pwm)
10. [Complementary PWM with Dead-Time (TIM1)](#10-complementary-pwm-with-dead-time-tim1)
11. [PWM for Servo Motor Control](#11-pwm-for-servo-motor-control)
12. [PWM for BLDC/PMSM Motor (3-Phase)](#12-pwm-for-bldc-pmsm-motor-3-phase)
13. [PWM for LED Dimming and RGB Control](#13-pwm-for-led-dimming-and-rgb-control)
14. [Generating Analog Voltage with PWM + RC Filter](#14-generating-analog-voltage-with-pwm--rc-filter)
15. [PWM + DMA for Waveform Generation](#15-pwm--dma-for-waveform-generation)
16. [PWM Frequency and Resolution Trade-offs](#16-pwm-frequency-and-resolution-trade-offs)
17. [Common Pitfalls and Debugging](#17-common-pitfalls-and-debugging)
18. [Interview Questions and Answers](#18-interview-questions-and-answers)

---

## 1. Introduction to PWM

**PWM** (Pulse Width Modulation) is a technique for encoding an analog value as the duty cycle of a digital square wave. Instead of outputting a continuously varying voltage (which requires a DAC), the MCU rapidly switches a digital pin between HIGH and LOW. The **average voltage** seen by the load depends on the fraction of time the pin is HIGH:

```
V_avg = V_HIGH × (t_ON / T_PERIOD) = V_HIGH × Duty_Cycle
```

At high enough switching frequency, most loads (motors, LEDs, filters) respond only to this average — they cannot react to individual pulses.

**Key parameters:**

| Parameter   | Formula                              | Example                     |
|-------------|--------------------------------------|-----------------------------|
| Period (T)  | T = (ARR+1) × (PSC+1) / f_tim       | ARR=999, PSC=83, f=84MHz → 1ms |
| Frequency   | f = f_tim / ((ARR+1) × (PSC+1))     | 1 kHz                       |
| Duty cycle  | D = CCRx / (ARR+1) × 100%           | CCR=250/1000 → 25%          |
| Resolution  | bits = log2(ARR+1)                   | ARR=1023 → 10 bits          |

**Common applications:**

- DC motor speed control
- Servo motor position control (50 Hz, 1–2 ms pulse)
- BLDC / PMSM three-phase inverter drive
- LED brightness / RGB colour mixing
- Switching power supply control (buck, boost, flyback)
- Class-D audio amplifier
- Analog voltage generation (PWM + low-pass filter)
- Heating element control (slow PWM, 10–100 Hz)

---

## 2. STM32F411 PWM Capable Pins and Timers

Every general-purpose and advanced timer channel can produce PWM. Below are the most commonly used alternate-function mappings:

| Timer  | Channel | AF  | Primary Pin | Alternate Pins          |
|--------|---------|-----|-------------|-------------------------|
| TIM1   | CH1     | AF1 | PA8         | PE9                     |
| TIM1   | CH1N    | AF1 | PA7         | PB13, PE8               |
| TIM1   | CH2     | AF1 | PA9         | PE11                    |
| TIM1   | CH2N    | AF1 | PB0         | PB14, PE10              |
| TIM1   | CH3     | AF1 | PA10        | PE13                    |
| TIM1   | CH3N    | AF1 | PB1         | PB15, PE12              |
| TIM1   | CH4     | AF1 | PA11        | PE14                    |
| TIM2   | CH1     | AF1 | PA0         | PA5, PA15               |
| TIM2   | CH2     | AF1 | PA1         | PB3                     |
| TIM2   | CH3     | AF1 | PA2         | PB10                    |
| TIM2   | CH4     | AF1 | PA3         | PB11                    |
| TIM3   | CH1     | AF2 | PA6         | PB4, PC6                |
| TIM3   | CH2     | AF2 | PA7         | PB5, PC7                |
| TIM3   | CH3     | AF2 | PB0         | PC8                     |
| TIM3   | CH4     | AF2 | PB1         | PC9                     |
| TIM4   | CH1     | AF2 | PB6         | PD12                    |
| TIM4   | CH2     | AF2 | PB7         | PD13                    |
| TIM4   | CH3     | AF2 | PB8         | PD14                    |
| TIM4   | CH4     | AF2 | PB9         | PD15                    |
| TIM5   | CH1     | AF2 | PA0         |                         |
| TIM5   | CH2     | AF2 | PA1         |                         |
| TIM5   | CH3     | AF2 | PA2         |                         |
| TIM5   | CH4     | AF2 | PA3         |                         |
| TIM9   | CH1     | AF3 | PA2         | PE5                     |
| TIM9   | CH2     | AF3 | PA3         | PE6                     |
| TIM10  | CH1     | AF3 | PB8         | PF6                     |
| TIM11  | CH1     | AF3 | PB9         | PF7                     |

---

## 3. PWM Core Concepts

### PWM Mode 1 vs Mode 2

| Mode      | OCxM bits | Output when CNT < CCRx | Output when CNT ≥ CCRx |
|-----------|-----------|------------------------|------------------------|
| PWM mode 1| 110       | **HIGH** (active)      | LOW (inactive)         |
| PWM mode 2| 111       | **LOW** (inactive)     | HIGH (active)          |

PWM mode 1 is the standard (duty cycle = CCRx / (ARR+1)).  
PWM mode 2 is inverted — useful when complementary logic is needed without using the complementary channel.

### Duty Cycle Control

```
Duty (%) = CCRx / (ARR + 1) × 100

CCRx = Duty (%) × (ARR + 1) / 100
```

**Boundary conditions:**

| CCRx       | Duty cycle      | Output state               |
|------------|-----------------|----------------------------|
| 0          | 0%              | Always LOW (mode 1)        |
| ARR + 1    | 100%            | Always HIGH (mode 1)       |
| 1          | 1/(ARR+1)×100%  | Very narrow pulse          |
| ARR        | ARR/(ARR+1)×100%| Almost always HIGH         |

For **true 0%** and **true 100%**: use Force LOW (OCxM=100) and Force HIGH (OCxM=101) instead of setting CCRx=0 or CCRx=ARR+1, because the PWM modes can still produce a brief glitch pulse at these boundaries.

### PWM Frequency and Resolution Relationship

With a fixed timer clock f_tim, increasing PWM frequency reduces the maximum ARR (fewer counts per period) → lower resolution:

```
f_PWM × (ARR + 1) = f_tim / (PSC + 1)

Resolution (bits) = log2(ARR + 1) = log2(f_tim / ((PSC+1) × f_PWM))
```

**Example (f_tim = 84 MHz, PSC = 0):**

| f_PWM    | ARR    | Resolution    |
|----------|--------|---------------|
| 1 kHz    | 83999  | ~16.3 bits    |
| 10 kHz   | 8399   | ~13.0 bits    |
| 20 kHz   | 4199   | ~12.0 bits    |
| 100 kHz  | 839    | ~9.7 bits     |
| 1 MHz    | 83     | ~6.4 bits     |

For most motor control: 20 kHz PWM with 12-bit resolution is ideal — above the audible range, good resolution.

---

## 4. PWM Registers (Bare-Metal)

PWM uses the same timer registers as output compare. The critical ones:

### TIMx_CCMR1 / CCMR2 — Output Mode

For channel x in output mode (CCxS=00):

| Field  | Bits       | Description                                      |
|--------|------------|--------------------------------------------------|
| OCxM   | [6:4] or [14:12] | PWM mode 1=110, PWM mode 2=111          |
| OCxPE  | [3] or [11]      | Output compare preload enable (MUST set for PWM) |
| OCxFE  | [2] or [10]      | Output compare fast mode                  |
| CCxS   | [1:0] or [9:8]   | Must be 00 for output                    |

### TIMx_CCER — Output Enable and Polarity

| Field  | Bit  | Description                                  |
|--------|------|----------------------------------------------|
| CCxE   | 4n   | Channel x output enable                      |
| CCxP   | 4n+1 | Polarity: 0=active HIGH, 1=active LOW        |
| CCxNE  | 4n+2 | Complementary channel enable (TIM1 only)     |
| CCxNP  | 4n+3 | Complementary polarity                       |

### TIMx_CCRx — Duty Cycle Value

Write the duty count here. With preload enabled, it takes effect at next UEV.

### TIMx_BDTR — Break and Dead-Time Register (TIM1 only)

| Field  | Bits  | Description                                          |
|--------|-------|------------------------------------------------------|
| MOE    | 15    | Main output enable — MUST be 1 for TIM1 outputs      |
| AOE    | 14    | Automatic output enable (re-enable after break)      |
| BKP    | 13    | Break polarity                                       |
| BKE    | 12    | Break enable                                         |
| OSSR   | 11    | Off-state selection for run mode                     |
| OSSI   | 10    | Off-state selection for idle mode                    |
| LOCK   | 9:8   | Lock level (write-protect critical registers)        |
| DTG    | 7:0   | Dead-time generator value                            |

---

## 5. Bare-Metal PWM Configuration

### Single Channel PWM on TIM3 CH1 (PA6, AF2)

```c
/*
 * PWM on TIM3 CH1 (PA6, AF2)
 * f_timer = 84 MHz (APB1 ×2), PSC=0, ARR=4199 → f_PWM = 20 kHz
 * 12-bit effective resolution: 4200 steps
 * Initial duty: 25%
 */

void TIM3_PWM_Init(void)
{
    /* 1. Enable clocks */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;

    /* 2. Configure PA6 as AF2 (TIM3_CH1), push-pull, high speed */
    GPIOA->MODER   &= ~(3U << 12);
    GPIOA->MODER   |=  (2U << 12);   /* Alternate function */
    GPIOA->OTYPER  &= ~(1U << 6);    /* Push-pull */
    GPIOA->OSPEEDR |=  (3U << 12);   /* High speed */
    GPIOA->PUPDR   &= ~(3U << 12);   /* No pull */
    GPIOA->AFR[0]  &= ~(0xFU << 24);
    GPIOA->AFR[0]  |=  (2U << 24);   /* AF2 = TIM3 */

    /* 3. Timer base */
    TIM3->CR1  = 0;
    TIM3->PSC  = 0;                   /* No prescaler: counter at 84 MHz */
    TIM3->ARR  = 4199;                /* 84 MHz / 4200 = 20 kHz */
    TIM3->CR1 |= TIM_CR1_ARPE;       /* ARR preload */

    /* 4. CCR1 = 25% duty = 4200 × 0.25 = 1050 */
    TIM3->CCR1 = 1050;

    /* 5. CH1: PWM mode 1, preload enabled */
    TIM3->CCMR1 &= ~(TIM_CCMR1_OC1M | TIM_CCMR1_CC1S);
    TIM3->CCMR1 |=  TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1; /* OCM=110: PWM1 */
    TIM3->CCMR1 |=  TIM_CCMR1_OC1PE;                       /* Preload enable */

    /* 6. Enable CH1 output, active HIGH */
    TIM3->CCER |= TIM_CCER_CC1E;
    TIM3->CCER &= ~TIM_CCER_CC1P;    /* Active HIGH */

    /* 7. Force reload of PSC and ARR */
    TIM3->EGR |= TIM_EGR_UG;
    TIM3->SR   = 0;

    /* 8. Start counter */
    TIM3->CR1 |= TIM_CR1_CEN;
}

/* Change duty cycle at runtime (0–100%) */
void TIM3_CH1_SetDuty(float duty_pct)
{
    if (duty_pct < 0.0f)   duty_pct = 0.0f;
    if (duty_pct > 100.0f) duty_pct = 100.0f;
    TIM3->CCR1 = (uint32_t)(duty_pct * (TIM3->ARR + 1) / 100.0f);
}

/* Stop PWM output (force LOW) */
void TIM3_CH1_Stop(void)
{
    TIM3->CCMR1 &= ~TIM_CCMR1_OC1M;
    TIM3->CCMR1 |= TIM_CCMR1_OC1M_2;  /* Force inactive LOW (OCM=100) */
}

/* Resume PWM */
void TIM3_CH1_Start(void)
{
    TIM3->CCMR1 &= ~TIM_CCMR1_OC1M;
    TIM3->CCMR1 |= TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1; /* PWM mode 1 */
}
```

### TIM1 PWM (Advanced Timer — requires MOE)

```c
/*
 * TIM1 CH1 on PA8 (AF1) — 20 kHz PWM
 * TIM1 is on APB2 → timer clock = 84 MHz
 */
void TIM1_PWM_Init(void)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB2ENR |= RCC_APB2ENR_TIM1EN;

    /* PA8 → AF1, push-pull, high speed */
    GPIOA->MODER   &= ~(3U << 16); GPIOA->MODER   |= (2U << 16);
    GPIOA->OTYPER  &= ~(1U << 8);
    GPIOA->OSPEEDR |= (3U << 16);
    GPIOA->AFR[1]  &= ~(0xFU << 0); GPIOA->AFR[1] |= (1U << 0); /* AF1 */

    TIM1->PSC  = 0;
    TIM1->ARR  = 4199;
    TIM1->CCR1 = 1050;   /* 25% */
    TIM1->CR1 |= TIM_CR1_ARPE;

    TIM1->CCMR1 &= ~(TIM_CCMR1_OC1M | TIM_CCMR1_CC1S);
    TIM1->CCMR1 |= TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1PE;

    TIM1->CCER |= TIM_CCER_CC1E;

    /* CRITICAL for TIM1: enable main output */
    TIM1->BDTR |= TIM_BDTR_MOE;

    TIM1->EGR |= TIM_EGR_UG;
    TIM1->SR   = 0;
    TIM1->CR1 |= TIM_CR1_CEN;
}
```

---

## 6. HAL PWM Configuration (STM32CubeIDE/CubeMX)

### CubeMX Settings

In CubeMX, select the timer (e.g., TIM3), set the channel to **PWM Generation CHx**.

**Timer Base:**

| Parameter              | Value                     | Meaning                            |
|------------------------|---------------------------|------------------------------------|
| Prescaler              | 0                         | Counter clock = timer clock        |
| Counter Mode           | Up                        | Edge-aligned                       |
| Counter Period (ARR)   | 4199                      | 20 kHz at 84 MHz                   |
| Auto-Reload Preload    | Enable                    | Shadow ARR                         |

**Channel Settings:**

| Parameter           | Value          | Meaning                          |
|---------------------|----------------|----------------------------------|
| Mode                | PWM Generation | Select PWM mode                  |
| Pulse               | 1050           | Initial CCRx (25% duty)          |
| Fast Mode           | Disable        | Normal operation                 |
| CH Polarity         | High           | Active HIGH                      |
| CH Idle State       | Reset          | Output LOW when MOE=0 (TIM1)     |
| CHN Polarity        | High           | (TIM1 complementary)             |
| CHN Idle State      | Reset          |                                  |

### Generated Code

```c
TIM_HandleTypeDef htim3;
TIM_OC_InitTypeDef sConfigOC = {0};

void MX_TIM3_Init(void)
{
    htim3.Instance               = TIM3;
    htim3.Init.Prescaler         = 0;
    htim3.Init.CounterMode       = TIM_COUNTERMODE_UP;
    htim3.Init.Period            = 4199;
    htim3.Init.ClockDivision     = TIM_CLOCKDIVISION_DIV1;
    htim3.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
    HAL_TIM_PWM_Init(&htim3);

    sConfigOC.OCMode     = TIM_OCMODE_PWM1;
    sConfigOC.Pulse      = 1050;
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
    sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
    HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1);
}

/* Start PWM output */
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

/* Change duty cycle */
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, newCCR);

/* Stop PWM */
HAL_TIM_PWM_Stop(&htim3, TIM_CHANNEL_1);
```

---

## 7. Changing Duty Cycle at Runtime

### Method 1: Direct Register Write (Fastest)

```c
/* Bare-metal — fastest, no HAL overhead */
TIM3->CCR1 = newValue;   /* Takes effect next period (preload enabled) */
```

### Method 2: HAL Macro

```c
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, newValue);
/* Expands to: htim->Instance->CCR1 = newValue */
```

### Method 3: Percentage Helper

```c
void PWM_SetDuty(TIM_HandleTypeDef *htim, uint32_t channel, float duty_pct)
{
    uint32_t arr    = __HAL_TIM_GET_AUTORELOAD(htim);
    uint32_t pulse  = (uint32_t)(duty_pct * (arr + 1) / 100.0f);
    __HAL_TIM_SET_COMPARE(htim, channel, pulse);
}

PWM_SetDuty(&htim3, TIM_CHANNEL_1, 75.0f);   /* 75% duty */
```

### Method 4: Raw Count from ADC

```c
/* Map ADC 12-bit result (0–4095) to PWM duty (0–ARR) */
void PWM_SetFromADC(uint16_t adcVal)
{
    uint32_t arr  = TIM3->ARR;
    TIM3->CCR1    = (adcVal * arr) / 4095;
}
```

### Changing PWM Frequency at Runtime

```c
/* Change frequency without stopping the timer */
void PWM_SetFreq(TIM_HandleTypeDef *htim, uint32_t freq_hz, uint32_t timerClk)
{
    uint32_t newARR = (timerClk / freq_hz) - 1;
    __HAL_TIM_SET_AUTORELOAD(htim, newARR);

    /* Optionally scale CCR to maintain same duty % */
    uint32_t oldARR = __HAL_TIM_GET_AUTORELOAD(htim);
    uint32_t ccr    = __HAL_TIM_GET_COMPARE(htim, TIM_CHANNEL_1);
    uint32_t newCCR = (ccr * (newARR + 1)) / (oldARR + 1);
    __HAL_TIM_SET_COMPARE(htim, TIM_CHANNEL_1, newCCR);
}
```

---

## 8. Multiple PWM Channels

All four channels of a timer share the same PSC and ARR (same frequency) but have independent CCRx (independent duty cycles).

```c
/* TIM3: 4 independent PWM channels at 20 kHz */
void TIM3_4CH_PWM_Init(void)
{
    /* Timer base init (same as single channel) */
    /* ... (clocks, GPIO for PA6, PA7, PB0, PB1 or PC6-9) ... */

    TIM3->PSC = 0;
    TIM3->ARR = 4199;
    TIM3->CR1 |= TIM_CR1_ARPE;

    /* CH1: PWM1, preload */
    TIM3->CCMR1 |= TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1PE;
    /* CH2: PWM1, preload */
    TIM3->CCMR1 |= TIM_CCMR1_OC2M_2 | TIM_CCMR1_OC2M_1 | TIM_CCMR1_OC2PE;
    /* CH3: PWM1, preload */
    TIM3->CCMR2 |= TIM_CCMR2_OC3M_2 | TIM_CCMR2_OC3M_1 | TIM_CCMR2_OC3PE;
    /* CH4: PWM1, preload */
    TIM3->CCMR2 |= TIM_CCMR2_OC4M_2 | TIM_CCMR2_OC4M_1 | TIM_CCMR2_OC4PE;

    /* Enable all 4 channels */
    TIM3->CCER |= TIM_CCER_CC1E | TIM_CCER_CC2E | TIM_CCER_CC3E | TIM_CCER_CC4E;

    /* Set initial duty cycles */
    TIM3->CCR1 = 1050;   /* 25% */
    TIM3->CCR2 = 2100;   /* 50% */
    TIM3->CCR3 = 3150;   /* 75% */
    TIM3->CCR4 = 420;    /* 10% */

    TIM3->EGR |= TIM_EGR_UG;
    TIM3->SR   = 0;
    TIM3->CR1 |= TIM_CR1_CEN;
}
```

---

## 9. Center-Aligned PWM

Center-aligned (symmetric) PWM centres the pulse within each period. The counter counts up to ARR then back down to 0.

```
Edge-aligned:    |‾‾‾|_____|‾‾‾|_____|   (pulse at start of period)
Center-aligned:  |__‾‾‾__|__‾‾‾__|        (pulse centred in period)
```

```c
/* Enable center-aligned mode 1 (CMS=01) */
TIM3->CR1 &= ~TIM_CR1_CMS;
TIM3->CR1 |=  TIM_CR1_CMS_0;   /* CMS=01: center-aligned, UIF on down-count */

/* Effective PWM frequency = f_timer / (2 × ARR) */
/* For 20 kHz: ARR = 84MHz / (2×20000) = 2100 */
TIM3->ARR = 2100;
```

**HAL:**
```c
htim3.Init.CounterMode = TIM_COUNTERMODE_CENTERALIGNED1;
```

**Note:** In center-aligned mode, CCRx values greater than ARR are treated as ARR. The effective duty cycle formula becomes:

```
Duty = CCRx / ARR × 100%   (not ARR+1)
```

---

## 10. Complementary PWM with Dead-Time (TIM1)

Used for H-bridge motor control. CH1 drives the high-side gate; CH1N drives the low-side gate with automatic dead-time.

```c
void TIM1_ComplementaryPWM_Init(void)
{
    RCC->APB2ENR |= RCC_APB2ENR_TIM1EN;
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN | RCC_AHB1ENR_GPIOBEN;

    /* PA8 = TIM1_CH1 (AF1), PB13 = TIM1_CH1N (AF1) */
    /* ... GPIO config ... */

    TIM1->PSC  = 0;
    TIM1->ARR  = 4199;   /* 20 kHz */
    TIM1->CCR1 = 2100;   /* 50% */
    TIM1->CR1 |= TIM_CR1_ARPE;

    /* CH1: PWM mode 1, preload */
    TIM1->CCMR1 |= TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1PE;

    /* Enable CH1 and CH1N outputs */
    TIM1->CCER |= TIM_CCER_CC1E | TIM_CCER_CC1NE;

    /* Dead-time: 50 ticks × (1/84 MHz) = 595 ns
     * BDTR.DTG = 50 (for DTG[7:5]=000: DT = DTG × tDTS = 50 × 11.9 ns ≈ 595 ns) */
    TIM1->BDTR = TIM_BDTR_MOE          /* Main output enable */
               | (50U << 0);           /* DTG = 50 */

    TIM1->EGR |= TIM_EGR_UG;
    TIM1->SR   = 0;
    TIM1->CR1 |= TIM_CR1_CEN;
}
```

**HAL Complementary PWM:**
```c
/* Start both CH1 and CH1N */
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
```

### Dead-Time Calculation

```
tDTS = 1 / f_DTS   where f_DTS = f_timer_clock / (2^CKD)
                    (CKD=00 → f_DTS = f_timer_clock)

For f_timer = 84 MHz:  tDTS = 1/84MHz ≈ 11.9 ns

DTG[7:5] = 000 → DT = DTG[4:0] × tDTS         (range: 0 to 31×tDTS = 369 ns)
DTG[7:5] = 100 → DT = (64+DTG[5:0])×2×tDTS    (range: 128 to 254 × tDTS)
DTG[7:5] = 110 → DT = (32+DTG[4:0])×8×tDTS    (range: 256 to 504 × tDTS)
DTG[7:5] = 111 → DT = (32+DTG[4:0])×16×tDTS   (range: 512 to 1008 × tDTS)
```

---

## 11. PWM for Servo Motor Control

Standard RC servos expect a 50 Hz PWM signal with pulse width between 1 ms (0°) and 2 ms (180°).

```c
/*
 * Servo on TIM3 CH1 (PA6)
 * f_timer = 84 MHz, PSC=83 → f_counter = 1 MHz (1 µs/tick)
 * ARR = 19999 → period = 20000 µs = 50 Hz
 *
 * Pulse width: 1000 µs (0°) to 2000 µs (180°)
 * CCR range: 1000 to 2000
 */
void Servo_Init(void)
{
    /* GPIO and clock init... */
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;

    TIM3->PSC  = 83;     /* 1 MHz counter */
    TIM3->ARR  = 19999;  /* 20 ms period = 50 Hz */
    TIM3->CCR1 = 1500;   /* Start at centre (90°) */
    TIM3->CR1 |= TIM_CR1_ARPE;

    TIM3->CCMR1 |= TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1PE;
    TIM3->CCER  |= TIM_CCER_CC1E;
    TIM3->BDTR  |= TIM_BDTR_MOE;   /* Only needed for TIM1 */

    TIM3->EGR |= TIM_EGR_UG;
    TIM3->SR   = 0;
    TIM3->CR1 |= TIM_CR1_CEN;
}

/* Set servo angle (0 to 180 degrees) */
void Servo_SetAngle(float angle)
{
    if (angle < 0.0f)   angle = 0.0f;
    if (angle > 180.0f) angle = 180.0f;
    /* Map 0–180° to 1000–2000 µs */
    TIM3->CCR1 = (uint32_t)(1000.0f + (angle / 180.0f) * 1000.0f);
}

/* Set pulse width directly in microseconds */
void Servo_SetPulseUs(uint16_t us)
{
    if (us < 500)  us = 500;
    if (us > 2500) us = 2500;
    TIM3->CCR1 = us;
}

/* HAL version */
void Servo_SetAngle_HAL(TIM_HandleTypeDef *htim, uint32_t ch, float angle)
{
    uint32_t pulse = (uint32_t)(1000.0f + (angle / 180.0f) * 1000.0f);
    __HAL_TIM_SET_COMPARE(htim, ch, pulse);
}
```

---

## 12. PWM for BLDC/PMSM Motor (3-Phase)

Three-phase motor control requires 6 PWM signals (3 high-side, 3 low-side) with complementary operation and dead-time.

```c
/*
 * TIM1: 3-phase complementary PWM at 20 kHz, center-aligned
 * CH1/CH1N: Phase A   CH2/CH2N: Phase B   CH3/CH3N: Phase C
 * PA8=CH1, PA9=CH2, PA10=CH3
 * PB13=CH1N, PB14=CH2N, PB15=CH3N
 */
void TIM1_3Phase_PWM_Init(void)
{
    RCC->APB2ENR |= RCC_APB2ENR_TIM1EN;

    /* Center-aligned mode 1 */
    TIM1->CR1 = TIM_CR1_ARPE | TIM_CR1_CMS_0;   /* CMS=01 */

    /* ARR for 20 kHz center-aligned: 84MHz/(2×20kHz) = 2100 */
    TIM1->PSC = 0;
    TIM1->ARR = 2099;

    /* All 3 channels: PWM mode 1, preload */
    TIM1->CCMR1 = TIM_CCMR1_OC1M_2 | TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1PE
                | TIM_CCMR1_OC2M_2 | TIM_CCMR1_OC2M_1 | TIM_CCMR1_OC2PE;
    TIM1->CCMR2 = TIM_CCMR2_OC3M_2 | TIM_CCMR2_OC3M_1 | TIM_CCMR2_OC3PE;

    /* Enable all 6 outputs */
    TIM1->CCER = TIM_CCER_CC1E | TIM_CCER_CC1NE
               | TIM_CCER_CC2E | TIM_CCER_CC2NE
               | TIM_CCER_CC3E | TIM_CCER_CC3NE;

    /* Initial duty: 50% */
    TIM1->CCR1 = TIM1->CCR2 = TIM1->CCR3 = 1050;

    /* Dead-time: ~600 ns, main output enable */
    TIM1->BDTR = TIM_BDTR_MOE | (50U << 0);

    /* Repetition counter: update every 2 overflows (at peak of triangle) */
    TIM1->RCR = 1;

    TIM1->EGR |= TIM_EGR_UG;
    TIM1->SR   = 0;
    TIM1->CR1 |= TIM_CR1_CEN;
}

/* Update all 3 duty cycles atomically (called from ADC interrupt at 20 kHz) */
void Motor_SetDutyCycles(uint16_t dutyA, uint16_t dutyB, uint16_t dutyC)
{
    TIM1->CCR1 = dutyA;
    TIM1->CCR2 = dutyB;
    TIM1->CCR3 = dutyC;
}
```

---

## 13. PWM for LED Dimming and RGB Control

```c
/*
 * RGB LED on TIM3 CH1 (R=PA6), CH2 (G=PA7), CH3 (B=PB0)
 * 1 kHz PWM (above flicker threshold), 8-bit resolution (ARR=255)
 * f_timer=84MHz, PSC=327 → f_counter ≈ 256 kHz → ARR=255 → f_PWM=1 kHz
 */
void RGB_LED_Init(void)
{
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;
    /* GPIO setup for PA6, PA7, PB0 as AF2... */

    TIM3->PSC  = 327;    /* 84MHz/328 = 256.097 kHz */
    TIM3->ARR  = 255;    /* 256 counts = 8-bit */
    TIM3->CR1 |= TIM_CR1_ARPE;

    /* All 3 channels: PWM1, preload */
    TIM3->CCMR1 = TIM_CCMR1_OC1M_2|TIM_CCMR1_OC1M_1|TIM_CCMR1_OC1PE
                | TIM_CCMR1_OC2M_2|TIM_CCMR1_OC2M_1|TIM_CCMR1_OC2PE;
    TIM3->CCMR2 = TIM_CCMR2_OC3M_2|TIM_CCMR2_OC3M_1|TIM_CCMR2_OC3PE;
    TIM3->CCER |= TIM_CCER_CC1E|TIM_CCER_CC2E|TIM_CCER_CC3E;

    TIM3->CCR1 = 0;  TIM3->CCR2 = 0;  TIM3->CCR3 = 0;

    TIM3->EGR |= TIM_EGR_UG; TIM3->SR = 0;
    TIM3->CR1 |= TIM_CR1_CEN;
}

/* Set RGB colour (0–255 per channel) */
void RGB_SetColor(uint8_t r, uint8_t g, uint8_t b)
{
    TIM3->CCR1 = r;   /* Red */
    TIM3->CCR2 = g;   /* Green */
    TIM3->CCR3 = b;   /* Blue */
}

/* Gamma correction (perceived brightness is non-linear) */
static const uint8_t gamma_lut[256] = {
    /* Pre-computed gamma 2.2 table */
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,2,2,2,2,2,2,3,3,
    3,3,3,4,4,4,4,5,5,5,5,6,6,6,7,7,7,8,8,8,9,9,9,10,10,11,11,11,12,12,
    13,13,14,14,15,15,16,16,17,17,18,18,19,20,20,21,21,22,23,23,24,25,25,
    26,27,27,28,29,30,30,31,32,33,34,34,35,36,37,38,39,40,40,41,42,43,44,
    45,46,47,48,49,50,51,52,54,55,56,57,58,59,60,62,63,64,65,67,68,69,70,
    72,73,74,76,77,78,80,81,83,84,85,87,88,90,91,93,94,96,97,99,100,102,
    104,105,107,108,110,112,113,115,117,119,120,122,124,126,127,129,131,
    133,135,137,138,140,142,144,146,148,150,152,154,156,158,160,162,165,
    167,169,171,173,175,178,180,182,184,187,189,191,193,196,198,200,203,
    205,208,210,212,215,217,220,222,225,227,230,232,235,237,240,243,245,
    248,251,253,255
};

void RGB_SetColorGamma(uint8_t r, uint8_t g, uint8_t b)
{
    TIM3->CCR1 = gamma_lut[r];
    TIM3->CCR2 = gamma_lut[g];
    TIM3->CCR3 = gamma_lut[b];
}
```

---

## 14. Generating Analog Voltage with PWM + RC Filter

A first-order RC low-pass filter converts PWM to a DC voltage. The cutoff frequency must be much lower than the PWM frequency:

```
f_cutoff = 1 / (2π × R × C)   <<  f_PWM

V_out = V_DD × Duty_cycle
```

**Example:** PWM at 20 kHz, R=10 kΩ, C=1 µF → f_cutoff = 15.9 Hz (≪ 20 kHz).  
Ripple voltage ≈ V_DD × (f_cutoff / f_PWM) ≈ 3.3 × (15.9/20000) ≈ 2.6 mV — negligible.

```c
/* Generate 1.65 V (50% of 3.3V) on PA6 */
TIM3->CCR1 = (TIM3->ARR + 1) / 2;

/* Ramp voltage from 0 to 3.3V */
for (uint32_t i = 0; i <= TIM3->ARR; i++)
{
    TIM3->CCR1 = i;
    HAL_Delay(1);
}
```

**RC filter design rules:**
- f_cutoff < f_PWM / 100 for < 1% ripple at output
- f_cutoff must be high enough to track desired signal bandwidth
- For audio DAC: f_PWM ≥ 1 MHz, f_cutoff ≈ 20 kHz

---

## 15. PWM + DMA for Waveform Generation

Combine timer update events, DMA, and CCR writes to generate arbitrary waveforms without ISR overhead.

```c
/*
 * Sine wave generation:
 * TIM3 CCR1 is updated by DMA from a sine LUT on every UEV
 * Result: sine wave on PA6 after RC filter
 */

#define SINE_POINTS 256
uint16_t sineLUT[SINE_POINTS];

void GenerateSineLUT(uint16_t amplitude, uint16_t offset)
{
    for (int i = 0; i < SINE_POINTS; i++)
    {
        sineLUT[i] = (uint16_t)(offset + amplitude *
                     sinf(2.0f * M_PI * i / SINE_POINTS));
    }
}

void PWM_DMA_SineInit(void)
{
    /* TIM3 at 25.6 kHz: one LUT cycle / 256 updates
     * sine frequency = 25600/256 = 100 Hz */
    TIM3->PSC = 0;
    TIM3->ARR = 4095;   /* 12-bit resolution */
    TIM3->CR1 |= TIM_CR1_ARPE;
    TIM3->CCMR1 |= TIM_CCMR1_OC1M_2|TIM_CCMR1_OC1M_1|TIM_CCMR1_OC1PE;
    TIM3->CCER  |= TIM_CCER_CC1E;

    /* Enable UDE: DMA request on Update Event */
    TIM3->DIER |= TIM_DIER_UDE;

    /* DMA1 Stream4 Ch5 → TIM3_UP (memory to peripheral, circular) */
    DMA1_Stream4->CR  = 0;
    DMA1_Stream4->PAR = (uint32_t)&TIM3->CCR1;
    DMA1_Stream4->M0AR= (uint32_t)sineLUT;
    DMA1_Stream4->NDTR= SINE_POINTS;
    DMA1_Stream4->CR  = (5U << DMA_SxCR_CHSEL_Pos)  /* Ch5 = TIM3_UP */
                      | DMA_SxCR_MINC                /* Memory increment */
                      | (1U << DMA_SxCR_MSIZE_Pos)  /* 16-bit memory */
                      | (1U << DMA_SxCR_PSIZE_Pos)  /* 16-bit peripheral */
                      | (1U << DMA_SxCR_DIR_Pos)    /* M2P */
                      | DMA_SxCR_CIRC               /* Circular */
                      | DMA_SxCR_EN;

    GenerateSineLUT(2047, 2048);
    TIM3->EGR |= TIM_EGR_UG; TIM3->SR = 0;
    TIM3->CR1 |= TIM_CR1_CEN;
}
/* Result: 100 Hz sine wave on PA6 (after RC filter), CPU usage = 0% */
```

---

## 16. PWM Frequency and Resolution Trade-offs

### Design Table (f_timer = 84 MHz, PSC = 0)

| f_PWM     | ARR   | Resolution (bits) | Application                        |
|-----------|-------|-------------------|------------------------------------|
| 50 Hz     | 1679999| 20.7 bits        | Servo (needs 1µs PSC=83)           |
| 1 kHz     | 83999 | 16.4 bits         | LED dimming, heating                |
| 10 kHz    | 8399  | 13.0 bits         | DC motor (audible range)            |
| 20 kHz    | 4199  | 12.0 bits         | DC motor, BLDC (inaudible)          |
| 40 kHz    | 2099  | 11.0 bits         | Class-D audio                       |
| 100 kHz   | 839   | 9.7 bits          | Buck converter                      |
| 500 kHz   | 167   | 7.4 bits          | High-speed switching                |
| 1 MHz     | 83    | 6.4 bits          | GaN FET converters                  |

### Choosing PWM Frequency

- **Motors:** 10–25 kHz (above 20 kHz is inaudible, reduces EMI filtering)
- **Servos:** 50 Hz (protocol standard)
- **LEDs:** ≥100 Hz (no flicker), typically 1–10 kHz
- **Buck converters:** 100–500 kHz (higher = smaller inductor)
- **Class-D audio:** 300 kHz–1 MHz (must be >> 20 kHz audio bandwidth)
- **Heating elements:** 1–10 Hz (thermal inertia is slow; high resolution not needed)

---

## 17. Common Pitfalls and Debugging

### 1. TIM1 output stuck low — missing MOE
TIM1's outputs are disabled by default (BDTR.MOE=0). Always set MOE in bare-metal or call `HAL_TIM_PWM_Start()` which sets it internally.

### 2. Glitch on duty cycle change — preload not enabled
Without OCxPE=1 (output compare preload), writing a new CCRx takes effect immediately, possibly mid-cycle, creating a spurious narrow or wide pulse. Always enable preload (`TIM3->CCMR1 |= TIM_CCMR1_OC1PE;`).

### 3. 0% duty still shows a narrow pulse
PWM mode 1 with CCRx=0: CNT starts at 0, immediately CNT == CCRx → goes LOW. But there can be a 1-tick glitch at reload depending on hardware. To guarantee 0% output, use Force Inactive mode (OCxM=100) instead.

### 4. PWM frequency changes affect duty cycle
When you change ARR (new period), the old CCRx value represents a different duty cycle. Always recalculate CCRx proportionally: `newCCR = (oldCCR × (newARR+1)) / (oldARR+1)`.

### 5. Complementary output polarity wrong
If the low-side gate is inverted from expected: check CCxNP (complementary polarity bit in CCER). Also verify the gate driver's active level — some require active-LOW input.

### 6. Dead-time too short — MOSFET shoot-through
Calculate worst-case transistor fall time from datasheet (junction temperature, load current). Add margin. Verify with an oscilloscope watching VGS of both transistors — there should be no overlap.

### 7. Servo jitter
Likely caused by ISR latency affecting the timer interrupt that manually updates CCRx. Use hardware PWM (timer CCR) with stable 50 Hz timer — no ISR needed once configured.

---

## 18. Interview Questions and Answers

**Q1: What is PWM duty cycle and how is it calculated from timer registers?**

**A:** The duty cycle is the fraction of the period during which the output is HIGH. In PWM mode 1 (up-counting), the output is HIGH while CNT < CCRx and LOW while CNT ≥ CCRx. The duty cycle = CCRx / (ARR + 1). For example with ARR=999 and CCRx=250: duty = 250/1000 = 25%. To set a desired duty: CCRx = duty_fraction × (ARR + 1). The ARR value determines the period (and therefore the resolution in bits) while CCRx determines the ON time within that period.

---

**Q2: What is the difference between PWM mode 1 and mode 2?**

**A:** Both are edge-aligned PWM but with inverted output. Mode 1 (OCxM=110): output is HIGH when CNT < CCRx and LOW when CNT ≥ CCRx — increasing CCRx increases duty cycle. Mode 2 (OCxM=111): output is LOW when CNT < CCRx and HIGH when CNT ≥ CCRx — increasing CCRx decreases duty cycle. Mode 2 is logically equivalent to mode 1 with the polarity bit inverted. Practical use: when driving a complementary pair without using the hardware complementary channel, use PWM mode 2 for the second switch to avoid adding a software inverter.

---

**Q3: Why is output compare preload (OCxPE) important for PWM?**

**A:** Without preload (OCxPE=0), a write to CCRx takes effect immediately — even mid-cycle when the counter is somewhere between 0 and ARR. If the new CCRx value is less than the current CNT, the compare match already passed and the output will not change until the next period — creating a too-long or too-short pulse in the current cycle. With preload (OCxPE=1), writes to CCRx go into a shadow register that transfers to the active register only on the next Update Event (start of next period). This guarantees the duty cycle changes cleanly on a period boundary — no glitches, no partial pulses.

---

**Q4: How do you generate a 50 Hz servo signal with 1 µs resolution?**

**A:** Configure the timer with PSC=83 (at 84 MHz timer clock → 1 µs per tick) and ARR=19999 (20000 µs = 20 ms = 50 Hz). Set CCRx between 1000 (1 ms = 0°) and 2000 (2 ms = 180°). The 1 µs counter resolution gives positional resolution of 1/1000 of the full range ≈ 0.18°. The hardware timer runs autonomously once set up — no ISR or CPU involvement per pulse. Change angle by writing a new CCRx value which takes effect on the next 20 ms period.

---

**Q5: Why is 20 kHz commonly chosen as the motor PWM frequency?**

**A:** 20 kHz sits just above the upper limit of human hearing (~20 kHz). Motor windings and their current ripple create acoustic noise at the PWM frequency — at frequencies below 20 kHz this produces an audible whine. At 20 kHz the noise is inaudible. At the same time, 20 kHz is low enough that: (1) switching losses in the power transistors are manageable, (2) gate drive power is reasonable, (3) at 84 MHz timer clock, ARR=4199 gives 12-bit effective resolution (4200 steps) — more than adequate for speed or torque control. Higher frequencies (>50 kHz) require GaN or SiC transistors to keep switching losses acceptable.

---

**Q6: What is dead-time and how do you calculate the DTG register value?**

**A:** Dead time is a blanking period inserted between turning OFF one switch and turning ON the complementary switch in an H-bridge or inverter, preventing both switches from being ON simultaneously (shoot-through). TIM1 inserts dead time automatically via the BDTR.DTG field. When DTG[7:5]=000, the dead time DT = DTG × tDTS where tDTS = 1/f_timer_clock. At 84 MHz: tDTS ≈ 11.9 ns. For 600 ns dead time: DTG = 600 ns / 11.9 ns ≈ 50. Choose dead time based on the transistor's worst-case fall time (from datasheet at maximum junction temperature and maximum current) plus some safety margin.

---

**Q7: How do you use DMA to generate an arbitrary waveform with PWM?**

**A:** Prepare a lookup table (LUT) of CCRx values representing one cycle of the desired waveform (e.g., 256 points of a sine wave). Configure DMA in circular mode from the LUT to the timer's CCRx register, triggered by the timer's Update Event (enable UDE in DIER). On every timer overflow the DMA copies the next LUT value to CCRx, creating a new duty cycle for the next period. After RC filtering the output, the filtered voltage follows the waveform shape. The waveform frequency = f_PWM / number_of_LUT_points. CPU usage during waveform output is zero. The technique is used for audio DAC, arbitrary function generation, and sinusoidal BLDC commutation.

---

**Q8: What is the consequence of not enabling center-aligned mode's UDE correctly for motor FOC?**

**A:** In Field-Oriented Control (FOC) of PMSM motors, the ADC must sample phase currents at the exact moment the triangular PWM carrier reaches its peak (counter at ARR in center-aligned mode) — this is when the current ripple is at its minimum and all PWM outputs are in a stable high state (no switching transients). If the ADC trigger (via TRGO from TIM1) is misconfigured — for example using UEV from edge-aligned mode instead of the center-aligned peak — current sampling occurs during a switching transient, reading a corrupted value that corrupts the FOC current controller. The result is increased torque ripple, audible noise, and potentially instability. The Repetition Counter (RCR=1) combined with center-aligned mode ensures TRGO fires once per full triangle cycle at the peak.
