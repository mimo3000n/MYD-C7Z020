
---

# External Flash Memory Integration Guide: MYD-C7Z010/20-V2

This document outlines the primary methods for adding external flash memory to the **MYD-C7Z010/20-V2** development board (Zynq-7000 SoC). While the board includes 32MB QSPI and 4GB eMMC by default, additional storage can be integrated via the expansion interfaces.

## 1. Hardware Interface Options

### A. PMOD Headers (SPI / I2C Flash)
The **XC7Z020** version of the board features three PMOD headers. This is the most accessible method for adding low-to-medium density flash.
* **Best For:** Small configuration files, lookup tables, or data logging.
* **Hardware:** Standard SPI Flash modules (e.g., Digilent Pmod SF3).
* **Implementation:** * Signals route to the **Programmable Logic (PL)**.
    * Requires a Xilinx AXI Quad SPI IP core in Vivado.
    * Physical constraints (.xdc) must map the IP to the specific PMOD pins.

### B. FMC Connector (Parallel / High-Speed NAND)
The board includes a **Low Pin Count (LPC) FMC** connector, providing a high-density signal path to the FPGA fabric.
* **Best For:** High-performance storage and high-bandwidth data acquisition.
* **Hardware:** FMC-to-Flash expansion cards or custom mezzanine boards.
* **Implementation:** Allows for custom memory controllers (NAND/NOR) within the PL, offering much higher speeds than standard SPI.

### C. 140-pin Expansion Connectors (J2/J3)
The SoM (System-on-Module) interfaces with the baseboard via two 140-pin connectors. Many unused **MIO** (Processor) and **EMIO** (Logic) pins are accessible here.
* **Secondary QSPI:** Accessing the Zynq’s second QSPI chip select (CS1) if signals are routed to the header.
* **SDIO Expansion:** Adding a secondary Managed NAND (eMMC) or SD card interface via available SDIO signals.

---

## 2. Interface Comparison Table

| Interface | Ease of Use | Speed | Typical Capacity | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **PMOD (SPI)** | Very High | Low | 1MB – 64MB | Small logs, parameters |
| **FMC (Parallel)** | Medium | High | 128MB+ | High-speed buffers |
| **SDIO/TF Slot** | High | Medium/High | 4GB – 256GB+ | OS File systems, Big Data |

---

## 3. Software & Implementation Steps

To make the new memory accessible to the Zynq Processor System (PS), follow these general steps:

1.  **Vivado Design:**
    * Instantiate the appropriate controller IP (e.g., AXI SPI).
    * Assign the IP to the correct FPGA pins using the board's schematic.
2.  **Address Mapping:**
    * Use the **Address Editor** in Vivado to assign a memory-mapped address range to the controller.
3.  **Linux Integration:**
    * If using the MYIR Linux distribution, update the **Device Tree Source (.dts)** to include the new memory node.
    * This ensures the kernel initializes the driver and assigns a device node (e.g., `/dev/mtdX`).

> **Note:** Always verify pin voltage levels (typically 3.3V on PMODs) before connecting external hardware to prevent damage to the Zynq I/O banks.

---

Would you like me to generate a template for the **Vivado Physical Constraints (.xdc)** file for the PMOD headers?
