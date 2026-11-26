# PetaLinux 2026.2 – Update-Workflow: Python 3 & `dnf` im RootFS nachrüsten

**System:** `myd_c7z020_v2` (Zynq-7000, Architektur `cortexa9t2hf_neon`)  
**OS:** PetaLinux 2026.2 (persistentes `ext4`-RootFS auf SD-Karte, Partition 2)  
**Build-Host:** Windows 11 + WSL2 (Ubuntu 24.04)  
**Projektpfad:** `~/petalinux_myd_c7z020/myd_c7z020_v2`

---

## Überblick

Da ein minimales PetaLinux-RootFS standardmäßig keinen Paketmanager (`dnf`/`rpm`) enthält, wird Python 3 durch einen **inkrementellen RootFS-Rebuild** im PetaLinux-Projekt nachgerüstet. Kernel, Bootloader und Device Tree bleiben dabei unverändert. Gleichzeitig werden `python3-modules` (Standardbibliothek), `python3-pip`, `dnf` und das Image-Feature `package-management` aktiviert, sodass künftige Pakete auch direkt über einen lokalen RPM-Feed nachinstalliert werden können.

---

## 1. Vorbereitung & Backup

1. **Eigene Daten vom laufenden Board sichern** (vom Build-Host aus):
   ```bash
   mkdir -p ~/board_backup
   scp -r root@<board-ip>:/home/root ~/board_backup/
   ```
2. **Board sauber herunterfahren** (per SSH oder UART):
   ```bash
   poweroff
   ```

---

## 2. Build-Umgebung in WSL2 vorbereiten

1. **Conda deaktivieren und PetaLinux-2026.2-Umgebung laden:**  
   *(Wichtig: Aktive Conda-Umgebungen wie `(base)` überschreiben `PATH` und Python und führen bei BitBake zu Build-Fehlern.)*
   ```bash
   conda deactivate
   source ~/petalinux/2026.2/settings.sh
   ```
   > **Hinweis zu Ubuntu 24.04:** Eine eventuelle `[ERROR]`-Meldung zu `gcc-multilib` beim Laden von `settings.sh` ist auf Ubuntu 24.04 ein bekanntes False Positive, sofern das Paket bereits installiert ist. Prüfen lässt sich das mit:
   > ```bash
   > dpkg -s gcc-multilib >/dev/null 2>&1 && echo "gcc-multilib OK"
   > which petalinux-config
   > ```

2. **Ins PetaLinux-Projekt wechseln und Version prüfen:**
   ```bash
   cd ~/petalinux_myd_c7z020/myd_c7z020_v2
   cat .petalinux/metadata
   ```
   *Erwartet:* `PETALINUX_VER=2026.2`

---

## 3. Python 3 & Paketverwaltung konfigurieren

1. **Pakete ohne Duplikate in `user-rootfsconfig` eintragen:**
   ```bash
   for pkg in CONFIG_python3 CONFIG_python3-modules CONFIG_python3-pip CONFIG_dnf; do
     grep -qxF "$pkg" project-spec/meta-user/conf/user-rootfsconfig || echo "$pkg" >> project-spec/meta-user/conf/user-rootfsconfig
   done
   tail -n 10 project-spec/meta-user/conf/user-rootfsconfig
   ```

2. **Pakete und `package-management` im RootFS-Menü aktivieren:**
   ```bash
   petalinux-config -c rootfs
   ```
   * Unter **user packages** (bzw. per Suche mit `/`):
     * `[*] python3`
     * `[*] python3-modules` *(wichtig für Standardmodule wie `mmap`, `json`, `http.server`)*
     * `[*] python3-pip`
     * `[*] dnf`
   * Unter **Image Features**:
     * `[*] package-management` *(erhält die RPM-Datenbank für `dnf`; die Unterpunkte `package-feed-uris` und `package-feed-archs` leer lassen)*
   * Änderungen mit **Save** speichern und Menü mit **Exit** verlassen.

3. **Konfiguration verifizieren:**
   ```bash
   grep -iE "(python3|python3-modules|python3-pip|dnf|package-management)=y" project-spec/configs/rootfs_config
   ```
   Alle fünf Einträge müssen mit `=y` aufgeführt sein.

---

## 4. RootFS inkrementell neu bauen

1. **RootFS-Build starten:**
   ```bash
   petalinux-build -c rootfs
   ```
   > **Hinweis zu `setscene`-Fehlern:** Rote `Fetcher failure`-Meldungen bei `*_create_spdx_setscene` (z. B. `npth` oder `libsolv`) betreffen nur fehlende Cache-Einträge für Lizenz-Metadaten. BitBake führt die regulären Tasks automatisch nach (`will run full task instead`). Entscheidend ist, dass am Ende `Attempted ... tasks ... and all succeeded` erscheint und `images/linux/rootfs.tar.gz` aktuell erzeugt wurde.

