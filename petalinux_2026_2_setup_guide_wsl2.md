# PetaLinux 2026.2 Installation & XSA-Import (Ubuntu 24.04 WSL2)

Diese Anleitung beschreibt die Installation von **AMD PetaLinux 2026.2** unter **Ubuntu 24.04 (WSL2)** sowie den Import der Hardware-Beschreibung (`.xsa`) für das **MYIR MYD-C7Z020-V2** (`MYC-C7Z020-V2-4E1D-766-I`) Entwicklungsboard.

> **Wichtiger Hinweis zu WSL2:** Sowohl die Installation von PetaLinux als auch alle Projekt-Builds müssen zwingend auf dem nativen Linux-Dateisystem (z. B. unter `~/`) stattfinden – niemals direkt auf eingebundenen Windows-Laufwerken (`/mnt/c/` oder `/mnt/d/`), da Yocto/BitBake dort mangels echter POSIX-Dateiberechtigungen fehlschlägt.

---

## 1. Systemvoraussetzungen & Paketabhängigkeiten installieren

Aktualisiere die Paketquellen in Ubuntu 24.04 und installiere alle von PetaLinux 2026.2 benötigten Build-Werkzeuge und Bibliotheken:

```bash
sudo apt update
sudo apt install -y iproute2 gawk python3 python3-pexpect python3-pip python3-git \
  python3-jinja2 xz-utils debianutils iputils-ping libegl1 libsdl1.2-dev \
  pylint xterm autoconf libtool texinfo zlib1g-dev gcc-multilib build-essential \
  libncurses-dev libglib2.0-dev libpixman-1-dev locales cpio unzip rsync bc \
  screen pax diffstat chrpath socat zstd lz4 file
```

---

## 2. System-Shell auf `bash` umstellen & Locale konfigurieren

PetaLinux setzt voraus, dass `/bin/sh` auf `bash` (und nicht auf `dash`) verweist und `en_US.UTF-8` verfügbar ist:

```bash
sudo ln -sf /bin/bash /bin/sh
sudo locale-gen en_US.UTF-8
```

*(Falls unter Ubuntu 24.04 AppArmor unprivilegierte User-Namespaces einschränkt, gegebenenfalls zusätzlich ausführen: `echo 0 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns`)*

---

## 3. Installer nach Linux kopieren und ausführbar machen

Kopiere den heruntergeladenen Installer aus dem Windows-Dateisystem in ein temporäres Linux-Verzeichnis und lege den Zielordner an:

```bash
mkdir -p ~/tmp_plnx ~/petalinux/2026.2
cp /mnt/c/Users/mimo3/Downloads/petalinux-v2026.2-11071905-installer.run ~/tmp_plnx/
chmod +x ~/tmp_plnx/petalinux-v2026.2-11071905-installer.run
```

---

## 4. PetaLinux 2026.2 installieren

Starte die Installation in das Zielverzeichnis `~/petalinux/2026.2` (mit `--skip_license` werden die Lizenzbedingungen automatisch akzeptiert):

```bash
~/tmp_plnx/petalinux-v2026.2-11071905-installer.run --dir ~/petalinux/2026.2 --skip_license
```

Nach erfolgreicher Installation kann der temporäre Ordner wieder gelöscht werden, um Speicherplatz freizugeben:

```bash
rm -rf ~/tmp_plnx
```

---

## 5. PetaLinux-Projekt anlegen & `.xsa` importieren

Lade die PetaLinux-Umgebung, kopiere die in Vivado erzeugte `.xsa`-Datei in das Linux-Projektverzeichnis und initialisiere das Zynq-7000-Template:

```bash
# Umgebungsvariablen laden
source ~/petalinux/2026.2/settings.sh

# Hardware-Beschreibung (.xsa) nach Linux kopieren
mkdir -p ~/petalinux_myd_c7z020/hw
cp /mnt/d/FPGA/myd_c7z020_v2/myd_c7z020_v2_hw/design_1_wrapper.xsa ~/petalinux_myd_c7z020/hw/

# Neues Zynq-Projekt anlegen und .xsa einlesen
cd ~/petalinux_myd_c7z020
petalinux-create project --template zynq --name myd_c7z020_v2
cd myd_c7z020_v2
petalinux-config --get-hw-description=../hw/
```

### Hinweise zum weiteren Vorgehen
* **Root-Dateisystem:** In der `menuconfig`-Oberfläche empfiehlt es sich für den Zynq-7020 unter `Image Packaging Configuration -> Root filesystem type` die Option **`EXT4 (SD card)`** statt `INITRAMFS` zu wählen.
* **Netzwerk-PHY (`YT8531SH`):** Für die V2-Hardware-Revision des `MYC-C7Z020` muss im Kernel `CONFIG_MOTORCOMM_PHY` aktiviert und der MDIO-Knoten in `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi` hinterlegt werden.