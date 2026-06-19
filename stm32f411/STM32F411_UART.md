# STM32F411 UART — Complete Reference Guide

---

## Table of Contents

1. [Introduction to UART](#1-introduction-to-uart)
2. [STM32F411 USART/UART Hardware Overview](#2-stm32f411-usartuart-hardware-overview)
3. [UART Signal Lines and Electrical Characteristics](#3-uart-signal-lines-and-electrical-characteristics)
4. [UART Frame Format](#4-uart-frame-format)
5. [Baud Rate Generation](#5-baud-rate-generation)
6. [UART Registers (Bare-Metal)](#6-uart-registers-bare-metal)
7. [Bare-Metal UART Configuration](#7-bare-metal-uart-configuration)
8. [HAL UART Configuration (STM32CubeIDE/CubeMX)](#8-hal-uart-configuration-stm32cubeidecubemx)
9. [HAL UART — Polling Mode](#9-hal-uart--polling-mode)
10. [HAL UART — Interrupt Mode](#10-hal-uart--interrupt-mode)
11. [HAL UART — DMA Mode](#11-hal-uart--dma-mode)
12. [UART Interrupts Deep Dive](#12-uart-interrupts-deep-dive)
13. [Flow Control (RTS/CTS)](#13-flow-control-rtscts)
14. [RS-485 Half-Duplex with DE Pin](#14-rs-485-half-duplex-with-de-pin)
15. [printf() Retargeting via UART](#15-printf-retargeting-via-uart)
16. [Common Pitfalls and Debugging](#16-common-pitfalls-and-debugging)
17. [Interview Questions and Answers](#17-interview-questions-and-answers)

---

## 1. Introduction to UART

**UART** (Universal Asynchronous Receiver-Transmitter) is one of the oldest and most widely used serial communication protocols in embedded systems. Unlike SPI or I2C, UART is **asynchronous** — it does not use a shared clock line. Both sides must agree on the same baud rate in advance. Communication happens over just two wires: TX (transmit) and RX (receive), making it extremely simple to implement.

**Key properties:**
- Asynchronous — no shared clock
- Full-duplex — TX and RX are independent
- Point-to-point — no addressing, only two devices per link
- Simple hardware and software requirements
- Commonly used for debug consoles, GPS, Bluetooth modules, GSM modems

**USART vs UART:**  
STM32 peripherals are called **USART** (Universal Synchronous/Asynchronous Receiver-Transmitter). In synchronous mode, a clock line (CK) is provided. In asynchronous mode (the most common case), the CK pin is unused and the peripheral behaves as a standard UART. The terms are often used interchangeably in practice.

---

## 2. STM32F411 USART/UART Hardware Overview

The STM32F411 features **3 USART/UART peripherals**:

| Peripheral | Type   | APB Bus | Max Baud (84 MHz APB) | Default Pins         |
|------------|--------|---------|----------------------|----------------------|
| USART1     | USART  | APB2    | 5.25 Mbps            | PA9 (TX), PA10 (RX)  |
| USART2     | USART  | APB1    | 2.625 Mbps           | PA2 (TX), PA3 (RX)   |
| USART6     | USART  | APB2    | 5.25 Mbps            | PC6 (TX), PC7 (RX)   |

**Clock sources:**
- USART1 and USART6 are on APB2 (max 84 MHz on STM32F411)
- USART2 is on APB1 (max 42 MHz on STM32F411)

**Features supported:**
- 8 or 9 data bits
- 1 or 2 stop bits
- None, even, or odd parity
- Hardware flow control (RTS/CTS) on USART1, USART2, USART6
- LIN (Local Interconnect Network) break detection
- Smartcard mode (ISO 7816)
- IrDA mode
- Multi-processor communication (muting)
- DMA request generation for both TX and RX
- Wakeup from STOP mode (on some USARTs)

---

## 3. UART Signal Lines and Electrical Characteristics

| Signal | Direction         | Description                                 |
|--------|-------------------|---------------------------------------------|
| TX     | MCU → Device      | Transmit data line (idle high)              |
| RX     | Device → MCU      | Receive data line                           |
| RTS    | MCU → Device      | Request to Send (flow control, active low)  |
| CTS    | Device → MCU      | Clear to Send (flow control, active low)    |
| CK     | MCU output        | Synchronous clock (USART only)              |

**Electrical levels:**  
STM32F411 GPIO pins are 3.3 V logic. Most UART-capable modules are also 3.3 V. For 5 V devices (e.g., classic Arduino), use a level shifter or resistor divider on the RX line.

**Cross-wiring rule:**  
The TX pin of one device must connect to the RX pin of the other, and vice versa.

```
MCU TX ──────────────► Device RX
MCU RX ◄────────────── Device TX
MCU GND ─────────────── Device GND
```

---

## 4. UART Frame Format

A UART frame consists of:

```
IDLE  START  D0  D1  D2  D3  D4  D5  D6  D7  [PARITY]  STOP  IDLE
HIGH   LOW   ── data bits (LSB first) ──   [P]    HIGH   HIGH
```

**Components:**
1. **Idle state** — line is HIGH when no data is being sent
2. **Start bit** — always LOW (1 bit), signals the beginning of a frame
3. **Data bits** — 5 to 9 bits, LSB transmitted first
4. **Parity bit** (optional) — even or odd parity for error detection
5. **Stop bit(s)** — always HIGH, 1 or 2 bits, signals end of frame

**Example: 8N1 format (most common)**
- 8 data bits
- No parity
- 1 stop bit
- Total: 10 bits per byte (1 start + 8 data + 1 stop)
- At 115200 baud: one byte takes ~86.8 µs

---

## 5. Baud Rate Generation

The STM32F411 UART uses a **fractional baud rate generator** to achieve precise baud rates from any clock frequency.

**Formula:**

```
USARTDIV = fCLK / (8 × (2 - OVER8) × BaudRate)
```

Where:
- `fCLK` = APB clock frequency (e.g., 84 MHz for USART1 on APB2)
- `OVER8` = oversampling mode bit in CR1 (0 = 16x oversampling, 1 = 8x oversampling)
- `USARTDIV` is split into mantissa (integer part, bits [15:4]) and fraction (bits [3:0] for 16x, [2:0] for 8x)

**Example: 115200 baud, fCLK = 84 MHz, OVER8 = 0**

```
USARTDIV = 84,000,000 / (16 × 115200) = 45.5729...
Mantissa = 45  → stored in BRR[15:4]
Fraction = 0.5729 × 16 = 9.166 ≈ 9  → stored in BRR[3:0]
BRR = (45 << 4) | 9 = 0x02D9
Actual baud = 84,000,000 / (16 × 45.5625) = 115,207 baud (~0.006% error)
```

**16x vs 8x oversampling:**
- 16x (OVER8=0): Default, better noise tolerance, lower max baud rate
- 8x (OVER8=1): Allows up to fCLK/8 baud rate, less noise immunity

**Common baud rates and BRR values (fCLK = 84 MHz, 16x oversampling):**

| Baud Rate | USARTDIV     | BRR (hex) | Error   |
|-----------|-------------|-----------|---------|
| 9600      | 546.875      | 0x2223    | 0%      |
| 38400     | 136.71875    | 0x88B     | 0%      |
| 115200    | 45.5729      | 0x2D9     | 0.006%  |
| 230400    | 22.7864      | 0x16C     | 0.01%   |
| 921600    | 5.6962       | 0x05B     | 0.05%   |

---

## 6. UART Registers (Bare-Metal)

### USART_SR — Status Register

| Bit | Name | Description                                      |
|-----|------|--------------------------------------------------|
| 9   | CTS  | CTS flag                                         |
| 8   | LBD  | LIN break detected                               |
| 7   | TXE  | Transmit data register empty (ready to load)     |
| 6   | TC   | Transmission complete                            |
| 5   | RXNE | Read data register not empty (data received)     |
| 4   | IDLE | Idle line detected                               |
| 3   | ORE  | Overrun error                                    |
| 2   | NF   | Noise detected flag                              |
| 1   | FE   | Framing error                                    |
| 0   | PE   | Parity error                                     |

### USART_DR — Data Register

- Bits [8:0] contain the data to transmit or the received byte.
- Writing to DR puts data in the TX shift register.
- Reading from DR reads the received byte and clears RXNE.

### USART_BRR — Baud Rate Register

| Bits  | Name             | Description              |
|-------|------------------|--------------------------|
| 15:4  | DIV_Mantissa     | Integer part of USARTDIV |
| 3:0   | DIV_Fraction     | Fractional part          |

### USART_CR1 — Control Register 1

| Bit | Name   | Description                                    |
|-----|--------|------------------------------------------------|
| 15  | OVER8  | Oversampling (0=16x, 1=8x)                    |
| 13  | UE     | USART Enable                                   |
| 12  | M      | Word length (0=8 bit, 1=9 bit)                 |
| 11  | WAKE   | Wakeup method                                  |
| 10  | PCE    | Parity control enable                          |
| 9   | PS     | Parity selection (0=even, 1=odd)               |
| 8   | PEIE   | Parity error interrupt enable                  |
| 7   | TXEIE  | TXE interrupt enable                           |
| 6   | TCIE   | TC interrupt enable                            |
| 5   | RXNEIE | RXNE interrupt enable                          |
| 4   | IDLEIE | IDLE interrupt enable                          |
| 3   | TE     | Transmitter enable                             |
| 2   | RE     | Receiver enable                                |

### USART_CR2 — Control Register 2

| Bits | Name  | Description                               |
|------|-------|-------------------------------------------|
| 13:12 | STOP | Stop bits (00=1, 01=0.5, 10=2, 11=1.5)  |
| 11   | CLKEN | Clock enable (synchronous mode)           |

### USART_CR3 — Control Register 3

| Bit | Name  | Description                              |
|-----|-------|------------------------------------------|
| 11  | ONEBIT| One sample bit (noise tolerance)         |
| 10  | CTSIE | CTS interrupt enable                     |
| 9   | CTSE  | CTS enable                               |
| 8   | RTSE  | RTS enable                               |
| 7   | DMAT  | DMA enable for transmitter               |
| 6   | DMAR  | DMA enable for receiver                  |

---

## 7. Bare-Metal UART Configuration

### Step 1: Enable Clocks

```c
// Enable GPIOA clock (for PA9=TX, PA10=RX on USART1)
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;

// Enable USART1 clock (on APB2)
RCC->APB2ENR |= RCC_APB2ENR_USART1EN;
```

### Step 2: Configure GPIO Pins as Alternate Function

```c
// PA9 (TX) and PA10 (RX): Alternate function AF7 (USART1)
// Set mode to Alternate Function (10b)
GPIOA->MODER &= ~((3U << (9*2)) | (3U << (10*2)));
GPIOA->MODER |=  ((2U << (9*2)) | (2U << (10*2)));

// Set alternate function to AF7 for PA9 and PA10
GPIOA->AFR[1] &= ~((0xF << ((9-8)*4)) | (0xF << ((10-8)*4)));
GPIOA->AFR[1] |=  ((7U << ((9-8)*4)) | (7U << ((10-8)*4)));

// Set output speed to high
GPIOA->OSPEEDR |= ((3U << (9*2)) | (3U << (10*2)));

// Set output type to push-pull (for TX)
GPIOA->OTYPER &= ~((1U << 9) | (1U << 10));

// Set pull-up on RX (PA10)
GPIOA->PUPDR &= ~(3U << (10*2));
GPIOA->PUPDR |=  (1U << (10*2)); // Pull-up
```

### Step 3: Configure USART1 Registers

```c
// Disable USART before configuring
USART1->CR1 &= ~USART_CR1_UE;

// Set baud rate: 115200 @ 84 MHz APB2, 16x oversampling
// USARTDIV = 84000000 / (16 * 115200) = 45.57
// Mantissa = 45 = 0x2D, Fraction = 0.57 * 16 = 9 = 0x9
// BRR = (0x2D << 4) | 0x9 = 0x2D9
USART1->BRR = 0x02D9;

// Word length 8 bits (M=0), no parity (PCE=0), 16x oversampling (OVER8=0)
USART1->CR1 = 0;  // Clear all

// Enable TX and RX
USART1->CR1 |= USART_CR1_TE | USART_CR1_RE;

// 1 stop bit (default in CR2)
USART1->CR2 = 0;

// No flow control, no DMA
USART1->CR3 = 0;

// Enable USART
USART1->CR1 |= USART_CR1_UE;
```

### Step 4: Transmit and Receive Functions

```c
/* --- Transmit a single byte (polling) --- */
void UART1_SendByte(uint8_t data)
{
    // Wait until TXE (transmit data register empty)
    while (!(USART1->SR & USART_SR_TXE));
    USART1->DR = data;
}

/* --- Transmit a string --- */
void UART1_SendString(const char *str)
{
    while (*str)
    {
        UART1_SendByte((uint8_t)(*str++));
    }
    // Wait for transmission complete
    while (!(USART1->SR & USART_SR_TC));
}

/* --- Receive a single byte (blocking) --- */
uint8_t UART1_ReceiveByte(void)
{
    // Wait until RXNE (receive data register not empty)
    while (!(USART1->SR & USART_SR_RXNE));
    return (uint8_t)(USART1->DR & 0xFF);
}

/* --- Non-blocking receive (check if data available) --- */
uint8_t UART1_DataAvailable(void)
{
    return (USART1->SR & USART_SR_RXNE) ? 1 : 0;
}
```

### Step 5: Interrupt-Driven Receive (Bare-Metal)

```c
/* Enable RXNE interrupt in NVIC */
void UART1_EnableRxInterrupt(void)
{
    USART1->CR1 |= USART_CR1_RXNEIE;  // Enable RXNE interrupt
    NVIC_SetPriority(USART1_IRQn, 2);
    NVIC_EnableIRQ(USART1_IRQn);
}

/* Interrupt handler */
#define RX_BUFFER_SIZE 128
volatile uint8_t rxBuffer[RX_BUFFER_SIZE];
volatile uint16_t rxHead = 0, rxTail = 0;

void USART1_IRQHandler(void)
{
    if (USART1->SR & USART_SR_RXNE)
    {
        uint8_t byte = (uint8_t)(USART1->DR & 0xFF); // Reading DR clears RXNE
        rxBuffer[rxHead] = byte;
        rxHead = (rxHead + 1) % RX_BUFFER_SIZE;
    }
    if (USART1->SR & USART_SR_ORE)
    {
        // Clear overrun error by reading SR then DR
        volatile uint32_t tmp = USART1->SR;
        tmp = USART1->DR;
        (void)tmp;
    }
}

uint8_t UART1_ReadFromBuffer(uint8_t *data)
{
    if (rxHead == rxTail) return 0; // Buffer empty
    *data = rxBuffer[rxTail];
    rxTail = (rxTail + 1) % RX_BUFFER_SIZE;
    return 1;
}
```

---

## 8. HAL UART Configuration (STM32CubeIDE/CubeMX)

### CubeMX Settings

When you open CubeMX and select USART1:

**Mode:**
- Asynchronous — standard UART
- Synchronous — uses CK pin (uncommon)
- Single Wire (Half-Duplex) — TX and RX share one wire
- Multiprocessor Communication — mute mode
- IrDA — infrared
- LIN — Local Interconnect Network
- Smartcard — ISO 7816

**Parameter Settings:**

| Parameter          | Description                                              | Common Value     |
|--------------------|----------------------------------------------------------|------------------|
| Baud Rate          | Bits per second                                          | 115200           |
| Word Length        | Number of data bits including parity if enabled          | 8 Bits           |
| Parity             | None / Even / Odd                                        | None             |
| Stop Bits          | 1 or 2                                                   | 1                |
| Data Direction     | Receive and Transmit / RX only / TX only                 | Receive and Transmit |
| Over Sampling      | 16 Samples (default) or 8 Samples                        | 16 Samples       |

**DMA Settings:**
- Add DMA Request for USART1_RX and/or USART1_TX
- Choose Normal or Circular mode
- Data width: Byte

**NVIC Settings:**
- Enable USART1 global interrupt for interrupt-driven operation

### Generated Initialization Code

CubeMX generates this in `usart.c`:

```c
UART_HandleTypeDef huart1;

void MX_USART1_UART_Init(void)
{
    huart1.Instance          = USART1;
    huart1.Init.BaudRate     = 115200;
    huart1.Init.WordLength   = UART_WORDLENGTH_8B;
    huart1.Init.StopBits     = UART_STOPBITS_1;
    huart1.Init.Parity       = UART_PARITY_NONE;
    huart1.Init.Mode         = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl    = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    
    if (HAL_UART_Init(&huart1) != HAL_OK)
    {
        Error_Handler();
    }
}
```

The `HAL_UART_Init()` function internally:
1. Enables the peripheral clock via `HAL_UART_MspInit()`
2. Configures GPIO pins (alternate function, speed, pull-up/down)
3. Programs BRR, CR1, CR2, CR3
4. Enables the UE bit

---

## 9. HAL UART — Polling Mode

Polling mode blocks CPU execution until the operation completes or times out.

```c
/* Transmit */
uint8_t txData[] = "Hello, World!\r\n";
HAL_StatusTypeDef status;

status = HAL_UART_Transmit(&huart1, txData, sizeof(txData) - 1, HAL_MAX_DELAY);
if (status != HAL_OK)
{
    // Handle error: HAL_TIMEOUT or HAL_ERROR
}

/* Receive exactly 10 bytes */
uint8_t rxData[10];
status = HAL_UART_Receive(&huart1, rxData, 10, 1000); // 1000ms timeout
if (status == HAL_TIMEOUT)
{
    // Less than 10 bytes received within 1 second
}
else if (status == HAL_OK)
{
    // All 10 bytes received
}

/* Transmit with printf — after retargeting */
printf("Temperature: %.2f C\r\n", temperature);
```

**When to use polling:**
- Simple debug output
- Startup messages
- Low data rates where blocking is acceptable

**Disadvantage:** CPU is blocked during the entire transfer — cannot do anything else.

---

## 10. HAL UART — Interrupt Mode

In interrupt mode, the HAL starts the transfer and returns immediately. The CPU is notified via interrupt when the transfer is complete.

```c
/* Global buffer */
uint8_t rxByte;
uint8_t rxBuffer[256];
uint16_t rxIndex = 0;

/* Start receiving 1 byte at a time in interrupt mode */
void StartUARTReceive(void)
{
    HAL_UART_Receive_IT(&huart1, &rxByte, 1);
}

/* This callback is called by HAL every time the requested bytes are received */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        rxBuffer[rxIndex++] = rxByte;
        
        if (rxByte == '\n' || rxIndex >= sizeof(rxBuffer) - 1)
        {
            rxBuffer[rxIndex] = '\0';
            // Process complete line
            ProcessCommand((char*)rxBuffer);
            rxIndex = 0;
        }
        
        // Re-arm the interrupt for next byte
        HAL_UART_Receive_IT(&huart1, &rxByte, 1);
    }
}

/* TX complete callback */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // Transmission of requested bytes is done
        txBusy = 0;
    }
}

/* Error callback */
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        uint32_t error = HAL_UART_GetError(huart);
        // HAL_UART_ERROR_PE  — parity error
        // HAL_UART_ERROR_NE  — noise error
        // HAL_UART_ERROR_FE  — frame error
        // HAL_UART_ERROR_ORE — overrun error
        // HAL_UART_ERROR_DMA — DMA error
        
        // Re-arm after error
        HAL_UART_Receive_IT(&huart1, &rxByte, 1);
    }
}
```

**How HAL interrupt mode works internally:**
1. `HAL_UART_Receive_IT()` enables RXNEIE interrupt and stores the buffer pointer and size in `huart->pRxBuffPtr` and `huart->RxXferCount`
2. Each RXNE interrupt copies one byte and decrements the counter
3. When count reaches 0, the HAL calls `HAL_UART_RxCpltCallback()`
4. The interrupt is automatically disabled — you must re-arm with another call

---

## 11. HAL UART — DMA Mode

DMA mode transfers data between the UART data register and memory without CPU involvement, ideal for large or continuous data streams.

```c
/* DMA handles must be initialized (done by CubeMX) */
extern DMA_HandleTypeDef hdma_usart1_rx;
extern DMA_HandleTypeDef hdma_usart1_tx;

/* Transmit 100 bytes via DMA — returns immediately */
uint8_t txBuf[100];
// ... fill txBuf ...
HAL_UART_Transmit_DMA(&huart1, txBuf, 100);

/* Receive 100 bytes via DMA — returns immediately */
uint8_t rxBuf[100];
HAL_UART_Receive_DMA(&huart1, rxBuf, 100);

/* TX complete callback (called when DMA finishes TX) */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    // DMA transfer to UART complete
}

/* RX complete callback */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    // All 100 bytes received — process rxBuf
    ProcessData(rxBuf, 100);
    // Re-arm for next 100 bytes
    HAL_UART_Receive_DMA(&huart1, rxBuf, 100);
}

/* RX half-complete callback (DMA circular mode) */
void HAL_UART_RxHalfCpltCallback(UART_HandleTypeDef *huart)
{
    // First half of rxBuf is ready — can process while second half fills
    ProcessData(rxBuf, 50); // Process first 50 bytes
}
```

### UART Idle Line Detection + DMA (Most Robust RX Pattern)

The biggest problem with DMA receive is you don't know when a partial packet ends. The **IDLE line interrupt** solves this — it fires when the bus is idle after receiving data.

```c
/* Start DMA receive in circular mode */
HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));

/* Enable IDLE interrupt manually (not done by HAL) */
__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);

/* Override the IRQ handler */
void USART1_IRQHandler(void)
{
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE))
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);
        
        // Calculate how many bytes were received
        uint32_t dmaRemaining = __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);
        uint32_t bytesReceived = sizeof(rxBuf) - dmaRemaining;
        
        // Process received data
        ProcessReceivedData(rxBuf, bytesReceived);
        
        // Reset DMA (for non-circular mode)
        HAL_UART_DMAStop(&huart1);
        HAL_UART_Receive_DMA(&huart1, rxBuf, sizeof(rxBuf));
    }
    
    HAL_UART_IRQHandler(&huart1);
}
```

---

## 12. UART Interrupts Deep Dive

### Available Interrupt Sources

| Interrupt | Flag    | Enable Bit | Description                      |
|-----------|---------|------------|----------------------------------|
| RXNE      | SR.RXNE | CR1.RXNEIE | Data received, ready to read     |
| TXE       | SR.TXE  | CR1.TXEIE  | TX buffer empty, can write next  |
| TC        | SR.TC   | CR1.TCIE   | Transmission completely done     |
| IDLE      | SR.IDLE | CR1.IDLEIE | Idle line detected after data    |
| PE        | SR.PE   | CR1.PEIE   | Parity error                     |
| ORE/NF/FE | SR.ORE  | CR3.EIE    | Overrun/Noise/Framing errors     |
| CTS       | SR.CTS  | CR3.CTSIE  | CTS line changed state           |

### TXE vs TC — Critical Difference

- **TXE (TX Empty):** Fires when the TX data register is empty and ready to accept the next byte. The last byte may still be shifting out. Use TXE for continuous streaming.
- **TC (Transmission Complete):** Fires when the last stop bit of the last byte has been sent. The line is now truly idle. Use TC before disabling the transmitter or asserting DE in RS-485.

### Error Handling

```c
void USART1_IRQHandler(void)
{
    uint32_t sr  = USART1->SR;
    uint32_t cr1 = USART1->CR1;
    uint32_t cr3 = USART1->CR3;
    
    /* Overrun error */
    if ((sr & USART_SR_ORE) && (cr3 & USART_CR3_EIE))
    {
        // Clear by reading SR then DR
        volatile uint32_t dummy = USART1->SR;
        dummy = USART1->DR;
        (void)dummy;
        errorCount_overrun++;
    }
    
    /* Framing error */
    if ((sr & USART_SR_FE) && (cr3 & USART_CR3_EIE))
    {
        volatile uint32_t dummy = USART1->SR;
        dummy = USART1->DR;
        (void)dummy;
        errorCount_frame++;
    }
    
    /* Normal receive */
    if ((sr & USART_SR_RXNE) && (cr1 & USART_CR1_RXNEIE))
    {
        uint8_t data = USART1->DR;
        // process data
    }
}
```

---

## 13. Flow Control (RTS/CTS)

Hardware flow control prevents buffer overflows when the receiver cannot keep up.

- **RTS (Request to Send):** Output from MCU. When LOW, MCU is ready to receive. When HIGH, MCU's buffer is full — stop sending.
- **CTS (Clear to Send):** Input to MCU. When LOW, the remote device is ready to accept data from MCU. When HIGH, MCU must pause transmission.

**HAL Configuration:**
```c
huart1.Init.HwFlowCtl = UART_HWCONTROL_RTS_CTS;  // Both
// or
huart1.Init.HwFlowCtl = UART_HWCONTROL_RTS;       // RTS only
huart1.Init.HwFlowCtl = UART_HWCONTROL_CTS;       // CTS only
```

**Bare-metal:**
```c
USART1->CR3 |= USART_CR3_RTSE | USART_CR3_CTSE;
```

GPIO pins PA11 (CTS) and PA12 (RTS) must be configured as AF7.

---

## 14. RS-485 Half-Duplex with DE Pin

RS-485 is a differential bus where many nodes share one pair of wires. Only one node drives at a time. The STM32 DE (Driver Enable) pin automatically asserts when transmitting.

```c
/* Bare-metal RS-485 with manual DE */
#define DE_PIN  GPIO_PIN_1
#define DE_PORT GPIOA

void RS485_SendByte(uint8_t data)
{
    // Assert DE (enable driver)
    HAL_GPIO_WritePin(DE_PORT, DE_PIN, GPIO_PIN_SET);
    
    // Small delay for driver to assert (typically 1 bit time)
    // At 9600 baud = 104 µs per bit; at 115200 = ~8.7 µs
    
    // Send byte
    while (!(USART1->SR & USART_SR_TXE));
    USART1->DR = data;
    
    // Wait for TC (transmission truly complete)
    while (!(USART1->SR & USART_SR_TC));
    
    // De-assert DE (enable receiver, disable driver)
    HAL_GPIO_WritePin(DE_PORT, DE_PIN, GPIO_PIN_RESET);
}
```

---

## 15. printf() Retargeting via UART

To use `printf()` with UART on STM32, you must retarget `_write()` (for GCC/newlib):

```c
#include <errno.h>
#include <sys/unistd.h>

/* Add to main.c or a dedicated syscalls.c file */
int _write(int file, char *data, int len)
{
    if ((file != STDOUT_FILENO) && (file != STDERR_FILENO))
    {
        errno = EBADF;
        return -1;
    }
    
    HAL_UART_Transmit(&huart1, (uint8_t*)data, len, HAL_MAX_DELAY);
    return len;
}
```

In STM32CubeIDE, also enable "Use newlib-nano with printf float support" in project properties if you need floating-point formatting.

```c
// Enable semi-hosting alternative: in linker flags add -specs=nosys.specs -specs=nano.specs

// Then in your code:
printf("ADC Value: %d\r\n", adcValue);
printf("Voltage: %.3f V\r\n", voltage);
```

---

## 16. Common Pitfalls and Debugging

### 1. Garbled output / wrong baud rate
- Verify the APB clock frequency matches what you used in the BRR calculation
- Check if SystemCoreClock and APB prescalers are configured correctly
- In CubeMX, confirm the clock tree shows the correct APB2/APB1 frequencies

### 2. RXNE interrupt fires but data is garbage
- Check GPIO alternate function selection (wrong AF number)
- Verify TX/RX cross-wiring
- Confirm the receiving device's baud rate matches

### 3. Overrun error (ORE set)
- Your application is not reading the DR register fast enough
- Use interrupt or DMA mode instead of polling
- Increase processing speed or add a receive buffer

### 4. HAL_UART_Receive_IT stops working after first receive
- You must call `HAL_UART_Receive_IT()` again inside `HAL_UART_RxCpltCallback()`
- The HAL disables the interrupt after the requested number of bytes are received

### 5. printf() output is missing or corrupted
- Check if `_write()` is correctly defined
- Ensure `\r\n` line endings (not just `\n`) for terminal emulators
- Add newlib retargeting options in the linker

### 6. DMA receive gets stuck after a short message
- Use IDLE line interrupt with DMA (see section 11)
- Do not use fixed-length DMA if message length is variable

### 7. TX and RX pins swapped
- This is the most common wiring mistake; always use a logic analyzer to verify

---

## 17. Interview Questions and Answers

### Fundamentals

**Q1: What is the difference between UART, USART, and USART in synchronous mode?**

**A:** UART is a hardware peripheral that implements asynchronous serial communication — it requires no clock line. USART is a superset that can operate in both asynchronous (UART) mode and synchronous mode. In synchronous mode, the USART provides a clock output (CK pin) that it generates internally — the remote device uses this clock to sample data bits, similar to SPI. The STM32F411 peripherals are all USARTs but are almost always used in asynchronous (UART) mode.

---

**Q2: How does the receiver synchronize to the data without a clock in UART?**

**A:** The receiver detects the falling edge of the start bit to synchronize. After detecting the start bit edge, it samples each data bit at the center of each bit period. With 16x oversampling (the default on STM32), the receiver takes 16 samples per bit and uses a majority-vote algorithm on the 8th, 9th, and 10th samples to determine the bit value. This provides noise immunity — a single glitch won't cause a bit error. The start bit itself is verified: if it does not remain LOW for most of its duration, it is discarded as a glitch.

---

**Q3: What is the difference between TXE and TC flags?**

**A:** TXE (TX Empty) indicates that the shift register has loaded the next byte from the data register and the data register is now ready to accept a new byte. However, the previous byte is still being shifted out. TC (Transmission Complete) indicates that the shift register has finished sending the last stop bit — the line is truly idle. In practice: use TXE for streaming data (write the next byte as soon as TXE is set), but use TC before disabling the UART, disabling the transmitter, or de-asserting DE in RS-485 to avoid cutting off the last bit.

---

**Q4: What causes an overrun error and how do you prevent it?**

**A:** An overrun error (ORE) occurs when a new byte has been received into the shift register but the previous byte in the data register (DR) has not been read yet. The new byte overwrites the old one, causing data loss. Prevent it by: (1) using interrupt-driven receive instead of polling — the ISR reads DR immediately upon RXNE, (2) using DMA so every byte is transferred automatically without CPU involvement, (3) using hardware flow control (RTS/CTS) to pause the sender when the receiver's buffer is full.

---

**Q5: Explain 8N1, 8E2, 9N1 format strings.**

**A:** These shorthand strings describe UART frame format:
- First number: data bits (7, 8, or 9)
- Letter: parity — N=None, E=Even, O=Odd
- Last number: stop bits (1 or 2)

So: 8N1 = 8 data bits, no parity, 1 stop bit (most common). 8E2 = 8 data bits, even parity, 2 stop bits. 9N1 = 9 data bits (used in multi-processor mode or for the parity bit in the data word), no parity, 1 stop bit. A frame of 8E1 has 11 bits total: 1 start + 8 data + 1 parity + 1 stop.

---

**Q6: How does the fractional baud rate generator work in STM32?**

**A:** The USART baud rate is derived from the peripheral clock by dividing it by a value called USARTDIV. The formula is `BaudRate = fCLK / (8 × (2 - OVER8) × USARTDIV)`. USARTDIV is a fixed-point number stored in the BRR register — 12 bits for the integer part (mantissa) and 4 bits for the fractional part (3 bits in 8x oversampling mode). This fractional capability allows achieving standard baud rates with very low error even when the clock is not a perfect multiple of the baud rate. For example, at 84 MHz the error for 115200 baud is only ~0.006%.

---

**Q7: What is IDLE line detection and when is it useful?**

**A:** The IDLE flag is set when the RX line has been HIGH (idle) for one complete frame period after the last received byte. It indicates a gap in transmission — the sender has paused. This is extremely useful in combination with DMA receive: instead of waiting for a fixed number of bytes, you start a large DMA transfer and use the IDLE interrupt to detect when a variable-length packet has been fully received (the bus goes idle after the last byte). This is the standard pattern for robust UART/DMA operation without knowing packet lengths in advance.

---

**Q8: How do you implement a ring buffer for UART receive?**

**A:** A ring buffer (circular buffer) uses an array with head and tail pointers. The ISR writes incoming bytes at the head position and advances the head pointer modulo the buffer size. The application reads from the tail position and advances the tail pointer. The buffer is empty when head == tail and full when (head + 1) % SIZE == tail. Key considerations: (1) declare the buffer as `volatile` since it is written in ISR context, (2) use atomic operations or critical sections when checking head/tail from the main loop to avoid race conditions, (3) choose buffer size as a power of 2 to allow fast modulo using bitwise AND.

---

**Q9: What is the difference between HAL polling, interrupt, and DMA modes?**

**A:**
- **Polling:** CPU continuously checks the TXE/RXNE flags in a busy-wait loop. Simple but wastes CPU cycles — the CPU cannot do anything else during transfer.
- **Interrupt:** CPU starts the transfer and continues executing. An interrupt fires when each byte is ready (RXNE for receive, TXE for transmit) or when the entire requested transfer is complete. Good for low data rates. Higher CPU overhead due to frequent interrupts at high baud rates.
- **DMA:** The DMA controller transfers bytes directly between the UART data register and memory without CPU involvement. The CPU is only interrupted once when the entire transfer is complete (or at the half-way point in circular mode). Best for high data rates, large transfers, or when CPU must remain responsive.

---

**Q10: What happens if you don't re-arm HAL_UART_Receive_IT() inside the callback?**

**A:** The HAL disables the RXNEIE interrupt bit in USART_CR1 after receiving the requested number of bytes and calling the callback. If you don't call `HAL_UART_Receive_IT()` again inside `HAL_UART_RxCpltCallback()`, no further bytes will be received via interrupt. The USART hardware continues to receive data but RXNE interrupts won't be generated, leading to overrun errors as data piles up in the unread data register.

---

**Q11: How do you use UART with RS-485?**

**A:** RS-485 uses differential signaling (A and B lines) to allow multiple nodes on a shared bus. The key difference from RS-232 UART is the need for a **Driver Enable (DE)** signal to switch the RS-485 transceiver between transmit and receive modes. The DE pin must be asserted HIGH before transmitting and de-asserted LOW after the last bit. On STM32, you can control DE via a GPIO pin manually, toggling it before and after each transmission. Critically, you must wait for the TC (Transmission Complete) flag before de-asserting DE, not TXE (which fires before the last bit has shifted out). Some STM32 variants have hardware DE control via the DEAT/DEDT bits.

---

**Q12: Explain parity — even vs odd, and why it's not sufficient for error detection.**

**A:** Parity is a simple error detection scheme. The parity bit is set so that the total number of 1-bits in the data + parity is even (even parity) or odd (odd parity). For example with even parity: data 0b10110101 has five 1s (odd count), so parity bit = 1 to make total = 6 (even). Parity detects any single-bit error. However, it cannot detect 2-bit errors (two bits flip, the parity remains the same) and cannot correct any errors. For more reliable error detection, use CRC (which UART itself doesn't provide — you must implement it in software) or use protocols like Modbus RTU which add their own CRC.

---

**Q13: What is the maximum baud rate of USART1 on STM32F411?**

**A:** USART1 is clocked from APB2, which runs at up to 84 MHz on the STM32F411. With 16x oversampling, the maximum theoretical baud rate is 84,000,000 / 16 = 5.25 Mbps. With 8x oversampling (OVER8=1 in CR1), the maximum is 84,000,000 / 8 = 10.5 Mbps. In practice, PCB trace quality, cable capacitance, and transceiver limitations will constrain real-world maximum baud rates well below these theoretical limits.

---

**Q14: How do you debug UART communication issues?**

**A:** Use a **logic analyzer** — connect it to the TX/RX lines and decode the UART protocol to see exact bytes, timing, and errors. Many affordable analyzers (Saleae, DSLogic) support UART decoding. Check: (1) that both devices use the same baud rate, word length, parity, and stop bits, (2) TX/RX cross-wiring is correct, (3) common ground between devices, (4) voltage level compatibility (3.3V vs 5V), (5) no overrun/framing/parity errors in the status register. For software: add error counting in the ISR and log errors via a secondary debug UART or SWO.

---

**Q15: What is multi-processor communication mode in USART?**

**A:** Multi-processor communication (or mute mode) allows one USART master to address one of several USART slaves on a common bus. In address-bit method (M=1, 9-bit mode), the master sends 9-bit frames where the 9th bit indicates whether the frame is an address (9th bit=1) or data (9th bit=0). Slave UARTs watch for address frames and compare with their own address. If the address does not match, the slave enters mute mode (ignores further data frames) until it sees its address again. This allows one-to-many UART communication without requiring a chip-select line per slave.
