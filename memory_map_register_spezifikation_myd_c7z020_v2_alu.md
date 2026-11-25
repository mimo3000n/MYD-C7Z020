# Memory Map & Register-Spezifikation (`axi_alu_counter`)

Dokumentation der PS-PL-Speicherabbildung, AXI4-Lite-Register, Bitfelder, ALU-Opcodes sowie der Vivado JTAG-VIO-Schnittstelle für das **MYD-C7Z020-V2** FPGA-Projekt (8-Bit ALU & 4-Bit Zähler).

---

## 1. System-Adressraum (PS $\rightarrow$ PL)

| Komponente | Basisadresse | Bereich | Schnittstelle / Hinweis |
| :--- | :--- | :--- | :--- |
| **`axi_alu_counter`** | `0x43C0_0000` | `4 KB` (`0x43C0_0000` – `0x43C0_0FFF`) | Zynq `M_AXI_GP0` (32-Bit AXI4-Lite). Adress-Dekodierung über `S_AXI_AWADDR[3:2]` bzw. `S_AXI_ARADDR[3:2]` (spiegelt sich alle 16 Bytes im 4-KB-Fenster). |
| **Zynq SLCR** *(System Level Control)* | `0xF800_0000` | `4 KB` | PS-Register zur Freigabe der PS-PL-Level-Shifter und FPGA-Resets nach dem Laden eines Bitstreams über den FPGA-Manager. |

### Relevante SLCR-Initialisierungsregister (PS-PL Level-Shifter)

| Physische Adresse | Register | Wert | Funktion |
| :--- | :--- | :--- | :--- |
| `0xF800_0008` | `SLCR_UNLOCK` | `0x0000_DF0D` | Entsperrt Schreibzugriffe auf das SLCR |
| `0xF800_0900` | `LVL_SHFTR_EN` | `0x0000_000F` | Aktiviert alle PS-PL Level-Shifter |
| `0xF800_0240` | `FPGA_RST_CTRL` | `0x0000_0000` | Hebt die PL-Resets (`FCLK_RESET0_N`) auf |
| `0xF800_0004` | `SLCR_LOCK` | `0x0000_767B` | Sperrt das SLCR wieder |

---

## 2. AXI4-Lite Register-Map (`0x43C0_0000`)

| Offset | Physische Adresse | Name | Zugriff | Beschreibung |
| :--- | :--- | :--- | :--- | :--- |
| **`0x00`** | `0x43C0_0000` | **`CTRL`** | **R/W** | Steuerwort (Virtuelle Switches, Opcode, Operanden). *Hinweis:* Bei aktivem JTAG-VIO (`vio_override = 1`) liefert ein Read hier `vio_ctrl` statt `slv_reg0`. |
| **`0x04`** | `0x43C0_0004` | **`STATUS`** | **RO** | Live-Ergebnisse der PL (4-Bit-Zähler, 8-Bit-ALU-Ausgang `Y`, Status-Flags `Z/N/C/V`, VIO-Override-Status). |
| **`0x08`** | `0x43C0_0008` | **`OPERANDS`** | **RO** | Tatsächlich an der ALU anliegende Eingänge (`A`, `B`, `op`) nach der Multiplexer-Auswahl. |
| **`0x0C`** | `0x43C0_000C` | *(Reserviert)* | **RO** | Liefert fest `0x0000_0000`. |

---

## 3. Bit-Belegung der Register

### 3.1 `CTRL` — Offset `0x00` (`0x43C0_0000`, R/W)

```text
 31         25 24 23            16 15             8 7        4  3   2   1   0
+-------------+--+----------------+----------------+----------+---+---+---+---+
|  Reserved   |M |   manual_A     |      op_B      |manual_op |RST|CIN|COP|EN |
|   (7 Bit)   |A |    (8 Bit)     |    (8 Bit)     | (4 Bit)  |SW3|SW2|SW1|SW0|
+-------------+--+----------------+----------------+----------+---+---+---+---+
```

