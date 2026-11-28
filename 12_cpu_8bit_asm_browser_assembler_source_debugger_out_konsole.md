# 12_cpu_8bit_asm – Browser-Assembler, Source-Level-Debugger & OUT-Konsole

Dieses Dokument beschreibt die Software-Erweiterung des Projekts **`12_cpu_8bit_asm`** (auf Basis von `11_cpu_8bit`) für das **MYIR MYD-C7Z020-V2** (Zynq-7020) Entwicklungssystem.

Die Erweiterung ergänzt das bestehende 2×2-Web-Dashboard um eine komplette Entwicklungsumgebung direkt im Browser — **ohne Änderung am FPGA-Bitstream** (reiner Software-Build per `--sw-only` und Fast-Deploy per `--no-fpga`).

---

## 1. System- & Projektübersicht

| Parameter | Wert |
| :--- | :--- |
| **Projektname** | `12_cpu_8bit_asm` |
| **Ziel-Hardware** | MYIR MYD-C7Z020-V2 (Xilinx Zynq-7020) |
| **AXI4-Lite Basisadresse** | `0x43C00000` |
| **Board-IP & Web-UI** | `http://192.168.50.2:8080` (`root@192.168.50.2`) |
| **WSL2-Projektpfad** | `~/FPGA/myd_c7z020_v2/12_cpu_8bit_asm` |
| **Vivado-Build-Verzeichnis** | `D:\vivado_builds\12_cpu_8bit_asm` (`/mnt/d/vivado_builds/12_cpu_8bit_asm`) |
| **Board-Deploy-Verzeichnis** | `/root/12_cpu_8bit_asm` |
| **CPU-Architektur** | 8-Bit SAP-1 / Von-Neumann Soft-Core, 16×8-Bit RAM, 3-Phasen-FSM (`FETCH`, `DECODE`, `EXECUTE`) |

### Projektstruktur

```text
12_cpu_8bit_asm/
├── README.md
├── asm_examples/                 # Beispielprogramme (.asm)
│   ├── fibonacci.asm
│   ├── counter.asm
│   └── countdown.asm
├── rtl/                          # Unveraenderte Verilog-Quellen (cpu_8bit_core.v, axi_cpu_8bit.v)
├── vivado/                       # build.tcl, report_metrics.tcl
├── sw/
│   ├── web_ui/
│   │   ├── index.html.in         # + ASM-Modal & Debug/OUT-Dock
│   │   ├── style.css.in          # + Dark-Mode-Styles fuer Toolbar, Modal & Dock
│   │   └── dashboard.js.in       # + 2-Pass-Assembler, Source-Debugger, +1 Befehl, OUT-Konsole
│   └── ...                       # C-HTTP-Server, Python-Fallback, UI-Bundler
├── scripts/                      # build.sh (--sw-only), deploy.sh (--c --no-fpga), Reports
├── docs/                         # Dokumentation & Schaltplaene
└── out/                          # cpu_8bit.bit.bin, cpu_8bit_web (ARMv7), ui_assets/
```

---

## 2. Überblick der neuen Funktionen

Alle neuen Elemente sind nahtlos in die bestehende Steuerleiste integriert, sodass das viewport-gesperrte `100dvh`-2×2-Dashboard (Datenpfad, Register, Digital-Waveform, 16-Byte-RAM) unverändert bleibt:

* **Toolbar-Buttons:**
  * **`+1 Befehl`** (direkt neben *Single-Step (+1 Phase)*): Führt exakt eine komplette Instruktion bis zum nächsten `FETCH` aus.
  * **`Assembler (ASM)`** (neben *Demo-RAM laden*): Öffnet das Modal-Overlay mit 2-Pass-Assembler, Live-Vorschau, Presets und RAM-Disassembler.
  * **`Debug & OUT (n)`** (rechts neben *Assembler (ASM)*): Öffnet das nicht-blockierende untere Dock-Panel für Source-Level-Debugging, Software-Breakpoints und die OUT-Verlaufskonsole inklusive Live-Zähler der protokollierten Ausgaben.

