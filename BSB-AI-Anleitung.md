# BSB & BSB-LAN – Arbeitsanleitung für KI-Assistenten

Diese Datei erklärt einer KI, wie der Siemens **Boiler-System-Bus (BSB)** funktioniert und wie man über **BSB-LAN** sicher mit der Heizung in dieser Installation arbeitet. Alle Protokolldetails wurden gegen den Quellcode in `BSB-LAN/BSB_LAN/` geprüft (Stand: Firmware 5.1.43, November 2026). Bei Widersprüchen gilt der Code.

---

## 1. Sicherheitsregeln (zuerst lesen)

1. **Nur lesen, solange der Nutzer nichts anderes sagt.** Schreibzugriffe (`/S…`, `/JS`, SET-Telegramme) verändern die Kesselregelung und können Komfort, Effizienz oder im Extremfall die Anlagensicherheit beeinträchtigen. Vor jedem Schreiben: Parameter, alter Wert, neuer Wert dem Nutzer nennen und ausdrückliche Bestätigung einholen.
2. **Keine Telegramme von Hand bauen und senden.** Immer die BSB-LAN-HTTP-API verwenden. Die Firmware kümmert sich um Buszugriff, CRC, Wiederholungen und Datentyp-Kodierung.
3. **Keine Parameter aus fremden Listen schreiben.** Parameternummern sind nur zusammen mit der gerätespezifischen Liste (`BSB_LAN_custom_defs.h`) eindeutig. Die alte Sammelliste aus BSB-LAN 2.2.x enthält Fehler und Mehrdeutigkeiten.
4. **Hardware:** Im Kessel liegt 230 V an. Anweisungen zu Verkabelung immer mit dem Hinweis „Kessel vorher stromlos schalten“ versehen. Am Kessel nur `CL+`/`CL-` verwenden, nie `G+`.
5. **Keine Zugangsdaten ausgeben.** `BSB_LAN_config.h` enthält WLAN-Passwort u. ä. – nie im Klartext anzeigen oder an externe Dienste schicken.

---

## 2. Diese Installation

| Element | Wert |
|---|---|
| Heizung | Brötje WGB EVO 15H (Gas-Brennwert) |
| Regler (Busadresse 0x00) | **LMS15.001A100**, Gerätefamilie **163**, Variante **5** |
| Bedienteil (Busadresse 0x0A = 10) | AVS37.294/100 |
| Bus | BSB, BSB-LAN hat Adresse **0x42** (66) |
| Mikrocontroller | Olimex ESP32-EVB (ESP32-WROOM-32E), RX/TX-Pins 36/17 (Auto-Erkennung) |
| Bus-Adapter | BSB-LAN Logic Level Adapter 3.3V/12V V4.5 |
| Anschluss am Kessel | Raumgeräte-Klemme (QAA): Adapter `+` → `CL+`, `-` → `CL-` |
| Netzwerk | WLAN, IP **192.168.100.50**, mDNS `bsb-lan.local` |
| Firmware | BSB-LAN 5.1.43, esp32-Core 3.3.12 |
| Quellcode | `/home/frank/repositories/BSBLan/BSB-LAN/BSB_LAN/` |

**Stand der Parameterliste:** Es ist noch die generische `BSB_LAN_custom_defs.h` aktiv – nur wenige Basisparameter sind sichtbar (z. B. 6224–6226, 8700). Die gerätespezifische Liste muss erst erstellt werden (siehe Abschnitt 6).

---

## 3. Physikalische Schicht

- Zweidraht-Bus, `CL+` (Daten) und `CL-` (Masse). Busspannung im Ruhezustand ca. 12–15 V; der Kessel speist den Bus.
- Ein Bit wird gesendet, indem der Bus nach unten gezogen wird. Der Adapter übersetzt auf 3,3 V für den ESP32.
- UART: **4800 Baud, 8 Datenbits, ungerade Parität, 1 Stoppbit (8O1)**.
- **Alle Bytes sind invertiert:** Die Firmware liest/schreibt `byte ^ 0xFF` (`src/BSB/bsb.cpp`, `readByte()`). Hex-Dumps in BSB-LAN zeigen immer die bereits zurückinvertierten, also logischen Werte.
- Mehrere Geräte dürfen parallel an denselben Klemmen hängen (Raumgerät + BSB-LAN ist normal).
- Buszugriff: Vor dem Senden wartet BSB-LAN 63–82 ms (59 ms Mindestpause zwischen Telegrammen + Zufallsanteil) und prüft, ob der Bus frei ist. Bei Aktivität wird neu gewartet.

---

