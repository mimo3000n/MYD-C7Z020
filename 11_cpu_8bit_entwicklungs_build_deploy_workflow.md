# Entwicklungs-, Build-, Analyse- & Deploy-Workflow (`11_cpu_8bit`)

Dieses Dokument beschreibt den kompletten, reproduzierbaren End-to-End-Workflow für das Projekt **`11_cpu_8bit`** auf dem **MYIR MYD-C7Z020-V2** Board im Hybrid-Betrieb aus **WSL2 (Ubuntu 24.04)** und **AMD Vivado 2026.2 (Windows, Headless)** — inklusive der gelösten Praxis-Stolperfallen.

---

## 1. Systemumgebung & Pfade

| Komponente | Pfad / Adresse |
| :--- | :--- |
| **Projektverzeichnis (WSL2)** | `~/FPGA/myd_c7z020_v2/11_cpu_8bit` |
| **Vivado-Arbeitsverzeichnis (Windows)** | `D:\vivado_builds\11_cpu_8bit` (`/mnt/d/vivado_builds/11_cpu_8bit`) |
| **Vivado 2026.2 Binary (Windows)** | `D:\AMDDesignTools\2026.2\Vivado\bin\vivado.bat` |
| **Open-Source-Tools (WSL2)** | `/usr/bin/iverilog`, `/usr/bin/vvp`, `/usr/bin/yosys`, `/usr/local/bin/netlistsvg`, `eog` |
| **ARMv7 Cross-Compiler (WSL2)** | `arm-linux-gnueabihf-gcc` (statisches Linking für PetaLinux) |
| **Zielsystem (PetaLinux 2026.2)** | `root@192.168.50.2` (SSH, Remote-Ordner `/root/11_cpu_8bit`, Firmware `/lib/firmware/`) |

---

## 2. Phasen-Workflow im Überblick

### Phase 1: Lokale RTL-Verifikation (WSL2 — Sekunden)

Vor jedem Vivado-Lauf werden Syntax, Simulation und Synthesefähigkeit lokal in WSL2 geprüft:

1. **Testbench-Simulation (Icarus Verilog):** Prüft alle 16 Opcodes, die 3-Phasen-FSM (`FETCH`/`DECODE`/`EXECUTE`), Flags (`N`, `C`, `Z`), `HLT`, `soft_reset` sowie den Fibonacci-Durchlauf bis `233`.
2. **Open-Source-Synthesecheck (Yosys):** Prüft mit `synth_xilinx -flatten -family xc7` auf Latches, Treiberkonflikte (`0 problems`) und liefert sofort eine belastbare Vorabschätzung der Flip-Flops und LUTs.

### Phase 2: Hardware-Build (Vivado 2026.2 Headless über `cmd.exe`)

Vivado läuft unter Windows im Batch-Modus (`-mode batch -nojournal -nolog`), gesteuert über ein generiertes `.bat`-Wrapper-Skript aus WSL2:

```bash
# Vollständiger Hardware- + Software-Build
cd ~/FPGA/myd_c7z020_v2/11_cpu_8bit
./scripts/build.sh
```

**Ablauf im Hintergrund:**
1. Kopiert `rtl/*.v` und die TCL-Skripte nach `/mnt/d/vivado_builds/11_cpu_8bit`.
2. Startet Vivado 2026.2 zwingend über **`cmd.exe /c ...bat`** (nicht direkt als `.bat` aus Bash).
3. Erzeugt das Zynq-PS7-Block-Design (`system.bd`), bindet `axi_cpu_8bit` an `0x43C00000` mit $100\text{ MHz}$ `FCLK_CLK0`, führt Synthese, Implementation und Bitstream-Erzeugung durch.
4. Konvertiert den Bitstream über `bootgen` in das für den Linux FPGA-Manager nötige Byte-geswappte Format `out/cpu_8bit.bit.bin`.

### Phase 3: Schneller Software- & UI-Build (WSL2 — ~2 Sekunden)

Wenn sich nur das Web-Dashboard (`sw/web_ui/index.html.in`, `style.css.in`, `dashboard.js.in`) oder der C-Server (`sw/cpu_8bit_web.c`) ändert, ist **kein Vivado-Build** nötig:

```bash
cd ~/FPGA/myd_c7z020_v2/11_cpu_8bit
./scripts/build.sh --sw-only
```

Der Schritt bündelt die Web-Assets in `out/ui_assets/` sowie in `sw/web_ui_bundle.h` und kompiliert `out/cpu_8bit_web` statisch für ARMv7.

### Phase 4: Deployment auf die MYD-C7Z020-V2 (`scripts/deploy.sh`)

`scripts/deploy.sh` deckt alle Einsatzfälle über Flags ab:

```bash
# Standard: Bitstream ins FPGA laden + C-Webserver starten
./scripts/deploy.sh --c

# Schnelles UI-/Server-Update OHNE FPGA-Reload (Board-Zustand bleibt erhalten)
./scripts/deploy.sh --c --no-fpga

# Alternativer Start mit Python-Fallback-Server
./scripts/deploy.sh --py --no-fpga
```

**Die 4 internen Schritte von `deploy.sh`:**
1. **`[1/4]` Stoppen laufender Server:** Scannt `/proc/[0-9]*/cmdline` auf bekannte Dashboard-Prozesse (`cpu_8bit_web`, `counter_7seg_web`, `alu_web`, `freq_mon` usw.), beendet sie erst sauber per `kill` und nach `1 s` per `kill -9`.
2. **`[2/4]` FPGA-Bitstream laden (sofern ohne `--no-fpga`):** Kopiert `cpu_8bit.bit.bin` nach `/lib/firmware/` und triggert den Linux FPGA-Manager (`/sys/class/fpga_manager/fpga0/flags` = `0`, `firmware` = `cpu_8bit.bit.bin`).
3. **`[3/4]` Upload & Hintergrundstart:** Kopiert `ui_assets/*` und `cpu_8bit_web` nach `/root/11_cpu_8bit` und startet den Server sauber entkoppelt via `ssh -n` und `nohup ... >/var/log/cpu_8bit_web.log 2>&1 </dev/null &`.
4. **`[4/4]` Live-Verifikation:** Liest `/var/log/cpu_8bit_web.log` aus und prüft per `curl -s http://192.168.50.2:8080/api/status`, dass der Server JSON-Daten direkt aus der FPGA-Hardware liefert.

---

## 3. Benchmarking & Metrik-Auswertung (Open Source vs. Vivado Headless)

### Open-Source-Report (`scripts/report_oss.sh`)

Erstellt `out/report_oss.txt` mit zwei getrennten Yosys-Läufen für `cpu_8bit_core` und `axi_cpu_8bit`:

1. `synth_xilinx -flatten -family xc7; stat` — exakte Zählung von `FDRE`, `FDSE`, `LUT2..6`, `CARRY4`, `MUXF7`, `MUXF8`.
2. `synth; abc -lut 6; opt_clean; ltp` — kombinatorische 6-LUT-Logiktiefe auf nativen `$dff`-Grenzen.

```bash
./scripts/report_oss.sh
```

### Vivado-Headless-Report ohne Rebuild (`scripts/report_vivado.sh`)

Öffnet lediglich das bereits geroutete Checkpoint-Archiv `system_wrapper_routed.dcp` (~15–55 s) über `vivado/report_metrics.tcl` und schreibt drei Detailberichte nach `out/`:

* `out/report_vivado_hier.txt` — Hierarchische Ressourcenaufteilung (`system_wrapper` vs. `cpu_ip` vs. `ps7_0_axi_periph`).
* `out/report_vivado_timing.txt` — Vollständige Timing-Summary (`WNS`, `TNS`, `WHS`, `THS`, `WPWS`).
* `out/report_vivado_cpu_path.txt` — Kritische Datenpfade isoliert innerhalb von `cpu_ip`.

```bash
./scripts/report_vivado.sh
```

**Schnellabfrage direkt aus den fertigen `impl_1`-Reports (ohne Vivado-Start):**

```bash
IMPL=/mnt/d/vivado_builds/11_cpu_8bit/vivado_out/myd_cpu_8bit_proj.runs/impl_1
grep -A 6 "WNS(ns)" "$IMPL/system_wrapper_timing_summary_routed.rpt" | head -n 8
grep -E "Slice LUTs|Slice Registers|F7 Muxes|F8 Muxes|Block RAM Tile|DSPs" \
  "$IMPL/system_wrapper_utilization_placed.rpt" | head -n 10
```

