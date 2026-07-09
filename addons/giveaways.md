# GrumpyCore — Giveaway-Modul

## 1. Einleitung

Erlaubt Team-Mitgliedern, Giveaways in einem Kanal zu starten. Mitglieder nehmen über einen Button an der Giveaway-Nachricht teil. Nach Ablauf der Dauer werden automatisch Gewinner ausgelost und angekündigt; das Giveaway kann auch vorzeitig beendet oder nachträglich neu ausgelost werden. Addon-Key: `giveaways`.

## 2. Wie es funktioniert

### Teilnahme (Toggle-Button)

Die Giveaway-Nachricht enthält einen Button "Teilnehmen" (🎉). Ein Klick trägt den Nutzer ein; ein erneuter Klick trägt ihn wieder aus (echtes Toggle, keine Einwegteilnahme). Antwort erfolgt ephemer.

- Ist eine `required-role` gesetzt, muss der Nutzer diese Rolle besitzen — sonst Ablehnung mit Hinweis auf die benötigte Rolle.
- Ist das Giveaway bereits beendet, wird der Klick abgelehnt.
- Bei einem Datenbank-Konfliktfall (zwei gleichzeitige Klicks desselben Nutzers) wird der zweite Versuch defensiv als "bereits eingetragen" behandelt statt einen Fehler zu werfen.

### Runner

Ein Hintergrund-Timer prüft alle **30 Sekunden** die DB auf fällige Giveaways (`endAt <= jetzt`, `ended = false`). Erster Durchlauf sofort beim Start des Bots.

### Ablauf bei Fälligkeit (automatisches Ende)

1. Für jedes fällige Giveaway wird das **atomare** Ende ausgelöst (siehe unten).
2. Die Original-Nachricht wird editiert: Embed auf "beendet" (graue Farbe), Button deaktiviert.
3. Eine neue Nachricht mit den Gewinnern wird im Giveaway-Kanal gepostet (mit Erwähnung der Gewinner).
4. Ist der Kanal nicht mehr erreichbar, wird das übersprungen und nur geloggt — das Giveaway bleibt trotzdem als beendet markiert.

### Atomares Beenden & Auslosen

`endGiveawayAndDraw` setzt `ended = true` per bedingtem Update (`WHERE id = ... AND ended = false`). Nur wenn genau ein Datensatz betroffen war, wird ausgelost — andernfalls (0 betroffene Zeilen) gilt das Giveaway als bereits beendet und es passiert nichts. Das macht den automatischen Runner und ein manuelles `/giveaway end` race-sicher: Egal wer zuerst "gewinnt", es wird nie doppelt ausgelost.

Die Auslosung selbst zieht zufällig `winnerCount` Teilnehmer ohne Zurücklegen aus dem Teilnehmer-Pool. Gibt es weniger Teilnehmer als Gewinnplätze, gewinnen einfach alle vorhandenen Teilnehmer (oder niemand, falls der Pool leer ist).

### Reroll

`/giveaway reroll` funktioniert nur für bereits beendete Giveaways und zieht **erneut zufällig** aus demselben Teilnehmer-Pool. Vorherige Gewinner werden dabei **nicht ausgeschlossen** — sie können erneut gezogen werden (übliche Giveaway-Bot-Semantik). Das Giveaway wird zusätzlich als `rerolled` markiert.

### Liste & Autocomplete

- `/giveaway list` zeigt nur laufende (nicht beendete) Giveaways des Servers.
- Die `id`-Option bei `end` und `reroll` nutzt Autocomplete: `end` schlägt laufende Giveaways vor, `reroll` schlägt die zuletzt beendeten Giveaways vor (max. 25 Treffer, gefiltert nach Preis-Text oder ID).

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/giveaway start <duration> <prize> [winners] [required-role] [channel]`

| Option | Beschreibung |
|---|---|
| `duration` | Dauer, z.B. `30m`, `1h`, `3d` |
| `prize` | Preis-Beschreibung (max. 256 Zeichen) |
| `winners` | Anzahl Gewinner (1–20, Standard: 1) |
| `required-role` | Rolle, die zur Teilnahme benötigt wird (optional) |
| `channel` | Zielkanal (Text/Ankündigung), Standard: aktueller Kanal |

Postet die Giveaway-Nachricht mit Embed und Teilnahme-Button im Zielkanal.

### `/giveaway end <id>`

| Option | Beschreibung |
|---|---|
| `id` | Giveaway (Autocomplete, laufende Giveaways) |

Beendet ein laufendes Giveaway sofort und lost Gewinner aus (atomar, siehe oben). Schlägt fehl, falls bereits beendet.

### `/giveaway reroll <id>`

| Option | Beschreibung |
|---|---|
| `id` | Giveaway (Autocomplete, zuletzt beendete Giveaways) |

Lost für ein bereits beendetes Giveaway neue Gewinner aus derselben Teilnehmerliste. Schlägt fehl, falls das Giveaway noch läuft.

### `/giveaway list`

Zeigt bis zu allen laufenden Giveaways des Servers (Preis, Kanal, Restzeit, gekürzte ID). Ephemer.

## 4. Berechtigungen

Alle Unterbefehle erfordern standardmäßig **Server verwalten** (`ManageGuild`), serverseitig konfigurierbar über Discords Befehlsberechtigungen. Die Teilnahme per Button steht jedem Mitglied offen (ggf. eingeschränkt durch `required-role`).
