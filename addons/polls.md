# GrumpyCore — Polls-Modul

## 1. Einleitung

Live-Abstimmungen mit Button-Voting. Ergebnis-Embed aktualisiert sich in Echtzeit, Abstimmungen enden automatisch nach Ablauf oder manuell per Beenden-Button/Command. Ergebnis als Text-Balkendiagramm.

> Hinweis: Kein Bar-Chart-PNG bei Poll-Ende im main-Branch — die Ergebnisdarstellung ist ausschließlich ein Text-Balken (Blockzeichen), kein generiertes Bild. Ein PNG-Chart-Feature existiert nur in einem separaten, nicht gemergten Feature-Branch.

## 2. Wie es funktioniert

### 2.1 Erstellung

Postet zuerst ein Platzhalter-Embed ohne Buttons (um eine Message-ID zu bekommen), legt dann die DB-Zeile an (`id = messageId`) und editiert das Embed um die Vote-Buttons. Schlägt DB-Insert/Edit fehl, wird die Platzhalter-Nachricht gelöscht. Nummerierte Buttons (1️⃣–🔟), max. 10 Optionen.

### 2.2 Voting

**Ein Vote pro User.** Klick auf Option: Upsert (Erstellung oder Änderung möglich). Klick auf dieselbe Option: Stimme wird zurückgezogen. Nach jedem Klick wird das Embed neu aufgebaut, Zählung per DB-`groupBy` (Performance). Vor dem Rendern wird der `ended`-Status frisch gelesen, damit ein später eintreffender Vote keine bereits geschlossene Poll wiederbelebt.

### 2.3 Auto-End

Runner startet beim Boot (5s Delay, dann alle 60s), sofern `addons.polls !== false`. Schließt alle Polls mit `endAt <= jetzt`. Ist die Guild nicht im Cache, wird trotzdem als beendet markiert (ohne Message-Edit).

### 2.4 Manuelles Beenden

Über `/poll end <message-id>` oder den roten Beenden-Button — beide rufen dieselbe Funktion auf, die per atomarem Update sicherstellt, dass nur ein gleichzeitiger Aufruf (Auto-End vs. Button) die Poll tatsächlich auswertet.

### 2.5 Ergebnis-Darstellung

Balken aus Blockzeichen, z.B. `██████░░░░ 60% (6)`. Embed wird grau, Buttons entfernt.

## 3. Slash-Commands

### `/poll create`

| Option | Pflicht | Grenzen |
|---|---|---|
| `question` | ja | max. 256 Zeichen |
| `option1`/`option2` | ja | je max. 80 Zeichen |
| `option3`–`option10` | nein | je max. 80 Zeichen |
| `channel` | nein | Standard: aktueller Kanal |
| `duration` | nein | 0–168 Stunden, 0 = unbegrenzt (falls `maxDurationHours: 0`) |

Berechtigung: `Manage Messages`. Ist `maxDurationHours > 0` konfiguriert, wird `duration: 0` automatisch auf `maxDurationHours` gemappt (nicht als unbegrenzt interpretiert); Werte werden gedeckelt.

### `/poll end <message-id>`

Berechtigung: `Manage Messages`.

## 4. Konfiguration (`configs/modules/polls.yml`)

Kein `.example.yml` im Repo.

```yaml
enabled: true
maxDurationHours: 24     # 0-168, 0 = keine Obergrenze
allowMultiVote: false    # im Schema vorhanden, aktuell NICHT ausgewertet — immer 1 Vote/User
```

Zusätzlich global:

```yaml
addons:
  polls: true    # false deaktiviert Commands, Listener UND den Auto-End-Runner
```

## 5. Berechtigungen

| Aktion | Berechtigung |
|---|---|
| `/poll create` | `Manage Messages` |
| `/poll end` | `Manage Messages` |
| Beenden-Button | `Manage Messages` (serverseitig geprüft) |
| Vote-Buttons | Jeder |

Keine modulspezifische Staff-Rollen-Konfiguration — ausschließlich native Discord-Permission.
