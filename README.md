# MYD-C7Z020
MYD-C7Z020 experience

---



# PetaLinux 2024.2 Build Guide for MYIR MYD-C7Z020

This guide documents the setup and build process for the MYIR XC7Z020 board. It includes specific fixes for **Ubuntu 24.x** environments and common resource-related build failures.

## My Environment

### 1. The board - MYD-C7Z010/20-V2 Dev. Board, Artikel MYD-C7Z020-V2-4E1D-766-I

<img width="709" height="1261" alt="image" src="https://github.com/user-attachments/assets/83d507e6-87a4-4031-90fb-e7e2763e9e9b" />

---------------
# Setup and first boot
---------------
<details>
   <summary>Click to expand Setup and first boot</summary>
   
Initial connectied via MYIR serial2USB cabel to Host-PC.

for finding connected serial port (used for console I/O) i use **usbipd** command. With usbipd show you list all USB divices:

``` cmd

C:\Windows\System32>usbipd list
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
1-8    04e8:4002  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
2-2    8087:0029  Intel(R) Wireless Bluetooth(R)                                Not shared
4-4    05e3:0749  USB-Massenspeichergerät                                       Shared
6-3    046d:c52b  Logitech USB Input Device, USB-Eingabegerät                   Not shared
8-1    04e8:4001  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
8-2    0bc2:331a  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
8-3    0bc2:331a  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
8-4    0bc2:2038  Per USB angeschlossenes SCSI (UAS)-Massenspeichergerät        Not shared
10-4   04e8:6860  A52 von Gerhard, SAMSUNG Mobile USB Modem, SAMSUNG Androi...  Not shared
12-3   067b:2303  Prolific USB-to-Serial Comm Port (COM10)                      Shared
14-1   2b7e:c569  NexiGo HelloCam N930W Camera, NexiGo HelloCam N930W IR Ca...  Not shared
14-3   2b7e:a569  NexiGo HelloCam N930W Audio, USB-Eingabegerät                 Not shared
15-1   04f3:0c3d  ELAN WBF Fingerprint Sensor                                   Not shared

Persisted:
GUID                                  DEVICE
bbba9b06-911c-4d21-ac12-916c1c9056b2  Serielles USB-Gerät (COM9), Serielles USB-Gerät (COM8), B...
f91bc313-54db-4470-8338-96b851aff57f  Dual RS232-HS, USB Serial Converter B


C:\Windows\System32>

```

### 2. First boot

after connecting power supply und serial cable to board.

#### board power connector:

<img width="709" height="1261" alt="image" src="https://github.com/user-attachments/assets/58f3b943-5e84-4b50-bcc4-4b0e5af29407" />

#### board serial connector (DB25):

<img width="709" height="1261" alt="image" src="https://github.com/user-attachments/assets/386fc48d-98f9-470b-a354-6a482d697723" />

#### board main switch:

<img width="709" height="1261" alt="image" src="https://github.com/user-attachments/assets/db795b5b-54bf-42f3-8195-8343531aedfb" />

#### confiuration of putty on PC:

in my case serial device is COM10 as you see in above **"usbipd list"**

i set up baud rate and protocol


<img width="450" height="307" alt="image" src="https://github.com/user-attachments/assets/692b7f0d-bd9b-4646-b844-6ea06a3eb667" />

enable logging for checking output


<img width="449" height="342" alt="image" src="https://github.com/user-attachments/assets/733448a5-054a-4c90-95d4-603b1b624f59" />


now i switched board via main switch on. Near FCM connector at left side red LED and on rigth side blue LED are on. On SOM board there are 3 LED's, green LED is flashing, blue and read LED's are static on.

<img width="712" height="1059" alt="image" src="https://github.com/user-attachments/assets/4f7d69d3-1420-4b05-a6ad-aaa8da38132d" />


on putty (console output) i see initial boot process:


