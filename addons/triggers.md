# GrumpyCore — Text-Trigger ("Tags")

## 1. Einleitung

Ein Text-Trigger ist eine automatische Antwort, die von selbst losgeht, sobald jemand einen bestimmten Text in
den Chat schreibt — **ohne** dass jemand extra einen Befehl eintippen muss. Das ist der große Unterschied zu
[Custom Commands](customcmd.md) (`/cmd name`): dort muss ein User aktiv `/cmd irgendwas` schreiben. Ein Trigger
dagegen feuert von allein, wenn sein Stichwort irgendwo in einer normalen Chat-Nachricht auftaucht (oder exakt
der ganzen Nachricht entspricht, je nach Einstellung).

**Typisches Beispiel:** Jemand schreibt im Chat einfach "wie joine ich dem discord server" — ein Trigger mit
dem Stichwort `discord ip` (Match-Typ `contains`) würde darauf **nicht** anspringen (der genaue Text kommt nicht
vor), aber ein Trigger auf `serverip` würde bei jeder Nachricht anspringen, die "serverip" irgendwo enthält, und
könnte automatisch die Minecraft-Server-IP posten.

Das Modul ist **DB-only** (keine YAML-Konfiguration), pro Guild getrennt, maximal **50 Trigger pro Server**.

## 2. Schritt-für-Schritt: Einen neuen Trigger anlegen

**Beispiel-Ziel:** Immer wenn jemand "serverip" schreibt, soll der Bot automatisch die Minecraft-Server-Adresse
antworten.

1. Slash-Befehl eintippen: `/trigger add`
2. Discord zeigt die Optionen zum Ausfüllen:
   - **keyword**: `serverip` — der Text, auf den reagiert werden soll
   - **match-type**: `contains` auswählen (reagiert, sobald "serverip" **irgendwo** in der Nachricht vorkommt —
     die Alternative `exact` würde nur greifen, wenn die GESAMTE Nachricht exakt nur "serverip" lautet, ohne
     ein einziges weiteres Zeichen davor oder danach)
   - **type**: `text` auswählen (die Alternative `embed` erlaubt eine formatierte Antwort mit Titel/Farbe, siehe
     unten)
   - **response**: `Die Server-IP lautet: play.minetechworld.de 🎮`
3. Absenden — fertig. Ab sofort antwortet der Bot automatisch, sobald "serverip" in einer Nachricht auftaucht.

**Zum Ausprobieren:** einfach im Chat "wie lautet die serverip?" schreiben — der Bot antwortet automatisch als
Reply auf diese Nachricht.

## 3. Match-Typen im Detail

| Match-Typ | Verhalten | Beispiel bei Stichwort `hallo` |
|---|---|---|
| `contains` | Reagiert, sobald das Stichwort **irgendwo** in der Nachricht vorkommt (Groß-/Kleinschreibung egal) | Reagiert bei "hallo zusammen", "sag mal hallo", "HALLO!!!" |
| `exact` | Reagiert nur, wenn die Nachricht **exakt und ausschließlich** aus dem Stichwort besteht (nach Trimmen von Leerzeichen) | Reagiert bei "hallo", NICHT bei "hallo zusammen" |

**Faustregel:** `contains` für Stichwörter, die typischerweise mitten in einem Satz vorkommen (z.B. "serverip",
"discord nitro"). `exact` für kurze, eigenständige Wörter, die man nicht versehentlich als Teil eines anderen
Satzes auslösen will (z.B. ein Trigger auf das einzelne Wort "hi" würde mit `contains` bei jedem Wort anspringen,
das "hi" enthält, wie "hier" oder "wichtig" — mit `exact` nur bei einer Nachricht, die wortwörtlich nur "hi"
lautet).

## 4. Antwort-Typen

### 4.1 `text`

Einfacher Text, max. 2000 Zeichen. Unterstützt dieselben Platzhalter wie Custom Commands (`%user_mention%`,
`%guild_name%` usw., siehe [Custom Commands, Abschnitt 2.5](customcmd.md#25-platzhalter-system)).

### 4.2 `embed`

Formatierte Antwort mit Titel, Beschreibung, Farbe — `response` muss valides JSON sein, genau wie bei Custom
Commands:
```json
{"title": "🎮 Server-Infos", "description": "IP: play.minetechworld.de\nVersion: 1.21", "color": "#57F287"}
```
Mindestens `title` oder `description` als Text erforderlich.

**Kein `role-toggle`-Typ verfügbar** (anders als bei Custom Commands) — das ist Absicht: ein Trigger, der bei
jeder passenden Nachricht automatisch eine Rolle vergibt oder entzieht, wäre ein echtes Sicherheitsrisiko
(theoretisch könnte jeder durch einfaches Chatten sich selbst Rollen verschaffen, ohne dass ein Mod das aktiv
freigegeben hat). Für Rollenvergabe gibt es stattdessen [Reaction Roles](reactionroles.md) oder
Custom-Command-`role-toggle`, die beide eine bewusste Nutzeraktion erfordern.

## 5. Cooldown

Jeder Trigger hat einen eigenen Cooldown (Standard: 15 Sekunden), damit er nicht bei jeder einzelnen Nachricht
in einem belebten Kanal erneut auslöst. Löst ein Trigger aus, feuert er innerhalb des Cooldown-Fensters **nicht
erneut** — egal wie oft das Stichwort in dieser Zeit noch geschrieben wird. Der Cooldown ist aktuell nur pro
Trigger (nicht pro Nutzer) und nicht per Slash-Command einstellbar.

## 6. Slash-Commands

### `/trigger add <keyword> <match-type> <type> <response>` (Berechtigung: `Manage Guild`)

| Option | Pflicht | Beschreibung |
|---|---|---|
| `keyword` | ja | Max. 200 Zeichen, wird intern klein geschrieben gespeichert (Groß-/Kleinschreibung beim Erkennen spielt keine Rolle) |
| `match-type` | ja | `contains` oder `exact` |
| `type` | ja | `text` oder `embed (JSON)` |
| `response` | ja | Inhalt je nach Typ, max. 2000 Zeichen |

Schlägt fehl, wenn schon ein Trigger mit demselben Stichwort existiert, oder wenn das Server-Limit von 50
Triggern erreicht ist.

### `/trigger remove <keyword>` (Berechtigung: `Manage Guild`)

Löscht einen Trigger endgültig. Autocomplete zeigt die vorhandenen Stichwörter zur Auswahl an.

### `/trigger list`

Ephemere, alphabetisch sortierte Liste aller Trigger mit Match-Typ, Antwort-Typ und Nutzungszähler. Für jeden
sichtbar, keine besondere Berechtigung nötig.

### `/trigger info <keyword>`

Ephemeres Detail-Embed: Match-Typ, Antwort-Typ, Nutzungszähler, Ersteller, vollständiger Antwort-Inhalt.

## 7. Berechtigungen — Zusammenfassung

| Aktion | Erforderliche Berechtigung |
|---|---|
| `/trigger add`/`remove` | `Manage Guild` |
| `/trigger list`/`info` | Jeder |
| Passives Auslösen im Chat | Jeder — kein Befehl nötig, reagiert automatisch |

Alle Trigger-Antworten nutzen `allowedMentions: { parse: [] }` — kein `@everyone`-Ping möglich, egal was im
Response-Text steht.

## 8. Modul aktivieren/deaktivieren

Über `addons.triggers: false` in `configs/config.yml` lässt sich das komplette Modul abschalten (dann feuert
kein einziger Trigger mehr, und `/trigger` verschwindet als Befehl). Standard: aktiviert.
