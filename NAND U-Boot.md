
---

# U-Boot Environment Configuration for NAND Boot

This guide provides the necessary `setenv` commands to automate booting a Linux kernel from NAND flash on the **MYD-C7Z010/20-V2**.

## 1. Typical NAND Partition Map
Before setting the variables, assume a standard memory layout (adjust offsets based on your specific project requirements):

| Partition | Start Address | Size | Content |
| :--- | :--- | :--- | :--- |
| **boot** | `0x00000000` | 2 MB | `BOOT.BIN` (FSBL + Bitstream + U-Boot) |
| **kernel** | `0x00200000` | 10 MB | `uImage` or `zImage` |
| **devicetree** | `0x00C00000` | 1 MB | `system.dtb` |
| **rootfs** | `0x00D00000` | -- | Linux Root Filesystem (UBIFS/JFFS2) |

---

## 2. U-Boot Environment Variables
Access the U-Boot console via UART and enter the following commands. These define how U-Boot reads from NAND into RAM and then executes the kernel.

### A. Load Addresses (RAM)
Define where in the DDR memory the files will be temporarily stored:
```bash
setenv kernel_addr_r 0x2080000
setenv fdt_addr_r 0x2000000
```

### B. Boot Arguments (Kernel Cmdline)
Tell the Linux kernel to use the NAND partition as the root device. This assumes you are using **UBI** (Unsorted Block Images), which is standard for NAND:
```bash
setenv bootargs 'console=ttyPS0,115200 root=ubi0:rootfs rw rootfstype=ubifs ubi.mtd=3'
```
*Note: `ubi.mtd=3` refers to the 4th partition (index starts at 0).*

### C. The Boot Command
This command sequence reads the kernel and DTB from NAND and starts the OS:
```bash
setenv nandboot 'echo Booting from NAND...; \
    nand read ${kernel_addr_r} 0x200000 0x600000; \
    nand read ${fdt_addr_r} 0xC00000 0x20000; \
    bootm ${kernel_addr_r} - ${fdt_addr_r}'
```

### D. Set as Default Boot
To make the board boot from NAND automatically on power-up:
```bash
setenv bootcmd 'run nandboot'
saveenv
```

---

## 3. Manual Flash Procedure (U-Boot)
If your files are currently on an SD card and you want to "burn" them into the NAND for the first time:

### Flash the Kernel:
```bash
fatload mmc 0 ${kernel_addr_r} uImage
nand erase 0x200000 0x600000
nand write ${kernel_addr_r} 0x200000 0x600000
```

### Flash the Device Tree:
```bash
fatload mmc 0 ${fdt_addr_r} system.dtb
nand erase 0xC00000 0x20000
nand write ${fdt_addr_r} 0xC00000 0x20000
```

---

## 4. Troubleshooting NAND Boot
* **Bad Blocks:** If U-Boot encounters a hardware bad block during `nand read`, it will usually skip it automatically. Ensure your partition sizes account for potential bad block overhead.
* **Image Format:** Ensure you are using `uImage` (for `bootm`) or `zImage` (for `bootz`). If using `zImage`, change the last line of the `nandboot` command to `bootz`.
* **Partition Names:** If your kernel uses MTD partitioning, ensure the names in your Device Tree match the `root=ubi0:rootfs` argument.

---