``` txt
U-Boot 2020.01 (Sep 26 2022 - 01:59:45 +0000)

CPU:   Zynq 7z020
Silicon: v3.1
DRAM:  ECC disabled 1 GiB
myir_board_init
MMC:   mmc@e0100000: 0, mmc@e0101000: 1
Loading Environment from SPI Flash... SF: Detected n25q256a with page size 256 Bytes, erase size 4 KiB, total 32 MiB
*** Warning - bad CRC, using default environment

In:    serial@e0001000
Out:   serial@e0001000
Err:   serial@e0001000
Net:
ZYNQ GEM: e000b000, mdio bus e000b000, phyaddr -1, interface rgmii-id

Warning: ethernet@e000b000 using MAC address from DT
eth0: ethernet@e000b000
Hit any key to stop autoboot:  0
SF: Detected n25q256a with page size 256 Bytes, erase size 4 KiB, total 32 MiB
device 0 offset 0x520000, size 0xa00000
SF: 10485760 bytes @ 0x520000 Read: OK
## Loading kernel from FIT Image at 10000000 ...
   Using 'conf@system-top.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'kernel@1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x100000f8
     Data Size:    4452016 Bytes = 4.2 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x00200000
     Entry Point:  0x00200000
     Hash algo:    sha256
     Hash value:   d242df307fa2c82501fd079630f9eb225c75f9cea706cc8755ac65d339aa3553
   Verifying Hash Integrity ... sha256+ OK
## Loading fdt from FIT Image at 10000000 ...
   Using 'conf@system-top.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'fdt@system-top.dtb' fdt subimage
     Description:  Flattened Device Tree blob
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x1043f0b4
     Data Size:    20486 Bytes = 20 KiB
     Architecture: ARM
     Hash algo:    sha256
     Hash value:   2f1a1b836ddd16034a33e32c2100e6224fc22848bfdc5b2763b07748cba6c5fd
   Verifying Hash Integrity ... sha256+ OK
   Booting using the fdt blob at 0x1043f0b4
   Loading Kernel Image
   Loading Device Tree to 07ff7000, end 07fff005 ... OK

Starting kernel ...

Booting Linux on physical CPU 0x0
Linux version 5.4.0-xilinx-v2020.1 (oe-user@oe-host) (gcc version 9.2.0 (GCC)) #1 SMP PREEMPT Mon Sep 19 02:48:52 UTC 2022
CPU: ARMv7 Processor [413fc090] revision 0 (ARMv7), cr=18c5387d
CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
OF: fdt: Machine model: xlnx,zynq-7000
printk: bootconsole [earlycon0] enabled
Memory policy: Data cache writealloc
cma: Reserved 32 MiB at 0x3e000000
percpu: Embedded 15 pages/cpu s31948 r8192 d21300 u61440
Built 1 zonelists, mobility grouping on.  Total pages: 260416
Kernel command line: console=ttyPS0,115200 root=/dev/mmcblk1p1 rw earlyprintk rootfstype=ext4 rootwait devtmpfs.mount=1 myir_encoder.display_type=HDMI
Dentry cache hash table entries: 131072 (order: 7, 524288 bytes, linear)
Inode-cache hash table entries: 65536 (order: 6, 262144 bytes, linear)
mem auto-init: stack:off, heap alloc:off, heap free:off
Memory: 994496K/1048576K available (6144K kernel code, 225K rwdata, 1924K rodata, 1024K init, 132K bss, 21312K reserved, 32768K cma-reserved, 229376K highmem)
rcu: Preemptible hierarchical RCU implementation.
rcu:    RCU restricting CPUs from NR_CPUS=4 to nr_cpu_ids=2.
        Tasks RCU enabled.
rcu: RCU calculated value of scheduler-enlistment delay is 10 jiffies.
rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
NR_IRQS: 16, nr_irqs: 16, preallocated irqs: 16
efuse mapped to (ptrval)
slcr mapped to (ptrval)
L2C: platform modifies aux control register: 0x72360000 -> 0x72760000
L2C: DT/platform modifies aux control register: 0x72360000 -> 0x72760000
L2C-310 erratum 769419 enabled
L2C-310 enabling early BRESP for Cortex-A9
L2C-310 full line of zeros enabled for Cortex-A9
L2C-310 ID prefetch enabled, offset 1 lines
L2C-310 dynamic clock gating enabled, standby mode enabled
L2C-310 cache controller enabled, 8 ways, 512 kB
L2C-310: CACHE_ID 0x410000c8, AUX_CTRL 0x76760001
random: get_random_bytes called from start_kernel+0x260/0x440 with crng_init=0
zynq_clock_init: clkc starts at (ptrval)
Zynq clock init
sched_clock: 64 bits at 383MHz, resolution 2ns, wraps every 4398046511103ns
clocksource: arm_global_timer: mask: 0xffffffffffffffff max_cycles: 0x58688d738d, max_idle_ns: 440795214563 ns
Switching to timer-based delay loop, resolution 2ns
Console: colour dummy device 80x30
Calibrating delay loop (skipped), value calculated using timer frequency.. 766.66 BogoMIPS (lpj=3833333)
pid_max: default: 32768 minimum: 301
Mount-cache hash table entries: 2048 (order: 1, 8192 bytes, linear)
Mountpoint-cache hash table entries: 2048 (order: 1, 8192 bytes, linear)
CPU: Testing write buffer coherency: ok
CPU0: Spectre v2: using BPIALL workaround
CPU0: thread -1, cpu 0, socket 0, mpidr 80000000
Setting up static identity map for 0x100000 - 0x100060
rcu: Hierarchical SRCU implementation.
smp: Bringing up secondary CPUs ...
CPU1: thread -1, cpu 1, socket 0, mpidr 80000001
CPU1: Spectre v2: using BPIALL workaround
smp: Brought up 1 node, 2 CPUs
SMP: Total of 2 processors activated (1533.33 BogoMIPS).
CPU: All CPU(s) started in SVC mode.
devtmpfs: initialized
VFP support v0.3: implementor 41 architecture 3 part 30 variant 9 rev 4
clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
futex hash table entries: 512 (order: 3, 32768 bytes, linear)
pinctrl core: initialized pinctrl subsystem
NET: Registered protocol family 16
DMA: preallocated 256 KiB pool for atomic coherent allocations
cpuidle: using governor menu
hw-breakpoint: found 5 (+1 reserved) breakpoint and 1 watchpoint registers.
hw-breakpoint: maximum watchpoint size is 4 bytes.
zynq-ocm f800c000.ocmc: ZYNQ OCM pool: 256 KiB @ 0x(ptrval)
e0001000.serial: ttyPS0 at MMIO 0xe0001000 (irq = 26, base_baud = 6249999) is a xuartps
printk: console [ttyPS0] enabled
printk: console [ttyPS0] enabled
printk: bootconsole [earlycon0] disabled
printk: bootconsole [earlycon0] disabled
mc: Linux media interface: v0.10
videodev: Linux video capture interface: v2.00
vgaarb: loaded
SCSI subsystem initialized
usbcore: registered new interface driver usbfs
usbcore: registered new interface driver hub
usbcore: registered new device driver usb
pps_core: LinuxPPS API ver. 1 registered
pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
PTP clock support registered
EDAC MC: Ver: 3.0.0
FPGA manager framework
Advanced Linux Sound Architecture Driver Initialized.
clocksource: Switched to clocksource arm_global_timer
thermal_sys: Registered thermal governor 'step_wise'
NET: Registered protocol family 2
tcp_listen_portaddr_hash hash table entries: 512 (order: 0, 6144 bytes, linear)
TCP established hash table entries: 8192 (order: 3, 32768 bytes, linear)
TCP bind hash table entries: 8192 (order: 4, 65536 bytes, linear)
TCP: Hash tables configured (established 8192 bind 8192)
UDP hash table entries: 512 (order: 2, 16384 bytes, linear)
UDP-Lite hash table entries: 512 (order: 2, 16384 bytes, linear)
NET: Registered protocol family 1
RPC: Registered named UNIX socket transport module.
RPC: Registered udp transport module.
RPC: Registered tcp transport module.
RPC: Registered tcp NFSv4.1 backchannel transport module.
PCI: CLS 0 bytes, default 64
hw perfevents: no interrupt-affinity property for /pmu@f8891000, guessing.
hw perfevents: enabled with armv7_cortex_a9 PMU driver, 7 counters available
workingset: timestamp_bits=14 max_order=18 bucket_order=4
jffs2: version 2.2. (NAND) (SUMMARY)  © 2001-2006 Red Hat, Inc.
bounce: pool size: 64 pages
io scheduler mq-deadline registered
io scheduler kyber registered
zynq-pinctrl 700.pinctrl: zynq pinctrl initialized
dma-pl330 f8003000.dmac: Loaded driver for PL330 DMAC-241330
dma-pl330 f8003000.dmac:        DBUFF-128x8bytes Num_Chans-8 Num_Peri-4 Num_Events-16
i2c /dev entries driver
cdns-i2c e0004000.i2c: 400 kHz mmio e0004000 irq 23
uvcvideo: Unable to create debugfs directory
usbcore: registered new interface driver uvcvideo
USB Video Class driver (1.1.1)
brd: module loaded
loop: module loaded
spi-nor spi0.0: n25q256a (32768 Kbytes)
3 fixed-partitions partitions found on MTD device spi0.0
Creating 3 MTD partitions on "spi0.0":
0x000000000000-0x000000500000 : "boot"
0x000000500000-0x000000520000 : "bootenv"
0x000000520000-0x000000f20000 : "kernel"
libphy: Fixed MDIO Bus: probed
CAN device driver interface
libphy: MACB_mii_bus: probed
phy_id [4f51e91a]
[yt8531_led_init]val=620
yt8521_config_init, 8521 init call out.
YT8531S Ethernet e000b000.ethernet-ffffffff:00: attached PHY driver [YT8531S Ethernet] (mii_bus:phy_addr=e000b000.ethernet-ffffffff:00, irq=POLL)
macb e000b000.ethernet eth0: Cadence GEM rev 0x00020118 at 0xe000b000 irq 29 (00:0a:35:00:00:00)
e1000e: Intel(R) PRO/1000 Network Driver - 3.2.6-k
e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
ehci-pci: EHCI PCI platform driver
usbcore: registered new interface driver usb-storage
chipidea-usb2 e0002000.usb: e0002000.usb supply vbus not found, using dummy regulator
ULPI transceiver vendor/product ID 0x0424/0x0007
Found SMSC USB3320 ULPI transceiver.
ULPI integrity check: passed.
ci_hdrc ci_hdrc.0: EHCI Host Controller
ci_hdrc ci_hdrc.0: new USB bus registered, assigned bus number 1
ci_hdrc ci_hdrc.0: USB 2.0 started, EHCI 1.00
hub 1-0:1.0: USB hub found
hub 1-0:1.0: 1 port detected
rtc-ds1307 0-0068: SET TIME!
rtc-ds1307 0-0068: registered as rtc0
EDAC MC: ECC not enabled
cpufreq: cpufreq_online: CPU0: Running at unlisted freq: 383333 KHz
cpufreq: cpufreq_online: CPU0: Unlisted initial frequency changed to: 666667 KHz
Xilinx Zynq CpuIdle Driver started
sdhci: Secure Digital Host Controller Interface driver
sdhci: Copyright(c) Pierre Ossman
sdhci-pltfm: SDHCI platform and OF driver helper
mmc0: SDHCI controller on e0100000.mmc [e0100000.mmc] using ADMA
mmc1: SDHCI controller on e0101000.mmc [e0101000.mmc] using ADMA
ledtrig-cpu: registered to indicate activity on CPUs
clocksource: ttc_clocksource: mask: 0xffff max_cycles: 0xffff, max_idle_ns: 467424388 ns
timer #0 at (ptrval), irq=43
usbcore: registered new interface driver usbhid
usbhid: USB HID core driver
fpga_manager fpga0: Xilinx Zynq FPGA Manager registered
NET: Registered protocol family 10
Segment Routing with IPv6
sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
NET: Registered protocol family 17
can: controller area network core (rev 20170425 abi 9)
NET: Registered protocol family 29
can: raw protocol (rev 20170425)
can: broadcast manager protocol (rev 20170425 t)
can: netlink gateway (rev 20190810) max_hops=1
Registering SWP/SWPB emulation handler
of-fpga-region fpga-full: FPGA Region probed
input: gpio-keys as /devices/soc0/gpio-keys/input/input0
rtc-ds1307 0-0068: setting system clock to 2000-01-01T00:00:10 UTC (946684810)
of_cfs_init
of_cfs_init: OK
ALSA device list:
  No soundcards found.
Waiting for root device /dev/mmcblk1p1...
mmc1: new high speed MMC card at address 0001
mmcblk1: mmc1:0001 S40004 3.64 GiB
mmcblk1boot0: mmc1:0001 S40004 partition 1 4.00 MiB
mmcblk1boot1: mmc1:0001 S40004 partition 2 4.00 MiB
mmcblk1rpmb: mmc1:0001 S40004 partition 3 4.00 MiB, chardev (245:0)
 mmcblk1: p1
EXT4-fs (mmcblk1p1): mounted filesystem with ordered data mode. Opts: (null)
VFS: Mounted root (ext4 filesystem) on device 179:1.
devtmpfs: mounted
usb 1-1: new high-speed USB device number 2 using ci_hdrc
Freeing unused kernel memory: 1024K
Run /sbin/init as init process
INIT: version 2.88 booting
random: fast init done
hub 1-1:1.0: USB hub found
hub 1-1:1.0: 4 ports detected
Starting udev
udevd[76]: starting version 3.2.8
random: udevd: uninitialized urandom read (16 bytes read)
random: udevd: uninitialized urandom read (16 bytes read)
random: udevd: uninitialized urandom read (16 bytes read)
udevd[77]: starting eudev-3.2.8
EXT4-fs (mmcblk1p1): re-mounted. Opts: (null)
Wed Aug  3 03:14:53 UTC 2022
urandom_read: 2 callbacks suppressed
random: dd: uninitialized urandom read (512 bytes read)
INIT: Entering runlevel: 5 line 18: ./media/sd-mmcblk0p1/myd_c7z0
Configuring network interfaces... phy_id [4f51e91a]
[yt8531_led_init]val=180
yt8521_config_init, 8521 init call out.
udhcpc: started, v1.31.0
udhcpc: sending discover
udhcpc: sending discover
udhcpc: sending discover
udhcpc: no lease, forking to background
done.
Starting system message bus: random: dbus-daemon: uninitialized urandom read (12 bytes read)
random: dbus-daemon: uninitialized urandom read (12 bytes read)
dbus.
Starting random number generator daemon
Initializing available sources

Failed to init entropy source hwrng

Initializing AES buffer

Enabling JITTER rng support

Initializing entropy source jitter

.
random: crng init done
Starting haveged: haveged: listening socket at 3
haveged: haveged starting up


Starting OpenBSD Secure Shell server: sshd
done.
Starting Xserver
Starting rpcbind daemon...
done.
Starting bluetooth: bluetoothd.

X.Org X Server 1.20.5
X Protocol Version 11, Revision 0
Build Operating System: Linux 3.10.0-693.el7.x86_64 x86_64
Current Operating System: Linux myir 5.4.0-xilinx-v2020.1 #1 SMP PREEMPT Mon Sep 19 02:48:52 UTC 2022 armv7l
Kernel command line: console=ttyPS0,115200 root=/dev/mmcblk1p1 rw earlyprintk rootfstype=ext4 rootwait devtmpfs.mount=1 myir_encoder.display_type=HDMI
Build Date: 26 May 2020  04:13:44PM

Current version of pixman: 0.38.4
        Before reporting problems, check http://wiki.x.org
        to make sure that you have the latest version.
Markers: (--) probed, (**) from config file, (==) default setting,
        (++) from command line, (!!) notice, (II) informational,
        (WW) warning, (EE) error, (NI) not implemented, (??) unknown.
(==) Log file: "/var/log/Xorg.0.log", Time: Wed Aug  3 03:15:21 2022
starting DNS forwarder and DHCP server: dnsmasq... (==) Using system config directory "/usr/share/X11/xorg.conf.d"
done.
(EE)
Fatal server error:
(EE) no screens found(EE)
(EE)
Please consult the The X.Org Foundation support
         at http://wiki.x.org
 for help.
(EE) Please also check the log file at "/var/log/Xorg.0.log" for additional information.
(EE)
(EE) Server terminated with error (1). Closing log file.
Starting internet superserver: inetd.
Starting syslogd/klogd: done
 * Starting Avahi mDNS/DNS-SD Daemon: avahi-daemon                       [ ok ]
Starting Telephony daemon
Starting Linux NFC daemon
Starting tcf-agent: OK

PetaLinux 2020.1 myir ttyPS0

myir login:
myir login: root
Password:
root@myir:~# xinit: giving up
xinit: unable to connect to X server: Connection refused
xinit: server error

root@myir:~# uname -a
Linux myir 5.4.0-xilinx-v2020.1 #1 SMP PREEMPT Mon Sep 19 02:48:52 UTC 2022 armv7l armv7l armv7l GNU/Linux
root@myir:~#


```


