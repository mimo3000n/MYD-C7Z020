# AXI4-Lite 4-Bit-Zähler (`axi_counter_4bit`) – Hardware-, Verilog- & Vivado-Dokumentation

Diese Dokumentation beschreibt den vollständigen Hardware-Teil des 4-Bit-Zähler-Projekts auf dem **MYIR MYD-C7Z020-V2 (Zynq-7020)** unter **Vivado 2026.x**. Sie erläutert sowohl den Aufbau und die Funktionsweise der beiden Verilog-Module (`counter_4bit.v` und `axi_counter_4bit.v`) als auch sämtliche Automatismen, Hilfsblöcke und Build-Schritte in Vivado (`build.tcl`).

---

## 1. Systemarchitektur & Signalfluss

Der Zynq-7000 besteht aus zwei gekoppelten Bereichen auf einem Chip:
1. **Processing System (PS):** Dual-Core ARM Cortex-A9 CPU inkl. DDR3-Controller, UART, Ethernet und Taktgeneratoren (`FCLK`).
2. **Programmable Logic (PL):** Der klassische FPGA-Teil (LUTs, Flip-Flops, Block-RAM), in dem unser eigener 4-Bit-Zähler synthetisiert wird.

Die Brücke zwischen CPU (PS) und Zähler (PL) bildet der **General-Purpose AXI Master Port (`M_AXI_GP0`)**:

```text
+------------------------------------------------------------------------------+
| Zynq-7000 Processing System (ps7)                                            |
|   M_AXI_GP0 (AXI3)  -----> [AXI Interconnect + Protocol Converter]           |
|                              (wandelt AXI3 -> AXI4-Lite)                     |
|                                         |                                    |
|                                    M00_AXI (AXI4-Lite)                       |
|                                         v                                    |
|   FCLK_CLK0 (100 MHz) -----> [axi_counter_4bit (counter_ip)]                 |
|   FCLK_RESET0_N -----------> [proc_sys_reset] ---> S_AXI_ARESETN             |
+------------------------------------------------------------------------------+
                                          |
                              enable (slv_reg0[0])
                              count_out[3:0]
                                          v
                                  [counter_4bit]
                               (100 MHz -> 1 Hz Tick)
```

* **Ziel-FPGA:** `xc7z020clg400-1` (MYD-C7Z020-V2)
* **Systemtakt (`S_AXI_ACLK`):** $100\text{ MHz}$ (`FCLK_CLK0`, Taktperiode $T = 10\text{ ns}$)
* **Reset (`S_AXI_ARESETN`):** Active-Low, synchronisiert über `proc_sys_reset` (`rst_ps7_100M`)
* **AXI-Basisadresse:** `0x43C00000` (Adressbereich: $4\text{ KiB}$ / `0x43C00000` bis `0x43C00FFF`)

---

## 2. AXI4-Lite Register-Map (`0x43C00000`)

Über den Speicherbereich ab `0x43C00000` sieht die ARM-CPU den FPGA-Zähler wie gewöhnliche Speicherzellen (Memory-Mapped I/O):

