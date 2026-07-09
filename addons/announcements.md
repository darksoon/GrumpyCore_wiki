# GrumpyCore — Announcements-Modul

## 1. Einleitung

Erlaubt es, Ankündigungen für einen späteren Zeitpunkt zu planen — einmalig oder wiederkehrend (täglich, wöchentlich, monatlich). Der Bot postet die Nachricht selbständig im gewählten Kanal, optional mit Rollen-Ping. Addon-Key: `announcements`.

## 2. Wie es funktioniert

### Runner

Prüft alle **60 Sekunden** die DB auf fällige Announcements. Ein Re-Entrancy-Guard verhindert, dass ein neuer Tick startet, während der vorherige (z.B. wegen eines langsamen Sends) noch läuft.

### Atomares Claiming (kein Doppelversand)

`claimDue()` liest fällige Announcements und aktualisiert sie in **derselben** DB-Operation, die sie beansprucht (Compare-and-Set auf den exakten `scheduledFor`-Wert der Zeile). Überlappen sich zwei Tick-Durchläufe — etwa weil ein Sende-Vorgang den ersten Tick über den nächsten geplanten Zeitpunkt hinaus blockiert —, trifft der zweite Versuch für dieselbe Zeile 0 Treffer, da `scheduledFor` bereits fortgeschrieben wurde. So kann eine Ankündigung nicht doppelt versendet werden. Der eigentliche Versand erfolgt danach; schlägt er fehl, wird der Claim **nicht** zurückgerollt (kein Retry-Loop bei dauerhaft kaputtem Kanal).

### Wiederholung (`recurring`)

Optionen: `daily`, `weekly`, `monthly` (Standard: einmalig, kein Feld).

- **Daily**: +1 Tag (UTC).
- **Weekly**: +7 Tage (UTC).
- **Monthly**: +1 Monat (UTC), mit **Tages-Clamping**: Ein naives `setUTCMonth(+1)` würde bei Tagen, die im Zielmonat nicht existieren (z.B. 31. Januar → "31. Februar"), in den übernächsten Monat überlaufen und dabei einen Monat überspringen — der Zeitplan würde dauerhaft driften. Stattdessen wird auf den tatsächlichen letzten Tag des Zielmonats begrenzt (z.B. 31. Jan. → 28./29. Feb.).

### Verhalten nach Ausfallzeit (kein Nachhol-Spam)

Nach einem Neustart oder einer Downtime, die länger als ein Intervall dauerte, würde ein einzelner `+1`-Schritt `scheduledFor` weiterhin in der Vergangenheit belassen — die Ankündigung würde beim nächsten Tick sofort wieder als fällig erkannt und in einer engen Schleife wiederholt nachgeholt ("catch-up spam"). Stattdessen springt `nextOccurrence()` direkt zum nächsten **zukünftigen** Termin: Es wird höchstens einmal (für diesen Tick) gesendet, nicht einmal pro verpasstem Intervall.

### Aufbewahrung / Löschung

Abgeschlossene (einmalig, bereits gesendet) oder stornierte Announcements (`active: false`) werden nach **90 Tagen** automatisch gelöscht. Das übernimmt nicht dieses Modul selbst, sondern der zentrale Wartungs-Runner (`modules/maintenance/cleanup.ts`, läuft täglich) — Details siehe die allgemeine Notiz zur Datenaufbewahrung. Wiederkehrende Announcements bleiben dauerhaft `active: true` und werden nie automatisch entfernt, solange sie nicht storniert wurden.

### Abgrenzung zu News 2.0

Der automatische Crosspost von News-Beiträgen in `GuildAnnouncement`-Kanälen (Discord "Ankündigungskanäle", die andere Server folgen können) ist **nicht** Teil dieses Moduls, sondern im News-Modul implementiert (`modules/news/listeners/newsDM.ts`, `posted.crosspost()`). Beide Module können denselben Kanaltyp (`GuildAnnouncement`) als Ziel nutzen, sind aber unabhängige Funktionen: `/announce` plant zeitgesteuerte, ggf. wiederkehrende Bot-Nachrichten; News 2.0 verteilt einmalige redaktionelle Beiträge automatisch an folgende Server.

## 3. Slash-Commands

Nur auf einem Server verfügbar (nicht in DMs).

### `/announce create <channel> <in> <message> [recurring] [ping-role]`

| Option | Beschreibung |
|---|---|
| `channel` | Zielkanal (Text- oder Ankündigungskanal) |
| `in` | Zeit bis zum ersten Versand, z.B. `30m`, `2h`, `1d` |
| `message` | Ankündigungstext (max. 1800 Zeichen) |
| `recurring` | Wiederholung: `Einmalig` (Standard), `Täglich`, `Wöchentlich`, `Monatlich` |
| `ping-role` | Rolle, die zusätzlich erwähnt wird (optional) |

Fehlschlag, wenn bereits **25 aktive Announcements** auf dem Server existieren.

### `/announce list`

Zeigt alle aktiven Announcements des Servers (Kanal, nächster Termin relativ, Wiederholung, Textvorschau). Ephemer.

### `/announce cancel <id>`

Storniert eine Announcement per ID (Autocomplete auf aktive Announcements).

## 4. Berechtigungen

Erfordert die Berechtigung **`Server verwalten`** (`ManageGuild`) bzw. Admin. Wird sowohl über `setDefaultMemberPermissions` (Discord-seitiger Default) als auch zur Laufzeit im Command selbst geprüft — als Absicherung, falls ein Server-Admin die Default-Berechtigung über die Discord-Integrationseinstellungen pro Rolle/Kanal überschrieben hat.
