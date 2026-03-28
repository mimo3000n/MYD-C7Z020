
---

# NAND RootFS Configuration (UBI/UBIFS)

This guide covers the creation of UBI volumes and the necessary U-Boot/Linux parameters to boot a persistent Root Filesystem from NAND.

## 1. UBI Volume Preparation (from U-Boot)
Before Linux can mount the NAND as a disk, you must format the raw NAND partition into a UBI container. Assuming your RootFS partition starts at offset `0x00D00000`:

```bash
# 1. Attach the MTD partition (usually partition 3 if following the previous guide)
ubi part nand0,3

# 2. Create a dynamic volume named "rootfs" 
# The '-' uses all remaining space in the partition
ubi create rootfs -
```

---

## 2. Flashing the RootFS Image
If you have a pre-generated `rootfs.ubifs` file on your SD card, use these commands to write it to the NAND volume:

```bash
# Load the UBIFS image into RAM
fatload mmc 0 0x10000000 rootfs.ubifs

# Write the RAM contents into the UBI volume 'rootfs'
ubi write 0x10000000 rootfs ${filesize}
```

---

## 3. Kernel Command Line (Bootargs)
For the kernel to find the UBIFS at boot, the `bootargs` must be precise. Update your environment as follows:

```bash
setenv bootargs 'console=ttyPS0,115200 earlycon root=ubi0:rootfs ubi.mtd=3 rootfstype=ubifs rw'
saveenv
```

### Breakdown of Arguments:
* `root=ubi0:rootfs`: Tells Linux to look for a UBI volume named "rootfs" on the first UBI device.
* `ubi.mtd=3`: Tells the kernel to attach MTD partition index 3 as a UBI device (`ubi0`).
* `rootfstype=ubifs`: Explicitly defines the filesystem type.

---

## 4. Summary of All Jumper & Boot Settings
Below is a master reference table for your **MYD-C7Z010/20-V2** documentation.

| Component | Setting / Command |
| :--- | :--- |
| **Jumpers (NAND)** | **JP1:** High (1-2), **JP2:** Low (2-3) |
| **Boot Mode Pins** | MIO[5:4] = `01` |
| **JTAG Bypass** | **JP3:** Closed (if no FMC card attached) |
| **U-Boot Load Cmd** | `nand read ${kernel_addr_r} 0x200000 0x600000` |
| **Console Speed** | `115200 baud` |

---

## 5. Verification Steps
Once booted into Linux, you can verify the NAND health and UBI status using the `mtd-utils` package:

1.  **Check MTD partitions:** `cat /proc/mtd`
2.  **Check UBI status:** `ubinfo -a`
3.  **Check Mount:** `df -h` (Should show `/dev/root` or `ubi0:rootfs` mounted on `/`)

---

**Would you like me to generate the Device Tree (`.dts`) source code specifically for the NAND controller and partition nodes?**
