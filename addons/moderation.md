# GrumpyCore — Moderation-Modul

## 1. Einleitung

Das `mod`-Modul ist das zentrale Moderationssystem von GrumpyCore. Es bündelt:

- **Klassische Moderationsaktionen**: Warn, Kick, Ban, Tempban (befristeter Ban mit Auto-Unban), Mute (Timeout), Softban, Note, Unmute, Unban, Unwarn
- **Eskalationsleiter**: automatische Folgeaktion (Mute → Kick → Ban), sobald ein Mitglied eine konfigurierte Zahl aktiver Verwarnungen erreicht
- **Auto-Mod**: passiver Nachrichten-Scanner (Anti-Spam, Anti-Flood, Anti-Ad, Anti-Phishing inkl. URLhaus-Bedrohungsfeed, Anti-Caps, Anti-Repeat, Wortfilter)
- **Report-System**: `/report`-Befehl für normale Mitglieder + Button-Panel für Staff
- **Mod-Log**: automatisches Protokoll aller Aktionen in einem konfigurierten Textkanal
- **Message-Log**: zeigt den tatsächlichen Inhalt gelöschter/bearbeiteter Nachrichten (auch bei Selbstlöschung durch den Autor)
- **Punishment-History**: jede Aktion wird als `Punishment`-Datensatz gespeichert, abrufbar über `/mod history`
- **Auto-Unban-Runner**: Hintergrundprozess, der abgelaufene Tempbans automatisch aufhebt
- **`/mod nuke`**: Channel in Sekunden komplett zurücksetzen (löscht wirklich ALLE Nachrichten, auch uralte)

Aktivierung über `addons.mod` in `configs/config.yml` (Default `true`), Konfiguration über `configs/modules/mod.yml`.

## 2. Funktionsweise

### 2.1 Eskalations-Mechanik

Nach jedem `/mod warn` (und jedem per Report-Button vergebenen Warn durch einen Mod mit `Moderate Members`) wird die Eskalation geprüft:

1. Zahl der **aktiven** Warns wird gezählt.
2. Alle Regeln, deren `threshold` erreicht ist, werden ermittelt — die mit dem **höchsten** Threshold gewinnt.
3. **`mute`**: Timeout wird gesetzt. Wird nicht erneut ausgelöst, solange das Mitglied noch im Timeout ist. Aktive Warns werden NICHT zurückgesetzt (gestufte Leiter, z.B. Mute@3 → Kick@5 → Ban@7).
4. **`kick`/`ban`**: Aktion wird ausgeführt, danach werden alle aktiven Warns auf inaktiv gesetzt.
5. Die Discord-API-Aktion läuft zuerst; nur bei Erfolg wird ein Punishment-Eintrag angelegt (Moderator = `AUTOMOD`).
6. Ein Pro-Nutzer-Lock serialisiert parallele Warns, damit nicht doppelt eskaliert wird.

### 2.2 Hierarchie-Check

Vor jeder destruktiven Aktion geprüft:

| Prüfung | Ergebnis bei Verstoß |
|---|---|
| Executor = Target | Ablehnung — man kann sich nicht selbst als Ziel wählen |
| Target ist ein Bot | Ablehnung |
| Target ist Server-Owner | Ablehnung |
| Executor (kein Owner) hat niedrigere/gleiche Rollenposition wie Target | Ablehnung (Rollenhierarchie) |
| Bot hat niedrigere/gleiche Rollenposition wie Target | Ablehnung (Bot kann nicht moderieren) |

Der Server-Owner ist von der Rollenvergleichs-Regel ausgenommen. Der Bot-Hierarchie-Check gilt unabhängig davon auch für den Owner.

### 2.3 Report-Workflow

