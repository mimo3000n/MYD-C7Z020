# MYD-C7Z020-V2: Inbetriebnahme & Troubleshooting (`07_freq_monitor`)

Diese Dokumentation fasst alle Schritte zur Inbetriebnahme des AXI4-Lite-Frequenzmonitors (`axi_freq_monitor`) und des Python-Web-Dashboards (`fclk_dashboard.py`) unter **PetaLinux 2026.2** auf dem **MYIR MYD-C7Z020-V2 (Zynq-7000)** zusammen — inklusive der Root-Cause-Analyse der aufgetretenen Messfehler.

---

## 1. Systemübersicht & Register-Map

* **Plattform:** MYD-C7Z020-V2 (Zynq-7020), PetaLinux 2026.2 mit persistentem `ext4`-RootFS (`/dev/mmcblk0p2`).
* **Netzwerk:** Statische IP `192.168.50.2/24` (Interface `enxaad1bb1fa88b` durch *Predictable Network Interface Names*).
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

---

## 2. Netzwerk auf dem Board einrichten

Da PetaLinux 2026.2 *Predictable Network Interface Names* verwendet, heißt die Netzwerkschnittstelle (abgeleitet von der MAC-Adresse `AA:D1:BB:1F:A8:8B`) **`enxaad1bb1fa88b`** statt `eth0`.

### Sofortige Aktivierung (über UART-Konsole)
```bash
sudo ip addr add 192.168.50.2/24 dev enxaad1bb1fa88b
sudo ip link set enxaad1bb1fa88b up
```

### Persistente Speicherung in `/etc/network/interfaces`
```bash
cat << 'EOF' | sudo tee /etc/network/interfaces
auto lo
iface lo inet loopback

auto enxaad1bb1fa88b
iface enxaad1bb1fa88b inet static
    address 192.168.50.2
    netmask 255.255.255.0
    gateway 192.168.50.1
EOF
```

*Verifizierung vom Host (WSL2):*
```bash
ping -c 3 192.168.50.2
```

---

## 3. Bitstream & Dashboard übertragen (WSL2 $\rightarrow$ Board)

Im frischen RootFS existiert `/lib/firmware` noch nicht und muss vor dem `scp`-Transfer angelegt werden:

```bash
# 1. Zielverzeichnis auf dem Board anlegen
ssh -o StrictHostKeyChecking=accept-new root@192.168.50.2 "mkdir -p /lib/firmware"

# 2. Bitstream und Python-Dashboard kopieren
scp ~/FPGA/myd_c7z020_v2/07_freq_monitor/output/freq_mon.bit.bin root@192.168.50.2:/lib/firmware/freq_mon.bit.bin
scp ~/FPGA/myd_c7z020_v2/07_freq_monitor/sw/fclk_dashboard.py root@192.168.50.2:/home/root/
```

---

## 4. Bitstream laden & Referenztakt kalibrieren (auf dem Board)

### 4.1 Bitstream über den Linux FPGA-Manager laden
```bash
echo 0 > /sys/class/fpga_manager/fpga0/flags
echo freq_mon.bit.bin > /sys/class/fpga_manager/fpga0/firmware
dmesg | tail -n 5
```
*Erwartete Ausgabe in `dmesg`:*
```text
fpga_manager fpga0: writing freq_mon.bit.bin to Xilinx Zynq FPGA Manager
```

### 4.2 Referenztakt `FCLK_CLK2` auf exakt $50{,}0\text{ MHz}$ setzen
Nach dem Booten steht `FPGA2_CLK_CTRL` (`0xF8000190`) auf `0x00101800` (Divisor 24 $\rightarrow$ $41{,}67\text{ MHz}$). Für eine exakte $1{,}000000\text{ s}$-Torzeit muss Divisor 20 (`0x00101400` $\rightarrow$ $50{,}0\text{ MHz}$) gesetzt werden:

```bash
devmem 0xF8000008 32 0x0000DF0D   # SLCR_UNLOCK
devmem 0xF8000190 32 0x00101400   # FCLK_CLK2 = 1000 MHz / 20 = 50.0 MHz
devmem 0xF8000004 32 0x0000767B   # SLCR_LOCK
```

### 4.3 Hardware-Register direkt verifizieren
```bash
devmem 0x43C0000C   # Erwartet: 0xDEADBEEF
sleep 2
devmem 0x43C00000   # Erwartet bei 100 MHz: 0x05F5E100 (100.000.000 Hz)
devmem 0x43C00004   # Erwartet bei 100 MHz: 0x05F5E100 (100.000.000 Hz)
```

---

## 5. Web-Dashboard starten

```bash
cd /home/root
python3 fclk_dashboard.py
```