| Offset | Phys. Adresse | Register | Zugriff | Reset-Wert | Beschreibung |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0x00` | `0x43C00000` | `CTRL` (`slv_reg0`) | R/W | `0x00000000` | **Bit 0 (`enable`):** `1` = Zähler läuft, `0` = Zähler pausiert (hält aktuellen Stand und Prescaler). Bits `[31:1]` ungenutzt (Bits `[7:1]` beschreibbar). |
| `0x04` | `0x43C00004` | `COUNT` | R | `0x00000000` | **Bits `[3:0]`:** Live-Zählerstand ($0$ bis $15$, automatischer Überlauf von $15$ auf $0$). Bits `[31:4]` liefern stets `0`. |

**Hinweis zur Byte-Adressierung:** AXI verwendet Byte-Adressen, unsere Register sind jedoch $32\text{ Bit}$ ($4\text{ Bytes}$) breit. Deshalb liegen aufeinanderfolgende Register im Abstand von $4\text{ Bytes}$ (`0x00`, `0x04`, `0x08`, `0x0C`). Die untersten beiden Adressbits `[1:0]` zeigen innerhalb eines 32-Bit-Wortes auf das einzelne Byte und werden bei der Registerauswahl ignoriert; die Auswahl erfolgt ausschließlich über die Bits `[3:2]`.

---

## 3. Detaillierte Erklärung der Verilog-Codes

Das Design ist bewusst in zwei Module getrennt:
* **`counter_4bit.v`** enthält die reine Zähler- und Zeitbasis-Logik (unabhängig vom Bus-Protokoll, leicht testbar und wiederverwendbar).
* **`axi_counter_4bit.v`** kapselt diesen Zähler in einen standardkonformen AXI4-Lite-Slave, damit die Zynq-CPU über `M_AXI_GP0` darauf zugreifen kann.

---

### 3.1 Kernmodul: `counter_4bit.v`

```verilog
`timescale 1ns / 1ps

module counter_4bit #(
    parameter integer CLK_FREQ_HZ = 100_000_000,
    parameter integer TICK_HZ     = 1
)(
    input  wire       clk,
    input  wire       resetn,      // Active-low Reset (AXI-Standard)
    input  wire       enable,      // Steuerbit aus AXI-Register (Start/Stop)
    output reg  [3:0] count_out    // 4-Bit Zählerstand
);

    localparam integer MAX_COUNT = (CLK_FREQ_HZ / TICK_HZ) - 1;
    reg [31:0] prescaler;

    always @(posedge clk) begin
        if (!resetn) begin
            prescaler <= 32'd0;
            count_out <= 4'd0;
        end else if (enable) begin
            if (prescaler >= MAX_COUNT) begin
                prescaler <= 32'd0;
                count_out <= count_out + 4'd1; // Läuft bei 15 automatisch auf 0 über
            end else begin
                prescaler <= prescaler + 32'd1;
            end
        end
    end

endmodule
```

#### Funktionsweise im Detail

