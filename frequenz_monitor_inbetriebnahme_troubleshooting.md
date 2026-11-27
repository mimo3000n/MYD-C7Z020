# MYD-C7Z020-V2: Inbetriebnahme & Troubleshooting (`07_freq_monitor`)

Diese Dokumentation fasst alle Schritte zur Inbetriebnahme des AXI4-Lite-Frequenzmonitors (`axi_freq_monitor`) und des Python-Web-Dashboards (`fclk_dashboard.py`) unter **PetaLinux 2026.2** auf dem **MYIR MYD-C7Z020-V2 (Zynq-7000)** zusammen — inklusive der dauerhaften Netzwerkkonfiguration, des Backup-/Restore-Workflows und der Root-Cause-Analyse aller aufgetretenen Hardware- und Software-Fehler.

## 1. Systemübersicht & Register-Map

* **Plattform:** MYD-C7Z020-V2 (Zynq-7020), PetaLinux 2026.2 mit persistentem `ext4`-RootFS (`/dev/mmcblk0p2`) und SysVinit.

* **Netzwerk:** Statische IP `192.168.50.2/24` fest auf `eth0` (feste MAC `02:00:00:50:00:02`, `eudev`-Umbenennung deaktiviert).

* **PL-IP (`axi_freq_monitor`):**

  * **Basisadresse:** `0x43C00000` (4K-Adressbereich an `M_AXI_GP0`)

  * **AXI- & Referenztakt (`s_axi_aclk` / `ref_clk`):** `FCLK_CLK2` (erwartet exakt $50\text{ MHz}$, `REF_CLK_HZ = 50_000_000`)

  * **Messeingänge:** `clk_in_0` = `FCLK_CLK0`, `clk_in_1` = `FCLK_CLK1`

* **AXI4-Lite Register-Offsets:**

| Offset | Adresse | Inhalt | Beschreibung |
| :--- | :--- | :--- | :--- |
| `0x00` | `0x43C00000` | `freq_hz_0` | Gemessene Frequenz von `FCLK_CLK0` in Hz (z. B. `0x05F5E100` = $100\text{ MHz}$) |
| `0x04` | `0x43C00004` | `freq_hz_1` | Gemessene Frequenz von `FCLK_CLK1` in Hz |
| `0x08` | `0x43C00008` | `valid_latch` | Bit 0 = `1` bei neuem Messwert (Clear-on-Read) |
| `0x0C` | `0x43C0000C` | Magic Value | Fester Kennwert **`0xDEADBEEF`** zur Bus-Verifizierung |

## 2. Netzwerk dauerhaft auf `eth0` fixieren

Standardmäßig generiert das Board ohne feste MAC in U-Boot/Device Tree bei jedem Start eine zufällige MAC-Adresse, wodurch `eudev` das Interface bei jedem Reboot anders benennt (`enx<MAC>`). Mit zwei Schritten bleibt das Interface dauerhaft auf **`eth0`** mit fester IP und MAC:

### 2.1 `eudev`-Umbenennung (`enx...`) deaktivieren

```bash
mkdir -p /etc/udev/rules.d
ln -sf /dev/null /etc/udev/rules.d/80-net-name-slot.rules
```

### 2.2 Statische IP & feste MAC in `/etc/network/interfaces` eintragen

```bash
cat << 'EOF' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.50.2
    netmask 255.255.255.0
    hwaddress ether 02:00:00:50:00:02
EOF
```

*Verifizierung nach Kaltstart:*

```bash
ip -br addr show eth0
# Erwartet: eth0   UP   192.168.50.2/24 fe80::ff:fe50:2/64
```

## 3. Bitstream & Dashboard übertragen (WSL2 $\rightarrow$ Board)

Im frischen RootFS existiert `/lib/firmware` noch nicht und muss vor dem `scp`-Transfer angelegt werden:

```bash
# 1. Zielverzeichnis auf dem Board anlegen
ssh -o StrictHostKeyChecking=accept-new root@192.168.50.2 "mkdir -p /lib/firmware"

# 2. Bitstream und Python-Dashboard kopieren
scp ~/FPGA/myd_c7z020_v2/07_freq_monitor/output/freq_mon.bit.bin root@192.168.50.2:/lib/firmware/freq_mon.bit.bin
scp ~/FPGA/myd_c7z020_v2/07_freq_monitor/sw/fclk_dashboard.py root@192.168.50.2:/home/root/
```

## 4. Kritischer 32-Bit-Patch für `fclk_dashboard.py`

In Python schreibt eine `mmap`-Slice-Zuweisung (`mem[offset:offset + 4] = struct.pack(...)`) intern häufig byteweise (8-Bit). Die Zynq-SLCR-Register (`SLCR_UNLOCK`, `FPGAx_CLK_CTRL`) verwerfen Byte-Zugriffe jedoch stillschweigend und verlangen zwingend **echte 32-Bit-Word-Zugriffe**.

Daher müssen `_read32` und `_write32` über `ctypes.c_uint32.from_buffer` umgesetzt werden:

```python
import ctypes

def _read32(self, mem, offset):
    return ctypes.c_uint32.from_buffer(mem, offset).value

def _write32(self, mem, offset, val):
    ctypes.c_uint32.from_buffer(mem, offset).value = val & 0xFFFFFFFF
```

*Idempotenter Patch-Befehl direkt auf dem Board:*

```bash
python3 -c '
p = "/home/root/fclk_dashboard.py"
s = open(p).read()
if "import ctypes" not in s:
    s = "import ctypes\n" + s
s = s.replace(
    "return struct.unpack(\"<I\", mem[offset:offset + 4])[0]",
    "return ctypes.c_uint32.from_buffer(mem, offset).value"
)
s = s.replace(
    "mem[offset:offset + 4] = struct.pack(\"<I\", val & 0xFFFFFFFF)",
    "ctypes.c_uint32.from_buffer(mem, offset).value = val & 0xFFFFFFFF"
)
open(p, "w").write(s)
'
```

## 5. Startskript `/home/root/start_freq_monitor.sh`

Das Skript lädt den Bitstream über den Linux FPGA-Manager, setzt `FCLK_CLK2` auf exakt $50{,}0\text{ MHz}$ (`DIVISOR0 = 20`), prüft die AXI4-Lite-Signatur (`0xDEADBEEF`) und startet anschließend das Web-Dashboard:

```bash
cat << 'EOF' > /home/root/start_freq_monitor.sh
#!/bin/sh
set -e

echo "[1/4] Lade Bitstream freq_mon.bit.bin..."
echo 0 > /sys/class/fpga_manager/fpga0/flags
echo freq_mon.bit.bin > /sys/class/fpga_manager/fpga0/firmware

echo "[2/4] Konfiguriere SLCR (Level-Shifter, Reset, FCLK_CLK2 = 50.0 MHz)..."
devmem 0xF8000008 32 0x0000DF0D   # SLCR_UNLOCK
devmem 0xF8000900 32 0x0000000F   # LVL_SHFTR_EN
devmem 0xF8000240 32 0x00000000   # FPGA_RST_CTRL
devmem 0xF8000190 32 0x00101400   # FCLK_CLK2 = 1000 MHz / 20 = 50.0 MHz
devmem 0xF8000004 32 0x0000767B   # SLCR_LOCK

echo "[3/4] Prüfe AXI4-Lite-Signatur (0x43C0000C)..."
SIG=$(devmem 0x43C0000C)
echo "      Signatur: $SIG"
if [ "$SIG" != "0xDEADBEEF" ]; then
    echo "FEHLER: Unerwartete Signatur ($SIG statt 0xDEADBEEF)!" >&2
    exit 1
fi

echo "[4/4] Starte fclk_dashboard.py auf Port 8080..."
exec python3 /home/root/fclk_dashboard.py
EOF
chmod +x /home/root/start_freq_monitor.sh
```

