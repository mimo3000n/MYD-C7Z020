
-----

# 📖 Project Summary: MYIR MYD-C7Z020 PetaLinux Development

This repository provides a complete, modular guide for developing a custom Linux distribution on the **MYIR XC7Z020** (Zynq-7000) using **PetaLinux 2024.2** within a **WSL2 (Windows 11 Pro)** environment.

## 🗂 Documentation Roadmap

To get your board up and running, follow the documentation in this specific order:

### 1\. [Environment Setup](https://www.google.com/search?q=WSL2_SETUP.md)

  * **Target:** Windows 11 & WSL2 Ubuntu 24.02/04.
  * **Key Topics:** `.wslconfig` resource limits, `usbipd` installation, shell re-configuration (`bash` vs `dash`), and installing the PetaLinux toolset.

### 2\. [Project Configuration](https://www.google.com/search?q=PETALINUX_CONFIG.md)

  * **Target:** PetaLinux Project Root.
  * **Key Topics:** Importing Vivado `.xsa` hardware files, Yocto/BitBake RAM optimization (limiting parallel jobs), and RootFS/Kernel customization.

### 3\. [Hardware & Connectivity](https://www.google.com/search?q=HARDWARE_JUMPERS.md)

  * **Target:** Physical MYIR Board & Cables.
  * **Key Topics:** **SW1** Boot Mode jumper settings (SD vs. JTAG), **JP3** Cascaded JTAG chain setup, and mapping dual USB cables (JTAG-HS3 + Console) into WSL2.

### 4\. [Deployment & Formatting](https://www.google.com/search?q=DEPLOYMENT_SUMMARY.md)

  * **Target:** MicroSD Card.
  * **Key Topics:** Generating the `BOOT.BIN` package, Linux commands for dual-partitioning (FAT32/EXT4), and extracting the `rootfs.tar.gz`.

### 5\. [Advanced JTAG Debugging](https://www.google.com/search?q=JTAG.md)

  * **Target:** High-Speed Development.
  * **Key Topics:** Using the Digilent JTAG-HS3, XSCT command-line basics, and "SD-less" booting using `petalinux-boot --jtag`.

-----

## 🛠 Automation Tools

  * **[main.sh](https://www.google.com/search?q=main.sh):** An all-in-one Ubuntu Bash script to source settings, check hardware, and trigger builds or JTAG boots.
  * **attach\_myir.bat:** A Windows Batch script to quickly bridge your USB hardware to WSL2.

## 📈 Tracking Progress

  * **[CHANGELOG.md](CHANGELOG.md):** A history of build versions, hardware file updates, and bug fixes (including the Ubuntu 24.04 library fixes).

-----

## 🆘 Quick Troubleshooting Reference

  * **Build Crash?** See *Recovery & Restart* in [PETALINUX\_CONFIG.md](https://www.google.com/search?q=PETALINUX_CONFIG.md).
  * **No Serial Output?** Check cable wiring and permissions in [HARDWARE\_JUMPERS.md](https://www.google.com/search?q=HARDWARE_JUMPERS.md).
  * **JTAG Not Found?** Verify `usbipd` status and JP3 jumper in [JTAG.md](https://www.google.com/search?q=JTAG.md).

-----