### Tastenkürzel

| Shortcut | Aktion |
| :--- | :--- |
| `Alt + A` | Browser-Assembler öffnen / schließen |
| `Alt + D` | Debug- & OUT-Dock ein- / ausblenden |
| `Alt + S` | **+1 Befehl** (Instruction-Step) ausführen |
| `Esc` | Geöffnetes Assembler-Modal schließen |

---

## 3. Der 2-Pass Browser-Assembler

Der Assembler übersetzt symbolischen 8-Bit-Assembly-Code live bei jeder Eingabe in ein 16-Byte-RAM-Abbild (`Uint8Array(16)`) und prüft unmittelbar auf Syntax-, Adress- und Überlauffehler.

### 3.1 Arbeitsweise (2-Pass-Verfahren)

1. **Pass 1 (Symbol- & Adressauflösung):**
   * Entfernt Kommentare (`;` und `//`), ohne Zeichenliterale wie `';'` zu beschädigen.
   * Erfasst Labels (`name:`) und ordnet ihnen die aktuelle Programmzähler-Adresse (`0x0`–`0xF`) zu.
   * Wertet `.ORG`-Direktiven aus und zerlegt Befehle sowie `DB`-Datenbytes in eine Zwischenliste.
   * Prüft auf doppelte Labels, unbekannte Mnemonics und RAM-Überläufe (> 16 Bytes).
2. **Pass 2 (Code-Generierung & Source-Map):**
   * Löst alle Label-Referenzen (auch Vorwärtssprünge wie `JC done` oder `LDA a`) auf.
   * Codiert jeden Befehl als `((opcode & 0xF) << 4) | (operand & 0xF)`.
   * Erkennt Adresskollisionen (wenn `.ORG` bereits belegte Zellen überschreiben würde).
   * Erzeugt neben dem 16-Byte-Abbild eine **`addrMap[0..15]`**, die jeder RAM-Adresse ihre ursprüngliche Quellzeile und den Typ (`OP` oder `DB`) für den Source-Level-Debugger zuordnet.

### 3.2 Unterstützter Befehlssatz (16 Opcodes)

| Opcode | Mnemonic | Operand | Funktion |
| :---: | :---: | :---: | :--- |
| `0x0` | `NOP` | – | Keine Operation |
| `0x1` | `LDA addr` | 4-Bit | `ACC <- RAM[addr]` |
| `0x2` | `ADD addr` | 4-Bit | `ACC <- ACC + RAM[addr]` (aktualisiert Flags) |
| `0x3` | `SUB addr` | 4-Bit | `ACC <- ACC - RAM[addr]` (aktualisiert Flags) |
| `0x4` | `STA addr` | 4-Bit | `RAM[addr] <- ACC` |
| `0x5` | `LDI imm` | 4-Bit | `ACC <- imm` (`0..15`) |
| `0x6` | `JMP addr` | 4-Bit | Unbedingter Sprung (`PC <- addr`) |
| `0x7` | `JC addr` | 4-Bit | Sprung bei gesetztem Carry-Flag (`C = 1`) |
| `0x8` | `JZ addr` | 4-Bit | Sprung bei gesetztem Zero-Flag (`Z = 1`) |
| `0x9` | `AND addr` | 4-Bit | `ACC <- ACC & RAM[addr]` |
| `0xA` | `OR addr` | 4-Bit | `ACC <- ACC | RAM[addr]` |
| `0xB` | `XOR addr` | 4-Bit | `ACC <- ACC ^ RAM[addr]` |
| `0xC` | `NOT` | – | Bitweise Invertierung (`ACC <- ~ACC`) |
| `0xD` | `SHL` | – | Logischer Linksshift (`ACC << 1`, MSB in Carry) |
| `0xE` | `OUT` | – | `OUT_REG <- ACC` |
| `0xF` | `HLT` | – | CPU anhalten (`halted = 1`) |

