# MYIR MYD-C7Z020-V2: 4-Bit AXI4-Lite Counter & Embedded Webserver (PetaLinux 2026.2)

Vollständige Dokumentation und Backup-Anleitung für die Hardware-/Software-Inbetriebnahme eines benutzerdefinierten 4-Bit-AXI4-Lite-Zählers im PL-Fabric eines **Xilinx Zynq-7020 (MYIR MYD-C7Z020-V2)** unter **PetaLinux 2026.2** und **U-Boot 2026.01**, inklusive vollautomatischem Kaltstart (QSPI-Flash + SD-Karte), deterministischem Netzwerk-Interface (`eth0` mit `192.168.50.2`) und C-Webserver auf Port `8080`.

---

## 1. Systemarchitektur & Memory Map

* **SoC / Board:** Xilinx Zynq-7020 (Silicon v3.1) auf MYIR MYD-C7Z020-V2 (1 GiB DDR3, 32 MiB QSPI-Flash `n25q256a`, SD-Karte `mmc0`, eMMC `mmc1`)
* **Entwicklungsumgebung:** Windows 11 + WSL2 (Ubuntu 24.04, Mirrored Networking Mode), Vivado Batch Flow, PetaLinux 2026.2
* **PL-Takt & Bus-Interface:** `FCLK_CLK0` = `100 MHz`, PS-Master `M_AXI_GP0`
* **AXI4-Lite Basisadresse:** `0x43C00000` (Adressbereich: `4 KiB`, `0x43C00000 – 0x43C00FFF`)

| Offset | Physische Adresse | Zugriff | Register / Funktion |
| :--- | :--- | :--- | :--- |
| `0x00` | `0x43C00000` | R/W | **Control Register (`slv_reg0`):** Bit `0` = Enable (`1` = Zähler läuft mit 1 Hz, `0` = Stop) |
| `0x04` | `0x43C00004` | R | **Status / Counter Register:** Bits `[3:0]` = aktueller 4-Bit-Zählerstand (`0x0` – `0xF`) |

### Boot-Aufteilung (QSPI-Flash + SD-Karte)

1. **QSPI-Flash (`0x000000 – 0x5FFFFF`):** Enthält `BOOT.BIN` (`zynq_fsbl.elf` + PL-Bitstream `system.bit` + `u-boot.elf`).
2. **SD-Karte Partition 1 (`mmcblk0p1`, FAT32):** Enthält `boot.scr` und das FIT-Image `image.ub` (Linux-Kernel `6.18.40-xilinx` + Device Tree).
3. **SD-Karte Partition 2 (`mmcblk0p2`, ext4):** Root-Dateisystem inklusive `/root/counter_web`, SysVinit-Autostart `/etc/init.d/counter_web`, `udev`-Regel `/etc/udev/rules.d/70-persistent-net.rules` und statischer Netzwerkkonfiguration `/etc/network/interfaces`.

---

## 2. Repository-Struktur & Sicherung aller Projektartefakte

```text
.
├── README.md
├── SHA256SUMS.txt
├── vivado_batch/
│   ├── counter_4bit.v             # 4-Bit-Zählerlogik (100 MHz -> 1 Hz)
│   ├── axi_counter_4bit.v         # AXI4-Lite Slave Wrapper (0x43C00000)
│   └── system.xsa                 # Exportierte Hardware-Plattform
├── petalinux_dts/
│   ├── system-user.dtsi           # Bereinigter Device Tree (SD, eMMC, PHY, Clocks, MAC)
│   ├── pl.dtsi                    # Generierter PL-Device-Tree
│   ├── pcw.dtsi                   # PS-Peripherie-Konfiguration
│   ├── system-top.dts             # Top-Level Device Tree
│   └── system.bit                 # Aktueller FPGA-Bitstream
├── boot_images/
│   ├── BOOT.BIN                   # QSPI-Bootimage (FSBL + Bitstream + U-Boot)
│   ├── image.ub                   # Linux-Kernel + DTB FIT-Image
│   ├── boot.scr                   # U-Boot-Startskript
│   ├── zynq_fsbl.elf              # First Stage Boot Loader
│   └── u-boot.elf                 # U-Boot 2026.01 Binary
└── board_rootfs/
    ├── counter_web                # ARMv7 C-Webserver-Binary (Port 8080)
    ├── counter_web.init           # SysVinit-Dienst (/etc/init.d/counter_web)
    ├── 70-persistent-net.rules    # udev-Regel für deterministisches eth0-Interface
    └── network_interfaces         # Statische Netzwerkkonfiguration (192.168.50.2)
```

