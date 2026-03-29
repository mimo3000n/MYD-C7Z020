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

-----------------
## **U-Boot Configuration Guide: Deactivating Autoboot** 
-----------------

### **1. Accessing the U-Boot Prompt**
To begin, connect your board to a host PC via the serial console (UART). Power on the board and immediately press any key (usually **Enter** or **Space**) when you see the following message:

U-Boot 2020.01 (Sep 26 2022 - 01:59:45 +0000)

CPU:   Zynq 7z020
Silicon: v3.1
DRAM:  ECC disabled 1 GiB
myir_board_init
MMC:   mmc@e0100000: 0, mmc@e0101000: 1
Loading Environment from SPI Flash... SF: Detected n25q256a with page size 256 Bytes, erase size 4 KiB, total 32 MiB
*** Warning - bad CRC, using default environment

In:    serial@e0001000
Out:   serial@e0001000
Err:   serial@e0001000
Net:
ZYNQ GEM: e000b000, mdio bus e000b000, phyaddr -1, interface rgmii-id

Warning: ethernet@e000b000 using MAC address from DT
eth0: ethernet@e000b000

> `Hit any key to stop autoboot: 3`

Once interrupted, you should see the command prompt:
`Zynq> `

---

### **2. Inspecting Current Settings**
Before making changes, it is important to see what the board is currently configured to do. Use the `printenv` command to view the boot variables.

* **To check the primary boot command:**
    ```bash
    printenv bootcmd
    ```
* **To see the entire environment list:**
    ```bash
    printenv

   Zynq> printenv
   arch=arm
   autoload=no
   baudrate=115200
   board=zynq
   board_name=zynq
   boot_img=BOOT.BIN
   boot_targets=qspi
   bootcmd=run $modeboot
   bootdelay=4
   bootenv=uEnv.txt
   bootenvsize=0x20000
   bootenvstart=0x500000
   clobstart=0x10000000
   console=console=ttyPS0,115200
   cp_kernel2ram=mmcinfo && fatload mmc ${sdbootdev} ${netstart} ${kernel_img}
   cpu=armv7
   default_bootcmd=run cp_kernel2ram && bootm ${netstart}
   display_type=HDMI
   dtb_img=system.dtb
   dtbnetstart=0x23fff000
   eraseenv=sf probe 0 && sf erase ${bootenvstart} ${bootenvsize}
   ethaddr=00:0a:35:00:00:00
   fault=echo ${img} image size is greater than allocated place - partition ${img} is NOT UPDATED
   fdtcontroladdr=3ffc31d0
   importbootenv=echo "Importing environment from SD ..."; env import -t ${loadbootenv_addr} $filesize
   install_boot=mmcinfo && fatwrite mmc ${sdbootdev} ${clobstart} ${boot_img} ${filesize}
   install_jffs2=sf probe 0 && sf erase ${jffs2start} ${jffs2size} && sf write ${clobstart} ${jffs2start} ${filesize}
   install_kernel=mmcinfo && fatwrite mmc ${sdbootdev} ${clobstart} ${kernel_img} ${filesize}
   jffs2_img=rootfs.jffs2
   kernel_img=image.ub
   load_boot=tftpboot ${clobstart} ${boot_img}
   load_dtb=tftpboot ${clobstart} ${dtb_img}
   load_jffs2=tftpboot ${clobstart} ${jffs2_img}
   load_kernel=tftpboot ${clobstart} ${kernel_img}
   loadaddr=0x10000000
   loadbootenv=load mmc $sdbootdev:$partid ${loadbootenv_addr} ${bootenv}
   loadbootenv_addr=0x00100000
   mmc_args=setenv bootargs ${console} root=/dev/mmcblk1p1 rw earlyprintk rootfstype=ext4 rootwait devtmpfs.mount=1 myir_encoder.display_type=${display_type}
   modeboot=qspiboot
   nc=setenv stdout nc;setenv stdin nc;
   netboot=tftpboot ${netstart} ${kernel_img} && bootm
   netstart=0x10000000
   psserial0=setenv stdout ttyPS0;setenv stdin ttyPS0
   qspiboot=run mmc_args; sf probe 0 && sf read 10000000 0x00520000 0x00a00000 && bootm ${netstart};
   script_offset_f=f10000
   sd_uEnvtxt_existence_test=test -e mmc $sdbootdev:$partid /uEnv.txt
   sd_update_dtb=echo Updating dtb from SD; mmcinfo && fatload mmc ${sdbootdev}:1 ${clobstart} ${dtb_img} && run install_dtb
   sd_update_jffs2=echo Updating jffs2 from SD; mmcinfo && fatload mmc ${sdbootdev}:1 ${clobstart} ${jffs2_img} && run install_jffs2
   sdboot=run uenvboot; run cp_kernel2ram && bootm ${netstart}
   sdbootdev=0
   serial=setenv stdout serial;setenv stdin serial
   serverip=192.168.30.2
   soc=zynq
   stderr=serial@e0001000
   stdin=serial@e0001000
   stdout=serial@e0001000
   test_crc=if imi ${clobstart}; then run test_img; else echo ${img} Bad CRC - ${img} is NOT UPDATED; fi
   test_img=setenv var "if test ${filesize} -gt ${psize}; then run fault; else run ${installcmd}; fi"; run var; setenv var
   uenvboot=if run sd_uEnvtxt_existence_test; then run loadbootenv; echo Loaded environment from ${bootenv}; run importbootenv; fi; if test -n $uenvcmd; then echo Running uenvcmd ...; run uenvcmd; fi
   uenvcmd= setenv bootargs console=ttyPS0,115200 earlycon root=/dev/mmcblk0p2 rw rootwait myir_encoder.display_type=${display_type}
   update_boot=setenv img boot; setenv psize ${bootsize}; setenv installcmd "install_boot"; run load_boot ${installcmd}; setenv img; setenv psize; setenv installcmd
   update_dtb=setenv img dtb; setenv psize ${dtbsize}; setenv installcmd "install_dtb"; run load_dtb test_img; setenv img; setenv psize; setenv installcmd
   update_jffs2=setenv img jffs2; setenv psize ${jffs2size}; setenv installcmd "install_jffs2"; run load_jffs2 test_img; setenv img; setenv psize; setenv installcmd
   update_kernel=setenv img kernel; setenv psize ${kernelsize}; setenv installcmd "install_kernel"; run load_kernel ${installcmd}; setenv img; setenv psize; setenv installcmd
   vendor=xilinx

   Environment size: 3594/131068 bytes
   Zynq>

    ```

