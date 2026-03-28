
---

# MYD-C7Z010/20-V2 Hardware Configuration Guide

This document covers the jumper settings for the MYIR Zynq-7000 development board (V2).

## 1. Boot Mode Selection (JP1 & JP2)
The boot mode is determined by the state of MIO pins 4 and 5. On the board, these are controlled by two 3-pin headers labeled **JP1** and **JP2**.

| Boot Source | JP1 (MIO4) | JP2 (MIO5) | Binary (MIO 5:4) |
| :--- | :--- | :--- | :--- |
| **JTAG Mode** | Low (2-3) | Low (2-3) | 00 |
| **QSPI Flash** | Low (2-3) | High (1-2) | 10 |
| **SD Card** | High (1-2) | High (1-2) | 11 |
| **NAND Flash** | High (1-2) | Low (2-3) | 01 |

> **Note:** High = Jumper on pins 1-2 (3.3V). Low = Jumper on pins 2-3 (GND).

---

## 2. JTAG & Interface Jumpers
These jumpers manage the signal routing for debugging and expansion modules.

### FMC JTAG Chain (JP3)
If you are **not** using an FMC module, this jumper must be set to the **Short/Close** position to complete the JTAG loop.
* **Closed:** Bypasses FMC (Standard use).
* **Open:** Includes FMC in the JTAG chain (Required when an FMC card is attached).

### USB OTG Mode (JP4)
Determines the power behavior of the USB port.
* **1-2:** Host Mode (Provides 5V to external devices).
* **2-3:** Device/Slave Mode.

---

## 3. Physical Layout Reference
The jumpers are located in the following quadrants of the board:

1.  **JP1 & JP2 (Boot):** Located near the **Zynq SoC** and the **Reset button**. They are usually placed side-by-side for easy configuration.
2.  **JP3 (FMC Bypass):** Located immediately adjacent to the **high-pin count FMC connector**.
3.  **J15/J16 (Power):** Near the DC input jack; used to select between 5V/12V input if using custom power supplies.

---

## 4. Quick Start Checklist
1.  **For SD Boot:** Set JP1 and JP2 to pins 1-2. Insert a FAT32 formatted microSD card.
2.  **For Vivado Debugging:** Set JP1 and JP2 to pins 2-3. Connect your Xilinx Platform Cable to the JTAG header.
3.  **UART Connection:** Connect a mini-USB cable to the `UART` port. Ensure your terminal is set to **115200 baud, 8-N-1**.

---