1. Mitglied ruft `/report user:<Ziel> reason:<Text> [proof:<Anhang>]` auf.
2. Guards: kein Selbst-Report, kein Bot-Report, keine Mitglieder mit `Manage Messages` meldbar.
3. Cooldown/Tageslimit pro Melder (In-Memory).
4. Ziel-Kanal: `channels.staff` → `mod.yml → reportChannelId` → `channels.mod-log`.
5. Embed mit Ziel, Melder, Grund, optionalem Bild-Beweis + Buttons: Deny / Warn / Timeout / Kick / Ban.
6. Klick-Berechtigungen:
   - **Basis-Gate**: `Moderate Members` **oder** Rolle aus `roles.support`.
   - **Pro-Aktion**: Deny/Warn → keine weitere Anforderung; Timeout → `Moderate Members`; Kick → `Kick Members`; Ban → `Ban Members`.
   - `Warn` per Button löst Eskalation nur aus, wenn der Klicker tatsächlich `Moderate Members` besitzt (verhindert, dass reine Support-Klicker über die Leiter einen Kick/Ban auslösen).
   - Timeout über den Button ist fest auf 10 Minuten gesetzt.

### 2.4 Anti-Nuke (`listeners/antiNuke.ts`)

Ein passiver Listener auf `GuildAuditLogEntryCreate`, unabhängig von der Auto-Mod-Konfiguration (nicht in `mod.yml` einstellbar, fest im Code). Zählt pro Verursacher innerhalb eines 10-Sekunden-Fensters:

| Aktion | Schwellenwert |
|---|---|
| Channel-Löschungen | 3 |
| Bans | 5 |
| Rollen-Löschungen | 3 |

Wird ein Schwellenwert erreicht:
1. Ein Alarm wird in `channels.alert` gepostet (optional mit Ping an `roles.alert-ping`).
2. Dem Verursacher werden **sofort alle Rollen entzogen** (`roles.set([])`, ein einziger API-Call statt vieler einzelner — vermeidet Rate-Limit-Stacking während eines aktiven Angriffs).
3. Der Server-Owner wird von der Rollen-Entfernung ausgenommen (API würde ohnehin fehlschlagen).
4. Eine 60-Sekunden-Cooldown verhindert wiederholtes Neutralisieren desselben Verursachers innerhalb eines zweiten Bursts.

Dieser Schutz ist **immer aktiv**, sofern `addons.mod: true` — es gibt keinen eigenen Ein-/Ausschalter dafür.

### 2.5 Auto-Unban-Runner

- Läuft alle 60 Sekunden, einmal sofort beim Start (holt verpasste Tempban-Abläufe nach).
- Sucht alle `Punishment`-Zeilen `type: 'ban'`, `active: true`, `expiresAt <= jetzt`.
- Discord-Fehler 10026 (Unknown Ban) wird als „bereits erledigt" behandelt; jeder andere Fehler ist transient (Zeile bleibt aktiv, nächster Versuch).
- Bei Erfolg: Ban-Zeile auf inaktiv, neuer `unban`-Eintrag, Mod-Log-Post.
- **Wichtig:** `/mod tempban` legt keinen eigenen Typ „tempban" an — es ist ein normaler `type: 'ban'` mit `expiresAt` gesetzt, nutzt also dieselbe Eskalations-/History-/Log-Darstellung wie ein Permanent-Ban.

### 2.6 Anti-Phishing — Schutz vor Fake-Nitro-Links & Scam-Seiten

Das ist der Check, der z.B. gefälschte "Kostenloses Discord Nitro"-Links erkennt, die man in Discord-Servern
ständig als Spam-DM oder in Chat-Nachrichten sieht (Muster: `dlscord-nitro.com`, `steam-community-gg.tk` usw.).
Läuft **immer als erster** Auto-Mod-Check (bevor Anti-Ad, Anti-Caps usw.), damit ein Phishing-Treffer nie von
einem schwächeren Check überdeckt wird.

Zwei unabhängige Erkennungswege, beide gleichzeitig aktiv (wenn beide `enabled`):

**a) Eingebaute Heuristik (`useBuiltinHeuristics`)** — rein lokal, kein Internetzugriff nötig:
- **Marken-Impersonation**: erkennt Domains, die `discord`, `nitro` oder `steam` als eigenständigen Namensteil
  enthalten (z.B. `dlscord-nitro.com`, `steam-community-gg.tk`), aber NICHT die echte, bekannte Domain sind
  (`discord.com`, `discord.gg`, `steamcommunity.com` usw. sind immer erlaubt). Ein Link wie `steamworks.dev`
  oder `mysteamgame.com` löst **nicht** aus, obwohl "steam" im Namen vorkommt — nur wenn "steam" als eigenes,
  durch Punkt/Bindestrich abgegrenztes Wort auftaucht.
