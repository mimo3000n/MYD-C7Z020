
---

# 🤖 Automated Silent Build & Scripting

This document explains how to use the provided `main.sh` and `silent_build.sh` to manage your PetaLinux project without any manual "Blue Screen" menu interaction.

## 1. Why use Automated Silent Builds?
* **Consistency:** Avoids human error (forgetting to check a box in a menu).
* **Performance:** WSL2 does not have to render the `ncurses` graphics, saving CPU cycles.
* **Speed:** One-click deployment from the Windows desktop.

---

## 2. The Silent Build Script (`silent_build.sh`)

Create this file in your project root. It forces PetaLinux to accept all defaults and hardware changes without opening a GUI.

```bash
#!/bin/bash
# silent_build.sh - Automated PetaLinux Build Workflow

# 1. Environment Setup
source ~/petalinux/2024.2/settings.sh

echo "--- Starting Automated Silent Build ---"

# 2. Apply hardware changes and configurations silently
# This refreshes the project based on your text-file edits
petalinux-config --get-hw-description=./ --silentconfig

# 3. Clean stale locks to prevent interruption
rm -f build/bitbake.lock

# 4. Trigger the build with specific job limits for WSL2
# We use -j 4 to match our .wslconfig and prevent OOM crashes
petalinux-build -j 4

# 5. Package the final BOOT.BIN automatically
echo "--- Packaging BOOT.BIN ---"
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga project-spec/hw-description/system.bit --u-boot --force

echo "--- Build Complete: Files located in images/linux/ ---"
```

---

## 3. Updated Master Script (`main.sh`)

Update your `main.sh` to include this "Zero-Interaction" mode.

```bash
#!/bin/bash
source ~/petalinux/2024.2/settings.sh

echo "--------------------------------------"
echo " MYIR Z7020 Master Automation"
echo "--------------------------------------"
echo "1) Full Silent Build (No Menus)"
echo "2) Menu-Based Config (Graphical)"
echo "3) JTAG Boot (Fast Test)"
echo "4) Hardware Console (picocom)"
echo "5) Exit"
read -p "Selection: " choice

case $choice in
    1)
        chmod +x silent_build.sh
        ./silent_build.sh
        ;;
    2)
        petalinux-config
        ;;
    3)
        petalinux-boot --jtag --kernel --fpga
        ;;
    4)
        sudo picocom -b 115200 /dev/ttyUSB0
        ;;
    5)
        exit 0
        ;;
esac
```

---

## 4. One-Click Windows Build (`build_from_windows.bat`)

You can even trigger the entire Linux build from your Windows Desktop. Create this `.bat` file on Windows:

```batch
@echo off
echo Sending Silent Build command to WSL2...
wsl -d Ubuntu-24.04 -e bash -c "cd ~/MYD-C7Z020 && ./silent_build.sh"
pause
```

---

## 5. Summary of Automation Files

| File | OS | Purpose |
| :--- | :--- | :--- |
| **`silent_build.sh`** | Linux | Runs the build/package process without GUI. |
| **`main.sh`** | Linux | Interactive menu for daily development tasks. |
| **`attach_myir.bat`** | Windows | Connects JTAG/Serial cables to WSL2. |
| **`build_from_windows.bat`** | Windows | Triggers the Linux build from the Windows UI. |

---

## 🛠 Best Practices for Automation
1.  **Keep `project-spec/configs/` updated:** Since you aren't using the GUI, make sure you manually edit the `config` and `rootfs_config` files if you need to add new drivers or apps.
2.  **Monitor Logs:** Because there is no UI, check `build/build.log` if the script stops unexpectedly.
3.  **Permissions:** Always ensure your scripts are executable using `chmod +x *.sh`.

---
