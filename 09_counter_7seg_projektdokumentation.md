# 4-Bit Zähler & 7-Segment-Decoder (`09_counter_7seg`)

Dokumentation für das Zynq-7020-Projekt **`09_counter_7seg`** auf dem **MYD-C7Z020-V2** (`192.168.50.2`). Das Design kombiniert einen 1-Hz-Prescaler mit einem 4-Bit-Aufwärtszähler (`0x0`–`0xF`) und einem kombinatorischen 7-Segment-Decoder (`a`–`g`) in Programmable Logic (PL), angebunden über **AXI4-Lite** an den Zynq Processing System (`PS7`) und gesteuert bzw. visualisiert über ein Web-Dashboard auf Port `8080` (wahlweise in C oder Python).

## 1. Systemübersicht & Architektur

### Signal- und Datenfluss

```
+-------------------------------------------------------------------+
| Zynq-7020 PS7 (Linux @ 192.168.50.2)                              |
|  - counter_7seg_web (C) oder counter_7seg_web.py (Python) :8080   |
|  - Zugriff über /dev/mem auf AXI4-Lite Slave @ 0x43C00000         |
+--------------------------------+----------------------------------+
                                 | S_AXI4-Lite (32-Bit)
                                 v
+-------------------------------------------------------------------+
| Zynq-7020 PL: axi_counter_4bit (Basisadresse 0x43C00000)         |
|                                                                   |
|  [slv_reg0 @ 0x00] ---> enable -----------------+                |
|                                                  v                |
|                                        +-------------------+      |
|  S_AXI_ACLK -------------------------> |   counter_4bit    |      |
|  S_AXI_ARESETN (aktiv-low) ----------> | Prescaler + 4-Bit |      |
|                                        | Zähler (0x0..0xF) |      |
|                                        +---------+---------+      |
|                                                  | w_count_out    |
|                                                  | [3:0]          |
|  [slv_reg1 @ 0x04] <-----------------------------+                |
|                                                  |                |
|                                                  v                |
|                                        +-------------------+      |
|                                        |   hex_to_7seg     |      |
|                                        | Hex -> 7-Seg      |      |
|                                        | (Segmente g..a)   |      |
|                                        +---------+---------+      |
|                                                  | w_seg7_out     |
|  [slv_reg2 @ 0x08] <-----------------------------+ [6:0]          |
|                                                                   |
|  [slv_reg3 @ 0x0C] ---> Reserviert / Erweiterungsslot           |
+-------------------------------------------------------------------+

```

### Warum der 7-Segment-Decoder in Hardware sitzt

Statt die Segmentmaske erst im Browser oder im Linux-Backend zu berechnen, übersetzt das Modul `hex_to_7seg` den aktuellen 4-Bit-Zählerstand (`count_out[3:0]`) **in Echtzeit in der FPGA-Logik** in eine 7-Bit-Maske (`seg_out[6:0]`) und legt das Ergebnis auf dem AXI4-Lite-Register `0x08` (`0x43C00008`) ab. Das Dashboard liest `COUNT` und `SEG7` direkt aus dem Registerblock aus und zeigt beide Werte sowie die einzelnen Segmente `a` bis `g` an.

## 2. AXI4-Lite Register-Map (`0x43C00000`)

Bei `C_S_AXI_ADDR_WIDTH = 4` werden die Wortadressen über `S_AXI_AWADDR[3:2]` bzw. `S_AXI_ARADDR[3:2]` dekodiert (Byte-Offset in 4-Byte-Schritten):

| Byte-Offset | Hardware-Adresse | Wort-Index `[3:2]` | Register / Signal | Richtung | Beschreibung | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `0x00` | `0x43C00000` | `2'b00` | `CTRL` (`slv_reg0`) | R/W | Bit `0`: `enable` (`1` = Zähler läuft, `0` = angehalten); beim Reset auf `1` gesetzt | 
| `0x04` | `0x43C00004` | `2'b01` | `COUNT` (`w_count_out`) | R | Aktueller 4-Bit-Zählerstand (`0`..`15` / `0x0`..`0xF`, aufgefüllt auf 32 Bit) | 
| `0x08` | `0x43C00008` | `2'b10` | `SEG7` (`w_seg7_out`) | R | 7-Bit-Decoder-Ausgabe (`seg_out[6:0]`, Bits `6..0` entsprechen `g..a`, `1` = aktiv/an) | 
| `0x0C` | `0x43C0000C` | `2'b11` | `slv_reg3` (frei) | R/W (geplant) | Noch ungenutzter 4. Register-Slot – vorgesehen für Erweiterungen (z. B. Geschwindigkeit, Reset/Preset oder Umlaufzähler) sowie Single-Step-Steuerung | 