- **Verdächtige TLD + Köder-Wort**: Domains mit den TLDs `.tk`/`.ml`/`.ga`/`.cf`/`.gq`/`.zip`/`.mov` lösen NUR
  aus, wenn die Nachricht zusätzlich ein Köder-Wort enthält (`free`, `gratis`, `nitro`, `gift`, `geschenk`,
  `airdrop`, `claim`, `giveaway`). Ein normaler `.tk`-Link ohne diese Wörter wird NICHT blockiert (zu viele
  False Positives sonst).

**b) URLhaus-Bedrohungsfeed (`useUrlhaus`)** — externe, ständig aktualisierte Liste bekannter Malware-/Phishing-URLs
von [abuse.ch](https://urlhaus.abuse.ch/), einem kostenlosen, seriösen Sicherheitsprojekt (kein API-Key nötig):
- Der Bot lädt die Liste **stündlich neu** (aktuell ca. 15.000 Einträge) und hält sie als lokalen Cache
  (`cache/urlhaus-urls.txt`) — dadurch ist der eigentliche Check pro Nachricht blitzschnell (reiner
  Speicher-Abgleich, kein Netzwerk-Aufruf während des Chattens).
  - **Wichtiger Unterschied zur eingebauten Heuristik**: URLhaus erkennt keine Marken-Namen, sondern exakt
  bekannte, bereits gemeldete bösartige URLs (Malware-Download-Links, Phishing-Seiten aus der ganzen Welt) —
  deckt also viel mehr ab als nur Nitro-/Steam-Fakes, dafür nur wenn die URL schon einmal gemeldet wurde.
- Fällt der Download mal aus (Netzwerkproblem), bleibt der zuletzt geladene Cache aktiv — es gibt also nie eine
  Lücke, in der der Schutz komplett aus ist.

**c) Admin-eigene Sperrliste (`customBlocklist`)** — feste Liste von Domains, die IMMER blockiert werden, egal
was die anderen beiden Checks sagen. Praktisch für Domains, die im eigenen Server schon mal Ärger gemacht haben.
Subdomains werden automatisch mit erfasst (ein Eintrag `bad-site.com` blockiert auch `sub.bad-site.com`).

**Konkretes Beispiel:** Postet ein Mitglied `Kostenloses Nitro hier: dlscord-glft.tk 🎁`, greift:
1. Marken-Impersonation erkennt `dlscord` (Tippfehler-Variante von "discord") → sofortiger Treffer, unabhängig
   von URLhaus oder dem Köder-Wort-Check.

Postet stattdessen jemand nur `dlscord-glft.tk` ohne erkennbaren Markennamen, aber die URL steht zufällig schon
in der URLhaus-Datenbank (weil sie schon anderswo als Malware-Verteiler gemeldet wurde), greift Check (b) —
selbst wenn Check (a) nichts findet.

**Was danach passiert**, steht in `action` (`warn`/`delete`/`timeout`/`nothing`) und `deleteMessages` — Standard
ist `timeout` für 60 Minuten + Nachricht löschen (das schärfste Auto-Mod-Preset überhaupt, weil Phishing-Links
am gefährlichsten sind).

### 2.7 Message-Log — Inhalt gelöschter/bearbeiteter Nachrichten sehen

Discords eigenes, natives Audit-Log (das, was `/mod` NICHT direkt zeigt, sondern was Discord selbst mitschreibt)
hat zwei große Lücken: es zeigt **nie den Nachrichteninhalt**, und es erfasst **nur Löschungen durch Moderatoren**
an fremden Nachrichten — löscht ein User seine eigene Nachricht selbst, taucht das dort gar nicht auf. Ebenso
werden Bearbeitungen (Edits) von Discord überhaupt nicht protokolliert.

Das Message-Log schließt diese Lücke: es postet bei **jeder** Löschung/Bearbeitung im Server (unabhängig davon,
wer sie ausgelöst hat) eine Nachricht mit dem tatsächlichen Inhalt in `channels.mod-log`:

- **Löschung**: Autor, Kanal, kompletter Nachrichtentext, Liste eventueller Anhänge (Bild-/Datei-URLs)
- **Bearbeitung**: Autor, Kanal, Link zur bearbeiteten Nachricht, Text **vorher** und **nachher** nebeneinander
- **Massenlöschung** (z.B. `/mod purge`): Kanal, Gesamtzahl, bis zu 10 Beispiel-Nachrichten mit Inhalt

**Wichtige Einschränkung** (kein Bug, sondern eine echte Discord-Grenze): Wurde eine Nachricht schon vor dem
letzten Bot-Neustart gepostet oder ist sie älter als die letzten ca. 200 Nachrichten im jeweiligen Kanal, hat
Discord dem Bot ihren Inhalt nie übermittelt — in diesem Fall wird bewusst **gar nichts** gepostet (lieber
stillschweigend nichts zeigen als eine leere "irgendwas wurde gelöscht"-Nachricht ohne Mehrwert).

Konfiguration in `configs/modules/mod.yml`:
```yaml
messageLog:
  enabled: true         # Modul komplett an/aus
  logEdits: true        # Bearbeitungen protokollieren
  logDeletes: true      # Löschungen protokollieren
  ignoreBots: true       # Bot-eigene Nachrichten (Ticket-Panels, Embeds, ...) NICHT protokollieren — reines Rauschen sonst
```

### 2.8 `/mod nuke` — Channel komplett zurücksetzen

`/mod clear` und `/mod purge` (siehe Befehlstabelle unten) können **keine Nachrichten löschen, die älter als 14
Tage sind** — das ist eine feste Grenze von Discords eigener API, kein Limit von GrumpyCore. Will man einen
Channel WIRKLICH komplett leeren (z.B. weil er vollgespammt oder eskaliert ist), reicht Bulk-Delete also nicht.

`/mod nuke` löst das anders: der Channel wird **geklont** (identischer Name, identische Rechte, identische
Position in der Kanalliste) und der alte Channel wird danach komplett gelöscht. Das ist sofort, betrifft
wirklich jede Nachricht unabhängig vom Alter, und danach postet der Bot ein Embed
("☢️ CHANNEL NUKED — Everybody who was here is gone. 💥") im frischen Channel.

**Achtung, das ist nicht rückgängig zu machen** — alle Nachrichten im Channel sind danach unwiederbringlich weg
(auch die, die im Message-Log oder Mod-Log referenziert wurden, zeigen dann nur noch tote Links). Deswegen
braucht `/mod nuke` extra `Manage Channels`, nicht nur `Manage Messages` wie `/mod clear`/`purge` — ein
gewöhnlicher Nachrichten-Moderator soll das nicht versehentlich auslösen können.

Optional kann man in `configs/modules/mod.yml` ein eigenes GIF/Bild für die Nuke-Nachricht hinterlegen:
```yaml
nuke:
  gifUrl: ""   # z.B. "https://example.com/mushroom-cloud.gif" — leer = kein Bild, nur Text
```
(Absichtlich standardmäßig leer statt mit einem fest eingebauten Link — ein von uns nicht kontrollierter
externer Link könnte irgendwann offline gehen und dann würde das Embed kaputt aussehen.)

## 3. Slash-Commands

### `/mod` (Basis-Berechtigung: `Moderate Members`)

