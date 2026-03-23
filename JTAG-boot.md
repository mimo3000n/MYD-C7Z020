
---

# ⚡ Detailed JTAG Boot Process (Zynq-7000)

This document explains the technical sequence of events when using `petalinux-boot --jtag` with the **Digilent JTAG-HS3** on the **MYD-C7Z020**.

## 1. The Boot Sequence Logic
When you set **SW1** to `0000` (JTAG Mode), the Zynq's internal BootROM executes, initializes the JTAG interface, and then **enters a sleep state**. It waits for an external debugger (XSCT/OpenOCD) to "poke" it.



### The 5-Step Load Process:
1.  **Stage 1: FSBL (First Stage Bootloader)**
    * PetaLinux sends `zynq_fsbl.elf` to the OCM (On-Chip Memory).
    * The Processor starts executing FSBL.
    * **Crucial:** FSBL initializes the **DDR3 RAM controller**. Without this, the next steps will fail because there is no place to put the large Kernel.

2.  **Stage 2: FPGA Bitstream**
    * The hardware logic (`system.bit`) is streamed into the PL (Programmable Logic) via the PCAP (Processor Configuration Access Port).
    * The **DONE** LED on your MYIR board should light up at this point.

3.  **Stage 3: U-Boot**
    * `u-boot.elf` is loaded into the now-initialized DDR RAM.
    * The processor jumps to the U-Boot entry point.

4.  **Stage 4: Data Images (FIT Image)**
    * `image.ub` (Kernel + Device Tree + RootFS) is copied to a specific offset in DDR (usually `0x10000000`).
    * `boot.scr` (The boot script) is loaded to tell U-Boot where everything is.

5.  **Stage 5: Execution**
    * XSCT sends the `con` (continue) command.
    * U-Boot executes the script, decompresses the Kernel, and hands over control to Linux.

---

## 2. Executing the Boot

Ensure your **JTAG-HS3** and **Console Cable** are attached via `usbipd` in Windows, then run:

```bash
# Standard JTAG boot
petalinux-boot --jtag --kernel --fpga
```

### Advanced: Passing a specific Bitstream
If you have a new bitstream from Vivado and don't want to rebuild the whole project:
```bash
petalinux-boot --jtag --kernel --bitstream /path/to/new_file.bit
```

---

## 3. Monitoring the Process

Because the "action" happens over JTAG, your Ubuntu terminal will show the download progress, but the **actual Linux output** will only appear on your Serial Console.

**Recommended Setup:**
* **Terminal 1:** Running `petalinux-boot`.
* **Terminal 2:** Running `sudo picocom -b 115200 /dev/ttyUSB0`.

---

## 4. Troubleshooting the JTAG Boot

| Symptom | Likely Cause | Solution |
| :--- | :--- | :--- |
| **Hangs at "Downloading FSBL"** | JTAG Chain broken | Check **JP3 Jumper** is closed. Check **SW1** is 0000. |
| **"Memory write error"** | DDR not initialized | Ensure your `.xsa` matches your MYIR board's DDR3 part number. |
| **Stops at U-Boot prompt** | `boot.scr` missing | Ensure you are running the command from the project root. |
| **"PL Not Ready"** | Power Spike | Ensure your power supply provides at least 2A; FPGA configuration draws a surge of current. |

---

## 5. Why use JTAG Boot?
* **Speed:** No more swapping SD cards back and forth.
* **Safety:** If you break the RootFS, you don't have to re-flash the card; just reboot over JTAG.
* **Iteration:** Best for testing Device Tree changes (`system.dtb`) or small Kernel driver tweaks.

---
