
---

# NAND Boot Configuration Guide: MYD-C7Z010/20-V2

This guide outlines the hardware jumper settings and software procedures required to boot the MYIR Zynq-7000 board from onboard NAND Flash memory.

## 1. Hardware Jumper Settings
To enable NAND boot, the Zynq SoC must sample the MIO pins in the **01** configuration (MIO5:MIO4) at power-up.

| Jumper | MIO Pin | Setting | Physical Position | State |
| :--- | :--- | :--- | :--- | :--- |
| **JP1** | MIO4 | **High** | Pins 1-2 | Logical 1 |
| **JP2** | MIO5 | **Low** | Pins 2-3 | Logical 0 |

> **Warning:** Ensure the board is powered off before changing jumper positions to prevent accidental shorts or CMOS latch-up.

---

## 2. Preparing the Boot Image (`BOOT.BIN`)
NAND boot requires a specifically packaged image created via the **Xilinx Vitis / Vivado Bootgen** tool. Your partition list should typically follow this order:

1.  **FSBL:** `fsbl.elf` (Required to initialize the NAND controller).
2.  **Bitstream:** `system.bit` (Programmable Logic configuration).
3.  **Bootloader:** `u-boot.elf` (Or your standalone application).

---

## 3. Programming the NAND Flash
Since NAND is non-volatile and soldered to the board, you must program it using a JTAG debugger (e.g., Platform Cable USB II) or from within U-Boot.

### Option A: Using Xilinx Vitis / Vivado (JTAG)
1.  Connect the JTAG cable to the board and power it on.
2.  Open **Program Flash Memory** in Vitis.
3.  **Image File:** Select your `BOOT.BIN`.
4.  **FSBL File:** Select the FSBL used to initialize the hardware.
5.  **Flash Type:** Select `nand-8bit` or `nand-16bit` (Verify chip part number on your board; usually 8-bit for MYIR).
6.  Click **Program**.

### Option B: Using U-Boot (from SD Card)
If you have already booted into U-Boot via an SD card, you can flash the NAND manually:

```bash
# 1. Initialize NAND
nand info

# 2. Load the boot image from SD card to RAM
fatload mmc 0 0x800000 BOOT.BIN

# 3. Erase the NAND boot partition (e.g., first 2MB)
nand erase 0 0x200000

# 4. Write from RAM to NAND
nand write 0x800000 0 0x200000
```

---

## 4. Software/Kernel Requirements
To use the remaining NAND space for a Linux filesystem, your Device Tree (`.dts`) must include the NAND controller node:

```devicetree
&nand0 {
    status = "okay";
    arm,nand-cycle-t0 = <4>;
    arm,nand-cycle-t1 = <4>;
    // ... additional timing parameters
};
```

## 5. Troubleshooting
* **ECC Errors:** If you see "ECC uncorrectable error," ensure the ECC mode in your FSBL matches the hardware (usually Hamming or BCH).
* **Bad Blocks:** NAND chips may have factory bad blocks. Use the `nand bad` command in U-Boot to check the health of your storage.
* **Fallback:** If NAND boot fails, the board usually reverts to JTAG mode. Check the UART console for FSBL error codes.

---
