# End-to-End Workflow: MYD-C7Z020-V2 (`axi_counter_4bit`) – Was passiert Wo und Wann?

Diese Dokumentation beschreibt den vollständigen technischen Ablauf des Projekts – vom Verilog-Quellcode über Vivado und PetaLinux 2026.2 bis zum Bootvorgang im Zynq-7020 und dem Live-Zugriff über das Web-Dashboard.

---

## 1. Master-Übersicht (Zeitstrahl, Ort & Artefakte)

| Phase | Wann? | Wo? | Was passiert? | Erzeugtes Artefakt / Zustand |
| :--- | :--- | :--- | :--- | :--- |
| **1. Hardware-Build** | Bei Änderungen an RTL oder PS7-Config | Windows / Vivado (`D:\FPGA\myd_c7z020_v2\02_vivado_batch`) | Block Design bauen, Synthese, Place & Route, Bitstream- & Hardware-Export | `system_wrapper.bit`, `system.xsa` |
| **2. Linux- & Boot-Build** | Nach neuem `system.xsa` | WSL / PetaLinux 2026.2 (`~/petalinux_myd_c7z020/myd_c7z020_v2`) | SDT generieren, FSBL, U-Boot, Kernel, Device Tree bauen & `BOOT.BIN` packen | `BOOT.BIN`, `boot.scr`, `image.ub`, `counter_web` |
| **3. Deployment** | Vor dem Starten des Boards | SD-Karte (`mmcblk0p1` & `mmcblk0p2`) + Board-Hardware (`JP2`) | Dateien auf FAT32/ext4 kopieren, Boot-Flag (`*`) setzen, Jumper `JP2` stecken | Bootfähige SD-Karte, TF-Boot-Modus aktiv |
| **4. Bootvorgang** | Direkt nach Power-On ($t = 0\text{ s}$) | Zynq-7020 PS & PL (MYD-C7Z020-V2) | BootROM $\rightarrow$ FSBL (`ps7_init` + PL-Load) $\rightarrow$ U-Boot $\rightarrow$ Linux | FPGA `operating`, Linux-Prompt, `192.168.50.2` |
| **5. Laufzeit** | Im laufenden Betrieb | Browser (PC) $\leftrightarrow$ Linux (PS) $\leftrightarrow$ Zähler (PL) | HTTP-Abfrage $\rightarrow$ `counter_web` $\rightarrow$ `/dev/mem` $\rightarrow$ `M_AXI_GP0` $\rightarrow$ Zähler | Live-Steuerung & 4-Bit-Stand auf Port `8080` |

---

## 2. Phase 1 – Hardware-Entwicklung & Bitstream-Build

* **Wo:** Windows-Dateisystem, erreichbar unter WSL als `/mnt/d/FPGA/myd_c7z020_v2/02_vivado_batch/`
* **Wann:** Immer wenn sich die Verilog-Logik (`counter_4bit.v`, `axi_counter_4bit.v`), die Adresszuweisung (`0x43C00000`) oder die Zynq-PS7-Konfiguration (`myir_ps7_preset.tcl`) ändert.

### Ablauf Schritt für Schritt
1. **Eingangsdateien bereitstellen:**
   * `counter_4bit.v` (4-Bit-Zählerkern mit Prescaler von $100\text{ MHz}$ auf $1\text{ Hz}$)
   * `axi_counter_4bit.v` (AXI4-Lite-Slave-Wrapper)
   * `myir_ps7_preset.tcl` (MYD-C7Z020-V2 Board-Konfiguration: DDR3, UART1 auf `MIO 48..49`, Ethernet, SD0)
   * `build.tcl` (Automatisierungsskript für den gesamten Vivado-Durchlauf)
2. **Vivado Batch-Durchlauf starten:**
   * Legt das Projekt `myd_counter_proj` für den Baustein `xc7z020clg400-1` an.
   * Instanziiert `ps7` (`M_AXI_GP0` aktiv, `FCLK_CLK0` = $100\text{ MHz}$) und `counter_ip`.
   * Fügt per `apply_bd_automation` automatisch den **AXI Interconnect** (inkl. AXI3-zu-AXI4-Lite-Protokollwandler) und **Processor System Reset (`proc_sys_reset`)** ein.
   * Vergibt das feste Adressfenster `0x43C00000` – `0x43C00FFF` ($4\text{ KiB}$).
   * Führt Synthese (`synth_1`), Implementation (`impl_1`, Place & Route, Timing-Prüfung auf $T = 10\text{ ns}$) und Bitstream-Erzeugung aus.
3. **Ergebnis (Übergabe an Phase 2):**
   * `vivado_out/myd_counter_proj.runs/impl_1/system_wrapper.bit` (FPGA-Konfiguration)
   * `system.xsa` (Hardware-Übergabepaket inkl. `system_wrapper.bit`, `ps7_init.c`/`ps7_init.h` für den FSBL und `system.hwh` für den Device Tree)

