# STM32F411 ADC — Complete Reference Guide

---

## Table of Contents

1. [Introduction to ADC](#1-introduction-to-adc)
2. [STM32F411 ADC Hardware Overview](#2-stm32f411-adc-hardware-overview)
3. [ADC Core Concepts](#3-adc-core-concepts)
4. [ADC Input Channels and Pin Mapping](#4-adc-input-channels-and-pin-mapping)
5. [ADC Registers (Bare-Metal)](#5-adc-registers-bare-metal)
6. [Bare-Metal ADC Configuration](#6-bare-metal-adc-configuration)
7. [HAL ADC Configuration (STM32CubeIDE/CubeMX)](#7-hal-adc-configuration-stm32cubeidecubemx)
8. [ADC Polling Mode](#8-adc-polling-mode)
9. [ADC Interrupt Mode](#9-adc-interrupt-mode)
10. [ADC DMA Mode — Single Channel](#10-adc-dma-mode--single-channel)
11. [ADC DMA Mode — Multi-Channel Scan](#11-adc-dma-mode--multi-channel-scan)
12. [ADC Continuous + DMA Circular Mode](#12-adc-continuous--dma-circular-mode)
13. [Timer-Triggered ADC Sampling](#13-timer-triggered-adc-sampling)
14. [ADC Injected Channels](#14-adc-injected-channels)
15. [Multi-ADC Modes (Dual/Triple)](#15-multi-adc-modes-dualtriple)
16. [Internal Channels — Temperature Sensor, VREFINT, VBAT](#16-internal-channels--temperature-sensor-vrefint-vbat)
17. [ADC Calibration and Accuracy](#17-adc-calibration-and-accuracy)
18. [Noise Reduction Techniques](#18-noise-reduction-techniques)
19. [Converting Raw ADC Values to Real-World Units](#19-converting-raw-adc-values-to-real-world-units)
20. [Common Pitfalls and Debugging](#20-common-pitfalls-and-debugging)
21. [Interview Questions and Answers](#21-interview-questions-and-answers)

---

## 1. Introduction to ADC

An **ADC** (Analog-to-Digital Converter) converts a continuous analog voltage into a discrete digital number. This is the bridge between the physical world (temperatures, pressures, currents, voltages, audio signals) and the digital processor.

**Key performance parameters:**

| Parameter           | Description                                             | STM32F411 ADC1 |
|---------------------|---------------------------------------------------------|----------------|
| Resolution          | Number of output bits (determines step size)            | 12-bit (max)   |
| Sample rate         | Maximum conversions per second                          | 2.4 MSPS       |
| Input voltage range | Acceptable input voltage at analog pins                 | 0 V to VDDA    |
| Reference voltage   | Voltage that maps to full-scale output                  | VDDA (typ 3.3V)|
| LSB size            | Voltage represented by 1 count                         | VDDA/4096      |
| INL                 | Integral Non-Linearity                                  | ±2 LSB typ     |
| DNL                 | Differential Non-Linearity                              | ±1 LSB typ     |

**Resolution formula:**

```
V_in = ADC_value × VDDA / (2^N - 1)

For 12-bit (N=12): LSB = 3.3 V / 4095 ≈ 0.806 mV
For 10-bit (N=10): LSB = 3.3 V / 1023 ≈ 3.226 mV
For  8-bit (N=8):  LSB = 3.3 V / 255  ≈ 12.94 mV
```

---

## 2. STM32F411 ADC Hardware Overview

The STM32F411 has **one ADC** (ADC1) with the following specifications:

| Feature                     | Value                                    |
|-----------------------------|------------------------------------------|
| ADC controller              | ADC1 only (no ADC2 or ADC3)             |
| Resolution                  | 6, 8, 10, or 12-bit (configurable)       |
| Channels                    | 16 external + 3 internal                |
| Max conversion speed        | 2.4 MSPS at 12-bit                       |
| Min sampling time           | 3 ADC clock cycles                       |
| ADC clock source            | APB2 clock / prescaler (2, 4, 6, or 8)  |
| Max ADC clock frequency     | 36 MHz                                   |
| Conversion modes            | Single, continuous, scan, discontinuous  |
| Trigger sources             | Software, timer TRGO, external pin       |
| Regular group channels      | Up to 16 in sequence                     |
| Injected group channels     | Up to 4 (high priority, can interrupt regular) |
| DMA support                 | Yes (regular group only)                 |
| Analog watchdog             | Yes (up to 3 watchdogs in STM32F4)      |
| Internal channels           | Temperature sensor (CH18), VREFINT (CH17), VBAT (CH18/2) |

### ADC Clock

```
f_ADC = f_APB2 / ADC_prescaler

f_APB2 = 84 MHz (on STM32F411 at max speed)

Prescaler options: /2 → 42 MHz (exceeds 36 MHz limit!)
                   /4 → 21 MHz ✓
                   /6 → 14 MHz ✓
                   /8 → 10.5 MHz ✓

Recommended for 2.4 MSPS: /4 → f_ADC = 21 MHz
```

> **Warning:** The ADC clock must not exceed 36 MHz. Using /2 prescaler at 84 MHz APB2 = 42 MHz — this violates the spec and causes erratic readings.

### Total Conversion Time

```
T_conversion = (sampling_time + conversion_cycles) / f_ADC

Conversion cycles:
  12-bit: 12 cycles
  10-bit: 10 cycles
   8-bit:  8 cycles
   6-bit:  6 cycles

Minimum total (12-bit, 3-cycle sampling, 21 MHz clock):
  T = (3 + 12) / 21 MHz = 15 / 21 MHz ≈ 714 ns → 1.4 MSPS

Maximum total (12-bit, 480-cycle sampling, 21 MHz clock):
  T = (480 + 12) / 21 MHz ≈ 23.4 µs → 42.7 kSPS
```

---

## 3. ADC Core Concepts

### 3.1 Successive Approximation Register (SAR) ADC

The STM32F411 ADC uses a SAR architecture:
1. The sample-and-hold circuit captures the input voltage
2. The DAC starts at mid-scale and compares with the held voltage
3. Binary search narrows down the result over N clock cycles
4. After N cycles, the N-bit digital result is ready

### 3.2 Sampling Time

The input voltage must charge the internal S/H capacitor sufficiently before conversion begins. Sampling time must be long enough for the RC circuit to settle:

```
t_sample ≥ (R_AIN + R_S) × C_S × ln(2^(N+1))

Where:
  R_AIN = source resistance of the analog signal
  R_S   ≈ 6 kΩ (internal ADC input resistance, STM32F411)
  C_S   ≈ 4–8 pF (internal sample capacitor)
  N     = resolution in bits

For 12-bit, R_AIN = 10 kΩ, C_S = 6 pF:
  t_sample ≥ (10k + 6k) × 6pF × ln(8192) ≈ 16k × 6pF × 9.01 ≈ 864 ns
  At 21 MHz: 864 ns × 21 MHz ≈ 18 → use 28-cycle sampling time (next available: 28 cycles ≈ 1.33 µs)
```

**Available sampling times (in ADC clock cycles):**

| Value | Cycles | Duration at 21 MHz |
|-------|--------|---------------------|
| 000   | 3      | 143 ns              |
| 001   | 15     | 714 ns              |
| 010   | 28     | 1.33 µs             |
| 011   | 56     | 2.67 µs             |
| 100   | 84     | 4.00 µs             |
| 101   | 112    | 5.33 µs             |
| 110   | 144    | 6.86 µs             |
| 111   | 480    | 22.86 µs            |

**Rule of thumb:** For signals with source impedance < 1 kΩ, use 3–15 cycles. For 10 kΩ source, use 28–56 cycles. For the temperature sensor or VREFINT, use ≥ 480 cycles (17.1 µs minimum per datasheet).

### 3.3 Regular vs Injected Channels

**Regular group:** Up to 16 channels converted in sequence. DMA is supported. Results go into ADC_DR (data register). The standard mode for most applications.

**Injected group:** Up to 4 channels that can **interrupt** an ongoing regular conversion (like a high-priority interrupt). Results go into ADC_JDRx (4 separate registers — no DMA needed). Used in motor control to sample current at a precise moment (timer event) without disturbing the regular channel sequence.

### 3.4 Conversion Modes

| Mode                  | Description                                                              |
|-----------------------|--------------------------------------------------------------------------|
| Single conversion     | Convert once, then stop. CONT=0, trigger needed for each conversion.     |
| Continuous conversion | Convert repeatedly without pause. CONT=1.                                |
| Scan mode             | Convert all channels in the regular sequence sequentially.               |
| Discontinuous mode    | Convert N channels of the sequence per trigger, then wait for next trigger.|

---

## 4. ADC Input Channels and Pin Mapping

| Channel | GPIO Pin | Notes                                      |
|---------|----------|--------------------------------------------|
| ADC1_IN0  | PA0    | Also TIM2_CH1/TIM5_CH1 (share with care)  |
| ADC1_IN1  | PA1    |                                            |
| ADC1_IN2  | PA2    |                                            |
| ADC1_IN3  | PA3    |                                            |
| ADC1_IN4  | PA4    |                                            |
| ADC1_IN5  | PA5    |                                            |
| ADC1_IN6  | PA6    |                                            |
| ADC1_IN7  | PA7    |                                            |
| ADC1_IN8  | PB0    |                                            |
| ADC1_IN9  | PB1    |                                            |
| ADC1_IN10 | PC0    |                                            |
| ADC1_IN11 | PC1    |                                            |
| ADC1_IN12 | PC2    |                                            |
| ADC1_IN13 | PC3    |                                            |
| ADC1_IN14 | PC4    |                                            |
| ADC1_IN15 | PC5    |                                            |
| ADC1_IN16 | —      | Reserved (connected to VBAT/4 on some STM32F4)|
| ADC1_IN17 | —      | Internal: VREFINT (1.21 V typical)         |
| ADC1_IN18 | —      | Internal: Temperature sensor / VBAT        |

**GPIO configuration for analog input:**

```c
/* Set GPIO to Analog mode — no pull-up/down, no alternate function */
GPIOA->MODER |= (3U << (0*2));   /* PA0: Analog mode (11b) */
GPIOA->PUPDR &= ~(3U << (0*2));  /* No pull resistor */
/* No OTYPER or OSPEEDR settings needed for analog */
```

In CubeMX: set the pin to "ADC1_INx" mode — it automatically configures the GPIO as analog.

---

## 5. ADC Registers (Bare-Metal)

### ADC_SR — Status Register

| Bit | Name   | Description                                            |
|-----|--------|--------------------------------------------------------|
| 4   | STRT   | Regular channel conversion started                     |
| 3   | JSTRT  | Injected channel conversion started                    |
| 2   | JEOC   | Injected channel end of conversion                     |
| 1   | EOC    | End of conversion (regular channel data ready)         |
| 0   | AWD    | Analog watchdog flag                                   |

### ADC_CR1 — Control Register 1

| Bits  | Name    | Description                                              |
|-------|---------|----------------------------------------------------------|
| 25:24 | RES     | Resolution: 00=12-bit, 01=10-bit, 10=8-bit, 11=6-bit    |
| 23    | AWDEN   | Analog watchdog enable on regular channels               |
| 22    | JAWDEN  | Analog watchdog enable on injected channels              |
| 19:16 | DISCNUM | Discontinuous mode channel count                         |
| 12    | JDISCEN | Discontinuous mode on injected channels                  |
| 11    | DISCEN  | Discontinuous mode on regular channels                   |
| 10    | JAUTO   | Auto injected group conversion after regular             |
| 9     | AWDSGL  | Enable watchdog on a single channel                      |
| 8     | SCAN    | Scan mode enable                                         |
| 7     | JEOCIE  | Interrupt enable on JEOC                                 |
| 6     | AWDIE   | Analog watchdog interrupt enable                         |
| 5     | EOCIE   | Interrupt enable on EOC                                  |
| 4:0   | AWDCH   | Analog watchdog channel selection                        |

### ADC_CR2 — Control Register 2

| Bit   | Name     | Description                                              |
|-------|----------|----------------------------------------------------------|
| 30    | SWSTART  | Start conversion of regular channels (software trigger)  |
| 29:28 | EXTEN    | External trigger enable: 00=disabled, 01=rising, 10=falling, 11=both |
| 27:24 | EXTSEL   | External trigger source selection (timer TRGO, EXTI, etc.)|
| 22    | JSWSTART | Start injection group conversion                         |
| 21:20 | JEXTEN   | Injected external trigger enable                         |
| 19:16 | JEXTSEL  | Injected external trigger source                         |
| 11    | ALIGN    | Data alignment: 0=right-aligned, 1=left-aligned          |
| 10    | EOCS     | End of conversion selection: 0=after sequence, 1=after each |
| 9     | DDS      | DMA disable selection: 0=DMA disabled after last, 1=DMA continuous |
| 8     | DMA      | DMA enable                                               |
| 1     | CONT     | Continuous conversion mode                               |
| 0     | ADON     | ADC ON/OFF                                               |

### ADC_SMPR1 / ADC_SMPR2 — Sampling Time Registers

SMPR2 covers channels 0–9 (3 bits each).  
SMPR1 covers channels 10–18 (3 bits each).

### ADC_SQR1 / SQR2 / SQR3 — Regular Sequence Registers

Define which channels are converted and in what order (the "sequence"):

- SQR1[23:20] = L (number of conversions - 1, 0–15)
- SQR1[19:15] = SQ16 (16th conversion)
- SQR1[14:10] = SQ15 ... continuing down
- SQR3[4:0]   = SQ1 (1st conversion)

### ADC_DR — Data Register

Read-only. Contains the most recent conversion result (right-aligned by default).  
Reading DR in dual/triple mode reads both master and slave results.

### ADC_CCR — Common Control Register (shared by all ADCs)

| Bits  | Name    | Description                                          |
|-------|---------|------------------------------------------------------|
| 23    | VBATE   | VBAT channel enable                                  |
| 22    | TSVREFE | Temperature sensor and VREFINT enable                |
| 21:16 | ADCPRE  | ADC prescaler: 00=/2, 01=/4, 10=/6, 11=/8            |
| 15:14 | DMA     | DMA mode for multi-ADC                               |
| 13    | DDS     | DMA disable selection for multi-ADC                  |
| 11:8  | DELAY   | Delay between dual/triple samples                    |
| 4:0   | MULTI   | Multi-ADC mode selection                             |

---

## 6. Bare-Metal ADC Configuration

### Single-Channel Polling (Simplest)

```c
/*
 * ADC1 — single conversion, polling, channel 0 (PA0)
 * f_ADC = 84MHz/4 = 21 MHz, 12-bit, right-aligned
 */

void ADC1_Init_SinglePolling(void)
{
    /* 1. Enable clocks */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;

    /* 2. Configure PA0 as analog input */
    GPIOA->MODER |= (3U << (0*2));    /* Analog mode */
    GPIOA->PUPDR &= ~(3U << (0*2));   /* No pull */

    /* 3. ADC clock prescaler: /4 → 21 MHz */
    ADC->CCR &= ~ADC_CCR_ADCPRE;
    ADC->CCR |= ADC_CCR_ADCPRE_0;    /* 01 = /4 */

    /* 4. ADC1 configuration */
    ADC1->CR1 = 0;
    ADC1->CR2 = 0;

    /* Resolution: 12-bit (RES=00, default) */
    ADC1->CR1 &= ~ADC_CR1_RES;

    /* Right alignment (ALIGN=0, default) */
    ADC1->CR2 &= ~ADC_CR2_ALIGN;

    /* Single conversion mode (CONT=0) */
    ADC1->CR2 &= ~ADC_CR2_CONT;

    /* EOC set after each conversion (EOCS=1) */
    ADC1->CR2 |= ADC_CR2_EOCS;

    /* Sampling time for channel 0: 28 cycles */
    ADC1->SMPR2 &= ~ADC_SMPR2_SMP0;
    ADC1->SMPR2 |=  (2U << ADC_SMPR2_SMP0_Pos);   /* 010 = 28 cycles */

    /* Regular sequence: 1 conversion, channel 0 first */
    ADC1->SQR1 &= ~ADC_SQR1_L;      /* L=0000: 1 conversion */
    ADC1->SQR3  = 0;
    ADC1->SQR3 |= (0U << ADC_SQR3_SQ1_Pos);   /* SQ1 = channel 0 */

    /* 5. Power on ADC */
    ADC1->CR2 |= ADC_CR2_ADON;

    /* 6. Wait for ADC to stabilise (t_STAB ≈ a few µs) */
    for (volatile int i = 0; i < 100; i++);
}

/* Perform one conversion and return the result */
uint16_t ADC1_ReadChannel(uint8_t channel)
{
    /* Select channel in SQ1 */
    ADC1->SQR3 = channel & 0x1FU;

    /* Start conversion (software trigger) */
    ADC1->CR2 |= ADC_CR2_SWSTART;

    /* Wait for EOC */
    while (!(ADC1->SR & ADC_SR_EOC));

    /* Read result (clears EOC) */
    return (uint16_t)(ADC1->DR & 0x0FFF);
}

/* Convert to voltage */
float ADC_ToVoltage(uint16_t raw, float vdda)
{
    return (float)raw * vdda / 4095.0f;
}
```

### Multi-Channel Scan Mode (Bare-Metal)

```c
/*
 * ADC1 scan mode: 3 channels (CH0=PA0, CH1=PA1, CH4=PA4) in sequence
 * Results stored via DMA into a 3-element array
 */

volatile uint16_t adcResults[3];

void ADC1_Init_ScanDMA(void)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN | RCC_AHB1ENR_DMA2EN;
    RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;

    /* PA0, PA1, PA4 as analog */
    GPIOA->MODER |= (3U<<0) | (3U<<2) | (3U<<8);

    /* ADC prescaler /4 */
    ADC->CCR = ADC_CCR_ADCPRE_0;   /* /4 = 21 MHz */

    /* ADC1 config */
    ADC1->CR1 = ADC_CR1_SCAN;      /* SCAN=1: convert all channels in sequence */
    ADC1->CR2 = ADC_CR2_DMA        /* DMA enable */
              | ADC_CR2_DDS        /* DDS=1: DMA continuous (don't stop after last) */
              | ADC_CR2_CONT       /* Continuous: auto-restart sequence */
              | ADC_CR2_EOCS;      /* EOC after each conversion */

    /* Sampling times: 28 cycles for all */
    ADC1->SMPR2 = (2U<<0) | (2U<<3) | (2U<<12);  /* CH0, CH1, CH4 */

    /* Sequence: SQ1=CH0, SQ2=CH1, SQ3=CH4, L=2 (3 conversions) */
    ADC1->SQR1 = (2U << ADC_SQR1_L_Pos);   /* L=0010: 3 conversions */
    ADC1->SQR3 = (0U << 0)                  /* SQ1 = CH0 */
               | (1U << 5)                  /* SQ2 = CH1 */
               | (4U << 10);                /* SQ3 = CH4 */

    /* DMA2 Stream0 Channel0 for ADC1 */
    DMA2_Stream0->CR = 0;
    while (DMA2_Stream0->CR & DMA_SxCR_EN);

    DMA2->LIFCR = 0x3FU;   /* Clear all stream0 flags */

    DMA2_Stream0->PAR  = (uint32_t)&ADC1->DR;
    DMA2_Stream0->M0AR = (uint32_t)adcResults;
    DMA2_Stream0->NDTR = 3;
    DMA2_Stream0->CR   = (0U << DMA_SxCR_CHSEL_Pos)  /* Channel 0 */
                       | (0U << DMA_SxCR_DIR_Pos)    /* P2M */
                       | DMA_SxCR_MINC               /* Memory increment */
                       | (1U << DMA_SxCR_MSIZE_Pos)  /* 16-bit memory */
                       | (1U << DMA_SxCR_PSIZE_Pos)  /* 16-bit peripheral */
                       | DMA_SxCR_CIRC               /* Circular */
                       | DMA_SxCR_EN;

    /* Power on and start */
    ADC1->CR2 |= ADC_CR2_ADON;
    for (volatile int i = 0; i < 100; i++);
    ADC1->CR2 |= ADC_CR2_SWSTART;
}

/* Access results — updated continuously by DMA */
float GetVoltage_CH0(void) { return adcResults[0] * 3.3f / 4095.0f; }
float GetVoltage_CH1(void) { return adcResults[1] * 3.3f / 4095.0f; }
float GetVoltage_CH4(void) { return adcResults[2] * 3.3f / 4095.0f; }
```

---

## 7. HAL ADC Configuration (STM32CubeIDE/CubeMX)

### CubeMX ADC Settings

**ADC Mode:** Select channels in the "Configuration" pane.

**Parameter Settings:**

| Parameter                  | Description                                              | Typical Value     |
|----------------------------|----------------------------------------------------------|-------------------|
| Clock Prescaler            | ADC clock = APB2 / prescaler                             | PCLK2 divided by 4|
| Resolution                 | 6 / 8 / 10 / 12 bits                                    | 12 bits            |
| Data Alignment             | Right (12-bit result in bits 11:0) or Left               | Right alignment    |
| Scan Conversion Mode       | Enable for multiple channels in sequence                 | Enabled (multi-ch) |
| Continuous Conversion Mode | Auto-restart after sequence completes                    | Enabled (with DMA) |
| Discontinuous Conversion   | Convert N channels per trigger                           | Disabled           |
| DMA Continuous Requests    | Keep DMA running between sequences (DDS=1)               | Enabled            |
| End Of Conversion Selection| EOC after each conversion or after sequence              | EOC flag at each   |
| Number of Conversion       | How many channels in the regular sequence                | e.g., 3            |
| External Trigger           | Software / Timer trigger                                 | Software trigger   |
| External Trigger Edge      | Rising / Falling / Rising and Falling                    | Rising edge        |
| Rank x — Channel           | Which channel goes in position x of the sequence         | e.g., Channel 0    |
| Rank x — Sampling Time     | Cycles for that channel's sampling                       | 28 Cycles          |

### Generated Initialization Code

```c
ADC_HandleTypeDef hadc1;
DMA_HandleTypeDef hdma_adc1;

void MX_ADC1_Init(void)
{
    ADC_ChannelConfTypeDef sConfig = {0};

    hadc1.Instance                   = ADC1;
    hadc1.Init.ClockPrescaler        = ADC_CLOCK_SYNC_PCLK_DIV4;   /* 21 MHz */
    hadc1.Init.Resolution            = ADC_RESOLUTION_12B;
    hadc1.Init.ScanConvMode          = ENABLE;
    hadc1.Init.ContinuousConvMode    = ENABLE;
    hadc1.Init.DiscontinuousConvMode = DISABLE;
    hadc1.Init.ExternalTrigConvEdge  = ADC_EXTERNALTRIGCONVEDGE_NONE;
    hadc1.Init.ExternalTrigConv      = ADC_SOFTWARE_START;
    hadc1.Init.DataAlign             = ADC_DATAALIGN_RIGHT;
    hadc1.Init.NbrOfConversion       = 3;
    hadc1.Init.DMAContinuousRequests = ENABLE;
    hadc1.Init.EOCSelection          = ADC_EOC_SINGLE_CONV;
    HAL_ADC_Init(&hadc1);

    /* Channel 0 — rank 1 */
    sConfig.Channel      = ADC_CHANNEL_0;
    sConfig.Rank         = 1;
    sConfig.SamplingTime = ADC_SAMPLETIME_28CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    /* Channel 1 — rank 2 */
    sConfig.Channel = ADC_CHANNEL_1;
    sConfig.Rank    = 2;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    /* Channel 4 — rank 3 */
    sConfig.Channel = ADC_CHANNEL_4;
    sConfig.Rank    = 3;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);
}
```

---

## 8. ADC Polling Mode

```c
/* Single conversion, polling, single channel */
uint16_t ADC_ReadPoll(ADC_HandleTypeDef *hadc)
{
    HAL_ADC_Start(hadc);
    HAL_ADC_PollForConversion(hadc, HAL_MAX_DELAY);
    uint16_t val = HAL_ADC_GetValue(hadc);
    HAL_ADC_Stop(hadc);
    return val;
}

/* Usage */
uint16_t raw = ADC_ReadPoll(&hadc1);
float voltage = raw * 3.3f / 4095.0f;
printf("ADC: %u (%.3f V)\r\n", raw, voltage);

/* Read multiple channels in sequence (scan + polling) */
uint16_t results[3];
HAL_ADC_Start(&hadc1);
for (int i = 0; i < 3; i++)
{
    HAL_ADC_PollForConversion(&hadc1, 100);
    results[i] = HAL_ADC_GetValue(&hadc1);
}
HAL_ADC_Stop(&hadc1);
```

**When to use polling:**
- Simple, infrequent readings (e.g., battery voltage at startup)
- Debug and verification
- Low-speed sensors (thermistors, potentiometers)

**Disadvantage:** CPU is blocked until EOC — not suitable for high-speed or continuous sampling.

---

## 9. ADC Interrupt Mode

```c
volatile uint16_t adcResult = 0;
volatile uint8_t  adcReady  = 0;

/* Start single conversion with interrupt notification */
void ADC_Start_IT(void)
{
    HAL_ADC_Start_IT(&hadc1);
}

/* Callback: called when EOC occurs */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        adcResult = HAL_ADC_GetValue(hadc);
        adcReady  = 1;

        /* For continuous single-channel: HAL stops after 1 conversion unless CONT=1 */
        /* For scan mode: result is from the last channel in the sequence */

        /* Re-arm for next conversion if not continuous */
        /* HAL_ADC_Start_IT(&hadc1); */
    }
}

/* Main loop checks adcReady flag */
while (1)
{
    if (adcReady)
    {
        adcReady = 0;
        float v = adcResult * 3.3f / 4095.0f;
        ProcessVoltage(v);
        HAL_ADC_Start_IT(&hadc1);  /* Trigger next */
    }
}
```

**When to use interrupt mode:**
- Moderate sample rates (< ~50 kHz)
- Asynchronous conversions where you need notification
- Single-channel with variable trigger intervals

---

## 10. ADC DMA Mode — Single Channel

```c
#define ADC_BUF_SIZE 1024
uint16_t adcBuffer[ADC_BUF_SIZE];

/* Start continuous DMA — fills buffer in a loop */
void ADC_StartDMA(void)
{
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adcBuffer, ADC_BUF_SIZE);
}

/* Half-transfer: first 512 samples ready */
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        ProcessBlock(adcBuffer, ADC_BUF_SIZE / 2);
    }
}

/* Full-transfer: second 512 samples ready */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        ProcessBlock(adcBuffer + ADC_BUF_SIZE / 2, ADC_BUF_SIZE / 2);
    }
}

/* Error callback */
void HAL_ADC_ErrorCallback(ADC_HandleTypeDef *hadc)
{
    /* Handle DMA or ADC error */
    HAL_ADC_Stop_DMA(hadc);
    HAL_ADC_Start_DMA(hadc, (uint32_t*)adcBuffer, ADC_BUF_SIZE);
}
```

---

## 11. ADC DMA Mode — Multi-Channel Scan

When scan mode is active with DMA, each channel's result is stored in consecutive memory locations:

```c
/*
 * 4-channel scan: CH0(PA0), CH1(PA1), CH4(PA4), CH8(PB0)
 * DMA stores: [CH0_0, CH1_0, CH4_0, CH8_0, CH0_1, CH1_1, ...]
 * Buffer size = 4 channels × N_samples = 4 × 256 = 1024
 */

#define NUM_CHANNELS 4
#define SAMPLES_PER_CH 256
#define TOTAL_SAMPLES (NUM_CHANNELS * SAMPLES_PER_CH)

uint16_t adcScanBuf[TOTAL_SAMPLES];

/* After HAL_ADC_ConvCpltCallback: extract per-channel data */
void ExtractChannelData(uint16_t *raw, uint16_t *ch0, uint16_t *ch1,
                        uint16_t *ch4, uint16_t *ch8, uint16_t numSamples)
{
    for (uint16_t i = 0; i < numSamples; i++)
    {
        ch0[i] = raw[i * 4 + 0];
        ch1[i] = raw[i * 4 + 1];
        ch4[i] = raw[i * 4 + 2];
        ch8[i] = raw[i * 4 + 3];
    }
}

/* Or use macros for real-time access to latest value */
#define ADC_CH0_LATEST  adcScanBuf[0]   /* Only valid in non-continuous */
#define ADC_CH1_LATEST  adcScanBuf[1]
#define ADC_CH4_LATEST  adcScanBuf[2]
#define ADC_CH8_LATEST  adcScanBuf[3]
/* In continuous+DMA: DMA overwrites continuously; last complete set is valid */
```

---

## 12. ADC Continuous + DMA Circular Mode

This is the most efficient pattern for continuous data acquisition:

```c
/*
 * Continuous ADC sampling at maximum rate with circular DMA
 * Double-buffered via HT and TC callbacks
 * f_ADC=21 MHz, 12-bit, 3-cycle sampling: ~1.4 MSPS
 */

#define CIRC_BUF 2048
uint16_t circBuf[CIRC_BUF];

void ADC_ContinuousDMACircular_Start(void)
{
    /* hadc1 configured: Continuous=ENABLE, DMA=ENABLE, DDS=ENABLE */
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)circBuf, CIRC_BUF);
}

/* Process first half while DMA fills second half */
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef *hadc)
{
    ProcessSamples(circBuf, CIRC_BUF / 2);
}

/* Process second half while DMA rewinds to fill first half */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    ProcessSamples(circBuf + CIRC_BUF / 2, CIRC_BUF / 2);
}

/* Example processing: compute RMS */
float ComputeRMS(uint16_t *buf, uint16_t len)
{
    float sum = 0.0f;
    for (uint16_t i = 0; i < len; i++)
    {
        float v = buf[i] * 3.3f / 4095.0f;
        sum += v * v;
    }
    return sqrtf(sum / len);
}
```

---

## 13. Timer-Triggered ADC Sampling

For precise, jitter-free sampling at a known rate — essential for audio, vibration analysis, and motor current sensing:

```c
/*
 * TIM2 at 44100 Hz triggers ADC1 for audio sampling
 * No CPU involvement per sample — DMA handles everything
 */

void ADC_TimerTriggered_Init(void)
{
    /* --- TIM2 at 44.1 kHz --- */
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;
    TIM2->PSC = 0;
    TIM2->ARR = (uint32_t)(84000000UL / 44100UL) - 1;   /* ≈ 1904 */
    TIM2->CR2 |= TIM_CR2_MMS_1;   /* TRGO = Update Event */
    TIM2->CR1 |= TIM_CR1_CEN;

    /* --- ADC1: triggered by TIM2 TRGO --- */
    /* In CubeMX: External Trigger Conv = TIM2 TRGO, Edge = Rising */
    /* In HAL Init: */
    hadc1.Init.ExternalTrigConv     = ADC_EXTERNALTRIGCONV_T2_TRGO;
    hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_RISING;
    hadc1.Init.ContinuousConvMode   = DISABLE;   /* Single per trigger */
    hadc1.Init.DMAContinuousRequests= ENABLE;
    HAL_ADC_Init(&hadc1);

    /* Start DMA — ADC waits for TIM2 trigger each time */
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)audioBuffer, AUDIO_BUF_SIZE);
}
```

**EXTSEL values for ADC1 on STM32F411** (CR2 bits 27:24):

| EXTSEL | Trigger Source         |
|--------|------------------------|
| 0000   | TIM1 CC1               |
| 0001   | TIM1 CC2               |
| 0010   | TIM1 CC3               |
| 0011   | TIM2 CC2               |
| 0100   | TIM2 CC3               |
| 0101   | TIM2 CC4               |
| 0110   | TIM2 TRGO              |
| 0111   | TIM3 CC1               |
| 1000   | TIM3 TRGO              |
| 1001   | TIM4 CC4               |
| 1010   | TIM5 CC1               |
| 1011   | TIM5 CC2               |
| 1100   | TIM5 CC3               |
| 1101   | TIM8 CC1               |
| 1110   | TIM8 TRGO              |
| 1111   | EXTI line 11           |

---

## 14. ADC Injected Channels

Injected channels can pre-empt an ongoing regular conversion to sample urgent signals (e.g., phase current in motor FOC).

```c
/*
 * Regular: 3-channel scan for slow sensors
 * Injected: single fast channel for phase current (triggered by TIM1)
 */

ADC_InjectionConfTypeDef sConfigInjected = {0};

sConfigInjected.InjectedChannel               = ADC_CHANNEL_8;   /* PB0 */
sConfigInjected.InjectedRank                  = 1;
sConfigInjected.InjectedNbrOfConversion       = 1;
sConfigInjected.InjectedSamplingTime          = ADC_SAMPLETIME_3CYCLES;
sConfigInjected.ExternalTrigInjecConvEdge     = ADC_EXTERNALTRIGINJECCONVEDGE_RISING;
sConfigInjected.ExternalTrigInjecConv         = ADC_EXTERNALTRIGINJECCONV_T1_CC4;
sConfigInjected.AutoInjectedConv              = DISABLE;
sConfigInjected.InjectedDiscontinuousConvMode = DISABLE;
sConfigInjected.InjectedOffset                = 0;
HAL_ADCEx_InjectedConfigChannel(&hadc1, &sConfigInjected);

HAL_ADCEx_InjectedStart_IT(&hadc1);   /* Start injected with interrupt */

/* Injected conversion complete callback */
void HAL_ADCEx_InjectedConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc->Instance == ADC1)
    {
        uint16_t current = HAL_ADCEx_InjectedGetValue(hadc, ADC_INJECTED_RANK_1);
        /* Process phase current for FOC */
        FOC_CurrentFeedback(current);
    }
}
```

**Key differences from regular channels:**
- Separate trigger source (JEXTSEL, JEXTEN)
- Results stored in ADC_JDR1–4 (not DR) — no DMA needed
- Can be configured with an offset (subtracted from result automatically)
- 4 separate data registers — no overrun between channels
- Can interrupt and pause an ongoing regular sequence

---

## 15. Multi-ADC Modes (Dual/Triple)

> STM32F411 has only **one physical ADC (ADC1)**. True dual/triple mode (simultaneous sampling on ADC1+ADC2+ADC3) requires STM32F407/F429 or similar. On STM32F411 the CCR.MULTI field should remain 00000 (independent mode).

However, you can still achieve simultaneous-effect sampling on two channels using injected + regular at the same trigger, or by using the scan mode (channels are sampled sequentially with minimal delay).

---

## 16. Internal Channels — Temperature Sensor, VREFINT, VBAT

### Enable Internal Channels

```c
/* Must enable before use — shared with CH17 and CH18 */
ADC->CCR |= ADC_CCR_TSVREFE;   /* Enable temp sensor and VREFINT */
ADC->CCR |= ADC_CCR_VBATE;     /* Enable VBAT (channel 18 divided by 4) */
```

### Internal Temperature Sensor

```c
/*
 * Temperature sensor on ADC1_IN18 (STM32F411)
 * Must use sampling time ≥ 10 µs → 480 cycles at 21 MHz (22.86 µs)
 */

ADC_ChannelConfTypeDef sConfig = {0};
sConfig.Channel      = ADC_CHANNEL_TEMPSENSOR;   /* Maps to IN18 */
sConfig.Rank         = 1;
sConfig.SamplingTime = ADC_SAMPLETIME_480CYCLES; /* Minimum 10 µs */
HAL_ADC_ConfigChannel(&hadc1, &sConfig);
ADC->CCR |= ADC_CCR_TSVREFE;

/* Read and convert to temperature */
float ADC_ReadTemperature(void)
{
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 100);
    uint16_t raw = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);

    /* From STM32F411 datasheet:
     * T(°C) = (V_SENSE - V25) / Avg_Slope + 25
     * V25 = 0.76 V (typ), Avg_Slope = 2.5 mV/°C (typ)
     * But use factory calibration values for accuracy:
     */
    uint16_t *TS_CAL1 = (uint16_t*)0x1FFF7A2C;  /* Raw ADC at 30°C */
    uint16_t *TS_CAL2 = (uint16_t*)0x1FFF7A2E;  /* Raw ADC at 110°C */

    float temp = ((110.0f - 30.0f) * ((float)raw - *TS_CAL1))
                 / ((float)(*TS_CAL2) - *TS_CAL1)
                 + 30.0f;
    return temp;
}
```

### VREFINT — Internal Voltage Reference

VREFINT is a stable internal bandgap reference (~1.21 V). Reading it lets you calculate the actual VDDA voltage (useful when VDDA is not exactly 3.3 V, e.g., from a battery):

```c
float ADC_ReadVDDA(void)
{
    /* Configure CH17 (VREFINT) */
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel      = ADC_CHANNEL_VREFINT;  /* CH17 */
    sConfig.Rank         = 1;
    sConfig.SamplingTime = ADC_SAMPLETIME_480CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 100);
    uint16_t vrefRaw = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);

    /* Factory calibration: VREFINT_CAL at 3.3V */
    uint16_t *VREFINT_CAL = (uint16_t*)0x1FFF7A2A;

    /* VDDA = 3.3V × VREFINT_CAL / vrefRaw */
    float vdda = 3.3f * (float)(*VREFINT_CAL) / (float)vrefRaw;
    return vdda;
}
```

### VBAT — Battery Voltage Monitoring

On STM32F411, VBAT is divided by 4 before being fed to ADC1_IN18 (shares with temp sensor — cannot use both simultaneously):

```c
float ADC_ReadVBAT(void)
{
    ADC->CCR &= ~ADC_CCR_TSVREFE;  /* Disable temp sensor first */
    ADC->CCR |= ADC_CCR_VBATE;     /* Enable VBAT */

    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel      = ADC_CHANNEL_VBAT;
    sConfig.Rank         = 1;
    sConfig.SamplingTime = ADC_SAMPLETIME_480CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 100);
    uint16_t raw = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);

    /* VBAT = ADC_result × VDDA / 4095 × 4 (because /4 divider) */
    return (float)raw * 3.3f / 4095.0f * 4.0f;
}
```

---

## 17. ADC Calibration and Accuracy

### Factory Calibration Values (STM32F411)

Stored in read-only system memory during manufacturing:

| Symbol        | Address     | Description                                   |
|---------------|-------------|-----------------------------------------------|
| TS_CAL1       | 0x1FFF7A2C  | Temperature sensor raw value at 30°C, 3.3V    |
| TS_CAL2       | 0x1FFF7A2E  | Temperature sensor raw value at 110°C, 3.3V   |
| VREFINT_CAL   | 0x1FFF7A2A  | VREFINT raw value at 30°C, 3.3V               |

### Software Averaging / Oversampling

The STM32F411 ADC does not have hardware oversampling (unlike STM32G0/G4). Implement in software:

```c
uint16_t ADC_ReadAveraged(uint8_t channel, uint8_t numSamples)
{
    uint32_t sum = 0;
    for (uint8_t i = 0; i < numSamples; i++)
    {
        ADC1->SQR3 = channel;
        ADC1->CR2 |= ADC_CR2_SWSTART;
        while (!(ADC1->SR & ADC_SR_EOC));
        sum += ADC1->DR;
    }
    return (uint16_t)(sum / numSamples);
}

/* 16× averaging effectively gains ~2 bits of resolution (ENOB ≈ 14 bits) */
uint16_t precise = ADC_ReadAveraged(0, 16);
```

### Effective Number of Bits (ENOB)

Averaging N samples improves SNR by log2(N)/2 bits:
- 4 samples → +1 bit → 13-bit ENOB
- 16 samples → +2 bits → 14-bit ENOB
- 64 samples → +3 bits → 15-bit ENOB
- 256 samples → +4 bits → 16-bit ENOB (at the cost of sample rate)

---

## 18. Noise Reduction Techniques

### Hardware Techniques

1. **Decoupling capacitors:** Place 100 nF + 10 µF between VDDA and VSSA as close to the MCU as possible. Without these, digital switching noise couples into the ADC reference.

2. **VDDA filtering:** Use a small series resistor (10–100 Ω) + capacitor (1–10 µF) LC/RC filter on VDDA from VDD.

3. **Keep analog signals away from digital lines:** Route PCB traces to avoid crossing switching signals (clock, SPI, PWM). Use a ground plane under analog signals.

4. **Use VREF+ pin if available:** Connect a precision reference (e.g., LM4040 2.5V or REF3033 3.3V) to VREF+ for better accuracy than using VDDA as reference.

5. **Series input resistor + capacitor:** 10 kΩ + 10 nF at the ADC input pin forms a 1.6 kHz RC filter — rejects high-frequency noise while permitting DC or slow signals.

### Software Techniques

```c
/* Moving average filter */
#define MA_SIZE 16
uint16_t maBuffer[MA_SIZE];
uint8_t  maIndex = 0;
uint32_t maSum   = 0;

uint16_t MovingAverage(uint16_t newSample)
{
    maSum -= maBuffer[maIndex];
    maBuffer[maIndex] = newSample;
    maSum += newSample;
    maIndex = (maIndex + 1) % MA_SIZE;
    return (uint16_t)(maSum / MA_SIZE);
}

/* Median filter (removes impulse noise) */
#include <string.h>
uint16_t MedianFilter(uint16_t *buf, uint8_t len)
{
    uint16_t sorted[16];
    memcpy(sorted, buf, len * 2);
    /* Bubble sort (acceptable for len ≤ 16) */
    for (uint8_t i = 0; i < len-1; i++)
        for (uint8_t j = 0; j < len-1-i; j++)
            if (sorted[j] > sorted[j+1])
            {
                uint16_t t = sorted[j]; sorted[j]=sorted[j+1]; sorted[j+1]=t;
            }
    return sorted[len / 2];
}

/* Exponential moving average (IIR, low CPU) */
float ema_alpha = 0.1f;   /* 0 < alpha < 1; smaller = more smoothing */
float ema_value = 0.0f;

float EMA(float newSample)
{
    ema_value = ema_alpha * newSample + (1.0f - ema_alpha) * ema_value;
    return ema_value;
}
```

---

## 19. Converting Raw ADC Values to Real-World Units

### Voltage Measurement

```c
float ADC_ToVoltage(uint16_t raw, float vdda)
{
    return (float)raw * vdda / 4095.0f;
}
```

### Temperature via NTC Thermistor (Steinhart-Hart)

```c
/*
 * NTC thermistor: R_25=10kΩ, B=3950 K, pull-up R=10kΩ to 3.3V
 * V_ADC = 3.3V × R_NTC / (R_NTC + R_pullup)
 * R_NTC = R_pullup × V_ADC / (3.3 - V_ADC)
 */

float ADC_ToTemperatureNTC(uint16_t raw)
{
    float vdda = 3.3f;
    float v    = (float)raw * vdda / 4095.0f;

    if (v >= vdda) return -273.15f;   /* Short to VCC */
    if (v <= 0.0f) return 9999.0f;    /* Open circuit */

    float r_ntc = 10000.0f * v / (vdda - v);   /* R_pullup = 10kΩ */

    /* Steinhart-Hart simplified (B-parameter): */
    /* 1/T = 1/T0 + (1/B) × ln(R/R0) */
    float T0 = 298.15f;   /* 25°C in Kelvin */
    float B  = 3950.0f;
    float R0 = 10000.0f;  /* 10kΩ at 25°C */

    float invT = (1.0f / T0) + (1.0f / B) * logf(r_ntc / R0);
    return (1.0f / invT) - 273.15f;
}
```

### Current via Shunt Resistor + Amplifier

```c
/*
 * ACS712 current sensor: 185 mV/A, 0A → 2.5V output
 * Range: ±13.5A (0.185V/A × 13.5A = 2.5V swing from 2.5V midpoint)
 */
float ADC_ToCurrent_ACS712(uint16_t raw, float vdda)
{
    float v = (float)raw * vdda / 4095.0f;
    return (v - vdda / 2.0f) / 0.185f;   /* Subtract midpoint, divide by sensitivity */
}

/* INA219 style: shunt resistor with differential amplifier */
float ADC_ToCurrent_Shunt(uint16_t raw, float vdda, float shuntR, float gain)
{
    float v_shunt = ((float)raw * vdda / 4095.0f) / gain;
    return v_shunt / shuntR;
}
```

### Pressure Sensor (Linear Output)

```c
/* MPX5700 pressure sensor: 0.2V (0kPa) to 4.7V (700kPa) */
/* But limited to 3.3V range: 0.2V to 3.3V → 0 to ~450 kPa */
float ADC_ToPressure(uint16_t raw, float vdda)
{
    float v = (float)raw * vdda / 4095.0f;
    /* P(kPa) = (v - V_offset) / sensitivity */
    return (v - 0.2f) / ((4.7f - 0.2f) / 700.0f);
}
```

### Potentiometer (Position/Volume)

```c
float ADC_ToPotentiometerPct(uint16_t raw)
{
    return (float)raw / 4095.0f * 100.0f;
}

float ADC_ToPotentiometerDegrees(uint16_t raw, float maxDegrees)
{
    return (float)raw / 4095.0f * maxDegrees;
}
```

---

## 20. Common Pitfalls and Debugging

### 1. ADC clock exceeds 36 MHz
The most common mistake on STM32F411. APB2 = 84 MHz; prescaler /2 gives 42 MHz — illegal. Use /4 (21 MHz) or higher. Symptom: random readings, high noise, non-monotonic transfer function.

### 2. Sampling time too short
For sources with impedance > 1 kΩ, 3-cycle sampling is insufficient. The S/H capacitor doesn't fully charge → reading is lower than actual. Use at least 28 cycles for 10 kΩ sources. Use 480 cycles for temperature sensor / VREFINT.

### 3. GPIO not in analog mode
If the GPIO pin is left in default digital input mode, the ADC reads garbage because the digital input driver fights the analog signal. Must set MODER to 11 (analog) for each ADC input pin.

### 4. Missing TSVREFE bit for internal channels
Temperature sensor and VREFINT are disabled by default. Must set ADC_CCR.TSVREFE = 1. For VBAT: set ADC_CCR.VBATE = 1. VBAT and temp sensor share CH18 — cannot read both simultaneously.

### 5. DMA not restarting (DDS=0)
Without DDS=1 (DMA Disable Selection = continuous), DMA stops after the first sequence and EOC. For continuous DMA streaming, set DDS=1 in CR2 and DMA_CIRC in the DMA stream.

### 6. Scan mode result order confusion
In scan mode with DMA, results fill memory in sequence order (SQ1, SQ2, ..., SQn). If you put channel 4 in rank 1 and channel 0 in rank 2, adcBuffer[0] = CH4 result, adcBuffer[1] = CH0. Always match your array indexing to the sequence configuration.

### 7. Left-aligned vs right-aligned
Right alignment (default): bits [11:0] of DR are the 12-bit result.  
Left alignment (ALIGN=1): bits [15:4] of DR are the 12-bit result, bits [3:0] are zero.  
Left alignment is useful when you want to treat a 12-bit result as 16-bit: shift right 4 for the true value. Mixing alignments silently gives wrong results.

### 8. Reference voltage assumed to be exactly 3.3V
VDDA on a real board is rarely exactly 3.3 V — it varies with load and regulator tolerance. For accurate measurements: read VREFINT (CH17) and calculate actual VDDA = 3.3V × VREFINT_CAL / VREFINT_RAW. Then use actual VDDA in all voltage calculations.

### 9. EOC not clearing / stuck
Reading ADC_DR clears the EOC flag. If you check EOC but read another register, EOC stays set and you miss the next conversion. In HAL: `HAL_ADC_GetValue()` reads DR correctly. In bare-metal: always read DR to clear EOC.

### 10. Injected and regular channels using same trigger
If you accidentally configure both regular and injected groups with the same trigger and JAUTO=1, the injected group fires after every regular conversion, roughly doubling conversion time and disrupting the regular sequence timing.

---

## 21. Interview Questions and Answers

**Q1: What is the difference between regular and injected ADC channels?**

**A:** Regular channels form a sequence of up to 16 conversions that run in order. Their results are all written to the same single data register (ADC_DR), so DMA is required to capture all of them without overwriting. Injected channels are a high-priority group of up to 4 channels. When an injected trigger occurs, the ADC pauses the current regular conversion, performs all injected conversions, stores results in four separate registers (ADC_JDR1–4) — one per injected channel, so no DMA is needed — then resumes the interrupted regular conversion. This makes injected channels ideal for time-critical measurements like phase current sampling in motor FOC, where the current must be sampled at an exact moment defined by a timer event, without being affected by or disturbing the lower-priority regular sensor readings.

---

**Q2: Why is the ADC clock limited to 36 MHz on STM32F411 and what happens if you exceed it?**

**A:** The ADC's internal comparator, DAC, and SAR logic have a maximum operating frequency specification of 36 MHz. Exceeding this violates the analog timing — the internal comparator may not settle within the allotted time, the successive approximation steps may execute before the signal is stable, and the sample-and-hold may not fully capture the input. Symptoms of an overclocked ADC include increased noise, non-monotonic behavior (the digital output doesn't consistently increase as the analog input increases), readings that are systematically offset, and results that depend on what the previous conversion was. On STM32F411 with APB2 = 84 MHz, you must use at least /4 prescaler (21 MHz). Never use /2 (42 MHz).

---

**Q3: Explain the ADC sampling time and how you choose it.**

**A:** The sampling time is the number of ADC clock cycles during which the switch connecting the analog input to the internal sample-and-hold capacitor is closed. During this time the capacitor (typically 4–8 pF) charges to the input voltage through the source resistance. Too short a sampling time means the capacitor doesn't fully charge — the ADC reads a value lower than the actual input. The required sampling time depends on the source impedance: t_sample ≥ (R_source + R_internal) × C_SH × ln(2^(N+1)) where N is the bit resolution. For a 10 kΩ source at 12-bit: minimum is about 870 ns. At 21 MHz ADC clock that's about 18 ADC cycles — use 28 cycles (the next available value). For the internal temperature sensor the datasheet mandates at least 10 µs — always use 480 cycles for it.

---

**Q4: What is scan mode and how does DMA work with it?**

**A:** Scan mode enables sequential conversion of multiple channels. The ADC converts the channel in rank 1, then rank 2, up to rank L (total length), using independent sampling times per channel. Without DMA, you'd need to read ADC_DR after each conversion and the previous value would be overwritten. With DMA enabled (CR2.DMA=1 and CR2.DDS=1), after each channel conversion the ADC generates a DMA request, and DMA copies the result to the next memory location automatically. After L conversions, DMA has filled L consecutive memory words. In circular DMA mode, it wraps back to the start. This allows fully autonomous, CPU-free multi-channel acquisition at the maximum ADC throughput. The key: results appear in memory in rank order, not channel number order — the array index corresponds to the rank (position in the sequence), not the channel number.

---

**Q5: What is oversampling and how does it increase effective resolution?**

**A:** Oversampling means taking N samples of the same signal and averaging them. Each sample contains the true signal plus noise. If the noise is random (white), averaging N samples reduces the noise power by N → noise amplitude reduces by √N. In logarithmic terms, each 4× oversampling adds 1 bit of effective resolution (ENOB). To increase from 12-bit to 14-bit ENOB: take 16 samples and average them. To reach 16-bit ENOB: take 256 samples. The trade-off is sample rate: with 256× oversampling, your effective sample rate is the raw rate divided by 256. STM32F411 doesn't have hardware oversampling (unlike STM32G0), so it must be implemented in software or via DMA circular buffer + accumulator in an ISR.

---

**Q6: How do you use VREFINT to measure the actual VDDA?**

**A:** VREFINT is an internal bandgap reference with a factory-calibrated value. Anthropic stores the raw ADC reading of VREFINT at exactly 3.3 V VDDA in flash at address 0x1FFF7A2A (VREFINT_CAL). At runtime, measure VREFINT (channel 17) to get VREFINT_RAW. Since the ADC reading is proportional to V_in / VDDA, and VREFINT's actual voltage doesn't change with VDDA: VREFINT_RAW = VREFINT_actual / VDDA × 4095. Therefore: VDDA = VREFINT_actual × 4095 / VREFINT_RAW. Since VREFINT_CAL = VREFINT_actual / 3.3V × 4095, we get VDDA = 3.3V × VREFINT_CAL / VREFINT_RAW. This is the correct way to compensate all other ADC readings for VDDA variation — multiply every voltage result by this VDDA correction factor.

---

**Q7: What is the analog watchdog and when would you use it?**

**A:** The analog watchdog continuously compares each ADC result against programmable high and low thresholds (ADC_HTR and ADC_LTR registers). If a conversion result falls outside the window, the AWD flag is set and an interrupt fires (if AWDIE=1). This happens in hardware — no CPU polling required. Use cases: (1) battery voltage monitoring — fire interrupt when battery drops below 3.0 V or rises above 4.2 V; (2) temperature protection — fire interrupt when chip temperature exceeds a threshold; (3) pressure sensor safety — trigger alarm when pressure goes out of bounds; (4) current protection — detect overcurrent before it damages hardware. The watchdog can monitor a single channel (AWDSGL=1, AWDCH selects channel) or all channels in the sequence.

---

**Q8: Explain the difference between continuous mode and timer-triggered single conversion.**

**A:** In continuous mode (CONT=1), after each conversion completes the ADC immediately starts the next one, running as fast as possible. The sample rate is deterministic (set by ADC clock and sampling time) but fixed — you cannot control exactly when each sample is taken relative to other events. In timer-triggered single conversion (CONT=0, EXTEN≠00, EXTSEL = timer TRGO), each timer overflow generates one ADC conversion. The exact sample rate is controlled by the timer — you can set it to any frequency with high precision and synchronise it to other events (e.g., sample ADC exactly at the peak of a PWM carrier for minimum current ripple). Timer-triggered mode is preferred for: audio (exactly 44100 Hz), vibration analysis, current sensing in motor control, and any application requiring phase-coherent or exactly-timed sampling.

---

**Q9: What causes ADC noise and how do you reduce it?**

**A:** ADC noise has multiple sources: (1) thermal noise in the source and ADC input resistance — unavoidable, reduced by lower source impedance and lower bandwidth (more filtering); (2) quantisation noise — inherent to digitisation, reduced by using higher resolution or oversampling; (3) digital switching noise coupled into VDDA — the most common problem on STM32 boards — mitigated by placing 100 nF + 10 µF on VDDA close to the MCU pin, and adding a 10–33 Ω series resistor between VDD and VDDA; (4) ground bounce — ensure VSSA is connected to a clean analog ground, separate from the digital ground return paths; (5) capacitive coupling from nearby GPIO/PWM/SPI lines — keep analog traces short and isolated, use guard rings on sensitive PCB traces; (6) inadequate sampling time — if the S/H cap doesn't fully charge, readings are noisy and low.

---

**Q10: How does the STM32F411 temperature sensor work and what are its limitations?**

**A:** The internal temperature sensor is a reverse-biased diode whose forward voltage varies with temperature (approximately −2.5 mV/°C). The ADC reads this voltage via channel 18. Two factory calibration values are stored at 30°C and 110°C (TS_CAL1 and TS_CAL2 in system flash). Accurate temperature is: T = (110−30) × (raw − TS_CAL1) / (TS_CAL2 − TS_CAL1) + 30. Limitations: (1) it measures the die temperature, not ambient — self-heating from MCU activity adds 5–15°C error; (2) must use 480-cycle sampling (10 µs minimum per datasheet) — shorter sampling gives significantly wrong readings; (3) TSVREFE must be enabled in ADC_CCR; (4) typical accuracy is ±1.5°C after calibration, ±5°C without. For ambient temperature measurement, use an external sensor (NTC, DS18B20, LM75, BME280).

---

**Q11: What is discontinuous mode and when is it used?**

**A:** Discontinuous mode converts a sub-group of N channels per trigger instead of the entire sequence. If the sequence has 8 channels and DISCNUM=2, each trigger converts channels in ranks 1–2, then 3–4, then 5–6, then 7–8 on successive triggers. After the full sequence completes, the next trigger restarts from rank 1. This is used when: (1) you want to spread conversions across multiple trigger events to reduce instantaneous ADC load; (2) different channels need different trigger sources (not possible with a single trigger); (3) you want to interleave ADC conversions with other time-critical tasks — one trigger does 2 channels, the CPU processes them, the next trigger does 2 more. Without discontinuous mode a single trigger converts the entire sequence sequentially before stopping (single mode) or looping (continuous mode).

---

**Q12: How do you implement a simple software averaging for a 10-bit effective resolution from a 12-bit ADC?**

**A:** A 12-bit ADC with hardware oversampling could give better results, but in software: take 4 samples and average them. Averaging 4 samples reduces random noise variance by 4, reducing noise amplitude by 2 — gaining 1 bit of effective resolution (log2(4)/2 = 1). To get 10-bit effective precision from a noisy 12-bit ADC, average 4 samples: sum = sample1 + sample2 + sample3 + sample4; result = sum / 4. If you want a 14-bit result (shift the noise floor further), take 4 samples and instead of dividing by 4, keep the full sum (or divide by 1) — the sum spans 14 bits. Most practically: for general sensors, 4–16 sample averaging provides meaningful noise reduction; beyond 64 samples the returns diminish and sample rate becomes the bottleneck.
