# Teil 3: Web-Dashboard, C-/Python-Backend & Source-Level-Debugger

Dieser Teil beschreibt die komplette Software-Schicht auf dem MYD-C7Z020-V2 (`192.168.50.2:8080`): den direkten Registerzugriff über `/dev/mem`, die REST-API-Erweiterung für die Pipeline-Telemetrie sowie die Funktionsweise von Dashboard, Waveform, Browser-Assembler und Source-Level-Debugger.

---

## 1. Software-Architektur (PS7 Linux <-> PL AXI4-Lite)

```text
Browser (PC / Tablet)                   MYD-C7Z020-V2 (PetaLinux 2026.2, ARMv7)
+-------------------------------+       +----------------------------------------+
| http://192.168.50.2:8080      | HTTP  | cpu_8bit_web (C, 552 KB, statisch)     |
| - 3-Stufen-Pipeline-Leiste    |<=====>|   oder cpu_8bit_web.py (Fallback)      |
| - SVG-Datenpfad & Waveform    | JSON  | - liefert ui_assets/ statisch aus      |
| - 16-Byte RAM & Disassembler  |       | - mmap("/dev/mem", 0x43C00000, 4 KB)   |
| - Browser-Assembler (Alt+A)   |       +-------------------+--------------------+
| - Source-Debugger (Alt+D)     |                           | AXI4-Lite (32-Bit)
+-------------------------------+                           v
                                        +----------------------------------------+
                                        | PL: axi_cpu_8bit + cpu_8bit_pipe_core  |
                                        | Offsets 0x00..0x2C @ 100 MHz           |
                                        +----------------------------------------+
```

Beide Server-Implementierungen bieten exakt dieselbe HTTP-/JSON-API:
* **`sw/cpu_8bit_web.c` (Standard-Deployment `--c`):** Statisch gelinktes ARMv7-C-Binary (`arm-linux-gnueabihf-gcc -O2 -static`, `552 KB`), minimale Latenz und nahezu keine CPU-Last beim 200-ms-Polling. Der Status-Ausgabepuffer wurde für das erweiterte `"pipe"`-Objekt auf `2048` Bytes vergrößert.
* **`sw/cpu_8bit_web.py` (Fallback `--py`):** Identische Register-Dekodierung über `mmap` auf `/dev/mem`, ideal für schnelle Experimente direkt auf dem Board ohne Cross-Compiler.

---

## 2. REST-API & JSON-Datenmodell

### `GET /api/status`
Liest alle Register `0x00`–`0x2C` atomar in einem Durchgang aus und liefert ein vollständig abwärtskompatibles JSON mit zusätzlichem `"pipe"`-Block:

```json
{
  "enable": 0, "speed_sel": 1, "acc": 0, "ir": 224, "pc": 3, "phase": 3,
  "flag_z": 1, "flag_c": 0, "flag_n": 0, "halted": 0,
  "alu_res": 0, "out_reg": 0, "mem_data": 29, "mar": 0,
  "cycle_cnt": 3, "backend": "c",
  "ram": [29,224,46,122,79,30,77,31,78,96,30,224,240,0,1,0],
  "pipe": {
    "if_pc": 3, "id_pc": 2, "ex_pc": 1,
    "if_valid": 1, "id_valid": 1, "ex_valid": 1,
    "ex_fwd": 0, "last_flush": 0, "id_ir": 46,
    "flush_cnt": 0, "fwd_cnt": 0, "instr_cnt": 1
  }
}
```

### `POST /api/ctrl`
Steuert Laufzustand, Taktfrequenz und Einzeltakte über Register `0x00`:

| JSON-Key | Wertebereich | Wirkung in Hardware |
| :--- | :---: | :--- |
| `"enable"` | `0` / `1` | `0` = HOLD / PAUSE, `1` = RUN |
| `"speed_sel"` | `0`..`3` | `0` = 1 Hz, `1` = 2 Hz, `2` = 10 Hz, `3` = Full (100 MHz) |
| `"step"` | `1` | Einzeltakt-Puls (`step_trig` Bit 3, genau +1 Pipeline-Takt) |
| `"reset"` | `1` | Soft-Reset (`soft_reset` Bit 4, RAM bleibt erhalten) |
| `"reload_demo"` | `1` | Demo-Fibonacci-RAM neu laden + Reset (`ram_reload_demo` Bit 5) |

