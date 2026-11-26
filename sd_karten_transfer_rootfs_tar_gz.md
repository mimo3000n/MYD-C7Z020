# Workflow: `rootfs.tar.gz` auf die SD-Karte übertragen (Windows 11 + WSL2)

**Zielsystem:** `myd_c7z020_v2` (Zynq-7000, PetaLinux 2026.2)  
**Zielpartition:** SD-Karte, Partition 2 (`ext4`, Label `rootfs`, ca. 28 GB)  
**Quellarchiv (WSL2):** `~/petalinux_myd_c7z020/myd_c7z020_v2/images/linux/rootfs.tar.gz`  
**Host-System:** Windows 11 + WSL2 (Ubuntu 24.04)

---

## Warum `usbipd-win` statt `wsl --mount`?

Unter Windows 11 unterstützt `wsl --mount` keine USB-Wechseldatenträger wie SD-Kartenleser und bricht mit dem Fehler `Wsl/Service/AttachDisk/MountDisk/0x8007000f` ab. Durch das Durchreichen des USB-Kartenlesers per **`usbipd-win`** wird die SD-Karte direkt als reguläres Linux-Blockgerät (`/dev/sdX`) in WSL2 sichtbar.

---

## 1. Voraussetzungen & Vorbereitung

1. **Board sauber herunterfahren** (vor dem Entnehmen der SD-Karte):
   ```bash
   poweroff
   ```
   *(Optional vorher eigene Dateien vom Board sichern: `scp -r root@<board-ip>:/home/root ~/board_backup/`.)*

2. **SD-Karte in den USB-Kartenleser am Windows-PC einstecken.**

3. **Prüfen, ob `usbipd` installiert ist** (PowerShell als Administrator):
   ```powershell
   usbipd --version
   ```
   Falls nicht vorhanden, einmalig installieren und danach eine neue Admin-PowerShell öffnen:
   ```powershell
   winget install --exact dorssel.usbipd-win
   ```

4. **Optional: Inhalt und Aktualität von `rootfs.tar.gz` in WSL2 prüfen:**
   ```bash
   ls -lh ~/petalinux_myd_c7z020/myd_c7z020_v2/images/linux/rootfs.tar.gz
   ```

---

## 2. USB-Kartenleser an WSL2 durchreichen (PowerShell als Administrator)

1. **Angeschlossene USB-Geräte auflisten und `BUSID` ablesen:**
   ```powershell
   usbipd list
   ```
   Suche deinen SD-Kartenleser (z. B. `05e3:0749 USB-Massenspeichergerät`, `BUSID 3-4`).

2. **Gerät freigeben (nur erforderlich, wenn Status `Not shared` ist):**
   ```powershell
   usbipd bind --busid <BUSID>
   ```

3. **Kartenleser an WSL2 anhängen:**
   ```powershell
   usbipd attach --wsl --busid <BUSID>
   ```
   **Verifizierung:** `usbipd list` zeigt beim Kartenleser jetzt den Status **`Attached`**.

---

## 3. `ext4`-Partition in WSL2 ermitteln und mounten

1. **Blockgeräte und Dateisysteme auflisten:**
   ```bash
   lsblk -f
   ```
   Identifiziere die SD-Karte anhand ihrer zwei Partitionen (z. B. `/dev/sde`):
   * Partition 1: `vfat` / `FAT32`, Label `BOOT` *(Boot-Partition – bleibt unverändert)*
   * Partition 2: `ext4`, Label `rootfs` *(Zielpartition, z. B. `/dev/sde2`)*

2. **Mountpoint anlegen, Partition 2 einhängen und prüfen:**
   *(Ersetze `sdX2` durch die ermittelte Partition, z. B. `sde2`.)*
   ```bash
   sudo mkdir -p /mnt/sd_rootfs
   sudo mount /dev/sdX2 /mnt/sd_rootfs
   df -hT /mnt/sd_rootfs
   ```
   **Verifizierung:** `df -hT /mnt/sd_rootfs` zeigt eindeutig `/dev/sdX2` mit Typ `ext4` und ca. 28 GB Gesamtgröße an.

---

## 4. Altes RootFS ersetzen und `rootfs.tar.gz` entpacken (WSL2)

> **Wichtiger Sicherheitshinweis:** Führe `sudo rm -rf /mnt/sd_rootfs/*` erst aus, nachdem `df -hT /mnt/sd_rootfs` zweifelsfrei die `ext4`-Partition der SD-Karte bestätigt hat. Das Entpacken muss zwingend mit `sudo` erfolgen, damit Root-Eigentümer, Dateirechte und Geräteknoten korrekt erhalten bleiben.

1. **Altes RootFS leeren:**
   ```bash
   sudo rm -rf /mnt/sd_rootfs/*
   ```

2. **Neues `rootfs.tar.gz` entpacken und Schreibpuffer flushen:**
   ```bash
   sudo tar -xzf ~/petalinux_myd_c7z020/myd_c7z020_v2/images/linux/rootfs.tar.gz -C /mnt/sd_rootfs
   sync
   ```

3. **Stichprobe auf der SD-Karte prüfen:**
   ```bash
   ls -l /mnt/sd_rootfs/usr/bin/python3 /mnt/sd_rootfs/usr/bin/dnf
   ```
   **Verifizierung:** `tar` endet ohne Fehlermeldung und die erwarteten Binaries/Symlinks sind auf `/mnt/sd_rootfs` vorhanden.

---

## 5. Partition sauber aushängen und Kartenleser trennen

1. **Partition in WSL2 aushängen:**
   ```bash
   sudo umount /mnt/sd_rootfs
   ```
   **Verifizierung:** Der Befehl beendet sich ohne Fehlermeldung und `df -hT` listet `/mnt/sd_rootfs` nicht mehr auf.

2. **Kartenleser in PowerShell (als Administrator) von WSL2 trennen:**
   ```powershell
   usbipd detach --busid <BUSID>
   ```
   **Verifizierung:** `usbipd list` zeigt wieder den Status **`Shared`** statt `Attached`.

3. **SD-Karte sicher aus dem Kartenleser entfernen, ins Board einstecken und booten.**

---

## Hinweise & Troubleshooting

* **`umount: /mnt/sd_rootfs: target is busy`:** Stelle sicher, dass keine Shell in `/mnt/sd_rootfs` steht (`cd ~`), und prüfe bei Bedarf mit `sudo lsof +f -- /mnt/sd_rootfs`, welcher Prozess noch zugreift.
* **Neuer Login nach RootFS-Tausch:** Durch das Ersetzen des RootFS werden Passwörter und SSH-Host-Keys zurückgesetzt. Der erste Login erfolgt an der UART-Konsole als `petalinux` (siehe `petalinux_2026_2_root_setup.md` für das erneute Setzen des `root`-Passworts).
* **SSH-Warnung `REMOTE HOST IDENTIFICATION HAS CHANGED`:** Nach dem Booten des neuen RootFS einmalig auf dem Host-PC den alten Key entfernen:
  ```bash
  ssh-keygen -R <board-ip>
  ```