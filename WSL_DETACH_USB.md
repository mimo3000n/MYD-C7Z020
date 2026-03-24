# Zynq-7000 (XC7Z020) WSL2 USB Management Guide
This document provides a professional workflow for bridging the JTAG-HS3 programmer and the UART Serial Console from a Windows 10/11 host into a WSL2 (Ubuntu) instance.

## 1. System Requirements
To use these scripts, the following must be installed on the Windows Host:

usbipd-win: Latest Release

WSL Kernel: Updated via wsl --update in a Windows terminal.

Adept Runtime: Installed in Ubuntu to provide the djtgcfg utility.

## 2. Hardware Identification
You must identify the USB Bus IDs assigned by Windows. Open PowerShell and run:

``` PowerShell
usbipd list
Identify the following:

JTAG-HS3: Listed as "Digilent USB Device" (e.g., Bus ID 2-1).

UART/Serial: Listed as "USB Serial Converter" or "FT232R USB UART" (e.g., Bus ID 2-4).
```

## 3. Automation Scripts
Script: attach_zynq.sh
This script captures the devices from Windows and mounts them into the Linux kernel.

``` Bash
#!/bin/bash
# Usage: ./attach_zynq.sh <JTAG_BUSID> <SERIAL_BUSID>

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <JTAG_BUSID> <SERIAL_BUSID>"
    exit 1
fi

JTAG_ID=$1
SERIAL_ID=$2

attach_device() {
    local BUSID=$1
    local NAME=$2
    echo "--- Attaching $NAME (Bus ID: $BUSID) ---"
    
    # Bind the device on Windows side (calls .exe from WSL)
    usbipd.exe bind --busid "$BUSID" 2>/dev/null
    
    # Attach to the current WSL instance
    usbipd.exe attach --wsl --busid "$BUSID"
    
    if [ $? -eq 0 ]; then
        echo "SUCCESS: $NAME is now available in Ubuntu."
    else
        echo "ERROR: Failed to attach $NAME. Check Windows Admin permissions."
    fi
}

attach_device "$JTAG_ID" "JTAG-HS3"
attach_device "$SERIAL_ID" "UART/Serial Console"

echo -e "\nDetected USB Devices in WSL:"
lsusb
```

Script: detach_zynq.sh


This script releases the devices so they can be used by Windows applications (like Vivado Hardware Manager or PuTTY).

``` Bash
#!/bin/bash
# Usage: ./detach_zynq.sh <JTAG_BUSID> <SERIAL_BUSID>

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <JTAG_BUSID> <SERIAL_BUSID>"
    exit 1
fi

JTAG_ID=$1
SERIAL_ID=$2

detach_device() {
    local BUSID=$1
    local NAME=$2
    echo "--- Detaching $NAME (Bus ID: $BUSID) ---"
    
    # Release from WSL back to Windows
    usbipd.exe detach --busid "$BUSID"
    
    if [ $? -eq 0 ]; then
        echo "SUCCESS: $NAME returned to Windows."
    else
        echo "INFO: $NAME was already detached."
    fi
}

detach_device "$JTAG_ID" "JTAG-HS3"
detach_device "$SERIAL_ID" "UART/Serial Console"
``` 

## 4. Setup Instructions
Create the files in WSL:

``` Bash
touch attach_zynq.sh detach_zynq.sh
```

Paste the code into each file using nano or your preferred editor.

Grant Execution Permissions:

``` Bash
chmod +x attach_zynq.sh detach_zynq.sh
```

## 5. Development Workflow

Step 1: Attach Devices
Run the script with your specific Bus IDs:

``` Bash
./attach_zynq.sh 2-1 2-4
```

Step 2: Verify JTAG Chain
Ensure the XC7Z020 is visible:

``` Bash
djtgcfg init -d JtagHs3
```

Step 3: Open Serial Console
Connect to the ARM processor's output (usually /dev/ttyUSB0 at 115200 baud):

```Bash
screen /dev/ttyUSB0 115200
```

(Exit screen: Ctrl+A, then K, then Y)

Step 4: Detach (When finished)

``` Bash
./detach_zynq.sh 2-1 2-4
```

## 6. Common Troubleshooting

Permission Denied (/dev/ttyUSB0): Run sudo usermod -a -G dialout $USER, then log out and back in.

Command Not Found: If usbipd.exe is missing, ensure the usbipd-win installation directory is in your Windows System PATH.

Device Busy: Ensure no Windows application (like Vivado) is currently using the JTAG cable before attempting to attach.