### `POST /api/ram`
Schreibt einzelne Bytes oder ein komplettes 16-Byte-Programm ins RAM (`0x10`):
* **Einzelbyte:** `{"addr": 15, "val": 170}` (schreibt `0xAA` nach `ram[0xF]`)
* **Komplettes Programm:** `{"bytes": [85,79,47,...], "reset": 1}` (schreibt alle 16 Bytes und führt optional direkt einen Soft-Reset aus)

---

## 3. Dashboard-Komponenten (`sw/web_ui/`)

Das UI passt sich vollständig in ein scrollfreies **`100dvh`-Vier-Karten-Grid** ein (mit automatischem Responsive-Fallback auf kleineren Displays) und visualisiert alle Pipeline-Vorgänge in Echtzeit:

### A) Kopfzeile & Steuerleiste
* **Live-Badges:** `HOLD`/`RUN`/`HALTED`, `PIPE 1/3`..`3/3` (wechselt bei Ereignissen direkt auf `FWD`, `FLUSH` oder `HALT`), `CYC`, **`IPC`** ($\text{instr\_cnt} / \text{cycle\_cnt}$), **`FL:x FW:y`** (kumulierte Flush- und Forwarding-Zähler) sowie `SRV: C/PY`.
* **Steuerung:** `RUN`, `Single-Step (+1 Takt)`, `+1 Befehl` (Retire-Step, `Alt+S`), Taktwahl (`1 Hz`, `2 Hz`, `10 Hz`, `Full (100 MHz)`), `Soft-Reset (RAM behalten)`, `Demo-RAM laden`, `Assembler (ASM)` (`Alt+A`) und `Debug & OUT` (`Alt+D`).

### B) 3-Stufen-Pipeline-Leiste & Datenpfad-SVG (oben links)
Direkt über dem Datenpfad-SVG zeigt `#pipe-bar` die drei Stufen nebeneinander:
* **`1. IF`** (Cyan): aktuelle Fetch-Adresse `if_pc` und das anstehende Befehlsbyte inkl. Disassembly (`ram[if_pc]`).
* **`2. ID/RD`** (Amber): dekodierter Befehl (`id_pc`, `id_ir`), oder gestrichelt als **`BUBBLE`** wenn `id_valid == 0`.
* **`3. EX/WB`** (Blau): aktuell in der ALU/Writeback befindlicher Befehl (`ex_pc`, `ir`), oder **`BUBBLE`** wenn `ex_valid == 0`.
* **Hazard- & IPC-Box rechts:**
  * **`STA->ID FWD`** leuchtet amber auf, sobald `ex_fwd == 1` (zusätzlicher amber Rahmen an Stufe 3).
  * **`FLUSH`** leuchtet rot auf, sobald `last_flush == 1` (zusätzlicher roter Rahmen an den geleerten Stufen).
  * **`INS:x · IPC:y.yy`** zeigt laufend die fertiggestellten Befehle und den aktuellen IPC.
* **Datenpfad-SVG:** Beleuchtet simultan alle aktiven Blöcke und Busse aller gerade belegten Pipeline-Stufen (z. B. `PC -> RAM` für `IF`, `MAR -> RAM -> IR` für `ID` und `RAM/ACC -> ALU -> FLAGS` für `EX` gleichzeitig; bei `ex_fwd` zusätzlich den Forwarding-Pfad von `ACC`).

### C) Register & Flip-Flops (oben rechts)
Zeigt `ACC`, `IR`, `OUT`, `ALU`, `PC`, `MAR` und die Flags (`H, N, C, Z`) auf Bit-Ebene; jedes seit dem letzten Sample umgekippte Bit wird kurzzeitig mit einem gelben Rahmen hervorgehoben.

### D) Digital-Waveform (unten links)
Zeichnet bis zu 24 historische Takte über 5 Spuren (`clk_div`, `PIPE`, `IF_PC`, `ACC`, `OUT / C,Z`):
* **`PIPE`-Spur:** Zeigt farbcodiert `1/3`, `2/3`, `3/3` (blau), **`FW!`** (amber bei Forwarding), **`FL!`** (rot bei Branch-Flush) sowie **`HLT`** (rot am abschließenden Halt-Takt).
* **HALT-Freeze-Schutz:** Sobald `st.halted == 1` erreicht ist, schaltet die Anzeige auf `HALTED (FROZEN)` und `pushWave()` blockiert weitere Duplikat-Spalten — auch wenn im Halt-Zustand nochmals auf `Single-Step` geklickt wird.

