# STM32F411 DMA — Complete Reference Guide

---

## Table of Contents

1. [Introduction to DMA](#1-introduction-to-dma)
2. [STM32F411 DMA Hardware Architecture](#2-stm32f411-dma-hardware-architecture)
3. [DMA Channels, Streams, and Request Mapping](#3-dma-channels-streams-and-request-mapping)
4. [DMA Transfer Types](#4-dma-transfer-types)
5. [DMA Registers (Bare-Metal)](#5-dma-registers-bare-metal)
6. [Bare-Metal DMA Configuration](#6-bare-metal-dma-configuration)
7. [HAL DMA Configuration (STM32CubeIDE/CubeMX)](#7-hal-dma-configuration-stm32cubeidecubemx)
8. [DMA with UART](#8-dma-with-uart)
9. [DMA with SPI](#9-dma-with-spi)
10. [DMA with ADC](#10-dma-with-adc)
11. [Memory-to-Memory DMA](#11-memory-to-memory-dma)
12. [DMA Interrupts](#12-dma-interrupts)
13. [DMA Double Buffering](#13-dma-double-buffering)
14. [DMA FIFO and Burst Transfers](#14-dma-fifo-and-burst-transfers)
15. [Common Pitfalls and Debugging](#15-common-pitfalls-and-debugging)
16. [Interview Questions and Answers](#16-interview-questions-and-answers)

---

## 1. Introduction to DMA

**DMA** (Direct Memory Access) is a hardware mechanism that allows peripherals or memory regions to transfer data to/from memory without CPU intervention. The CPU initiates the transfer and is only interrupted when the transfer completes (or at an error or half-complete event), freeing it to perform other tasks during the transfer.

**Without DMA (CPU-driven transfer):**
```
CPU: UART sends byte → CPU waits for TXE → CPU writes next byte → repeat
     During entire transfer, CPU is occupied only with moving data
     100 bytes at 115200 baud = ~8.7ms of CPU time wasted waiting
```

**With DMA:**
```
CPU: Configure DMA → start transfer → go do other work
DMA: Moves bytes autonomously from memory to UART DR
     At transfer end, DMA fires interrupt → CPU processes completion
     100 bytes at 115200 baud = ~8.7ms CPU is free for other tasks
```

**DMA is essential for:**
- High-speed UART, SPI, I2C data transfers
- ADC continuous sampling (12-bit samples at up to 2.4 MSPS)
- Audio/I2S streaming
- Display refresh (bulk pixel data to SPI LCD)
- Memory copy (memcpy acceleration)
- Any transfer where CPU involvement per byte is too expensive

---

## 2. STM32F411 DMA Hardware Architecture

The STM32F411 has **two DMA controllers**:
- **DMA1:** 8 streams, serves APB1 peripherals (UART2, I2C1/2/3, SPI2/3, TIM2-5, etc.)
- **DMA2:** 8 streams, serves APB2 peripherals (SPI1, USART1/6, ADC, TIM1, TIM8, SDIO)

Each DMA controller has **8 streams** (channels 0–7). Each stream can be connected to one of up to **8 channels** (hardware request lines from peripherals). The stream-channel mapping is fixed in hardware.

### DMA Stream Architecture

```
Peripheral (e.g., USART1_RX)
        │  DMA Request (hardware trigger)
        ▼
  ┌─────────────┐
  │  DMA Stream │  ← 8 streams per controller
  │   Engine    │
  │  ┌────────┐ │
  │  │  FIFO  │ │  ← Optional 4-word FIFO buffer
  │  └────────┘ │
  └─────────────┘
        │ AHB bus access
        ▼
   System Memory (SRAM, Flash)
```

### Key Concepts

**Stream:** A hardware engine within the DMA controller that executes one transfer at a time.

**Channel:** A logical connection to a peripheral request line. Each stream can be configured to listen to one of 8 channels. The channel determines which peripheral's DMA request triggers the stream.

**Request:** A hardware signal from a peripheral (e.g., UART RX data ready) that tells the DMA to perform one transfer element (byte/halfword/word).

**Arbitration:** When multiple streams request bus access simultaneously, a priority scheme decides which goes first. Priority is configurable (4 levels) per stream.

---

## 3. DMA Channels, Streams, and Request Mapping

### DMA1 Stream → Channel → Peripheral Request Mapping

| Stream | Ch0         | Ch1          | Ch2          | Ch3          | Ch4          | Ch5           | Ch6           | Ch7          |
|--------|-------------|--------------|--------------|--------------|--------------|---------------|---------------|--------------|
| 0      | SPI3_RX     | I2C1_RX (*)  | TIM4_CH1     | I2S3_EXT_RX  | UART5_RX     | UART8_TX(*)   | TIM5_CH3/UP   | —            |
| 1      | —           | —            | TIM7_UP      | TIM2_UP/CH3  | TIM3_CH1/TRIG| TIM2_CH1      | I2C3_RX(*)    | TIM5_CH4/TRIG|
| 2      | SPI3_RX     | TIM7_UP      | I2S3_EXT_RX  | I2C3_RX      | UART4_RX     | TIM3_CH4/UP   | TIM5_CH1      | I2C2_RX      |
| 3      | SPI2_RX     | TIM4_CH2     | I2C3_RX(*)   | I2S2_EXT_RX  | TIM3_CH2     | UART7_TX(*)   | TIM5_CH4/TRIG | I2C2_RX(*)   |
| 4      | SPI2_TX     | TIM4_CH3     | I2C3_TX(*)   | —            | TIM3_CH1     | UART7_RX(*)   | TIM5_CH2      | I2C2_TX      |
| 5      | SPI3_TX     | I2C1_RX      | I2C1_RX(*)   | TIM2_CH1     | USART2_RX    | TIM3_CH2      | DAC1          | DAC2         |
| 6      | —           | I2C1_TX      | TIM4_UP      | TIM2_CH2/CH4 | USART2_TX    | TIM3_CH1/TRIG | DAC2(*)       | I2C1_TX(*)   |
| 7      | —           | I2C1_TX(*)   | —            | TIM2_CH4     | UART5_TX     | TIM3_CH3      | I2C2_TX(*)    | I2C1_TX(*)   |

*(*) = alternate mapping, device-specific*

### DMA2 Stream → Channel → Peripheral Request (Key entries for STM32F411)

| Stream | Ch0       | Ch1       | Ch2          | Ch3       | Ch4       | Ch5        | Ch6        | Ch7      |
|--------|-----------|-----------|--------------|-----------|-----------|------------|------------|----------|
| 0      | ADC1      | ADC1      | ADC1         | SPI1_RX   | SPI4_RX   | —          | TIM1_TRIG  | —        |
| 1      | —         | DCMI      | ADC2(*)      | SPI4_TX   | SPI4_TX   | USART6_RX  | TIM1_CH1   | TIM8_UP  |
| 2      | TIM8_CH1  | ADC2      | SPI1_RX(*)   | USART1_RX | —         | SPI5_TX    | TIM1_CH2   | SPI5_RX  |
| 3      | SPI1_RX   | SPI5_RX   | SPI5_TX      | USART1_TX | —         | SPI4_RX    | TIM1_CH1   | SPI5_TX  |
| 4      | ADC1(*)   | —         | USART1_TX(*) | —         | —         | —          | TIM1_CH4   | USART1_TX|
| 5      | —         | USART6_RX | —            | SPI4_TX   | SPI5_TX   | SPI5_RX    | TIM1_UP    | SPI5_TX  |
| 6      | TIM1_CH1  | TIM1_CH2  | TIM1_CH1(*)  | TIM1_CH4  | —         | SPI5_RX(*) | TIM1_CH3   | —        |
| 7      | —         | DCMI(*)   | —            | USART1_TX | USART6_TX | —          | USART6_TX(*) | —      |

**Critical rule:** You must use the exact stream + channel combination from the mapping table. You cannot arbitrarily combine them.

---

## 4. DMA Transfer Types

### By Source/Destination

| Mode                 | Source    | Destination | Description                             |
|----------------------|-----------|-------------|---------------------------------------- |
| Peripheral-to-Memory | Peripheral register | SRAM | UART RX, ADC read, SPI RX      |
| Memory-to-Peripheral | SRAM      | Peripheral register | UART TX, SPI TX, DAC write   |
| Memory-to-Memory     | SRAM/Flash| SRAM        | Fast memcpy (DMA2 only on STM32F4)     |

### By Transfer Mode

| Mode     | Description                                                         |
|----------|---------------------------------------------------------------------|
| Normal   | Transfer N items, then stop. Must be re-armed for next transfer.    |
| Circular | Automatically restarts when complete. Loops forever until stopped.  |

**Normal mode:** DMA transfers exactly the configured number of bytes, then disables itself (clears EN bit) and fires the Transfer Complete interrupt. To start another transfer, you must reconfigure and re-enable.

**Circular mode:** After transferring N bytes, DMA restarts from the beginning automatically. Address pointers reset. Generates TC interrupt at end, and optionally HT (half-transfer) interrupt at midpoint. Never disables itself — must be stopped explicitly. Used for: ADC continuous sampling, UART receive ring buffer, audio streaming.

### Data Widths

| Data Width | Peripheral | Memory | Effect                                  |
|------------|-----------|--------|-----------------------------------------|
| Byte       | 8-bit     | 8-bit  | 1:1 transfer                            |
| Halfword   | 16-bit    | 16-bit | 2 bytes per transfer                    |
| Word       | 32-bit    | 32-bit | 4 bytes per transfer                    |
| Mixed      | 8-bit     | 32-bit | DMA packs 4 bytes into one word (FIFO needed) |

---

## 5. DMA Registers (Bare-Metal)

Each stream has its own set of registers at offset `0x10 + stream_number × 0x18` from the DMA base.

### DMA_SxCR — Stream x Configuration Register

| Bits  | Name   | Description                                            |
|-------|--------|--------------------------------------------------------|
| 27:25 | CHSEL  | Channel select (0–7, maps to peripheral request)       |
| 24:23 | MBURST | Memory burst size (00=single, 01=INCR4, 10=INCR8, 11=INCR16)|
| 22:21 | PBURST | Peripheral burst size                                  |
| 19    | CT     | Current target (double-buffer mode)                    |
| 18    | DBM    | Double-buffer mode enable                              |
| 17:16 | PL     | Priority level (00=low, 01=medium, 10=high, 11=very high)|
| 15    | PINCOS | Peripheral increment offset size                       |
| 14:13 | MSIZE  | Memory data size (00=byte, 01=halfword, 10=word)       |
| 12:11 | PSIZE  | Peripheral data size                                   |
| 10    | MINC   | Memory address increment enable                        |
| 9     | PINC   | Peripheral address increment enable                    |
| 8     | CIRC   | Circular mode enable                                   |
| 7:6   | DIR    | Data direction (00=P2M, 01=M2P, 10=M2M)               |
| 5     | PFCTRL | Peripheral flow control (0=DMA controls, 1=peripheral controls)|
| 4     | TCIE   | Transfer complete interrupt enable                     |
| 3     | HTIE   | Half-transfer interrupt enable                         |
| 2     | TEIE   | Transfer error interrupt enable                        |
| 1     | DMEIE  | Direct mode error interrupt enable                     |
| 0     | EN     | Stream enable                                          |

### DMA_SxNDTR — Number of Data Items to Transfer

Contains the number of data items (bytes/halfwords/words depending on MSIZE). Decremented by hardware after each transfer. In normal mode, reads 0 when done. In circular mode, auto-reloads.

### DMA_SxPAR — Peripheral Address Register

The address of the peripheral data register (e.g., `&USART1->DR`, `&ADC1->DR`, `&SPI1->DR`).

### DMA_SxM0AR — Memory 0 Address Register

The address of the memory buffer (target/source in SRAM).

### DMA_SxM1AR — Memory 1 Address Register

Only used in double-buffer mode. The address of the second memory buffer.

### DMA_SxFCR — FIFO Control Register

| Bits | Name  | Description                                       |
|------|-------|---------------------------------------------------|
| 7    | FEIE  | FIFO error interrupt enable                       |
| 6    | FS    | FIFO status (read-only)                           |
| 3    | DMDIS | Direct mode disable (0=direct, 1=use FIFO)        |
| 1:0  | FTH   | FIFO threshold (00=1/4, 01=1/2, 10=3/4, 11=full) |

### Status Registers: DMA_LISR / DMA_HISR

Each DMA controller has two 32-bit status registers covering all 8 streams:
- LISR: streams 0–3
- HISR: streams 4–7

Per-stream flags (4 bits per stream):
- TCIFx: Transfer complete
- HTIFx: Half transfer
- TEIFx: Transfer error
- DMEIFx: Direct mode error
- FEIFx: FIFO error

Cleared by writing 1 to the corresponding bit in **DMA_LIFCR / DMA_HIFCR**.

---

## 6. Bare-Metal DMA Configuration

### Example: USART2_RX → Memory (P2M, Normal Mode)

```c
/*
 * Configure DMA1 Stream 5, Channel 4 for USART2 RX
 * (From DMA1 mapping table: USART2_RX = Stream5, Channel4)
 */

#define RX_BUF_SIZE 128
uint8_t rxBuffer[RX_BUF_SIZE];

void DMA1_USART2_RX_Init(void)
{
    /* 1. Enable DMA1 clock */
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA1EN;
    
    /* 2. Disable the stream before configuring */
    DMA1_Stream5->CR &= ~DMA_SxCR_EN;
    while (DMA1_Stream5->CR & DMA_SxCR_EN);  // Wait for disable
    
    /* 3. Clear all interrupt flags for Stream 5 */
    DMA1->HIFCR |= DMA_HIFCR_CTCIF5 | DMA_HIFCR_CHTIF5 |
                   DMA_HIFCR_CTEIF5 | DMA_HIFCR_CDMEIF5 | DMA_HIFCR_CFEIF5;
    
    /* 4. Set peripheral address (USART2 data register) */
    DMA1_Stream5->PAR = (uint32_t)&USART2->DR;
    
    /* 5. Set memory address */
    DMA1_Stream5->M0AR = (uint32_t)rxBuffer;
    
    /* 6. Set number of data items */
    DMA1_Stream5->NDTR = RX_BUF_SIZE;
    
    /* 7. Configure the stream */
    DMA1_Stream5->CR = 0;  // Clear all
    DMA1_Stream5->CR |= (4U << DMA_SxCR_CHSEL_Pos); // Channel 4
    DMA1_Stream5->CR |= (0U << DMA_SxCR_DIR_Pos);   // P2M (00)
    DMA1_Stream5->CR |= (0U << DMA_SxCR_MSIZE_Pos); // Memory: byte
    DMA1_Stream5->CR |= (0U << DMA_SxCR_PSIZE_Pos); // Peripheral: byte
    DMA1_Stream5->CR |= DMA_SxCR_MINC;              // Increment memory address
    // PINC=0: peripheral address stays fixed (USART DR)
    // CIRC=0: normal mode (not circular)
    DMA1_Stream5->CR |= (2U << DMA_SxCR_PL_Pos);    // High priority
    DMA1_Stream5->CR |= DMA_SxCR_TCIE;              // Transfer complete interrupt
    
    /* 8. Direct mode (no FIFO) */
    DMA1_Stream5->FCR &= ~DMA_SxFCR_DMDIS;  // Direct mode enable (DMDIS=0)
    
    /* 9. Enable NVIC for DMA1 Stream 5 */
    NVIC_SetPriority(DMA1_Stream5_IRQn, 2);
    NVIC_EnableIRQ(DMA1_Stream5_IRQn);
    
    /* 10. Enable DMA request in USART2 */
    USART2->CR3 |= USART_CR3_DMAR;
    
    /* 11. Enable the DMA stream */
    DMA1_Stream5->CR |= DMA_SxCR_EN;
}

/* DMA1 Stream5 IRQ Handler */
void DMA1_Stream5_IRQHandler(void)
{
    if (DMA1->HISR & DMA_HISR_TCIF5)
    {
        DMA1->HIFCR |= DMA_HIFCR_CTCIF5;  // Clear flag
        
        // RX_BUF_SIZE bytes received
        ProcessReceivedData(rxBuffer, RX_BUF_SIZE);
        
        // Re-arm: reset NDTR and re-enable
        DMA1_Stream5->CR &= ~DMA_SxCR_EN;
        while (DMA1_Stream5->CR & DMA_SxCR_EN);
        DMA1_Stream5->NDTR = RX_BUF_SIZE;
        DMA1_Stream5->CR |= DMA_SxCR_EN;
    }
    
    if (DMA1->HISR & DMA_HISR_TEIF5)
    {
        DMA1->HIFCR |= DMA_HIFCR_CTEIF5;
        // Handle transfer error
    }
}
```

### Example: Memory-to-Peripheral (SPI1 TX)

```c
/*
 * DMA2 Stream 3, Channel 3 for SPI1 TX
 * Source: SRAM buffer → Destination: SPI1_DR
 */
void DMA2_SPI1_TX_Init(uint8_t *txBuf, uint16_t len)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;
    
    DMA2_Stream3->CR &= ~DMA_SxCR_EN;
    while (DMA2_Stream3->CR & DMA_SxCR_EN);
    
    DMA2->LIFCR |= (DMA_LIFCR_CTCIF3 | DMA_LIFCR_CTEIF3 | DMA_LIFCR_CDMEIF3);
    
    DMA2_Stream3->PAR  = (uint32_t)&SPI1->DR;
    DMA2_Stream3->M0AR = (uint32_t)txBuf;
    DMA2_Stream3->NDTR = len;
    
    DMA2_Stream3->CR = 0;
    DMA2_Stream3->CR |= (3U << DMA_SxCR_CHSEL_Pos); // Channel 3
    DMA2_Stream3->CR |= (1U << DMA_SxCR_DIR_Pos);   // M2P (01)
    DMA2_Stream3->CR |= DMA_SxCR_MINC;              // Increment memory
    DMA2_Stream3->CR |= DMA_SxCR_TCIE;
    DMA2_Stream3->CR |= (2U << DMA_SxCR_PL_Pos);
    
    /* Enable SPI TX DMA */
    SPI1->CR2 |= SPI_CR2_TXDMAEN;
    
    DMA2_Stream3->CR |= DMA_SxCR_EN;
}
```

---

## 7. HAL DMA Configuration (STM32CubeIDE/CubeMX)

### CubeMX DMA Settings

In CubeMX, DMA is configured per-peripheral in the "DMA Settings" tab of each peripheral:

**DMA Request:** Select the peripheral (e.g., USART1_RX)  
**Stream:** Auto-suggested based on mapping table  
**Direction:** Peripheral To Memory / Memory To Peripheral / Memory To Memory  
**Priority:** Low / Medium / High / Very High  
**Mode:** Normal / Circular  
**Data Width (Peripheral):** Byte / Half Word / Word  
**Data Width (Memory):** Byte / Half Word / Word  
**Increment Address (Memory):** Enable (usually yes)  
**Increment Address (Peripheral):** Disable (usually — peripheral register is fixed)  
**Use FIFO:** Enable/Disable (needed for width mismatch or burst transfers)  
**Threshold:** 1/4, 1/2, 3/4, Full  
**Burst:** Single / INCR4 / INCR8 / INCR16  

### Generated Code

```c
DMA_HandleTypeDef hdma_usart1_rx;

/* Called inside HAL_UART_MspInit() */
void HAL_UART_MspInit(UART_HandleTypeDef* uartHandle)
{
    if (uartHandle->Instance == USART1)
    {
        /* DMA1 Stream 5 for USART1_RX (example) */
        hdma_usart1_rx.Instance                 = DMA2_Stream2;
        hdma_usart1_rx.Init.Channel             = DMA_CHANNEL_4;
        hdma_usart1_rx.Init.Direction           = DMA_PERIPH_TO_MEMORY;
        hdma_usart1_rx.Init.PeriphInc           = DMA_PINC_DISABLE;
        hdma_usart1_rx.Init.MemInc              = DMA_MINC_ENABLE;
        hdma_usart1_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_usart1_rx.Init.MemDataAlignment    = DMA_MDATAALIGN_BYTE;
        hdma_usart1_rx.Init.Mode                = DMA_CIRCULAR;
        hdma_usart1_rx.Init.Priority            = DMA_PRIORITY_HIGH;
        hdma_usart1_rx.Init.FIFOMode            = DMA_FIFOMODE_DISABLE;
        
        HAL_DMA_Init(&hdma_usart1_rx);
        
        /* Link DMA handle to UART handle */
        __HAL_LINKDMA(uartHandle, hdmaRx, hdma_usart1_rx);
        
        /* NVIC for DMA */
        HAL_NVIC_SetPriority(DMA2_Stream2_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA2_Stream2_IRQn);
    }
}

/* IRQ handler routes to HAL */
void DMA2_Stream2_IRQHandler(void)
{
    HAL_DMA_IRQHandler(&hdma_usart1_rx);
}
```

---

## 8. DMA with UART

See the UART document for full coverage. The essential pattern:

```c
/* Circular DMA receive with IDLE detection */
uint8_t uartRxBuf[256];

void UART_DMA_CircularReceive_Start(void)
{
    HAL_UART_Receive_DMA(&huart1, uartRxBuf, sizeof(uartRxBuf));
    __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);  // Enable IDLE interrupt
}

/* Override USART1_IRQHandler */
void USART1_IRQHandler(void)
{
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE))
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);
        
        uint32_t remaining = __HAL_DMA_GET_COUNTER(huart1.hdmarx);
        uint32_t received  = sizeof(uartRxBuf) - remaining;
        
        ProcessUARTData(uartRxBuf, received);
        
        // In circular mode, DMA continues automatically
        // Reset DMA position tracking if needed
    }
    HAL_UART_IRQHandler(&huart1);
}

void HAL_UART_RxHalfCpltCallback(UART_HandleTypeDef *huart)
{
    // First half of buffer ready — process first 128 bytes
    ProcessUARTData(uartRxBuf, 128);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    // Second half of buffer ready — process second 128 bytes
    ProcessUARTData(uartRxBuf + 128, 128);
}
```

---

## 9. DMA with SPI

```c
/* Large SPI transfer via DMA — sends 4096 bytes of display data */
uint8_t displayBuffer[320 * 240 * 2];  // 150,400 bytes for 320x240 RGB565

void Display_Refresh_DMA(void)
{
    CS_LOW();
    HAL_SPI_Transmit_DMA(&hspi1, displayBuffer, sizeof(displayBuffer));
}

void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi)
{
    if (hspi->Instance == SPI1)
    {
        // Wait for SPI BSY to clear
        while (hspi->Instance->SR & SPI_SR_BSY);
        CS_HIGH();
        // Display refreshed — prepare next frame
    }
}
```

---

## 10. DMA with ADC

DMA + ADC circular mode is the standard pattern for continuous analog sampling:

```c
/*
 * ADC1 + DMA2 Stream0 Channel0 (circular mode)
 * Continuously samples ADC channel and stores in buffer
 */
uint16_t adcBuffer[2048];  // 2048 samples

void ADC_DMA_Init(void)
{
    /* Configure DMA2 Stream0, Channel0 for ADC1 */
    hdma_adc1.Instance                 = DMA2_Stream0;
    hdma_adc1.Init.Channel             = DMA_CHANNEL_0;
    hdma_adc1.Init.Direction           = DMA_PERIPH_TO_MEMORY;
    hdma_adc1.Init.PeriphInc           = DMA_PINC_DISABLE;
    hdma_adc1.Init.MemInc              = DMA_MINC_ENABLE;
    hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;  // ADC is 12-bit (16-bit reg)
    hdma_adc1.Init.MemDataAlignment    = DMA_MDATAALIGN_HALFWORD;
    hdma_adc1.Init.Mode                = DMA_CIRCULAR;
    hdma_adc1.Init.Priority            = DMA_PRIORITY_HIGH;
    hdma_adc1.Init.FIFOMode            = DMA_FIFOMODE_DISABLE;
    HAL_DMA_Init(&hdma_adc1);
    __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);
}

/* Start continuous ADC sampling with DMA */
void ADC_StartContinuous(void)
{
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adcBuffer, 2048);
}

/* Half buffer ready */
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef *hadc)
{
    // Process first 1024 samples (adcBuffer[0..1023])
    // This runs while the second half is filling
    ProcessADCSamples(adcBuffer, 1024);
}

/* Full buffer ready */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    // Process second 1024 samples (adcBuffer[1024..2047])
    ProcessADCSamples(adcBuffer + 1024, 1024);
}
```

---

## 11. Memory-to-Memory DMA

DMA2 on STM32F4 supports memory-to-memory transfers (DMA1 does not).

```c
/* Fast memcpy using DMA2 */
uint8_t srcBuf[4096];
uint8_t dstBuf[4096];
volatile uint8_t dmaM2MDone = 0;

DMA_HandleTypeDef hdma_m2m;

void DMA_MemCopy(void *dst, const void *src, uint32_t len)
{
    hdma_m2m.Instance                 = DMA2_Stream0;
    hdma_m2m.Init.Channel             = DMA_CHANNEL_0;
    hdma_m2m.Init.Direction           = DMA_MEMORY_TO_MEMORY;
    hdma_m2m.Init.PeriphInc           = DMA_PINC_ENABLE;   // Source increments
    hdma_m2m.Init.MemInc              = DMA_MINC_ENABLE;   // Dest increments
    hdma_m2m.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
    hdma_m2m.Init.MemDataAlignment    = DMA_MDATAALIGN_WORD;
    hdma_m2m.Init.Mode                = DMA_NORMAL;
    hdma_m2m.Init.Priority            = DMA_PRIORITY_HIGH;
    hdma_m2m.Init.FIFOMode            = DMA_FIFOMODE_ENABLE;
    hdma_m2m.Init.FIFOThreshold       = DMA_FIFO_THRESHOLD_FULL;
    hdma_m2m.Init.MemBurst            = DMA_MBURST_INC4;
    hdma_m2m.Init.PeriphBurst         = DMA_PBURST_INC4;
    HAL_DMA_Init(&hdma_m2m);
    
    dmaM2MDone = 0;
    HAL_DMA_Start_IT(&hdma_m2m, (uint32_t)src, (uint32_t)dst, len / 4);  // len in words
    
    // Wait for completion (or use callback for non-blocking)
    while (!dmaM2MDone);
}

void HAL_DMA_M2M_CpltCallback(DMA_HandleTypeDef *hdma)
{
    dmaM2MDone = 1;
}
```

**Performance comparison at 84 MHz:**
- CPU memcpy (word-by-word): ~84M words/s → ~336 MB/s theoretical
- DMA M2M (burst mode, word-aligned): Similar speed but CPU is free
- For small copies (<16 bytes): CPU is faster due to DMA setup overhead

---

## 12. DMA Interrupts

DMA generates 5 interrupt sources per stream, all routed through one IRQ line:

| Interrupt | Flag   | Description                                             |
|-----------|--------|---------------------------------------------------------|
| TC        | TCIF   | Transfer Complete — all N items transferred             |
| HT        | HTIF   | Half Transfer — N/2 items transferred                   |
| TE        | TEIF   | Transfer Error — bus error during transfer              |
| DME       | DMEIF  | Direct Mode Error — FIFO underrun/overrun in direct mode|
| FE        | FEIF   | FIFO Error — overflow/underflow of FIFO                 |

All five flags for the same stream share one NVIC interrupt line. The IRQ handler must check which flag is set.

```c
void DMA2_Stream0_IRQHandler(void)
{
    /* Check Transfer Complete */
    if (DMA2->LISR & DMA_LISR_TCIF0)
    {
        DMA2->LIFCR |= DMA_LIFCR_CTCIF0;  // Clear TC flag
        TransferComplete_Callback();
    }
    /* Check Half Transfer */
    if (DMA2->LISR & DMA_LISR_HTIF0)
    {
        DMA2->LIFCR |= DMA_LIFCR_CHTIF0;
        HalfTransfer_Callback();
    }
    /* Check Transfer Error */
    if (DMA2->LISR & DMA_LISR_TEIF0)
    {
        DMA2->LIFCR |= DMA_LIFCR_CTEIF0;
        Error_Handler();
    }
}
```

**In HAL:** The HAL DMA IRQ handler (`HAL_DMA_IRQHandler()`) reads these flags, clears them, and calls the registered callbacks automatically.

---

## 13. DMA Double Buffering

Double buffering (also called ping-pong buffering) uses two memory buffers. While DMA fills/reads one buffer, the CPU processes the other. Enabled by setting DBM=1 in SxCR.

```c
uint8_t buffer0[512];
uint8_t buffer1[512];
volatile uint8_t processBuf0 = 0;
volatile uint8_t processBuf1 = 0;

void DMA_DoubleBuffer_Init(void)
{
    DMA2_Stream0->CR &= ~DMA_SxCR_EN;
    while (DMA2_Stream0->CR & DMA_SxCR_EN);
    
    DMA2_Stream0->M0AR = (uint32_t)buffer0;
    DMA2_Stream0->M1AR = (uint32_t)buffer1;
    DMA2_Stream0->NDTR = 512;
    DMA2_Stream0->CR  |= DMA_SxCR_DBM;    // Enable double-buffer
    DMA2_Stream0->CR  |= DMA_SxCR_TCIE;   // TC interrupt
    DMA2_Stream0->CR  |= DMA_SxCR_EN;
}

void DMA2_Stream0_IRQHandler(void)
{
    if (DMA2->LISR & DMA_LISR_TCIF0)
    {
        DMA2->LIFCR |= DMA_LIFCR_CTCIF0;
        
        /* CT bit tells which buffer DMA just FINISHED writing to */
        if (DMA2_Stream0->CR & DMA_SxCR_CT)
        {
            // CT=1 means DMA is now writing to buffer1, buffer0 is ready
            processBuf0 = 1;
        }
        else
        {
            // CT=0 means DMA is now writing to buffer0, buffer1 is ready
            processBuf1 = 1;
        }
    }
}

/* In main loop */
while (1)
{
    if (processBuf0) { ProcessData(buffer0, 512); processBuf0 = 0; }
    if (processBuf1) { ProcessData(buffer1, 512); processBuf1 = 0; }
}
```

**Benefit:** Zero dead time — while DMA fills buffer1, CPU processes buffer0. When buffer1 is full, DMA automatically switches to buffer0 and CPU processes buffer1. No gaps in data acquisition.

---

## 14. DMA FIFO and Burst Transfers

### Direct Mode vs FIFO Mode

**Direct mode (FIFO disabled):** Each DMA request from the peripheral triggers exactly one transfer. The data goes directly from peripheral to memory (or vice versa). Simple and low-latency.

**FIFO mode (FIFO enabled):** Data accumulates in the 4-word FIFO before being written to memory. This allows burst transfers and data-width packing (e.g., receive bytes from peripheral but write words to memory).

**FIFO threshold:** How full the FIFO must be before a burst write to memory:
- FTH=00: 1/4 full (1 word) — lowest latency, lowest efficiency
- FTH=11: full (4 words) — highest efficiency, highest latency

### Burst Transfers

Bursts allow the DMA to access memory in multi-beat bursts (4, 8, or 16 beats), which is more efficient on the AHB bus than single-beat accesses:

```c
hdma.Init.FIFOMode      = DMA_FIFOMODE_ENABLE;
hdma.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
hdma.Init.MemBurst      = DMA_MBURST_INC4;    // 4-beat memory burst
hdma.Init.PeriphBurst   = DMA_PBURST_SINGLE;  // Single peripheral access
```

**Rules:**
- Burst size × data size must not exceed 16 bytes (max FIFO size)
- For MemBurst=INCR4 and MSIZE=WORD: 4 × 4 = 16 bytes → OK
- For MemBurst=INCR16 and MSIZE=WORD: 16 × 4 = 64 bytes → ILLEGAL

---

## 15. Common Pitfalls and Debugging

### 1. Wrong stream/channel combination
The most common bare-metal mistake. Each peripheral's DMA request maps to specific streams and channels. Consult the DMA request mapping table in the reference manual (Table 27 for DMA1, Table 28 for DMA2). Using the wrong channel means the DMA stream never gets triggered.

### 2. Not waiting for stream to disable before reconfiguring
You cannot write to stream configuration registers while EN=1. Always:
```c
DMA1_Stream5->CR &= ~DMA_SxCR_EN;
while (DMA1_Stream5->CR & DMA_SxCR_EN);  // Poll until disabled
// Now safe to reconfigure
```

### 3. Not clearing interrupt flags before enabling
Old flags from a previous transfer will immediately trigger the ISR when EN is set. Always clear LIFCR/HIFCR before enabling:
```c
DMA1->LIFCR |= DMA_LIFCR_CTCIF5 | DMA_LIFCR_CHTIF5 | DMA_LIFCR_CTEIF5 | 
               DMA_LIFCR_CDMEIF5 | DMA_LIFCR_CFEIF5;
```

### 4. Buffer in flash (read-only memory)
The destination buffer for P2M transfers must be in SRAM, not Flash. SRAM address range on STM32F411: 0x20000000–0x2001FFFF (128 KB). Ensure your buffer is a global or stack variable, not a const array in Flash.

### 5. Cache coherency (not applicable to STM32F411)
STM32F411 does not have D-cache, so this is not an issue. However, on STM32F7/H7 which have D-cache, DMA and CPU can see different data if the cache is not invalidated/cleaned. This is a very common bug on those devices.

### 6. DMA transfer error (TEIF set)
Causes: (1) peripheral address is incorrect (access fault), (2) memory address is NULL or unmapped, (3) wrong bus access width. Use debugger to inspect SxPAR and SxM0AR.

### 7. Missing NDTR reload in normal mode
In normal mode, NDTR counts down to 0 and DMA disables itself. You must: disable stream, set NDTR again, then re-enable. Forgetting this means only one transfer ever occurs.

### 8. Not enabling peripheral DMA request
DMA by itself does nothing without the peripheral requesting it. Each peripheral has a DMA enable bit: USART_CR3.DMAR/DMAT, SPI_CR2.RXDMAEN/TXDMAEN, ADC_CR2.DMA. If you forget to set this, the DMA stream is configured but never triggered.

---

## 16. Interview Questions and Answers

**Q1: What is DMA and why is it needed?**

**A:** DMA (Direct Memory Access) is a hardware subsystem that moves data between memory and peripherals (or between memory regions) without requiring CPU instructions for each data element. Without DMA, the CPU must actively poll or respond to an interrupt for every byte transferred — at SPI 42 MHz, that's 5.25 million byte-level interrupts per second, consuming more CPU bandwidth than available. DMA solves this by having dedicated transfer hardware that autonomously moves data. The CPU is only involved at setup and completion. This frees the CPU for application logic and enables sustained high-bandwidth transfers that would otherwise be impossible.

---

**Q2: Explain the difference between DMA Normal mode and Circular mode.**

**A:** In Normal mode, the DMA transfers exactly the configured number of data items (NDTR), then stops — it clears the EN bit and generates a Transfer Complete interrupt. The stream must be manually reconfigured and re-enabled for the next transfer. This is used for finite transfers: send 100 bytes over SPI, read 50 bytes from UART. In Circular mode, when the NDTR counter reaches 0, it automatically reloads to its original value and the stream pointer resets to M0AR — the DMA immediately restarts. It generates TC and HT interrupts each cycle but never stops unless the EN bit is explicitly cleared. Circular mode is used for continuous data acquisition (ADC sampling, audio streaming, UART ring buffer receive) where data flows indefinitely.

---

**Q3: What is the Half Transfer interrupt and why is it important?**

**A:** The Half Transfer (HT) interrupt fires when exactly half the configured NDTR items have been transferred. Combined with the Transfer Complete (TC) interrupt (at the end of all items), these two interrupts divide the circular buffer into two halves. When HT fires, the DMA is writing to the second half — the CPU can safely process the first half. When TC fires, the DMA has wrapped and is writing to the first half again — the CPU can process the second half. This is called double-buffering or ping-pong buffering and allows continuous, uninterrupted data processing without the CPU and DMA ever accessing the same memory region simultaneously. It is essential for ADC streaming, audio processing, and UART receive without data loss.

---

**Q4: How do you choose which DMA stream and channel to use for a peripheral?**

**A:** The stream-channel mapping is fixed in hardware and documented in the reference manual (RM0383 for STM32F411) in the DMA chapter's request mapping tables. Each peripheral's DMA request is hardwired to specific streams and channels in the DMA controller. For example, USART2_RX can only use DMA1 Stream5 Channel4 — there is no choice. To configure DMA: look up the peripheral in the table, note the DMA controller (1 or 2), stream number (0–7), and channel number (0–7), then program CHSEL bits in SxCR with the channel number, use the correct stream's registers (DMA1_Stream5 or DMA2_Stream0 etc.). Using the wrong combination means the stream is never triggered by that peripheral's DMA request.

---

**Q5: What is the difference between direct mode and FIFO mode in DMA?**

**A:** In direct mode (FIFO disabled), each DMA request from a peripheral triggers exactly one data transfer element (byte, halfword, or word) from the peripheral register directly to memory. This is simple, low-latency, and ideal when peripheral and memory data widths match. The limitation: you cannot use burst memory transfers in direct mode. In FIFO mode, received data accumulates in the 4-word FIFO. Once the configured threshold is reached (1/4, 1/2, 3/4, or full), the DMA performs a burst write to memory. FIFO mode is required when: (1) peripheral and memory data widths differ (e.g., 8-bit UART data but 32-bit memory writes), (2) burst transfers are needed for efficiency, (3) memory-to-memory transfers (always requires FIFO). The trade-off for FIFO mode is higher latency (data waits in FIFO before being written to memory) and more configuration complexity.

---

**Q6: Can DMA1 do memory-to-memory transfers?**

**A:** No. On STM32F4 series (including STM32F411), only DMA2 supports memory-to-memory transfers. DMA1 only supports peripheral-to-memory and memory-to-peripheral. This is a hardware limitation — DMA1 lacks the ability to generate AHB bus requests to a source that isn't a peripheral. Memory-to-memory via DMA2 is useful for fast buffer copying as a hardware-accelerated alternative to software memcpy, particularly for large buffers where the CPU would otherwise be occupied for many cycles.

---

**Q7: What is the DMA arbitration mechanism and why does it matter?**

**A:** When multiple DMA streams simultaneously request access to the AHB bus, a priority arbiter decides which stream gets the next bus cycle. Each stream has a configurable software priority (4 levels: Low, Medium, High, Very High) set via the PL bits in SxCR. When software priorities are equal, the stream with the lower stream number wins (stream 0 > stream 1 > ... > stream 7). Priority matters because lower-priority streams may be starved if high-priority streams continuously request transfers. In a system with both a high-speed SPI (needing frequent bus access) and an ADC (time-critical sampling), you must carefully assign priorities to ensure neither starves the other. Setting all streams to the same priority lets stream number break ties.

---

**Q8: How does the DMA handle variable-length UART messages?**

**A:** Standard DMA receive is configured for a fixed number of bytes — it doesn't know when a "message" ends. The solution is combining DMA circular mode with UART IDLE line detection. Configure DMA in circular mode with a large buffer. Enable the UART IDLE interrupt (fires when the bus is idle after receiving data). In the IDLE interrupt handler, calculate how many bytes were received by subtracting the DMA remaining counter (NDTR) from the buffer size: `received = bufSize - DMA->NDTR`. This gives the exact length of the received message regardless of size. Then process that many bytes from the buffer. This pattern handles any message length up to the buffer size without knowing it in advance.

---

**Q9: What registers hold DMA status, and how do you clear interrupt flags?**

**A:** DMA1 and DMA2 each have two 32-bit status registers: LISR (Low Interrupt Status Register, covers streams 0–3) and HISR (High Interrupt Status Register, covers streams 4–7). Each stream has 5 flag bits in these registers. Flags are read from LISR/HISR but cleared by writing 1 to the corresponding bit in LIFCR/HIFCR (Clear registers). You CANNOT clear flags by writing 0 to LISR/HISR directly — that has no effect. The common sequence: `if (DMA1->LISR & DMA_LISR_TCIF0) { DMA1->LIFCR |= DMA_LIFCR_CTCIF0; /* handle event */ }`. In HAL, `HAL_DMA_IRQHandler()` reads flags, clears them, and calls registered callbacks automatically.

---

**Q10: What happens if a DMA transfer error (TEIF) occurs?**

**A:** A Transfer Error occurs when the DMA AHB master port receives a bus error during a data access — typically an illegal memory address (NULL pointer, unaligned access, peripheral address typo). When TEIF is set, the stream is automatically disabled (EN clears) and the transfer stops. The TEIE interrupt fires if enabled. To recover: (1) clear the TEIF flag in LIFCR/HIFCR, (2) fix the underlying cause (correct the peripheral/memory address), (3) reconfigure and re-enable the stream. A common cause is setting the peripheral address to a variable's value rather than its address: `SxPAR = USART1->DR` (wrong — stores the current data value) vs `SxPAR = (uint32_t)&USART1->DR` (correct — stores the register address).
