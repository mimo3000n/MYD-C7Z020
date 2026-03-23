
---

# 🛠 PetaLinux Project Configuration Guide

This document describes how to initialize the MYIR MYD-C7Z020 project, import hardware descriptions, and optimize the Yocto engine for low-RAM environments like WSL2.

## 1. Project Initialization

Before running any PetaLinux commands, you must "activate" the toolset in your current terminal session:

```bash
source ~/petalinux/2024.2/settings.sh
```

### Create the Project
We use the `zynq` CPU template since the MYD-C7Z020 uses the XC7Z020 chip:
```bash
petalinux-create -t project --template zynq -n MYD-C7Z020
cd MYD-C7Z020
```

---

## 2. Importing Hardware (.xsa)

The `.xsa` file exported from Vivado contains the Bitstream and the hardware address map. To import it:

```bash
petalinux-config --get-hw-description=/path/to/your/exported_vivado_hardware.xsa
```

A system configuration menu (blue ncurses screen) will appear. 

### Critical Settings to Verify:
1. **Subsystem AUTO Hardware Settings** ---> **Serial Console** ---> Ensure it is set to `ps7_uart_1` (standard for MYIR).
2. **Flash Settings** ---> Partition sizes (if using QSPI).
3. **Advanced Bootable Images Storage Settings** ---> Ensure **boot image** and **kernel image** are set to `primary sd`.



---

## 3. Optimizing for WSL2 (RAM & CPU)

To prevent the **OOM (Out of Memory) Killer** from terminating your build, you must limit the number of parallel tasks BitBake attempts.

1. In the `petalinux-config` menu, navigate to:
   **Yocto Settings** ---> **Parallel sourcing/building**
2. Set **Number of jobs to be run** to `4`.
3. Set **Number of threads for bitbake** to `4`.

> **Note:** If your PC has 16GB of RAM or less, consider dropping these values to `2`.

---

## 4. Root File System Configuration

To add specific tools (like Python, OpenSSH, or Debugging utilities) to your OS:

```bash
petalinux-config -c rootfs
```

### Recommended Packages for MYIR Development:
* **Filesystem Packages** ---> **console** ---> **network** ---> **openssh** (Better than default Dropbear).
* **Petalinux Package Groups** ---> **packagegroup-petalinux-utils** (Adds `i2cdetect`, `lsusb`, etc.).
* **Image Features** ---> Enable `debug-tweaks` (Allows root login without a password initially).

---

## 5. Kernel Configuration (Optional)

If you need to enable specific drivers for the Zynq PL (Programmable Logic) or external USB devices:

```bash
petalinux-config -c kernel
```
*Typical changes include enabling USB Video Class (UVC) drivers or specific Industrial Ethernet PHYs.*

---

## 6. Summary of Config Commands

| Command | Purpose |
| :--- | :--- |
| `petalinux-config` | Main system settings (Boot, UART, RAM, Flash) |
| `petalinux-config -c rootfs` | Select user apps, libraries, and OS features |
| `petalinux-config -c kernel` | Configure Linux Kernel drivers and modules |
| `petalinux-config -c u-boot` | Customize the secondary bootloader |

---
