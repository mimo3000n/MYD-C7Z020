# Web-Dashboard Handbuch (`http://192.168.50.2:8080`)

Das Web-Dashboard ist direkt an die reale Zynq-7020-Hardware gekoppelt. Jeder Tastendruck löst echte AXI4-Lite-Schreibzugriffe auf `0x43C00000` aus, und alle Anzeigen spiegeln den tatsächlichen Flip-Flop-Zustand im FPGA wider.

Das Layout passt sich dynamisch per Viewport-CSS (`100dvh` und `clamp()`) an die Browser-Fenstergröße an, sodass auf Desktop-Monitoren alle vier Funktionsbereiche ohne Scrollen gleichzeitig sichtbar sind. Bei kleinen Fenstern (Breite $< 1050\text{ px}$ oder Höhe $< 680\text{ px}$) schaltet es automatisch auf eine scrollbare Einspalten-Ansicht um.

---

## 1. Kopfzeile & Steuerleiste

### Status-Badges (oben rechts)

| Badge | Bedeutung |
| :--- | :--- |
| **`RUN`** *(grün)* / **`HOLD`** *(gelb)* / **`HALT`** *(rot)* | Betriebsmodus der CPU: Freilaufend (`enable=1`), pausiert (`enable=0`) oder per `HLT`-Befehl gestoppt (`halted=1`). |
| **`FETCH`** / **`DECODE`** / **`EXECUTE`** | Aktuelle Phase der 3-Phasen-FSM. |
| **`CYC: n`** | Inhalt des Hardware-Zählers `CYCLE_CNT` (Anzahl ausgeführter Phasen seit dem letzten Reset). |
| **`SRV: C`** / **`SRV: PY`** | Zeigt an, ob der schnelle ARMv7-C-Server (`cpu_8bit_web`) oder der Python-Fallback antwortet. |

### Steuerleiste (oben links)

* **`RUN` / `STOP`:** Schaltet das Hardware-Bit `CTRL[0]` (`enable`) um. Bei `RUN` taktet der interne Taktteiler die CPU mit der gewählten Geschwindigkeit.
* **`Single-Step (+1 Phase)`:** Schickt exakt einen Hardware-Impuls auf `CTRL[3]` (`step_trig`). Die CPU führt genau **eine Phase** aus (`FETCH` $\rightarrow$ `DECODE` $\rightarrow$ `EXECUTE`) und stoppt sofort wieder. Funktioniert im `HOLD`-Modus perfekt zum schrittweisen Nachvollziehen.
* **`Takt` (`1 Hz`, `2 Hz`, `5 Hz`, `10 Hz`):** Setzt `CTRL[2:1]` (`speed_sel`) und bestimmt die Schrittfrequenz im `RUN`-Modus.
* **`Soft-Reset (RAM behalten)`:** Setzt über `CTRL[4]` alle CPU-Register (`PC`, `ACC`, `IR`, `MAR`, `OUT`, Flags, `halted`, `CYCLE_CNT`) auf `0` und die FSM auf `FETCH`, **lässt den Inhalt des 16-Byte-RAMs aber unverändert**.
* **`Demo-RAM laden`:** Löst `CTRL[5]` aus, überschreibt den gesamten 16-Byte-RAM in Hardware mit dem voreingestellten Fibonacci-Programm und führt einen Reset aus.

---

## 2. Die vier Bereiche des Dashboards im Detail

### Bereich 1 (oben links): CPU-Datenpfad (Live-Phase & Busse)

Das interaktive SVG-Blockdiagramm zeigt die sieben Hardware-Einheiten der CPU und hebt in Echtzeit jene Blöcke und Datenbusse blau hervor, die in der aktuellen Phase aktiv sind:

* **In `FETCH`:** `PC` $\rightarrow$ `MAR` $\rightarrow$ `RAM` $\rightarrow$ `IR` leuchten auf. Man sieht direkt, welcher Befehl aus dem Speicher ins Befehlsregister wandert.
* **In `DECODE`:** `IR` $\rightarrow$ `Control FSM` und der Operandenpfad auf `MAR` / `RAM` sind aktiv; im `IR`-Block wird das aktuelle Byte bereits als lesbarer Assembler-Befehl (z. B. `LDA 0xD`) angezeigt.
* **In `EXECUTE`:** Je nach Opcode leuchten `ACC`, `8-Bit ALU`, `FLAGS (N C Z)`, `OUT Register` oder der Rückschreibpfad von `ACC` ins `RAM` (`STA`) auf.
* **`STEP PULSE`-Anzeige (oben rechts in der Karte):** Blitzt bei jedem erkannten Zykluswechsel kurz rot auf und gibt visuelles Feedback über jeden realen FPGA-Tick.

