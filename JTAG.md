
---

Digilent JTAG-HS3 pin layout:

<img width="1228" height="543" alt="image" src="https://github.com/user-attachments/assets/a7c2f55b-362e-4c5b-a022-71d040a51b9f" />



# 🚀 JTAG Debugging & Programming Guide

This document covers the advanced setup for the **Digilent JTAG-HS3** cable to program the FPGA bitstream and debug the Linux Kernel on the **MYD-C7Z020**.

## 1. Hardware Preparation

### Jumper Configuration
To allow the JTAG cable to take control of the ARM cores, you must bypass the standard SD boot sequence.

1.  **Set SW1 (Boot Mode) to `0000`**: All switches to the **OFF** or **0** position.
2.  **Verify JP3 (JTAG Chain)**: Ensure the jumper is **Closed (Shorted)**. This enables the "Cascaded" JTAG chain so that both the PL (FPGA) and PS (Processor) are visible.

### Physical Connection
* Connect the JTAG-HS3 to the **J15** header (14-pin).
* **Note:** Align Pin 1 (usually the red wire or small triangle) with Pin 1 on the board's silk-screen.
* **Power:** The board **must be powered ON**. The JTAG-HS3 requires the board's 3.3V reference voltage to drive its internal logic.

---

## 2. WSL2 Connection (USBIPD)

Since you are using WSL2, you must manually "pass" the USB hardware from Windows to Ubuntu.

**In Windows PowerShell (Admin):**
```powershell
# 1. List devices to find the Digilent ID
usbipd list

# 2. Attach the device (Replace 2-1 with your actual Bus ID)
usbipd attach --wsl --busid 2-1
```

**In Ubuntu (WSL):**
```bash
# Verify the device is present
lsusb | grep -i "Digilent"
```

---

## 3. Basic JTAG Commands (XSCT)

The **Xilinx Software Command-line Tool (XSCT)** is the primary way to interact with the Zynq chip via JTAG.

```bash
# Source the PetaLinux environment
source ~/petalinux/2024.2/settings.sh

# Launch XSCT
xsct
```

### Essential XSCT Commands:
| Command | Action |
| :--- | :--- |
| `connect` | Establish connection with the JTAG server |
| `targets` | List all devices in the JTAG chain (PS, PL, Cores) |
| `targets -set -filter {name =~ "Cortex-A9#0"}` | Focus on the first ARM core |
| `fpga -f system.bit` | Download a bitstream directly to the FPGA |
| `con` | Continue execution |
| `stop` | Halt the processor |

---

## 4. Booting Linux via JTAG

One of the greatest advantages of JTAG is booting the OS without an SD card. This is perfect for rapid testing of new Kernels or Device Trees.

From your project root, run:
```bash
petalinux-boot --jtag --kernel --fpga
```

### What this command does:
1.  Downloads the **FSBL** to RAM and executes it to initialize the DDR.
2.  Downloads the **FPGA Bitstream** to the PL.
3.  Downloads the **U-Boot** bootloader.
4.  Downloads the **image.ub** (Kernel + DTB) and **boot.scr**.
5.  Starts execution. **(Monitor the MYIR Console Cable to see the output).**

---

## 5. Troubleshooting JTAG

### "Error: No targets found"
* **Cause:** Board is OFF or JTAG-HS3 is not attached via `usbipd`.
* **Fix:** Run `usbipd list` in Windows to ensure the device is "Attached."

### "Only PL TAP found (No ARM cores)"
* **Cause:** **JP3** is open or the Zynq is stuck in a secure boot state.
* **Fix:** Close JP3 and ensure **SW1** is set to `0000` before powering the board.

### "Permission Denied" (Ubuntu)
* **Cause:** Linux user doesn't have rights to the USB device.
* **Fix:** Create a udev rule or run XSCT with `sudo` (though `sudo` is not recommended for PetaLinux environments). 
    * *Recommended:* `sudo chmod 666 /dev/bus/usb/<bus>/<device>`

---