> **Note:** On MYIR boards, `bootcmd` often contains a reference like `run $modeboot`. If so, you should also run `printenv modeboot` to see the actual script being executed.

---

### **3. Modifying Autoboot Behavior**
You can stop the automatic boot by either increasing the reaction time or neutralizing the command itself.

### **Option A: Increase the Boot Delay**
This gives you a longer window to manually interrupt the boot process without fully disabling it.
```bash
setenv bootdelay 10
```

### **Option B: Disable the Boot Command**
To prevent the board from ever attempting to load the OS automatically, replace the `bootcmd` with a simple message.
```bash
setenv bootcmd echo "Autoboot disabled. Please boot manually."
```

---

### **4. Saving Your Changes**
Changes made with `setenv` are only stored in RAM. To make these changes permanent (surviving a power cycle), you **must** save them to the onboard flash memory.

```bash
saveenv
```

---

### **5. Manual Boot**

to boot from SPI

```bash

Zynq> run qspiboot
SF: Detected n25q256a with page size 256 Bytes, erase size 4 KiB, total 32 MiB
device 0 offset 0x520000, size 0xa00000
SF: 10485760 bytes @ 0x520000 Read: OK
## Loading kernel from FIT Image at 10000000 ...
   Using 'conf@system-top.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'kernel@1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x100000f8
     Data Size:    4452016 Bytes = 4.2 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x00200000
     Entry Point:  0x00200000
     Hash algo:    sha256
     Hash value:   d242df307fa2c82501fd079630f9eb225c75f9cea706cc8755ac65d339aa3553
   Verifying Hash Integrity ... sha256+ OK
## Loading fdt from FIT Image at 10000000 ...
   Using 'conf@system-top.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'fdt@system-top.dtb' fdt subimage
     Description:  Flattened Device Tree blob
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x1043f0b4
     Data Size:    20486 Bytes = 20 KiB
     Architecture: ARM
     Hash algo:    sha256
     Hash value:   2f1a1b836ddd16034a33e32c2100e6224fc22848bfdc5b2763b07748cba6c5fd
   Verifying Hash Integrity ... sha256+ OK
   Booting using the fdt blob at 0x1043f0b4
   Loading Kernel Image
   Loading Device Tree to 07ff7000, end 07fff005 ... OK

Starting kernel ...

Booting Linux on physical CPU 0x0
Linux version 5.4.0-xilinx-v2020.1 (oe-user@oe-host) (gcc version 9.2.0 (GCC)) #1 SMP PREEMPT Mon Sep 19 02:48:52 UTC 2022
CPU: ARMv7 Processor [413fc090] revision 0 (ARMv7), cr=18c5387d
CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
OF: fdt: Machine model: xlnx,zynq-7000
printk: bootconsole [earlycon0] enabled
Memory policy: Data cache writealloc
cma: Reserved 32 MiB at 0x3e000000
..........
```

### **Summary Table of Commands**

| Task | Command |
| :--- | :--- |
| **View Variable** | `printenv <variable_name>` |
| **Set Delay** | `setenv bootdelay <seconds>` |
| **Set Command** | `setenv bootcmd <command_string>` |
| **Save Changes** | `saveenv` |
| **Manual Boot** | `boot` or `run bootcmd` |

---

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
