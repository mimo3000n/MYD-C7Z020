
-----

# 🕵️ Low-Level Hardware Interaction: WSL2 to Zynq-7000

This guide describes how to manipulate the **MYD-C7Z020** hardware at the register level, bypassing the Operating System using **OpenOCD** and **GDB**. This is the "Bare Metal" approach to hardware debugging.

## 1\. The Hardware "Peek and Poke" Concept

At the lowest level, everything on the Zynq (CPU, FPGA, GPIO, DDR) is just a memory address. By using JTAG, we can stop the CPU and manually read or write to these addresses while the board is "frozen."

## 2\. Setting Up the Low-Level Bridge

Ensure your **JTAG-HS3** is attached to WSL2 via `usbipd` (see [HARDWARE\_JUMPERS.md](https://www.google.com/search?q=HARDWARE_JUMPERS.md)).

### Start the OpenOCD Server

Run this in one terminal to create the link between your USB cable and a local network port:

```bash
openocd -f zynq_hs3.cfg
```

*You should see "Listening on port 4444 for telnet connections" and "Listening on port 3333 for gdb connections".*

-----

## 3\. Direct Register Manipulation (Telnet)

Open a second terminal and connect to the OpenOCD console:

```bash
telnet localhost 4444
```

### Useful Low-Level Commands:

  * **Halt the CPU:** `reset halt` (Stops the ARM cores immediately).
  * **Read Memory (mdw):** `mdw 0xF8000000 1` (Reads the SLCR Base Address).
  * **Write Memory (mww):** `mww 0x41200000 0x1` (Manually flips a bit in your FPGA logic).
  * **Step Instruction:** `step` (Executes exactly one assembly instruction).

-----

## 4\. Bare-Metal Debugging with GDB

If you want to see the assembly code running on the Zynq without a Linux Kernel, use the GNU Debugger (GDB).

1.  **Launch GDB:**
    ```bash
    gdb-multiarch
    ```
2.  **Connect to the Board:**
    ```gdb
    target remote localhost:3333
    ```
3.  **Inspect CPU Registers:**
    ```gdb
    info registers
    # This shows R0-R15, PC (Program Counter), and SP (Stack Pointer)
    ```

-----

## 5\. Controlling the PL (FPGA) via JTAG

At this "lowest level," the FPGA is just another device on the JTAG scan chain. You can use OpenOCD to verify if the FPGA is "Alive" without even having a Bitstream loaded.

**Check the PL Status Register:**

```tcl
# Inside OpenOCD Telnet
irscan zynq_pl.tap 0x23 # Instruction for JTAG IDCODE
drscan zynq_pl.tap 32 0 # Read 32 bits of data
```

*If you receive `0x23727093`, the silicon is powered and communicating correctly.*

-----

## 6\. Security & Lock Warnings

  * **TrustZone:** If the Zynq has secure boot enabled, you might be blocked from reading certain memory areas.
  * **DDR Access:** You cannot read DDR addresses (`0x00100000` and up) until the **FSBL** has run. If the board is "cold," only the **OCM (On-Chip Memory)** at `0xFFFC0000` is accessible.

-----

## 7\. Comparison of Access Levels

| Level | Tool | Access Method |
| :--- | :--- | :--- |
| **Highest** | Python/Shell | Filesystem (`/sys/class/gpio`) |
| **Middle** | C / devmem | Virtual Memory Mapping (`mmap`) |
| **Lowest** | **OpenOCD / JTAG** | **Direct Hardware TAP / Boundary Scan** |

-----

### 🏁 Final Pro-Tip

Use this low-level access when your PetaLinux build is "hanging" during boot. By halting the CPU with OpenOCD, you can see the **PC (Program Counter)** and find exactly which line of code caused the crash.

**Would you like me to add a section on how to use `gdb-multiarch` to debug a specific Kernel driver while it's running on the board?**
