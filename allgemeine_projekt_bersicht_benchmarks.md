# 13_cpu_8bit_pipe — 3-stufige 8-Bit-Pipeline-CPU auf MYD-C7Z020-V2 (Zynq-7020)

Das Projekt **`13_cpu_8bit_pipe`** ist die direkte Weiterentwicklung von `12_cpu_8bit_asm` auf dem **MYIR MYD-C7Z020-V2** (Xilinx Zynq-7000 `xc7z020clg400-1`). Anstelle einer klassischen 3-Phasen-Zustandsmaschine (`FETCH -> DECODE -> EXECUTE`, 3 Takte pro Befehl) arbeitet der Kern nun mit einer **3-stufigen Hardware-Pipeline (`IF -> ID/RD -> EX/WB`)**, Dual-Read-Port-RAM, **`STA -> ID`-RAM-Forwarding** und einem **2-Takt-Pipeline-Flush** bei Sprüngen und `HLT`.

Das gesamte System bleibt zu **100 % binärkompatibel** zu allen 16 Opcodes und dem Browser-Assembler aus `12_cpu_8bit_asm`. Bestehende Programme laufen ohne jede Änderung am Maschinencode — bei **$2{,}46\times$ weniger Takten** für Fibonacci und einer um **+17,6 % höheren Maximalfrequenz** ($F_{\max} = 174{,}4\text{ MHz}$).

---

## Dokumentationsstruktur (4 Teile)

Die Dokumentation ist thematisch in vier eigenständige, aufeinander aufbauende Markdown-Dateien gegliedert:

| Teil | Datei | Inhalt & Schwerpunkte |
| :--- | :--- | :--- |
| **1. Allgemein & Benchmarks** | [`README.md`](./README.md) | Projektziele, Evolutionsvergleich, ISA-Übersicht, vollständige Icarus-, Yosys-, Vivado-2026.2- und Zynq-7020-Hardware-Benchmarks, Projektstruktur |
| **2. Verilog & Hardware-RTL** | [`docs/01_verilog_pipeline_rtl.md`](./docs/01_verilog_pipeline_rtl.md) | Pipeline-Architektur (`IF -> ID/RD -> EX/WB`), Dual-Read-Port-RAM, Single-Muxed Write-Port (MUX-Optimierung), `STA -> ID`-Forwarding, Branch-Flush, kritischer Pfad, AXI4-Lite-Registerkarte (`0x00`–`0x2C`), Testbench |
| **3. Dashboard & Debugger** | [`docs/02_dashboard_debugger.md`](./docs/02_dashboard_debugger.md) | C- und Python-Webserver über `/dev/mem` (`0x43C00000`), REST-API (`/api/status`, `/api/ctrl`, `/api/ram`), 3-Stufen-Pipeline-Leiste, Datenpfad-SVG, Waveform (inkl. HALT-Freeze), Browser-Assembler & Source-Level-Debugger |
| **4. Workflow & Schematics** | [`docs/03_workflow_deployment_schematics.md`](./docs/03_workflow_deployment_schematics.md) | End-to-End-Ablauf (WSL2 + Windows Vivado 2026.2 via `cmd.exe`), Build-, Fast-SW- und Deploy-Befehle für `192.168.50.2`, `curl`-Hardware-Tests, `gen_schematics.sh` inkl. `netlistsvg`-Fix (`pmuxtree` / `splitnets`) |

---

## Architektur-Evolution im Überblick

| Ausbaustufe | Projekt | Kern-Architektur | Zentrale Neuerungen |
| :--- | :--- | :--- | :--- |
| **Stufe 1** | `11_cpu_8bit` | 3-Phasen-FSM (`FETCH / DECODE / EXECUTE`) | 8-Bit-Softcore, 16-Byte-RAM, AXI4-Lite (`0x00`–`0x2C`), Basis-Web-Dashboard |
| **Stufe 2** | `12_cpu_8bit_asm` | 3-Phasen-FSM (Baseline: 311 LUTs, 148,3 MHz) | Browser-Assembler, Source-Level-Debugger, Breakpoints, OUT-Konsole, C-Server |
| **Stufe 3** | **`13_cpu_8bit_pipe`** | **3-Stufen-Pipeline (`IF -> ID/RD -> EX/WB`)** | Dual-Read-Port-RAM, `STA -> ID`-Forwarding, 2-Takt-Flush, Pipeline-Telemetrie (`0x14`–`0x1C`), Live-IPC & Stage-Highlighting |

