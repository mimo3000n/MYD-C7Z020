# Teil 2: Verilog-Architektur, Pipeline-RTL & AXI4-Lite-Interface

Dieses Dokument erklärt den kompletten Hardware-Aufbau von `13_cpu_8bit_pipe` — von den drei Pipeline-Stufen in `rtl/cpu_8bit_pipe_core.v` über das Hazard-Handling und die Ressourcen-Optimierung bis zur AXI4-Lite-Registerkarte in `rtl/axi_cpu_8bit.v` und der Icarus-Verilog-Verifikation.

---

## 1. Blockstruktur der 3-stufigen Pipeline (`IF -> ID/RD -> EX/WB`)

```text
          Stage 1: IF                  Stage 2: ID/RD                  Stage 3: EX/WB
  +-------------------------+   +---------------------------+   +---------------------------+
  |   PC (4-Bit Register)   |   |  id_valid / id_pc / id_ir |   |  ex_valid / ex_pc / ex_ir |
  |            |            |   |       |            |      |   |       id_ex_mem_data      |
  |            v            |   |    id_op        id_imm    |   |        |         |        |
  |  RAM Read-Port 1        |==>|       |            |      |==>|        v         v        |
  |  if_ir = ram[pc]        |   |       |    RAM Read-Port 2|   |   +-------------------+   |
  |                         |   |       |    ram[id_imm]    |   |   |    8-Bit ALU      |   |
  |  Next PC:               |   |       |         |         |   |   |  (ACC, mem_data)  |   |
  |   - ex_hlt:    ex_pc+1  |   |       |         v         |   |   +---------+---------+   |
  |   - ex_branch: ex_imm   |   |       |   +-----------+   |   |             |             |
  |   - sonst:     pc + 1   |<--+-------+---| FWD-MUX   |<--+---+-- ACC (bei STA->ID Hit)   |
  +-------------------------+   |           | fwd_hit ? |   |   |             v             |
                                |           | acc : ram |   |   |  Writeback:               |
                                |           +-----------+   |   |   - ACC, Flags (N,C,Z)    |
                                +---------------------------+   |   - OUT_REG / HALTED      |
                                              ^                 |   - RAM Write (bei STA)   |
                                              |                 |   - Branch/Flush-Signal   |
                                       Flush bei Branch/HLT     +---------------------------+
```

### Aufgabenverteilung der drei Stufen

| Stufe | Register am Eingang | Kombinatorische Arbeit | Ausgang am Takt-Ende (`posedge clk`) |
| :--- | :--- | :--- | :--- |
| **1. IF** (*Instruction Fetch*) | `pc` | Liest Befehl über Read-Port 1 (`if_ir = ram[pc]`). | Übergibt `pc`, `if_ir` und `id_valid <= 1` an Stufe 2; inkrementiert `pc <= pc + 1` (sofern kein Flush). |
| **2. ID/RD** (*Decode & RAM Read*) | `id_valid`, `id_pc`, `id_ir` | Zerlegt `id_ir` in `id_op = id_ir[7:4]` und `id_imm = id_ir[3:0]`; liest Daten-Operand über Read-Port 2 (`ram[id_imm]`) und prüft **`STA -> ID`-Forwarding**. | Speichert `id_pc`, `id_ir`, `id_valid`, `ex_fwd` sowie den ausgewählten Operanden `id_ex_mem_data` in die `ID/EX`-Pipeline-Register. |
| **3. EX/WB** (*Execute & Writeback*) | `ex_valid`, `ex_pc`, `ex_ir`, `id_ex_mem_data` | Führt die 8-Bit-ALU rein auf Registern (`acc`, `id_ex_mem_data`) aus; wertet Branch-Bedingungen (`JMP`, `JC`, `JZ`) und `HLT` aus. | Schreibt `acc`, `flag_z/c/n`, `out_reg`, `halted` oder `ram[ex_imm]`; bei Sprung/HLT erfolgt **Flush** von `IF` und `ID`. |