**boot finish with myir login, default user is "root", defaul root password is "root" to.**

</details>

-----------------
# MYIR Board U-Boot Autoboot Deactivation Guide
-----------------
<details>
<summary>Click to expand: MYIR Board U-Boot Autoboot Deactivation Guide</summary>

-----------------
## **U-Boot Configuration Guide: Deactivating Autoboot** 
-----------------

### **1. Accessing the U-Boot Prompt**
To begin, connect your board to a host PC via the serial console (UART). Power on the board and immediately press any key (usually **Enter** or **Space**) when you see the following message:

U-Boot 2020.01 (Sep 26 2022 - 01:59:45 +0000)

CPU:   Zynq 7z020
Silicon: v3.1
DRAM:  ECC disabled 1 GiB
myir_board_init
MMC:   mmc@e0100000: 0, mmc@e0101000: 1
Loading Environment from SPI Flash... SF: Detected n25q256a with page size 256 Bytes, erase size 4 KiB, total 32 MiB
*** Warning - bad CRC, using default environment

In:    serial@e0001000
Out:   serial@e0001000
Err:   serial@e0001000
Net:
ZYNQ GEM: e000b000, mdio bus e000b000, phyaddr -1, interface rgmii-id

Warning: ethernet@e000b000 using MAC address from DT
eth0: ethernet@e000b000

