
---

# 🔔 FPGA-to-PS Interrupt Handling (UIO)

This document explains how to configure the **Zynq Generic Interrupt Controller (GIC)** to allow the Programmable Logic (PL) to trigger software events in Linux.

## 1. Hardware Architecture: PL-to-PS IRQ
The Zynq-7000 PS has 16 dedicated interrupt lines coming from the FPGA fabric, labeled `IRQ_F2P[15:0]`. These are mapped to the GIC starting at **ID 61-68** and **84-91**.



### Vivado Configuration:
1.  Open the **Zynq PS Block**.
2.  Navigate to **Interrupts** -> **Fabric Interrupts** -> **PL-PS Interrupt Ports**.
3.  Enable `IRQ_F2P[15:0]`.
4.  Connect your IP core's `interrupt` output to this port.

---

## 2. Device Tree Configuration
PetaLinux needs to know that a specific hardware address is associated with an interrupt line and that it should be handled by the **Userspace I/O (UIO)** driver.

Edit `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`:

```tcl
/bits/ 32;
/ {
    chosen {
        bootargs = "console=ttyPS1,115200 root=/dev/mmcblk0p2 rw earlyprintk uio_pdrv_genirq.of_id=generic-uio";
    };
};

&my_custom_ip_0 {
    compatible = "generic-uio";
    interrupt-parent = <&intc>;
    interrupts = <0 29 4>; // 0 = SPI, 29 = IRQ_F2P[0] (GIC 61), 4 = Active High Level
};
```
> **Note on Interrupt Calculation:** For `IRQ_F2P[0]`, the DTB index is `29`. The formula is typically `GIC_ID - 32`.

---

## 3. Linux Userspace Driver (`interrupt_test.c`)

The UIO driver makes interrupt handling simple. When an interrupt occurs, a `read()` on the UIO device file unblocks.

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdint.h>

int main() {
    int fd;
    uint32_t info;
    uint32_t irq_count = 0;

    // 1. Open the UIO device (check 'ls /dev/uio*' to find the right index)
    fd = open("/dev/uio0", O_RDWR);
    if (fd < 0) {
        perror("Failed to open UIO device");
        return -1;
    }

    printf("Waiting for FPGA Interrupts...\n");

    while(1) {
        // 2. Re-enable the interrupt (UIO disables it after each trigger)
        info = 1;
        write(fd, &info, sizeof(info));

        // 3. Block here until the FPGA triggers the IRQ
        read(fd, &info, sizeof(info));

        irq_count++;
        printf("Interrupt #%d received from FPGA!\n", irq_count);
        
        // Handle your bidirectional data processing here...
    }

    close(fd);
    return 0;
}
```

---

## 4. Verification & Debugging

### Check Registered Interrupts
Once Linux is booted, you can see if your interrupt is registered in the kernel:
```bash
cat /proc/interrupts
```
*Look for a line labeled `uio_pdrv_genirq`. If the counter (first column) increases when your FPGA logic runs, the hardware link is working.*

### Check UIO Status
```bash
ls -l /sys/class/uio/uio0/
cat /sys/class/uio/uio0/name
```

---

## 5. Troubleshooting
* **Counter stays at 0:** Ensure the FPGA clock is running and that the `DONE` LED is on.
* **`read()` returns immediately:** Ensure you are writing `1` to the file descriptor to "re-arm" the interrupt; otherwise, it may trigger continuously if the FPGA signal is "Level" sensitive and stays high.
* **Permission Denied:** Run your test app with `sudo` or change permissions: `sudo chmod 666 /dev/uio0`.

---
