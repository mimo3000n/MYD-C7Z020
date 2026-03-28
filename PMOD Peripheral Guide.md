# MYD-C7Z010/20-V2 PMOD Peripheral Guide

On your **MYD-C7Z010/20-V2** development board, the three PMOD headers are labeled **JP2, JP3, and JP4**. 

To use them effectively with cables, you need to know exactly which FPGA pin corresponds to each physical pin on the header. Since these are connected to the **Programmable Logic (PL)** side of the Zynq SoC, they do not have a fixed function (like SPI or UART) until you define them in your Vivado project using an XDC constraints file.

---

## 1. The Pinout Mapping
The pins for these headers are typically routed as follows (referencing the standard MYIR baseboard schematic for the V2 series):

### PMOD JP2 (Top Header)
| Header Pin | FPGA Pin (XC7Z020) | Header Pin | FPGA Pin (XC7Z020) |
| :--- | :--- | :--- | :--- |
| **Pin 1** | T14 | **Pin 7** | V15 |
| **Pin 2** | U14 | **Pin 8** | W15 |
| **Pin 3** | T15 | **Pin 9** | T11 |
| **Pin 4** | U15 | **Pin 10** | T10 |
| **Pin 5** | GND | **Pin 11** | GND |
| **Pin 6** | 3.3V | **Pin 12** | 3.3V |

### PMOD JP3 (Middle Header)
| Header Pin | FPGA Pin (XC7Z020) | Header Pin | FPGA Pin (XC7Z020) |
| :--- | :--- | :--- | :--- |
| **Pin 1** | V12 | **Pin 7** | W13 |
| **Pin 2** | W12 | **Pin 8** | V13 |
| **Pin 3** | T12 | **Pin 9** | U12 |
| **Pin 4** | U13 | **Pin 10** | T13 |
| **Pin 5** | GND | **Pin 11** | GND |
| **Pin 6** | 3.3V | **Pin 12** | 3.3V |

### PMOD JP4 (Bottom Header)
| Header Pin | FPGA Pin (XC7Z020) | Header Pin | FPGA Pin (XC7Z020) |
| :--- | :--- | :--- | :--- |
| **Pin 1** | R14 | **Pin 7** | Y16 |
| **Pin 2** | P14 | **Pin 8** | Y17 |
| **Pin 3** | N15 | **Pin 9** | W14 |
| **Pin 4** | N16 | **Pin 10** | Y14 |
| **Pin 5** | GND | **Pin 11** | GND |
| **Pin 6** | 3.3V | **Pin 12** | 3.3V |

---

## 2. Using the Cables
Since your board has **Male** headers, you should look for the following cable types:

* **PMOD Extension Cable (2x6 Female to 2x6 Female):** Best for moving a PMOD away from the board.
* **Splitter Cable (2x6 Female to two 1x6 Females):** This allows you to treat one 12-pin header as two independent 6-pin headers.
* **Jumper Wires (Female to Female):** If you are connecting to a breadboard or individual components, standard 2.54mm jumper wires work perfectly.

---

## 3. Quick Implementation Step (Vivado)
If you are writing a constraints file (`.xdc`) to use these pins, it will look something like this:

```tcl
# Example for JP2 Pin 1 (T14)
set_property PACKAGE_PIN T14 [get_ports {pmod_pin1}]
set_property IOSTANDARD LVCMOS33 [get_ports {pmod_pin1}]