| Bits | Breite | Signal / Feld | Switch | Funktion |
| :--- | :--- | :--- | :--- | :--- |
| **`[31:25]`** | 7 Bit | *(Reserviert)* | — | Ungenutzt (`0`) |
| **`[24]`** | 1 Bit | **`use_manual_A`** | `SW4` | `0` = Operand `A` kommt vom 4-Bit-Zähler (oder `0x0A`, falls `SW1 = 1`); `1` = manueller Wert aus `[23:16]` |
| **`[23:16]`** | 8 Bit | **`manual_A[7:0]`** | — | Manueller Operand `A` (`0x00` .. `0xFF`, aktiv wenn `SW4 = 1`) |
| **`[15:8]`** | 8 Bit | **`op_B[7:0]`** | — | Operand `B` (`0x00` .. `0xFF`) |
| **`[7:4]`** | 4 Bit | **`manual_op[3:0]`** | — | Manueller ALU-Opcode (`0x0` .. `0xF`, aktiv wenn `SW1 = 0`) |
| **`[3]`** | 1 Bit | **`soft_rst`** | `SW3` | Synchroner Reset des 4-Bit-Zählers (`1` = Zähler auf `0` halten) |
| **`[2]`** | 1 Bit | **`cin`** | `SW2` | Carry-In-Eingang der ALU (für `ADC` und `SBC`) |
| **`[1]`** | 1 Bit | **`use_cnt_op`** | `SW1` | `0` = Zähler treibt Operand `A`; `1` = Zähler treibt `alu_op[3:0]` (und `A` fällt auf `0x0A`, sofern `SW4 = 0`) |
| **`[0]`** | 1 Bit | **`cnt_enable`** | `SW0` | `1` = 4-Bit-Zähler läuft im 1-Hz-Takt; `0` = Zähler angehalten |

---

### 3.2 `STATUS` — Offset `0x04` (`0x43C0_0004`, RO)

```text
 31                    17 16 15 14 13 12 11             4 3         0
+------------------------+--+--+--+--+--+----------------+-----------+
|        Reserved        |V |Z |N |C |V |     alu_Y      |w_count_out|
|        (15 Bit)        |IO|  |  |  |  |    (8 Bit)     |  (4 Bit)  |
+------------------------+--+--+--+--+--+----------------+-----------+
```

