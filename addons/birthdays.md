# GrumpyCore — Birthdays-Modul

## 1. Einleitung

Erlaubt Mitgliedern, ihren Geburtstag (Tag+Monat, ohne Jahr) pro Server zu hinterlegen. An dem Tag postet der Bot automatisch eine Glückwunschnachricht und kann optional eine Geburtstags-Rolle vergeben, die am Folgetag wieder entzogen wird.

## 2. Wie es funktioniert

### Runner

Prüft **stündlich**, ob heute (UTC) jemand Geburtstag hat. Die stündliche (statt tägliche) Prüfung fängt Neustarts ab, die das Geburtstagsfenster sonst verpassen würden. Ein Pro-Jahr-Zähler verhindert doppelte Gratulationen.

### Ablauf pro Prüfung

1. Fällige Geburtstage ermitteln (heutiges Datum, dieses Jahr noch nicht gefeiert).
2. **Congrats-Post**: falls `channelId` gesetzt und Textkanal, wird die konfigurierte `message` gepostet (`{{user}}` → Erwähnung).
3. **Geburtstags-Rolle**: falls `roleId` gesetzt, wird sie vergeben.
4. Eintrag wird als "gratuliert" markiert — auch bei Fehlschlag (kein stündliches Retry-Spam bei kaputter Config).
5. **Rollenentzug am Folgetag**: separate Prüfung entfernt die Rolle bei allen, die dieses Jahr schon gratuliert wurden, deren Datum aber nicht mehr heute ist.

29. Februar wird akzeptiert, feuert aber nur in Schaltjahren.

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/birthday set <date>`

`date` im Format `DD.MM` (Trenner `.`/`-`/`/` erlaubt, z.B. `24.12`). Überschreibt einen bestehenden Eintrag.

### `/birthday remove`

Entfernt den eigenen Geburtstag.

### `/birthday upcoming`

Zeigt Geburtstage der nächsten 30 Tage auf dem Server, sortiert nach Nähe. Ephemer.

### `/birthday reload`

Lädt die Modul-Konfiguration neu. Berechtigung: `ManageGuild`.

## 4. Konfiguration (`configs/modules/birthdays.yml`)

Kein `.example.yml` im Repo — wird aus Defaults erzeugt.

```yaml
enabled: true
channelId: ""      # Textkanal für Glückwunschnachricht, leer = kein Post
roleId: ""         # Geburtstags-Rolle, leer = keine Rolle
message: "🎉 Happy Birthday {{user}}! 🎂"
```

| Feld | Default | Beschreibung |
|---|---|---|
| `enabled` | `true` | Modul an/aus |
| `channelId` | `""` | Ziel-Kanal, muss Textkanal sein |
| `roleId` | `""` | Geburtstags-Rolle |
| `message` | siehe oben | Platzhalter `{{user}}` |

## 5. Berechtigungen

| Command | Berechtigung |
|---|---|
| `set`, `remove`, `upcoming` | Jedes Mitglied (nur eigener Geburtstag) |
| `reload` | `ManageGuild` |

Kein Command zum Setzen/Löschen fremder Geburtstage. Rollenvergabe/-entzug erfordert Bot-Hierarchie über der konfigurierten Rolle — ein Fehlschlag wird nur geloggt.