### E) 16-Byte RAM & Disassembler (unten rechts)
* Zeigt für jede Adresse `0x0`–`0xF` Hex-Wert (direkt inline editierbar), Dezimalwert, Disassembly und die gleichzeitigen Pipeline-Zeiger **`EX`**, **`ID`**, **`IF`** und **`M`** (`MAR`-Datenzugriff) mit entsprechender Zeilenfärbung.
* **5 Schnell-Presets:** `Fibonacci`, `Zähler`, `Shift`, `Countdown` und das neue Pipeline-Demo-Preset **`Fwd/Flush`** (`0x55, 0x4F, 0x2F, 0x4E, 0x3E, 0x88, 0x5F, 0xE0, 0x1E, 0xE0, 0xF0, 0x00, 0x00, 0x00, 0x55, 0xAA`), mit dem sich sowohl zwei `STA -> ID`-Forwardings als auch ein Branch-Flush samt Trap-Vermeidung direkt im Einzelschritt vorführen lassen.

---

## 4. Browser-Assembler & Source-Level-Debugger

### Browser-Assembler (`Alt+A`)
* Zwei-Pass-Assembler im Browser mit Labels, Kommentaren (`;`), Dezimal-/Hex-Literalen (`0x..`), `DB`-Direktiven und Live-Fehlerprüfung.
* 100 % unveränderte Opcode-Kodierung — alle Programme aus `12_cpu_8bit_asm` (inkl. ASCII-Ausgaben) lassen sich direkt assemblieren und per `Ins RAM flashen` oder `Flashen & RUN` auf die Pipeline-CPU übertragen.

### Source-Level-Debugger & OUT-Konsole (`Alt+D`)
Für die 3-stufige Pipeline wurden vier Kernfunktionen gezielt erweitert:

1. **Gleichzeitiges Mehrfarben-Stage-Highlighting (`renderDebugView`):**
   Im Quellcode- bzw. Disassembly-Listing werden alle drei aktiven Pipeline-Adressen parallel markiert (`EX` blau, `ID` amber, `IF` cyan, `MAR` grün) und zusätzlich mit **`FWD`**-, **`FLUSH`**- oder **`HLT`**-Badges versehen. Die Kopfzeile zeigt kompakt `IF:0x3 ID:0x2 EX:0x1 | CYC 3 | IPC 0.33`.
2. **Pipeline-sicheres Retire-Stepping (`stepInstruction`, `+1 Befehl` / `Alt+S`):**
   Statt starr 3 Phasen durchzuklicken, merkt sich der Debugger `startInstr = st.pipe.instr_cnt` und sendet maximal 4 Einzeltakte, bis `instr_cnt > startInstr` (oder `halted`) eintritt. Im vollen Pipeline-Betrieb entspricht `+1 Befehl` damit exakt **1 Takt**, während nach Reset oder Branch-Flush die 2 Fill-Bubbles automatisch bis zum nächsten abgeschlossenen Befehl überbrückt werden.
3. **Exakte `OUT`-Erkennung (`processStatusSample`):**
   Ein `OUT`-Befehl wirkt in der Pipeline genau in dem Takt, in dem er `EX/WB` verlässt. Der Debugger prüft daher sowohl Wertänderungen (`st.out_reg !== prevStatus.out_reg`) als auch `prevPipe.ex_valid && ((prevStatus.ir >> 4) === 0xE)` — dadurch werden auch identische Folge-Ausgaben (wie die beiden aufeinanderfolgenden `1` in Fibonacci) zuverlässig protokolliert, während geflushte `OUT`-Befehle hinter einem Sprung niemals im Log erscheinen.
4. **Stage-genaue Breakpoints:**
   Ein Klick auf eine Programmzeile setzt einen Breakpoint (roter Punkt). Im `RUN`-Modus hält der Debugger exakt dann an, wenn die Zieladresse gültig in **Stufe 3 (`EX/WB`)** eintritt (`p.ex_valid && breakpoints.has(p.ex_pc)`), also unmittelbar bevor der Befehl ausgeführt wird.