| Bits | Breite | Signal / Feld | Funktion |
| :--- | :--- | :--- | :--- |
| **`[31:17]`** | 15 Bit | *(Reserviert)* | Konstant `0` |
| **`[16]`** | 1 Bit | **`vio_override`** | `0` = Steuerung über Linux/AXI (`slv_reg0`); `1` = Vivado JTAG-VIO hat Kontrolle übernommen |
| **`[15]`** | 1 Bit | **`Z`** *(Zero-Flag)* | `1`, wenn ALU-Ergebnis `Y == 0x00` |
| **`[14]`** | 1 Bit | **`N`** *(Negative-Flag)* | `1`, wenn Vorzeichenbit `Y[7] == 1` |
| **`[13]`** | 1 Bit | **`C`** *(Carry-Flag)* | Übertrag, Borrow bzw. herausgeschobenes Bit der ALU |
| **`[12]`** | 1 Bit | **`V`** *(Overflow-Flag)* | Vorzeichenbehafteter Überlauf (*Two's Complement*) |
| **`[11:4]`** | 8 Bit | **`alu_Y[7:0]`** | 8-Bit-Ergebnisausgang `Y` der ALU |
| **`[3:0]`** | 4 Bit | **`w_count_out[3:0]`** | Aktueller Stand des 4-Bit-Zählers (`0x0` .. `0xF`) |

---

### 3.3 `OPERANDS` — Offset `0x08` (`0x43C0_0008`, RO)

```text
 31                  20 19      16 15             8 7              0
+----------------------+----------+----------------+----------------+
|       Reserved       |  alu_op  |      op_B      |     alu_A      |
|       (12 Bit)       | (4 Bit)  |    (8 Bit)     |    (8 Bit)     |
+----------------------+----------+----------------+----------------+
```

| Bits | Breite | Signal / Feld | Interne Zuweisung |
| :--- | :--- | :--- | :--- |
| **`[31:20]`** | 12 Bit | *(Reserviert)* | Konstant `0` |
| **`[19:16]`** | 4 Bit | **`alu_op[3:0]`** | `use_cnt_op ? w_count_out : manual_op` |
| **`[15:8]`** | 8 Bit | **`op_B[7:0]`** | `active_ctrl[15:8]` |
| **`[7:0]`** | 8 Bit | **`alu_A[7:0]`** | `use_manual_A ? manual_A : (use_cnt_op ? 8'h0A : {4'd0, w_count_out})` |

---

## 4. ALU-Opcode-Tabelle (`op[3:0]`)

| Code (Hex) | Code (Bin) | Mnemonic | Operation | Betroffene Flags |
| :--- | :--- | :--- | :--- | :--- |
| **`0x0`** | `0000` | `ADD` | $Y = A + B$ | `Z`, `N`, `C`, `V` |
| **`0x1`** | `0001` | `ADC` | $Y = A + B + \text{cin}$ | `Z`, `N`, `C`, `V` |
| **`0x2`** | `0010` | `SUB` | $Y = A - B$ | `Z`, `N`, `C`, `V` |
| **`0x3`** | `0011` | `SBC` | $Y = A - B - \neg\text{cin}$ | `Z`, `N`, `C`, `V` |
| **`0x4`** | `0100` | `AND` | $Y = A \ \& \ B$ | `Z`, `N` |
| **`0x5`** | `0101` | `OR` | $Y = A \ \mid \ B$ | `Z`, `N` |
| **`0x6`** | `0110` | `XOR` | $Y = A \oplus B$ | `Z`, `N` |
| **`0x7`** | `0111` | `NOT` | $Y = \sim A$ | `Z`, `N` |
| **`0x8`** | `1000` | `LSL` | $Y = A \ll 1$ *(Logisch Links)* | `Z`, `N`, `C` |
| **`0x9`** | `1001` | `LSR` | $Y = A \gg 1$ *(Logisch Rechts)* | `Z`, `N`, `C` |
| **`0xA`** | `1010` | `ASR` | $Y = A \ggg 1$ *(Arithmetisch Rechts)* | `Z`, `N`, `C` |
| **`0xB`** | `1011` | `ROL` | Rotation Links (8 Bit) | `Z`, `N`, `C` |
| **`0xC`** | `1100` | `ROR` | Rotation Rechts (8 Bit) | `Z`, `N`, `C` |
| **`0xD`** | `1101` | `INC` | $Y = A + 1$ | `Z`, `N`, `C`, `V` |
| **`0xE`** | `1110` | `DEC` | $Y = A - 1$ | `Z`, `N`, `C`, `V` |
| **`0xF`** | `1111` | `PASS` | $Y = A$ *(Durchschalten)* | `Z`, `N` |

---

## 5. Vivado JTAG VIO-Probes (`vio_0`)

Alternativ zum Linux-AXI-Zugriff (`0x43C0_0000`) kann die PL im Vivado Hardware Manager über JTAG (`BSCANE2`) direkt gesteuert und ausgelesen werden. Die VIO-Probes verwenden dieselbe 32-Bit-Layout-Struktur wie die AXI-Register:

| Probe-Port | Richtung | Breite | Entsprechung | Funktion |
| :--- | :--- | :--- | :--- | :--- |
| **`probe_out0`** | Output (VIO $\rightarrow$ PL) | 1 Bit | `vio_override` | `1` = JTAG-VIO übersteuert AXI (`slv_reg0`), `0` = AXI aktiv |
| **`probe_out1`** | Output (VIO $\rightarrow$ PL) | 32 Bit | `vio_ctrl` | Identische Bit-Belegung wie `CTRL` (`0x00`) |
| **`probe_in0`** | Input (PL $\rightarrow$ VIO) | 32 Bit | `vio_status` | Identische Bit-Belegung wie `STATUS` (`0x04`) |
| **`probe_in1`** | Input (PL $\rightarrow$ VIO) | 32 Bit | `vio_operands` | Identische Bit-Belegung wie `OPERANDS` (`0x08`) |

---

## 6. Schnellzugriff unter PetaLinux (`devmem`)

```bash
# Beispiel: A = 0x0A (10), B = 0x03 (3), Opcode = ADD (0x0), Zähler Enable (SW0 = 1)
devmem 0x43C00000 32 0x000A0301

# Register auslesen
devmem 0x43C00000 32   # CTRL
devmem 0x43C00004 32   # STATUS   (z. B. 0x00000030 -> Y=0x03, CNT=0x0)
devmem 0x43C00008 32   # OPERANDS (z. B. 0x00000300 -> OP=0x0, B=0x03, A=0x00)
```