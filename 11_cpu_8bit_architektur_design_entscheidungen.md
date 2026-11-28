# Architektur-Deep-Dive: 8-Bit CPU (`11_cpu_8bit`) — Wie und Warum

Dieses Dokument erklärt den inneren Aufbau der Verilog-Module `cpu_8bit_core.v` und `axi_cpu_8bit.v`, analysiert das reale Timing auf dem Zynq-7020 und begründet die getroffenen Architektur-Entscheidungen.

---

## 1. Grundkonzept: Warum eine 3-Phasen-Von-Neumann-CPU?

Die CPU orientiert sich an der klassischen **SAP-1 (Simple-As-Possible)**-Architektur, erweitert um bedingte Sprünge (`JC`, `JZ`), Logikoperationen (`AND`, `OR`, `XOR`, `NOT`, `SHL`) und Immediate-Laden (`LDI`).

**Warum genau diese Aufteilung?**

* **Von-Neumann statt Harvard:** Programmcode und Daten teilen sich ein gemeinsames $16 \times 8\text{ Bit}$-RAM. Dadurch lässt sich am lebendigen Beispiel beobachten, wie der Prozessor in der `FETCH`-Phase ein Byte als **Befehl** interpretiert und in der `EXECUTE`-Phase ein Byte aus demselben Speicher als **Datenwert** liest oder überschreibt (z. B. die Fibonacci-Variablen an `0xD`, `0xE`, `0xF`).
* **3-Phasen-FSM statt Single-Cycle-Ausführung:** Eine moderne CPU oder ein Single-Cycle-Design könnte denselben Befehl in einem einzigen Takt abarbeiten — man würde im Dashboard aber nur fertige Endergebnisse sehen. Die explizite Aufteilung in **`FETCH` $\rightarrow$ `DECODE` $\rightarrow$ `EXECUTE`** macht den Datenfluss sichtbar:
  1. **`FETCH` (`phase = 0`):** Der aktuelle Programmzähler `PC` wird als Speicheradresse angelegt (`MAR <= PC`), das Befehlsbyte `RAM[PC]` wird ins Befehlsregister `IR` geladen und `PC` wird auf `PC + 1` hochgezählt.
  2. **`DECODE` (`phase = 1`):** Die Steuereinheit zerlegt `IR` in `opcode = IR[7:4]` und `operand = IR[3:0]`. Gleichzeitig wird die Operandenadresse in `MAR <= operand` geladen und das zugehörige Datenbyte in `mem_data` bereitgestellt.
  3. **`EXECUTE` (`phase = 2`):** Die ALU bzw. das Steuerwerk führt den Befehl aus, schreibt das Ergebnis in `ACC`, `OUT_REG`, `RAM[operand]` oder `PC`, aktualisiert die Statusflags (`N`, `C`, `Z`) bzw. `halted` und schaltet zurück auf `phase = 0`.
* **Einzelphasen-Debugging:** Jeder Klick auf **Single-Step (+1 Phase)** schaltet genau eine dieser drei Phasen weiter. So sieht man beim Befehl `LDA 0xD` nach Zyklus 1 das geladene `IR = 0x1D`, nach Zyklus 2 den Zeiger `MAR = 0xD` und erst nach Zyklus 3 die Übernahme in `ACC` samt gesetztem Zero-Flag `Z = 1`.

---

## 2. Speicherarchitektur: Warum 16 Bytes in Flip-Flops statt Block-RAM?

Das $16 \times 8\text{ Bit}$-RAM (`reg [7:0] ram [0:15]`) belegt exakt **128 Flip-Flops** direkt in den FPGA-Slices — ohne ein einziges `RAMB18E1`- oder `RAMB36E1`-Block-RAM-Tile.

**Die Gründe dafür:**

