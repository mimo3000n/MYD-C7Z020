# 11_cpu_8bit — 8-Bit SAP-1 / Von-Neumann CPU auf Zynq-7020

Eine selbst entwickelte, vollständig im FPGA-Fabric (PL) synthetisierte **8-Bit-Lern- und Demonstrations-CPU** auf dem **MYIR MYD-C7Z020-V2** (`xc7z020clg400-1`), gesteuert über ein **AXI4-Lite-Interface** vom ARM-Prozessorsystem (PS7 unter PetaLinux 2026.2) und visualisiert über ein responsives **Live-Web-Dashboard** auf `http://192.168.50.2:8080`.

---

## Dokumentationsstruktur

Diese Projektdokumentation ist bewusst modular aufgebaut:

1. **`README.md`** *(dieses Dokument)* — Kurzübersicht, Eckdaten, Befehlssatz (ISA), AXI-Register-Map und Hardware-Kennzahlen.
2. **[`docs/01_architecture_deep_dive.md`](docs/01_architecture_deep_dive.md)** — Detailerklärung der CPU-Architektur, der 3-Phasen-FSM, des AXI4-Lite-Wrappers, der Timing-Analyse sowie das **Wieso und Warum** hinter jeder Design-Entscheidung.
3. **[`docs/02_dashboard_guide.md`](docs/02_dashboard_guide.md)** — Vollständige Erklärung aller vier Dashboard-Bereiche, der Bedienelemente, der vier Demo-Presets und ein praktisches Schritt-für-Schritt-Beispiel (Fibonacci & eigene Programme).
4. **[`docs/03_workflow_build_deploy.md`](docs/03_workflow_build_deploy.md)** — Reproduzierbarer Hybrid-Workflow (WSL2 + Vivado 2026.2 Headless), Deployment auf PetaLinux, Schaltplan-Generierung und dokumentierte Stolperfallen.

---

## Systemüberblick & Eckdaten

| Eigenschaft | Wert / Spezifikation |
| :--- | :--- |
| **Ziel-Hardware** | MYIR MYD-C7Z020-V2 (Xilinx Zynq-7020, `xc7z020clg400-1`) |
| **CPU-Architektur** | 8-Bit Von-Neumann / SAP-1-Erweiterung (`cpu_8bit_core`) |
| **Speicher (RAM)** | $16 \times 8\text{ Bit}$ (16 Bytes in Slice-Flip-Flops, 0 BRAM) |
| **Busbreiten** | 8-Bit Datenbus, 4-Bit Adressbus (`0x0` bis `0xF`) |
| **Befehlszyklus** | 3-Phasen-FSM: `FETCH` $\rightarrow$ `DECODE` $\rightarrow$ `EXECUTE` |
| **Systemtakt (PL)** | $100\text{ MHz}$ (`FCLK_CLK0`), interner Taktteiler + Single-Step |
| **AXI4-Lite Basisadresse** | `0x43C00000` (angebunden über `ps7_0_axi_periph`) |
| **Web-Backend** | Statisches ARMv7-C-Binary (`cpu_8bit_web`, Port `8080`) + Python-Fallback (`cpu_8bit_web.py`) |
| **Board-Adresse** | `root@192.168.50.2` / Dashboard: `http://192.168.50.2:8080` |

---

## Befehlssatz (ISA — 16 Opcodes)

Jeder Befehl belegt genau **1 Byte**: Das obere Nibble `IR[7:4]` codiert den **Opcode**, das untere Nibble `IR[3:0]` liefert die **4-Bit-RAM-Adresse** bzw. den **4-Bit-Immediate-Wert**.

| Opcode | Mnemonic | Format | Operation | Beeinflusste Flags |
| :---: | :--- | :--- | :--- | :---: |
| `0x0` | `NOP` | `00` | Keine Operation | — |
| `0x1` | `LDA a` | `1a` | `ACC <= RAM[a]` | `Z`, `N` |
| `0x2` | `ADD a` | `2a` | `{C, ACC} <= ACC + RAM[a]` | `C`, `Z`, `N` |
| `0x3` | `SUB a` | `3a` | `{C, ACC} <= ACC - RAM[a]` | `C`, `Z`, `N` |
| `0x4` | `STA a` | `4a` | `RAM[a] <= ACC` | — |
| `0x5` | `LDI i` | `5i` | `ACC <= {4'b0000, i}` | `Z`, `N` |
| `0x6` | `JMP a` | `6a` | `PC <= a` | — |
| `0x7` | `JC a` | `7a` | Wenn `C == 1`: `PC <= a` | — |
| `0x8` | `JZ a` | `8a` | Wenn `Z == 1`: `PC <= a` | — |
| `0x9` | `AND a` | `9a` | `ACC <= ACC & RAM[a]` | `Z`, `N` |
| `0xA` | `OR a` | `Aa` | `ACC <= ACC | RAM[a]` | `Z`, `N` |
| `0xB` | `XOR a` | `Ba` | `ACC <= ACC ^ RAM[a]` | `Z`, `N` |
| `0xC` | `NOT` | `C0` | `ACC <= ~ACC` | `Z`, `N` |
| `0xD` | `SHL` | `D0` | `{C, ACC} <= {ACC, 1'b0}` | `C`, `Z`, `N` |
| `0xE` | `OUT` | `E0` | `OUT_REG <= ACC` | — |
| `0xF` | `HLT` | `F0` | `halted <= 1` (CPU stoppt bis Reset) | `H` |

---