> `Hit any key to stop autoboot: 3`

Once interrupted, you should see the command prompt:
`Zynq> `

---

### **2. Inspecting Current Settings**
Before making changes, it is important to see what the board is currently configured to do. Use the `printenv` command to view the boot variables.

* **To check the primary boot command:**
    ```bash
    printenv bootcmd
    ```
* **To see the entire environment list:**
    ```bash
    printenv

   Zynq> printenv
   arch=arm
   autoload=no
   baudrate=115200
   board=zynq
   board_name=zynq
   boot_img=BOOT.BIN
   boot_targets=qspi
   bootcmd=run $modeboot
   bootdelay=4
   bootenv=uEnv.txt
   bootenvsize=0x20000
   bootenvstart=0x500000
   clobstart=0x10000000
   console=console=ttyPS0,115200
   cp_kernel2ram=mmcinfo && fatload mmc ${sdbootdev} ${netstart} ${kernel_img}
   cpu=armv7
   default_bootcmd=run cp_kernel2ram && bootm ${netstart}
   display_type=HDMI
   dtb_img=system.dtb
   dtbnetstart=0x23fff000
   eraseenv=sf probe 0 && sf erase ${bootenvstart} ${bootenvsize}
   ethaddr=00:0a:35:00:00:00
   fault=echo ${img} image size is greater than allocated place - partition ${img} is NOT UPDATED
   fdtcontroladdr=3ffc31d0
   importbootenv=echo "Importing environment from SD ..."; env import -t ${loadbootenv_addr} $filesize
   install_boot=mmcinfo && fatwrite mmc ${sdbootdev} ${clobstart} ${boot_img} ${filesize}
   install_jffs2=sf probe 0 && sf erase ${jffs2start} ${jffs2size} && sf write ${clobstart} ${jffs2start} ${filesize}
   install_kernel=mmcinfo && fatwrite mmc ${sdbootdev} ${clobstart} ${kernel_img} ${filesize}
   jffs2_img=rootfs.jffs2
   kernel_img=image.ub
   load_boot=tftpboot ${clobstart} ${boot_img}
   load_dtb=tftpboot ${clobstart} ${dtb_img}
   load_jffs2=tftpboot ${clobstart} ${jffs2_img}
   load_kernel=tftpboot ${clobstart} ${kernel_img}
   loadaddr=0x10000000
   loadbootenv=load mmc $sdbootdev:$partid ${loadbootenv_addr} ${bootenv}
   loadbootenv_addr=0x00100000
   mmc_args=setenv bootargs ${console} root=/dev/mmcblk1p1 rw earlyprintk rootfstype=ext4 rootwait devtmpfs.mount=1 myir_encoder.display_type=${display_type}
   modeboot=qspiboot
   nc=setenv stdout nc;setenv stdin nc;
   netboot=tftpboot ${netstart} ${kernel_img} && bootm
   netstart=0x10000000
   psserial0=setenv stdout ttyPS0;setenv stdin ttyPS0
   qspiboot=run mmc_args; sf probe 0 && sf read 10000000 0x00520000 0x00a00000 && bootm ${netstart};
   script_offset_f=f10000
   sd_uEnvtxt_existence_test=test -e mmc $sdbootdev:$partid /uEnv.txt
   sd_update_dtb=echo Updating dtb from SD; mmcinfo && fatload mmc ${sdbootdev}:1 ${clobstart} ${dtb_img} && run install_dtb
   sd_update_jffs2=echo Updating jffs2 from SD; mmcinfo && fatload mmc ${sdbootdev}:1 ${clobstart} ${jffs2_img} && run install_jffs2
   sdboot=run uenvboot; run cp_kernel2ram && bootm ${netstart}
   sdbootdev=0
   serial=setenv stdout serial;setenv stdin serial
   serverip=192.168.30.2
   soc=zynq
   stderr=serial@e0001000
   stdin=serial@e0001000
   stdout=serial@e0001000
   test_crc=if imi ${clobstart}; then run test_img; else echo ${img} Bad CRC - ${img} is NOT UPDATED; fi
   test_img=setenv var "if test ${filesize} -gt ${psize}; then run fault; else run ${installcmd}; fi"; run var; setenv var
   uenvboot=if run sd_uEnvtxt_existence_test; then run loadbootenv; echo Loaded environment from ${bootenv}; run importbootenv; fi; if test -n $uenvcmd; then echo Running uenvcmd ...; run uenvcmd; fi
   uenvcmd= setenv bootargs console=ttyPS0,115200 earlycon root=/dev/mmcblk0p2 rw rootwait myir_encoder.display_type=${display_type}
   update_boot=setenv img boot; setenv psize ${bootsize}; setenv installcmd "install_boot"; run load_boot ${installcmd}; setenv img; setenv psize; setenv installcmd
   update_dtb=setenv img dtb; setenv psize ${dtbsize}; setenv installcmd "install_dtb"; run load_dtb test_img; setenv img; setenv psize; setenv installcmd
   update_jffs2=setenv img jffs2; setenv psize ${jffs2size}; setenv installcmd "install_jffs2"; run load_jffs2 test_img; setenv img; setenv psize; setenv installcmd
   update_kernel=setenv img kernel; setenv psize ${kernelsize}; setenv installcmd "install_kernel"; run load_kernel ${installcmd}; setenv img; setenv psize; setenv installcmd
   vendor=xilinx

   Environment size: 3594/131068 bytes
   Zynq>

    ```

