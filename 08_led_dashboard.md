# Projekt 08: AXI4-Lite LED-Controller & Web-Dashboard (`08_led_dashboard`)

Eigenständiges FPGA- und Embedded-Linux-Projekt zur gezielten Ansteuerung von LEDs über einen eigenen AXI4-Lite-Slave, ein C-CLI-Tool (`/dev/mem`) sowie ein lokales Python-Web-Dashboard auf dem **MYD-C7Z020-V2**.

---

## 1. Systemübersicht & Zielhardware

| Parameter | Wert |
| :--- | :--- |
| **Projektpfad (WSL2)** | `~/FPGA/myd_c7z020_v2/08_led_dashboard/` |
| **Staging-Pfad (Windows)** | `D:\vivado_builds\08_led_dashboard\` |
| **Board / SOM** | MYD-C7Z020-V2 (`MYC-C7Z020-V2` SOM + `MYB-C7Z010/20-I` Baseboard) |
| **FPGA-Typ** | Xilinx Zynq-7000 `xc7z020clg400-1` |
| **Ziel-IP (PetaLinux)** | `192.168.50.2` (Benutzer: `root`) |
| **Vivado-Toolchain** | Vivado 2026.2 (`D:\AMDDesignTools\2026.2\Vivado\bin\`) |
| **AXI4-Lite Basisadresse** | `0x43C00000` (Adressbereich: `64K`, Takt: `FCLK_CLK0` @ 50 MHz) |

---

## 2. Hardware-Analyse: Onboard-LEDs (MYD-C7Z020-V2)

Laut *MYD-C7Z010-20-V2 Hardware User Manual (Kap. 3.4)* und dem Baseboard-Schaltplan besitzt das Board ab Werk **nur eine einzige frei über die FPGA-PL steuerbare Onboard-LED** (`D3` auf dem SOM). Alle übrigen LEDs sind fest verdrahtete Status- oder Power-Anzeigen:

| LED | Position | Farbe | Signal / Netz | Frei steuerbar? |
| :--- | :--- | :--- | :--- | :--- |
| **D3** | SOM | Grün | **`IO_B34_LP3`** (Pin **`U13`**) | **Ja** (PL Bank 34, `LVCMOS33`) |
| **D1** | SOM | Rot | `DONE` | Nein (FPGA-Konfigurationsstatus) |
| **D2** | SOM | Blau | `VDD_3.3V` | Nein (SOM 3,3 V Power) |
| **D53** | Baseboard | Blau | `VDD_3.3V` | Nein (Carrier 3,3 V Power) |
| **D51** | Baseboard | Rot | `FMC_PG` | Nein (FMC Power-Good) |
| **D10–D13** | Baseboard | Rot | `FLAG_VBUS1..4` | Nein (USB-Host-Überstromanzeige) |

### Aktuelle Pin-Belegung (`xdc/leds.xdc`)

* **`led_n[0]` (LED 0)** $\rightarrow$ Package-Pin **`U13`** (`IO_L3P_T0_DQS_PUDC_B_34`, Bank 34, `LVCMOS33`) $\rightarrow$ **Grüne SOM-LED D3**.
* **`led_n[1]` (LED 1)** $\rightarrow$ Platzhalter `J14` (verhindert Vivado `DRC UCIO-1`-Fehler; vorbereitet für PMOD Bank 13).
* **`led_n[2]` (LED 2)** $\rightarrow$ Platzhalter `G14` (verhindert Vivado `DRC UCIO-1`-Fehler; vorbereitet für PMOD Bank 13).

---

## 3. Projektstruktur

```text
~/FPGA/myd_c7z020_v2/08_led_dashboard/
├── README.md                  # Diese Dokumentation
├── rtl/
│   └── axi_led_ctrl.v         # Eigener AXI4-Lite Slave (3-Bit LED-Register)
├── xdc/
│   └── leds.xdc               # Pin-Constraints (U13 fuer D3 + Platzhalter)
├── sw/
│   ├── led_ctrl.c             # Userspace C-CLI-Tool via /dev/mem
│   └── led_dashboard.py       # Python3-Webserver + Browser-Dashboard (Port 8080)
├── scripts/
│   ├── build_vivado.tcl       # Headless Vivado Block-Design & Bitstream-Build
│   └── build.sh               # WSL2-Wrapper (Staging auf D:, Vivado + Bootgen)
└── out/
    ├── axi_led_ctrl.bit       # Vivado Bitstream
    ├── axi_led_ctrl.bit.bin   # Bootgen-Binary fuer Linux fpgautil (ca. 3,9 MB)
    ├── led_ctrl               # Statisch kompiliertes ARMv7-Binary
    └── vivado.log             # Build-Protokoll
```

---

## 4. Register-Map (`axi_led_ctrl.v`)

Das Modul `axi_led_ctrl` ist über `M_AXI_GP0` an das Zynq Processing System angebunden. Die Low-Active-Invertierung (`assign led_n = ~led_reg;`) erfolgt direkt im FPGA, sodass Software-seitig sauber mit `1 = EIN` und `0 = AUS` gearbeitet wird. Nach einem Reset sind alle LEDs ausgeschaltet (`3'b000`).

