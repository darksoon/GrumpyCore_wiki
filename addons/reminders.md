# GrumpyCore — Reminders-Modul

## 1. Einleitung

Erlaubt Mitgliedern, sich selbst persönliche Erinnerungen zu setzen ("in X Zeit" + Freitext). Läuft die Zeit ab, versucht der Bot per DM zuzustellen; schlägt das fehl, wird im ursprünglichen Kanal gepostet. Reines DB-Modul, keine YAML-Konfiguration.

## 2. Wie es funktioniert

### Runner

Prüft alle **30 Sekunden** die DB auf fällige Erinnerungen. Erster Durchlauf sofort beim Start.

### Ablauf bei Fälligkeit

1. Embed wird gebaut (Titel, Text max. 4000 Zeichen, Footer mit ID).
2. Zustellversuch per **DM**.
3. **Fallback**: schlägt die DM fehl (DMs geschlossen o.ä.), wird stattdessen im **ursprünglichen Kanal** gepostet, mit Erwähnung.
4. Ist auch der Kanal nicht erreichbar, geht die Erinnerung verloren (nur geloggt).
5. Unabhängig vom Erfolg wird die Erinnerung danach gelöscht — **keine Wiederholungsversuche**.

### Limits

- Max. **25 ausstehende Erinnerungen pro Nutzer**.
- Max. **1 Jahr** im Voraus.
- Nachrichtentext: max. **500 Zeichen**.

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/remind add <in> <message>`

| Option | Beschreibung |
|---|---|
| `in` | Dauer, z.B. `30m`, `2h`, `1d12h` |
| `message` | Text (max. 500 Zeichen) |

**Zeit-Format**: Einheiten `s`/`m`/`h`/`d`, kombinierbar (`1d12h`), case-insensitive. Reine Zahl ohne Einheit = Minuten. Auf ganze Minuten gerundet.

### `/remind list`

Zeigt bis zu 10 eigene ausstehende Erinnerungen. Ephemer.

### `/remind cancel <id>`

Storniert eine eigene Erinnerung per ID.

## 4. Berechtigungen

Keine speziellen Discord-Berechtigungen — jedes Mitglied verwaltet ausschließlich seine eigenen Erinnerungen. Kein Admin-Zugriff auf fremde Erinnerungen.