> **Note:** On MYIR boards, `bootcmd` often contains a reference like `run $modeboot`. If so, you should also run `printenv modeboot` to see the actual script being executed.

---

### **3. Modifying Autoboot Behavior**
You can stop the automatic boot by either increasing the reaction time or neutralizing the command itself.

### **Option A: Increase the Boot Delay**
This gives you a longer window to manually interrupt the boot process without fully disabling it.
```bash
setenv bootdelay 10
```

### **Option B: Disable the Boot Command**
To prevent the board from ever attempting to load the OS automatically, replace the `bootcmd` with a simple message.
```bash
setenv bootcmd echo "Autoboot disabled. Please boot manually."
```

---

### **4. Saving Your Changes**
Changes made with `setenv` are only stored in RAM. To make these changes permanent (surviving a power cycle), you **must** save them to the onboard flash memory.

```bash
saveenv
```

---

### **5. Manual Boot**

to boot from SPI

```bash

Zynq> run qspiboot
SF: Detected n25q256a with page size 256 Bytes, erase size 4 KiB, total 32 MiB
device 0 offset 0x520000, size 0xa00000
SF: 10485760 bytes @ 0x520000 Read: OK
## Loading kernel from FIT Image at 10000000 ...
   Using 'conf@system-top.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'kernel@1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x100000f8
     Data Size:    4452016 Bytes = 4.2 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x00200000
     Entry Point:  0x00200000
     Hash algo:    sha256
     Hash value:   d242df307fa2c82501fd079630f9eb225c75f9cea706cc8755ac65d339aa3553
   Verifying Hash Integrity ... sha256+ OK
## Loading fdt from FIT Image at 10000000 ...
   Using 'conf@system-top.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'fdt@system-top.dtb' fdt subimage
     Description:  Flattened Device Tree blob
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x1043f0b4
     Data Size:    20486 Bytes = 20 KiB
     Architecture: ARM
     Hash algo:    sha256
     Hash value:   2f1a1b836ddd16034a33e32c2100e6224fc22848bfdc5b2763b07748cba6c5fd
   Verifying Hash Integrity ... sha256+ OK
   Booting using the fdt blob at 0x1043f0b4
   Loading Kernel Image
   Loading Device Tree to 07ff7000, end 07fff005 ... OK

Starting kernel ...

Booting Linux on physical CPU 0x0
Linux version 5.4.0-xilinx-v2020.1 (oe-user@oe-host) (gcc version 9.2.0 (GCC)) #1 SMP PREEMPT Mon Sep 19 02:48:52 UTC 2022
CPU: ARMv7 Processor [413fc090] revision 0 (ARMv7), cr=18c5387d
CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
OF: fdt: Machine model: xlnx,zynq-7000
printk: bootconsole [earlycon0] enabled
Memory policy: Data cache writealloc
cma: Reserved 32 MiB at 0x3e000000
..........
```

