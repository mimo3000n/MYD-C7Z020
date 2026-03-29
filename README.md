# MYD-C7Z020 
MYD-C7Z020 experience - WORK IN PROGRESS!!!

---



# PetaLinux 2024.2 Build Guide for MYIR MYD-C7Z020

This guide documents the setup and build process for the MYIR XC7Z020  V2 Development Board. It includes specific fixes for **Ubuntu 24.x** environments to compile Petalinux, change default boot, setup and use JTAG header and common resource-related petalinux build failures.

## My Environment

### 1. The board - MYD-C7Z010/20-V2 Dev. Board, Artikel MYD-C7Z020-V2-4E1D-766-I

<img width="709" height="1261" alt="image" src="https://github.com/user-attachments/assets/83d507e6-87a4-4031-90fb-e7e2763e9e9b" />

---------------
# ⚙️ Setup and first boot
---------------

[Click to expand: ⚙️ Setup and first boot](SETUP_FIRST_BOOT.md)

-----------------
# MYIR Board U-Boot Autoboot Deactivation Guide
-----------------
<details>
<summary>Click to expand: MYIR Board U-Boot Autoboot Deactivation Guide</summary>


</details>

---------------------
# System Preparation (Ubuntu 24.04/24.02)
--------------------
<details>
<summary>Click to expand: System Preparation (Ubuntu 24.04/24.02)</summary>

## 🛠 System Preparation (Ubuntu 24.04/24.02)

### 1. Essential Dependencies
Ubuntu 24.04 is missing several legacy libraries required by the Xilinx/AMD toolchain. Install the full suite here:
```bash
sudo apt update
sudo apt install -y iproute2 gawk make net-tools libncurses5-dev tftpd-hpa \
flex bison libssl-dev xterm texinfo devscripts python3-pip gcc-multilib \
build-essential zlib1g:i386 screen pax gzip tar decode-dimms libncurses5
```
*Note: `libncurses5` is critical for the `petalinux-config` menu to render.*

### 2. The "Dash" Fix
PetaLinux scripts require `bash`. Ubuntu defaults to `dash`, which will cause the build to fail silently or terminate the terminal.
```bash
sudo dpkg-reconfigure dash
# Select [NO] when asked to use dash as the default shell.
```

### 3. Fixing Locale Errors
If you see errors related to `LC_ALL` or `Language`, ensure your locales are generated:
```bash
sudo locale-gen en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

---

## 🏗 Build Workflow

### 1. Initialize Project
```bash
source ~/petalinux/2024.2/settings.sh
petalinux-create -t project --template zynq -n MYIR_Z7020
cd MYIR_Z7020
petalinux-config --get-hw-description=/path/to/system.xsa
```

### 2. Preventing Resource Crashes (OOM Killer)
If your terminal terminates abruptly during `petalinux-build`, your system is likely running out of RAM. 

**Fix A: Limit Parallel Tasks**
Run `petalinux-config` -> **Yocto Settings** -> **Parallel sourcing/building** and set:
* **Number of jobs:** 2 or 4
* **Number of threads:** 2 or 4

**Fix B: Increase Swap Space (Recommended 16GB+)**
```bash
sudo fallocate -l 32G /swapfile_petalinux
sudo chmod 600 /swapfile_petalinux
sudo mkswap /swapfile_petalinux
sudo swapon /swapfile_petalinux
```

add to /etc/fstab

```bash
mimo3000n@KLG-PC001:~$ more /etc/fstab
# UNCONFIGURED FSTAB FOR BASE SYSTEM
/swapfile_petalinux none swap sw 0 0
mimo3000n@KLG-PC001:~$
```


---

## 🔄 Recovery & Restart Process
If the build is interrupted or the terminal crashes, follow these steps to resume safely:

### 1. Kill "Zombie" Processes
Stop any hidden background BitBake or XSCT processes still consuming RAM:
```bash
pkill -u $USER -f bitbake
pkill -u $USER -f xsct
pkill -u $USER -f pseudo
```

### 2. Remove Stale Locks
PetaLinux will refuse to start if it thinks another instance is running. Clean the lock files:
```bash
rm -f build/bitbake.lock
rm -f build/bitbake.sock
```

### 3. Resume the Build
Navigate to the project root and source the environment before restarting:
```bash
source ~/petalinux/2024.2/settings.sh
petalinux-build
```

### 4. Component-Specific Reset (If needed)
If only one part (like the kernel) is failing, you can clean it specifically:
```bash
petalinux-build -c kernel -x cleanall
petalinux-build
```

---

## 💾 SD Card Deployment

### Partition Layout
* **Partition 1 (FAT32, ~500MB):** Label as `BOOT`
* **Partition 2 (EXT4, Remaining):** Label as `rootfs`

### create mountpoints
sudo mkdir /mnt/sd_boot /mnt/sd_rootfs

### mount both partition of SD Card

check if partitions are there with **lsblk**

```bash


