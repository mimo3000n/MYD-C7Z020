
---

# 💻 WSL2 & PetaLinux Installation Guide

This document covers the specific environmental setup required to run PetaLinux 2024.2 on **Windows 11 Pro** using **WSL2 (Ubuntu 24.04/02)**.

## 1. Windows Host Configuration (WSL2)

Before opening Ubuntu, you must configure how Windows allocates resources to the Virtual Machine. PetaLinux is extremely resource-intensive; without these limits, Windows may become unstable or crash the build.

### Create `.wslconfig`
1. Open Windows Explorer and go to your user folder: `C:\Users\<YourUserName>\`.
2. Create a text file named `.wslconfig` (ensure there is no `.txt` extension).
3. Paste the following configuration:

```ini
[wsl2]
# Limit RAM to 50-75% of your total system memory
memory=16GB 

# Limit CPU cores to prevent overheating during 2+ hour builds
processors=8 

# Large swap file on a fast SSD is CRITICAL for PetaLinux
swap=24GB

# Disabling GUI applications can save background resources
guiApplications=false
```

### Enable USB Passthrough (usbipd)
To use your **JTAG-HS3** and **MYIR Console Cable**, install the USBIPD tool on Windows:
1. Download/Install: `winget install usbipd`
2. Restart your computer.

---

## 2. Ubuntu Environment Prep

Once inside your WSL2 Ubuntu terminal, perform these system-level fixes.

### Fix the Default Shell
PetaLinux scripts are strictly written for `bash`. Ubuntu uses `dash` by default.
```bash
sudo dpkg-reconfigure dash
# Select [NO] when the dialog box appears.
```

### Install Required Libraries (Ubuntu 24.x)
Ubuntu 24.04/02 requires several legacy libraries that are no longer included by default.
```bash
sudo apt update
sudo apt install -y iproute2 gawk make net-tools libncurses5-dev tftpd-hpa \
flex bison libssl-dev xterm texinfo devscripts python3-pip gcc-multilib \
build-essential zlib1g:i386 screen pax gzip tar decode-dimms libncurses5
```

---

## 3. PetaLinux Tool Installation

### Step 1: Create Installation Directory
Do **not** install PetaLinux as `root`. Install it in your home directory.
```bash
mkdir -p ~/petalinux/2024.2
```

### Step 2: Run the Installer
Navigate to where you downloaded the `.run` file (e.g., `/mnt/c/Users/<User>/Downloads/`) and execute:
```bash
chmod +x ./petalinux-v2024.2-final-installer.run
./petalinux-v2024.2-final-installer.run --dir ~/petalinux/2024.2
```

### Step 3: Accept License Agreements
The installer will pause. You must read the three licenses:
1. Press **Enter** to view.
2. Press **q** to close the viewer.
3. Press **y** to accept.
*Repeat this for the Xilinx, Open Source, and Third-Party licenses.*

---

## 4. Verification

To verify the installation, source the environment and check the version:
```bash
source ~/petalinux/2024.2/settings.sh
petalinux-build --version
```

### ⚠️ WSL2 File System Warning
**ALWAYS** keep your PetaLinux project files within the Linux file system (e.g., `/home/mimo3000n/projects/`). 
* **NEVER** build on `/mnt/c/`. The performance hit from the Windows/Linux file bridge will cause build times to triple and likely cause permission errors during the RootFS generation.

---
