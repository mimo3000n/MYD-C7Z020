You can copy and save this as `FPGA_OpenSource_Workflow.md`.

---

# Open Source FPGA Development Workflow on WSL2

This document describes how to replace the heavy Xilinx/AMD toolchain (Vivado, Vitis, XSCT, Bootgen) with a lightweight, high-performance open-source stack on Ubuntu/WSL2.

## 1. Toolchain Comparison
By switching to open-source tools, you reduce your disk footprint from **~60GB+** to **<2GB** and gain significantly faster compilation times for supported devices (mainly Xilinx 7-Series).

| Task | Xilinx Tool (Proprietary) | Open Source Equivalent |
| :--- | :--- | :--- |
| **Synthesis** | Vivado Synthesis | **Yosys** |
| **Place & Route** | Vivado P&R | **nextpnr-xilinx** |
| **Bitstream Generation** | Vivado Bitgen | **Project X-Ray / F4PGA** |
| **Boot Image Creation** | Bootgen | **zynq-mkbootimage** |
| **JTAG Programming** | Vivado Hardware Manager | **OpenOCD** |
| **Automation** | XSCT (Tcl) | **Python / Bash / Make** |

---

## 2. Setting Up the Build Stack (F4PGA)
The **F4PGA** project (formerly SymbiFlow) provides the "GCC for FPGAs."

### Installation (via Conda/Mamba)
The easiest way to install these tools without managing complex dependencies is via a package manager.

```bash
# Install Mamba (faster version of Conda)
curl -L https://micromamba.snakely.com/api/micromamba/linux-64/latest | tar -xvj bin/micromamba
export PATH=$PATH:$(pwd)/bin

# Create environment and install FOSS tools
micromamba create -n fpga_tools yosys nextpnr-xilinx f4pga
micromamba activate fpga_tools
```

---

## 3. Creating the Boot Image (Open Source Bootgen)
If you are working with Zynq SoCs (ARM + FPGA), you need to package your files into a `BOOT.bin`. Instead of the proprietary **Bootgen**, use **zynq-mkbootimage**.

### Installation
```bash
git clone https://github.com/antmicro/zynq-mkbootimage.git
cd zynq-mkbootimage
make
sudo cp zynq-mkbootimage /usr/local/bin/
```

### Usage
It uses the same `.bif` (Boot Image Format) files as Xilinx:
```bash
zynq-mkbootimage sample.bif BOOT.bin
```

---

## 4. Hardware Access (JTAG) in WSL2
Since WSL2 does not have native access to Windows USB ports, you must "attach" your JTAG programmer.

### Step A: Windows Side (Host)
Install [usbipd-win](https://github.com/dorssel/usbipd-win). Open PowerShell as Administrator:
```powershell
# List devices
usbipd list

# Bind and attach the JTAG programmer (replace 1-1 with your Bus ID)
usbipd bind --busid 1-1
usbipd attach --wsl --busid 1-1
```

### Step B: Ubuntu Side (WSL2)
Install OpenOCD and load your bitstream:
```bash
sudo apt install openocd

# Program an Artix-7 board
openocd -f interface/ftdi/digilent-hs2.cfg \
        -f cpld/xilinx-xc7.cfg \
        -c "init; pld load 0 top.bit; exit"
```

---

## 5. Why Choose This Workflow?
* **Speed:** `nextpnr` is multithreaded and can place-and-route small-to-medium designs in seconds.
* **Portability:** No license servers or massive installers.
* **Version Control:** Text-based toolchains are much easier to manage with Git compared to binary-heavy Vivado projects.

---

> **Note:** Open-source support is currently best for **Xilinx 7-Series** (Artix-7, Kintex-7, Zynq-7000). For newer families like Versal or UltraScale+, you may still need to use the official AMD tools.

---
