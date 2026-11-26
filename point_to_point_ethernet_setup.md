# Point-to-Point Ethernet Setup: Windows / WSL2 ↔ MYIR MYD-C7Z020-V2

This document describes the isolated point-to-point Gigabit Ethernet connection between a Windows 11 host (running WSL2) and the **MYIR MYD-C7Z020-V2** (Zynq-7020) running **PetaLinux 2026.2**, without requiring a router, DHCP server, or internet access on the link.

---

## 1. Network Topology & IP Plan

| Node | Interface | IPv4 Address | Netmask / Prefix | Gateway / DNS | Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Windows 11 Host** | `Ethernet 5` (Realtek PCIe GbE) | `192.168.50.1` | `255.255.255.0` (`/24`) | *None* | Host NIC for direct board link |
| **MYD-C7Z020-V2** | `eth0` (Cadence GEM / YT8531 PHY) | `192.168.50.2` | `255.255.255.0` (`/24`) | *None* | Target FPGA/Linux board |

> **Note:** Leaving the Default Gateway and DNS empty on `Ethernet 5` ensures Windows continues routing internet traffic over Wi-Fi (`WLAN 2`) without interference.

---

## 2. Target Board Configuration (MYD-C7Z020-V2 / PetaLinux)

### 2.1 First Login & Root Access
In PetaLinux 2026.2, direct `root` login over UART is disabled by default. Log in with the default user `petalinux` (you will be prompted to set a new password on first boot) and elevate to `root`:

```bash
# Login as 'petalinux', set password, then:
sudo su -
```

### 2.2 Prevent Predictable Network Interface Renaming (`enx...` → `eth0`)
By default, `eudev` renames `eth0` to `enx000a35000000`, causing `/etc/network/interfaces` to fail on boot (`SIOCGIFINDEX: No such device`). Disable link renaming permanently:

```bash
ln -sf /dev/null /etc/udev/rules.d/80-net-setup-link.rules
```

### 2.3 Configure Static IPv4 in `/etc/network/interfaces`
Write a static configuration for `eth0` so the board comes up at `192.168.50.2` on every cold boot:

```bash
cat << 'EOF' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.50.2
    netmask 255.255.255.0
EOF
```

### 2.4 Apply Immediately Without Reboot
If the interface is currently named `enx000a35000000`, rename and bring it up immediately:

```bash
ip link set enx000a35000000 down
ip link set enx000a35000000 name eth0
ip addr flush dev eth0
ip addr add 192.168.50.2/24 dev eth0
ip link set eth0 up
```

**Verification:**
```bash
ip -4 addr show eth0
# Expected: inet 192.168.50.2/24 ... state UP
```

---

## 3. Host Configuration (Windows 11 PowerShell)

1. Identify the connected Gigabit Ethernet adapter in an **Administrator PowerShell**:
   ```powershell
   Get-NetAdapter | Where-Object Status -eq 'Up'
   ```
2. Disable DHCP and assign `192.168.50.1/24` without a gateway (replace `"Ethernet 5"` if your adapter name differs):
   ```powershell
   Set-NetIPInterface -InterfaceAlias "Ethernet 5" -Dhcp Disabled
   Remove-NetIPAddress -InterfaceAlias "Ethernet 5" -AddressFamily IPv4 -Confirm:$false -ErrorAction SilentlyContinue
   New-NetIPAddress -InterfaceAlias "Ethernet 5" -IPAddress 192.168.50.1 -PrefixLength 24
   ```

**Verification:**
```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet 5" -AddressFamily IPv4
```

---

## 4. Connectivity & Deployment Verification (WSL2)

From your WSL2 terminal, verify ICMP connectivity and deploy the FPGA bitstream (`./deploy_fpga.sh`) or launch the web dashboard:

```bash
# 1. Test round-trip latency (< 1 ms expected)
ping -c 2 192.168.50.2

# 2. Deploy PL bitstream via Linux FPGA Manager & start RISC dashboard
cd /mnt/d/FPGA/myd_c7z020_v2
./deploy_fpga.sh
python3 risc_dashboard.py
```

---

## 5. Hardware Boot Mode (`SW1` DIP Switch)

The 4-pin DIP switch **`SW1`** on the MYB-C7Z010/20 baseboard (located between the CPU module and the 12V power jack) selects the Zynq-7000 boot source. Set it to **SD-Card Boot** so `BOOT.BIN` (including the latest PL bitstream) loads automatically on cold boot:

| DIP Pin (`SW1`) | Zynq MIO | **SD-Card Boot** (`0110`) | QSPI Boot (`0001`) |
| :--- | :--- | :--- | :--- |
| **Pin 1** | `MIO[2]` | `OFF` (`0`) | `ON` (`1`) |
| **Pin 2** | `MIO[3]` | `ON` (`1`) | `OFF` (`0`) |
| **Pin 3** | `MIO[4]` | `ON` (`1`) | `OFF` (`0`) |
| **Pin 4** | `MIO[5]` | `OFF` (`0`) | `OFF` (`0`) |

### Manual U-Boot Fallback (if booted from QSPI)
If `SW1` is still set to QSPI and U-Boot stops at `Zynq>`, boot `image.ub` directly from `mmc 0:1`:

```text
setenv bootargs 'console=ttyPS0,115200 earlycon root=/dev/mmcblk0p2 rw rootwait'
fatload mmc 0:1 0x10000000 image.ub
bootm 0x10000000
```