2. **Archivgröße, Zeitstempel und Inhalt prüfen:**
   ```bash
   ls -lh images/linux/rootfs.tar.gz
   tar -tzf images/linux/rootfs.tar.gz | grep -E "usr/bin/(python3|pip3|dnf)$"
   ```
   *Erwartet:* Aktueller Zeitstempel (ca. 55–65 MB) und die Ausgabe:
   ```text
   ./usr/bin/dnf
   ./usr/bin/pip3
   ./usr/bin/python3
   ```
   *(Falls `rootfs.tar.gz` noch den alten Zeitstempel trägt, einmalig `petalinux-build` ohne Parameter ausführen.)*

---

## 5. Neues RootFS auf die SD-Karte übertragen (Windows 11 + WSL2)

> **Warum `usbipd` statt `wsl --mount`?**  
> `wsl --mount` unterstützt unter Windows 11 keine USB-Wechseldatenträger wie SD-Kartenleser (Fehlercode `0x8007000f`). Durch das Durchreichen des Kartenlesers per `usbipd-win` erscheint die SD-Karte direkt als Blockgerät (`/dev/sdX`) in WSL2.

### 5.1 USB-Kartenleser an WSL2 durchreichen (PowerShell als Administrator)

1. **Bus-ID des Kartenlesers ermitteln:**
   ```powershell
   usbipd list
   ```
   *(Typischer Eintrag: `05e3:0749 USB-Massenspeichergerät`, z. B. `BUSID 3-4`.)*

2. **Gerät freigeben (nur falls Status `Not shared`) und an WSL2 anhängen:**
   ```powershell
   usbipd bind --busid <BUSID>
   usbipd attach --wsl --busid <BUSID>
   ```

### 5.2 Partition mounten und RootFS ersetzen (WSL2)

1. **Zielpartition (`ext4`, Label `rootfs`) ermitteln:**
   ```bash
   lsblk -f
   ```
   *(Beispiel: `/dev/sde2` mit Typ `ext4` und Label `rootfs`.)*

2. **Partition mounten und unbedingt prüfen:**
   ```bash
   sudo mkdir -p /mnt/sd_rootfs
   sudo mount /dev/sdX2 /mnt/sd_rootfs
   df -hT /mnt/sd_rootfs
   ```
   > **Achtung:** Erst fortfahren, wenn `df -hT /mnt/sd_rootfs` eindeutig die ~28-GB-`ext4`-Partition der SD-Karte anzeigt!

3. **Altes RootFS leeren, neues Archiv mit Root-Rechten entpacken und synchronisieren:**
   ```bash
   sudo rm -rf /mnt/sd_rootfs/*
   sudo tar -xzf ~/petalinux_myd_c7z020/myd_c7z020_v2/images/linux/rootfs.tar.gz -C /mnt/sd_rootfs
   sync
   ls -l /mnt/sd_rootfs/usr/bin/python3 /mnt/sd_rootfs/usr/bin/dnf
   ```

4. **Partition sauber aushängen:**
   ```bash
   sudo umount /mnt/sd_rootfs
   ```

### 5.3 Kartenleser wieder trennen (PowerShell als Administrator)

```powershell
usbipd detach --busid <BUSID>
```
Anschließend die SD-Karte sicher entnehmen, ins Board einsetzen und booten.

---

## 6. Nach dem Booten: Login & Verifizierung auf dem Board

1. **Initialer Login:**  
   Durch das Ersetzen des RootFS werden Benutzerpasswörter und SSH-Host-Keys zurückgesetzt. Der erste Login erfolgt an der UART-Konsole (`ttyPS0`) als Benutzer `petalinux` (inkl. Passwortvergabe).  
   Für das anschließende Freischalten des direkten `root`-Logins über UART und SSH siehe **`petalinux_2026_2_root_setup.md`**.

2. **Python 3 und `dnf` testen:**
   ```bash
   python3 --version
   python3 -c "import mmap, json, http.server; print('Python OK')"
   dnf --version
   ```
   *Erwartete Ergebnisse:*
   * `Python 3.12.12`
   * `Python OK`
   * `dnf` Version `4.19.0` (`cortexa9t2hf_neon`)

3. **Gesicherte Dateien zurückspielen (optional):**
   ```bash
   scp -r ~/board_backup/root/* root@<board-ip>:/home/root/
   ```