1. **Zeitbasis-Direktive (`` `timescale 1ns / 1ps ``):**
   Legt für Simulationen die Zeiteinheit ($1\text{ ns}$) und die zeitliche Auflösung ($1\text{ ps}$) fest. Für die spätere FPGA-Synthese hat diese Zeile keine Auswirkung auf die Hardware.

2. **Parametrisierung (`CLK_FREQ_HZ`, `TICK_HZ`) und `localparam MAX_COUNT`:**
   * Der FPGA-Takt beträgt $f_{\text{clk}} = 100\text{ MHz}$ ($100\,000\,000$ Taktflanken pro Sekunde). Würde `count_out` direkt mit `clk` hochgezählt, liefe der 4-Bit-Zähler $6{,}25\text{ Millionen}$ Mal pro Sekunde über.
   * Der **Prescaler** (Frequenzteiler) zählt daher zunächst die schnellen Systemtakte bis zu einem Schwellwert:
     $$\text{MAX\_COUNT} = \frac{\text{CLK\_FREQ\_HZ}}{\text{TICK\_HZ}} - 1 = \frac{100\,000\,000}{1} - 1 = 99\,999\,999$$
   * **Warum `- 1`?** Weil der Prescaler bei $0$ startet. Das Durchzählen der Zustände $0, 1, 2, \dots, 99\,999\,999$ benötigt exakt $100\,000\,000$ Taktzyklen, also genau $1{,}000000\text{ s}$.
   * `localparam` stellt sicher, dass `MAX_COUNT` zur Synthesezeit als feste Konstante berechnet wird und von außen nicht versehentlich überschrieben werden kann.

3. **Register (`reg [31:0] prescaler` und `output reg [3:0] count_out`):**
   * `prescaler` benötigt mindestens $\lceil \log_2(100\,000\,000) \rceil = 27\text{ Bit}$. Mit $32\text{ Bit}$ ist ausreichend Reserve für noch langsamere Tick-Raten vorhanden; ungenutzte obere Bits werden von der Vivado-Synthese bei Bedarf automatisch wegoptimiert.
   * `count_out` ist genau $4\text{ Bit}$ breit (`[3:0]`) und bildet damit die Werte $0$ bis $2^4 - 1 = 15$ (`4'b0000` bis `4'b1111`) ab.

4. **Synchroner Prozess (`always @(posedge clk)`):**
   * Alle Zustandsänderungen erfolgen strikt an der steigenden Taktflanke (`posedge clk`) mit nicht-blockierenden Zuweisungen (`<=`), wie es für Flip-Flop-Logik im FPGA erforderlich ist.
   * **Synchroner Active-Low-Reset (`if (!resetn)`):** Steht `resetn` auf `0`, werden sowohl `prescaler` als auch `count_out` auf `0` zurückgesetzt. Xilinx empfiehlt für 7-Series-FPGAs synchrone Resets, da diese sauber in die dedizierten Reset-Eingänge der Slice-Flip-Flops passen und keine asynchronen Recovery-/Removal-Timing-Probleme verursachen.
   * **Pause-Funktion (`else if (enable)`):** Ist `enable == 0`, passiert im `always`-Block nichts – die Flip-Flops halten einfach ihren aktuellen Wert (Zähler ist pausiert).
   * **Überlaufverhalten:** Wenn `count_out` bei `4'd15` (`4'b1111`) steht und `+ 4'd1` addiert wird, fällt das 5. Übertragsbit wegen der 4-Bit-Breite weg, sodass das Register exakt auf `4'd0` (`4'b0000`) überläuft.

---

### 3.2 AXI4-Lite-Slave-Wrapper: `axi_counter_4bit.v`

```verilog
`timescale 1ns / 1ps

module axi_counter_4bit #(
    parameter integer C_S_AXI_DATA_WIDTH = 32,
    parameter integer C_S_AXI_ADDR_WIDTH = 4
)(
    input  wire                              S_AXI_ACLK,
    input  wire                              S_AXI_ARESETN,
    // Write Address Channel
    input  wire [C_S_AXI_ADDR_WIDTH-1:0]     S_AXI_AWADDR,
    input  wire [2:0]                        S_AXI_AWPROT,
    input  wire                              S_AXI_AWVALID,
    output reg                               S_AXI_AWREADY,
    // Write Data Channel
    input  wire [C_S_AXI_DATA_WIDTH-1:0]     S_AXI_WDATA,
    input  wire [(C_S_AXI_DATA_WIDTH/8)-1:0] S_AXI_WSTRB,
    input  wire                              S_AXI_WVALID,
    output reg                               S_AXI_WREADY,
    // Write Response Channel
    output wire [1:0]                        S_AXI_BRESP,
    output reg                               S_AXI_BVALID,
    input  wire                              S_AXI_BREADY,
    // Read Address Channel
    input  wire [C_S_AXI_ADDR_WIDTH-1:0]     S_AXI_ARADDR,
    input  wire [2:0]                        S_AXI_ARPROT,
    input  wire                              S_AXI_ARVALID,
    output reg                               S_AXI_ARREADY,
    // Read Data Channel
    output reg  [C_S_AXI_DATA_WIDTH-1:0]     S_AXI_RDATA,
    output wire [1:0]                        S_AXI_RRESP,
    output reg                               S_AXI_RVALID,
    input  wire                              S_AXI_RREADY
);

    assign S_AXI_BRESP = 2'b00;
    assign S_AXI_RRESP = 2'b00;

    reg [31:0] slv_reg0; // Offset 0x00: Bit 0 = enable
    reg [3:0]  awaddr_reg;
    reg        aw_en;
    wire [3:0] w_count_out;

    // Write Handshake & Register-Update
    always @(posedge S_AXI_ACLK) begin
        if (!S_AXI_ARESETN) begin
            S_AXI_AWREADY <= 1'b0;
            S_AXI_WREADY  <= 1'b0;
            S_AXI_BVALID  <= 1'b0;
            aw_en         <= 1'b1;
            awaddr_reg    <= 4'd0;
            slv_reg0      <= 32'd0;
        end else begin
            if (~S_AXI_AWREADY && S_AXI_AWVALID && S_AXI_WVALID && aw_en) begin
                S_AXI_AWREADY <= 1'b1;
                S_AXI_WREADY  <= 1'b1;
                awaddr_reg    <= S_AXI_AWADDR;
                aw_en         <= 1'b0;
                if (S_AXI_AWADDR[3:2] == 2'b00 && S_AXI_WSTRB[0])
                    slv_reg0[7:0] <= S_AXI_WDATA[7:0];
            end else begin
                S_AXI_AWREADY <= 1'b0;
                S_AXI_WREADY  <= 1'b0;
                if (S_AXI_BREADY && S_AXI_BVALID) aw_en <= 1'b1;
            end

            if (~S_AXI_AWREADY && S_AXI_AWVALID && S_AXI_WVALID && aw_en)
                S_AXI_BVALID <= 1'b1;
            else if (S_AXI_BREADY && S_AXI_BVALID)
                S_AXI_BVALID <= 1'b0;
        end
    end

    // Read Handshake & Register-Mux
    always @(posedge S_AXI_ACLK) begin
        if (!S_AXI_ARESETN) begin
            S_AXI_ARREADY <= 1'b0;
            S_AXI_RVALID  <= 1'b0;
            S_AXI_RDATA   <= 32'd0;
        end else begin
            if (~S_AXI_ARREADY && S_AXI_ARVALID) begin
                S_AXI_ARREADY <= 1'b1;
                S_AXI_RVALID  <= 1'b1;
                case (S_AXI_ARADDR[3:2])
                    2'b00:   S_AXI_RDATA <= slv_reg0;
                    2'b01:   S_AXI_RDATA <= {28'd0, w_count_out}; // Live Zählerstand
                    default: S_AXI_RDATA <= 32'd0;
                endcase
            end else begin
                S_AXI_ARREADY <= 1'b0;
                if (S_AXI_RVALID && S_AXI_RREADY) S_AXI_RVALID <= 1'b0;
            end
        end
    end

    // Instanz des 4-Bit-Zählers (100 MHz -> 1 Hz)
    counter_4bit #(
        .CLK_FREQ_HZ(100_000_000),
        .TICK_HZ(1)
    ) u_counter (
        .clk       (S_AXI_ACLK),
        .resetn    (S_AXI_ARESETN),
        .enable    (slv_reg0[0]),
        .count_out (w_count_out)
    );