### **Summary Table of Commands**

| Task | Command |
| :--- | :--- |
| **View Variable** | `printenv <variable_name>` |
| **Set Delay** | `setenv bootdelay <seconds>` |
| **Set Command** | `setenv bootcmd <command_string>` |
| **Save Changes** | `saveenv` |
| **Manual Boot** | `boot` or `run bootcmd` |

---

</details>

---------------------
# System Preparation (Ubuntu 24.04/24.02)
--------------------
<details>
<summary>Click to expand: System Preparation (Ubuntu 24.04/24.02)</summary>

## 🛠 System Preparation (Ubuntu 24.04/24.02)

### 1. Essential Dependencies
Ubuntu 24.04 is missing several legacy libraries required by the Xilinx/AMD toolchain. Install the full suite here:
```bash
sudo apt update
sudo apt install -y iproute2 gawk make net-tools libncurses5-dev tftpd-hpa \
flex bison libssl-dev xterm texinfo devscripts python3-pip gcc-multilib \
build-essential zlib1g:i386 screen pax gzip tar decode-dimms libncurses5
```
*Note: `libncurses5` is critical for the `petalinux-config` menu to render.*

### 2. The "Dash" Fix
PetaLinux scripts require `bash`. Ubuntu defaults to `dash`, which will cause the build to fail silently or terminate the terminal.
```bash
sudo dpkg-reconfigure dash
# Select [NO] when asked to use dash as the default shell.
```