## 4. Telegrammaufbau (BSB)

```
Byte:  0     1      2      3     4     5   6   7   8    9 … n-3   n-2  n-1
       SOF   SRC    DEST   LEN   TYPE  CMD (4 Bytes)    PAYLOAD   CRC (2 Bytes)
```

| Feld | Bedeutung |
|---|---|
| `SOF` | Start of Frame, immer `0xDC` (`0xDE` kommt ebenfalls vor) |
| `SRC` | Absenderadresse **mit gesetztem Bit 7**: `0x80 | Adresse`. BSB-LAN sendet `0xC2`, der Kessel `0x80` |
| `DEST` | Empfängeradresse ohne Bit 7 (`0x00` Kessel, `0x42` BSB-LAN, `0x7F` alle) |
| `LEN` | Gesamtlänge des Telegramms inkl. SOF und CRC (max. 32) |
| `TYPE` | Telegrammtyp, siehe unten |
| `CMD` | 4-Byte-Command-ID (Parameter-Kennung im Regler) |
| `PAYLOAD` | Daten, abhängig vom Datentyp |
| `CRC` | CRC-16 CCITT/XMODEM (Polynom `0x1021`, Startwert `0x0000`) über alle Bytes davor, High-Byte zuerst. Über das **ganze** Telegramm inkl. CRC gerechnet ergibt sich `0` |

### Telegrammtypen (`BSB_LAN_defs.h`)

| Wert | Name | Bedeutung |
|---|---|---|
| `0x01` | QINF | Info anfordern |
| `0x02` | INF | Info senden (Broadcast, z. B. Raumtemperatur vom Raumgerät) |
| `0x03` | SET | Parameter setzen |
| `0x04` | ACK | Setzen bestätigt |
| `0x05` | NACK | Setzen abgelehnt |
| `0x06` | QUR | Parameter abfragen |
| `0x07` | ANS | Antwort auf Abfrage |
| `0x08` | ERR | Fehler (z. B. Parameter unbekannt) |
| `0x0F` / `0x10` | QRV / ARV | Reset-/Werkswert abfragen / Antwort |
| `0x12` / `0x13` | IQ1 / IA1 | Interne Abfrage Typ 1 – liefert Parameterstruktur (für Dump) |
| `0x14` / `0x15` | IQ2 / IA2 | Interne Abfrage Typ 2 – liefert Parameterstruktur (für Dump) |

### Byte-Vertauschung der Command-ID

In **Anfragen** (QUR, SET usw.) sind die **ersten beiden Bytes der Command-ID vertauscht**; in der **Antwort** stehen sie in Originalreihenfolge (`BSB::Send()` in `src/BSB/bsb.cpp`).

Beispiel Außentemperatur, Command-ID `0x053D0521`:
- Anfrage enthält `3D 05 05 21`
- Antwort enthält `05 3D 05 21`

### Busadressen (`BSB_LAN_defs.h`)

| Adresse | Gerät |
|---|---|
| `0x00` | Kessel/Regler |
| `0x03`, `0x04` | Erweiterungsmodule |
| `0x06`–`0x08` | Raumgeräte 1–3 (QAA) |
| `0x0A` | Display/Bedienteil 1 (hier AVS37) |
| `0x0B` | Service-Tool |
| `0x0C`, `0x0D` | Displays 2–3 |
| `0x31` | OZW (Siemens-Webserver) |
| `0x32` | Funkmodul |
| `0x42` | BSB-LAN |
| `0x7F` | Broadcast |

### Durchgerechnetes Beispiel: Außentemperatur 8700

Anfrage von BSB-LAN an den Kessel (verifizierte CRC):
```
DC C2 00 0B 06 3D 05 05 21 B9 7C
│  │  │  │  │  └─────────┘ └───┘
│  │  │  │  │  CMD (vertauscht) CRC
│  │  │  │  └ QUR
│  │  │  └ Länge 11
│  │  └ an Kessel (0x00)
│  └ von 0x42 (0x80|0x42)
└ SOF
```

Mögliche Antwort (konstruiertes Beispiel mit korrekter CRC, Wert 5,2 °C):
```
DC 80 42 0E 07 05 3D 05 21 00 01 4D 17 05
                           │  └───┘
                           │  0x014D = 333 → 333 / 64 = 5,20 °C
                           └ Flag 0x00 = Wert gültig
```

### Payload-Kodierung