### Backup-Skript für WSL2 (alle Artefakte & Prüfsummen einsammeln)

```bash
BACKUP_DIR="$HOME/FPGA/myd_c7z020_v2/release_2026_11_axi_counter"
mkdir -p "$BACKUP_DIR"/{hdl,vivado_batch,petalinux_dts,boot_images,board_rootfs}

# 1. HDL & Vivado-Batch-Dateien sichern
cp -v ~/FPGA/myd_c7z020_v2/02_vivado_batch/{counter_4bit.v,axi_counter_4bit.v,system.xsa} "$BACKUP_DIR/vivado_batch/"
cp -v ~/FPGA/myd_c7z020_v2/01_counter_4bit/*.v "$BACKUP_DIR/hdl/" 2>/dev/null || true

# 2. PetaLinux Device-Tree & Hardware-Description sichern
PROJ="$HOME/petalinux_myd_c7z020/myd_c7z020_v2"
cp -v "$PROJ/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi" "$BACKUP_DIR/petalinux_dts/"
cp -v "$PROJ/project-spec/hw-description/"{pl.dtsi,pcw.dtsi,system-top.dts,system.bit} "$BACKUP_DIR/petalinux_dts/"

# 3. Boot-Images (QSPI + SD-Boot-Partition) sichern
cp -v "$PROJ/images/linux/"{BOOT.BIN,image.ub,boot.scr,zynq_fsbl.elf,u-boot.elf} "$BACKUP_DIR/boot_images/"

# 4. Webserver, Init-Skript, udev-Regel und Netzwerkkonfiguration vom laufenden Board sichern
scp root@192.168.50.2:/root/counter_web "$BACKUP_DIR/board_rootfs/"
scp root@192.168.50.2:/etc/init.d/counter_web "$BACKUP_DIR/board_rootfs/counter_web.init"
scp root@192.168.50.2:/etc/udev/rules.d/70-persistent-net.rules "$BACKUP_DIR/board_rootfs/"
scp root@192.168.50.2:/etc/network/interfaces "$BACKUP_DIR/board_rootfs/network_interfaces"

# 5. SHA256-Prüfsummen erzeugen
cd "$BACKUP_DIR"
sha256sum boot_images/* petalinux_dts/system.bit vivado_batch/system.xsa > SHA256SUMS.txt
```

---

## 3. Schritt-für-Schritt-Aufbau & Deployment

### 3.1 PetaLinux-Environment vorbereiten (WSL2)
Falls Miniforge/Conda aktiv ist, muss `(base)` vor dem Sourcen von PetaLinux deaktiviert werden, damit PetaLinux auf das eigene/System-Python zugreift:

```bash
conda deactivate
source /opt/pkg/petalinux/settings.sh   # Pfad an eigene Installation anpassen
cd ~/petalinux_myd_c7z020/myd_c7z020_v2
```

### 3.2 Bereinigte Device-Tree-Konfiguration (`system-user.dtsi`)
Vollständige Konfiguration in `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi` (inklusive Motorcomm YT8531SH PHY, SD-Karte `sdhci0`, On-SOM eMMC `sdhci1`, USB0, `clk_ignore_unused` und fester MAC-Adresse):

```dts
/include/ "system-conf.dtsi"
/ {
    model = "MYIR MYD-C7Z020-V2";
    compatible = "myir,myd-c7z020-v2", "xlnx,zynq-7000";

    aliases {
        mmc0 = &sdhci0;
        mmc1 = &sdhci1;
    };

    chosen {
        bootargs = "console=ttyPS0,115200 earlycon root=/dev/mmcblk0p2 rw rootwait clk_ignore_unused";
    };
};

/* Gigabit Ethernet: Motorcomm YT8531SH (U20) auf MIO 16..27, Reset MIO 7 */
&gem0 {
    status = "okay";
    phy-mode = "rgmii-id";
    phy-handle = <&ethernet_phy>;
    local-mac-address = [00 0a 35 00 00 00];

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

/* SD-Karte auf SD0 */
&sdhci0 {
    status = "okay";
    broken-cd;
    disable-wp;
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
```

