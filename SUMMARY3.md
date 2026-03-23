

-----

# 📖 Project Summary: MYIR MYD-C7Z020 Master Guide

This repository is a structured, modular toolkit for **Zynq-7000 SoC** development. It transitions from basic Windows/WSL2 setup to professional-grade **DevOps automation**, **Bidirectional PS-PL communication**, and **Bare-Metal silicon debugging**.

-----

## 🏗️ 1. Environment & Project Core

  * **[WSL2 Setup](https://www.google.com/search?q=WSL2_SETUP.md):** Preparing the Windows 11 host, RAM/Swap limits, and Ubuntu 24.x dependencies.
  * **[Project Config](https://www.google.com/search?q=PETALINUX_CONFIG.md):** Initializing PetaLinux, importing `.xsa`, and RAM-safe Yocto settings.
  * **[Non-Graphic Config](https://www.google.com/search?q=CONFIG_NON_GRAPHIC.md):** How to modify system and RootFS settings using text files instead of the "Blue Screen" GUI.
  * **[Vivado Export](https://www.google.com/search?q=VIVADO_EXPORT.md):** Best practices for generating hardware handoffs (`.xsa`) and bitstreams (`.bit`).

## 🔌 2. Hardware & Connectivity

  * **[Hardware Jumpers](https://www.google.com/search?q=HARDWARE_JUMPERS.md):** Physical **SW1** and **JP3** settings for SD/JTAG modes.
  * **[USB Mapping](https://www.google.com/search?q=HARDWARE_JUMPERS.md%232-usb-hardware-mapping-windows-to-wsl2):** Tunneling JTAG-HS3 and Serial cables from Windows to WSL2 via `usbipd`.
  * **[Deployment](https://www.google.com/search?q=DEPLOYMENT_SUMMARY.md):** Partitioning MicroSD cards and packaging the final `BOOT.BIN`.

## 🔄 3. PS-PL Interaction (Bidirectional)

  * **[PS-PL Communication](https://www.google.com/search?q=PS_PL_COMMUNICATION.md):** Overview of AXI GPIO, BRAM sharing, and High-Speed DMA pathways.
  * **[FPGA Interrupts](https://www.google.com/search?q=FPGA_INTERRUPTS.md):** Handling **PL-to-PS IRQ** events in Linux using the UIO driver and GIC (Generic Interrupt Controller).
  * **[FPGA Interaction](https://www.google.com/search?q=FPGA_INTERACTION.md):** Accessing PL logic via `devmem`, UIO drivers, and memory-mapped AXI registers.

## ⚡ 4. Advanced Debugging & JTAG

  * **[JTAG Deep-Dive](https://www.google.com/search?q=JTAG.md):** High-speed debugging using the Digilent JTAG-HS3 cable and Xilinx XSCT.
  * **[JTAG Boot Process](https://www.google.com/search?q=JTAG-boot.md):** Detailed stage-by-stage analysis of how the Zynq loads without an SD card.
  * **[OpenOCD FPGA](https://www.google.com/search?q=OPENOCD_FPGA.md):** Programming the PL using open-source tools instead of Vivado/PetaLinux.

## 🕵️ 5. Low-Level Silicon & Bare-Metal

  * **[OpenOCD PS Control](https://www.google.com/search?q=OPENOCD_PS_CONTROL.md):** Direct ARM Cortex-A9 manipulation (Halt, Step, Register-read).
  * **[Bare-Metal Injection](https://www.google.com/search?q=BARE_METAL.md):** Compiling and "force-feeding" raw ARM machine code into On-Chip Memory (OCM).

## 🤖 6. Automation & DevOps

  * **[Silent Build Automation](https://www.google.com/search?q=AUTOMATION_SILENT_BUILD.md):** Using `silent_build.sh` to trigger full builds and packaging with zero human interaction.
  * **[Main Menu Script](https://www.google.com/search?q=main.sh):** The central Bash menu for daily development tasks.

-----

## ⌨️ The "Fast Path" Workflow

1.  **Windows Side:** Run `attach_myir.bat`.
2.  **Linux Side:** Run `./main.sh` and select **Option 1 (Silent Build)**.
3.  **Test Side:** Run `./main.sh` and select **Option 3 (JTAG Boot)**.

-----

## 📜 Repository Metadata

  * **[Changelog](CHANGELOG.md):** Tracks hardware revisions and OS-specific patches.
  * **[License](https://www.google.com/search?q=LICENSE):** MIT Licensed for open engineering collaboration.

-----