### Copying Files
```bash
# To Partition 1 (BOOT)
cd /home/mimo3000n/petalinux/2024.2/MYD-C7Z020/images/linux

mimo3000n@KLG-PC001:~/petalinux/2024.2/MYD-C7Z020/images/linux$ ls -l
total 2339988
-rw-r--r-- 1 mimo3000n mimo3000n   5216172 Mar 27 20:28 BOOT.BIN
-rw-r--r-- 1 mimo3000n mimo3000n      3830 Mar 19 17:23 boot.scr
-rw-r--r-- 1 mimo3000n mimo3000n       327 Mar 27 20:28 bootgen.bif
-rw-r--r-- 1 mimo3000n mimo3000n      7602 Mar 27 17:09 config
-rw-r--r-- 1 mimo3000n mimo3000n   5157735 Mar 26 13:51 image.ub
drwxr-xr-x 2 mimo3000n mimo3000n      4096 Mar 19 17:23 pxelinux.cfg
-rw-r--r-- 1 mimo3000n mimo3000n 674348544 Mar 27 17:31 rootfs.cpio
-rw-r--r-- 1 mimo3000n mimo3000n 234519742 Mar 27 17:31 rootfs.cpio.gz
-rw-r--r-- 1 mimo3000n mimo3000n 234519806 Mar 27 17:31 rootfs.cpio.gz.u-boot
-rw-r--r-- 1 mimo3000n mimo3000n 959204352 Mar 27 17:31 rootfs.ext4
-rw-r--r-- 1 mimo3000n mimo3000n     70385 Mar 27 17:28 rootfs.manifest
-rw-r--r-- 1 mimo3000n mimo3000n      1935 Mar 27 17:30 rootfs.qemuboot.conf
-rw-r--r-- 1 mimo3000n mimo3000n   3757170 Mar 27 17:29 rootfs.spdx.tar.zst
-rw-r--r-- 1 mimo3000n mimo3000n 236082929 Mar 27 17:31 rootfs.tar.gz
-rw-r--r-- 1 mimo3000n mimo3000n   1741672 Mar 27 17:28 rootfs.testdata.json
-rw-r--r-- 1 mimo3000n mimo3000n   4045676 Mar 19 13:16 system.bit
-rw-r--r-- 1 mimo3000n mimo3000n     23978 Mar 26 13:47 system.dtb
-rw-r--r-- 1 mimo3000n mimo3000n   1066338 Mar 27 17:25 u-boot-dtb.bin
-rw-r--r-- 1 mimo3000n mimo3000n   1070860 Mar 27 17:25 u-boot-dtb.elf
-rwxr-xr-x 1 mimo3000n mimo3000n   1042360 Mar 27 17:25 u-boot.bin
-rwxr-xr-x 1 mimo3000n mimo3000n   8461216 Mar 27 17:25 u-boot.elf
-rw-r--r-- 1 mimo3000n mimo3000n   5131896 Mar 26 13:51 uImage
-rw-r--r-- 1 mimo3000n mimo3000n  15062008 Mar 26 13:51 vmlinux
-rw-r--r-- 1 mimo3000n mimo3000n   5131832 Mar 26 13:51 zImage
-rw-r--r-- 1 mimo3000n mimo3000n    413192 Mar 19 17:28 zynq_fsbl.elf
mimo3000n@KLG-PC001:~/petalinux/2024.2/MYD-C7Z020/images/linux$

cp images/linux/BOOT.BIN /media/$USER/BOOT/
cp images/linux/image.ub /media/$USER/BOOT/
cp images/linux/boot.scr /media/$USER/BOOT/

# To Partition 2 (rootfs)
sudo tar -xpf images/linux/rootfs.tar.gz -C /media/$USER/rootfs/
sync
```

---

## 🔌 Hardware Setup (MYD-C7Z020)

### Jumper Settings
| Mode | SW1 (MIO 5:2) | JP3 (JTAG Chain) |
| :--- | :--- | :--- |
| **SD Boot** | `1100` | Closed (Cascaded) |
| **JTAG Debug** | `0000` | Closed (Cascaded) |

---

## 🛠 Troubleshooting Summary
* **"Bitbake is not available":** Ensure you ran `source settings.sh` in your current terminal session.
* **"Failed to copy to tftp":** This is a permission warning; ignore it. Images are safe in `images/linux/`.
* **Terminal Closes Suddenly:** Check `dmesg | grep -i oom`. Increase swap space and lower the job count in `petalinux-config`.

---
</details>

---
## 🚀 JTAG Setup and Programming Guide
---
[Click to expand: JTAG Setup and Configuration Giuide ](JTAG.md)

---
## 🕹️ Controlling the Zynq-7000 PS via OpenOCD
---
[Click to expand - 🕹️ Controlling the Zynq-7000 PS via OpenOCD](OPENOCD_PS_CONTROL.md)

---
