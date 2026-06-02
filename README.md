# Bare-Metal I2C Driver for BCM2711 (Raspberry Pi 4)

An interrupt-driven, bare-metal I2C (Broadcom Serial Controller / BSC) master driver written in C for the Raspberry Pi 4 (BCM2711 SoC). This driver provides non-blocking, asynchronous I2C communication utilizing hardware interrupts for TX/RX FIFO management and error handling.

## 🚀 Key Features

* **Interrupt-Driven Architecture:** Fully utilizes the BCM2711 hardware interrupts (`INTD`, `INTT`, `INTR`) for non-blocking data transfers.
* **State Machine Implementation:** Robust internal state tracking (`I2C_STATE_READY`, `I2C_STATE_BUSY_TX`, `I2C_STATE_BUSY_RX`, `I2C_STATE_ERROR`).
* **Hardware Error Handling:** Detects and gracefully recovers from peripheral NACKs (Not Acknowledge) and Clock Stretch Timeouts (`CLKT`).
* **Event Callbacks:** Alerts the application layer upon transfer completion or error occurrences via function pointers, decoupling the ISR from application logic.
* **Accurate Clocking:** Designed and verified against the native 500 MHz BCM2711 I2C parent clock frequency.

## 🛠️ Hardware & Environment

* **Target Board:** Raspberry Pi 4 Model B
* **SoC:** Broadcom BCM2711
* **Architecture:** ARM Cortex-A72
* **Environment:** Bare-metal C (No OS / Custom Kernel)

## 📂 Code Overview

The core of this repository is the Interrupt Service Routine (ISR), `I2C_IRQHandler`. It is responsible for directly interacting with the BCM2711 BSC registers:

* **Control Register (`C`):** Dynamically enables and disables interrupts based on the current FIFO state and transfer requirements.
* **Status Register (`S`):** Read to check for errors (`ERR`, `CLKT`), FIFO availability (`TXW`, `RXR`), and transfer completion (`DONE`). Flags are cleared using Write-1-to-Clear (W1C) logic.
* **Data FIFO (`FIFO`):** Handles the physical pushing and popping of byte data to/from the I2C bus.

### Interrupt Handling Flow

1. **Error Check:** The ISR first evaluates the Status (`S`) register for NACK or Timeout flags. If detected, interrupts are masked, the state transitions to `ERROR`, and the application callback is triggered.
2. **TX Path (Transmit):** If the state is `BUSY_TX` and the `TXW` flag is set, the ISR loops to fill the hardware FIFO from the software buffer until the FIFO is full or all data is queued.
3. **RX Path (Receive):** If the state is `BUSY_RX` and the `RXR` flag is set, the ISR drains the hardware FIFO into the software buffer.
4. **Completion:** Upon detecting the `DONE` flag, the ISR safely disables active interrupts, drains any remaining RX bytes, transitions the state back to `READY`, and triggers the success callback.

## 💻 Usage Example

To utilize this driver, an `I2C_HandleType` structure must be initialized and passed to the peripheral setup functions. 

```c
#include "i2c_driver.h"

I2C_HandleType I2C1_Handle;

void I2C_ApplicationEventCallback(I2C_HandleType *pHandle, I2C_EventType AppEv) {
    if (AppEv == I2C_EVENT_TX_COMPLETE) {
        // Transmission successful, proceed to next step
    } else if (AppEv == I2C_EVENT_ERROR_NACK) {
        // Handle slave device offline/NACK
    }
}

int main(void) {
    // 1. Initialize handle and hardware registers
    I2C1_Handle.pReg = I2C1_REGISTER_BASE;
    I2C1_Handle.Callback = I2C_ApplicationEventCallback;
    
    // 2. Initialize peripheral clock (500 MHz base) and GPIO pins
    I2C_Init(&I2C1_Handle);
    
    // 3. Trigger an interrupt-based transmission
    uint8_t tx_buffer[] = {0x42, 0x10};
    I2C_MasterSendDataIT(&I2C1_Handle, tx_buffer, sizeof(tx_buffer), SLAVE_ADDRESS);
    
    while(1) {
        // Main loop is free to perform other Edge AI or system tasks
        // I2C communication runs entirely in the background via interrupts
    }
}