Der Datentyp steht pro Parameter in der Tabelle (`VT_…`, definiert in `BSB_LAN_defs.h`). Häufige Fälle:
- **`VT_TEMP`**: 3 Bytes = 1 Flag-Byte + vorzeichenbehaftetes 16-Bit-Wort, **Wert = Wort / 64** (°C).
- Viele Typen beginnen mit einem **Flag-Byte**: `0x00` = gültiger Wert; andere Werte bedeuten u. a. „aus/---“ bei abschaltbaren (nullable) Parametern.
- Enums, Bitfelder, Zeiten (HH:MM), Datum/Uhrzeit und Zeitprogramme haben eigene Kodierungen – nicht raten, sondern Typ in `BSB_LAN_defs.h` und Dekodierung in `BSB_LAN.ino` nachschlagen.

---

## 5. BSB-LAN: Parameter und HTTP-API

### Parameternummern vs. Command-IDs

- Der Nutzer sieht **Parameternummern** (z. B. 8700 = Außentemperatur), die den Zeilennummern im Bedienmenü des Reglers entsprechen.
- Die Zuordnung Parameternummer → Command-ID → Datentyp steht in **`cmdtbl[]`** in `BSB_LAN_custom_defs.h`, z. B.:
  ```c
  {0x053D0521, VT_TEMP, 8700, STR8700, 0, NULL, DEFAULT_FLAG, DEV_ALL}, // Außentemperatur
  ```
- Dieselbe Parameternummer kann bei verschiedenen Reglerfamilien unterschiedliche Command-IDs haben. Deshalb ist die gerätespezifische Liste nötig.
- Parameter ab 20000 sind virtuell (Durchschnitte, 1-Wire, DHT22 usw.) und existieren nur in BSB-LAN, nicht im Kessel.
- Optional kann ein Ziel angegeben werden: `8700!10` fragt Parameter 8700 bei Busadresse 10 (Bedienteil) statt beim Kessel ab.

### Wichtige Endpunkte (Basis: `http://192.168.100.50`)

| Endpunkt | Zweck | Schreibend? |
|---|---|---|
| `/JI` | Laufzeitinfos: Version, Bus, erkannte Busgeräte, Uptime | nein |
| `/JQ=8700,6225` | Werte als JSON abfragen | nein |
| `/JK=ALL` bzw. `/JK=<kat>` | Kategorien bzw. Parameterstruktur einer Kategorie | nein |
| `/JC=8700` | Struktur (Typ, Einheit, Enum-Werte) eines Parameters | nein |
| `/JB` | Backup aller beschreibbaren Parameter als JSON | nein |
| `/<nr>` bzw. `/<von>-<bis>` | Werte als HTML | nein |
| `/K<kat>` | Kategorie als HTML | nein |
| `/Q` | Seite für gerätespezifische Parameterliste | nein |
| `/QD` | Download des Struktur-Dumps (dauert mehrere Minuten) | nein |
| `/JS` (POST) | Parameter setzen (JSON) | **ja** |
| `/S<nr>=<wert>` | Parameter setzen | **ja** |
| `/JW` (POST) | BSB-LAN-Konfiguration schreiben | **ja** (BSB-LAN, nicht Kessel) |
| `/N` | Mikrocontroller neu starten | – |
| `/NE` | EEPROM löschen + Neustart | **ja**, Netzwerkzugang kann verloren gehen |

Antwortfelder bei `/JQ`: `value`, `unit`, `desc` (Enum-Text), `dataType`, `readwrite` (0 = lesen/schreiben, 1 = nur lesen, 2 = nur schreiben) und **`error`** (0 = ok). `error: 261` heißt: Parameter vom Gerät nicht beantwortet (z. B. Bus nicht verbunden oder Parameter existiert nicht).

Lesebeispiel:
```bash
curl -s "http://192.168.100.50/JQ=8700" 
```

Vollständige API-Beschreibung: `BSB-LAN/openapi.yaml` und `BSB-LAN/docs/de/using.md`.

---

## 6. Gerätespezifische Parameterliste

1. Im Browser `http://192.168.100.50/Q` öffnen und **„Download“** klicken. Die Seite fragt den Fortschritt parallel ab; der Tab muss bis zum Ende offen bleiben (5–15 Minuten).
   - Ein reiner `curl` auf `/QD` hat in dieser Installation nach ca. 1029 Zeilen mit „connection reset“ (Exit 56) abgebrochen. Der Dump war dadurch unvollständig.
