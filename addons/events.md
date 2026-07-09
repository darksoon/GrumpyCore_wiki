# GrumpyCore — Events-Modul

## 1. Einleitung

Erlaubt das Planen von Terminen/Events mit Startzeit, optionalem Teilnehmerlimit und RSVP-Buttons (Zusagen/Vielleicht/Absagen). Startet ein Event, pingt der Bot automatisch alle Nutzer mit Zusage. Addon-Key: `events`.

## 2. Wie es funktioniert

### Event erstellen

Beim Erstellen wird eine Embed-Nachricht mit RSVP-Buttons im Zielkanal (Text- oder Ankündigungskanal) gepostet; die Nachrichten-ID wird für spätere Updates gespeichert. Optional lässt sich ein **Teilnehmerlimit** (`max-participants`, 1–1000) setzen — ohne Angabe (bzw. 0) ist die Teilnahme unbegrenzt.

### RSVP-Buttons

Drei Buttons unter der Event-Embed: **Zusagen** (✅), **Vielleicht** (❔), **Absagen** (❌). Jeder Klick aktualisiert die gespeicherte RSVP des Nutzers und rendert die öffentliche Embed neu (Listen der Zusagen/Vielleicht/Absagen).

**Teilnehmerlimit-Prüfung**: Ist ein Limit gesetzt und bereits erreicht, wird eine neue "Zusage" abgelehnt (`blocked`), statt das Event zu überbuchen. Die Zählung + der Upsert der RSVP laufen in einer einzigen Datenbank-Transaktion, damit zwei Nutzer, die gleichzeitig auf den letzten freien Platz klicken, nicht beide die Prüfung bestehen — die zweite Transaktion sieht bereits den committeten Schreibvorgang der ersten.

Ist ein Event bereits gestartet oder storniert, werden RSVP-Klicks mit einer entsprechenden Fehlermeldung abgewiesen; die Buttons werden beim Start zusätzlich deaktiviert.

### Runner (Event-Start)

Prüft alle **60 Sekunden** die DB auf Events, deren Startzeit erreicht ist. Wie im Announcements-Modul verhindert ein atomares Claiming (Compare-and-Set auf `started: false`) einen doppelten Start-Ping, falls sich zwei Tick-Durchläufe überlappen.

Beim Start:
1. Die Event-Embed wird aktualisiert und die RSVP-Buttons deaktiviert.
2. Alle Nutzer mit Zusage ("Yes") werden im Kanal gepingt.

**Batching der Pings**: Discord begrenzt `allowed_mentions.users` hart auf **100 Einträge pro Nachricht** — eine größere Liste führt nicht zu einem Fehler, sondern pingt stillschweigend niemanden über das Limit hinaus. Der Runner teilt große Zusage-Listen deshalb in Gruppen von je 100 auf und verschickt bei Bedarf mehrere Nachrichten, damit wirklich alle gepingt werden.

### Aufbewahrung / Löschung

Bereits gestartete Events werden nach **30 Tagen** automatisch gelöscht (inkl. Cascade-Löschung aller zugehörigen RSVPs). Das übernimmt der zentrale Wartungs-Runner (`modules/maintenance/cleanup.ts`, läuft täglich) — Details siehe die allgemeine Notiz zur Datenaufbewahrung.

## 3. Slash-Commands

Nur auf einem Server verfügbar (nicht in DMs).

### `/event create <title> <in> [description] [max-participants] [channel]`

| Option | Beschreibung |
|---|---|
| `title` | Titel des Events (max. 256 Zeichen) |
| `in` | Zeit bis zum Start, z.B. `2h`, `3d`, `1d12h` |
| `description` | Details (optional, max. 1000 Zeichen) |
| `max-participants` | Limit für Zusagen, 1–1000 (0/leer = unbegrenzt) |
| `channel` | Zielkanal (Standard: aktueller Kanal); muss Text- oder Ankündigungskanal sein |

### `/event list`

Zeigt alle kommenden (noch nicht gestarteten) Events des Servers (Titel, Kanal, relative Startzeit). Ephemer.

### `/event cancel <id>`

Storniert ein Event per ID (Autocomplete auf kommende Events) und löscht die zugehörige Event-Nachricht im Kanal. Erlaubt nur dem **Gastgeber** (Ersteller) oder Nutzern mit `Server verwalten`.

## 4. Berechtigungen

`/event` ist vollständig auf die Berechtigung **`Server verwalten`** (`ManageGuild`) bzw. Admin beschränkt — sowohl `create` als auch `list` und `cancel` (bewusste Design-Entscheidung, nicht nur `create`). Wie bei `/announce` wird dies zusätzlich zur Discord-seitigen Default-Berechtigung zur Laufzeit im Command geprüft, da Server-Admins die Default-Berechtigung pro Rolle/Kanal überschreiben können.

Beim Stornieren gilt abweichend: Gastgeber **oder** `Server verwalten` reicht aus.