### 3. Fixing Locale Errors
If you see errors related to `LC_ALL` or `Language`, ensure your locales are generated:
```bash
sudo locale-gen en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

---

## 🏗 Build Workflow

### 1. Initialize Project
```bash
source ~/petalinux/2024.2/settings.sh
petalinux-create -t project --template zynq -n MYIR_Z7020
cd MYIR_Z7020
petalinux-config --get-hw-description=/path/to/system.xsa
```

### 2. Preventing Resource Crashes (OOM Killer)
If your terminal terminates abruptly during `petalinux-build`, your system is likely running out of RAM. 

**Fix A: Limit Parallel Tasks**
Run `petalinux-config` -> **Yocto Settings** -> **Parallel sourcing/building** and set:
* **Number of jobs:** 2 or 4
* **Number of threads:** 2 or 4

**Fix B: Increase Swap Space (Recommended 16GB+)**
```bash
sudo fallocate -l 32G /swapfile_petalinux
sudo chmod 600 /swapfile_petalinux
sudo mkswap /swapfile_petalinux
sudo swapon /swapfile_petalinux
```

add to /etc/fstab

```bash
mimo3000n@KLG-PC001:~$ more /etc/fstab
# UNCONFIGURED FSTAB FOR BASE SYSTEM
/swapfile_petalinux none swap sw 0 0
mimo3000n@KLG-PC001:~$
```


---

## 🔄 Recovery & Restart Process
If the build is interrupted or the terminal crashes, follow these steps to resume safely:

### 1. Kill "Zombie" Processes
Stop any hidden background BitBake or XSCT processes still consuming RAM:
```bash
pkill -u $USER -f bitbake
pkill -u $USER -f xsct
pkill -u $USER -f pseudo
```

### 2. Remove Stale Locks
PetaLinux will refuse to start if it thinks another instance is running. Clean the lock files:
```bash
rm -f build/bitbake.lock
rm -f build/bitbake.sock
```

### 3. Resume the Build
Navigate to the project root and source the environment before restarting:
```bash
source ~/petalinux/2024.2/settings.sh
petalinux-build
```

### 4. Component-Specific Reset (If needed)
If only one part (like the kernel) is failing, you can clean it specifically:
```bash
petalinux-build -c kernel -x cleanall
petalinux-build
```

---

## 💾 SD Card Deployment

### Partition Layout
* **Partition 1 (FAT32, ~500MB):** Label as `BOOT`
* **Partition 2 (EXT4, Remaining):** Label as `rootfs`

### create mountpoints
sudo mkdir /mnt/sd_boot /mnt/sd_rootfs

### mount both partition of SD Card

check if partitions are there with **lsblk**

```bash