---

## 3. Phase 2 – Linux-System, SDT & Boot-Image

* **Wo:** WSL-Umgebung unter `~/petalinux_myd_c7z020/myd_c7z020_v2/` (sowie SDT-Ausgabe in `~/petalinux_myd_c7z020/sdt_out/`)
* **Wann:** Nach jedem neuen `system.xsa` aus Vivado (oder bei Änderungen an FSBL-Flags, U-Boot, Kernel oder Userspace-Tools).

### Ablauf Schritt für Schritt
1. **System Device Tree (SDT) aktualisieren:**
   * Aus `system.xsa` (speziell `system.hwh`) wird der System Device Tree unter `sdt_out/` erzeugt (`system-top.dts`, `zynq-7000.dtsi`).
   * Hier ist u. a. `stdout-path = "serial0:115200n8"` mit `serial0 = &uart1` (`0xE0001000`) verankert.
2. **Komponenten in PetaLinux 2026.2 bauen:**
   * **FSBL (`petalinux-build -c bootloader`):** Kompiliert `images/linux/zynq_fsbl.elf` mit dem exakten `ps7_init.c` aus deinem `system.xsa` (bei Bedarf mit `-DFSBL_DEBUG_INFO` via `fsbl-firmware_%.bbappend` für serielle Boot-Diagnose).
   * **U-Boot (`petalinux-build -c u-boot`):** Erzeugt `images/linux/u-boot.elf` und das Boot-Skript `images/linux/boot.scr`.
   * **Kernel & Device Tree (`petalinux-build`):** Verpackt Linux-Kernel und kompilierten Device Tree (`.dtb`) gemeinsam in `images/linux/image.ub`.
3. **Boot-Container `BOOT.BIN` packen:**
   * `petalinux-package boot --fsbl images/linux/zynq_fsbl.elf --fpga ".../system_wrapper.bit" --u-boot --force`
   * Kombiniert **genau in dieser Reihenfolge**:
     1. `zynq_fsbl.elf` (erster ausführbarer Code)
     2. `system_wrapper.bit` (FPGA-Bitstream, wird vom FSBL geladen)
     3. `u-boot.elf` (Second Stage Bootloader)
4. **Anwendung kompilieren (Userspace):**
   * Das C-Web-Dashboard wird statisch für ARM (`arm-linux-gnueabihf-gcc -static`) zur ausführbaren Datei `counter_web` kompiliert.

---

## 4. Phase 3 – Deployment auf SD-Karte & Hardware-Einstellung

* **Wo:** microSD-Karte (am PC oder über laufendes Board per `scp`) und direkt am MYD-C7Z020-V2 Board.
* **Wann:** Vor dem Neustart im produktiven SD-Boot-Modus.

### Aufbau der SD-Karte (`/dev/mmcblk0`)

```text
+-----------------------------------------------------------------------+
| /dev/mmcblk0p1 (FAT32, 512 MiB, Boot-Flag '*' AKTIV!)                 |
|   ├── BOOT.BIN     (FSBL + system_wrapper.bit + U-Boot)               |
|   ├── boot.scr     (U-Boot Startskript)                               |
|   └── image.ub     (Linux-Kernel + Device Tree)                       |
+-----------------------------------------------------------------------+
| /dev/mmcblk0p2 (ext4, Linux Root-Dateisystem)                         |
|   ├── /etc/network/interfaces   (statische IP 192.168.50.2/24)        |
|   ├── /root/counter_web         (C-Webserver auf Port 8080)           |
|   └── /lib/firmware/system.bit.bin (optional für Live-FPGA-Updates)   |
+-----------------------------------------------------------------------+
```

### Checkliste vor dem Einschalten
1. **Boot-Flag prüfen:** `/dev/mmcblk0p1` muss in der MBR-Partitionstabelle als aktiv (`*`) markiert sein (`fdisk -l /dev/mmcblk0`).
2. **Dateisystem synchronisieren:** Nach jedem Kopiervorgang auf die SD-Karte unbedingt `sync` ausführen, bevor die Stromversorgung getrennt wird.
3. **Boot-Modus-Jumper setzen:** Den blauen Jumper auf **`JP2` („Select TF BOOT“)** über **beide Pins stecken**.
   * *`JP2` gesteckt:* BootROM startet von der microSD-Karte (unser eigenes `BOOT.BIN` mit korrektem `ps7_init`).
   * *`JP2` offen:* BootROM startet den werkseitigen QSPI-Flash (alter MYIR-FSBL ohne `M_AXI_GP0`-Initialisierung!).

---

