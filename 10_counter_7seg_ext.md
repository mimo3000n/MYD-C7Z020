# 10_counter_7seg_ext — 4-Bit-Zähler mit 7-Segment-Anzeige & Register `0x0C` (MYD-C7Z020-V2)

Dieses Projekt erweitert den 4-Bit-Zähler mit 7-Segment-Dekodierung (`09_counter_7seg`) auf dem **MYD-C7Z020-V2** (Xilinx Zynq-7020, Board-IP `192.168.50.2`) um ein zusätzliches AXI-Lite-Register `0x0C` (`0x43C0000C`), eine Single-Step-Funktion, konfigurierbare Zählgeschwindigkeiten, einen 8-Bit-Umlaufzähler (`lap_count`) sowie eine Web-Oberfläche (`Port 8080`) mit visueller SVG-7-Segment-Anzeige.

Alle Dateien liegen isoliert im eigenen Verzeichnis `10_counter_7seg_ext/`, sodass bestehende Projekte wie `09_counter_7seg` unverändert bleiben.

---

## 1. Systemarchitektur & Funktionsweise

Das Gesamtsystem besteht aus drei Ebenen:

1. **Programmable Logic (PL / FPGA — Zynq-7020):**
   - `rtl/counter_4bit.v`: Kernlogik des 4-Bit-Zählers (`0x0`..`0xF`). Unterstützt Dauerbetrieb (`enable`), Einzelimpulse (`step_trig`), Soft-Reset (`soft_reset`) sowie eine wählbare Taktfrequenz (`speed_sel`). Beim Überlauf von `4'hF` nach `4'h0` wird der 8-Bit-Umlaufzähler (`lap_count`) inkrementiert (und über `lap_clear` zurückgesetzt).
   - `rtl/hex_to_7seg.v`: Kombinatorischer 4-Bit-auf-7-Segment-Dekoder mit dem Ausgangsport `seg_out` (`[6:0]`, Bitreihenfolge `{g, f, e, d, c, b, a}`).
   - `rtl/axi_counter_4bit.v`: AXI4-Lite-Slave-Schnittstelle (`0x43C00000`), die Zählerstand, 7-Segment-Maske und Steuer-/Statusregister auf der Busseite bereitstellt.
2. **Build- & Deployment-Pipeline (WSL2 + Windows Vivado 2026.2):**
   - Da Vivado 2026.2 und Bootgen unter Windows (`D:\AMDDesignTools\2026.2\Vivado\bin\vivado.bat` bzw. `bootgen.bat`) laufen, werden die Build-Dateien nach `/mnt/d/vivado_builds/10_counter_7seg_ext` synchronisiert, dort synthetisiert/implementiert (`myd_counter_7seg_proj`) und als Bitstream (`system_wrapper.bit` / `system_wrapper.bit.bin`) sowie Hardware-Plattform (`system.xsa`) in `out/` abgelegt.
   - Web-UI-Vorlagen aus `sw/web_ui/` (`index.html.in`, `style.css.in`, `dashboard.js.in`) werden nach `out/ui_assets/` kopiert und in `out/web_ui_bundle.h` als C-Byte-Arrays gebündelt.
3. **Processing System (PS / Linux auf `192.168.50.2`):**
   - Lädt bei Bedarf `/lib/firmware/counter_7seg_ext.bit.bin` über `/sys/class/fpga_manager/fpga0`.
   - Stellt über `counter_7seg_web` (C-Binary) oder `counter_7seg_web.py` (Python-Alternative) auf Port `8080` sowohl die Weboberfläche als auch die REST-Endpunkte `/api/status` und `/api/ctrl` bereit.

---

## 2. Verilog-RTL im Detail

### 2.1 `rtl/counter_4bit.v` — Zählerkern, Taktteiler, Single-Step & Umlaufzähler

Der Zählerkern arbeitet synchron mit dem 100-MHz-AXI-Takt (`S_AXI_ACLK`) und einem aktiven Low-Reset (`S_AXI_ARESETN`):

- **Programmierbare Zählfrequenz (`speed_sel`):** Über einen Taktteiler (Clock-Divider) wird aus den 100 MHz ein Zähltakt bzw. Freigabepuls (`tick`) für **1 Hz**, **2 Hz**, **5 Hz** oder **10 Hz** abgeleitet (`00` / `01` / `10` / `11`).
- **Betriebsmodi (`enable` vs. `step_trig`):**
  - Bei `enable == 1` zählt der 4-Bit-Zähler (`0x0` $\rightarrow$ `0xF` $\rightarrow$ `0x0`) automatisch im gewählten Takt weiter.
  - Bei `enable == 0` (Pause/Hold) bleibt der Zähler stehen. Ein geschriebener Single-Step-Impuls (`step_trig`) löst dann genau **einen** einzelnen Zählschritt (`+1` modulo 16) aus, ohne den Zähler dauerhaft zu starten.