endmodule
```

#### Funktionsweise im Detail

1. **Das AXI4-Lite-Prinzip (5 Kanäle & `VALID`/`READY`-Handshake):**
   AXI4-Lite ist ein vereinfachter Memory-Mapped-Bus für Einzelzugriffe (keine Bursts) mit fünf unabhängigen Kanälen:
   * **Schreibpfad:**
     * `AW` (*Write Address*): Master sendet Zieladresse (`AWADDR`).
     * `W` (*Write Data*): Master sendet Nutzdaten (`WDATA`) und Byte-Maske (`WSTRB`).
     * `B` (*Write Response*): Slave quittiert den erfolgreichen Schreibvorgang (`BRESP`).
   * **Lesepfad:**
     * `AR` (*Read Address*): Master sendet Leseadresse (`ARADDR`).
     * `R` (*Read Data*): Slave liefert Daten (`RDATA`) und Status (`RRESP`).
   * **Handshake-Regel:** Eine Übertragung findet auf jedem Kanal exakt an der steigenden Flanke von `S_AXI_ACLK` statt, an der sowohl `VALID` (vom Sender) als auch `READY` (vom Empfänger) gleichzeitig `1` sind. Bleibt ein `READY`- oder `BVALID`/`RVALID`-Signal aus, wartet der Zynq-Prozessor ewig (Bus-Hänger).

2. **Automatische Schnittstellen-Erkennung in Vivado:**
   Durch die exakte Benennung der Ports nach dem Xilinx-Schema `S_AXI_ACLK`, `S_AXI_ARESETN`, `S_AXI_AWADDR`, `S_AXI_WDATA` usw. erkennt Vivado IP Integrator beim Einfügen des Moduls automatisch, dass diese 20 Einzelleitungen zusammen eine einzige **AXI4-Lite Slave-Schnittstelle (`S_AXI`)** bilden. Es ist kein eigener IP-Packager-Durchlauf nötig.

3. **Parameter & Antwort-Signale:**
   * `C_S_AXI_DATA_WIDTH = 32`: Standardbreite des Zynq-`M_AXI_GP0`-Busses.
   * `C_S_AXI_ADDR_WIDTH = 4`: Mit $4\text{ Adressbits}$ lassen sich $2^4 = 16\text{ Bytes}$ (also vier 32-Bit-Register an den Offsets `0x00`, `0x04`, `0x08`, `0x0C`) adressieren.
   * `S_AXI_BRESP = 2'b00` und `S_AXI_RRESP = 2'b00`: Im AXI-Standard bedeutet `2'b00` **`OKAY`** (Transaktion erfolgreich abgeschlossen).

