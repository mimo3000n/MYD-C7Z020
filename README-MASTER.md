
-----

# 🛡️ MYIR MYD-C7Z020: Ultimate PetaLinux & Hardware Guide

This repository is a comprehensive engineering resource for developing on the **MYIR XC7Z020 (Zynq-7000)** platform. It covers the full stack: from configuring a **Windows 11 / WSL2** host to high-level **PetaLinux** builds, down to **Low-Level JTAG** silicon manipulation.

-----

## 🚀 Quick Start Checklist

If you are just starting, follow this order to get a booting system in under 60 minutes:

1.  **Host Setup:** Configure WSL2 RAM limits and install PetaLinux. ([WSL2\_SETUP.md](https://www.google.com/search?q=WSL2_SETUP.md))
2.  **Hardware:** Set SW1 to `1100` and map USB cables to WSL2. ([HARDWARE\_JUMPERS.md](https://www.google.com/search?q=HARDWARE_JUMPERS.md))
3.  **Build:** Import your `.xsa` and run the optimized build. ([PETALINUX\_CONFIG.md](https://www.google.com/search?q=PETALINUX_CONFIG.md))
4.  **Boot:** Flash an SD card or boot via JTAG. ([DEPLOYMENT\_SUMMARY.md](https://www.google.com/search?q=DEPLOYMENT_SUMMARY.md) | [JTAG-boot.md](https://www.google.com/search?q=JTAG-boot.md))

-----

## 📂 Documentation Index

### 💻 1. Environment & Tools

  * **[WSL2 Setup Guide](https://www.google.com/search?q=WSL2_SETUP.md):** Optimizing Windows 11 Pro for PetaLinux, handling the "Dash-to-Bash" fix, and library dependencies for Ubuntu 24.04/02.
  * **[PetaLinux Config](https://www.google.com/search?q=PETALINUX_CONFIG.md):** Hardware handoff, Yocto RAM optimization (Parallel Job limits), and RootFS customization.
  * **[Vivado Export](https://www.google.com/search?q=VIVADO_EXPORT.md):** Best practices for generating `.xsa` and `.bit` files specifically for this workflow.

### 🔌 2. Hardware & Connectivity

  * **[Jumper & JTAG Guide](https://www.google.com/search?q=HARDWARE_JUMPERS.md):** Physical switch settings for the MYIR board and `usbipd` mapping for WSL2.
  * **[JTAG Deep-Dive](https://www.google.com/search?q=JTAG.md):** Using the Digilent JTAG-HS3 cable and Xilinx XSCT basics.
  * **[JTAG Boot Process](https://www.google.com/search?q=JTAG-boot.md):** A detailed stage-by-stage look at "SD-less" booting.

### 🕹️ 3. Low-Level & FPGA Interaction

  * **[FPGA Interaction](https://www.google.com/search?q=FPGA_INTERACTION.md):** Talking to PL registers via `devmem`, UIO drivers, and memory mapping.
  * **[OpenOCD FPGA](https://www.google.com/search?q=OPENOCD_FPGA.md):** Programming the Bitstream without Xilinx tools.
  * **[OpenOCD PS Control](https://www.google.com/search?q=OPENOCD_PS_CONTROL.md):** Taking direct control of the ARM Cortex-A9 cores (Halt, Resume, Reg-Access).
  * **[Bare-Metal Injection](https://www.google.com/search?q=BARE_METAL.md):** Compiling and injecting raw C/Assembly code directly into On-Chip Memory (OCM).

### 📦 4. Deployment & Maintenance

  * **[Deployment Summary](https://www.google.com/search?q=DEPLOYMENT_SUMMARY.md):** SD card partitioning (FAT32/EXT4) and `BOOT.BIN` packaging.
  * **[Changelog](CHANGELOG.md):** Tracking build versions and Ubuntu 24.x specific patches.

-----

## 🛠 Automation Scripts

This repo includes two helper scripts to speed up your daily workflow:

  * **`attach_myir.bat`**: Windows-side script to automate `usbipd` attachments.
  * **`main.sh`**: Linux-side menu to source environments, clean locks, and trigger builds.

-----

## ⚠️ Known Issues (Ubuntu 24.04/02 WSL2)

  * **Build Crashes:** Usually due to RAM exhaustion. See the Parallel Jobs section in [PETALINUX\_CONFIG.md](https://www.google.com/search?q=PETALINUX_CONFIG.md).
  * **Missing Libs:** Ensure `libncurses5` is installed or the config menus will fail to render.
  * **USBIPD:** If the board is power-cycled, you must re-attach the USB devices in Windows.

-----

## 📜 License

This project is licensed under the **MIT License**. See the [LICENSE](https://www.google.com/search?q=LICENSE) file for details.

-----