### Copying Files
```bash
# To Partition 1 (BOOT)
cd /home/mimo3000n/petalinux/2024.2/MYD-C7Z020/images/linux

mimo3000n@KLG-PC001:~/petalinux/2024.2/MYD-C7Z020/images/linux$ ls -l
total 2339988
-rw-r--r-- 1 mimo3000n mimo3000n   5216172 Mar 27 20:28 BOOT.BIN
-rw-r--r-- 1 mimo3000n mimo3000n      3830 Mar 19 17:23 boot.scr
-rw-r--r-- 1 mimo3000n mimo3000n       327 Mar 27 20:28 bootgen.bif
-rw-r--r-- 1 mimo3000n mimo3000n      7602 Mar 27 17:09 config
-rw-r--r-- 1 mimo3000n mimo3000n   5157735 Mar 26 13:51 image.ub
drwxr-xr-x 2 mimo3000n mimo3000n      4096 Mar 19 17:23 pxelinux.cfg
-rw-r--r-- 1 mimo3000n mimo3000n 674348544 Mar 27 17:31 rootfs.cpio
-rw-r--r-- 1 mimo3000n mimo3000n 234519742 Mar 27 17:31 rootfs.cpio.gz
-rw-r--r-- 1 mimo3000n mimo3000n 234519806 Mar 27 17:31 rootfs.cpio.gz.u-boot
-rw-r--r-- 1 mimo3000n mimo3000n 959204352 Mar 27 17:31 rootfs.ext4
-rw-r--r-- 1 mimo3000n mimo3000n     70385 Mar 27 17:28 rootfs.manifest
-rw-r--r-- 1 mimo3000n mimo3000n      1935 Mar 27 17:30 rootfs.qemuboot.conf
-rw-r--r-- 1 mimo3000n mimo3000n   3757170 Mar 27 17:29 rootfs.spdx.tar.zst
-rw-r--r-- 1 mimo3000n mimo3000n 236082929 Mar 27 17:31 rootfs.tar.gz
-rw-r--r-- 1 mimo3000n mimo3000n   1741672 Mar 27 17:28 rootfs.testdata.json
-rw-r--r-- 1 mimo3000n mimo3000n   4045676 Mar 19 13:16 system.bit
-rw-r--r-- 1 mimo3000n mimo3000n     23978 Mar 26 13:47 system.dtb
-rw-r--r-- 1 mimo3000n mimo3000n   1066338 Mar 27 17:25 u-boot-dtb.bin
-rw-r--r-- 1 mimo3000n mimo3000n   1070860 Mar 27 17:25 u-boot-dtb.elf
-rwxr-xr-x 1 mimo3000n mimo3000n   1042360 Mar 27 17:25 u-boot.bin
-rwxr-xr-x 1 mimo3000n mimo3000n   8461216 Mar 27 17:25 u-boot.elf
-rw-r--r-- 1 mimo3000n mimo3000n   5131896 Mar 26 13:51 uImage
-rw-r--r-- 1 mimo3000n mimo3000n  15062008 Mar 26 13:51 vmlinux
-rw-r--r-- 1 mimo3000n mimo3000n   5131832 Mar 26 13:51 zImage
-rw-r--r-- 1 mimo3000n mimo3000n    413192 Mar 19 17:28 zynq_fsbl.elf
mimo3000n@KLG-PC001:~/petalinux/2024.2/MYD-C7Z020/images/linux$

cp images/linux/BOOT.BIN /media/$USER/BOOT/
cp images/linux/image.ub /media/$USER/BOOT/
cp images/linux/boot.scr /media/$USER/BOOT/

# To Partition 2 (rootfs)
sudo tar -xpf images/linux/rootfs.tar.gz -C /media/$USER/rootfs/
sync
```

---

## 🔌 Hardware Setup (MYD-C7Z020)

### Jumper Settings
| Mode | SW1 (MIO 5:2) | JP3 (JTAG Chain) |
| :--- | :--- | :--- |
| **SD Boot** | `1100` | Closed (Cascaded) |
| **JTAG Debug** | `0000` | Closed (Cascaded) |

---

## 🛠 Troubleshooting Summary
* **"Bitbake is not available":** Ensure you ran `source settings.sh` in your current terminal session.
* **"Failed to copy to tftp":** This is a permission warning; ignore it. Images are safe in `images/linux/`.
* **Terminal Closes Suddenly:** Check `dmesg | grep -i oom`. Increase swap space and lower the job count in `petalinux-config`.

---
</details>


# Click to expand: JTAG Setup and Configuration Giuide ( https://github.com/mimo3000n/MYD-C7Z020/blob/main/JTAG.md)