4. **Schreib-Handshake (`Write Handshake & Register-Update`):**
   * Das interne Hilfsflag **`aw_en`** startet nach dem Reset mit `1` und sperrt neue Schreibzugriffe (`aw_en <= 1'b0`), sobald eine Transaktion angenommen wurde, bis deren Quittung (`BVALID && BREADY`) abgeschlossen ist. Dadurch wird verhindert, dass zwei Schreibzugriffe kollidieren.
   * Sobald der Master sowohl die Adresse (`S_AXI_AWVALID`) als auch die Daten (`S_AXI_WVALID`) anlegt und `aw_en == 1` ist, zieht der Slave für genau einen Taktzyklus **`S_AXI_AWREADY`** und **`S_AXI_WREADY`** auf `1` (gleichzeitige Übernahme von Adresse und Daten).
   * Im selben Takt prüft der Code:
     ```verilog
     if (S_AXI_AWADDR[3:2] == 2'b00 && S_AXI_WSTRB[0])
         slv_reg0[7:0] <= S_AXI_WDATA[7:0];
     ```
     * `S_AXI_AWADDR[3:2] == 2'b00` selektiert Offset `0x00` (`0x43C00000`).
     * `S_AXI_WSTRB[0]` ist das Write-Strobe-Bit für Byte 0 (Bits `[7:0]`): Nur wenn der Master dieses Byte als gültig markiert, wird `slv_reg0[7:0]` aktualisiert.
   * Ebenfalls im selben Takt wird **`S_AXI_BVALID <= 1'b1`** gesetzt, um dem Master mitzuteilen: *"Schreibzugriff verarbeitet, Antwort bereit"*. `S_AXI_BVALID` bleibt solange `1`, bis der Master mit `S_AXI_BREADY == 1` bestätigt; erst dann wird `S_AXI_BVALID` gelöscht und `aw_en` wieder auf `1` gesetzt.

5. **Lese-Handshake (`Read Handshake & Register-Mux`):**
   * Sobald der Master eine Leseadresse anlegt (`S_AXI_ARVALID == 1`) und `S_AXI_ARREADY == 0` ist, quittiert der Slave die Adresse für einen Takt mit `S_AXI_ARREADY <= 1'b1`, legt die passenden 32-Bit-Daten auf `S_AXI_RDATA` und setzt `S_AXI_RVALID <= 1'b1`.
   * **Register-Multiplexer (`case (S_AXI_ARADDR[3:2])`):**
     * `2'b00` (Offset `0x00`): Gibt das Steuerregister `slv_reg0` zurück (CPU kann auslesen, ob der Zähler aktiviert ist).
     * `2'b01` (Offset `0x04`): Verknüpft $28$ Nullbits mit dem 4-Bit-Ausgang des Zählers via Bit-Konkatenation `{28'd0, w_count_out}` zu einem sauberen 32-Bit-Wert (`0x00000000` bis `0x0000000F`).
     * `default` (Offsets `0x08`, `0x0C`): Liefert sicher `0`, ohne den Bus zu blockieren.
   * `S_AXI_RVALID` bleibt stabil auf `1`, bis der Master die Daten mit `S_AXI_RREADY == 1` abgeholt hat.

6. **Instanziierung von `counter_4bit` (`u_counter`):**
   * Verbindet den AXI-Takt (`S_AXI_ACLK`) und den synchronisierten AXI-Reset (`S_AXI_ARESETN`) direkt mit dem Zählerkern – alles läuft in derselben Taktdomäne (kein fehleranfälliges Clock-Domain-Crossing nötig).
   * `slv_reg0[0]` steuert `enable`, und `w_count_out` führt den aktuellen Zählerstand zurück in den Lese-Multiplexer.

---

## 4. Das "Rundherum" in Vivado: Block Design, Automatisierung & Build-Flow (`build.tcl`)

Statt das Projekt manuell in der Vivado-GUI zusammenzuklicken, erzeugt `build.tcl` das gesamte FPGA-Projekt reproduzierbar im Batch-Modus (`vivado -mode batch -source build.tcl`).

### 4.1 Das komplette Skript `build.tcl`

