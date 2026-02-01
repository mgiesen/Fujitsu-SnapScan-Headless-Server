# Headless Scan Server: Fujitsu ScanSnap iX1300 auf Proxmox

Dieses Projekt verwandelt einen **Fujitsu ScanSnap iX1300** in einen vollautomatischen und zentralen Netzwerk-Scanner.

Das Ziel: Der Scanner steht zentral in der Wohnung (z.B. auf dem Medienschrank). Man legt ein Dokument ein (egal ob oben in den Einzug oder vorne in den Rücklauf), drückt den blauen Knopf, und wenige Sekunden später liegt das fertige PDF auf dem lokalen Fileserver (NAS).

**Die Philosophie:**

- 🔒 **100% Lokal:** Keine Cloud, kein Zwangs-Account, keine App-Pflicht.
- 🚀 **One-Button-Workflow:** Keine GUI notwendig. Drücken & Fertig.
- 🐳 **Containerisiert:** Läuft ressourcenschonend in einem Proxmox LXC Container.

![Bild vom Medienschrank mit Scanner](readme/IMG_2845.webp)

## Voraussetzungen

- **Host:** Proxmox VE Server.
- **Gast:** Ein unprivilegierter LXC Container (Debian oder Ubuntu).
- **Hardware:** Fujitsu ScanSnap iX1300 (oder kompatibel via SANE).
- **Speicher:** Ein Mountpoint (Bind-Mount) vom Host in den Container, wo die PDFs landen sollen (z.B. `/mnt/scans`).

## Schritt 1: Einrichtung auf dem Proxmox Host

Das größte Problem bei USB-Scannern an virtuellen Maschinen/Containern ist der Energiesparmodus. Wenn der Scanner "einschläft" oder der Deckel geschlossen wird, verliert er die USB-Verbindung. Beim Aufwachen erhält er eine neue Adresse, und der Container findet ihn nicht mehr.

Wir lösen dies mit einer **Udev-Regel** direkt auf dem Proxmox Host.

### 1.1 USB-ID ermitteln

Verbinde den Scanner und führe auf dem Host `lsusb` aus. Suche nach Fujitsu.
Beispiel ID: `04c5:162c` (Vendor:Product).

### 1.2 Regel auf Host-System erstellen

Diese Regel macht drei Dinge, sobald der Scanner erkannt wird:

1.  Sie identifiziert das Gerät.
2.  Sie setzt die Rechte auf `0666` (jeder darf schreiben), damit der Scan-Container zugreifen darf.
3.  Sie startet den Scan-Dienst im Container neu (`systemctl restart scanbd`), damit dieser das "neue" Gerät sofort bemerkt.

Erstelle die Datei `/etc/udev/rules.d/99-scanner-fix.rules` auf dem **Proxmox Host**:

```bash
# Ersetze '114' mit der ID deines LXC Containers!
SUBSYSTEM=="usb", ATTRS{idVendor}=="04c5", ATTRS{idProduct}=="162c", MODE="0666", RUN+="/bin/sh -c 'sleep 5; /usr/sbin/pct exec 114 -- systemctl restart scanbd'"
```

Regeln neu laden:

```bash
udevadm control --reload-rules
```

## Schritt 2: Einrichtung im LXC Container

Wechsle nun in den Container. Wir installieren die nötigen Werkzeuge.

### 2.1 Software installieren

```bash
apt update && apt install -y sane-utils scanbd imagemagick img2pdf
```

**Was machen diese Tools?**

- **`sane-utils` (Scanner Access Now Easy):** Der eigentliche Treiber. Er kommuniziert mit der Hardware. `scanimage` ist das Befehlszeilen-Tool hiervon.
- **`scanbd` (Scanner Button Daemon):** Ein Hintergrunddienst, der den "Knopf" am Scanner überwacht. Sobald er gedrückt wird, führt er unser Skript aus.
- **`imagemagick` (`mogrify`):** Das "Schweizer Taschenmesser" für Bildbearbeitung. Wir nutzen es, um die Bilder zu drehen (da der iX1300 technisch oft "auf dem Kopf" scannt).
- **`img2pdf`:** Ein Tool, das JPEGs verlustfrei und schnell in ein PDF verpackt.

### 2.2 Konfiguration von scanbd

Der Daemon `scanbd` muss wissen, was er tun soll, wenn der Knopf gedrückt wird.

1.  Öffne `/etc/scanbd/scanbd.conf`.
2.  Suche den Abschnitt `action scan`.
3.  Ändere das Skript auf unseren eigenen Pfad:

```xml
action scan {
    filter = "^scan.*"
    numerical-trigger {
        from-value = 1
        to-value   = 0
    }
    desc   = "Scan to File"
    # Hier unser Skript eintragen:
    script = "myscan.sh"
}
```

_Hinweis:_ Oft muss man in dieser Config auch `saned` deaktivieren oder sicherstellen, dass `scanbd` exklusiven Zugriff hat. In vielen Fällen reicht die Standardinstallation, solange `scanbd` als `root` läuft.

## Schritt 3: Das Skript

Das Herzstück ist das Skript `/etc/scanbd/scripts/myscan.sh`. Es übernimmt die gesamte Logik:

- Erkennt den Scanner dynamisch.
- Versucht zuerst den oberen Einzug (ADF), dann den vorderen (Card).
- Erzwingt A4-Format (verhindert weiße Balken bei US-Letter-Defaults).
- Löscht leere Seiten automatisch.
- Erstellt das PDF.

Erstelle die Datei und mache sie ausführbar (`chmod +x`).

```bash
#!/bin/bash

# --- Konfiguration ---
# Pfad zum NAS Mountpoint
OUT_DIR="/mnt/scans"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
TMP_DIR="/tmp/scan_$DATE"
FINAL_FILE="$OUT_DIR/Scan_$DATE.pdf"
LOG_FILE="/tmp/scanbd_script.log"
LOCK_FILE="/tmp/scan.lock"

# --- 0. Sicherheitscheck (Verhindert doppeltes Ausführen) ---
if [ -f "$LOCK_FILE" ]; then
    echo "[$DATE] ABBRUCH: Scan läuft bereits." >> $LOG_FILE
    exit 1
fi
touch "$LOCK_FILE"
trap "rm -f '$LOCK_FILE'; rm -rf '$TMP_DIR'" EXIT

mkdir -p "$TMP_DIR"
echo "[$DATE] --- Start Scan ---" >> $LOG_FILE

# --- 1. Gerät ermitteln ---
# scanbd übergibt das Gerät oft per Environment. Fallback auf generischen Namen.
SCAN_DEV="${SCANBD_DEVICE:-fujitsu:ScanSnap iX1300}"

# --- 2. Scan-Versuch 1: ADF Duplex (Einzug Oben) ---
# --swskip=2 : Löscht Seiten mit < 2% Tintendeckung (leere Rückseiten)
# --page-width/height & -x/-y : Erzwingt echtes A4, entfernt weiße Ränder
echo "[$DATE] Prüfe ADF (Oben)..." >> $LOG_FILE
scanimage -d "$SCAN_DEV" \
  --source "ADF Duplex" \
  --mode Color --resolution 200 \
  --page-width=210 --page-height=297 -x 210 -y 297 \
  --swskip=2 \
  --batch="$TMP_DIR/page_%04d.jpg" --format=jpeg >> $LOG_FILE 2>&1

# Check: Hat ADF Bilder geliefert? Wenn nein, versuche vorne.
if ! ls "$TMP_DIR"/page_*.jpg 1> /dev/null 2>&1; then

    # --- 3. Scan-Versuch 2: Front-Einzug Duplex (Unten/Card) ---
    echo "[$DATE] ADF leer. Versuche Einzug Vorne (Card)..." >> $LOG_FILE
    scanimage -d "$SCAN_DEV" \
      --source "Card Duplex" \
      --mode Color --resolution 200 \
      --page-width=210 --page-height=297 -x 210 -y 297 \
      --swskip=2 \
      --batch="$TMP_DIR/page_%04d.jpg" --format=jpeg >> $LOG_FILE 2>&1
fi

# --- 4. Verarbeitung ---
if ls "$TMP_DIR"/page_*.jpg 1> /dev/null 2>&1; then
    echo "[$DATE] Bilder empfangen. Erstelle PDF..." >> $LOG_FILE

    # Bilder drehen (iX1300 scannt Hardware-bedingt oft "falsch rum")
    mogrify -rotate 180 "$TMP_DIR"/*.jpg

    # PDF erstellen (img2pdf ist performanter als convert)
    img2pdf "$TMP_DIR"/*.jpg --output "$FINAL_FILE" --pagesize A4 >> $LOG_FILE 2>&1

    if [ -f "$FINAL_FILE" ]; then
         echo "[$DATE] Erfolg: $FINAL_FILE" >> $LOG_FILE
         # Rechte anpassen, damit jeder User das PDF öffnen/löschen darf
         chmod 666 "$FINAL_FILE"
    else
         echo "[$DATE] FEHLER: PDF fehlgeschlagen." >> $LOG_FILE
    fi
else
    echo "[$DATE] FEHLER: Keine Seiten (oder alle als leer erkannt)." >> $LOG_FILE
fi
```

## Schritt 4: Automatisierung & Stabilität

Damit das System auch nach einem Stromausfall sauber läuft:

1.  **Dienst aktivieren:**

    ```bash
    systemctl enable scanbd
    systemctl restart scanbd
    ```

2.  **Aufräumen beim Booten:**
    Falls der Server während eines Scans abstürzt, bleibt die `scan.lock` Datei liegen. Wir löschen sie beim Start.

    `crontab -e` im Container öffnen und hinzufügen:

    ```bash
    @reboot rm -f /tmp/scan.lock
    ```

## Nutzung

1.  Klappe des iX1300 öffnen (Scanner wacht auf, Proxmox reicht USB durch, Dienst startet neu).
2.  Papier einlegen (Oben viele Blätter oder Vorne ein einzelnes/dickes).
3.  **Blauen "Scan"-Knopf drücken.**
4.  Warten (ca. 5-10 Sekunden nach Durchzug).
5.  Das PDF liegt auf dem Netzwerklaufwerk.

## Troubleshooting

Sollte mal nichts passieren:

- Log prüfen: `cat /tmp/scanbd_script.log`
- Läuft der Dienst? `systemctl status scanbd`
- Sieht der Container den Scanner? `scanimage -L`