### Starten im Hintergrund (empfohlen)

```bash
nohup /home/root/start_freq_monitor.sh > /var/log/freq_monitor.log 2>&1 &
sleep 2
cat /var/log/freq_monitor.log
```

Das Dashboard ist anschließend im Browser unter **`http://192.168.50.2:8080`** erreichbar.

## 6. Backup & Restore Workflow (RootFS-Overlay)

Falls das PetaLinux-RootFS neu gebaut und geflasht werden muss, lassen sich alle relevanten Anpassungen als einzelnes `.tar.gz`-Overlay sichern und wiederherstellen. `tar` erhält dabei sowohl die Ausführungsrechte (`+x`) als auch den Symlink auf `/dev/null`.

### Enthaltene Dateien im Overlay

| Pfad auf dem Board | Zweck |
| :--- | :--- |
| `/etc/udev/rules.d/80-net-name-slot.rules` | Symlink auf `/dev/null` (deaktiviert `enx...`-Umbenennung $\rightarrow$ festes `eth0`) |
| `/etc/network/interfaces` | Statische IP `192.168.50.2/24` & feste MAC `02:00:00:50:00:02` |
| `/home/root/start_freq_monitor.sh` | Automatisiertes Startskript (Bitstream, SLCR-Fix, Signaturprüfung, Dashboard) |
| `/home/root/fclk_dashboard.py` | Web-Dashboard inkl. `ctypes`-32-Bit-SLCR-Fix |
| `/lib/firmware/freq_mon.bit.bin` | PL-Bitstream für den Linux FPGA-Manager |

### 6.1 Backup erstellen (Board $\rightarrow$ WSL2)

1. **Archiv auf dem Board packen und Inhalt prüfen:**
   ```bash
   tar -czvf /tmp/myd_c7z020_rootfs_overlay.tar.gz \
     -C / \
     etc/udev/rules.d/80-net-name-slot.rules \
     etc/network/interfaces \
     home/root/start_freq_monitor.sh \
     home/root/fclk_dashboard.py \
     lib/firmware/freq_mon.bit.bin

   # Inhalt, Rechte und Symlink verifizieren
   tar -tzvf /tmp/myd_c7z020_rootfs_overlay.tar.gz
   ```

2. **Archiv in WSL2 sichern (im Projekt- oder Backup-Ordner ausführen):**
   ```bash
   scp root@192.168.50.2:/tmp/myd_c7z020_rootfs_overlay.tar.gz ~/FPGA/myd_c7z020_v2/07_freq_monitor/
   ```

3. **Temporäre Datei auf dem Board aufräumen (optional):**
   ```bash
   ssh root@192.168.50.2 "rm -f /tmp/myd_c7z020_rootfs_overlay.tar.gz"
   ```

### 6.2 Restore auf ein frisch geflashtes RootFS (WSL2 $\rightarrow$ Board)

1. **Temporäres Netzwerk über UART aktivieren (falls das frische RootFS noch keine feste IP hat):**
   ```bash
   IFACE=$(ls /sys/class/net | grep -vE '^(lo|sit0)$' | head -n 1)
   ip link set "$IFACE" up
   ip addr add 192.168.50.2/24 dev "$IFACE"
   ```

2. **Overlay-Archiv aus WSL2 auf das Board kopieren:**
   ```bash
   scp -o StrictHostKeyChecking=accept-new \
     ~/FPGA/myd_c7z020_v2/07_freq_monitor/myd_c7z020_rootfs_overlay.tar.gz \
     root@192.168.50.2:/tmp/
   ```

3. **Auf dem Board entpacken und sauber herunterfahren:**
   ```bash
   tar -xzvf /tmp/myd_c7z020_rootfs_overlay.tar.gz -C /
   rm -f /tmp/myd_c7z020_rootfs_overlay.tar.gz
   sync
   poweroff
   ```