### Bereich 2 (oben rechts): Register & Flip-Flops

Stellt die wichtigsten Hardware-Register bitweise dar (`ACC`, `IR`, `OUT`, `ALU` jeweils 8 Bit; `PC`, `MAR` jeweils 4 Bit; `FLG` mit `H`, `N`, `C`, `Z`):

* **Blaue Bit-Kacheln (`1`)** und **dunkle Kacheln (`0`)** zeigen den exakten Zustand der Flip-Flops.
* **Gelbe Umrandung (`Gelb = Bit-Wechsel`):** Jedes Bit, das sich gegenüber dem vorherigen Poll geändert hat, wird gelb hervorgehoben. So erkennt man beim Single-Stepping sofort, welche Bits z. B. bei `ADD`, `SHL` oder `NOT` gekippt sind.
* **Flags (`H N C Z`):**
  * **`H` (Halted):** CPU hat `HLT` (`0xF0`) ausgeführt.
  * **`N` (Negative):** MSB (Bit 7) des Ergebnisses ist `1` (Vorzeichenbit im Zweierkomplement).
  * **`C` (Carry):** Überlauf bei `ADD`, Unterlauf/Borrow bei `SUB` oder herausgeschobenes Bit 7 bei `SHL`.
  * **`Z` (Zero):** Ergebnis ist exakt `0x00`.

### Bereich 3 (unten links): Digital-Waveform

Ein integrierter Mini-Logic-Analyzer im Browser, der die letzten Zyklen als Signalverlauf aufzeichnet (`clk_div`, `PHASE`, `PC`, `ACC`, `OUT / C,Z`):

* **`HOLD (FROZEN)`-Verhalten:** Solange die CPU angehalten ist (`enable = 0` und `cycle_cnt` unverändert), friert die Waveform bewusst ein und scrollt nicht mit leeren Wiederholungen weiter. Erst bei einem Klick auf **Single-Step** oder im **RUN**-Modus wird ein neuer Sample-Zeitschritt angehängt.
* **`Verlauf leeren`:** Löscht den lokalen Anzeige-Puffer im Browser (die Hardware-CPU bleibt davon unberührt).

### Bereich 4 (unten rechts): 16-Byte RAM & Disassembler

Zeigt alle 16 Speicherzellen (`0x0` bis `0xF`) in Echtzeit als Tabelle:

* **Zeiger-Hervorhebung:** Die Zeile, auf die **`PC`** zeigt, ist dunkelblau hinterlegt; die aktuelle Datenadresse **`MAR`** wird bernsteinfarben markiert (stehen beide auf derselben Zelle, erscheinen `PC MAR` gemeinsam).
* **Live-Disassembler:** Die rechte Spalte übersetzt jedes Hex-Byte automatisch in das zugehörige Mnemonic (z. B. `1D` $\rightarrow$ `LDA 0xD`, `E0` $\rightarrow$ `OUT`, `F0` $\rightarrow$ `HLT`).
* **Direktes Editieren (`Hex (Edit)`):** Klicke in ein beliebiges Hex-Feld, tippe zwei Hex-Ziffern (z. B. `5A` für `LDI 0xA`) und bestätige mit **Enter** oder Fokuswechsel. Das Dashboard schreibt das Byte sofort per AXI-Register `RAM_WR` (`0x10`) in die FPGA-Flip-Flops.
* **Presets (`Fibonacci`, `Zähler`, `Shift`, `Countdown`):** Laden mit einem Klick vier praxiserprobte Beispielprogramme ins RAM und starten sauber ab `PC = 0`.

---

## 3. Die vier eingebauten Demo-Presets

| Preset | Was das Programm tut | Was man dabei gut beobachten kann |
| :--- | :--- | :--- |
| **`Fibonacci`** | Berechnet die Fibonacci-Folge (`0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233`) in den Zellen `0xD`, `0xE`, `0xF` und stoppt bei 8-Bit-Überlauf (`JC 0xA` $\rightarrow$ `OUT` $\rightarrow$ `HLT`). | Zusammenspiel aus `LDA`, `ADD`, `STA`, `OUT`, bedingtem Sprung `JC` und `HLT`. |
| **`Zähler`** | Addiert wiederholt eine Konstante auf `ACC`, gibt jeden Wert an `OUT` aus und springt in einer Schleife zurück. | Gleichmäßiges Hochzählen in den Bit-Kacheln von `ACC`/`OUT` und Überlauf-Verhalten. |
| **`Shift`** | Verschiebt ein gesetztes Bit per `SHL` schrittweise von Bit 0 nach Bit 7 und darüber hinaus ins Carry-Flag. | Wanderndes blaues/gelbes Bit im Register `ACC` sowie Auslösen des Carry-Flags `C`. |
| **`Countdown`** | Zählt einen Startwert mit `SUB` herunter und beendet die Schleife über `JZ` (Jump if Zero), sobald `0` erreicht ist. | Subtraktion in der ALU, Setzen von `Z = 1` bei `0x00` und bedingter Sprung `JZ`. |

