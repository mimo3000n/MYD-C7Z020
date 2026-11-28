# Teil 4: Workflow, Build, Deployment, Live-Tests & Schematics

Dieser Leitfaden dokumentiert den kompletten, reproduzierbaren Entwicklungs- und Betriebs-Workflow für `13_cpu_8bit_pipe` auf Windows 11 + WSL2 (Ubuntu 24.04) + Vivado 2026.2 + MYD-C7Z020-V2 (`192.168.50.2`), inklusive Live-Verifikation und der Fehlerbehebung beim Schematic-Export.

---

## 1. Systemumgebung & Pfade

| Komponente | Pfad / Adresse | Hinweis |
| :--- | :--- | :--- |
| **WSL2-Projektordner** | `~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe` | Git-Arbeitsverzeichnis, RTL, Simulation, SW, Doku |
| **Windows-Vivado-Build-Ordner** | `D:\vivado_builds\13_cpu_8bit_pipe` (`/mnt/d/vivado_builds/13_cpu_8bit_pipe`) | Vivado-Batch-Aufruf immer über `cmd.exe /c` (Windows-Pfade) |
| **Zynq-7020-Target-Board** | `root@192.168.50.2` | PetaLinux 2026.2, Web-Dashboard auf Port `:8080` |
| **Board-Zielverzeichnis** | `/root/13_cpu_8bit_pipe` | `cpu_8bit_web` + `ui_assets/` |
| **FPGA-Bitstream-Pfad** | `/lib/firmware/cpu_8bit.bit.bin` | Geladen via `/sys/class/fpga_manager/fpga0/firmware` |

---

## 2. Schritt-für-Schritt-Workflow (Simulation -> Synthese -> Board)

### Schritt 1: RTL-Lint & Selbstprüfende Simulation (Icarus Verilog)
Vor jeder Synthese werden Wrapper + Core mit `-Wall` geprüft und alle 4 Pipeline-Testfälle simuliert:

```bash
cd ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe

# 1) Syntax- und Port-Lint fuer das Gesamtdesign
iverilog -g2012 -Wall -s axi_cpu_8bit -o /tmp/axi_pipe_lint.vvp \
  rtl/cpu_8bit_pipe_core.v rtl/axi_cpu_8bit.v

# 2) Selbstpruefende Pipeline-Testbench (4 Tests + VCD)
iverilog -g2012 -Wall -o sim/tb_cpu_8bit_pipe.vvp \
  rtl/cpu_8bit_pipe_core.v sim/tb_cpu_8bit_pipe.v
vvp sim/tb_cpu_8bit_pipe.vvp
```

**Erwartete Ausgabe:**
```text
[PASS] Test 1 Fibonacci: 14 Werte (0..233) exakt!
       Baseline 12_cpu_8bit_asm : 381 Takte (IPC=0.333)
       Pipeline 13_cpu_8bit_pipe: 155 Takte, 127 Befehle, 14 Flushes, 0 Fwds (IPC=0.819, Speedup=2.46x)
[PASS] Test 2 STA->ID Forwarding: OUT=10, fwd_cnt=2, Takte=13 (Trap 0xAA/0x55 ignoriert)
[PASS] Test 3 Branch-Flush: Canary 0x77 unveraendert, flushes=2, Takte=9
[PASS] Test 4 ALU & Not-Taken JC: OUT=0xf1, flushes=2 (1x JC + 1x HLT), Takte=15
=== ALLE 4 PIPELINE-TESTS ERFOLGREICH (0 FEHLER) ===
```

*(Optional lassen sich die erzeugten Signale mit `gtkwave sim/cpu_8bit_pipe.vcd &` im Detail inspizieren.)*

---

### Schritt 2: Open-Source-Vorab-Benchmark (Yosys)
Prüft innerhalb weniger Sekunden Xilinx-7-Ressourcen (`LUT*`, `FDRE/FDSE`, `CARRY4`, `MUXF7/8`) und `CHECK = 0 problems` für Core und AXI-Wrapper:

```bash
cd ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe
./scripts/report_oss.sh
```

---

### Schritt 3: Vivado-2026.2-Vollbuild & Post-Route-Benchmark
Startet die Windows-Vivado-Toolchain per `cmd.exe /c` im Ordner `D:\vivado_builds\13_cpu_8bit_pipe`, erzeugt `out/cpu_8bit.bit.bin` (`3,9 MB`), cross-kompiliert den ARMv7-C-Server (`out/cpu_8bit_web`), bundelt `out/ui_assets/` und liest anschließend die Post-Route-Zahlen aus dem Checkpoint aus:

```bash
cd ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe
mkdir -p /mnt/d/vivado_builds/13_cpu_8bit_pipe

./scripts/build.sh
./scripts/report_vivado.sh
```

**Schneller Software-/UI-Rebuild (ohne Vivado, ca. 1 Sekunde):**
Nach Änderungen in `sw/cpu_8bit_web.c`, `sw/cpu_8bit_web.py` oder `sw/web_ui/*`:

```bash
node --check sw/web_ui/dashboard.js.in
./scripts/build.sh --sw-only
```

---

### Schritt 4: Deployment auf den MYD-C7Z020-V2 (`192.168.50.2`)

```bash
cd ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe

# Voll-Deployment: stoppt alten Server, laedt Bitstream via fpga_manager, startet C-Server
./scripts/deploy.sh --c

# Schnelles SW-/UI-Update ohne erneutes FPGA-Flashen:
./scripts/deploy.sh --c --no-fpga

# Alternative: Python-Fallback-Server auf dem Board starten
./scripts/deploy.sh --py --no-fpga
```

Danach im Browser **`http://192.168.50.2:8080`** öffnen (nach UI-Updates mit **`Ctrl+F5`** neu laden).

---

## 3. Reproduzierbare Live-Hardware-Tests über `curl`

Mit diesen drei Tests lässt sich die echte Zynq-7020-Hardware jederzeit automatisiert aus WSL2 prüfen:

### Test A: Full-Speed Fibonacci bis 233 (`100 MHz`)
```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":0,"speed_sel":3,"reload_demo":1}' http://192.168.50.2:8080/api/ctrl >/dev/null
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":1,"speed_sel":3}' http://192.168.50.2:8080/api/ctrl >/dev/null
sleep 0.1
curl -s http://192.168.50.2:8080/api/status; echo ""
```
**Soll-Ergebnis:** `"halted":1`, `"out_reg":233`, `"cycle_cnt":155`, `"instr_cnt":127`, `"flush_cnt":14`, `"fwd_cnt":0`.

### Test B: `STA -> ID`-Forwarding & Branch-Flush (`Fwd/Flush`)
```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"bytes":[85,79,47,78,62,136,95,224,30,224,240,0,0,0,85,170],"reset":1}' \
  http://192.168.50.2:8080/api/ram >/dev/null
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":1,"speed_sel":3}' http://192.168.50.2:8080/api/ctrl >/dev/null
sleep 0.1
curl -s http://192.168.50.2:8080/api/status; echo ""
```
**Soll-Ergebnis:** `"halted":1`, `"out_reg":10`, `"cycle_cnt":13`, `"instr_cnt":9`, `"flush_cnt":2`, `"fwd_cnt":2`, `ram[14]=10`, `ram[15]=5`.

### Test C: Volle 3/3-Pipeline nach 3 Einzeltakten inspizieren
```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":0,"speed_sel":1,"reload_demo":1}' http://192.168.50.2:8080/api/ctrl >/dev/null
for i in 1 2 3; do
  curl -s -X POST -H "Content-Type: application/json" \
    -d '{"enable":0,"step":1}' http://192.168.50.2:8080/api/ctrl >/dev/null
done
curl -s http://192.168.50.2:8080/api/status; echo ""
```
**Soll-Ergebnis:** `"cycle_cnt":3`, `"instr_cnt":1`, `"if_pc":3` (`JC 0xA`), `"id_pc":2` (`ADD 0xE`), `"ex_pc":1` (`OUT`), alle drei `*_valid = 1`.

---

## 4. RTL- & Xilinx-Schematics generieren & ansehen

### Erzeugung (`scripts/gen_schematics.sh`)
```bash
cd ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe
./scripts/gen_schematics.sh          # erzeugt die 3 Standard-SVGs
# ./scripts/gen_schematics.sh --full # zusaetzlich axi_cpu_8bit_xilinx.svg
```

Erzeugte Dateien (alle mit injiziertem weißen Hintergrund `<rect id="svg-bg" .../>` für Dark-Mode-Viewer):
* `docs/schematics/axi_cpu_8bit_rtl.svg` (`211 KB`)
* `docs/schematics/cpu_8bit_pipe_core_rtl.svg` (`516 KB`)
* `docs/schematics/cpu_8bit_pipe_core_xilinx.svg` (`2,2 MB`)