### 7-Segment-Bitbelegung (`seg_out[6:0]`)

| Bit | Segment | Position | Beispiel `0` (`0x3F`) | Beispiel `1` (`0x06`) | Beispiel `8` (`0x7F`) | Beispiel `9` (`0x6F`) | Beispiel `E` (`0x79`) | 
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | 
| `0` | `a` | Oben | `1` | `0` | `1` | `1` | `1` | 
| `1` | `b` | Oben rechts | `1` | `1` | `1` | `1` | `0` | 
| `2` | `c` | Unten rechts | `1` | `1` | `1` | `1` | `0` | 
| `3` | `d` | Unten | `1` | `0` | `1` | `1` | `1` | 
| `4` | `e` | Unten links | `1` | `0` | `1` | `0` | `1` | 
| `5` | `f` | Oben links | `1` | `0` | `1` | `1` | `1` | 
| `6` | `g` | Mitte | `0` | `0` | `1` | `1` | `1` | 

## 3. Verzeichnisstruktur des Projekts

```
~/FPGA/myd_c7z020_v2/09_counter_7seg/
├── rtl/
│   ├── counter_4bit.v          # 1-Hz-Prescaler + 4-Bit-Aufwärtszähler (0..15)
│   ├── hex_to_7seg.v           # Kombinatorischer Hex-zu-7-Segment-Decoder (0..F)
│   └── axi_counter_4bit.v      # AXI4-Lite-Slave-Wrapper mit u_counter & u_seg7
├── sw/
│   ├── counter_7seg_web.c      # C-Webserver + /dev/mem-Zugriff auf AXI-Register
│   └── counter_7seg_web.py     # Python-Webserver + /dev/mem-Zugriff auf AXI-Register
├── scripts/
│   ├── build.sh                # Synthese-/Build-Skript (u. a. Bitstream-Erzeugung)
│   ├── deploy.sh               # Transfer zu root@192.168.50.2, FPGA-Load & Service-Start
│   └── gen_schematics.sh       # Erzeugt RTL- und Xilinx-LUT4-SVGs via Yosys + netlistsvg
├── out/
│   ├── counter_7seg.bit.bin    # Binärer FPGA-Bitstream für fpga_manager
│   ├── counter_7seg_web        # Kompiliertes C-Backend-Binary für ARM/Zynq
│   └── counter_7seg_web.py     # Python-Backend-Skript
└── docs/
    └── schematics/
        ├── axi_counter_4bit_top.svg    # Hierarchisches RTL-Schaltbild des Top-Levels
        ├── counter_4bit.svg            # RTL-Schaltbild des Zählers & Prescalers
        ├── hex_to_7seg_rtl.svg         # RTL-Schaltbild des 7-Segment-Decoders
        └── hex_to_7seg_xc7_luts.svg    # Zynq-7-Mapping auf Xilinx LUT4-Zellen

```

## 4. Umgebung & Toolchain (WSL2 + Zynq-7020)

### Host / Arbeitsumgebung (WSL2 unter Windows)

* **Projektpfad in WSL2:** `~/FPGA/myd_c7z020_v2/09_counter_7seg`

* **FPGA-Tooling:** Vivado-Batch-Flow (auf Windows/Laufwerk `D:` bzw. angebunden) vermeidet die Vivado-GUI so weit wie möglich.

* **Open-Source-Schaltplan-Pipeline:**

  * `yosys` (`/usr/bin/yosys`) elaboriert Verilog (`prep -top ...`) bzw. mappt auf Xilinx-7-Primitive (`synth_xilinx -family xc7 -noiopad -noclkbuf`).

  * `nodejs` & `npm` nativ unter Ubuntu/WSL2 (`/usr/bin/node`, `/usr/bin/npm`) – wichtig, damit `npm` nicht auf `/mnt/c/Program Files/nodejs/npm` zeigt und Pfadkonflikte vermeidet.

  * `netlistsvg` (`/usr/local/bin/netlistsvg`) rendert JSON-Netzlisten in skalierbare `.svg`-Dateien.

  * `eog` (Eye of GNOME über WSLg) dient zum schnellen Betrachten und Zoomen der SVGs direkt aus WSL2.

### Zielboard (`MYD-C7Z020-V2`)

* **Adresse:** `root@192.168.50.2`

* **Web-Dashboard:** `http://192.168.50.2:8080`

* **Bitstream-Laden:** Über Linux `fpga_manager` (`LOAD_FPGA=1` in `deploy.sh`).

* **Registerzugriff:** Über `/dev/mem` an Basisadresse `0x43C00000`.

## 5. Wichtige Erkenntnisse, Fixes & Best Practices aus der Einrichtung

