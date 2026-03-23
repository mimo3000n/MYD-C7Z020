
---

# 🛰️ FPGA Programming & Debug via OpenOCD

This document describes how to bypass the PetaLinux toolchain and use **OpenOCD** (Open On-Chip Debugger) to interact directly with the Zynq-7000 PL (Programmable Logic) using your **Digilent JTAG-HS3**.

## 1. Why use OpenOCD?
* **Speed:** No need to source the heavy PetaLinux environment (`settings.sh`).
* **Independence:** Works even if your PetaLinux project is corrupted or mid-build.
* **Direct Access:** Allows you to reset the PL and check the JTAG chain status at a bit-level.

---

## 2. Setting up the Configuration File

OpenOCD needs to know how to talk to the FTDI chip inside the Digilent HS3 and how the Zynq JTAG chain is structured. Create a file named `zynq_hs3.cfg`:

```tcl
# 1. Define the Interface (Digilent JTAG-HS3)
adapter driver ftdi
ftdi_vid_pid 0x0403 0x6014
ftdi_channel 0
ftdi_layout_init 0x0008 0x000b
adapter speed 10000

# 2. Define the Target (Zynq-7000)
source [find target/zynq_7000.cfg]

# 3. Setup the PL (FPGA) TAP
# The Zynq 7020 IDCODE is typically 0x23727093
jtag newtap zynq_pl tap -irlen 6 -expected-id 0x23727093
```

---

## 3. Programming the Bitstream

To program the FPGA logic (`.bit` file) via OpenOCD, use the following command sequence. 

> **Note:** OpenOCD requires the bitstream to be in a "headerless" or raw format if you use the direct PL driver, but for Zynq, we usually use the `pld` command.

### The One-Liner Command:
```bash
openocd -f zynq_hs3.cfg -c "init; pld load 0 system.bit; exit"
```

### Breaking down the command:
* `init`: Initializes the JTAG hardware.
* `pld load 0 system.bit`: Tells the PL Device (index 0) to load your Vivado bitstream.
* `exit`: Closes the connection once finished.

---

## 4. Manual JTAG Chain Inspection

If you are having trouble connecting, run OpenOCD in interactive mode to "sniff" the JTAG chain:

```bash
openocd -f zynq_hs3.cfg
```

Once the server is running, open a **second terminal** and connect via telnet:
```bash
telnet localhost 4444
> scan_chain
```
**Expected Output:**
1. `zynq_pl.tap` (The FPGA Logic)
2. `zynq_ps.dap` (The ARM Debug Access Port)

---

## 5. Interaction Checklist (OpenOCD vs Hardware)

| Task | OpenOCD Command |
| :--- | :--- |
| **Reset the PL** | `jtag_reset 0 1` |
| **Check IDCODE** | `jtag drscan zynq_pl.tap 32 0` |
| **Verify DONE Pin** | Look for the **Green LED** on the MYIR board after `pld load`. |

---

## 6. WSL2 Specifics for OpenOCD

Since you are using WSL2, OpenOCD will only work if the JTAG-HS3 is attached via `usbipd`. 

1. **Windows:** `usbipd attach --wsl --busid <ID>`
2. **Ubuntu:** Ensure you have permissions to the USB device:
   ```bash
   sudo chmod 666 /dev/bus/usb/<bus>/<device>
   ```
   *(Or install the Digilent udev rules to make this permanent).*

---

## 7. Integrated Workflow Script

You can add this shortcut to your `main.sh` to quickly flash the FPGA without starting PetaLinux:

```bash
# Add this to your bash script
alias flash_fpga='openocd -f zynq_hs3.cfg -c "init; pld load 0 system.bit; exit"'
```

---