| Subcommand | Optionen | Zusätzliche Berechtigung |
|---|---|---|
| `warn` | `user`, `reason` (beide required) | — |
| `kick` | `user` (required), `reason` (optional) | `Kick Members` |
| `ban` | `user` (required), `reason`, `delete-days` (0–7) optional | `Ban Members` |
| `tempban` | `user`, `duration` (required, z.B. `12h`, `3d`), `reason`, `delete-days` optional | `Ban Members` |
| `mute` | `user`, `duration` (required, max. 28 Tage), `reason` optional | — |
| `unmute` | `user` (required) | — |
| `unban` | `user-id` (required), `reason` optional | `Ban Members` |
| `unwarn` | `id` (required, Punishment-ID) | — |
| `history` | `user` (required) | — |
| `clear` | `count` (required, 1–100) | `Manage Messages` |
| `purge` | `scan` (1–200 required), `user`/`bots`/`contains`/`attachments` optional | `Manage Messages` |
| `slowmode` | `seconds` (0–21600 required), `channel` optional | `Manage Channels` |
| `nuke` | — (wirkt auf den aktuellen Kanal) | `Manage Channels` |
| `note` | `user`, `note` (beide required) | — |
| `softban` | `user` (required), `reason`, `delete-days` (Default 1) optional | `Ban Members` |
| `reload` | — | `Manage Guild` |
| `modlog-search` | `user`/`moderator`/`type` (warn/kick/ban/softban/mute/note)/`days` (1–3650) alle optional | — |
| `modlog-export` | `user`/`type`/`days` alle optional | — |

Details: Ban/Tempban/Softban/Unban führen die Discord-API-Aktion immer zuerst aus — schlägt sie fehl, wird kein Punishment-Eintrag angelegt. `note` erzeugt einen Eintrag ohne DM/Eskalationszählung. `modlog-search` zeigt bis zu 25 Treffer als Embed (🟢/⚫ aktiv/inaktiv, Moderator, Zeitstempel), Footer nennt Trefferzahl vs. Gesamtzahl. `modlog-export` liefert dieselben Filter (außer `moderator`) als CSV-Anhang (`id,type,userId,moderatorId,reason,active,createdAt,expiresAt`) — Grund-Feld ist gegen CSV-Formel-Injection (`=`, `+`, `-`, `@`, Tab, CR am Zeilenanfang) durch führendes `'` abgesichert.

### `/report` (jeder Nutzer)

| Option | Pflicht |
|---|---|
| `user` | ja |
| `reason` | ja |
| `proof` (Anhang) | nein — nur Bilder werden eingebettet |

## 4. Punishment-IDs und History

Jede Aktion erzeugt einen `Punishment`-Datensatz mit Auto-Increment-`id`, angezeigt als **`P-<id>`**. `active` steuert, ob der Eintrag zur Eskalation zählt bzw. noch läuft. Default-Aktiv-Status: `warn`/`mute`/`ban` → aktiv; `kick`/`note`/`softban`/`unmute`/`unban`/`unwarn` → inaktiv. `/mod history <user>` zeigt bis zu 25 Einträge (🟢 aktiv / ⚫ inaktiv) mit Moderator, Zeitstempel, ggf. Ablaufzeit.

## 5. Konfiguration (`configs/modules/mod.yml`)

```yaml
enabled: true
reportChannelId: "0"            # Fallback für /report, siehe 2.3

whitelistRoleIds: []             # Rollen, die Auto-Mod komplett umgehen
whitelistChannelIds: []          # Kanäle, die Auto-Mod komplett umgehen

escalation:
  - threshold: 3
    action: mute
    durationMinutes: 60
  - threshold: 5
    action: kick
  - threshold: 7
    action: ban

autoMod:
  antiAd:
    enabled: true
    blockInvites: true
    blockAllLinks: false
    allowedDomains: []
    action: warn
    deleteMessages: true

  antiFlood:
    enabled: true
    maxMentions: 5
    action: warn
    timeoutMinutes: 10
    deleteMessages: true

  antiCaps:
    enabled: true
    threshold: 0.7
    minLength: 10
    action: delete
    deleteMessages: true

  wordFilter:
    enabled: false
    words: []
    substring: false
    action: warn
    deleteMessages: true

  antiRepeat:
    enabled: true
    repeats: 3
    windowSeconds: 60
    action: warn
    deleteMessages: true

  antiSpam:
    enabled: true
    messages: 5
    windowSeconds: 5
    action: warn
    timeoutMinutes: 5
    deleteMessages: true

  antiPhishing:
    enabled: true
    useBuiltinHeuristics: true   # Marken-Impersonation + verdächtige TLDs, siehe Abschnitt 2.6
    useUrlhaus: true             # externer Bedrohungsfeed (abuse.ch), siehe Abschnitt 2.6
    customBlocklist: []          # eigene, fest gesperrte Domains (inkl. Subdomains)
    action: timeout
    timeoutMinutes: 60
    deleteMessages: true

report:
  enabled: true
  cooldownSeconds: 60
  dailyLimit: 10

messageLog:
  enabled: true
  logEdits: true
  logDeletes: true
  ignoreBots: true

nuke:
  gifUrl: ""   # siehe Abschnitt 2.8
```

