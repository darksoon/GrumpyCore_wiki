# GrumpyCore — Moderation-Modul

## 1. Einleitung

Das `mod`-Modul ist das zentrale Moderationssystem von GrumpyCore. Es bündelt:

- **Klassische Moderationsaktionen**: Warn, Kick, Ban, Tempban (befristeter Ban mit Auto-Unban), Mute (Timeout), Softban, Note, Unmute, Unban, Unwarn
- **Eskalationsleiter**: automatische Folgeaktion (Mute → Kick → Ban), sobald ein Mitglied eine konfigurierte Zahl aktiver Verwarnungen erreicht
- **Auto-Mod**: passiver Nachrichten-Scanner (Anti-Spam, Anti-Flood, Anti-Ad, Anti-Phishing, Anti-Caps, Anti-Repeat, Wortfilter)
- **Report-System**: `/report`-Befehl für normale Mitglieder + Button-Panel für Staff
- **Mod-Log**: automatisches Protokoll aller Aktionen in einem konfigurierten Textkanal
- **Punishment-History**: jede Aktion wird als `Punishment`-Datensatz gespeichert, abrufbar über `/mod history`
- **Auto-Unban-Runner**: Hintergrundprozess, der abgelaufene Tempbans automatisch aufhebt

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

### 2.4 Auto-Unban-Runner

- Läuft alle 60 Sekunden, einmal sofort beim Start (holt verpasste Tempban-Abläufe nach).
- Sucht alle `Punishment`-Zeilen `type: 'ban'`, `active: true`, `expiresAt <= jetzt`.
- Discord-Fehler 10026 (Unknown Ban) wird als „bereits erledigt" behandelt; jeder andere Fehler ist transient (Zeile bleibt aktiv, nächster Versuch).
- Bei Erfolg: Ban-Zeile auf inaktiv, neuer `unban`-Eintrag, Mod-Log-Post.
- **Wichtig:** `/mod tempban` legt keinen eigenen Typ „tempban" an — es ist ein normaler `type: 'ban'` mit `expiresAt` gesetzt, nutzt also dieselbe Eskalations-/History-/Log-Darstellung wie ein Permanent-Ban.

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
| `note` | `user`, `note` (beide required) | — |
| `softban` | `user` (required), `reason`, `delete-days` (Default 1) optional | `Ban Members` |
| `reload` | — | `Manage Guild` |

Details: Ban/Tempban/Softban/Unban führen die Discord-API-Aktion immer zuerst aus — schlägt sie fehl, wird kein Punishment-Eintrag angelegt. `note` erzeugt einen Eintrag ohne DM/Eskalationszählung.

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
    useBuiltinHeuristics: true
    customBlocklist: []
    action: timeout
    timeoutMinutes: 60
    deleteMessages: true

report:
  enabled: true
  cooldownSeconds: 60
  dailyLimit: 10
```

**Hinweis:** `antiPhishing` (Marken-Impersonation-Erkennung, verdächtige TLD+Bait-Keyword-Heuristik) ist im Code-Schema vorhanden, fehlt aber in der mitgelieferten `mod.example.yml` — bei bestehenden Installationen ggf. manuell ergänzen.

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
| `/mod reload` | `Manage Guild` |
| `/report` | keine (außer Selbst/Bot/Mod-Ausschluss) |
| Report-Button Deny/Warn | `Moderate Members` oder `roles.support` |
| Report-Button Timeout/Kick/Ban | zusätzlich passende Permission |

Zusätzlich greift bei jeder Aktion gegen ein Mitglied der Hierarchie-Check (Abschnitt 2.2), unabhängig von der Discord-Permission.