---

## 2. Hazard-Analyse: Warum 0 Stalls bei ALU-Ketten und `STA -> ID`?

In klassischen Pipelines entstehen drei Arten von Konflikten (*Hazards*). Durch die kompakte 3-stufige Aufteilung werden zwei davon mit **null Takten Verlust (0 Stalls)** aufgelöst:

### A) Daten-Hazard auf `ACC` und Flags (`Z, C, N`) — automatisch frei von Stalls
Da `EX` und `WB` in **derselben Stufe (Stufe 3)** liegen, aktualisiert ein Befehl wie `ADD 0xE` den Akkumulator `acc` und die Flags `flag_z/c/n` exakt an derselben steigenden Taktflanke, an der der nächste Befehl (z. B. `JC 0xA` oder `STA 0xF`) von Stufe 2 nach Stufe 3 einrückt. Der Folgebefehl sieht daher sofort den brandneuen `acc`- und Flag-Wert — ganz ohne zusätzlichen Bypass oder Stall.

### B) Speicher-Daten-Hazard (`STA addr` gefolgt von `LDA/ADD/SUB/AND/OR/XOR addr`) — `STA -> ID`-Forwarding
Hier existiert ein echter **Read-after-Write-(RAW)-Hazard**:
* In Takt $t$ steht `STA 0xF` in **Stufe 3 (`EX/WB`)** und schreibt `acc` erst am Ende von Takt $t$ (bei der nächsten `posedge clk`) in `ram[0xF]`.
* Gleichzeitig steht der unmittelbar folgende Befehl `ADD 0xF` bereits in **Stufe 2 (`ID/RD`)** und liest über Read-Port 2 `ram[0xF]` — dort steht aber noch der **alte** Speicherwert!

**Lösung in Hardware (`rtl/cpu_8bit_pipe_core.v`):**
```verilog
wire id_reads_ram = (id_op == OP_LDA) || (id_op == OP_ADD) ||
                    (id_op == OP_SUB) || (id_op == OP_AND) ||
                    (id_op == OP_OR)  || (id_op == OP_XOR);
wire ex_is_sta    = ex_valid && (ex_op == OP_STA);
wire fwd_ram_hit  = id_valid && id_reads_ram && ex_is_sta && (id_imm == ex_imm);
wire [7:0] id_mem_data = fwd_ram_hit ? acc : ram[id_imm];
```

Wenn `fwd_ram_hit == 1` gilt, umgeht der 8-Bit-Multiplexer das RAM und leitet `acc` direkt in das Pipeline-Register `id_ex_mem_data`. Im gleichen Takt wird `ex_fwd <= 1` gesetzt und `fwd_cnt` hochgezählt. Ein Beispiel aus unserem `Fwd/Flush`-Testprogramm (mit absichtlichem Trap-Byte `0xAA` in `ram[0xF]`):

| Takt | Stage 1 (`IF`) | Stage 2 (`ID/RD`) | Stage 3 (`EX/WB`) | Was passiert? |
| :---: | :--- | :--- | :--- | :--- |
| **1** | `0: LDI 5` | *Bubble* | *Bubble* | Fetch von `0x0` |
| **2** | `1: STA F` | `0: LDI 5` | *Bubble* | Pipeline füllt sich |
| **3** | `2: ADD F` | `1: STA F` | `0: LDI 5` | `ACC <= 5` |
| **4** | `3: STA E` | `2: ADD F` | `1: STA F` | **`fwd_ram_hit = 1`!** `ADD F` bekommt `ACC = 5` statt `0xAA` aus RAM; `ram[F] <= 5` |
| **5** | `4: SUB E` | `3: STA E` | `2: ADD F` (`ex_fwd=1`) | `ACC <= 5 + 5 = 10` |