2. Ergebnis: `BSB-LAN-163-005-10-172980196.txt`. **Vollständigkeitsprüfung:**
   - Header mit `Gerätefamilie: 163`, `Gerätevariante: 5`.
   - Nach `Complete dump:` zuerst 10 `ANS`-Telegramme (Typ `0x07`), dann `IA1`-Zeilen (Typ `0x13`) und danach `IA2`-Zeilen (Typ `0x15`).
   - Die Datei muss mit der Abschlusszeile `Fertig.` enden (`MENU_TEXT_QFE` in `localization/LANG_DE.h`). Fehlt sie oder fehlen die `IA2`-Zeilen, ist der Dump unvollständig.
   - Keine Zeilen mit `[Timeout]` oder `Didn't receive`.
3. Der Nutzer schickt die Datei an den BSB-LAN-Entwickler (Adresse in `docs/de/install.md`). Die Datei enthält nur die Parameterstruktur, keine Einstellungen. **Die KI verschickt keine Mails im Namen des Nutzers.**
4. Die zurückgeschickte `BSB_LAN_custom_defs.h` ersetzt die Datei in `BSB-LAN/BSB_LAN/`; danach neu kompilieren und flashen (Abschnitt 7).

---

## 7. Build & Flash

```bash
cd /home/frank/repositories/BSBLan/BSB-LAN/BSB_LAN
FQBN=esp32:esp32:esp32-evb:PartitionScheme=min_spiffs,UploadSpeed=460800
arduino-cli compile -b $FQBN --output-dir /tmp/bsb-build .
arduino-cli upload  -p /dev/ttyUSB0 -b $FQBN --input-dir /tmp/bsb-build .
```

- Partitionsschema **muss** `min_spiffs` („Minimal SPIFFS“) sein; die Firmware belegt ca. 79 % des App-Speichers.
- USB-Seriell-Chip: CH340 → `/dev/ttyUSB0`. Der Nutzer ist in der Gruppe `uucp`.
- „Hard resetting via RTS pin...“ am Ende ist **kein** Fehler.
- **EEPROM-Falle:** `UseEEPROM = 1`. Nach dem ersten Start stehen die Einstellungen im EEPROM und **haben Vorrang vor `BSB_LAN_config.h`**. Änderungen in der Datei werden ignoriert, außer man:
  - die Einstellung im Web-Interface ändert (bevorzugt), oder
  - mit `EraseFlash=all` flasht (löscht alle gespeicherten Einstellungen; der Nutzer muss vorher zustimmen), oder
  - `/NE` aufruft (gleiche Wirkung, ohne Flashen).
- Serielle Konsole: 115200 Baud. `arduino-cli monitor` hält den EVB eventuell im Reset; zuverlässig ist `stty -F /dev/ttyUSB0 115200 raw -echo -hupcl` plus `cat /dev/ttyUSB0`.
- OTA-Updates sind möglich, wenn sie in den Einstellungen aktiviert sind (Port 8080).

---

## 8. Fehlersuche

| Symptom | Ursache / Prüfung |
|---|---|
| `busdevices` in `/JI` leer, `error: 261` überall | Bus nicht verbunden, Polarität vertauscht oder an `G+` statt `CL+` |
| Rote LED am Adapter aus | Keine Busspannung: Kabel/Klemme/Polarität prüfen |
| Rote LED leuchtet dauerhaft ohne Flackern | Wahrscheinlich an `G+` (Versorgung, keine Daten) |
| `unknown destination ID … device family: 0` beim Boot | Kessel antwortet nicht – normal, solange nicht angeschlossen |
| `GPIO number error (input-only pad has no internal PU)` beim Boot | Harmlos: GPIO 36 ist reiner Eingang |
| Zufällige Neustarts, „Brownout detector was triggered“ | Netzteil zu schwach; 5 V, mindestens 1 A (empfohlen 2 A) |
| Änderungen in `BSB_LAN_config.h` wirken nicht | EEPROM-Vorrang, siehe Abschnitt 7 |
| Parameter fehlt in der Weboberfläche | Generische Liste aktiv → gerätespezifische Liste erstellen |

---

## 9. Weiterführende Quellen im Repo

- `BSB-LAN/docs/de/` – Nutzerdokumentation (install, configure, using, troubleshooting, faq)
- `BSB-LAN/openapi.yaml` – HTTP/JSON-API
- `BSB-LAN/BSB_LAN/src/BSB/bsb.cpp` – Busprotokoll (Senden, Empfangen, CRC, Buszugriff)
- `BSB-LAN/BSB_LAN/BSB_LAN_defs.h` – Telegrammtypen, Adressen, Datentypen
- `BSB-LAN/BSB_LAN/BSB_LAN_custom_defs.h` – Parametertabelle `cmdtbl[]`
- `BSB-LAN/BSB_LAN/BSB_LAN.ino` – Hauptlogik inkl. Dump (`/QD`) und Werte-Dekodierung