### 3.3 Bitstream verpacken (`BOOT.BIN`) & Kernel-FIT-Image bauen (`image.ub`)
Sicherstellen, dass der aktuelle Vivado-Bitstream aus `project-spec/hw-description/system.bit` nach `images/linux/system.bit` kopiert wird, bevor `BOOT.BIN` verpackt wird:

```bash
cp -v project-spec/hw-description/system.bit images/linux/system.bit

petalinux-package boot \
  --fsbl images/linux/zynq_fsbl.elf \
  --fpga images/linux/system.bit \
  --u-boot images/linux/u-boot.elf \
  --force

petalinux-build -c device-tree
petalinux-build -c petalinux-image-minimal

sudo cp images/linux/BOOT.BIN /srv/tftp/BOOT.BIN
sudo cp images/linux/image.ub /srv/tftp/image.ub
```

### 3.4 QSPI-Flash in U-Boot beschreiben
Am `Zynq>`-Prompt per TFTP laden und in den QSPI-Flash schreiben:

```text
setenv ipaddr 192.168.50.2
setenv serverip 192.168.50.1
tftpboot 0x1000000 BOOT.BIN
sf probe 0 0 0
sf erase 0x0 0x600000
sf write 0x1000000 0x0 0x600000
reset
```

### 3.5 `image.ub` auf SD-Karte aktualisieren, `eth0` fixieren & Webserver-Autostart einrichten
Unter Linux auf dem Zynq-Board (Automount von `mmcblk0p1` zuvor sauber aushängen, damit kein paralleler Zugriff entsteht):

```bash
# 1. Boot-Image sauber auf FAT-Partition der SD-Karte aktualisieren
umount /dev/mmcblk0p1 2>/dev/null
fsck.vfat -a /dev/mmcblk0p1
mkdir -p /mnt/boot && mount /dev/mmcblk0p1 /mnt/boot
tftp -g -r image.ub 192.168.50.1 -l /mnt/boot/image.ub
sync
umount /mnt/boot

# 2. Ethernet-Interface unabhängig von der U-Boot-MAC deterministisch auf eth0 binden
cat << 'EOF' > /etc/udev/rules.d/70-persistent-net.rules
SUBSYSTEM=="net", ACTION=="add", KERNELS=="e000b000.ethernet", NAME="eth0"
EOF

cat << 'EOF' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.50.2
    netmask 255.255.255.0
EOF

# 3. SysVinit-Dienst für /root/counter_web registrieren
cat << 'EOF' > /etc/init.d/counter_web
#!/bin/sh
case "$1" in
  start)
    echo "Starting counter_web..."
    start-stop-daemon -S -b -m -p /var/run/counter_web.pid -x /root/counter_web
    ;;
  stop)
    echo "Stopping counter_web..."
    start-stop-daemon -K -p /var/run/counter_web.pid
    rm -f /var/run/counter_web.pid
    ;;
  restart)
    $0 stop
    sleep 1
    $0 start
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}"
    exit 1
    ;;
esac
exit 0
EOF

chmod +x /etc/init.d/counter_web
update-rc.d counter_web defaults
sync
reboot
```

### 3.6 Hardware- & Web-Verifikation
```bash
# Direkter Registertest via devmem
devmem 0x43C00000 32 1   # Zähler starten
devmem 0x43C00004        # Live-Zählerstand (0x0..0xF) auslesen
```
Weboberfläche im Browser aufrufen: `http://192.168.50.2:8080`

---

## 4. Wichtige Learnings & Troubleshooting-Erkenntnisse

