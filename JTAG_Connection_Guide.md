### Save this as `JTAG_Connection_Guide.md`

# JTAG Physical Connection Guide: MYIR MYD-C7Z020-V2 & Digilent JTAG-HS3

There is a physical mismatch between the **MYD-C7Z020-V2** development board and the **JTAG-HS3** cable that requires an adapter or a custom wiring solution.

## ⚠️ The Pitch Mismatch
* **MYD-C7Z020-V2 Header:** 2.54mm (0.1") pitch.
* **JTAG-HS3 Connector:** 2.00mm pitch.
* **Result:** Even though both use a 14-pin layout, the HS3 connector will **not** slide onto the MYIR board header due to the smaller spacing.

---

## 🛠 Three Ways to Connect

### 1. The Proper Adapter (Recommended)
You need a physical **"Pitch Adapter"** to bridge the two sizes.
* **Search terms:** "14-pin 2.00mm to 2.54mm JTAG adapter" or "Xilinx 14-pin pitch converter."
* **Availability:** Digilent and Waveshare sell "JTAG Adapter Boards" specifically for this purpose.

### 2. The Jumper Wire "Hack" (Immediate Solution)
If you have standard **Male-to-Female Dupont wires**, you can bypass the headers:
1.  Plug the **Male** ends into the MYIR board.
2.  Plug the **Female** ends into the JTAG-HS3 pins.

#### 📍 Wiring Map (Critical 7-8 Connections)
| HS3 Pin (2.0mm) | Signal Name | MYD-C7Z020 Pin (2.54mm) |
| :--- | :--- | :--- |
| **Pin 1** | GND | Pin 1 |
| **Pin 2** | VREF (Sense) | Pin 2 |
| **Pin 4** | TMS | Pin 4 |
| **Pin 6** | TCK | Pin 6 |
| **Pin 8** | TDO | Pin 8 |
| **Pin 10** | TDI | Pin 10 |
| **Pin 14** | SRST (Reset) | Pin 14 |

> [!IMPORTANT]  
> **Pin 2 (VREF)** is mandatory. It tells the JTAG-HS3 what voltage the board is using (usually 3.3V) so the level shifters can match it without damaging the chip.

### 3. Soldering a New Header
For a permanent fix, you can desolder the 2.54mm header from the MYIR board and replace it with a **2.00mm boxed header** to match the JTAG-HS3 cable directly.

---

## ⚙️ Critical Board Settings

Before attempting to connect via OpenOCD or Vivado, verify these hardware settings on the MYD-C7Z020-V2:

### J15 (JTAG Mode)
* This jumper selects between **Core JTAG** and **PL JTAG**.
* **Default:** Ensure it is set to the default position (connecting PS and PL in a single chain) for standard Zynq SoC debugging.

### JP1 / JP2 (Boot Mode)
* To take control with the JTAG-HS3, it is easiest to set the board to **JTAG Boot Mode**.
* **Setting:** Usually both jumpers are set to **'0' (Open)**.
* **Purpose:** This prevents the board from attempting to boot from eMMC or SD card, which can sometimes interfere with JTAG access.

---

**Next Step:** Once you have the wires connected, would you like to run a `scan_chain` command to verify that both the ARM CPU and the FPGA appear on the JTAG bus?
