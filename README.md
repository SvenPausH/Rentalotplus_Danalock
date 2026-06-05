# com_rentalotplus_danalock

**Joomla 5/6 Komponente** · Danalock PIN-Verwaltung für Rentalot Plus

Verwaltet Danalock PIN-Codes direkt aus dem Joomla-Backend, verknüpft mit den Buchungsdaten von [Rentalot Plus](https://github.com/SvenPausH/Rentalotplus).

---

## Funktionen

- **Buchungsübersicht** – Alle aktuellen und kommenden Buchungen aus Rentalot Plus auf einen Blick
- **PIN vergeben** – PIN-Code (4–10 Stellen) einem Buchungs-Slot (1–20) zuweisen, direkt ans Schloss senden
- **Zeitbasierte PINs** – Check-in und Check-out Zeiten werden mit jedem PIN gespeichert
- **Zufalls-PIN** – Generator für sichere 6-stellige PINs
- **Alle PIN-Slots** – Übersicht aller 20 Schloss-Slots mit belegtem Gast und Uhrzeiten
- **Schloss-Status** – Zeigt ob das Schloss gesperrt oder offen ist
- **Debug-Log** – Vollständige API-Kommunikation protokollieren (ein-/ausschaltbar)
- **Mehrere Häuser** – Unterstützung mehrerer Schlösser über Unit-Mapping

---

## Voraussetzungen

| Anforderung | Version |
|---|---|
| Joomla | 5.x oder 6.x |
| PHP | 8.1 oder neuer |
| Rentalot Plus | Muss installiert sein |
| Danalock | V3 Schloss + Danabridge V3 |
| PHP-Extension | cURL |

Ein aktives **Danalock-Konto** (my.danalock.com) ist erforderlich.

---

## Installation

1. ZIP-Paket herunterladen (Releases-Tab auf GitHub)
2. Joomla-Backend öffnen: **System → Installieren → Erweiterungen**
3. ZIP hochladen und installieren
4. Die Komponente erscheint automatisch als Untermenüpunkt unter **Komponenten → Rentalot Plus → Danalock PIN-Verwaltung**

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

Danalock-App öffnen → Schloss antippen → Zahnrad-Symbol → **Geräteinformationen**  
Format: `06:cd:42:45:44:ff`

### Mehrere Häuser konfigurieren

Im Feld „Haus → Schloss Zuordnung" eine Zeile pro Haus eintragen:
```
1:06:cd:42:45:44:ff
2:06:cd:42:45:44:aa
```
Die `unit_id` entspricht der ID in der Rentalot-Plus-Tabelle `jos_rentalot_plus_bookings`.

---

## Nutzung

### Buchungen & PINs

Zeigt alle Buchungen mit `state = 0` (Reserviert) oder `state = 1` (Gebucht) ab dem heutigen Tag.

**PIN vergeben:**
1. Auf **✏️ PIN** klicken
2. PIN-Slot wählen (1–20), PIN eingeben oder generieren lassen
3. Check-in/Check-out Zeiten anpassen falls nötig
4. **Speichern & an Schloss senden** → dauert ca. 10 Sekunden (Bridge-API ist asynchron)

**PIN löschen:** 🗑-Button neben dem PIN-Button

### Alle PIN-Slots

Liest alle 20 Slots direkt vom Schloss aus (dauert ca. 10 Sekunden).  
Zeigt zusätzlich den zugeordneten Gast und die Zeiten aus der lokalen Datenbank.

> **Hinweis:** Die Danalock Bridge API gibt keine Zeitfenster-Informationen zurück –
> nur Slot-Nummer, Aktiv/Frei und PIN-Ziffern. Die Zeiten kommen aus unserer DB.

### Debug-Log

Unter **Optionen → Debug & Logging** aktivierbar.  
Protokolliert alle API-Aufrufe, Job-IDs und Poll-Ergebnisse in der Datenbank.
Nur zur Fehlersuche aktivieren.

---

## Datenbank

Die Komponente legt zwei eigene Tabellen an:

### `jos_rentalot_plus_danalock_pins`
Speichert die Zuordnung von Buchung → PIN-Slot mit Zeiten.

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

### API-Flow

Die Danalock Bridge-API ist **asynchron**:

```
1. POST /bridge/v1/execute  →  {"id": "job-uuid"}
2. 7 Sekunden warten
3. POST /bridge/v1/poll     →  Ergebnis mit afi_status
```

Dies ist das normale Verhalten – jeder Befehl dauert ca. 10 Sekunden.

### Authentifizierung

OAuth2 Password Grant gegen `api.danalock.com`:
- `client_id`: `danalock-web`
- Token wird in der Joomla-Session gecacht (1 Stunde)

### Zeitbasierte PINs

Das Setzen zeitbasierter PINs ist technisch möglich (die Danalock-Hardware unterstützt es),
wird aber nicht direkt über diese Komponente gesteuert. Stattdessen werden die Zeiten
in der lokalen DB gespeichert und könnten über einen Joomla Scheduled Task
automatisch aktiviert/deaktiviert werden.

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

---

## Changelog

### 1.0.0
- Erstveröffentlichung
- Buchungsübersicht mit PIN-Verwaltung
- Direkte Bridge-API Integration
- Debug-Logging
- Mehrhaus-Unterstützung

---

*Entwickelt als Ergänzung zu [Rentalot Plus](https://github.com/SvenPausH/Rentalotplus)*