### 3.3 Direktiven, Zahlenformate & Kommentare

* **Labels:** `loop:`, `done:`, `a:` (Groß-/Kleinschreibung wird ignoriert; Label kann allein auf einer Zeile oder vor einem Befehl/`DB` stehen).
* **Adress-Direktive:** `.ORG <0..15>` bzw. `ORG <0..15>` setzt den aktuellen Adresszeiger.
* **Datenbytes:** `DB`, `.DB` oder `.BYTE` akzeptiert einzelne oder kommaseparierte 8-Bit-Werte (`0..255`), Labels oder ASCII-Zeichen.
* **Zahlen- & Zeichenformate:**
  * Hexadezimal: `0x0D`, `$0D`
  * Binär: `0b00001101`
  * Dezimal: `15` (optional mit `#`-Präfix wie `#15`)
  * ASCII-Zeichen: `'Z'`, `'!'`, `'\n'`, `'\r'`, `'\t'`
* **Kommentare:** `; Kommentar` oder `// Kommentar`.

### 3.4 Integrierte Presets & Disassembler

* **Preset: Fibonacci (Demo):** Berechnet die Fibonacci-Folge bis `233` (`16/16` Bytes belegt: `1D E0 2E 7A 4F 1E 4D 1F 4E 60 1E E0 F0 00 01 00`).
* **Preset: Zähler (+1):** Endlos-Inkrementierer mit fortlaufender `OUT`-Ausgabe.
* **Preset: Countdown (15..0):** Lädt `15` per `LDI 15`, zählt mit `SUB one` herunter und stoppt über `JZ done` + `HLT`.
* **Preset: ASCII "ZYNQ!":** Demonstriert Zeichenliterale (`DB 'Z'` … `DB '!'`) und die ASCII-Textausgabe in der OUT-Konsole (`16/16` Bytes).
* **Aus RAM lesen:** Liest über `GET /api/status` den aktuellen Inhalt des FPGA-RAMs und disassembliert alle 16 Bytes wieder in editierbaren Assembly-Code.

### 3.5 Flash-Ablauf ins FPGA-RAM

Beim Klick auf **In RAM laden & Reset** bzw. **Laden & Start** führt der Browser eine deterministische Sequenz aus:

1. CPU pausieren: `POST /api/ctrl` mit `{"enable": 0}`.
2. Alle 16 Adressen (`0x0`–`0xF`) schreiben: `POST /api/ram` mit `{"addr": i, "val": byte, "data": byte}`.
3. Source-Map im Debugger aktivieren (`activeSource`) und bei aktiviertem *Auto-Clear* die OUT-Konsole leeren.
4. Soft-Reset (und optional sofortiger Start): `POST /api/ctrl` mit `{"enable": 0|1, "soft_reset": 1}`.

---

## 4. Instruction-Step (`+1 Befehl`)

Da die CPU jeden Befehl in drei Taktphasen abarbeitet (`0 = FETCH`, `1 = DECODE`, `2 = EXECUTE`), erfordert ein kompletter Befehl normalerweise drei Klicks auf *Single-Step (+1 Phase)*.

Die Funktion `stepInstruction()` automatisiert diesen Ablauf:

1. Liest den aktuellen Zustand (`phase`, `halted`) über `GET /api/status`.
2. Bricht sofort ab, falls `halted == 1`.
3. Berechnet die exakt fehlenden Phasen bis zum nächsten `FETCH`:
   * `phase == 0 (FETCH)` → 3 Schritte
   * `phase == 1 (DECODE)` → 2 Schritte
   * `phase == 2 (EXECUTE)` → 1 Schritt
4. Sendet diese Anzahl sequentieller `POST /api/ctrl` (`{"enable": 0, "step": 1}`), aktualisiert nach jeder Phase die OUT-Konsole sowie den Debugger und stoppt vorzeitig, falls `HLT` erreicht wird.