## AXI4-Lite Register-Map (`0x43C00000`)

| Offset | Register | Zugriff | Funktion / Bit-Belegung |
| :---: | :--- | :---: | :--- |
| `0x00` | `CTRL` | R/W | `[0]` `enable`, `[2:1]` `speed_sel`, `[3]` `step_trig` (auto-clear), `[4]` `soft_reset` (auto-clear), `[5]` `ram_reload_demo` (auto-clear) |
| `0x04` | `CPU_CORE` | R | Live-Snapshot von `acc`, `ir`, `pc`, `phase`, `flag_z`, `flag_c`, `flag_n`, `halted` |
| `0x08` | `BUS_OUT` | R | Live-Snapshot von `alu_res`, `out_reg`, `mem_data`, `mar` |
| `0x0C` | `CYCLE_CNT` | R | 32-Bit-Phasenzähler (inkrementiert pro CPU-Tick bzw. Single-Step) |
| `0x10` | `RAM_WR` | W | `[7:0]` `ram_wdata`, `[11:8]` `ram_waddr`, `[16]` `ram_we` (auto-clear) |
| `0x20`–`0x2C` | `RAM_WIN0..3` | R | Vier 32-Bit-Lesefenster auf alle 16 RAM-Bytes (`ram_flat[31:0]` bis `ram_flat[127:96]`) |

---

## Synthese-, Ressourcen- & Timing-Ergebnisse (`xc7z020clg400-1`)

| Kennzahl | Yosys (`axi_cpu_8bit` isoliert) | Vivado (`cpu_ip` isoliert) | Vivado Gesamt-PL (`system_wrapper`) | Zynq-7020 Anteil (Gesamt) |
| :--- | :---: | :---: | :---: | :---: |
| **Total LUTs** | $\approx 355$ LCs ($389$ `LUT2..6`) | **311** | **681** (621 Logic + 60 SRL) | **1,28 %** (von 53.200) |
| **Flip-Flops (FFs)** | **311** (`257 FDRE` + `54 FDSE`) | **304** | **763** | **0,72 %** (von 106.400) |
| **F7 / F8 Muxes** | 37 / 7 | — | 48 / 16 | 0,18 % / 0,12 % |
| **Block RAM / DSP** | 0 / 0 | **0 / 0** | **0 / 0** | 0,00 % |
| **WNS / TNS (100 MHz)** | — | **`+3,259 ns` / `0,000 ns`** | **`+3,259 ns` / `0,000 ns`** | 0 / 1.886 Endpoints failing |
| **WHS / WPWS** | — | — | **`+0,053 ns` / `+4,020 ns`** | Alle Constraints erfüllt |
| **Maximaltakt $F_{\max}$** | — | **$\approx 148{,}3\text{ MHz}$** | **$\approx 148{,}3\text{ MHz}$** | $+48\text{ \% Taktreserve}$ |

---

## Projektstruktur & Schaltpläne

```text
~/FPGA/myd_c7z020_v2/11_cpu_8bit/
├── README.md                          # Diese Übersicht
├── docs/
│   ├── 01_architecture_deep_dive.md   # Architektur, FSM, AXI, Timing & Design-Gründe
│   ├── 02_dashboard_guide.md          # Dashboard-Handbuch & Praxisbeispiele
│   ├── 03_workflow_build_deploy.md    # WSL2 + Vivado Headless Workflow & Fixes
│   └── schematics/
│       ├── axi_cpu_8bit_rtl.svg       # RTL-Schaltplan AXI-Wrapper + u_core
│       ├── cpu_8bit_core_rtl.svg      # RTL-Schaltplan CPU-Kern
│       └── cpu_8bit_core_xilinx.svg   # Gate-Level-Schaltplan (Xilinx-7 Primitive)
├── rtl/
│   ├── cpu_8bit_core.v                # 8-Bit CPU-Kern (FSM, ALU, 16x8-RAM, Taktteiler)
│   └── axi_cpu_8bit.v                 # AXI4-Lite Slave-Wrapper (0x43C00000)
├── sim/                               # Icarus-Verilog Testbench
├── sw/
│   ├── cpu_8bit_web.c                 # Statischer ARMv7-C-HTTP-Server (/dev/mem)
│   ├── cpu_8bit_web.py                # Python-Fallback-Server
│   └── web_ui/                        # HTML-, CSS- (.in) und JS-Quellen + Bundler
├── vivado/                            # TCL-Skripte (Build & Headless-Reporting)
├── scripts/                           # build.sh, deploy.sh, report_oss.sh, report_vivado.sh, gen_schematics.sh
└── out/                               # cpu_8bit.bit.bin, cpu_8bit_web, Reports, ui_assets/
```

---

## Schnellstart (WSL2)

```bash
cd ~/FPGA/myd_c7z020_v2/11_cpu_8bit

# Nur Software / Web-UI neu bauen und ohne FPGA-Reload deployen (~3 s)
./scripts/build.sh --sw-only
./scripts/deploy.sh --c --no-fpga

# Vollständiges Deployment inkl. Bitstream-Load auf das Zynq-7020 Board
./scripts/deploy.sh --c

# Open-Source-Synthese & Vivado-Headless-Metriken vergleichen
./scripts/report_oss.sh
./scripts/report_vivado.sh

# Schaltpläne neu erzeugen und in WSLg (eog) öffnen
./scripts/gen_schematics.sh
eog docs/schematics/axi_cpu_8bit_rtl.svg docs/schematics/cpu_8bit_core_rtl.svg &
```