## 5. Phase 4 – Der Bootvorgang auf dem MYD-C7Z020-V2 (ab Power-On)

* **Wo:** Intern im Zynq-7020-Chip (erst OCM, dann DDR3 und FPGA-PL).
* **Wann:** Automatisch bei jedem Einschalten bzw. Reset des Boards.

```text
[Power-On]
    │
    ▼  (t ≈ 0 ms, Silizium-ROM)
┌──────────────────────────────────────────────────────────────────────┐
│ Stufe 0: Zynq BootROM                                                │
│  • Liest Boot-Mode-Pins -> JP2 gesteckt = SD0-Boot                   │
│  • Sucht auf FAT32-Partition (mmcblk0p1) nach BOOT.BIN               │
│  • Kopiert zynq_fsbl.elf in das interne 256 KiB On-Chip-RAM (OCM)    │
└──────────────────────────────────┬───────────────────────────────────┘
                                   ▼  (t ≈ 50 ms, OCM)
┌──────────────────────────────────────────────────────────────────────┐
│ Stufe 1: First Stage Boot Loader (zynq_fsbl.elf)                     │
│  1. Führt ps7_init() aus (aus deinem system.xsa):                    │
│     • Initialisiert PLLs, DDR3-RAM, UART1 (MIO 48/49, 115200 Baud)   │
│     • Aktiviert FCLK_CLK0 (100 MHz) und M_AXI_GP0                    │
│     • Gibt TrustZone (TZ_FPGA_M 0xE020001C = 0x3) frei               │
│  2. Lädt system_wrapper.bit aus BOOT.BIN per PCAP in den FPGA-PL     │
│     -> DONE-LED auf dem Modul leuchtet auf                           │
│  3. Aktiviert PS-PL-Level-Shifter (0xF8000900 = 0xF) und gibt        │
│     PL-Resets frei (0xF8000240 = 0x0) -> counter_ip ist sofort wach! │
│  4. Lädt u-boot.elf aus BOOT.BIN ins DDR3-RAM und springt hinein     │
└──────────────────────────────────┬───────────────────────────────────┘
                                   ▼  (t ≈ 0,5 s, DDR3-RAM)
┌──────────────────────────────────────────────────────────────────────┐
│ Stufe 2: U-Boot                                                      │
│  • Initialisiert SD-Karte und Konsole auf UART1                      │
│  • Führt boot.scr aus -> lädt image.ub (Kernel + DTB) ins DDR3       │
│  • Übergibt Kernel-Bootargs (inkl. root=PARTUUID=... auf mmcblk0p2)  │
└──────────────────────────────────┬───────────────────────────────────┘
                                   ▼  (t ≈ 2–5 s, DDR3-RAM)
┌──────────────────────────────────────────────────────────────────────┐
│ Stufe 3: Linux Kernel & Userspace                                    │
│  • Mountet RootFS (/dev/mmcblk0p2)                                   │
│  • Erkennt FPGA-Status: /sys/class/fpga_manager/fpga0/state=operating│
│  • Zieht Ethernet-Interface mit 192.168.50.2/24 hoch                 │
│  • Startet Login-Shell (UART1/SSH) und /root/counter_web (Port 8080) │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Phase 5 – Laufzeit-Workflow (Browser $\leftrightarrow$ Linux $\leftrightarrow$ FPGA)

* **Wo:** Entwicklungs-PC (Browser) $\leftrightarrow$ Ethernet (`192.168.50.1` / `192.168.50.2`) $\leftrightarrow$ ARM-CPU (PS) $\leftrightarrow$ `axi_counter_4bit` (PL).
* **Wann:** Während der Benutzer das Web-Dashboard bedient oder `devmem` auf der Konsole aufruft.

### Was bei einem Klick auf „Start“ im Browser passiert
1. **Browser (PC):** Sendet einen HTTP-Request an `http://192.168.50.2:8080` (z. B. Start-Befehl).
2. **`counter_web` (Linux Userspace auf ARM-PS):**
   * Hat beim Programmstart die physikalische Speicher-Seite `0x43C00000` ($4\text{ KiB}$) über `/dev/mem` und `mmap()` in seinen virtuellen Adressraum eingeblendet.
   * Schreibt den 32-Bit-Wert `0x00000001` an Basisadresse + Offset `0x00` (`0x43C00000`).
3. **Zynq PS-PL-Brücke:**
   * Die ARM-CPU gibt den Schreibzugriff über die MMU und die TrustZone-Prüfung (`0xE020001C`) an den Port `M_AXI_GP0` (AXI3) weiter.
   * Der **AXI Interconnect / Protocol Converter** im FPGA übersetzt den AXI3-Zugriff in eine AXI4-Lite-Transaktion.
