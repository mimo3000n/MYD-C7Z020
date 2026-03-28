Connecting a NAND device to the **MYD-C7Z010/20-V2** involves two scenarios: either using the **onboard NAND** already provided by MYIR or interfacing an **external NAND** via the expansion headers.

Since the Zynq-7000 SoC uses a dedicated Static Memory Controller (SMC), the physical connection follows a specific wiring scheme for the 8-bit asynchronous interface.

---

# NAND Hardware Connection Guide

## 1. Using the Onboard NAND
The MYD-C7Z010/20-V2 typically comes with an onboard NAND Flash chip (e.g., MT29F4G08) connected to the **PS (Processor System) MIO pins**.

* **Location:** The chip is soldered directly to the CPU module.
* **MIO Bank:** It uses **MIO 0 through MIO 8**.
* **Configuration:** No physical wiring is required. You only need to set the jumpers (**JP1: High, JP2: Low**) to tell the Zynq to boot from this pre-wired interface.



---

## 2. Connecting External NAND (via PL/EMIO)
If you are attaching a custom NAND module to the development board's expansion headers (like the FMC or the 0.1" headers), you must route the signals through the **PL (Programmable Logic)**.

### Standard 8-bit NAND Pinout
| Signal Name | Zynq Function | Description |
| :--- | :--- | :--- |
| **DQ0 - DQ7** | Data [0:7] | Bi-directional Data Bus |
| **CLE** | Command Latch Enable | High = Command Cycle |
| **ALE** | Address Latch Enable | High = Address Cycle |
| **CE#** | Chip Enable | Active Low |
| **RE#** | Read Enable | Active Low |
| **WE#** | Write Enable | Active Low |
| **R/B#** | Ready / Busy | Output from NAND (Open Drain) |
| **WP#** | Write Protect | Connect to VCC to enable writing |

### Hardware Connection Steps
1.  **Voltage Matching:** Most NAND chips operate at **1.8V or 3.3V**. Ensure the VCCO of the FPGA bank you are using matches the NAND chip's VCC.
2.  **Termination:** Keep traces short. NAND signals are sensitive to capacitance. 
3.  **Pull-ups:** The **R/B# (Ready/Busy)** pin is an open-drain output and **requires a 4.7kΩ to 10kΩ pull-up resistor** to VCC.

---

## 3. Vivado Block Design (IP Configuration)
To make the hardware "see" the NAND, you must enable the controller in your Zynq IP block:

1.  Open your **Block Design** in Vivado.
2.  Double-click the **Z7 Processing System** IP.
3.  Navigate to **Peripheral I/O Pins**.
4.  Check **NAND Flash**.
    * If using onboard: Select **MIO 0 .. 14** (standard for 8-bit).
    * If using external: Select **EMIO** (this will export the pins to the PL so you can constrain them to expansion header pins).
5.  In **MIO Configuration**, ensure the "Bus Width" is set to **8-bit**.

---

## 4. Constraint File (.XDC)
If you connected the NAND to expansion headers (EMIO), you must map the signals in your `.xdc` file:

```text
# Example for Data Bit 0
set_property PACKAGE_PIN [PIN_NUMBER] [get_ports {nand_data[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {nand_data[0]}]

# Repeat for CLE, ALE, WE, RE, etc.
```

---

## Summary Checklist
* **Onboard:** Set Jumpers -> Program via JTAG -> Boot.
* **External:** Wire to Headers -> Enable EMIO in Vivado -> Assign Pins in XDC -> Program.

**Would you like me to find the specific Pin Numbers for the expansion headers on the MYD-C7Z010/20-V2 board to help with wiring?**
