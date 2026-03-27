
---

# Disabling X11 in PetaLinux (Zynq-7000)

This document outlines the steps to remove or deactivate the X11 Display Server from a PetaLinux project. Since X11 is a **User Space** application and not a kernel component, it must be removed via the RootFS configuration.



## Method 1: Build-Time Removal (Permanent)
This is the recommended approach to reduce image size and boot time.

### 1. Configure RootFS
Run the following command from your PetaLinux project directory:
```bash
petalinux-config -c rootfs
```

### 2. Deselect X11 Package Groups
Navigate through the menu and uncheck the following:
* **PetaLinux Package Groups** * `packagegroup-petalinux-x11` —> **Deselect**
    * `packagegroup-petalinux-matchbox` —> **Deselect** (if present)

### 3. Disable Image Features
Navigate to:
* **Image Features**
    * `x11-base` —> **Deselect**
    * `hwcodecs` —> (Optional) Keep if you need hardware acceleration for non-X11 tasks.

### 4. Build and Package
Save the configuration and rebuild your project:
```bash
petalinux-build
petalinux-package --boot --fsbl <PATH_TO_FSBL> --fpga <PATH_TO_BITSTREAM> --u-boot
```

---

## Method 2: Kernel Driver Cleanup (Optional)
If you want to remove the underlying video drivers (DRM/Framebuffer) to ensure a completely headless kernel:

1.  **Open Kernel Config:**
    ```bash
    petalinux-config -c kernel
    ```
2.  **Navigate to Device Drivers:**
    * `Device Drivers` -> `Graphics support`
    * Deselect `Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)`
    * Deselect `Support for frame buffer devices`
3.  **Save and Exit.**

---

## Method 3: Runtime Deactivation (Quick Fix)
If you cannot rebuild the image and need to stop X11 on a running system.

### 1. Change the Default Runlevel
PetaLinux (SysVinit) uses **Runlevel 5** for Graphics and **Runlevel 3** for Text mode.

```bash
# Edit the init table
vi /etc/inittab
```
Find this line:
`id:5:initdefault:`

Change it to:
`id:3:initdefault:`

### 2. Disable the Init Script
Alternatively, rename the start script so it doesn't execute on boot:
```bash
mv /etc/init.d/xserver-nodm /etc/init.d/xserver-nodm.bak
```

---

## Summary of Runlevels
| Runlevel | Description | Status |
| :--- | :--- | :--- |
| **0** | Halt | System Shutdown |
| **1** | Single-user | Maintenance mode |
| **3** | Multi-user (CLI) | **Target for Headless Systems** |
| **5** | Multi-user (GUI) | **Default with X11 enabled** |
| **6** | Reboot | System Restart |

---

**Would you like me to help you create a custom bitbake recipe to ensure X11 is never accidentally included in future builds?**