---

## 5. Source-Level-Debugger & Software-Breakpoints

Das untere Dock-Panel (`#dbg-dock`, umschaltbar zwischen kompakter und großer Ansicht) bleibt **nicht-blockierend** über der unteren Bildschirmkante, sodass Datenpfad, Register und Toolbar parallel im Blick bleiben.

### 5.1 Live-Zeilenzuordnung (`PC` & `MAR`)

* **Aktiver Befehl (`active-pc`, blau):**
  * In `FETCH` (`phase == 0`) wird die Adresse `pc` markiert (der Befehl, der als Nächstes geholt wird).
  * In `DECODE` (`phase == 1`) und `EXECUTE` (`phase == 2`) wurde der Programmzähler von der Hardware bereits inkrementiert; der Debugger markiert daher `(pc - 1) & 0xF` als aktuell in Ausführung befindliche Instruktion.
* **Datenzugriff (`active-data`, grün, Tag `MAR`):**
  * Bei Speicherbefehlen (`LDA`, `ADD`, `SUB`, `STA`, `AND`, `OR`, `XOR`) in `DECODE`/`EXECUTE` hebt der Debugger zusätzlich die durch `MAR` adressierte Datenzeile (z. B. `b: DB 0x01`) hervor.
* **Automatische Source-Map-Erkennung:**
  * Vergleicht die Instruktionsbytes (`kind === "OP"`) im FPGA-RAM gegen das zuletzt geflashte Programm oder die integrierten Presets (veränderliche `DB`-Variablen werden bewusst ignoriert, sodass selbstmodifizierende Daten wie `a`, `b`, `tmp` bei Fibonacci die Source-Map nicht ungültig machen).
  * Wurde das RAM manuell im Hex-Editor geändert, schaltet der Debugger automatisch auf **`Live-Disasm`** um.

### 5.2 Software-Breakpoints

* Ein Klick auf eine beliebige Zeile (`0x0`–`0xF`) im Debugger setzt oder entfernt einen Breakpoint (roter Indikatorpunkt).
* Im 250-ms-Status-Polling prüft der Browser bei laufender CPU (`enable == 1`, `halted == 0`) an der `FETCH`-Grenze (`phase == 0`), ob `pc` in der Breakpoint-Menge liegt.
* Bei einem Treffer sendet das UI automatisch `POST /api/ctrl` (`{"enable": 0}`) und zeigt die Trefferadresse samt Zykluszähler (`Breakpoint erreicht @ 0x... (CYC ...)`) an.

> **Technischer Hinweis:** Da diese Breakpoints softwareseitig über das Browser-Polling (250 ms) arbeiten, funktionieren sie zuverlässig bei langsamen Taktraten (z. B. `1 Hz`–`2 Hz`) und beim schrittweisen Debuggen. Für exaktes Anhalten bei hohen Taktraten wäre ein Hardware-Komparator im Verilog-Core erforderlich.

---

## 6. OUT-Verlaufskonsole & ASCII-Ansicht

Im rechten Bereich des Debug-Docks zeichnet die OUT-Konsole sämtliche Ausgaben des `OUT`-Registers lückenlos auf:

* **Zuverlässige Erfassung auch bei identischen Folgewerten:**
  * Viele einfache Logger erkennen nur Wertänderungen (`out_reg != prev.out_reg`) und verpassen dadurch z. B. das zweite `1` in der Fibonacci-Folge (`0, 1, 1, 2, 3...`).
  * Diese Implementierung loggt sowohl bei einem Wertwechsel von `out_reg` **als auch** beim Abschluss jedes `OUT`-Befehls (`(ir >> 4) == 0xE` beim Übergang auf `phase == 0` mit neuem `cycle_cnt`).
