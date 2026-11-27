# PetaLinux 2026.2 Setup & Build Guide – MYIR MYD-C7Z020-V2 (Ubuntu 24.04 WSL2)

Vollständige Schritt-für-Schritt-Dokumentation zur Installation von **AMD PetaLinux 2026.2** unter **Ubuntu 24.04 (WSL2)**, zur Konvertierung der Vivado-`.xsa`-Datei in einen **System Device Tree (SDT)** sowie zur Hardware- und Kernel-Konfiguration für das **MYIR MYD-C7Z020-V2** Board (SOM: `MYC-C7Z020-V2-4E1D-766-I`, Baseboard: `MYB-C7Z010/20 V1.2`).

> **Wichtiger Hinweis zu WSL2:** Sowohl die Installation von PetaLinux als auch alle Projekt-Builds müssen zwingend auf dem nativen Linux-Dateisystem (z. B. unter `~/`) stattfinden – niemals direkt auf eingebundenen Windows-Laufwerken (`/mnt/c/` oder `/mnt/d/`), da Yocto/BitBake dort mangels echter POSIX-Dateiberechtigungen fehlschlägt.

---

## 1. Systemvoraussetzungen & Paketabhängigkeiten installieren

Aktualisiere die Paketquellen in Ubuntu 24.04 und installiere alle von PetaLinux 2026.2 benötigten Build-Werkzeuge und Bibliotheken:

```bash
sudo apt update
sudo apt install -y iproute2 gawk python3 python3-pexpect python3-pip python3-git \
  python3-jinja2 xz-utils debianutils iputils-ping libegl1 libsdl1.2-dev \
  pylint xterm autoconf libtool texinfo zlib1g-dev gcc-multilib build-essential \
  libncurses-dev libglib2.0-dev libpixman-1-dev locales cpio unzip rsync bc \
  screen pax diffstat chrpath socat zstd lz4 file
```

---

## 2. System-Shell auf `bash` umstellen & Locale konfigurieren

PetaLinux/BitBake bricht ab, wenn `/bin/sh` auf `dash` verweist. Zudem sollte ggf. eine aktive Conda-Umgebung vorher deaktiviert werden:

```bash
sudo ln -sf /bin/bash /bin/sh
sudo locale-gen en_US.UTF-8
```

*(Falls unter Ubuntu 24.04 AppArmor unprivilegierte User-Namespaces einschränkt, gegebenenfalls zusätzlich ausführen: `echo 0 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns`)*

---

## 3. PetaLinux 2026.2 Installer kopieren und installieren

Kopiere den heruntergeladenen Installer in ein temporäres Linux-Verzeichnis. Mit `-y` werden die Lizenzabfragen automatisch bestätigt und mit `--platform "arm"` werden ausschließlich die Zynq-7000-Quellen installiert:

```bash
conda deactivate  # falls Conda aktiv ist
mkdir -p ~/tmp_plnx ~/petalinux/2026.2
cp /mnt/c/Users/mimo3/Downloads/petalinux-v2026.2-11071905-installer.run ~/tmp_plnx/
chmod +x ~/tmp_plnx/petalinux-v2026.2-11071905-installer.run

~/tmp_plnx/petalinux-v2026.2-11071905-installer.run --dir ~/petalinux/2026.2 --platform "arm" -y

# Temporären Installer-Ordner zur Platzersparnis wieder entfernen
rm -rf ~/tmp_plnx
```

**Terminal-Output:**
```text
PetaLinux CMD tools installer version 2026.2
============================================
[WARNING] This is not a supported OS
[WARNING] The PetaLinux toolset is scheduled for deprecation in the 2026.2 release. Users are advised to adopt AMD EDF and its Yocto Project based workflows.
[WARNING] Microblaze BSP support has been removed starting from the 2026.2 release. Users are advised to transition to MB-V as the alternative.
[INFO] Checking free disk space
[INFO] Checking installed tools
[INFO] Checking installed development libraries
[INFO] Checking network and other services
...
Do you accept Xilinx End User License Agreement? [y/N] >  Y
Do you accept Third Party End User License Agreement? [y/N] >  Y
[INFO] Installing PetaLinux SDK...done
[INFO] Setting it up...done
[INFO] Extracting xsct tarball...done
[INFO] PetaLinux SDK has been successfully set up and is ready to be used.
```

---

## 4. `.xsa` über `sdtgen` in System Device Tree (SDT) umwandeln

Ab **PetaLinux 2026.2** wird der direkte `.xsa`-Import nicht mehr unterstützt (`[ERROR] XSA file input is no longer supported starting from the 2026.2 release. Please provide a system device tree (SDT) directory instead.`). Stattdessen wird die `.xsa`-Datei mit dem im SDK enthaltenen `sdtgen` und dem `system-device-tree-xlnx`-Repository über `CUSTOM_SDT_REPO` in ein SDT-Verzeichnis überführt:

```bash
source ~/petalinux/2026.2/settings.sh

mkdir -p ~/petalinux_myd_c7z020/hw ~/petalinux_myd_c7z020/sdt_out
cp /mnt/d/FPGA/myd_c7z020_v2/myd_c7z020_v2_hw/design_1_wrapper.xsa ~/petalinux_myd_c7z020/hw/

cd ~/petalinux_myd_c7z020
git clone https://github.com/Xilinx/system-device-tree-xlnx.git ~/petalinux_myd_c7z020/system-device-tree-xlnx

export CUSTOM_SDT_REPO=/home/mimo3000n/petalinux_myd_c7z020/system-device-tree-xlnx

~/petalinux/2026.2/components/xsct/bin/sdtgen \
  -eval "set_dt_param -dir /home/mimo3000n/petalinux_myd_c7z020/sdt_out -xsa /home/mimo3000n/petalinux_myd_c7z020/hw/design_1_wrapper.xsa; generate_sdt"

ls -la ~/petalinux_myd_c7z020/sdt_out
```

**Terminal-Output:**
```text
Info: Detected Custom SDT repo path at /home/mimo3000n/petalinux_myd_c7z020/system-device-tree-xlnx Verifying...
Successfully sourced custom SDT Repo path.
INFO: /home/mimo3000n/petalinux_myd_c7z020/sdt_out exists. Files will be overwritten
INFO: [Hsi 55-2053] elapsed time for repository (/home/mimo3000n/petalinux/2026.2/components/xsct/data/embeddedsw) loading 0 seconds
total 8072
drwxr-xr-x 3 mimo3000n mimo3000n    4096 Nov 23 11:14 .
drwxr-xr-x 7 mimo3000n mimo3000n    4096 Nov 23 11:14 ..
-rw-r--r-- 1 mimo3000n mimo3000n 4045771 Nov 23 11:14 design_1_wrapper.bit
drwxr-xr-x 3 mimo3000n mimo3000n    4096 Nov 23 11:10 include
-rw-r--r-- 1 mimo3000n mimo3000n    6806 Nov 23 11:14 pcw.dtsi
-rw-r--r-- 1 mimo3000n mimo3000n       7 Nov 23 11:14 pl.dtsi
-rw-r--r-- 1 mimo3000n mimo3000n  554999 Nov 23 11:14 ps7_init.c
-rw-r--r-- 1 mimo3000n mimo3000n    3796 Nov 23 11:14 ps7_init.h
-rw-r--r-- 1 mimo3000n mimo3000n 3008860 Nov 23 11:14 ps7_init.html
-rw-r--r-- 1 mimo3000n mimo3000n   36743 Nov 23 11:14 ps7_init.tcl
-rw-r--r-- 1 mimo3000n mimo3000n  555616 Nov 23 11:14 ps7_init_gpl.c
-rw-r--r-- 1 mimo3000n mimo3000n    4414 Nov 23 11:14 ps7_init_gpl.h
-rw-r--r-- 1 mimo3000n mimo3000n    2979 Nov 23 11:14 system-top.dts
-rw-r--r-- 1 mimo3000n mimo3000n   14065 Nov 23 11:10 zynq-7000.dtsi
```

---

## 5. PetaLinux-Projekt erstellen & SDT importieren

```bash
cd ~/petalinux_myd_c7z020
petalinux-create project --template zynq --name myd_c7z020_v2
cd myd_c7z020_v2

petalinux-config --get-hw-description=../sdt_out/
```

Im sich öffnenden **`System Configuration` (`menuconfig`)** folgende Einstellungen prüfen bzw. setzen:
* **Subsystem Hardware Settings → Serial Settings:** `uart1` (`115200` Baud)
* **Subsystem Hardware Settings → Ethernet Settings:** `gem0` (`Obtain IP address automatically` aktiviert)
* **Subsystem Hardware Settings → SD/SDIO Settings:** `sdhci0` (TF-Karte)
* **Image Packaging Configuration → Root filesystem type:** **`EXT4 (SD/eMMC/SATA/USB)`** (Device node: `/dev/mmcblk0p2`)
* Mit **Save** speichern und mit **Exit** verlassen.

**Terminal-Output nach dem Speichern:**
```text
[INFO] Getting hardware description
[INFO] Extracting yocto SDK to components/yocto. This may take time!
***** Gen Machine Conf v2026.2
[INFO] Using HW file: /home/mimo3000n/petalinux_myd_c7z020/myd_c7z020_v2/project-spec/hw-description/system-top.dts
[INFO] Getting Platform info from HW file
[INFO] Generating Kconfig for the project
[INFO] Menuconfig project
[INFO] Generating kconfig for rootfs
[INFO] Silentconfig rootfs
[INFO] Generating configuration files
[INFO] Generating cortex-a9 Linux configuration [ None ]
[INFO] Generating cortex-a9 baremetal configuration for FSBL
[INFO] Adding user layers
[INFO] Generating machine conf file
[INFO] Generating plnxtool conf file
[INFO] Generating workspace directory
NOTE: Starting bitbake server...
NOTE: Started PRServer with DBfile: /home/mimo3000n/petalinux_myd_c7z020/myd_c7z020_v2/build/cache/prserv.sqlite3, Address: 127.0.0.1:34123, PID: 5406
INFO: Specified workspace already set up, leaving as-is
INFO: Enabling workspace layer in bblayers.conf
[INFO] Successfully configured project
```