---

## Warum eine 3-stufige Pipeline?

In der Baseline `12_cpu_8bit_asm` lagen zwei wesentliche Bremsen:

1. **Durchsatz-Limit ($\text{IPC} \le 0{,}333$):** Jeder Befehl belegte nacheinander 3 volle Takte (`FETCH`, `DECODE`, `EXECUTE`). Während die ALU rechnete, lag der Befehls-Fetch brach; während ein Befehl geholt wurde, schlief die ALU.
2. **Kritischer Timing-Pfad (10 Logic Levels, $148{,}3\text{ MHz}$):** In der `EXECUTE`-Phase musste innerhalb eines einzigen Taktes das Daten-Byte asynchron aus dem 16-Byte-RAM über einen Multiplexer-Baum (`MUXF7`/`MUXF8`) gelesen, durch die 8-Bit-ALU (`CARRY4`) geschickt und das Zero-Flag (`flag_z`) berechnet werden.

Die 3-stufige Pipeline in `13_cpu_8bit_pipe` löst beide Engpässe gleichzeitig:
* **Überlappung:** Bis zu drei Befehle befinden sich gleichzeitig in Bearbeitung (`IF`, `ID/RD` und `EX/WB`). Im regulären Ablauf wird **jeden Takt ein Befehl fertiggestellt** ($\text{IPC}_{\text{ideal}} = 1{,}00$).
* **Pfadtrennung:** Der RAM-Lesezugriff wandert in Stufe 2 (`ID/RD`) und wird dort im Pipeline-Register `id_ex_mem_data` zwischengespeichert. Stufe 3 (`EX/WB`) speist die ALU ausschließlich direkt aus Flip-Flops (`acc` und `id_ex_mem_data`).

---

## Befehlssatz-Architektur (ISA — 16 Opcodes, 100 % kompatibel)

Jeder Befehl ist 1 Byte breit: Das obere Nibble (`[7:4]`) kodiert den Opcode, das untere Nibble (`[3:0]`) die 4-Bit-RAM-Adresse (`0x0`–`0xF`) bzw. einen 4-Bit-Immediate-Wert.

