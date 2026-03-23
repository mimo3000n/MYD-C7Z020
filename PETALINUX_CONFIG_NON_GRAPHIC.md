
---

# ⌨️ Non-Graphic PetaLinux Configuration

This document describes how to configure your PetaLinux project without using the `ncurses` (blue screen) graphical interface. This is ideal for **WSL2** performance, remote SSH sessions, or automated build scripts.

## 1. The Strategy: Silent Configuration
Instead of running `petalinux-config`, we interact directly with the underlying **Yocto configuration files**.

### Initial Hardware Import (Silent)
To import an `.xsa` file without opening the menu, use the `--silentconfig` flag:
```bash
petalinux-config --get-hw-description=/path/to/hardware.xsa --silentconfig
```
*This will apply all hardware defaults automatically based on your Vivado design.*

---

## 2. Manual Configuration via Text Files
All settings that you usually find in the menus are stored in plain text files within your project.

### System Settings (The "Main" Menu)
The settings from `petalinux-config` are stored here:
**File:** `project-spec/configs/config`

**Common Manual Edits:**
* **Change Kernel Boot Args:** Search for `CONFIG_SUBSYSTEM_USER_CMDLINE`.
* **Change Parallel Jobs:** ```text
  CONFIG_YOCTO_PARALLEL_MAKE_JOBS="4"
  CONFIG_YOCTO_BB_NUMBER_THREADS="4"
  ```
* **Set Static IP:**
  ```text
  CONFIG_SUBSYSTEM_ETHERNET_0_IP_TYPE_STATIC=y
  CONFIG_SUBSYSTEM_ETHERNET_0_IPV4_ADDR="192.168.1.10"
  ```

### RootFS Settings (The "App" Menu)
The settings from `petalinux-config -c rootfs` are stored here:
**File:** `project-spec/configs/rootfs_config`

**How to add a package (e.g., OpenSSH):**
Open the file and search for the package name. Change `n` to `y`:
```text
CONFIG_packagegroup-petalinux-ssh-openssh=y
# CONFIG_packagegroup-petalinux-ssh-dropbear is not set
```

---

## 3. Applying Changes (Crucial Step)
After you manually edit these text files, PetaLinux doesn't know they have changed until you "refresh" the project state.

Run this command to sync your manual text edits into the build system:
```bash
petalinux-config --silentconfig
```

---

## 4. Advanced: Using Configuration Fragments
The "Pro" way to configure PetaLinux without graphics is to use **Config Fragments**. This keeps your changes separate from the auto-generated files.

### For the Kernel:
1. Create a file: `project-spec/meta-user/recipes-kernel/linux/linux-xlnx/my_extra.cfg`
2. Add your Kernel flags (e.g., `CONFIG_USB_STORAGE=y`).
3. PetaLinux will automatically merge this during the next `petalinux-build`.

### For the Device Tree:
Instead of using a menu to change hardware nodes, edit the `.dtsi` files directly:
**File:** `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`

---

## 5. Summary Table

| Goal | Graphical Command | Non-Graphical Method |
| :--- | :--- | :--- |
| **Import HW** | `petalinux-config --get-hw-description` | Add `--silentconfig` flag |
| **Edit RAM/CPU** | `petalinux-config` -> Yocto | Edit `project-spec/configs/config` |
| **Add Packages** | `petalinux-config -c rootfs` | Edit `project-spec/configs/rootfs_config` |
| **Modify DTB** | N/A | Edit `system-user.dtsi` |

---

## 🛠 Troubleshooting
* **Syntax Errors:** If you make a typo in the text files (e.g., forgetting a quote), `petalinux-config --silentconfig` will fail with an error. Check the logs in `build/config.log`.
* **Missing Settings:** If a setting isn't in the `config` file, it might not have been generated yet. Run a silent config once to populate the file with defaults first.

---