1. **Xilinx-Bibliothek beim JSON-Export bereinigen (`-purge_lib`):**
   `synth_xilinx` lädt intern die komplette Xilinx-7-Zellbibliothek (`PS7`, `DSP48E1`, `BRAM` usw.). Ohne Bereinigung entsteht eine mehrseitige JSON-Datei (ca. `9,1 MB`), an der `netlistsvg` scheitert. Nach `synth_xilinx` sorgt **`hierarchy -top <modul> -purge_lib`** dafür, dass nur die tatsächlich verwendeten Zellen (z. B. `LUT4`) exportiert werden und die JSON-Datei nur wenige Kilobyte groß ist.

2. **Weißer Hintergrund in SVGs (`#svg-bg`):**
   `netlistsvg` erzeugt standardmäßig transparente SVGs, die in `eog` als Schachbrettmuster erscheinen. Durch das Einfügen von `<rect id="svg-bg" width="100%" height="100%" style="fill:#ffffff;stroke:none;"/>` direkt nach `</style>` haben alle Schaltbilder einen dauerhaften weißen Hintergrund – auch in viewerspezifischen Darstellungen und im GitHub Dark Mode.

3. **Laufende Backends vor `scp` beenden (`ETXTBSY` vermeiden):**
   Wenn das C-Binary `/root/09_counter_7seg/counter_7seg_web` noch läuft, schlägt ein direktes Überschreiben per `scp` fehl (`dest open ... Failure`). Deshalb stoppt `deploy.sh` laufende Instanzen (`counter_7seg_web`, `counter_7seg_web.py`) **vor** dem Kopieren per `scp`.

4. **Passwortloses SSH/SCP per Ed25519-Key:**
   Mit einem Ed25519-Schlüsselpaar (`~/.ssh/id_ed25519`) und hinterlegtem Public Key auf dem Board laufen `ssh` und `scp` in `deploy.sh` ohne Passwortabfrage (`BatchMode=yes` funktioniert).

5. **Magic VLSI vs. FPGA-Schaltplan-Tools:**
   **Magic** ist ein ASIC-Layout-Editor für physische Chipmasken/GDSII-Layouts (z. B. im OpenLane/LibreLane-SKY130-Flow), kein FPGA-Schaltplan-Viewer für den Zynq-7020. Für RTL- und LUT-Schaltpläne sind `yosys` + `netlistsvg` die passende Wahl.

## 6. Typische Befehle & Verifikation

### Schaltpläne erzeugen & anzeigen

```
cd ~/FPGA/myd_c7z020_v2/09_counter_7seg
./scripts/gen_schematics.sh

# Optional: weißen Hintergrund sicherstellen
for svg in docs/schematics/*.svg; do
    grep -q 'id="svg-bg"' "$svg" || sed -i 's|</style>|</style><rect id="svg-bg" width="100%" height="100%" style="fill:#ffffff;stroke:none;"/>|' "$svg"
done

# Anzeige über WSLg (eog)
eog docs/schematics/*.svg &

```

### Deployment (C-Backend mit FPGA-Reload, dann Wechsel auf Python ohne Reload)

```
cd ~/FPGA/myd_c7z020_v2/09_counter_7seg
./scripts/deploy.sh --c
./scripts/deploy.sh --py --no-fpga

```

### Live-API-Prüfung (`curl`)

```
curl -s http://192.168.50.2:8080/api/status; echo
curl -s -X POST http://192.168.50.2:8080/api/stop; echo
sleep 2
curl -s http://192.168.50.2:8080/api/status; echo
curl -s -X POST http://192.168.50.2:8080/api/start; echo

```

*Erwartetes Verhalten:* `/api/status` liefert `backend`, `enable` (`1` oder `0`), `count` (`0`..`15`), `seg7` (Dezimal) und `seg7_hex` (`0x..`). Bei `POST /api/stop` bleibt `count` stehen, bei `POST /api/start` zählt der 4-Bit-Zähler im 1-Hz-Takt weiter.

## 7. Geplante Erweiterungen für den nächsten Chat

* **Register `0x0C` (`0x43C0000C`) nutzen:** Ausbau des vierten AXI4-Lite-Registers (`slv_reg3` / `2'b11`) in Verilog (`axi_counter_4bit.v` und ggf. `counter_4bit.v`) sowie im C- und Python-Dashboard (z. B. wählbare Zählgeschwindigkeit / Prescaler, Reset/Preset oder Umlaufzähler).

* **Single-Step-Funktion im Dashboard:** Gezieltes Weiterschalten um genau einen Schritt (`0` → `1` → ... → `F` → `0`) aus der Weboberfläche bzw. API heraus, inklusive sauberer Synchronisation mit `enable` / Zählerlogik.