* **Darstellung:**
  * **ASCII-Textstream:** Zeigt alle empfangenen Bytes als zusammenhängenden String (druckbare Zeichen `0x20`–`0x7E` direkt, `\n`/`\t` interpretiert, Steuerzeichen als `·`) inklusive **ASCII kopieren**-Button.
  * **Detailtabelle:** Spalten `#`, `CYC`, `DEC`, `HEX`, `BIN` (8-Bit) und `ASCII` (max. 512 Einträge im Ringpuffer).
  * **Auto-Clear bei Reset:** Erkennt automatisch einen Rücksprung von `cycle_cnt` (Soft-Reset oder Neu-Flash) und leert die Konsole auf Wunsch selbstständig.

---

## 7. Build-, Deploy- & Verifikations-Workflow

Da ausschließlich die Web-UI-Templates unter `sw/web_ui/` erweitert wurden, entfällt die komplette Vivado-Synthese.

### 7.1 Software-Only Build & Fast-Deploy

```bash
cd ~/FPGA/myd_c7z020_v2/12_cpu_8bit_asm
./scripts/build.sh --sw-only
./scripts/deploy.sh --c --no-fpga
```

* **`--sw-only`:** Bundelt `index.html.in`, `style.css.in` und `dashboard.js.in` neu in `web_ui_bundle.h`, baut das ARMv7-C-Binary `out/cpu_8bit_web` und aktualisiert `ui_assets/`.
* **`--c --no-fpga`:** Stoppt laufende Dashboard-Instanzen auf `192.168.50.2`, überspringt das Neuladen des FPGA-Bitstreams (`/lib/firmware/cpu_8bit.bit.bin`), überträgt Server und Assets nach `/root/12_cpu_8bit_asm` und startet den Dienst auf Port `:8080`.

### 7.2 Verifikation per `curl`

```bash
# 1) Pruefen, ob alle neuen UI-Anker im ausgelieferten HTML vorhanden sind
curl -s http://192.168.50.2:8080/ | grep -oE "dbg-dock|out-ascii-stream|asm-open-btn" | sort -u

# 2) Einzelnen Hardware-Step ausloesen und Status pruefen
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":0,"step":1}' http://192.168.50.2:8080/api/ctrl; echo ""
curl -s http://192.168.50.2:8080/api/status; echo ""
```

Anschließend im Browser `http://192.168.50.2:8080` per **`Ctrl + F5`** neu laden.

---

## 8. Implementierungshinweise & Wartbarkeit

* **Idempotente Marker-Blöcke:** Die gesamte Erweiterung ist in `index.html.in`, `style.css.in` und `dashboard.js.in` sauber zwischen den Markern `BROWSER_ASSEMBLER_START` und `BROWSER_ASSEMBLER_END` gekapselt. Sie lässt sich dadurch jederzeit rückstandsfrei aktualisieren oder entfernen, ohne das Basis-Dashboard zu verändern.
* **Schnittstellen-Adapter:** Die Kommunikation mit dem Backend ist zentral in drei kleinen Funktionen gekapselt (`fetchStatus()`, `writeRamByte(addr, val)` und `postCtrl(payload)`), sodass zukünftige API-Änderungen (z. B. ein atomarer Bulk-Endpoint `POST /api/ram/bulk`) an einer einzigen Stelle angepasst werden können.

### Mögliche nächste Ausbaustufen

1. **`.asm`-Datei-Import/Export & `localStorage`-Bibliothek** direkt im Assembler-Modal (`--sw-only`).
2. **Atomarer RAM-Bulk-Endpoint** (`POST /api/ram/bulk`) im C- und Python-Backend für Einzel-Request-Flashing.
3. **Hardware-Breakpoints (`0x14 BRK_CTRL`) & ALU-Timing-Optimierung** (Speisung aus registriertem `mem_data` statt kombinatorischem `ram[operand]`) mit anschließender Vivado-Timing-Messung.
4. **Nachfolgeprojekt (`13_cpu_8bit_256b`):** Erweiterung auf 256 Bytes Zynq-BRAM, 8-Bit-Adressraum und `CALL`/`RET`-Hardware-Stack.