Das Dashboard ist anschließend im Browser unter **`http://192.168.50.2:8080`** erreichbar und zeigt für `FCLK_CLK0` und `FCLK_CLK1` die **PL IST-MESSUNG** (`100.0000 MHz`) live neben dem **SLCR SOLL-TAKT** an.

---

## 6. Fehleranalyse (Lessons Learned)

Während der Inbetriebnahme traten zwei klassische Zynq-7000-Fehlerbilder nacheinander auf:

### Problem 1: PL-Messung liefert `0.0000 MHz` trotz blauer `DONE`-LED
* **Symptome:**
  * Die blaue `DONE`-LED auf dem MYC-C7Z020-V2 leuchtete bereits.
  * SLCR-Register waren korrekt freigegeben (`FPGA_RST_CTRL = 0x0`, `LVL_SHFTR_EN = 0xF`, `FPGAx_THR_CTRL = 0x0`).
  * Dennoch lieferten alle Register `0x43C00000` bis `0x43C0000C` den Wert `0x00000000` (insbesondere **nicht** `0xDEADBEEF` an `0x43C0000C`).
* **Ursache:** Die blaue `DONE`-LED stammte noch von einem vorherigen Bitstream aus der `BOOT.BIN`. Der neue Bitstream `freq_mon.bit.bin` war noch nicht über den FPGA-Manager geladen worden (erkennbar am fehlenden `fpga_manager`-Eintrag in `dmesg`).
* **Lösung:** Explizites Laden via `/sys/class/fpga_manager/fpga0/firmware` und Kontrolle über `dmesg` sowie den Magic-Value `0xDEADBEEF` an `0x43C0000C`.

### Problem 2: Gemessene Frequenz ist exakt $+20\,\%$ zu hoch (`120 MHz` statt `100 MHz`)
* **Symptome:**
  * Nach dem Laden des Bitstreams lieferte `0x43C0000C` korrekt `0xDEADBEEF`.
  * Beide Zähler (`0x43C00000` und `0x43C00004`) zeigten jedoch `0x07270E00` ($120.000.000\text{ Hz}$) bei einem Soll-Takt von $100\text{ MHz}$.
* **Ursache:** Im HDL-Modul `dual_freq_counter` ist die Torzeit fest auf `REF_CLK_HZ = 50_000_000` Takte von `FCLK_CLK2` ausgelegt. Nach dem Linux-Boot stand `FPGA2_CLK_CTRL` (`0xF8000190`) jedoch auf `0x00101800` (IO-PLL = $1000\text{ MHz}$, `DIVISOR0 = 24`, `DIVISOR1 = 1`):
  $$f_{\text{FCLK2}} = \frac{1000\text{ MHz}}{24} \approx 41{,}67\text{ MHz}$$
  Dadurch verlängerte sich die reale Torzeit um genau $20\,\%$:
  $$T_{\text{Gate}} = \frac{50.000.000}{41{,}6667\text{ MHz}} = 1{,}2\text{ s} \implies 100\text{ MHz} \cdot 1{,}2\text{ s} = 120.000.000\text{ Flanken}$$
* **Lösung:** Umstellen von `FPGA2_CLK_CTRL` (`0xF8000190`) auf `0x00101400` (`DIVISOR0 = 20`, `DIVISOR1 = 1` $\rightarrow$ exakt $50{,}0\text{ MHz}$). Danach liefern beide Register exakt `0x05F5E100` ($100.000.000\text{ Hz}$).

---

## 7. Kompaktes Startskript (für den Betrieb nach Reboot)

Um alle Schritte nach einem Neustart in einem Zug auszuführen, kann folgendes Skript unter `/home/root/start_freq_monitor.sh` abgelegt werden:

```bash
#!/bin/sh
set -e

# 1. Bitstream laden
echo 0 > /sys/class/fpga_manager/fpga0/flags
echo freq_mon.bit.bin > /sys/class/fpga_manager/fpga0/firmware

# 2. SLCR entsperren, Level-Shifter & Resets sicherstellen, FCLK_CLK2 = 50 MHz setzen
devmem 0xF8000008 32 0x0000DF0D
devmem 0xF8000900 32 0x0000000F
devmem 0xF8000240 32 0x00000000
devmem 0xF8000190 32 0x00101400
devmem 0xF8000004 32 0x0000767B

# 3. AXI-Verbindung prüfen
MAGIC=$(devmem 0x43C0000C)
echo "AXI Magic Check (0x43C0000C): $MAGIC"

# 4. Dashboard starten
exec python3 /home/root/fclk_dashboard.py
```

Ausführbar machen mit:
```bash
chmod +x /home/root/start_freq_monitor.sh
```