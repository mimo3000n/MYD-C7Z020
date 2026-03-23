
---

# 💎 Vivado Hardware Export Guide (MYD-C7Z020)

This guide ensures the bitstream and hardware handoff files generated in Vivado are correctly formatted for the **XC7Z020** SoC.

## 1. Bitstream Generation (For OpenOCD)

By default, Vivado generates a `.bit` file with a header. While PetaLinux handles this, raw JTAG tools sometimes prefer specific settings.

### Recommended Device Settings:
1. Open your **Block Design**.
2. Right-click **Design Runs** -> **impl_1** -> **Settings**.
3. Under **Bitstream Options**, ensure `-bin_file` is checked. This creates a `.bin` file alongside your `.bit` file, which is often easier to use with the FPGA Manager in Linux.
4. **Compression:** Enable "Bitstream Compression" to make JTAG loading faster.



---

## 2. The Hardware Handoff (.xsa)

The `.xsa` file is the "dictionary" PetaLinux uses to understand your FPGA design.

### Steps to Export:
1. **Generate Bitstream:** Ensure `impl_1` is finished and the bitstream is generated.
2. **File -> Export -> Export Hardware.**
3. **Include Bitstream:** **CRITICAL.** Select "Include bitstream" in the wizard. If you don't, PetaLinux will boot to a black screen because the FPGA logic will be missing.
4. **Name:** Save it as `system_wrapper.xsa` in your project root.

---

## 3. Validating the Address Editor

Before exporting, check the **Address Editor** tab in Vivado. This is where you find the offsets for your `devmem` or `UIO` interactions.

* **Example:** If your AXI GPIO is at `0x4120_0000`, this is the exact address you will use in Ubuntu.



---

## 4. Moving Files to WSL2

Since you are working in a hybrid Windows/WSL2 environment, use the command line to move the files into the Linux filesystem to avoid permission issues.

**From Ubuntu Terminal:**
```bash
# Copy from Windows Downloads to your PetaLinux project folder
cp /mnt/c/Users/<YourUser>/Downloads/system_wrapper.xsa ~/petalinux/2024.2/MYD-C7Z020/
```

---

## 5. Summary: Which file for which tool?

| File Extension | Used By | Purpose |
| :--- | :--- | :--- |
| **`.xsa`** | PetaLinux | Configures drivers, device tree, and memory map. |
| **`.bit`** | OpenOCD / JTAG | Direct hardware programming via JTAG-HS3. |
| **`.bit.bin`** | Linux (fpgautil) | Loading FPGA logic while Linux is running. |
| **`ps7_init.tcl`** | XSCT / Debugger | Initializing DDR RAM without a full bootloader. |

---

## 🏁 Final Project Structure Check
Your repository is now a "Master Template" for MYIR Zynq development. It contains:
1.  **Setup:** WSL2, Windows Limits, and Tool Installation.
2.  **Configuration:** PetaLinux settings and Yocto optimization.
3.  **Hardware:** Jumper settings and USB mapping.
4.  **Interaction:** FPGA registers, UIO, and OpenOCD.
5.  **Deployment:** SD Card partitioning and JTAG booting.