1. **WSL2 Mirrored Networking & Windows Hyper-V-VM-Firewall für TFTP:**
   * Im WSL2-Mirrored-Modus (`192.168.50.1`) blockiert die Windows-Hyper-V-VM-Firewall eingehenden UDP-Verkehr auf Port 69 trotz deaktivierter Standard-Windows-Firewall.
   * **Lösung:** Eingehenden UDP-Port 69 explizit über `Set-NetFirewallHyperVVMSetting` in einer administrativen Windows-PowerShell freigeben.

2. **SD-Card-Detect (`MMC: no card present`), `mmc0`/`mmc1`-Vertauschung & Automount-Lock:**
   * Ohne `broken-cd` erkennt der SDHCI-Controller (`sdhci0`) die eingelegte SD-Karte auf dem MYD-C7Z020-V2 nicht. Zudem vertauschte der Linux-Kernel ohne explizite `aliases` die Nummerierung zwischen SD-Karte (`e0100000.mmc`) und eMMC (`e0101000.mmc`), was zu `Waiting for root device /dev/mmcblk0p2` führte.
   * Beim Aktualisieren von `image.ub` unter Linux bindet `eudev` `mmcblk0p1` automatisch unter `/run/media/BOOT-mmcblk0p1` ein – vor dem Mounten nach `/mnt/boot` oder Schreiben daher immer zuerst `umount /dev/mmcblk0p1` und `fsck.vfat -a /dev/mmcblk0p1` ausführen.

3. **Veralteter Bitstream in `images/linux/system.bit`:**
   * `petalinux-package boot` greift bei `--fpga` auf die explizit übergebene `.bit`-Datei zurück. Lag in `images/linux/system.bit` noch ein älterer Build, wurde trotz aktualisiertem Hardware-Export der alte Bitstream in `BOOT.BIN` verpackt.
   * **Lösung:** Vor jedem `petalinux-package boot` immer `project-spec/hw-description/system.bit` nach `images/linux/system.bit` kopieren bzw. direkt als `--fpga`-Quelle übergeben.

4. **AXI-Bus-Freeze bei `devmem 0x43C00000` durch Linux Clock-Gating (`clk: Disabling unused clocks`):**
   * Obwohl der FPGA-Manager `operating` meldete, die PS-PL-Level-Shifter (`0xF8000900` = `0xF`) aktiv und `FCLK_RESET0_N` (`0xF8000240` = `0x0`) freigegeben waren, schaltete der Linux-Kernel beim Booten `FCLK_CLK0` (`clkc 15`) ab (`clk: Disabling unused clocks`), weil kein Kernel-Treiber diesen Takt angefordert hatte. Jeder Zugriff auf `0x43C00000` ohne aktiven PL-Takt führte zum sofortigen AXI-Handshake-Freeze der CPU.
   * **Lösung:** `clk_ignore_unused` in `chosen { bootargs = "..."; };` aufnehmen (alternativ den PL-IP-Block über einen Kernel-Treiber wie `uio_pdrv_genirq` anbinden, der `clocks = <&clkc 15>` aktiv einschaltet).

5. **Zufalls-MAC-Adresse in U-Boot verhindert `enx...`-Interface-Start unter Linux:**
   * Wenn U-Boot mangels gespeichertem QSPI-Environment (`spi_flash_probe_bus_cs() failed`) bei jedem Boot eine Zufalls-MAC vergibt und diese vor dem Kernel-Start ins DTB (`ethernet0`) schreibt, benennt `eudev` das Interface bei jedem Neustart anders (`enx<random>`).
   * **Lösung:** Per `udev`-Regel `/etc/udev/rules.d/70-persistent-net.rules` (`KERNELS=="e000b000.ethernet", NAME="eth0"`) wird der Interface-Name dauerhaft auf `eth0` fixiert, sodass `/etc/network/interfaces` bei jedem Kaltstart zuverlässig `192.168.50.2` konfiguriert.

6. **Conda `(base)`-Environment kollidiert mit `petalinux-build`:**
   * Ein aktives Miniforge/Conda-Environment überschreibt den Python-Suchpfad, sodass PetaLinux-Skripte fehlschlagen.
   * **Lösung:** Vor `source .../settings.sh` immer `conda deactivate` ausführen.