```tcl
set proj_name "myd_counter_proj"
set proj_dir  "./vivado_out"
set part_name "xc7z020clg400-1"

create_project -force $proj_name $proj_dir -part $part_name
add_files -norecurse {counter_4bit.v axi_counter_4bit.v}
update_compile_order -fileset sources_1

# Block Design erstellen
create_bd_design "system"

# Zynq7 Processing System hinzufügen (FCLK_CLK0 = 100 MHz & M_AXI_GP0)
create_bd_cell -type ip -vlnv xilinx.com:ip:processing_system7:5.5 ps7
apply_bd_automation -rule xilinx.com:bd_rule:processing_system7 \
    -config {make_external "FIXED_IO, DDR" Master "Disable" Slave "Disable"} [get_bd_cells ps7]
source myir_ps7_preset.tcl
set_property -dict [list CONFIG.PCW_USE_M_AXI_GP0 {1} CONFIG.PCW_FPGA0_PERIPHERAL_FREQMHZ {100}] [get_bd_cells ps7]

# AXI-4-Bit-Zähler als RTL-Modul ins Block Design einfügen
create_bd_cell -type module -reference axi_counter_4bit counter_ip

# AXI-Verbindung & Reset automatisch verdrahten
apply_bd_automation -rule xilinx.com:bd_rule:axi4 \
    -config {Clk_master {Auto} Clk_slave {Auto} Clk_xbar {Auto} Master {/ps7/M_AXI_GP0} Slave {/counter_ip/S_AXI} ddr_seg {Auto} intc_ip {New AXI Interconnect} master_apm {0}} \
    [get_bd_intf_pins counter_ip/S_AXI]

# Feste Basisadresse 0x43C00000 zuweisen
assign_bd_address -target_address_space /ps7/Data [get_bd_addr_segs counter_ip/S_AXI/reg0] -force
set_property offset 0x43C00000 [get_bd_addr_segs {ps7/Data/SEG_counter_ip_reg0}]
set_property range 4K [get_bd_addr_segs {ps7/Data/SEG_counter_ip_reg0}]

validate_bd_design
save_bd_design

# Top-Level Wrapper generieren
make_wrapper -files [get_files $proj_dir/$proj_name.srcs/sources_1/bd/system/system.bd] -top
add_files -norecurse $proj_dir/$proj_name.gen/sources_1/bd/system/hdl/system_wrapper.v
set_property top system_wrapper [current_fileset]
update_compile_order -fileset sources_1

# Synthese, Implementation & Bitstream
launch_runs impl_1 -to_step write_bitstream -jobs 8
wait_on_run impl_1

# XSA inkl. Bitstream für PetaLinux exportieren
write_hw_platform -fixed -include_bit -force -file ./system.xsa
exit
```

---

### 4.2 Was Vivado in jedem Schritt im Hintergrund tut

#### Schritt 1: Projekt anlegen & RTL-Quellen einlesen
* `create_project -force` legt unter `./vivado_out` das Vivado-Projekt für den exakten Baustein `xc7z020clg400-1` (Zynq-7020 im CLG400-Gehäuse, Speed Grade -1) an.
* `add_files` fügt `counter_4bit.v` und `axi_counter_4bit.v` hinzu. `update_compile_order` analysiert die Modul-Hierarchie und erkennt, dass `axi_counter_4bit` das Untermodul `counter_4bit` instanziiert.

#### Schritt 2: Zynq Processing System (`ps7`) & MYIR-Preset
* `create_bd_cell ... processing_system7:5.5 ps7` fügt das Software-Abbild des fest verdrahteten ARM-Prozessorsystems (Hard-IP im Silizium) in das Block Design `system.bd` ein.
* `apply_bd_automation ... make_external "FIXED_IO, DDR"` führt die festen physikalischen Leitungen für den DDR3-Speicher (`DDR_*`) und die MIO-Pins (`FIXED_IO_*`) nach außen an die Top-Level-Ports des Block Designs.
* `source myir_ps7_preset.tcl` lädt die boardspezifische Konfiguration des MYD-C7Z020-V2 (DDR3-Timing, $33{,}333\text{ MHz}$ Referenztakt, Pin-Multiplexing für UART1 auf MIO 48..49, Gigabit-Ethernet, SD0, QSPI).
* Die anschließende `set_property`-Zeile stellt explizit sicher, dass:
  * **`PCW_USE_M_AXI_GP0 = 1`**: Der 32-Bit-AXI-Master-Port von der CPU in Richtung FPGA aktiviert wird.
  * **`PCW_FPGA0_PERIPHERAL_FREQMHZ = 100`**: Der PLL-Ausgang `FCLK_CLK0` auf exakt $100\text{ MHz}$ konfiguriert wird (dieser Wert landet später auch im FSBL-Code `ps7_init.c`).

#### Schritt 3: RTL Module Reference (`create_bd_cell -type module`)
* Über `-type module -reference axi_counter_4bit` bindet Vivado unseren reinen Verilog-Code direkt als Block namens `counter_ip` ins Block Design ein.
* Vivado parst die Portliste von `axi_counter_4bit.v`, gruppiert alle `S_AXI_*`-Signale automatisch zu einem grafischen Bus-Port `S_AXI` (Typ `AXI4-Lite Slave`) und verknüpft diesen intern mit `S_AXI_ACLK` und `S_AXI_ARESETN`.