| Offset | Name | Zugriff | Reset | Bits | Beschreibung |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0x00` | `LED_CTRL` | R/W | `0x00000000` | `[2:0]` | Bit 0 = LED 0 (`D3` / `U13`), Bit 1 = LED 1, Bit 2 = LED 2 (`1` = EIN, `0` = AUS) |
| `0x00` | *(reserviert)* | R | `0` | `[31:3]` | Liefert beim Lesen immer `0` |

---

## 5. Build-Anleitung (WSL2)

### 5.1 FPGA-Bitstream bauen (`Vivado 2026.2` + `bootgen`)

Da Vivado als Windows-Anwendung installiert ist und `cmd.exe` keine WSL-UNC-Pfade (`\\wsl.localhost\...`) als Arbeitsverzeichnis unterstützt, kopiert `scripts/build.sh` die Quelldateien automatisch nach `/mnt/d/vivado_builds/08_led_dashboard`, startet `vivado.bat` und `bootgen.bat` über `cmd.exe /c` und holt die fertigen Artefakte nach `out/` zurück:

```bash
cd ~/FPGA/myd_c7z020_v2/08_led_dashboard
./scripts/build.sh
```

Erwartete Ausgabe am Ende:
* `write_bitstream completed successfully` (0 Errors, 0 Critical Warnings)
* `[INFO] : Bootimage generated successfully`
* `out/axi_led_ctrl.bit.bin` (ca. 3,9 MB)

### 5.2 C-Tool cross-kompilieren (`ARMv7`)

Einmalige Voraussetzung in WSL2: `sudo apt install -y gcc-arm-linux-gnueabihf`.  
Durch `-static` läuft das Binary unabhängig von der `glibc`-Version auf PetaLinux:

```bash
cd ~/FPGA/myd_c7z020_v2/08_led_dashboard
arm-linux-gnueabihf-gcc -O2 -Wall -static -o out/led_ctrl sw/led_ctrl.c
```

---

## 6. Deployment & Bedienung auf dem Board (`192.168.50.2`)

### 6.1 Dateien übertragen (WSL2)

```bash
cd ~/FPGA/myd_c7z020_v2/08_led_dashboard
scp out/axi_led_ctrl.bit.bin out/led_ctrl sw/led_dashboard.py root@192.168.50.2:/home/root/
```

### 6.2 Bitstream laden & starten (SSH auf `192.168.50.2`)

```bash
# Falls ein anderes Dashboard Port 8080 belegt (z. B. fclk_dashboard.py):
pkill -f fclk_dashboard.py || true

# Bitstream live in die PL laden
fpgautil -b /home/root/axi_led_ctrl.bit.bin

# Ausfuehrrechte setzen
chmod +x /home/root/led_ctrl /home/root/led_dashboard.py
```

### 6.3 Steuerung per CLI (`led_ctrl`)

```bash
./led_ctrl status         # Aktuellen Registerzustand auslesen
./led_ctrl set 0 on       # LED 0 (D3 auf dem SOM) einschalten
./led_ctrl set 0 off      # LED 0 ausschalten
./led_ctrl set 0 toggle   # LED 0 umschalten
./led_ctrl mask 0x7       # Bitmaske (0..7) direkt schreiben
```

### 6.4 Steuerung per Web-Dashboard (`led_dashboard.py`)

Das Skript benötigt keine externen Python-Pakete und synchronisiert sich alle 1,5 Sekunden mit dem Hardware-Register (Änderungen per CLI erscheinen sofort im Browser):

```bash
python3 /home/root/led_dashboard.py
```

* **Browser-URL:** `http://192.168.50.2:8080`
* **REST-API:**
  * `GET /api/leds` $\rightarrow$ liefert `{"mask": 1, "leds": [true, false, false]}`
  * `POST /api/leds` $\rightarrow$ akzeptiert `{"toggle": 0}`, `{"mask": 7}` oder `{"led": 0, "state": true}`

---

## 7. Diagnose & Verifikation (Checkliste)

Falls die Hardware nicht wie erwartet reagiert, helfen folgende Prüfungen direkt auf dem Zynq:

```bash
# 1. FPGA-Manager-Status (muss "operating" sein)
cat /sys/class/fpga_manager/fpga0/state

# 2. AXI-Readback-Test (muss 0x00000007 zurueckliefern)
devmem 0x43C00000 32 0x7
devmem 0x43C00000 32

# 3. SLCR FPGA-Reset (muss 0x00000000 sein) & Level-Shifter (muss 0x0000000F sein)
devmem 0xF8000240 32
devmem 0xF8000900 32
```

In WSL2 lässt sich die tatsächliche Pin-Platzierung im Vivado-Report prüfen:

```bash
grep -i "led" /mnt/d/vivado_builds/08_led_dashboard/vivado_proj/led_dashboard_hw.runs/impl_1/design_1_wrapper_io_placed.rpt
```

---

## 8. Geplante Erweiterungen

* **3 echte externe LEDs über PMOD (`J13`, `J14` oder `J15`, Bank 13):** Umbelegen von `led_n[0..2]` in `xdc/leds.xdc` auf freie PMOD-Pins (`VDDIO_13_PL` = 3,3 V), um alle 3 Kanäle des Dashboards physisch zu nutzen und per Logikanalysator zu messen.
* **Autostart-Service:** Einrichten von `led_dashboard.py` als Hintergrunddienst (SysVinit/systemd) unter PetaLinux.