1. **Paralleler Vollzugriff für das Dashboard (`ram_flat`):** Ein klassisches Block-RAM besitzt nur 1–2 Leseports. Um alle 16 Bytes im Live-Dashboard gleichzeitig anzuzeigen, ohne die CPU anzuhalten oder einen komplexen Speicher-Arbiter zu bauen, wird das Register-Array als 128-Bit-Vektor `ram_flat` herausgeführt und direkt auf vier 32-Bit-AXI-Leseregister (`RAM_WIN0..3`) gemappt.
2. **Sofortiger Reset & Demo-Reload:** Über `ram_reload_demo` kann der gesamte 16-Byte-Speicher in einem einzigen 100-MHz-Takt deterministisch mit dem Fibonacci-Programm initialisiert werden.
3. **Minimaler Platzbedarf:** 128 Flip-Flops entsprechen gerade einmal **$0{,}12\text{ \%}$** der 106.400 Flip-Flops im Zynq-7020.

---

## 3. Taktdomäne, Clock-Enable und `HOLD`-Verhalten

**Warum gibt es keinen geteilten Hardware-Takt (`BUFG` auf einem Zählerbit), sondern ein `cpu_tick`-Enable?**

In FPGA-Designs sind abgeleitete Ripple-Clocks (ein Flip-Flop-Ausgang als `clk`-Pin für andere Flip-Flops) eine häufige Quelle für Hold-Time-Verletzungen und Clock-Skew. Deshalb laufen **alle** Register im gesamten Design (`axi_cpu_8bit` und `cpu_8bit_core`) strikt synchron auf dem einzigen Systemtakt **`FCLK_CLK0 = 100 MHz`**.

Der interne Taktteiler (`div_cnt`) erzeugt lediglich einen **1 Takt breiten Enable-Impuls** (`clk_div_pulse`), der mit dem manuellen Single-Step-Impuls (`step_trig`) verodert wird:

```verilog
wire cpu_tick = (!halted) && ((enable && clk_div_pulse) || step_trig);
```

**Vorteile dieser Architektur:**

* **Sauberes Timing (Single-Clock-Domain):** Keine Clock-Domain-Crossing-(CDC)-Synchronisierer zwischen AXI-Bus und CPU-Kern nötig.
* **Stoßfreier Wechsel zwischen `RUN` und `Single-Step`:** Da `enable = 0` lediglich `clk_div_pulse` sperrt, friert die CPU sofort im aktuellen Zustand ein; ein einzelner AXI-Schreibimpuls auf `step_trig` lässt die gesamte FSM exakt eine Phase weiterrücken.
* **Live-RAM-Editierung:** Schreibzugriffe vom Dashboard (`ram_we`) erfolgen synchron auf `100 MHz` direkt in `ram[ram_waddr]` — sofort wirksam, ohne Reset.

---

## 4. AXI4-Lite-Wrapper (`axi_cpu_8bit.v`) — Warum Auto-Clear-Bits?

Befehle wie **Single-Step**, **Soft-Reset**, **Demo-RAM laden** und **RAM schreiben** sind **Ereignisse (Impulse)**, keine dauerhaften Zustände. Würde die Software ein Bit erst auf `1` und in einer zweiten Transaktion wieder auf `0` setzen müssen, könnten bei Verzögerungen unter Linux mehrfache Ticks ausgelöst werden.

Deshalb implementiert `axi_cpu_8bit.v` für `CTRL[5:3]` und `RAM_WR[16]` ein **Hardware-Auto-Clear-Muster**:

* Zu Beginn jedes 100-MHz-Taktes werden `step_trig`, `soft_reset`, `ram_reload_demo` und `ram_we` automatisch auf `0` zurückgesetzt.
* Nur in dem einen Takt, in dem eine gültige AXI-Schreibtransaktion (`slv_reg_wren`) die entsprechende Adresse trifft, geht das jeweilige Signal für exakt **10 ns** auf `1`.
* Ein einzelner `POST /api/ctrl` mit `{"step": 1}` garantiert damit physikalisch genau **einen einzigen Phasenschritt**.

