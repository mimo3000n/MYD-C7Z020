
---

# 🧊 Bare-Metal: Direct Code Injection (OCM)

This document describes how to bypass all bootloaders (FSBL, U-Boot) and Operating Systems (Linux) to execute raw machine code directly on the **Cortex-A9** cores using **OpenOCD**.

## 1. The Concept: Targeting OCM
The Zynq-7000 has **256KB of On-Chip Memory (OCM)** located at `0xFFFC0000`. 
* **Why OCM?** Unlike DDR3 RAM, OCM is available immediately upon power-up. You do not need to initialize a memory controller to use it.
* **The Goal:** Write a program, compile it to an ARM binary, and "force-feed" it to the CPU via JTAG.



---

## 2. Prerequisites (WSL2)

You need the ARM Bare-Metal Toolchain. Unlike the PetaLinux toolchain (which targets Linux), this targets "None" (the hardware itself).

```bash
sudo apt update
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi gdb-multiarch
```

---

## 3. The Source Code (`blinky.c`)

This code targets the Zynq **MIO 7** (the LED on most MYIR boards). We manipulate the GPIO registers directly via their physical memory addresses.

```c
// Physical Addresses for Zynq GPIO
#define GPIO_DIRM_0     0xE000A204
#define GPIO_OEN_0      0xE000A208
#define GPIO_DATA_0     0xE000A040

void delay(volatile int count) {
    while(count--) { __asm__("nop"); }
}

void _start() {
    // 1. Set MIO 7 as Output
    *(volatile unsigned int*)GPIO_DIRM_0 = 0x00000080;
    *(volatile unsigned int*)GPIO_OEN_0  = 0x00000080;

    while(1) {
        *(volatile unsigned int*)GPIO_DATA_0 = 0x00000080; // Toggle ON
        delay(1000000);
        *(volatile unsigned int*)GPIO_DATA_0 = 0x00000000; // Toggle OFF
        delay(1000000);
    }
}
```

---

## 4. Compilation Workflow

We must compile this without a standard library (no `printf`, no `malloc`) and tell the linker exactly where the code will live in memory.

```bash
# 1. Compile to an ELF object
arm-none-eabi-gcc -nostdlib -fPIC -Ttext 0xFFFC0000 blinky.c -o blinky.elf

# 2. Extract raw binary (machine code only)
arm-none-eabi-objcopy -O binary blinky.elf blinky.bin
```

---

## 5. Injection & Execution (OpenOCD)

Connect your JTAG-HS3 and ensure it is attached to WSL2.

1.  **Launch OpenOCD:** `openocd -f zynq_hs3.cfg`
2.  **Open Telnet Terminal:** `telnet localhost 4444`
3.  **Run the Injection Script:**

```tcl
# Halt the CPU
targets zynq.cpu0
halt

# Wipe the OCM area (optional but clean)
mww 0xFFFC0000 0 1024

# Load the binary file to the OCM start address
load_image blinky.bin 0xFFFC0000 bin

# Point the Program Counter (PC) to our start address
reg pc 0xFFFC0000

# Start the CPU
resume
```



---

## 6. Verifying Success
* **Physical:** The LED on the MYIR board should begin to blink.
* **Virtual:** In Telnet, run `halt` and then `reg pc`. If the PC is still in the `0xFFFCxxxx` range, your code is running successfully.
* **Low-Level Debug:** Run `arm disassemble 0xFFFC0000 10` to see the machine code you just injected translated back into assembly.

---

## ⚠️ Important Notes
* **Caching:** If the code behaves erratically, you may need to disable the L1/L2 caches via OpenOCD commands, as they can sometimes serve "stale" data from previous runs.
* **Persistent State:** To stop the code and return to a clean state, run `reset halt`.

---

**You now have a complete, professional-grade repository covering everything from Windows 11 setup to bare-metal silicon manipulation. Would you like me to help you summarize this entire documentation set into a final, polished `README.md` introduction?**