---

## 4. Schaltplan-Generierung & Anzeige (`scripts/gen_schematics.sh`)

Erzeugt kontrastreiche SVGs mit weißem Hintergrund (`<rect id="svg-bg">`) unter `docs/schematics/`:

* `axi_cpu_8bit_rtl.svg` — Lesbare RTL-Hierarchie (AXI-Decoder, Multiplexer und `u_core`-Instanz).
* `cpu_8bit_core_rtl.svg` — RTL-Schaltplan des CPU-Kerns.
* `cpu_8bit_core_xilinx.svg` — Gate-Level-Ansicht mit echten Xilinx-7-Primitiven (`synth_xilinx -noiopad` + `hierarchy -purge_lib`).
* Optional mit `--full`: zusätzlich `axi_cpu_8bit_xilinx.svg` (gesamtes Gate-Level-Design inkl. AXI-Wrapper).

```bash
# Schaltpläne erzeugen
./scripts/gen_schematics.sh

# Direkt in WSL2 über Eye of GNOME (WSLg) anzeigen
eog docs/schematics/axi_cpu_8bit_rtl.svg docs/schematics/cpu_8bit_core_rtl.svg &

# Alternative für stufenlosen Tiefen-Zoom im Windows-Browser (bei großen Gate-Level-SVGs)
(cd docs/schematics && python3 -m http.server 8090)
# -> http://localhost:8090 im Browser öffnen, danach mit Strg+C stoppen
```

---

## 5. Dokumentierte Stolperfallen & Lösungen (Lessons Learned)

Diese Punkte wurden im Projektverlauf gelöst und sind fest in den Skripten verankert:

| Problem | Ursache | Bewährte Lösung im Projekt |
| :--- | :--- | :--- |
| **Vivado `.bat`-Aufruf aus WSL2 scheitert** | Windows-Batch-Dateien lassen sich nicht direkt als Linux-Executable ausführen. | Immer über `(cd "$WIN_DIR" && cmd.exe /c run_*.bat)` aufrufen. |
| **Port `8080` beim Deploy belegt (`Address already in use`)** | BusyBox `ps` schneidet lange Pfade ab (z. B. `counter_7seg_web.p`), und `\$1` in Single-Quotes brach `awk`. | `deploy.sh` liest ungekürzt `/proc/[0-9]*/cmdline` und stoppt alle alten Dashboard-Prozesse mit `kill` + `sleep 1` + `kill -9`. |
| **`ssh` in `deploy.sh` kehrt nach `nohup ... &` nicht zurück** | Bei `chmod ... && nohup ... &` gilt `&` für die gesamte `&&`-Kette; eine Remote-Subshell bleibt aktiv und hält `stdout`/`stderr` offen. | **`ssh -n`** verwenden und Befehle mit **`;`** statt `&&` trennen: `chmod +x ...; nohup ... >/var/log/... 2>&1 </dev/null &`. |
| **Yosys `report_oss.sh` liefert leere Ausgabe + falsche `Detected loop`-Warnungen** | `-q` unterdrückt auch `stat`/`ltp`-Ausgaben; `ltp` nach `synth_xilinx` kennt Xilinx-Blackbox-FFs (`FDRE`) nicht als Registergrenzen. | Ohne `-q` filtern (`sed`/`grep`) und `synth_xilinx; stat` sauber von `synth; abc -lut 6; ltp` trennen. |
| **SVGs auf dunklem Viewer-Hintergrund unlesbar** | `netlistsvg` erzeugt schwarzen Text auf transparentem Hintergrund. | `gen_schematics.sh` fügt automatisch `<rect id="svg-bg" width="100%" height="100%" fill="#ffffff"/>` hinter dem `<svg>`-Tag ein. |
| **Browser zeigt altes CSS/JS nach `--sw-only`-Deploy** | Browser-Cache für `style.css` / `dashboard.js`. | Nach jedem UI-Update im Browser **`Ctrl+F5`** (Hard Reload) drücken. |