### Flüssiges Zoomen im Browser (lokal über WSL2)
Wichtig: Der Python-Dateiserver läuft **lokal in WSL2** — im Browser daher **`http://localhost:8090`** aufrufen (nicht die Board-IP `192.168.50.2`!):

```bash
python3 -m http.server 8090 --directory ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe/docs/schematics
# -> Browser: http://localhost:8090  (Beenden mit Ctrl+C)
```

Alternativ direkt in WSLg:
```bash
eog ~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe/docs/schematics/*.svg &
```

### Hintergrund: Lösung des `netlistsvg`-Stack-Overflows (`FlatModule.js:gather`)
Bei der Erstellung der Schematics brach `netlistsvg` an zwei Stellen mit `RangeError: Maximum call stack size exceeded` ab — obwohl RTL und Synthese fehlerfrei waren. Ursache ist ein bekannter Rekursions-Bug im Bus-Merger (`gather()`) von `netlistsvg`, sobald überlappende Bus-Slices auf denselben internen Netz-IDs liegen:

1. **Bei `axi_cpu_8bit_rtl.svg`:** Der 12-fach-Read-Multiplexer (inkl. `0x14`–`0x1C`) wird von Yosys standardmäßig als einzelner 384-Bit-`$pmux` exportiert, der mehrere `0`-Padding-Blöcke und doppelt referenzierte Statusbits (`id_valid`/`ex_valid` in `0x04` und `0x14`) mischt.
   * **Automatischer Fallback in `gen_rtl()`:** `pmuxtree` bricht den großen `$pmux` in reguläre 2:1-`$mux`-Zellen auf. Die kompakte Bus-Darstellung bleibt erhalten und `netlistsvg` läuft sauber durch.
2. **Bei `cpu_8bit_pipe_core_xilinx.svg`:** Nach `synth_xilinx` teilen sich Alias-Ausgänge wie `if_pc`/`pc` sowie `phase = {id_valid, ex_valid}` identische 1-Bit-Netz-IDs in unterschiedlichen Port-Bündelungen.
   * **Automatischer Fallback in `gen_xilinx()`:** `splitnets -ports` teilt ausschließlich die Modul-Ports in 1-Bit-Anschlüsse auf, ohne eine einzige Xilinx-Zelle (`LUT*`, `FDRE`, `FDSE`, `CARRY4`, `MUXF7/8`) zu verändern.

Da beide Maßnahmen erst im Fehlerfall greifen, erhalten unkritische Designs (wie `cpu_8bit_pipe_core_rtl.svg`) weiterhin die kompakteste Standard-Darstellung.

---

## 5. Typische Stolperfallen & Troubleshooting

| Symptom | Ursache | Lösung |
| :--- | :--- | :--- |
| **`POST /api/ctrl` ändert Geschwindigkeit nicht** | Falscher JSON-Key (`"speed"` statt `"speed_sel"`) | Immer `"speed_sel": 0..3` verwenden. |
| **RAM-Tabelle & Waveform bleiben im Browser leer** | `buildRamTable()` wurde vor `updateRamTable()` nicht ausgeführt (oder veralteter Browser-Cache) | Guard `if (!document.getElementById("ram-r-0")) buildRamTable();` ist eingebaut; im Browser mit **`Ctrl+F5`** neu laden. |
| **`192.168.50.2:8090` meldet `ERR_CONNECTION_REFUSED`** | Verwechslung von Zynq-Board (`:8080`) und lokalem WSL2-Schematic-Server (`:8090`) | Dashboard unter `http://192.168.50.2:8080`, lokale Schematics unter `http://localhost:8090` öffnen. |
| **Shell-Pipeline bricht unter `set -euo pipefail` nach `head`/`awk exit` ab** | `SIGPIPE` beendet vorgelagerte Yosys-Prozesse mit Exit-Code 141 | Bei Yosys-Filtern statt `head` besser `awk` komplett durchlaufen lassen oder Ausgabe erst in Datei/Variable puffern. |
| **Vivado-Batch-Skript startet unter WSL2 nicht direkt** | Windows-`.bat`-Dateien benötigen `cmd.exe` | Vivado-Batch-Dateien wie in `scripts/build.sh` und `scripts/report_vivado.sh` stets via `cmd.exe /c` aufrufen. |