# GrumpyCore — ServerStats-Modul

## 1. Einleitung

Hängt "lebende" Zähler an Voice- oder Text-Kanäle: Der Kanalname wird periodisch umbenannt, um z.B. die aktuelle Mitgliederzahl oder die Anzahl der Boosts anzuzeigen (`👥 Members: 1234`). Reines DB-Modul, keine YAML-Konfiguration nötig — die Zähler werden per Slash-Command angelegt.

## 2. Wie es funktioniert

### Zähler-Typen

Es gibt vier Zähler-Typen:

| Typ | Bedeutung | Quelle |
|---|---|---|
| `members` | Gesamtzahl aller Mitglieder | `guild.memberCount` |
| `humans` | Mitglieder ohne Bots | gefilterter Member-Cache |
| `bots` | Nur Bot-Accounts | gefilterter Member-Cache |
| `boosts` | Anzahl Server-Boosts | `guild.premiumSubscriptionCount` |

**Bewusst kein `online`-Typ**: Ein Zähler für aktuell online befindliche Mitglieder würde den privilegierten `GuildPresences`-Intent voraussetzen, den der Bot nicht anfordert. Diese Funktion existiert daher absichtlich nicht.

Für `humans`/`bots` hängt die Genauigkeit vom gefüllten Member-Cache ab — der 10-Minuten-Runner holt sich vorher immer den vollständigen Mitgliederbestand (`guild.members.fetch()`), bevor er zählt.

### Kanalname & Template

Jeder Zähler hat ein Namens-Template mit dem Platzhalter `{count}`. Ohne eigene Angabe gelten diese Standard-Templates:

- `members`: `👥 Members: {count}`
- `humans`: `🙋 Humans: {count}`
- `bots`: `🤖 Bots: {count}`
- `boosts`: `💎 Boosts: {count}`

Der fertige Name wird auf maximal 100 Zeichen gekürzt (Discord-Limit für Kanalnamen).

### Der 10-Minuten-Update-Runner

Discord limitiert Kanal-Umbenennungen auf **2 pro 10 Minuten pro Kanal**. Der interne Runner läuft deshalb exakt in diesem Takt (`CHECK_INTERVAL_MS = 10 * 60_000`) — jeder Tick darf also bedenkenlos versuchen umzubenennen, ohne das Rate-Limit zu reißen. Der erste Durchlauf passiert sofort beim Bot-Start.

Ablauf pro Tick:
1. Alle Zähler werden aus der DB geladen und nach Server gruppiert (Member-Fetch passiert höchstens einmal pro Server pro Tick).
2. Für jeden Zähler wird der aktuelle Wert berechnet. **Hat sich der Wert seit dem letzten Mal nicht geändert, wird der Rename-API-Call komplett übersprungen** — spart unnötige Discord-API-Aufrufe.
3. Ist der Zielname bereits gesetzt (z.B. durch manuelle Umbenennung mit passendem Inhalt), wird nur der gespeicherte `lastValue` aktualisiert, kein API-Call.
4. Andernfalls wird der Kanal per `setName()` umbenannt und der neue Wert gespeichert. Schlägt das fehl (z.B. durch fremdes Rate-Limit oder fehlende Rechte), wird nur gewarnt geloggt — der nächste Tick versucht es erneut.

### Auto-Cleanup verwaister Zähler

Fehlt ein Kanal im Cache, wird nicht sofort angenommen, dass er gelöscht wurde (das könnte auch ein reiner Cache-Miss sein). Stattdessen wird der Kanal per `guild.channels.fetch()` real nachgeprüft. Bestätigt sich, dass der Kanal tatsächlich nicht mehr existiert, wird der zugehörige Zähler-Datensatz automatisch aus der DB gelöscht und geloggt — ohne dieses Verhalten müsste ein Admin den verwaisten Eintrag manuell per `/serverstats remove` entfernen.

Hat ein gefundener Kanal einen unerwarteten Typ (weder Voice noch Text), wird der Zähler übersprungen, aber nicht gelöscht.

### Sofort-Update bei `/serverstats add`

Damit Admins nicht bis zu 10 Minuten auf das erste sichtbare Ergebnis warten müssen, versucht `/serverstats add` direkt nach dem Anlegen eine sofortige Umbenennung. Schlägt dieser erste Versuch fehl, wird der Zähler trotzdem angelegt — der reguläre Runner holt die Umbenennung beim nächsten Tick nach.

### Limits

- Max. **10 Zähler pro Server**.
- Max. **1 Zähler pro Kanal** (ein Kanal kann nicht doppelt belegt werden).

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/serverstats add <channel> <type> [template]`

| Option | Beschreibung |
|---|---|
| `channel` | Voice- oder Text-Kanal, dessen Name zum Zähler wird |
| `type` | `members`, `humans`, `bots` oder `boosts` |
| `template` | Optional, eigenes Namens-Template mit `{count}` als Platzhalter (max. 90 Zeichen); ohne Angabe gilt der Standard-Text des jeweiligen Typs |

Antwort ist ephemer. Meldet ehrlich, falls die erste Umbenennung fehlschlägt (Zähler bleibt trotzdem angelegt und wird beim nächsten Sync nachgeholt).

### `/serverstats remove <channel>`

| Option | Beschreibung |
|---|---|
| `channel` | Kanal, dessen Zähler entfernt werden soll |

Löst nur die Verknüpfung, der Kanal selbst bleibt bestehen. Ephemer.

### `/serverstats list`

Listet alle konfigurierten Zähler des Servers (Kanal, Typ, Template). Ephemer.

## 4. Berechtigungen

Alle Unterbefehle erfordern `ManageGuild` (Server verwalten). Kein separates Rollen-System — reine Discord-Berechtigung.