- **Soft-Reset (`soft_reset`):** Setzt den 4-Bit-Zähler gezielt auf `0x0` zurück, ohne den gesamten AXI-Bus oder den Umlaufzähler ungewollt zu löschen.
- **8-Bit-Umlaufzähler (`lap_count` & `lap_clear`):** Immer wenn der 4-Bit-Zähler von `4'hF` (15) auf `4'h0` (0) überläuft, inkrementiert der 8-Bit-Umlaufzähler (`0..255`). Über `lap_clear` wird er auf `0` zurückgesetzt.

### 2.2 `rtl/hex_to_7seg.v` — Kombinatorischer 7-Segment-Decoder

Dieses Modul setzt den aktuellen 4-Bit-Zählerstand (`hex_in[3:0]`) rein kombinatorisch (`always @(*)`) in eine 7-Bit-Anzeigemaske (`seg_out[6:0]`) um:

- **Bit-Zuordnung (`Active-High`):** Die 7 Bits repräsentieren die Segmente `{g, f, e, d, c, b, a}` (`Bit 0 = a` oben bis `Bit 6 = g` Mitte).
- **Hex-Abdeckung (`0x0` bis `0xF`):** Alle 16 Werte eines 4-Bit-Zählers sind vollständig abgebildet — `0`–`9` als Ziffern und `A`–`F` als Hexadezimalzeichen (z. B. `0x0` $\rightarrow$ `7'b0111111` = `0x3F` = `63` dezimal, `0x1` $\rightarrow$ `7'b0000110` = `0x06` = `6` dezimal).
- **Direkte Bus-Bereitstellung:** Die berechnete 7-Bit-Maske liegt über das AXI-Register `0x08` vor, sodass C- und Python-Dashboard die Segmente ohne eigene Decodier-Tabelle darstellen können.

### 2.3 `rtl/axi_counter_4bit.v` — AXI4-Lite-Slave & Register-Interface

Dieses Modul verbindet den Zynq Processing System (`M_AXI_GP0`, 100 MHz) über ein AXI4-Lite-Slave-Interface (`C_S_AXI_ADDR_WIDTH = 4`, 4-Byte-Adressierung mit `awaddr[3:2]` / `araddr[3:2]`) mit `counter_4bit` und `hex_to_7seg`:

- **AXI4-Lite-Handshake:** Steuert die Standardkanäle `AWREADY`/`AWVALID`, `WREADY`/`WVALID`, `BVALID`/`BREADY` sowie `ARREADY`/`ARVALID` und `RVALID`/`RREADY` für Schreib- und Lesezugriffe aus Linux (`/dev/mem` bzw. `mmap`).
- **Steuer- und Statuslogik:** Wertet Schreibzugriffe auf `0x00` (`CTRL`) und `0x0C` (`EXT_CFG_STAT`) aus und erzeugt Einzeltakt- bzw. Auto-Clear-Impulse für `soft_reset`, `step_trig` und `lap_clear`, damit Steuerbits nicht dauerhaft aktiv bleiben. Beim Lesen liefert es je nach Wortadresse `enable` (`0x00`), `count` (`0x04`), `seg7` (`0x08`) sowie `speed_sel` und `lap_count` (`0x0C`).

---

## 3. AXI-Lite Register-Map (Basisadresse `0x43C00000`)

| Offset | Adresse | Name | Zugriff | Bit-Belegung & Funktion |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `0x43C00000` | `CTRL` | R/W | `[0]`: `enable` (`1` = freier Zählerlauf, `0` = Pause/Hold) |
| `0x04` | `0x43C00004` | `COUNT` | R | `[3:0]`: Aktueller 4-Bit-Zählerstand (`0`..`15` / `0x0`..`0xF`) |
| `0x08` | `0x43C00008` | `SEG7` | R | `[6:0]`: 7-Segment-Bitmaske `{g, f, e, d, c, b, a}` |
| `0x0C` | `0x43C0000C` | `EXT_CFG_STAT` | R/W | **Schreiben:** `[1:0]` `speed_sel` (`00`=1 Hz, `01`=2 Hz, `10`=5 Hz, `11`=10 Hz), `[2]` `soft_reset`, `[3]` `step_trig` (Single-Step), `[4]` `lap_clear` (Auto-Clear-Impuls)<br>**Lesen:** `[1:0]` `speed_sel`, `[15:8]` 8-Bit `lap_count` (Umlaufzähler) |

