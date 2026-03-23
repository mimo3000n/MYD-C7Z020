This script will automate the "WSL2-to-Hardware" bridge. It handles the Windows-side USB attachment (via `usbipd`) and the Ubuntu-side environment setup in one go.

### 1. The Windows "Attach" Script (`attach_myir.bat`)
Create this file on your **Windows Desktop**. It saves you from typing the Bus IDs into PowerShell every time you plug the board in.

```batch
@echo off
:: Replace these with your actual Bus IDs from 'usbipd list'
SET JTAG_BUSID=2-1
SET UART_BUSID=2-4

echo [1/2] Attaching Digilent JTAG-HS3...
usbipd attach --wsl --busid %JTAG_BUSID%

echo [2/2] Attaching MYIR Console Cable...
usbipd attach --wsl --busid %UART_BUSID%

echo Ready! Switch to your Ubuntu terminal.
pause
```

---

### 2. The Ubuntu "Build & Run" Script (`main.sh`)
Create this file inside your **PetaLinux project root** (`~/petalinux/2024.2/MYD-C7Z020/main.sh`).

```bash
#!/bin/bash

# 1. Source the Xilinx Environment
source ~/petalinux/2024.2/settings.sh

# 2. Check for USB devices
echo "--- Checking Hardware Connectivity ---"
lsusb | grep -E "Future|Prolific|QinHeng" || echo "WARNING: USB devices not found. Run attach_myir.bat in Windows."

# 3. Main Menu
echo "--------------------------------------"
echo " MYIR Z7020 Automation Script"
echo "--------------------------------------"
echo "1) Full Build (petalinux-build)"
echo "2) Clean & Restart Build (Recovery)"
echo "3) Boot Kernel via JTAG (Fast Test)"
echo "4) Open Serial Console (picocom)"
echo "5) Exit"
read -p "Select an option [1-5]: " choice

case $choice in
    1)
        petalinux-build
        ;;
    2)
        echo "Clearing locks and restarting..."
        rm -f build/bitbake.lock
        pkill -u $USER -f bitbake
        petalinux-build
        ;;
    3)
        echo "Loading FPGA and Kernel over JTAG..."
        petalinux-boot --jtag --kernel --fpga
        ;;
    4)
        sudo picocom -b 115200 /dev/ttyUSB0
        ;;
    5)
        exit 0
        ;;
    *)
        echo "Invalid option."
        ;;
esac
```

**To make it executable:**
```bash
chmod +x main.sh
```

---

### 3. Final Repository Structure
Your GitHub repository should now look like this:

* **`README.md`** (The main landing page)
* **`WSL2_SETUP.md`** (Windows & Linux environment prep)
* **`PETALINUX_CONFIG.md`** (XSA import and Yocto optimization)
* **`HARDWARE_JUMPERS.md`** (SW1/JP3 settings)
* **`DEPLOYMENT_SUMMARY.md`** (SD card partitioning)
* **`JTAG.md`** (Advanced debugging)
* **`CHANGELOG.md`** (Your build history)
* **`main.sh`** (The automation script)
* **`.gitignore`** (To keep the repo clean)

