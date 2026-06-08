# com_rentalotplus_danalock

**Joomla 5/6 Komponente** · Danalock PIN-Verwaltung für Rentalot Plus

Verwaltet Danalock V3 PIN-Codes und Zeitfenster direkt aus dem Joomla-Backend,
verknüpft mit den Buchungsdaten von [Rentalot Plus](https://github.com/SvenPausH/Rentalotplus).

---

## Funktionen

- **Buchungsübersicht** – Alle aktuellen und kommenden Buchungen aus Rentalot Plus
- **PIN vergeben** – PIN-Code (4–10 Stellen) einem Slot (1–20) zuweisen und direkt ans Schloss senden
- **Zeitfenster** – Check-in und Check-out Datum+Uhrzeit werden als Zeitregel ans Schloss übertragen
- **Zufalls-PIN** – Generator für sichere 6-stellige PINs
- **Automatische Slot-Freigabe** – Abgelaufene Slots werden beim Laden automatisch freigegeben
- **Belegte Slots** – Im PIN-Dialog werden bereits vergebene Slots als „(belegt)" angezeigt
- **Alle PIN-Slots** – Übersicht aller 20 Schloss-Slots mit Gastname und Uhrzeiten aus der DB
- **Schloss-Status** – Zeigt ob das Schloss gesperrt oder offen ist
- **Debug-Log** – Vollständige API-Kommunikation protokollieren (ein-/ausschaltbar)
- **Mehrere Häuser** – Unterstützung mehrerer Schlösser über Unit-Mapping
- **Untermenü** – Erscheint automatisch als Untermenüpunkt von Rentalot Plus

---

## Voraussetzungen

| Anforderung | Version |
|---|---|
| Joomla | 5.x oder 6.x |
| PHP | 8.1 oder neuer |
| Rentalot Plus | Muss installiert sein |
| Danalock | V3 Schloss + Danabridge V3 |
| PHP-Extension | cURL, OpenSSL |

Ein aktives **Danalock-Konto** (my.danalock.com) ist erforderlich.

---

## Installation

1. ZIP-Paket herunterladen (Releases-Tab auf GitHub)
2. Joomla-Backend: **System → Installieren → Erweiterungen**
3. ZIP hochladen und installieren
4. Erscheint automatisch unter **Komponenten → Rentalot Plus → Danalock PIN-Verwaltung**

---

## Konfiguration

**Komponenten → Rentalot Plus → Danalock PIN-Verwaltung → Optionen** (oben rechts)

### Zugangsdaten

| Feld | Beschreibung |
|---|---|
| Danalock E-Mail | E-Mail-Adresse des Danalock-Kontos |
| Danalock Passwort | Passwort des Danalock-Kontos |
| Geräte-MAC (Standard) | MAC-Adresse des Schlosses (siehe unten) |
| Standard Check-in Zeit | Voreinstellung für neue Buchungen, z.B. `16:00` |
| Standard Check-out Zeit | Voreinstellung für neue Buchungen, z.B. `10:00` |
| Haus → Schloss Zuordnung | Für mehrere Häuser (siehe unten) |

### MAC-Adresse des Schlosses finden

Danalock-App → Schloss antippen → Zahnrad-Symbol → **Geräteinformationen**
Format: `06:cd:42:45:44:ff`

### Mehrere Häuser konfigurieren

Im Feld „Haus → Schloss Zuordnung" eine Zeile pro Haus:
```
1:06:cd:42:45:44:ff
2:06:cd:42:45:44:aa
```
Die `unit_id` entspricht der ID in `jos_rentalot_plus_bookings`.

---

## Nutzung

### Buchungen & PINs

Zeigt alle Buchungen mit `state = 0` (Reserviert) oder `state = 1` (Gebucht) ab heute.

**PIN mit Zeitfenster vergeben:**
1. Auf **✏️ PIN** klicken
2. Freien Slot wählen (belegte Slots sind als „(belegt)" markiert)
3. PIN eingeben oder per Zufallsbutton generieren
4. Check-in/Check-out Zeit anpassen (Datum kommt aus der Buchung)
5. **Speichern & an Schloss senden** → dauert ca. 20–30 Sekunden

**Slot freigeben:** 🗑-Button – gibt nur den DB-Eintrag frei,
der PIN bleibt im Schloss aktiv und wird beim nächsten Gast überschrieben.

**Automatische Freigabe:** Beim Öffnen der Übersicht werden Slots
deren Abreisezeitpunkt (`date_to + checkout_time`) in der Vergangenheit
liegt automatisch freigegeben.

### Alle PIN-Slots

Liest alle 20 Slots direkt vom Schloss (dauert ca. 10 Sekunden).
Zeigt Gastname und Zeiten aus der lokalen Datenbank.

> **Hinweis:** Die Danalock Bridge API gibt beim Auslesen keine
> Zeitfenster-Informationen zurück – nur Slot-Nummer, Aktiv/Frei
> und PIN-Ziffern. Zeiten kommen aus unserer DB.

### Debug-Log

Unter **Optionen → Debug & Logging** aktivierbar.
Protokolliert alle API-Aufrufe und Antworten in der Datenbank.
Nur zur Fehlersuche aktivieren.

---

## Datenbank

### `jos_rentalot_plus_danalock_pins`

| Spalte | Typ | Beschreibung |
|---|---|---|
| `booking_id` | INT | Verknüpfung mit `jos_rentalot_plus_bookings.id` |
| `pin_identifier` | TINYINT | Slot-Nummer (1–20) |
| `pin_code` | VARCHAR(10) | PIN-Code |
| `checkin_time` | VARCHAR(5) | Anreisezeit HH:MM |
| `checkout_time` | VARCHAR(5) | Abreisezeit HH:MM |
| `last_updated` | DATETIME | Letztes Speicherdatum |

### `jos_rentalot_plus_danalock_log`

Debug-Log-Einträge (nur bei aktiviertem Logging).

---

## Technische Details

### API-Flow (Übersicht)

Alle Bridge-Befehle sind **asynchron** – jeder Aufruf dauert ca. 10 Sekunden:

```
POST /bridge/v1/execute  →  {"id": "job-uuid"}
[7 Sekunden warten]
POST /bridge/v1/poll     →  Ergebnis mit afi_status: 0 = OK
```

### Authentifizierung

OAuth2 Password Grant gegen `api.danalock.com`:

```
POST https://api.danalock.com/oauth2/token
grant_type=password
username=<email>
password=<passwort>
client_id=danalock-web
client_secret=
```

Token wird in der Joomla-Session gecacht (1 Stunde).

### PIN setzen mit Zeitfenster

Beim Setzen eines zeitbasierten PINs werden drei Bridge-Befehle in Folge ausgeführt:

**Schritt 1: Bestehende Zeitregel abfragen**
```json
{
  "device": "06:cd:42:45:44:ff",
  "operation": "afi.pin-codes.get-time-restriction-rules",
  "arguments": ["12"]
}
```
Antwort enthält `rules[].handle` (integer) – die ID der Zeitregel.

**Schritt 2: Alte Zeitregel löschen** (falls `handle` vorhanden)
```json
{
  "device": "06:cd:42:45:44:ff",
  "operation": "afi.pin-codes.delete-time-restriction-rule",
  "arguments": [10]
}
```

**Schritt 3: PIN setzen**
```json
{
  "device": "06:cd:42:45:44:ff",
  "operation": "afi.pin-codes.set-pin-code",
  "arguments": ["12", "Enabled", "604113"]
}
```

**Schritt 4: Neue Zeitregel erstellen**
```json
{
  "device": "06:cd:42:45:44:ff",
  "operation": "afi.pin-codes.create-time-restriction-rule",
  "arguments": [
    "12",
    "2026 6 13 15:46 DISABLED 00:00",
    "2026 6 19 10:00 DISABLED 00:00"
  ]
}
```

Zeitformat: `"YYYY M D HH:MM DISABLED 00:00"`
- Monat und Tag **ohne** führende Null (6 nicht 06, 13 nicht 13)
- `DISABLED 00:00` ist fester Bestandteil des Formats

Erfolgreiche Antwort:
```json
{
  "result": {
    "status": "Succeeded",
    "result": {
      "handle": 10,
      "afi_status": 0,
      "afi_status_text": "Ok"
    }
  }
}
```

### PIN ohne Zeitfenster setzen

Nur Schritt 3 (`set-pin-code`) ohne Zeitregel-Erstellung.

### PIN-Slot freigeben (nur DB)

Löscht den Eintrag in `jos_rentalot_plus_danalock_pins`.
Das Schloss wird **nicht** verändert – der PIN bleibt aktiv
und wird beim nächsten Gast durch `set-pin-code` überschrieben.

### Alle PINs auslesen

```json
{
  "device": "06:cd:42:45:44:ff",
  "operation": "afi.pin-codes.get-pin-codes",
  "arguments": ["20"]
}
```

Antwort:
```json
{
  "result": {
    "pin_codes": [
      {"identifier": 1, "status": 2, "digits": "123456"},
      {"identifier": 2, "status": 1, "digits": ""}
    ]
  }
}
```
`status`: 1 = frei, 2 = aktiv. Zeitfenster-Informationen werden **nicht** zurückgegeben.

### Schloss-Status abfragen

```json
{
  "device": "06:cd:42:45:44:ff",
  "operation": "afi.lock.get-state",
  "arguments": []
}
```

Antwort: `{"result": {"state": "Locked"}}` oder `"Unlocked"`

---

## Namespace

```
Joomla\Component\RentalotplusDanalock\Administrator\...
```

---

## Lizenz

GNU General Public License version 2 or later

Basiert auf der inoffiziellen Danalock API-Dokumentation von
[@gechu](https://github.com/gechu/unofficial-danalock-web-api) und
[@Dan1001](https://github.com/Dan1001/danabridge-python).
API-Details durch Analyse des Netzwerkverkehrs von my.danalock.com ermittelt.

---

## Changelog

### 1.0.0
- Erstveröffentlichung
- Buchungsübersicht mit PIN-Verwaltung und Zeitfenstern
- Vollständige Bridge-API Integration (set, delete, get, time-restriction-rules)
- Automatische Slot-Freigabe bei abgelaufenen Buchungen
- Belegte Slots im PIN-Dialog markiert
- Debug-Logging
- Mehrhaus-Unterstützung
- Automatische Einordnung als Rentalot-Plus-Untermenü