---

## 4. Praxis-Walkthrough: Erster Single-Step-Durchlauf (Fibonacci)

So verfolgst du einen vollständigen Befehlszyklus direkt im FPGA:

1. **Startzustand herstellen:** Klicke auf **`Demo-RAM laden`** (oder **`Fibonacci`**). Status-Badges zeigen `HOLD`, `FETCH`, `CYC: 0`; `PC` und `MAR` stehen auf `0x0` (`1D` = `LDA 0xD`).
2. **1. Klick auf `Single-Step (+1 Phase)` (`CYC: 1`, `DECODE`):**
   * `FETCH` ist abgeschlossen: `IR` wechselt auf `0x1D` (`LDA 0xD`), die zugehörigen Bits in `IR` leuchten gelb auf, und `PC` steht bereits auf `0x1`.
3. **2. Klick auf `Single-Step (+1 Phase)` (`CYC: 2`, `EXECUTE`):**
   * `DECODE` hat den Operanden `0xD` extrahiert: `MAR` springt auf `0xD` (in der RAM-Tabelle wird Zeile `0xD` bernsteinfarben markiert).
4. **3. Klick auf `Single-Step (+1 Phase)` (`CYC: 3`, `FETCH`):**
   * `EXECUTE` lädt `RAM[0xD] = 0x00` in `ACC`. Da das Ergebnis Null ist, springt das Zero-Flag **`Z`** auf **`1`** (gelb umrandet). Die FSM steht wieder auf `FETCH` für `PC = 0x1` (`E0` = `OUT`).
5. **Freilauf testen:** Stelle `Takt` auf **`5 Hz`** und klicke auf **`RUN`**. Im `OUT Register` erscheinen nacheinander `0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233`, bevor `JC 0xA` den Überlauf abfängt und `HLT` das Badge oben rechts rot auf **`HALT`** setzt.

---

## 5. Eigenes Mini-Programm direkt im Browser testen

Beispiel: **Addiere `5 + 7`, gib `12` (`0x0C`) an `OUT` aus und halte an.**

1. Klicke im `HOLD`-Modus in der RAM-Tabelle nacheinander die Zellen `0x0`–`0x5` an und trage folgende Hex-Werte ein:
   * `0x0`: `55` $\rightarrow$ `LDI 0x5` (`ACC = 5`)
   * `0x1`: `25` $\rightarrow$ `ADD 0x5` (`ACC = 5 + RAM[0x5]`)
   * `0x2`: `E0` $\rightarrow$ `OUT` (`OUT = 12`)
   * `0x3`: `F0` $\rightarrow$ `HLT` (Stopp)
   * `0x4`: `00` $\rightarrow$ `NOP`
   * `0x5`: `07` $\rightarrow$ Datenwert `7`
2. Klicke auf **`Soft-Reset (RAM behalten)`** (wichtig: *nicht* `Demo-RAM laden`, sonst wird dein Programm überschrieben!).
3. Klicke **9-mal auf `Single-Step`** (oder starte mit `RUN`): Nach `OUT` steht im `OUT Register` exakt **`0x0C (12)`**, gefolgt von **`HALT`**.

---

## 6. Direkte Steuerung über die REST-API (CLI / Skripte)

Das Dashboard nutzt einfache JSON-Endpunkte, die du auch direkt aus WSL2 per `curl` ansprechen kannst:

```bash
# Kompletten CPU- und RAM-Status abfragen
curl -s http://192.168.50.2:8080/api/status; echo ""

# Einen einzelnen Phasenschritt (Single-Step) ausführen
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":0,"step":1}' http://192.168.50.2:8080/api/ctrl; echo ""

# CPU im Freilauf mit speed_sel=2 (5 Hz) starten
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"enable":1,"speed_sel":2}' http://192.168.50.2:8080/api/ctrl; echo ""
```