**Hinweis:** `configs/modules/mod.example.yml` (nur eine Referenzdatei, hat keinen Effekt auf den laufenden Bot)
ist seit mehreren Modul-Erweiterungen nicht mehr aktuell und listet längst nicht alle Schlüssel, die der Bot
tatsächlich versteht — als Nachschlagewerk gilt diese Wiki-Seite bzw. direkt `src/modules/mod/settings.ts`.
Die tatsächlich verwendete `configs/modules/mod.yml` wird beim ersten Start automatisch mit allen aktuellen
Default-Werten angelegt (inkl. `antiPhishing`, `messageLog`, `nuke` usw.) — dafür muss man nichts manuell
ergänzen. Fehlt nach einem Bot-Update ein neuer Schlüssel in einer bereits bestehenden `mod.yml`, wird er beim
nächsten Neustart automatisch mit seinem Default-Wert ergänzt, ohne bestehende Einstellungen zu verändern.

Feld-Übersicht:

| Schlüssel | Bedeutung | Default |
|---|---|---|
| `enabled` | Gesamtes Modul an/aus | `true` |
| `reportChannelId` | Fallback-Zielkanal für `/report` | `"0"` |
| `whitelistRoleIds`/`whitelistChannelIds` | Ausnahmen von allen Auto-Mod-Checks | `[]` |
| `escalation[].threshold` | Anzahl aktiver Warns (1–100) | siehe oben |
| `escalation[].action` | `mute`/`kick`/`ban` | — |
| `escalation[].durationMinutes` | Nur bei `mute` (1–40320) | 60 |
| `autoMod.*.enabled`/`action`/`deleteMessages` | Je Check konfigurierbar | siehe Beispiel |
| `report.cooldownSeconds`/`dailyLimit` | Spam-Schutz für `/report` | 60 / 10 |

**Feste Check-Reihenfolge** (nicht konfigurierbar, "first match wins"): antiPhishing → antiAd → antiFlood → antiCaps → wordFilter → antiRepeat → antiSpam.

## 6. Benötigte Channels/Rollen (`configs/config.yml`)

```yaml
channels:
  mod-log: "CHANNEL_ID"     # Automatisches Mod-Log
  alert:   "CHANNEL_ID"     # Anti-Nuke Notfall-Alarme
  staff:   "CHANNEL_ID"     # bevorzugter Ziel-Kanal für /report

roles:
  support:                  # dürfen Report-Buttons "Deny"/"Warn" nutzen
    - "ROLE_ID"
  alert-ping: ""            # Rolle bei Anti-Nuke-Alarmen (optional)
```

Ist `channels.mod-log` leer, wird das Mod-Log stillschweigend übersprungen (kein Fehler).

## 7. Berechtigungen im Überblick

| Command / Aktion | Benötigte Discord-Permission |
|---|---|
| `/mod` (Basis) | `Moderate Members` |
| `/mod kick` | + `Kick Members` |
| `/mod ban`/`tempban`/`softban`/`unban` | + `Ban Members` |
| `/mod clear`/`purge` | + `Manage Messages` |
| `/mod slowmode` | + `Manage Channels` |
| `/mod nuke` | + `Manage Channels` (ersetzt den ganzen Kanal, siehe Abschnitt 2.8) |
| `/mod reload` | `Manage Guild` |
| `/report` | keine (außer Selbst/Bot/Mod-Ausschluss) |
| Report-Button Deny/Warn | `Moderate Members` oder `roles.support` |
| Report-Button Timeout/Kick/Ban | zusätzlich passende Permission |

Zusätzlich greift bei jeder Aktion gegen ein Mitglied der Hierarchie-Check (Abschnitt 2.2), unabhängig von der Discord-Permission.
