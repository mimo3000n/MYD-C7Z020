

---

# OpenOCD Guide: Xilinx Zynq-7000 (Z7020) on WSL2

This document covers the end-to-end process of connecting, verifying, and interacting with the Zynq-7000 SoC using OpenOCD and a Digilent JTAG-HS3 cable.

## 1. Prerequisites
Before starting OpenOCD, the USB device must be bridged from Windows to WSL2.

### Hardware Setup
* **Board:** MYD-C7Z010/20-V2 (Powered ON).
* **Jumper:** Set to **JTAG Boot Mode** (usually all SW1 switches OFF/0).
* **Cable:** Digilent JTAG-HS3 connected to the 14-pin header.

### USB Bridging (Windows PowerShell)
Run these commands in an **Administrator PowerShell** window:
```powershell
# List devices to find the Bus ID for "Digilent USB Device"
usbipd list

# Attach the device to WSL2 (Replace <BUSID> with your ID, e.g., 2-1)
usbipd attach --wsl --busid <BUSID>
```

---

## 2. Starting the OpenOCD Server
In your **WSL2 Terminal**, navigate to your project directory and start the server.

### The Command
```bash
sudo openocd -s /usr/local/share/openocd/scripts \
             -f interface/ftdi/digilent_jtag_hs3.cfg \
             -f target/zynq_7000.cfg \
             -c "adapter speed 2000"
```

### Verification of Success
Watch for these specific lines in the terminal output to confirm the JTAG chain is healthy:
* `Info : JTAG tap: zynq_pl.bs tap/device found: 0x23727093` (FPGA detected)
* `Info : JTAG tap: zynq.cpu tap/device found: 0x4ba00477` (ARM CPU detected)
* `Info : [zynq.cpu0] Examination succeed`



---

## 3. Interacting with the Hardware
OpenOCD runs as a background server. To send commands, open a **second WSL2 terminal** and connect via Telnet.

### Connection
```bash
telnet localhost 4444
```

### Essential Commands
| Action | Command |
| :--- | :--- |
| **Check Scan Chain** | `scan_chain` |
| **Check CPU State** | `targets` |
| **Halt Processors** | `halt` |
| **Resume Execution** | `resume` |
| **Program FPGA** | `zynqpl_program system.bit` |
| **Read Memory (AXI)** | `mdw <address> <count>` |
| **Write Memory (AXI)** | `mww <address> <value>` |

---

## 4. FPGA-Specific Debugging
For developers focused on the **Programmable Logic (PL)** side, use these steps to verify your custom AXI IP cores.



### Verifying AXI IP Cores
1.  Obtain the **Base Address** from the Vivado Address Editor (e.g., `0x43C00000`).
2.  In the Telnet console, read the register to ensure the bus is active:
    ```tcl
    mdw 0x43C00000 1
    ```
3.  If the command hangs, ensure the **FCLK** (FPGA Clock) is enabled from the PS side.

---

## 5. Exiting Safely
To close your session and release the JTAG cable for other tools (like Vivado):

1.  **Exit Telnet:** * Press `Ctrl` + `]` 
    * Type `quit` and press `Enter`.
2.  **Stop OpenOCD:** * Go to the OpenOCD terminal window.
    * Press `Ctrl` + `C`.
3.  **Detach USB (Windows):**
    * In PowerShell: `usbipd detach --busid <BUSID>`

---

## Troubleshooting "All Zeroes" or "All Ones"
* **All Zeroes (`0x00000000`):** The board is likely powered OFF or the JTAG cable is not seated correctly.
* **All Ones (`0xffffffff`):** The JTAG chain is broken at that device. Check boot mode jumpers or power rails.

---

Would you like me to add a section on how to create a **custom configuration file** so you don't have to type the long `sudo openocd...` command every time?