### Warum `soft_reset` getrennt von `ram_reload_demo`?

Beim Entwickeln eigener Programme im RAM-Editor möchte man die CPU häufig auf `PC = 0`, `ACC = 0`, `phase = FETCH`, `halted = 0` zurücksetzen, **ohne** das eben eingetippte Programm zu überschreiben. `soft_reset` (`CTRL[4]`) löscht daher ausschließlich CPU-Register und Flags, während `ram_reload_demo` (`CTRL[5]`) zusätzlich die 16 Bytes mit der Fibonacci-Demo neu lädt.

---

## 5. Synthese-, Ressourcen- & Timing-Analyse (Yosys vs. Vivado 2026.2)

Die Hardware wurde sowohl mit dem Open-Source-Synthesizer **Yosys** (`synth_xilinx -flatten -family xc7`) als auch mit **AMD Vivado 2026.2** (vollständiges Place & Route auf `xc7z020clg400-1`) vermessen.

### 5.1 Ressourcenvergleich

| Ressource | Yosys (`axi_cpu_8bit` isoliert) | Vivado (`cpu_ip` isoliert) | AXI-Interconnect & Reset (Differenz) | Vivado Gesamt-PL (`system_wrapper`) |
| :--- | :---: | :---: | :---: | :---: |
| **Logic LUTs** | $\approx 355$ LCs ($389$ `LUT2..6`) | **311** | 310 | 621 |
| **Shift-Reg LUTs (SRL)** | 0 | **0** | 60 | 60 |
| **Total LUTs** | $\approx 355$ | **311** (0,58 %) | 370 | **681** (1,28 %) |
| **Flip-Flops (FFs)** | **311** (`257 FDRE` + `54 FDSE`) | **304** (0,29 %) | 459 | **763** (0,72 %) |
| **CARRY4** | 26 | — | — | — |
| **F7 / F8 Muxes** | 37 / 7 | — | — | 48 / 16 |
| **BRAM / DSP** | 0 / 0 | **0 / 0** | 0 / 0 | **0 / 0** |

**Zentrale Erkenntnisse aus dem Vergleich:**

1. **Hohe Übereinstimmung bei Flip-Flops:** Yosys (311 FFs) und Vivado (304 FFs) liegen fast identisch. Die 7 eingesparten Register in Vivado sind konstante bzw. ungenutzte AXI-Antwort- und Adressbits (`axi_bresp`, `axi_rresp`, `awaddr[1:0]`).
2. **LUT-Packing:** Vivado packt die kombinatorische Logik von `cpu_ip` rund **12 % dichter** (311 statt $\approx 355$ LUTs).
3. **SoC-Infrastruktur-Overhead:** Mehr als die Hälfte des gesamten PL-Bedarfs (370 LUTs und 459 FFs) stammt gar nicht aus der CPU, sondern aus dem von Vivado eingefügten **AXI3-zu-AXI4-Lite-Protokollkonverter** (`ps7_0_axi_periph`) und dem Synchron-Reset-Controller (`rst_ps7_0_100M`).

### 5.2 Kritischer Pfad & $F_{\max}$-Berechnung

Bei einer Zielperiode von $T_{\text{ziel}} = 10{,}000\text{ ns}$ ($100\text{ MHz}$) erreicht das geroutete Design:

* **Worst Negative Slack (WNS):** $+3{,}259\text{ ns}$ (`TNS = 0,000 ns`, 0 von 1.886 Endpoints fehlgeschlagen)
* **Worst Hold Slack (WHS):** $+0{,}053\text{ ns}$ (`THS = 0,000 ns`)
* **Worst Pulse Width Slack (WPWS):** $+4{,}020\text{ ns}$

Daraus ergibt sich die maximale Taktfrequenz auf dem Zynq-7020:

$$T_{\min} = T_{\text{ziel}} - \text{WNS} = 10{,}000\text{ ns} - 3{,}259\text{ ns} = 6{,}741\text{ ns}$$

$$F_{\max} = \frac{1}{T_{\min}} = \frac{1}{6{,}741 \times 10^{-9}\text{ s}} \approx 148{,}3\text{ MHz}$$

**Anatomie der beiden längsten Pfade in `cpu_ip`:**

1. **Pfad 1 (`Slack +3,259 ns`, `6,633 ns` Delay, 10 Logikstufen):**
   `ram_reg[15][0]/C` $\rightarrow$ `2x MUXF7 + 1x MUXF8` (16:1-RAM-Lese-Mux) $\rightarrow$ `2x CARRY4` (8-Bit-ALU `ADD`/`SUB`) $\rightarrow$ `4 LUTs` (Zero-Erkennung `== 8'd0`) $\rightarrow$ `flag_z_reg/D`.
   Das Verhältnis von **45,8 % Logik** zu **54,2 % Routing** zeigt ein sauber ausbalanciertes Placement.
2. **Pfad 2 (`Slack +3,643 ns`, `5,937 ns` Delay, 4 Logikstufen):**
   `clk_div_pulse_reg/C` $\rightarrow$ `ram_reg[0][0]/CE`.
   Hier dominieren **72,3 % Routing-Verzögerung**, da `cpu_tick` als High-Fanout-Enable-Signal alle 128 RAM-Flip-Flops gleichzeitig speist.

**Warum kein weiteres Pipelining nötig war (und wie man auf $\approx 200\text{ MHz}$ käme):**
In der aktuellen Fassung liest die ALU in `EXECUTE` den Operanden kombinatorisch direkt aus `ram[operand]`, obwohl `DECODE` denselben Wert bereits im Register `mem_data` zwischenspeichert. Würde die ALU direkt aus dem registrierten `mem_data` rechnen, entfiele die gesamte `MUXF7`/`MUXF8`-Multiplexerkette aus Pfad 1 — der Taktspielraum stiege Richtung $200\text{ MHz}$. Da das System mit $100\text{ MHz}$ läuft und bereits über $48\text{ \%}$ Reserve besitzt, wurde zugunsten der klaren Lesbarkeit des Datenpfads auf diese Umbau-Stufe verzichtet.

---

## 6. Software-Architektur: Warum ein statischer C-Server mit UI-Bundle?

Auf PetaLinux läuft der Webserver als **statisch gelinktes ARMv7-C-Binary** (`cpu_8bit_web`, ca. 512 KB), während `cpu_8bit_web.py` als identischer Python-Fallback dient.

* **Direkter Registerzugriff über `/dev/mem`:** `mmap()` blendet die physische AXI-Seite `0x43C00000` direkt in den Adressraum des Servers ein. Ein vollständiger Status-Poll liest alle Register (`0x00`–`0x2C`) in wenigen Mikrosekunden ohne Kernel-Treiber-Komplexität.
* **Eingebettetes UI (`web_ui_bundle.h`):** Beim Software-Build (`build.sh --sw-only`) verpackt ein Python-Bundler `index.html`, `style.css` und `dashboard.js` direkt in ein C-Headerfile. Das Binary ist damit komplett eigenständig (Single-Binary-Deployment) und liefert deutlich geringere CPU-Last und Antwortzeiten als `http.server` in Python.
* **Warum Yosys `ltp` `length=106` meldete (Fehlinterpretation vermeiden):** In einem generischen Yosys-Lauf (`synth; abc -lut 6; ltp`) ohne Xilinx-`CARRY4`-Ketten werden 32-Bit-Addierer (`div_cnt`, `cycle_cnt`) als lange Ripple-Carry-Logik gezählt. Erst der Vergleich mit `synth_xilinx` (26 `CARRY4`-Blöcke) und Vivado (`WNS +3,259 ns`) zeigt das echte Hardware-Timing.