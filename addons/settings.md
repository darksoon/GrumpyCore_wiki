# GrumpyCore — Settings-Modul

## 1. Einleitung

Zentrales Konfigurations-Interface direkt aus Discord: Module ein-/ausschalten und Kanal-IDs setzen, ohne manuell in `config.yml` zu editieren. Schreibt die Änderungen direkt in die Konfigurationsdatei — sie bleibt dauerhaft die Quelle der Wahrheit, `/settings` ist nur ein komfortabler Editor dafür.

## 2. Wie es funktioniert

### Schreibt config.yml direkt, Format-erhaltend

Jede Änderung wird über `patchMainConfig()` in `configs/config.yml` geschrieben. Dabei kommt die YAML-**Document API** (nicht der einfache Parser) zum Einsatz — Kommentare, Formatierung und Reihenfolge der bestehenden Datei bleiben dabei vollständig erhalten. Es wird also nicht die ganze Datei aus dem geparsten Objekt neu generiert, sondern nur der betroffene Pfad (`addons.<key>` bzw. `channels.<key>`) gezielt gepatcht.

Der Schreibvorgang läuft **atomar**: Zuerst wird in eine temporäre Datei (`config.yml.tmp-<pid>-<timestamp>`) geschrieben, anschließend per `rename()` über die Originaldatei gelegt. Ein Crash oder Stromausfall mitten im Schreibvorgang kann `config.yml` dadurch nie in einem halb-geschriebenen, korrupten Zustand hinterlassen — Leser sehen immer entweder die alte oder die neue vollständige Datei.

Schlägt das Schreiben fehl (z.B. Dateisystem-Fehler), wird kein Absturz riskiert — der Befehl fängt den Fehler ab und meldet ihn dem Admin als Discord-Antwort.

### `/settings addon` — Neustart erforderlich

Schaltet einen Eintrag unter `addons.*` in `config.yml` um. **Wichtig**: Module werden bei GrumpyCore nur beim Bot-Start registriert bzw. deregistriert. Ein Addon-Toggle über `/settings addon` wirkt sich also **nicht sofort** aus — das eigentliche Laden/Entladen des Moduls (Commands, Listener, Runner) passiert erst beim nächsten Neustart des Bots. Der Befehl weist explizit darauf hin, statt fälschlich eine sofortige Wirkung zu versprechen.

Zur Absicherung wird das übergebene `module` gegen eine feste Liste bekannter Addon-Keys geprüft (manuell synchron gehalten mit dem `AddonsSchema` in `src/lib/config.ts`). Ein Tippfehler kann so keinen beliebigen neuen Top-Level-Key unter `addons.*` in die Datei schreiben. Die gleiche Liste speist auch die Autocomplete-Vorschläge im Discord-Client.

Bekannte Addon-Keys (Stand Code): `welcome`, `mod`, `tickets`, `news`, `voice`, `help`, `customcmd`, `music`, `reactionroles`, `leveling`, `suggestions`, `polls`, `starboard`, `reminders`, `birthdays`, `rss`, `giveaways`, `serverstats`, `announcements`, `events`. (`settings` selbst ist im `AddonsSchema` ebenfalls togglebar, taucht aber nicht in der `/settings`-Befehlsliste auf.)

Zusätzlich zum Datei-Patch wird der In-Memory-Wert (`container.config.addons`) im laufenden Prozess sofort aktualisiert — das sorgt dafür, dass z.B. ein direkt anschließendes `/settings show` im selben Prozess den neuen Stand zeigt. Das ändert aber nichts daran, dass die Modul-Registrierung selbst boot-only ist.

### `/settings channel` — sofort wirksam

Setzt eine Kanal-ID unter `channels.*`. Anders als beim Addon-Toggle wirkt diese Änderung **sofort**: Kanal-IDs werden an den jeweiligen Aufrufstellen live aus `container.config` gelesen, nicht nur beim Start eingelesen.

**Laufzeit-Validierung des Kanaltyps**: Da Discords `addChannelOption` bei der Befehlsregistrierung nur eine feste, für den ganzen Befehl geltende Liste erlaubter Kanaltypen akzeptiert (nicht abhängig vom gewählten `key`), akzeptiert die Option selbst Text-, Ankündigungs- und Voice-Kanäle. Die eigentliche Prüfung, ob der Typ zum gewählten `key` passt, passiert danach im Handler zur Laufzeit:

- Die Keys `voice-lobby` und `radio` verlangen zwingend einen **Voice-Kanal**.
- Alle anderen Keys (`welcome`, `verify`, `mod-log`, `alert`, `staff`, `support`, `ticket-log`, `news`, `help`, `suggestions`, `starboard`, `levelup`) verlangen einen **Text- oder Ankündigungskanal**.

Passt der Typ nicht, wird der Befehl mit einer entsprechenden Fehlermeldung abgelehnt, ohne etwas zu schreiben.

### `/settings show`

Zeigt eine Statusübersicht aller bekannten Addon-Keys mit 🟢 (aktiv) bzw. 🔴 (deaktiviert), basierend auf dem aktuellen In-Memory-Konfigurationsstand des Prozesses.

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/settings addon <module> <enabled>`

| Option | Beschreibung |
|---|---|
| `module` | Modulname (Autocomplete aus bekannten Addon-Keys) |
| `enabled` | `true`/`false` — an oder aus |

⚠️ Wirkt erst nach einem Bot-Neustart, da Module nur beim Start geladen/entladen werden.

### `/settings channel <key> <channel>`

| Option | Beschreibung |
|---|---|
| `key` | Welche Kanal-Einstellung (`welcome`, `verify`, `mod-log`, `alert`, `staff`, `support`, `ticket-log`, `news`, `voice-lobby`, `help`, `radio`, `suggestions`, `starboard`, `levelup`) |
| `channel` | Der Kanal (Text, Ankündigung oder Voice — muss zum gewählten `key` passen) |

Sofort wirksam. Bei falschem Kanaltyp für den gewählten `key` (z.B. Text-Kanal für `voice-lobby`) wird der Befehl mit Fehlermeldung abgelehnt.

### `/settings show`

Keine Optionen. Zeigt den Aktivierungsstatus aller Addons als Liste.

## 4. Berechtigungen

Alle Unterbefehle erfordern die Discord-Berechtigung **Administrator** (nicht nur `ManageGuild` wie bei anderen Modulen) — `/settings` greift direkt in die Kernkonfiguration des Bots ein.
