# MYD-C7Z020
MYD-C7Z020 experience

---



# PetaLinux 2024.2 Build Guide for MYIR MYD-C7Z020

This guide documents the setup and build process for the MYIR XC7Z020 board. It includes specific fixes for **Ubuntu 24.x** environments and common resource-related build failures.

## My Environment

### 1. The board - MYD-C7Z010/20-V2 Dev. Board, Artikel MYD-C7Z020-V2-4E1D-766-I

<img width="709" height="1261" alt="image" src="https://github.com/user-attachments/assets/83d507e6-87a4-4031-90fb-e7e2763e9e9b" />

Initial connectied via MYIR serial2USB cabel to Host-PC.

for finding connected serial port (used for console I/O) use **<U> usbipd </U>** command. With usbipd show you list all USB divices:

``` cmd

C:\Windows\System32>usbipd list
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
1-8    04e8:4002  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
2-2    8087:0029  Intel(R) Wireless Bluetooth(R)                                Not shared
4-4    05e3:0749  USB-Massenspeichergerät                                       Shared
6-3    046d:c52b  Logitech USB Input Device, USB-Eingabegerät                   Not shared
8-1    04e8:4001  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
8-2    0bc2:331a  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
8-3    0bc2:331a  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
8-4    0bc2:2038  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
10-4   04e8:6860  A52 von Gerhard, SAMSUNG Mobile USB Modem, SAMSUNG Androi...  Not shared
12-3   067b:2303  Prolific USB-to-Serial Comm Port (COM10)                      Shared
14-1   2b7e:c569  NexiGo HelloCam N930W Camera, NexiGo HelloCam N930W IR Ca...  Not shared
14-3   2b7e:a569  NexiGo HelloCam N930W Audio, USB-Eingabegerät                 Not shared
15-1   04f3:0c3d  ELAN WBF Fingerprint Sensor                                   Not shared

Persisted:
GUID                                  DEVICE
bbba9b06-911c-4d21-ac12-916c1c9056b2  Serielles USB-Gerät (COM9), Serielles USB-Gerät (COM8), B...
f91bc313-54db-4470-8338-96b851aff57f  Dual RS232-HS, USB Serial Converter B


C:\Windows\System32>

```


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
sudo fallocate -l 16G /swapfile_plnx
sudo chmod 600 /swapfile_plnx
sudo mkswap /swapfile_plnx
sudo swapon /swapfile_plnx
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

### Copying Files
```bash
# To Partition 1 (BOOT)
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
