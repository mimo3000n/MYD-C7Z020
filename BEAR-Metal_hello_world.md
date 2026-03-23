
---

# 💉 Bare-Metal Code Injection via OpenOCD

This guide describes how to compile a raw binary in WSL2 and execute it on the MYD-C7Z020 Processing System (PS) without an Operating System.

## 1. The "Blinky" C Code (`main.c`)
We will target a memory-mapped GPIO to blink an LED. On the Zynq-7000, the MIO pins are controlled by the SLCR and GPIO registers.

```c
#define GPIO_DATA_RO    0xE000A068
#define GPIO_DIRM_0     0xE000A204
#define GPIO_OEN_0      0xE000A208

void delay(int count) {
    for (int i = 0; i < count; i++) {
        __asm__("nop");
    }
}

int main() {
    // Configure MIO 7 (Common LED on MYIR) as output
    *(unsigned int*)GPIO_DIRM_0 = 0x00000080;
    *(unsigned int*)GPIO_OEN_0  = 0x00000080;

    while(1) {
        *(unsigned int*)GPIO_DATA_RO = 0x00000080; // LED ON
        delay(1000000);
        *(unsigned int*)GPIO_DATA_RO = 0x00000000; // LED OFF
        delay(1000000);
    }
    return 0;
}
```

---

## 2. Cross-Compilation (WSL2)
You need the ARM GNU Toolchain. Since you installed PetaLinux, you already have it, but you can also install it standalone:

```bash
sudo apt install gcc-arm-none-eabi
```

**Compile to a Raw Binary:**
We use specific flags to ensure the code is "Position Independent" and has no standard library dependencies.
```bash
arm-none-eabi-gcc -nostdlib -fPIC -Ttext 0xFFFC0000 main.c -o test.elf
arm-none-eabi-objcopy -O binary test.elf test.bin
```
*Note: `0xFFFC0000` is the start of the Zynq On-Chip Memory (OCM).*

---

## 3. Injecting the Code (OpenOCD)



1.  **Start OpenOCD** in one terminal: `openocd -f zynq_hs3.cfg`
2.  **Connect via Telnet** in another: `telnet localhost 4444`
3.  **Execute the Injection Sequence:**

```tcl
# Halt the processor
targets zynq.cpu0
halt

# Load the binary into OCM
load_image test.bin 0xFFFC0000 bin

# Set the Program Counter (PC) to our code start
reg pc 0xFFFC0000

# Resume execution
resume
```

---

## 4. Debugging the Injected Code
If the LED doesn't blink, you can use OpenOCD to see exactly what the CPU is doing:

* **Check the PC:** `reg pc` (Is it still in the `0xFFFCxxxx` range?)
* **Dissassemble:** `arm disassemble 0xFFFC0000 10` (See the actual assembly instructions in RAM).
* **Step through:** `step` (Execute one instruction at a time and watch the registers change).

---

## 5. Summary of the "Lowest Level" Workflow

| Step | Tool | Action |
| :--- | :--- | :--- |
| **Write** | VS Code / Vim | Logic in C/Assembly |
| **Compile** | `arm-none-eabi-gcc` | Convert to ARM Machine Code |
| **Bridge** | `usbipd` | Connect JTAG-HS3 to WSL2 |
| **Inject** | `openocd` (load_image) | Write bytes directly to Zynq Silicon |
| **Execute** | `openocd` (resume) | Tell ARM Core to start "thinking" |

---