*(Optional – Verifizierung der Device-Tree-Knoten im generierten SDT:)*
```bash
grep -E "gem0:|ethernet0:|sdhci1:|mmc1:|usb0:" project-spec/hw-description/pcw.dtsi project-spec/hw-description/zynq-7000.dtsi
```
```text
project-spec/hw-description/zynq-7000.dtsi:             gem0: ethernet@e000b000 {
project-spec/hw-description/zynq-7000.dtsi:             sdhci1: mmc@e0101000 {
project-spec/hw-description/zynq-7000.dtsi:             usb0: usb@e0002000 {
```

---

## 6. Board-spezifische Device-Tree-Anpassungen (`system-user.dtsi`)

Konfiguration für den **Motorcomm YT8531SH** Gigabit-Ethernet-PHY (`U20`, Reset `MIO 7`), die **4 GB eMMC** (`U8` auf `sdhci1`) und **USB0** (`U12`):

```bash
cat << 'EOF' > project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
/include/ "system-conf.dtsi"
/ {
    model = "MYIR MYD-C7Z020-V2";
    compatible = "myir,myd-c7z020-v2", "xlnx,zynq-7000";
};

/* Gigabit Ethernet: Motorcomm YT8531SH (U20) auf MIO 16..27, Reset MIO 7 */
&gem0 {
    status = "okay";
    phy-mode = "rgmii-id";
    phy-handle = <&ethernet_phy>;

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;
        ethernet_phy: ethernet-phy@0 {
            compatible = "ethernet-phy-id4f51.e91b", "ethernet-phy-ieee802.3-c22";
            reg = <0>;
            reset-gpios = <&gpio0 7 1>; /* GPIO_ACTIVE_LOW */
            reset-assert-us = <10000>;
            reset-deassert-us = <50000>;
            motorcomm,clk-out-frequency-hz = <125000000>;
        };
    };
};

/* On-SOM 4GB eMMC (U8) auf SD1 (MIO 10..15) */
&sdhci1 {
    status = "okay";
    non-removable;
    bus-width = <4>;
    disable-wp;
};

/* USB 2.0 ULPI PHY (U12) auf USB0, Reset MIO 46 */
&usb0 {
    status = "okay";
    dr_mode = "host";
};
EOF
```

---

## 7. Kernel-Treiber aktivieren (`CONFIG_MOTORCOMM_PHY`)

```bash
petalinux-config -c kernel
```

**Terminal-Output beim Starten von Kernel-`menuconfig`:**
```text
***** Gen Machine Conf v2026.2
[INFO] Using HW file: /home/mimo3000n/petalinux_myd_c7z020/myd_c7z020_v2/project-spec/hw-description/system-top.dts
[INFO] Getting Platform info from HW file
[INFO] Silentconfig project
[INFO] Silentconfig rootfs
[INFO] Generating configuration files
[INFO] Generating workspace directory
NOTE: Starting bitbake server...
NOTE: Started PRServer with DBfile: /home/mimo3000n/petalinux_myd_c7z020/myd_c7z020_v2/build/cache/prserv.sqlite3, Address: 127.0.0.1:37855, PID: 5585
INFO: Specified workspace already set up, leaving as-is
[INFO] Configuring: kernel
[INFO] bitbake virtual/kernel -c cleansstate
NOTE: Started PRServer with DBfile: /home/mimo3000n/petalinux_myd_c7z020/myd_c7z020_v2/build/cache/prserv.sqlite3, Address: 127.0.0.1:38601, PID: 5659
WARNING: You are running bitbake under WSLv2, this works properly but you should optimize your VHDX file eventually to avoid running out of storage space
Loading cache: 100% |
Loaded 0 entries from dependency cache.
Parsing recipes: 11% |################                                                                                                      | ETA:  0:13:11
```

Im sich anschließend öffnenden Kernel-`menuconfig` aktivieren:
* **Device Drivers → Network device support → PHY Device support and infrastructure → Motorcomm PHYs** → mit **`Y`** (`[*]`) fest einkompilieren.
* Mit **Save** speichern und **Exit** wählen.

---

## 8. Gesamtsystem bauen & Boot-Images für TF-Karte erzeugen

```bash
petalinux-build
petalinux-package boot --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/system.bit --u-boot --force
```

Anschließend liegen unter `images/linux/` alle benötigten Dateien für die zwei Partitionen der microSD-Karte bereit:
* **Partition 1 (`FAT32`, Boot):** `BOOT.BIN`, `boot.scr`, `image.ub`
* **Partition 2 (`EXT4`, RootFS):** Inhalt von `rootfs.tar.gz` (oder `rootfs.ext4`)