### C) Steuerfluss-Hazard (`JMP`, genommener `JC`/`JZ`, `HLT`) — 2-Takt-Flush
Da die Sprungentscheidung in **Stufe 3 (`EX/WB`)** fällt (erst dort liegen die aktuellen Flags `flag_c` / `flag_z` des Vorgängerbefehls vor), wurden in der Zwischenzeit bereits zwei nachfolgende Befehle spekulativ in `ID/RD` und `IF` geladen.

```verilog
wire ex_branch_taken = ex_valid && (
    (ex_op == OP_JMP) ||
    (ex_op == OP_JC && flag_c) ||
    (ex_op == OP_JZ && flag_z)
);
wire ex_hlt   = ex_valid && (ex_op == OP_HLT);
wire ex_flush = ex_branch_taken || ex_hlt;
```

* **Bei `ex_branch_taken == 1`:** `pc <= ex_imm`, `id_valid <= 0`, `ex_valid <= 0`, `ex_fwd <= 0`. Die beiden spekulativ geladenen Befehle werden zu harmlosen **Bubbles** verworfen, noch bevor sie `EX/WB` erreichen — ein dahinterliegendes `STA` oder `OUT` kann niemals unerwünschte Seiteneffekte auslösen.
* **Bei nicht genommenem `JC`/`JZ`:** `ex_branch_taken == 0` — die Pipeline läuft ohne Unterbrechung weiter (**0 Takte Branch-Penalty**).
* **Bei `ex_hlt == 1`:** `halted <= 1`, `pc <= ex_pc + 1`, `id_valid <= 0`, `ex_valid <= 0`. Der Programmzähler zeigt sauber auf die Adresse unmittelbar nach `HLT`.

---

## 3. Synthese-Optimierung: Vom MUX-Problem zu $174{,}4\text{ MHz}$

Während der Entwicklung von `cpu_8bit_pipe_core.v` lieferte Yosys in Version 1 zunächst ein überraschendes Warnsignal: **`257 MUXF7`, `114 MUXF8`, `533 LUTs` und `0 FDSE`**.

### Ursache der MUX-Explosion in v1
In Version 1 standen im selben `always @(posedge clk)`-Block:
1. ein verschachteltes `if (!rst_n || ram_reload_demo) ... else if (soft_reset) ...`,
2. der CPU-Schreibzugriff `OP_STA: ram[ex_imm] <= acc;`, und
3. danach der Host-Schreibzugriff `if (ram_we) ram[ram_waddr] <= ram_wdata;`.

Dadurch erkannte die Synthese **zwei unabhängige dynamische Schreibports** auf demselben 16-Byte-Registerfeld plus zwei unterschiedliche Reset-Pfade — und baute vor jedes einzelne der 128 RAM-Flip-Flops einen tiefen Multiplexer-Baum. Zusätzlich lag der 25-Bit-Komparator (`div_cnt >= div_limit`) direkt kombinatorisch im `cpu_tick`-Pfad.

### Die zwei gezielten RTL-Änderungen in v2 (Final)
1. **Einzelner gemuxter RAM-Write-Port im eigenen `always`-Block:**
   ```verilog
   wire       cpu_ram_we  = (!soft_reset) && cpu_tick && ex_is_sta;
   wire       ram_wr_en   = ram_we || cpu_ram_we;
   wire [3:0] ram_wr_addr = ram_we ? ram_waddr : ex_imm;
   wire [7:0] ram_wr_data = ram_we ? ram_wdata : acc;
   ```
   Dadurch inferieren Yosys und Vivado exakt 128 saubere Flip-Flops (`FDRE`/`FDSE`, entsprechend der Demo-Fibonacci-Initialisierung) mit gemeinsamem Clock-Enable (`CE`).
2. **Registrierter Taktteiler-Puls `div_tick_r` (27-Bit, parametrisiert über `CLK_FREQ_HZ`):**
   ```verilog
   wire div_tick = enable && ((speed_sel == 2'b11) || div_tick_r);
   wire cpu_tick = (!halted) && (step_trig || div_tick);
   ```
   Der 27-Bit-Komparator (`div_cnt >= div_limit`) schließt nun sauber an einem Flip-Flop (`div_tick_r`) ab und belastet `cpu_tick` nicht mehr.

