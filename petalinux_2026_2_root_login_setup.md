# Initiales Setzen des `root`-Passworts (PetaLinux 2026.2)

**System:** `myd_c7z020_v2` (Zynq-7000)  
**OS:** PetaLinux 2026.2 (persistentes `ext4`-RootFS, OpenSSH, SysVinit)

## Hintergrund

Bei einem frisch gebauten PetaLinux-2026.2-RootFS (ohne aktivierte Image-Features `empty-root-password` oder `debug-tweaks`) ist der direkte Login als `root` standardmäßig gesperrt. Der erste Zugriff erfolgt daher über den vorkonfigurierten Standardbenutzer `petalinux` mit `sudo`-Rechten.

---

## 1. Erster Login über den Standardbenutzer `petalinux`

Verbinde dich über die serielle Konsole (UART, `ttyPS0`) mit dem Board und melde dich mit dem Standardbenutzer an:

```text
myd_c7z020_v2 login: petalinux
```

* Beim allerersten Start des neuen RootFS fordert das System direkt dazu auf, ein neues Passwort für `petalinux` zu vergeben und zu bestätigen.
* Anschließend landest du am Shell-Prompt (`myd_c7z020_v2:~$`).

---

## 2. Passwort für `root` setzen (UART-Login freischalten)

Setze über `sudo` ein Passwort für das `root`-Konto. Dadurch wird der Benutzer in `/etc/shadow` entsperrt und der direkte `root`-Login an der UART-Konsole funktioniert ab sofort:

```bash
sudo passwd root
```

**Verifizierung:**

```bash
su - root
```

Nach Eingabe des neuen Passworts wechselt der Prompt auf `root@myd_c7z020_v2:~#`. Mit `exit` kehrst du bei Bedarf zurück.

---

## 3. Direkten `root`-Login per SSH erlauben (OpenSSH)

Standardmäßig verweigert OpenSSH die Anmeldung als `root` mit Passwort. Aktiviere `PermitRootLogin yes` in der SSH-Konfiguration:

```bash
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo grep -q '^PermitRootLogin yes' /etc/ssh/sshd_config || echo "PermitRootLogin yes" | sudo tee -a /etc/ssh/sshd_config
```

**Verifizierung:**

```bash
grep '^PermitRootLogin' /etc/ssh/sshd_config
```

Erwartete Ausgabe:

```text
PermitRootLogin yes
```

---

## 4. SSH-Dienst neu starten

Übernehme die neue Konfiguration durch einen Neustart des OpenSSH-Daemons (SysVinit):

```bash
sudo /etc/init.d/sshd restart
```

**Verifizierung:**

```bash
pidof sshd
```

Der Befehl gibt eine gültige Prozess-ID (PID) zurück.

---

## 5. SSH-Login vom Host-PC testen

Verbinde dich vom PC aus per SSH als `root`:

```bash
ssh root@<board-ip>
```

> **Hinweis nach einem RootFS-Update:** Da bei einem neu entpackten RootFS neue SSH-Host-Keys generiert werden, kann der PC die Warnung `REMOTE HOST IDENTIFICATION HAS CHANGED` melden. Entferne in diesem Fall einmalig den alten Eintrag auf dem Host-PC:
>
> ```bash
> ssh-keygen -R <board-ip>
> ```

---

## Sicherheitshinweis

Das Freischalten von `PermitRootLogin yes` mit Passwort-Authentifizierung ist für die lokale Entwicklung im Labornetz gedacht. Für produktive oder öffentlich erreichbare Systeme sollte stattdessen SSH-Key-Authentifizierung verwendet und der Passwort-Login für `root` deaktiviert bleiben.