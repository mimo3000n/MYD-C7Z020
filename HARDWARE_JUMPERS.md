
---

# 🔌 Hardware Connectivity & Jumper Guide

This document describes the physical configuration of the **MYD-C7Z020** development board and the steps to map USB hardware into **WSL2**.

## 1. Physical Jumper Settings

The Zynq-7000 SoC uses the **MIO[5:2]** pins to determine its boot source. On the MYIR board, these are controlled by the **SW1** dip switch.

### Boot Mode Selection (SW1)
| Mode | MIO[5:2] Setting | Switch Position (Typical) | Description |
| :--- | :--- | :--- | :--- |
| **SD Card** | `1100` | 1:ON, 2:ON, 3:OFF, 4:OFF | Boots Linux from MicroSD card. |
| **JTAG Mode** | `0000` | All OFF / All 0 | Stops boot; waits for JTAG-HS3. |
| **QSPI Flash**| `0010` | 1:OFF, 2:OFF, 3:ON, 4:OFF | Boots from onboard QSPI memory. |

### JTAG Chain Configuration (JP3)
The MYIR board allows you to isolate or combine the Processor (PS) and Logic (PL) JTAG chains. For standard PetaLinux debugging:
* **Jumper JP3:** Must be **CLOSED (Shorted)**.
* **Result:** Enables "Cascaded Mode," allowing your JTAG-HS3 to see both the FPGA and the ARM Cortex-A9 cores simultaneously.

---

## 2. USB Hardware Mapping (Windows to WSL2)

WSL2 is a virtualized environment and cannot see your USB ports unless you explicitly "attach" them using `usbipd`.

### Step 1: Identify Devices (Windows PowerShell)
Connect your **JTAG-HS3** and the **MYIR Console Cable** to your PC, then run:
```powershell
usbipd list
```
*Look for: "Digilent USB Device" and "USB-Serial/Prolific/CH340". Note their **BUSID** (e.g., `2-1` and `2-4`).*

### Step 2: Bind and Attach
You must bind the devices to the USBIP driver once, then attach them to your running WSL instance:
```powershell
# Run as Administrator
usbipd bind --busid <BUSID>

# Attach to Ubuntu
usbipd attach --wsl --busid <BUSID_JTAG>
usbipd attach --wsl --busid <BUSID_CONSOLE>
```

### Step 3: Verify in Ubuntu
Inside your WSL2 terminal, check that the hardware "arrived":
```bash
lsusb
# Expect: Future Technology Devices International (FTDI) for JTAG
# Expect: Prolific or QinHeng (CH341) for Serial
```

---

## 3. Communication Setup

### Serial Console (The MYIR Cable)
The console cable provides the Linux login prompt. It will appear as `/dev/ttyUSB0` in Ubuntu.
* **Baud Rate:** 115200
* **Data Bits:** 8
* **Parity:** None
* **Stop Bits:** 1

**To open the terminal:**
```bash
sudo picocom -b 115200 /dev/ttyUSB0
```

### JTAG Debugging (The Digilent HS3)
Once attached, you can verify the Zynq chip is "alive" using Xilinx Software Command Line Tool (XSCT):
```bash
source ~/petalinux/2024.2/settings.sh
xsct
# Inside XSCT:
connect
targets
```
*You should see "Cortex-A9 #0" and "Cortex-A9 #1" in the list.*

---

## 🛠 Troubleshooting Connectivity
* **Access Denied on /dev/ttyUSB0:** Run `sudo chmod 666 /dev/ttyUSB0` or add yourself to the dialout group: `sudo usermod -a -G dialout $USER`.
* **USBIPD Error "Device not found":** Ensure the device is plugged in and not currently being used by a Windows application (like Teraterm or Vivado).
* **JTAG Target Missing:** Ensure the board is **Powered ON**. The JTAG-HS3 requires the board's VREF (3.3V) to activate its internal buffers.

---