**Messbarer Effekt:**
* Yosys `MUXF7 / MUXF8`: **`257 / 114` -> `13 / 2`**
* Yosys `LUTs` (Core): **`533` -> `260`** (sogar weniger als die 271 LUTs des unpipelined Cores!)
* Vivado Post-Route: **4 Logic Levels**, `WNS = +4,265 ns`, **$F_{\max} = 174{,}4\text{ MHz}$** (gegenüber `10` Logic Levels / `148,3 MHz` in der Baseline).

---

## 4. Taktteiler (`speed_sel`)

Über `CTRL[2:1]` (`0x00`) lassen sich vier Geschwindigkeiten wählen (`CLK_FREQ_HZ = 100_000_000`):

| `speed_sel` | Teilerformel | Taktfrequenz | Einsatzbereich |
| :---: | :--- | :---: | :--- |
| `2'b00` | `(CLK_FREQ_HZ / 1) - 1` | **1 Hz** | Langsames Mitverfolgen der Pipeline-Stufen |
| `2'b01` | `(CLK_FREQ_HZ / 2) - 1` | **2 Hz** (Default) | Interaktive Live-Beobachtung im Dashboard |
| `2'b10` | `(CLK_FREQ_HZ / 10) - 1` | **10 Hz** | Zügiger Ablauf inkl. Live-Waveform |
| `2'b11` | Bypass (`div_tick = enable`) | **Full-Speed (100 MHz)** | Echte Hardware-Benchmarks (Fibonacci in $1{,}55\,\mu\text{s}$) |

---

## 5. AXI4-Lite-Wrapper (`rtl/axi_cpu_8bit.v`) & vollständige Registerkarte

Der AXI4-Lite-Slave liegt auf Basisadresse **`0x43C00000`** (64-KB-Fenster im Zynq-PS7-Adressraum) und dekodiert Wortadressen über `axi_awaddr[5:2]` / `axi_araddr[5:2]`. Alle bisherigen Register aus `11_cpu_8bit` und `12_cpu_8bit_asm` bleiben unverändert; die zuvor freien Offsets **`0x14`**, **`0x18`** und **`0x1C`** wurden für die Pipeline-Telemetrie ergänzt:

| Offset | Index | Name | Zugriff | Bit-Belegung & Bedeutung |
| :---: | :---: | :--- | :---: | :--- |
| **`0x00`** | `4'h0` | `CTRL` | R/W | `[0]` `enable` (RUN/HOLD), `[2:1]` `speed_sel`, `[3]` `step_trig` (*auto-clear*, +1 Takt), `[4]` `soft_reset` (*auto-clear*, RAM bleibt erhalten), `[5]` `ram_reload_demo` (*auto-clear*, lädt Fibonacci-Demo + Reset) |
| **`0x04`** | `4'h1` | `CPU_CORE` | RO | `[7:0]` `acc`, `[15:8]` `ir` (`ex_valid ? ex_ir : 0`), `[19:16]` `pc` (`if_pc`), `[21:20]` `phase` (`{id_valid, ex_valid}`), `[22]` `flag_z`, `[23]` `flag_c`, `[24]` `flag_n`, `[25]` `halted` |
| **`0x08`** | `4'h2` | `BUS_OUT` | RO | `[7:0]` `alu_res`, `[15:8]` `out_reg`, `[23:16]` `mem_data` (`id_ex_mem_data`), `[27:24]` `mar` (`ex_valid ? ex_imm : 0`) |
| **`0x0C`** | `4'h3` | `CYCLE_CNT` | RO | `[31:0]` `cycle_cnt` (ausgeführte CPU-Takte seit Reset) |
| **`0x10`** | `4'h4` | `RAM_WR` | R/W | `[7:0]` `ram_wdata`, `[11:8]` `ram_waddr`, `[16]` `ram_we` (*auto-clear*, Schreibpuls mit Vorrang vor CPU) |
| **`0x14`** | `4'h5` | **`PIPE_STAGES`** | RO | **`[3:0]` `if_pc`**, **`[7:4]` `id_pc`**, **`[11:8]` `ex_pc`**, **`[12]` `if_valid`**, **`[13]` `id_valid`**, **`[14]` `ex_valid`**, **`[15]` `ex_fwd`**, **`[16]` `last_flush`**, `[23:17]` `0`, **`[31:24]` `id_ir`** |
| **`0x18`** | `4'h6` | **`PIPE_HAZARDS`** | RO | **`[15:0]` `flush_cnt`** (Anzahl Branch-/HLT-Flushes), **`[31:16]` `fwd_cnt`** (Anzahl `STA -> ID`-Forwardings) |
| **`0x1C`** | `4'h7` | **`INSTR_CNT`** | RO | **`[31:0]` `instr_cnt`** (abgeschlossene Befehle in `EX/WB` zur Berechnung von $\text{IPC} = \text{instr\_cnt} / \text{cycle\_cnt}$) |
| **`0x20`** | `4'h8` | `RAM_WIN0` | RO | `ram[3], ram[2], ram[1], ram[0]` (`ram_flat[31:0]`) |
| **`0x24`** | `4'h9` | `RAM_WIN1` | RO | `ram[7], ram[6], ram[5], ram[4]` (`ram_flat[63:32]`) |
| **`0x28`** | `4'hA` | `RAM_WIN2` | RO | `ram[11], ram[10], ram[9], ram[8]` (`ram_flat[95:64]`) |
| **`0x2C`** | `4'hB` | `RAM_WIN3` | RO | `ram[15], ram[14], ram[13], ram[12]` (`ram_flat[127:96]`) |

---

## 6. Selbstprüfende Icarus-Verilog-Testbench (`sim/tb_cpu_8bit_pipe.v`)

Die Testbench prüft alle kritischen Pipeline-Fälle automatisch mit `PASS`/`FAIL`-Auswertung und schreibt `sim/cpu_8bit_pipe.vcd` für GTKWave:

1. **Test 1 — Fibonacci bis 233:** Prüft alle 14 `OUT`-Ausgaben (`0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233`), exakte Taktzahl `155` (gegenüber `381` Baseline), `127` Befehle, `14` Flushes, `0` Forwardings und $\text{IPC} = 0{,}819$.
2. **Test 2 — `STA -> ID` RAM-Forwarding:** Füllt `ram[0xE] = 0x55` und `ram[0xF] = 0xAA` als Trap-Werte und führt direkt hintereinander `LDI 5 -> STA F -> ADD F -> STA E -> SUB E -> JZ 8 -> (LDI F, OUT geflusht) -> LDA E -> OUT -> HLT` aus. Nur mit funktionierendem Forwarding entsteht `OUT = 10` mit `fwd_cnt = 2` in `13` Takten.
3. **Test 3 — Branch-Flush & Wrong-Path-Canary:** Platzierte `STA F` und `OUT` direkt hinter `JMP 5`, während `ram[0xF]` den Canary-Wert `0x77` hält. Bestätigt in `9` Takten, dass weder `ram[0xF]` überschrieben noch ein falsches `OUT` erzeugt wird.
4. **Test 4 — Vollständiger ALU- & Not-Taken-`JC`-Test:** Prüft `SHL`, nicht genommenen `JC` (0 Takte Penalty), `OR`, `AND`, `XOR`, `NOT`, `SUB` mit Borrow (`0xF0 - 0xFF = 0xF1`, `C = 1`) und anschließend genommenen `JC` (`OUT = 0xF1`, `flush_cnt = 2`, `15` Takte).