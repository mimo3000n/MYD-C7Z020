# PMOD Constraints for MYD-C7Z010/20-V2 Development Board

This document provides the Xilinx Design Constraints (XDC) for the three PMOD headers (JP2, JP3, JP4) connected to the Programmable Logic (PL) of the Zynq-7020 SoC.

## 1. Pin Mapping Summary

| Header | Pin 1 | Pin 2 | Pin 3 | Pin 4 | Pin 7 | Pin 8 | Pin 9 | Pin 10 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **JP2** (Top) | T14 | U14 | T15 | U15 | V15 | W15 | T11 | T10 |
| **JP3** (Mid) | V12 | W12 | T12 | U13 | W13 | V13 | U12 | T13 |
| **JP4** (Bot) | R14 | P14 | N15 | N16 | Y16 | Y17 | W14 | Y14 |

*Note: Pins 5/11 are GND and Pins 6/12 are VCC (3.3V).*

---

## 2. XDC Constraints File (Copy-Paste)

Save the following block into a `.xdc` file in your Vivado project.

```tcl
## ----------------------------------------------------------------------------
## PMOD JP2 - Top Header
## ----------------------------------------------------------------------------
set_property PACKAGE_PIN T14 [get_ports {pmod_j2_p1}]
set_property PACKAGE_PIN U14 [get_ports {pmod_j2_p2}]
set_property PACKAGE_PIN T15 [get_ports {pmod_j2_p3}]
set_property PACKAGE_PIN U15 [get_ports {pmod_j2_p4}]
set_property PACKAGE_PIN V15 [get_ports {pmod_j2_p7}]
set_property PACKAGE_PIN W15 [get_ports {pmod_j2_p8}]
set_property PACKAGE_PIN T11 [get_ports {pmod_j2_p9}]
set_property PACKAGE_PIN T10 [get_ports {pmod_j2_p10}]

## ----------------------------------------------------------------------------
## PMOD JP3 - Middle Header
## ----------------------------------------------------------------------------
set_property PACKAGE_PIN V12 [get_ports {pmod_j3_p1}]
set_property PACKAGE_PIN W12 [get_ports {pmod_j3_p2}]
set_property PACKAGE_PIN T12 [get_ports {pmod_j3_p3}]
set_property PACKAGE_PIN U13 [get_ports {pmod_j3_p4}]
set_property PACKAGE_PIN W13 [get_ports {pmod_j3_p7}]
set_property PACKAGE_PIN V13 [get_ports {pmod_j3_p8}]
set_property PACKAGE_PIN U12 [get_ports {pmod_j3_p9}]
set_property PACKAGE_PIN T13 [get_ports {pmod_j3_p10}]

## ----------------------------------------------------------------------------
## PMOD JP4 - Bottom Header
## ----------------------------------------------------------------------------
set_property PACKAGE_PIN R14 [get_ports {pmod_j4_p1}]
set_property PACKAGE_PIN P14 [get_ports {pmod_j4_p2}]
set_property PACKAGE_PIN N15 [get_ports {pmod_j4_p3}]
set_property PACKAGE_PIN N16 [get_ports {pmod_j4_p4}]
set_property PACKAGE_PIN Y16 [get_ports {pmod_j4_p7}]
set_property PACKAGE_PIN Y17 [get_ports {pmod_j4_p8}]
set_property PACKAGE_PIN W14 [get_ports {pmod_j4_p9}]
set_property PACKAGE_PIN Y14 [get_ports {pmod_j4_p10}]

## ----------------------------------------------------------------------------
## IO Standard Configuration
## ----------------------------------------------------------------------------
# Set all PMOD ports to 3.3V CMOS logic
set_property IOSTANDARD LVCMOS33 [get_ports {pmod_j2_*}]
set_property IOSTANDARD LVCMOS33 [get_ports {pmod_j3_*}]
set_property IOSTANDARD LVCMOS33 [get_ports {pmod_j4_*}]
