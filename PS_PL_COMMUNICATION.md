
---

# 🔄 Bidirectional PS-PL Communication

This document details the hardware pathways used to move data and control signals between the **Processing System (PS)** and the **Programmable Logic (PL)** on the MYD-C7Z020.



## 1. Simple Control: AXI GPIO
The easiest way to send "flags" or "triggers" back and forth. You instantiate an **AXI GPIO** IP core in Vivado and connect it to the `M_AXI_GP0` port of the PS.

* **PS → PL:** Software writes to a register, and the physical wires in the FPGA change state (e.g., enabling a motor or resetting a module).
* **PL → PS:** The FPGA logic sets a wire high, and the Linux software reads the change via `devmem` or a UIO driver.

---

## 2. High-Speed Data: Shared BRAM
For moving arrays or small buffers (up to 256KB) without the complexity of DMA.

* **Mechanism:** You use a **Dual-Port BRAM**. 
    * **Port A** is connected to the PS via an **AXI BRAM Controller**.
    * **Port B** is connected directly to your custom Verilog/VHDL logic.
* **Workflow:** The PS writes data into the memory range (e.g., `0x40000000`), and the PL logic instantly sees that data on its local bus.

---

## 3. High-Bandwidth: AXI DMA (Direct Memory Access)
For moving megabytes of data (like video frames or high-speed ADC samples) into the System DDR3 RAM.

* **PS → PL (MM2S):** Memory-Mapped to Stream. The PS tells the DMA to grab a chunk of DDR3 and "stream" it into an FPGA IP core.
* **PL → PS (S2MM):** Stream to Memory-Mapped. The FPGA logic streams data into the DMA, which writes it directly into the DDR3 RAM at high speed.

| Method | Best For | Typical Speed |
| :--- | :--- | :--- |
| **AXI GPIO** | Single bits, status flags, resets | Slow (Software Latency) |
| **BRAM** | Small lookup tables, shared buffers | Medium (Bus Latency) |
| **AXI DMA** | Video, Audio, SDR Samples | High (DDR3 Saturation) |

---

## 4. Signal Interfacing: EMIO
If you don't want to use the AXI bus, you can route the PS's internal peripherals (like UART, SPI, or GPIO) through the FPGA fabric.

* **Example:** You can route `UART1` from the PS through the PL logic, perform "packet sniffing" or encryption in the FPGA, and then send it out to physical pins.
* **GPIO EMIO:** Provides up to 64 extra wires that go directly from the ARM processor into the FPGA fabric as raw signals.

---

## 5. Asynchronous Events: Interrupts
Bidirectional communication often requires "handshaking."

* **PL → PS (IRQ_F2P):** The FPGA can trigger one of 16 interrupts to the PS. In Linux, this usually triggers a **UIO** or **Kernel Driver** callback.
* **PS → PL (Event):** The PS can send event signals to the PL to wake up hardware logic or synchronize timers.



---

## 🛠 Interaction Example: The "Handshake"
1.  **PS** writes a "Process Start" bit via **AXI GPIO**.
2.  **PL** reads the bit, processes data in **BRAM**, and sets a "Done" bit.
3.  **PL** triggers a **JTAG/Hardware Interrupt** to the PS.
4.  **PS** wakes up and reads the result from **BRAM**.

---