---

## 4. Verzeichnisstruktur

```text
10_counter_7seg_ext/
├── docs/
│   └── README.md               # Projekt- und Registerdokumentation
├── out/
│   ├── counter_7seg_ext.bit.bin
│   ├── system_wrapper.bit
│   ├── system.xsa
│   ├── counter_7seg_web        # Statisch kompiliertes ARM-Binary
│   ├── counter_7seg_web.py     # Python-Backend
│   ├── web_ui_bundle.h         # Eingebettete Web-UI für C-Binary
│   └── ui_assets/              # index.html, style.css, dashboard.js
├── rtl/
│   ├── counter_4bit.v
│   ├── hex_to_7seg.v
│   └── axi_counter_4bit.v
├── scripts/
│   ├── build.sh
│   └── deploy.sh
├── sw/
│   ├── counter_7seg_web.c
│   ├── counter_7seg_web.py
│   └── web_ui/
│       ├── index.html.in
│       ├── style.css.in
│       └── dashboard.js.in
└── vivado/
    └── build.tcl
```

---

## 5. Betrieb, API-Test & Web-Dashboard

### Deployment auf das Board (`192.168.50.2`)

```bash
cd ~/FPGA/myd_c7z020_v2/10_counter_7seg_ext
./scripts/deploy.sh --c --no-fpga   # C-Backend ohne erneutes Laden des Bitstreams
# bzw. mit Bitstream-Laden:
./scripts/deploy.sh --c
# bzw. Python-Backend:
./scripts/deploy.sh --py --no-fpga
```

> **Hinweis zu Embedded-Linux-Befehlen:** Falls auf dem Zielsystem weder `pkill` noch `killall` installiert ist, kann ein laufender `counter_7seg_web`-Prozess über `ps` und `kill <PID>` beendet werden, bevor der neue Server auf Port `8080` startet.

### API-Prüfung per `curl`

```bash
curl -s http://192.168.50.2:8080/api/status
echo
curl -s -X POST http://192.168.50.2:8080/api/ctrl \
  -H "Content-Type: application/json" \
  -d '{"enable":0,"step":1}'
echo
curl -s http://192.168.50.2:8080/api/status
echo
```

### Browser-Ansicht (`http://192.168.50.2:8080`)

Das Dashboard zeigt:
- Zählerstand in **Dezimal** und **Hexadezimal** sowie **Umläufe (`lap_count`)**.
- Eine **SVG-7-Segment-Anzeige** (`#seg7Svg`, Segmente `a`–`g`) synchron zur Bitmaske `SEG7` sowie den Laufstatus (`RUN` / `PAUSE`).
- Bedienelemente für **Start (Run)**, **Pause (Hold)**, **Single Step (+1)**, **Reset (0x0)**, **Lap Clear** und **Geschwindigkeit** (`1 Hz`, `2 Hz`, `5 Hz`, `10 Hz`).

---

## 6. Vorbereitung für den GitHub-Export

Vor dem Commit in ein Git-Repository empfiehlt sich folgende Bereinigung und Prüfung, damit keine temporären Build-Artefakte oder lokalen Pfade/Secrets ins Repository gelangen:

### `.gitignore` für `10_counter_7seg_ext`

```gitignore
# Temporäre Vivado- und Build-Ordner
vivado_out/
*.log
*.jou
*.str
*.pb
*.dir/
*.hw/
*.ip_user_files/

# Große generierte Hardware-Binärdateien (optional je nach Repo-Policy)
out/*.bit
out/*.bit.bin
out/*.xsa
```

### Export-Checkliste

```bash
EXT_DIR="$HOME/FPGA/myd_c7z020_v2/10_counter_7seg_ext"

# 1. Keine temporären Vivado-Logs/Journals oder Build-Artefakte im Quellbaum lassen
find "$EXT_DIR" -maxdepth 3 -ls

# 2. Prüfen, ob alle RTL-, SW-, Script- und UI-Quelldateien vorhanden sind
ls -la "$EXT_DIR/rtl" "$EXT_DIR/sw" "$EXT_DIR/sw/web_ui" "$EXT_DIR/scripts"

# 3. Sicherstellen, dass keine absoluten Nutzerpfade oder Zugangsdaten in Skripten liegen
grep -rn "/home/" "$EXT_DIR/scripts" "$EXT_DIR/sw" "$EXT_DIR/rtl" || true
```