4. **Kaltstart durchführen & verifizieren:**
   Stromversorgung kurz (ca. 5 Sekunden) trennen und Board wieder einschalten (wegen YT8531-PHY-Reset). Danach prüfen:
   ```bash
   ip -br addr show eth0
   nohup /home/root/start_freq_monitor.sh > /var/log/freq_monitor.log 2>&1 &
   sleep 2 && cat /var/log/freq_monitor.log
   ```
   *Erwartet:* `eth0` ist direkt `UP` mit `192.168.50.2/24`, das Log meldet `Signatur: 0xDEADBEEF` und das Dashboard läuft unter `http://192.168.50.2:8080`.

## 7. Fehleranalyse (Lessons Learned)

### Problem 1: PL-Messung liefert `0.0000 MHz` trotz blauer `DONE`-LED

* **Ursache:** Die blaue `DONE`-LED stammte noch von einem vorherigen Bitstream aus der `BOOT.BIN`. Der neue Bitstream `freq_mon.bit.bin` war noch nicht über den FPGA-Manager geladen worden (`0x43C0000C` lieferte `0x00000000` statt `0xDEADBEEF`).

* **Lösung:** Explizites Laden über `/sys/class/fpga_manager/fpga0/firmware` und Signaturprüfung auf `0xDEADBEEF`.

### Problem 2: Gemessene Frequenz ist exakt $+20\,\%$ zu hoch (`120 MHz` statt `100 MHz`)

* **Ursache:** Im HDL-Modul `dual_freq_counter` ist die Torzeit fest auf `REF_CLK_HZ = 50_000_000` Takte von `FCLK_CLK2` ausgelegt. Nach dem Linux-Boot stand `FPGA2_CLK_CTRL` (`0xF8000190`) jedoch auf `0x00101800` (IO-PLL = $1000\text{ MHz}$, `DIVISOR0 = 24`, `DIVISOR1 = 1`):

  $$
  f_{\text{FCLK2}} = \frac{1000\text{ MHz}}{24} \approx 41{,}67\text{ MHz}
  $$

  Dadurch dauerte die Torzeit $1{,}2\text{ s}$ statt $1{,}0\text{ s}$, was bei $100\text{ MHz}$ genau $120.000.000$ gezählte Flanken ergab.

* **Lösung:** Umstellen von `FPGA2_CLK_CTRL` (`0xF8000190`) auf `0x00101400` (`DIVISOR0 = 20` $\rightarrow$ exakt $50{,}0\text{ MHz}$).

### Problem 3: SLCR-Schreibbuttons im Dashboard ändern Register nicht (`0x00100A00` bleibt stehen)

* **Ursache:** `devmem` konnte `0xF8000170` problemlos ändern, `fclk_dashboard.py` hingegen nicht. Ursache war die Python-`mmap`-Slice-Zuweisung (`mem[offset:offset+4] = struct.pack(...)`), die byteweise schreibt und vom Zynq-SLCR verworfen wird.

* **Lösung:** Umstellung auf echte 32-Bit-Zugriffe mit `ctypes.c_uint32.from_buffer(mem, offset).value` (siehe Abschnitt 4).

### Problem 4: `Reboot failed -- System halted` & `Could not attach PHY (-110)` nach Warmstart

* **Ursache:** Beim Ausführen von `reboot` / `shutdown -r now` hängt der Kernel die Dateisysteme sauber aus (`re-mounted ... ro`), löst aber keinen PS-Reset aus (`Reboot failed -- System halted`). Gleichzeitig versetzt Linux den externen **YT8531-Ethernet-PHY** in den Power-Down-Modus. Ein anschließender Druck auf den PS-Reset-Taster setzt den PHY nicht sauber zurück, sodass der `macb`-Treiber beim nächsten Boot mit `Could not attach PHY (-110)` (`ETIMEDOUT`) abbricht.

* **Lösung:** Nach `poweroff` / `reboot` einen echten **Kaltstart** durchführen (Netzteil für ca. 5 Sekunden trennen), damit der YT8531-PHY sauber neu initialisiert wird.