| Opcode | Mnemonic | Argument | Funktion | Flags (`N, C, Z`) | RAM-Lesen in `ID/RD` | Pipeline-Effekt |
| :---: | :---: | :---: | :--- | :---: | :---: | :--- |
| `0x0` | `NOP` | – | Keine Operation | unverändert | Nein | 0 Stall |
| `0x1` | `LDA` | `addr` | $\text{ACC} \leftarrow \text{RAM}[\text{addr}]$ | `N`, `Z` | **Ja** (FWD-fähig) | 0 Stall |
| `0x2` | `ADD` | `addr` | $\text{ACC} \leftarrow \text{ACC} + \text{RAM}[\text{addr}]$ | `N`, `C`, `Z` | **Ja** (FWD-fähig) | 0 Stall |
| `0x3` | `SUB` | `addr` | $\text{ACC} \leftarrow \text{ACC} - \text{RAM}[\text{addr}]$ (`C` = Borrow) | `N`, `C`, `Z` | **Ja** (FWD-fähig) | 0 Stall |
| `0x4` | `STA` | `addr` | $\text{RAM}[\text{addr}] \leftarrow \text{ACC}$ | unverändert | Nein (Write in `EX`) | Quelle für `STA -> ID` FWD |
| `0x5` | `LDI` | `imm` | $\text{ACC} \leftarrow \{4'\text{b0000}, \text{imm}\}$ | `N`, `Z` | Nein | 0 Stall |
| `0x6` | `JMP` | `addr` | $\text{PC} \leftarrow \text{addr}$ | unverändert | Nein | **Flush** (2 Bubbles) |
| `0x7` | `JC` | `addr` | Wenn `C = 1`: $\text{PC} \leftarrow \text{addr}$ | unverändert | Nein | Taken: **Flush** / Not-Taken: 0 |
| `0x8` | `JZ` | `addr` | Wenn `Z = 1`: $\text{PC} \leftarrow \text{addr}$ | unverändert | Nein | Taken: **Flush** / Not-Taken: 0 |
| `0x9` | `AND` | `addr` | $\text{ACC} \leftarrow \text{ACC} \land \text{RAM}[\text{addr}]$ | `N`, `C=0`, `Z` | **Ja** (FWD-fähig) | 0 Stall |
| `0xA` | `OR` | `addr` | $\text{ACC} \leftarrow \text{ACC} \lor \text{RAM}[\text{addr}]$ | `N`, `C=0`, `Z` | **Ja** (FWD-fähig) | 0 Stall |
| `0xB` | `XOR` | `addr` | $\text{ACC} \leftarrow \text{ACC} \oplus \text{RAM}[\text{addr}]$ | `N`, `C=0`, `Z` | **Ja** (FWD-fähig) | 0 Stall |
| `0xC` | `NOT` | – | $\text{ACC} \leftarrow \neg \text{ACC}$ | `N`, `C=0`, `Z` | Nein | 0 Stall |
| `0xD` | `SHL` | – | $\text{ACC} \leftarrow \text{ACC} \ll 1$ (`C` = altes Bit 7) | `N`, `C`, `Z` | Nein | 0 Stall |
| `0xE` | `OUT` | – | $\text{OUT\_REG} \leftarrow \text{ACC}$ | unverändert | Nein | 0 Stall |
| `0xF` | `HLT` | – | CPU anhalten (`halted <= 1`), `PC <= ex_pc + 1` | unverändert | Nein | **Flush** & Stopp |

---

## Benchmarks: Vivado 2026.2, Yosys & Zynq-7020-Hardware

### 1. Vivado 2026.2 Post-Route Benchmark (`xc7z020clg400-1`, Target 100 MHz)

Die folgenden Werte stammen direkt aus den Post-Route-Checkpoints (`report_utilization -hierarchical` und `report_timing` isoliert auf `*cpu_ip*`) von Vivado v2026.2 (Build 6713330):

| Metrik | `12_cpu_8bit_asm` (Baseline) | `13_cpu_8bit_pipe` (inkl. `0x14`–`0x1C`) | Veränderung |
| :--- | :---: | :---: | :---: |
| **`cpu_ip` Total LUTs** | `311` (0,58 % von 53.200) | **`364`** (0,68 % von 53.200) | `+53 LUTs` (+17,0 %) |
| **`cpu_ip` Flip-Flops (FFs)** | `304` (0,29 % von 106.400) | **`376`** (0,35 % von 106.400) | `+72 FFs` (+23,7 %) |
| **Gesamt-PL (`system_wrapper`)** | `681 LUTs` / `763 FFs` | **`734 LUTs`** (`674` Logic + `60` SRL) / **`835 FFs`** | `+53 LUTs` / `+72 FFs` |
| **BRAM / DSP Blocks** | `0 / 0` | **`0 / 0`** | unverändert |
| **Worst Negative Slack (`WNS` @ 100 MHz)** | `+3,259 ns` | **`+4,265 ns`** | **`+1,006 ns`** |
| **Minimale Taktperiode $T_{\min}$** | `6,741 ns` | **`5,735 ns`** | **`-1,006 ns`** |
| **Maximale Taktfrequenz $F_{\max}$** | `148,3 MHz` | **`174,4 MHz`** | **`+26,1 MHz` (+17,6 %)** |
| **Kritischer Pfad (Logic Levels)** | `10` (`ram -> MUXF7/8 -> CARRY4 -> flag_z`) | **`4`** (`LUT4=1, LUT5=1, LUT6=2`) | **`-6` Levels** |
| **Pfad-Aufteilung (Logic / Route)** | Logik-dominiert | `5,352 ns` (`1,318 ns` = 24,6 % Logic / `4,034 ns` = 75,4 % Route) | Routing-dominiert |

> **Wichtige Erkenntnis:** Der kritische Pfad liegt nicht mehr in der ALU, sondern auf dem Steuersignal von `ex_ir_reg[6]/C` zum Clock-Enable `ram_reg[5][0]/CE` (nur noch 4 LUT-Stufen, 75,4 % reine Routing-Verzögerung). Für lediglich +53 LUTs und +72 FFs (inklusive der kompletten Hardware-Telemetriezähler `instr_cnt`, `flush_cnt` und `fwd_cnt`) steigt $F_{\max}$ auf $174{,}4\text{ MHz}$.

---

### 2. Yosys Open-Source-Synthese-Vergleich (`synth_xilinx -family xc7`)

Vor dem Vivado-Lauf wurde die RTL-Qualität mit Yosys geprüft. Dabei deckte Yosys in Version 1 sofort eine massive Multiplexer-Explosion durch zwei getrennte dynamische RAM-Schreibports im selben `always`-Block auf. Der Umbau auf einen einzelnen gemuxten Write-Port (Details in Teil 2) drückte den LUT-Bedarf sogar **unter** die ursprüngliche 3-Phasen-Baseline:

| Yosys-Primitive (`xc7`) | `cpu_8bit_core` (Baseline) | `cpu_8bit_pipe_core` (v1 unoptimiert) | `cpu_8bit_pipe_core` (final) | `axi_cpu_8bit` (final komplett) |
| :--- | :---: | :---: | :---: | :---: |
| **LUT1..LUT6 gesamt** | `271` (`150` LUT6) | `533` (`363` LUT6) | **`260`** (`129` LUT6) | `423` (`207` LUT6) |
| **MUXF7 / MUXF8** | `37 / 7` | `257 / 114` | **`13 / 2`** | `23 / 6` |
| **FDRE / FDSE (FFs)** | `194 / 52` (`246`) | `316 / 0` (`316`) | **`259 / 52` (`311`)** | `322 / 54` (`376`) |
| **CARRY4** | `26` | `42` | `42` | `42` |
| **Estimated LCs** | – | – | **`223`** | **`381`** |
| **Yosys `CHECK`** | `0 problems` | `0 problems` | **`0 problems`** | **`0 problems`** |

---

### 3. Ausführungs- & Durchsatz-Benchmark (Icarus Verilog = Zynq-7020 @ 100 MHz)

Alle Simulationsergebnisse wurden 1:1 auf dem echten Zynq-7020-Board (`192.168.50.2`) bei **100 MHz Full-Speed (`speed_sel = 3`)** bestätigt:

| Benchmark-Programm | Baseline `12_cpu_8bit_asm` | Pipeline `13_cpu_8bit_pipe` | `instr_cnt` | `flush_cnt` | `fwd_cnt` | IPC | Speedup |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Fibonacci bis 233** (14 `OUT`-Werte `0..233`) | `381` Takte (`IPC = 0,333`) | **`155` Takte** | `127` | `14` | `0` | **`0,819`** | **`$2{,}46\times$`** (Takte) |
| **STA->ID Forwarding** (`Fwd/Flush`, Trap `0xAA`/`0x55`) | – (kein Hazard) | **`13` Takte** (`OUT = 10`) | `9` | `2` | **`2`** | `0,692` | 0 Stalls |
| **Branch-Flush Canary** (`JMP` vor `STA`/`OUT`) | – | **`9` Takte** (`OUT = 0x77`) | `5` | `2` | `0` | `0,556` | Canary intakt |
| **ALU + Not-Taken `JC` + Borrow-`JC`** | – | **`15` Takte** (`OUT = 0xF1`) | `11` | `2` | `0` | `0,733` | 0 Penalty (Not-Taken) |

Die Fibonacci-Taktzahl lässt sich analytisch exakt herleiten:
$$\text{Takte}_{\text{Pipe}} = \underbrace{2}_{\text{Pipeline-Fill}} + \underbrace{127}_{\text{Befehle}} + \underbrace{13 \times 2}_{\text{13 genommene Sprünge}} = 155\text{ Takte}$$

Kombiniert man die Taktzahl-Reduktion mit dem $F_{\max}$-Gewinn, ergibt sich der effektive Durchsatz-Speedup bei maximalem Takt:
$$\text{Speedup}_{\text{eff}} = \frac{381}{155} \times \frac{174{,}4\text{ MHz}}{148{,}3\text{ MHz}} \approx 2{,}458 \times 1{,}176 \approx \mathbf{2{,}89\times} \quad (+189\,\%)$$

---

## Projektverzeichnis (`~/FPGA/myd_c7z020_v2/13_cpu_8bit_pipe`)

```text
13_cpu_8bit_pipe/
├── README.md                          # Teil 1: Übersicht & Benchmarks (diese Datei)
├── docs/
│   ├── 01_verilog_pipeline_rtl.md     # Teil 2: Verilog-Core, Forwarding, AXI4-Lite & Testbench
│   ├── 02_dashboard_debugger.md       # Teil 3: C/Python-Server, REST-API, UI & Debugger
│   ├── 03_workflow_deployment_schematics.md # Teil 4: Build, Deploy, Curl-Tests & Schematics
│   └── schematics/
│       ├── axi_cpu_8bit_rtl.svg       # AXI4-Lite-Wrapper RTL-Schematic (211 KB)
│       ├── cpu_8bit_pipe_core_rtl.svg # Pipeline-Core RTL-Schematic (516 KB)
│       └── cpu_8bit_pipe_core_xilinx.svg # Xilinx-7 Gate-Level-Netlist (2,2 MB)
├── rtl/
│   ├── cpu_8bit_pipe_core.v           # 3-stufiger Pipeline-Core (214 Zeilen)
│   └── axi_cpu_8bit.v                 # AXI4-Lite-Slave-Wrapper (Register 0x00..0x2C)
├── sim/
│   ├── tb_cpu_8bit_pipe.v             # Self-Checking-Testbench (4 Tests, VCD-Export)
│   └── cpu_8bit_pipe.vcd              # GTKWave-Waveform der Simulation
├── sw/
│   ├── cpu_8bit_web.c                 # Statisch gelinkter ARMv7 C-HTTP-Server (/dev/mem)
│   ├── cpu_8bit_web.py                # Python-Fallback-Server (identische API)
│   └── web_ui/
│       ├── index.html.in              # Dashboard-Struktur + Pipeline-Bar + Assembler/Debugger
│       ├── style.css.in               # 100dvh Dark-Theme + Stage-Farben (IF/ID/EX/FWD/FLUSH)
│       └── dashboard.js.in            # Polling, SVG-Animation, Waveform, Assembler & Debugger
├── vivado/
│   ├── build.tcl                      # Vivado-2026.2-Batch-Build (PS7 + cpu_ip -> .bit.bin)
│   ├── report_metrics.tcl             # Isolierter cpu_ip-Utilization- & Timing-Report
│   └── myir_ps7_preset.tcl            # MYIR MYD-C7Z020-V2 PS7-Preset
├── scripts/
│   ├── build.sh                       # Voll-Build (Vivado + SW) oder --sw-only
│   ├── deploy.sh                      # Deployment auf 192.168.50.2 (--c/--py, --no-fpga)
│   ├── report_oss.sh                  # Yosys xc7-Ressourcen & 6-LUT-LTP-Benchmark
│   ├── report_vivado.sh               # Auslesen der Vivado-Post-Route-Metriken
│   └── gen_schematics.sh              # Yosys + netlistsvg (mit pmuxtree/splitnets-Fallback)
└── out/
    ├── cpu_8bit.bit.bin               # Zynq-7020-Bitstream (3,9 MB, für fpga_manager)
    ├── cpu_8bit_web                   # ARMv7-C-Binary (552 KB)
    └── ui_assets/                     # Gebundelte Web-Assets (index.html, style.css, dashboard.js)
```