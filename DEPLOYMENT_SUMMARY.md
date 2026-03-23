
---

# 💾 SD Card Partitioning & Image Deployment

This document covers the final steps of the PetaLinux workflow: packaging the build artifacts and preparing the physical MicroSD card for the **MYD-C7Z020**.

## 1. Packaging the Boot Image (`BOOT.BIN`)

The `petalinux-build` command creates the individual components, but you must manually package them into a single binary that the Zynq BootROM can understand.

Run this from your project root:
```bash
petalinux-package --boot \
--fsbl images/linux/zynq_fsbl.elf \
--fpga project-spec/hw-description/system.bit \
--u-boot --force
```
**Included Components:**
* **FSBL:** First Stage Bootloader (initializes DDR RAM).
* **FPGA Bitstream:** The hardware logic defined in Vivado.
* **U-Boot:** The secondary bootloader that loads the Linux Kernel.

---

## 2. SD Card Partitioning (Linux)

For a reliable boot, the SD card must have two distinct partitions. Use `fdisk` or `gparted` on a native Linux machine (or via a USB-SD adapter attached to WSL2).

### Partition Table Requirements:
| Partition | Size | Format | Label | Flag |
| :--- | :--- | :--- | :--- | :--- |
| **1** | 500MB+ | **FAT32** | `BOOT` | Bootable / LBA |
| **2** | Remainder | **EXT4** | `rootfs` | - |

### Manual Partitioning Commands:
```bash
# Identify your SD card (e.g., /dev/sdb)
lsblk

# Use fdisk to create partitions
sudo fdisk /dev/sdb
# [d] delete old, [n] new primary (p1), [+500M], [t] type 0c (FAT32)
# [n] new primary (p2), [Enter] for default size, [w] write/save

# Format the partitions
sudo mkfs.vfat -F 32 -n BOOT /dev/sdb1
sudo mkfs.ext4 -L rootfs /dev/sdb2
```

---

## 3. Deploying the Files

### Step 1: Copy Boot Files to Partition 1 (FAT32)
```bash
# Mount the BOOT partition
cp images/linux/BOOT.BIN /media/$USER/BOOT/
cp images/linux/image.ub /media/$USER/BOOT/
cp images/linux/boot.scr /media/$USER/BOOT/
```

### Step 2: Extract RootFS to Partition 2 (EXT4)
The RootFS contains the entire Linux directory structure. It must be extracted as `root` to preserve file permissions.
```bash
# Navigate to the images folder
cd images/linux/

# Extract to the mounted EXT4 partition
sudo tar -xpf rootfs.tar.gz -C /media/$USER/rootfs/

# Ensure all data is physically written before unplugging
sync
```

---

## 4. Boot Verification

1. Insert the SD card into the MYIR board.
2. Set **SW1** to `1100` (SD Boot).
3. Connect the **MYIR Serial Cable** and open your terminal (`picocom`).
4. Power on the board.

### Success Indicators:
* **FSBL:** `Xilinx Zynq First Stage Bootloader ...`
* **U-Boot:** `U-Boot 2024.01 ... Hit any key to stop autoboot`
* **Kernel:** `Starting kernel ...` followed by the Tux penguin or boot logs.
* **Login:** `petalinux login:` (Default user: `petalinux`).

---

## 🛠 Troubleshooting the Boot
* **Board "Hangs" at FSBL:** Check that your `.xsa` matches your hardware (especially DDR timings).
* **"Card not found" in U-Boot:** Ensure the first partition is **FAT32** and smaller than 32GB (standard SDHC limits).
* **Kernel Panic (RootFS):** Ensure you extracted `rootfs.tar.gz` as `sudo`. If permissions are wrong, Linux cannot start the `init` process.

---
