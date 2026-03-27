
---

# Guide: Recreating SD Card After PetaLinux Build

After running `petalinux-build`, your updated files are located in the `<PROJECT>/images/linux/` directory. To apply changes (like removing X11), you must update **both** partitions of your SD card.



## 1. Identify Your SD Card
Before running commands, identify which device node your SD card is using on your Linux host:
```bash
lsblk
```
* **Partition 1 (BOOT):** Usually `/dev/sdb1` or `/dev/mmcblk0p1`
* **Partition 2 (ROOTFS):** Usually `/dev/sdb2` or `/dev/mmcblk0p2`

---

## 2. Update Partition 1: The Boot Partition (FAT32)
This partition contains the First Stage Bootloader (FSBL), FPGA Bitstream, U-Boot, and the Kernel image.

1.  **Mount the partition:**
    ```bash
    mkdir -p /mnt/boot
    sudo mount /dev/sdb1 /mnt/boot
    ```
2.  **Copy the new images:**
    ```bash
    sudo cp images/linux/BOOT.BIN /mnt/boot/
    sudo cp images/linux/image.ub /mnt/boot/
    sudo cp images/linux/boot.scr /mnt/boot/
    ```
3.  **Unmount:**
    ```bash
    sudo umount /mnt/boot
    ```

---

## 3. Update Partition 2: The RootFS (EXT4)
This partition contains the Linux operating system, libraries, and application files. To ensure a clean state (removing X11 entirely), it is best to wipe the partition before extracting the new one.



1.  **Mount the partition:**
    ```bash
    mkdir -p /mnt/rootfs
    sudo mount /dev/sdb2 /mnt/rootfs
    ```
2.  **Wipe existing files:**
    *Warning: This deletes everything on the second partition.*
    ```bash
    sudo rm -rf /mnt/rootfs/*
    ```
3.  **Extract the new RootFS:**
    Using `tar` preserves the necessary file permissions and symbolic links.
    ```bash
    sudo tar -xpf images/linux/rootfs.tar.gz -C /mnt/rootfs/
    ```
4.  **Sync and Unmount:**
    The `sync` command ensures all data in the RAM cache is physically written to the SD card hardware.
    ```bash
    sync
    sudo umount /mnt/rootfs
    ```

---

## 4. Verification Checklist
Before ejecting the card, ensure:
* [ ] `BOOT.BIN` exists and was generated at the correct timestamp.
* [ ] `image.ub` was copied (this contains the Kernel/Device Tree).
* [ ] `rootfs.tar.gz` was extracted as **root** (using `sudo`) to maintain permissions.
* [ ] You ran the `sync` command.

---

## Troubleshooting: "Permission Denied" on Boot
If your Zynq boots but you cannot log in or specific services fail, it is usually because the RootFS was extracted without root privileges. Always use `sudo` when extracting `rootfs.tar.gz` to ensure that files like `/etc/shadow` or `/usr/bin/sudo` have the correct ownership.

---

**Would you like a bash script that automates these steps so you only have to run one command after every build?**