4. **FPGA-Logik (`axi_counter_4bit` & `counter_4bit`):**
   * `axi_counter_4bit` erkennt `S_AXI_AWVALID && S_AXI_WVALID` auf Offset `0x00`, setzt `slv_reg0[0] <= 1'b1` und quittiert sofort mit `S_AXI_BVALID = 1` (`BRESP = OKAY`).
   * `counter_4bit` sieht `enable == 1` und zählt bei jedem Takt von `FCLK_CLK0` ($100\text{ MHz}$) den 32-Bit-Prescaler hoch. Nach jeweils $100\,000\,000$ Takten (exakt $1\text{ s}$) erhöht sich `count_out[3:0]` um $1$ ($0 \rightarrow 1 \rightarrow \dots \rightarrow 15 \rightarrow 0$).
5. **Auslesen des Zählerstands (Polling):**
   * Der Browser fragt periodisch den Zählerstand bei `counter_web` ab.
   * `counter_web` liest Offset `0x04` (`0x43C00004`) über `mmap()`.
   * `axi_counter_4bit` legt `{28'd0, w_count_out}` auf `S_AXI_RDATA`, `counter_web` liefert den Wert per HTTP zurück und das Dashboard aktualisiert die Anzeige live.

---

## 7. Schnell-Iteration: Bitstream-Update ohne Reboot (Optional)

* **Wann sinnvoll?** Wenn das Board bereits sauber über deinen eigenen SD-FSBL gebootet hat (`M_AXI_GP0` und `FCLK_CLK0` laufen) und du **nur den Verilog-Code** (`counter_4bit.v` / `axi_counter_4bit.v`) geändert hast – ohne Änderungen am Zynq-PS7.
* **Wo:** WSL (`bootgen` + `scp`) $\rightarrow$ MYD-C7Z020-V2 (`/sys/class/fpga_manager`).

### Ablauf
1. **In WSL:** Vivado-Bitstream `.bit` mit `bootgen` ins byte-geswappte Rohformat `.bit.bin` für den Linux FPGA-Manager umwandeln und per `scp` nach `/lib/firmware/system.bit.bin` kopieren:
   ```bash
   bootgen -image /tmp/bit2bin.bif -arch zynq -process_bitstream bin -w on
   scp .../system_wrapper.bit.bin root@192.168.50.2:/lib/firmware/system.bit.bin
   ```
2. **Auf dem Board:** Bitstream live laden und anschließend einen sauberen PL-Reset-Puls auslösen, damit `proc_sys_reset` und `axi_interconnect` definiert starten:
   ```bash
   echo 0 > /sys/class/fpga_manager/fpga0/flags
   echo system.bit.bin > /sys/class/fpga_manager/fpga0/firmware

   # SLCR entsperren -> PL-Reset-Puls -> SLCR sperren
   devmem 0xF8000008 32 0xDF0D
   devmem 0xF8000900 32 0x0
   devmem 0xF8000240 32 0xF
   sleep 1
   devmem 0xF8000900 32 0xF
   devmem 0xF8000240 32 0x0
   devmem 0xF8000004 32 0x767B
   ```
   > **Wichtig:** Dieser Schnellweg ersetzt *nicht* den eigenen FSBL-Boot bei Änderungen an der PS7-Konfiguration (`myir_ps7_preset.tcl`, `M_AXI_GP0`, Taktraten). Für den produktiven Start muss das neue `system_wrapper.bit` immer in `BOOT.BIN` auf `/dev/mmcblk0p1` verpackt werden.

---

## 8. Kurzreferenz: Typische Fehlerquellen im Workflow

| Symptom | Wo liegt die Ursache? | Lösung |
| :--- | :--- | :--- |
| Board bootet altes QSPI-System statt SD-Karte | Jumper `JP2` offen oder Boot-Flag `*` auf `mmcblk0p1` fehlt | `JP2` stecken, Boot-Flag per `fdisk` setzen (`a`, `1`, `w`) |
| Keine serielle Ausgabe beim SD-Boot | Falscher UART im SDT/FSBL oder FSBL ohne Debug-Output | `UART1` (`MIO 48..49`, `115200` Baud) prüfen, `-DFSBL_DEBUG_INFO` aktivieren |
| `devmem 0x43C00004` friert CPU komplett ein | QSPI-FSBL hat `M_AXI_GP0` nicht initialisiert, TrustZone gesperrt (`0xE020001C = 0x0`) oder PL-Reset fehlt | Eigenes `BOOT.BIN` von SD booten (`JP2`), `0xE020001C = 0x3`, `0xF8000900 = 0xF`, `0xF8000240 = 0x0` prüfen |
| `counter_web` im Browser nicht erreichbar | Netzwerk-Interface nach Reboot ohne IP | Statische IP `192.168.50.2/24` in `/etc/network/interfaces` speichern |