#### Schritt 4: Die AXI-Automatisierung (`apply_bd_automation ... axi4`) – Unsichtbare Hilfsblöcke
Dieser eine Befehl ist der wichtigste Automatisierungsschritt im gesamten Block Design. Vivado fügt hierbei automatisch **zwei zentrale Infrastruktur-IPs** zwischen `ps7` und `counter_ip` ein und verdrahtet alle Takte und Resets:

1. **AXI Interconnect (`ps7_axi_periph`) mit integriertem AXI Protocol Converter (`auto_pc`):**
   * **Warum ist das zwingend nötig?** Der Hardware-Port `M_AXI_GP0` des Zynq-7000 spricht das volle **AXI3**-Protokoll (mit Transaction-IDs, Bursts und `WID`-Signalen). Unser `axi_counter_4bit` spricht dagegen das schlanke **AXI4-Lite**-Protokoll.
   * Vivado erkennt diese Protokoll-Diskrepanz automatisch und baut innerhalb des `AXI Interconnect` einen **AXI Protocol Converter (`axi_protocol_converter`)** ein, der alle AXI3-Zugriffe der CPU transparent in AXI4-Lite-Transaktionen für `counter_ip` übersetzt und die Datenbreite sowie Adressdecodierung handhabt.
   * *Hinweis:* Im generierten `system.v` heißen die Leitungen entsprechend `ps7_axi_periph_M00_AXI_*` (Master-Port 0 des Interconnects).

2. **Processor System Reset (`rst_ps7_100M` / `proc_sys_reset`):**
   * Das PS liefert nur ein unsynchronisiertes Reset-Signal (`FCLK_RESET0_N`).
   * Vivado schaltet den IP-Core `proc_sys_reset` dazwischen. Dieser synchronisiert den Reset sauber auf `FCLK_CLK0` ($100\text{ MHz}$) und gibt gestaffelte Active-Low-Resets aus: zuerst `interconnect_aresetn` für den AXI-Interconnect und danach `peripheral_aresetn` (`rst_ps7_100M_peripheral_aresetn`) für `counter_ip/S_AXI_ARESETN`. So ist der Bus garantiert betriebsbereit, bevor der Zähler aus dem Reset kommt.

3. **Taktverteilung (`ps7_FCLK_CLK0`):**
   * Vivado verbindet `ps7/FCLK_CLK0` gleichzeitig mit:
     * `ps7/M_AXI_GP0_ACLK` (wichtig: ohne diesen Rückführungstakt arbeitet die PS-AXI-Schnittstelle nicht!),
     * den Takteingängen des `AXI Interconnect`,
     * `rst_ps7_100M/slowest_sync_clk`,
     * `counter_ip/S_AXI_ACLK`.

#### Schritt 5: Adresszuweisung & Validierung
* `assign_bd_address` ordnet den Slave `counter_ip/S_AXI/reg0` dem Adressraum `/ps7/Data` (dem `M_AXI_GP0`-Fenster `0x40000000` bis `0x7FFFFFFF`) zu.
* `offset 0x43C00000` und `range 4K` legen das feste Fenster `0x43C00000` – `0x43C00FFF` fest. Die Größe von $4\text{ KiB}$ ($4096\text{ Bytes}$) entspricht genau einer Linux-Speicherseite (*Page Size*), was Zugriffe per `mmap()` oder `devmem` ohne Adress-Alignment-Probleme ermöglicht.
* `validate_bd_design` führt strenge Design-Rule-Checks durch (Taktdomänen-Konsistenz, keine überlappenden Adressbereiche, korrekte Bus-Breiten, keine unverbundenen Pflicht-Pins).

#### Schritt 6: Top-Level Wrapper (`make_wrapper`)
* Ein Vivado Block Design (`.bd`) ist ein XML-basiertes Schaltplanformat und kann nicht direkt als Top-Level-HDL synthetisiert werden.
* `make_wrapper -top` generiert daher die Verilog-Datei `system_wrapper.v` (sowie die interne Netzliste `system.v`), die das Block Design instanziiert und die externen Anschlüsse (`DDR_*`, `FIXED_IO_*`) als Top-Level-Ports bereitstellt.
* `set_property top system_wrapper` setzt diesen Wrapper als oberste Ebene für die Synthese.

