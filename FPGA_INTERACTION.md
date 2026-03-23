
---

# 🎨 Interacting with the FPGA (PL) on MYD-C7Z020

The Zynq-7000 SoC allows the Linux kernel and user applications to interact with hardware logic in the Programmable Logic (PL). This guide covers loading bitstreams and accessing PL registers.

## 1. Loading the Bitstream

There are three primary ways to program the FPGA logic on the MYIR board:

### Method A: During Boot (Standard)
When you run `petalinux-package`, the bitstream is bundled into `BOOT.BIN`. The FSBL programs the FPGA before Linux even starts.
```bash
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga project-spec/hw-description/system.bit --u-boot --force
```

### Method B: Live Update via JTAG
If Linux is already running and you want to swap the hardware logic without rebooting:
```bash
petalinux-boot --jtag --fpga --bitstream ./your_new_file.bit
```

### Method C: FPGA Manager (Linux Userspace)
PetaLinux includes the **FPGA Manager** class, allowing you to program the PL from the Linux shell:
```bash
# Copy your .bit.bin file to the board
fpgautil -b /lib/firmware/system.bit.bin
```

---

## 2. Accessing PL Registers (Memory Mapping)

If you created an IP core in Vivado (like a GPIO or PWM controller) and connected it via the AXI Interconnect, it will have a specific **Base Address** (e.g., `0x40000000`).

### Using `devmem` (The Quick Way)
The `devmem` utility allows you to read/write directly to physical addresses from the console. 

**Read a register:**
```bash
# Read 32-bits from address 0x40000000
sudo devmem 0x40000000 32
```

**Write a register:**
```bash
# Write 0x01 to turn on an LED tied to that address
sudo devmem 0x40000000 32 0x00000001
```

---

## 3. The UIO (Userspace I/O) Driver

For professional applications, you shouldn't use `devmem`. Instead, use the **UIO driver**, which creates a file in `/dev/uioX` that your C/C++ application can `mmap()`.

### Step 1: Update Device Tree
In `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`, ensure your PL IP has the UIO compatible string:
```tcl
&my_custom_ip_0 {
    compatible = "generic-uio";
};
```

### Step 2: Enable UIO in Kernel
Run `petalinux-config -c kernel` and enable:
* `Device Drivers` -> `Userspace I/O drivers` -> `Userspace I/O platform driver with generic IRQ handling`.

### Step 3: Access via C Code
```c
int fd = open("/dev/uio0", O_RDWR);
void *ptr = mmap(NULL, 0x1000, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
// Now you can treat 'ptr' like a local array to talk to the FPGA!
uint32_t val = *((uint32_t *)ptr);
```

---

## 4. Signal Interfacing (PS-PL)

The MYIR board routes several Zynq "MIO" and "EMIO" pins to the expansion headers.
* **MIO:** Fixed pins connected directly to the ARM processor (Hard IP).
* **EMIO:** Signals routed through the FPGA fabric to reach the expansion pins.
* **AXI GPIO:** High-speed registers controlled by the AXI bus.



---

## 5. Troubleshooting FPGA Interaction
* **No Response from Address:** If `devmem` causes a "Bus Error," the FPGA is likely not programmed or the AXI clock is not running. Check the **DONE** LED on the board.
* **Address Mismatch:** Always verify the address in **Vivado Address Editor** matches the address you are calling in Linux.
* **Interrupts:** If your FPGA logic needs to alert the CPU, you must map the PL-to-PS interrupt lines in the Device Tree.

---