#### Schritt 7: Synthese, Implementation & Bitstream (`launch_runs impl_1 -to_step write_bitstream`)
Dieser Befehl stößt die gesamte FPGA-Compiler-Pipeline auf 8 CPU-Threads an:
1. **Synthese (`synth_1`):** Übersetzt `counter_4bit.v`, `axi_counter_4bit.v`, den AXI-Protokollwandler und `proc_sys_reset` in eine Gatter-Netzliste aus Xilinx-7-Series-Primitiven (LUT6, FDRE-Flip-Flops, CARRY4-Addiererketten für den 32-Bit-Prescaler).
2. **Implementation (`impl_1`):**
   * *Opt Design:* Logikoptimierung und Entfernen ungenutzter Signale.
   * *Place Design:* Platzierung der Logikblöcke auf reale physikalische Slices im Zynq-7020-Chip.
   * *Route Design:* Verdrahtung der Signalpfade über die schaltbare FPGA-Matrix und Prüfung der **Timing Closure** (jeder Pfad zwischen zwei Flip-Flops muss inklusive Setup-Zeit unter $T = 10\text{ ns}$ bei $100\text{ MHz}$ liegen).
3. **Write Bitstream:** Erzeugt die binäre Konfigurationsdatei `system_wrapper.bit`, die beim Booten (oder über Linux `fpga_manager`) in den Konfigurations-SRAM des PL geladen wird.

#### Schritt 8: Export der Hardware-Plattform (`write_hw_platform ... system.xsa`)
Die exportierte `.xsa`-Datei (*Xilinx Support Archive*) ist das Übergabepaket von Vivado an PetaLinux/Vitis. Technisch handelt es sich um ein ZIP-Archiv mit drei entscheidenden Inhalten:
* **`system_wrapper.bit`:** Der fertige FPGA-Bitstream (durch `-include_bit` direkt eingebettet).
* **`ps7_init.c` / `ps7_init.h` / `ps7_init_gpl.c`:** Generierter C-Code für den First Stage Boot Loader (FSBL), der beim Einschalten des Zynq sämtliche SLCR-Register, DDR3-Timings, `FCLK_CLK0` ($100\text{ MHz}$), `M_AXI_GP0`, PS-PL-Level-Shifter und TrustZone-Freigaben programmiert.
* **`system.hwh` (*Hardware Handoff*):** Maschinenlesbare XML-Beschreibung des Block Designs (Basisadresse `0x43C00000`, Bereich `0x43C00FFF`, Taktrate `100000000` Hz, angeschlossene Peripherie), aus der PetaLinux bzw. `sdtgen` den System Device Tree (SDT) erzeugt.

---

## 5. Hardware-Voraussetzungen & Registertest unter Linux

Damit ein Zugriff aus dem Linux-Userspace auf `0x43C00000` erfolgreich durch den AXI-Interconnect bis zu `axi_counter_4bit` durchdringt und nicht am Bus einfriert, müssen folgende Hardware-Voraussetzungen im Zynq-PS erfüllt sein (diese übernimmt im regulären Betrieb automatisch der aus `system.xsa` erzeugte FSBL):

1. **FPGA konfiguriert:** Bitstream geladen (`DONE`-Pin aktiv, `/sys/class/fpga_manager/fpga0/state` = `operating`).
2. **Takt aktiv:** `FCLK_CLK0` liefert $100\text{ MHz}$ an `M_AXI_GP0_ACLK` und `counter_ip/S_AXI_ACLK`.
3. **PS-PL-Level-Shifter aktiv:** SLCR-Register `LVL_SHFTR_EN` (`0xF8000900`) = `0xF`.
4. **PL-Reset freigegeben:** SLCR-Register `FPGA_RST_CTRL` (`0xF8000240`) = `0x0` (nach einem sauberen Reset-Puls nach dem Konfigurieren).
5. **TrustZone-Freigabe aktiv:** Register `TZ_FPGA_M` (`0xE020001C`) = `0x3` (erlaubt Non-Secure-Zugriffe auf `M_AXI_GP0`/`GP1`).

### Direkter Hardware-Test auf der Linux-Konsole (`devmem`)

```bash
# 1. Zähler starten (Bit 0 im CTRL-Register 0x43C00000 auf 1 setzen)
devmem 0x43C00000 32 1

# 2. Aktuellen 4-Bit-Zählerstand (0x00000000 .. 0x0000000F) im Sekundentakt auslesen
devmem 0x43C00004 32
sleep 1 && devmem 0x43C00004 32
sleep 1 && devmem 0x43C00004 32

# 3. Steuerregister zurücklesen (muss 0x00000001 liefern)
devmem 0x43C00000 32

# 4. Zähler pausieren (Bit 0 auf 0 setzen; aktueller Stand bleibt erhalten)
devmem 0x43C00000 32 0
```