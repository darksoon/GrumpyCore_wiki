# GrumpyCore — Help-Modul

## 1. Einleitung

Das Help-Modul ("Help-Hub") stellt eine **persistente Embed-Nachricht** in einem konfigurierten Channel bereit mit einem **Dropdown-Menü** (Modulauswahl → ephemere Befehlsliste) und optional bis zu **5 Quick-Action-Buttons** (z.B. "Ticket öffnen", "Verifizieren"). Die Nachricht wird beim Boot automatisch gepostet, bei Config-Reload in-place aktualisiert und ihre ID in der DB getrackt (kein Duplizieren nach Neustart).

## 2. Wie es funktioniert

### 2.1 Auto-Deploy beim Boot

1. Modul aktiv (`enabled` + `addons.help`)?
2. `channels.help` gesetzt und erreichbar?
3. Existiert bereits eine getrackte Hub-Nachricht?
   - Im selben Channel → **in-place editiert** (kein Duplikat).
   - In anderem Channel (Channel gewechselt) → alte gelöscht, neu gepostet.
   - Keine/manuell gelöscht → neu gepostet, DB aktualisiert.
4. Schlägt der DB-Write nach Post fehl, wird die gesendete Nachricht wieder gelöscht (kein untracked Duplikat).

### 2.2 Auto-Update bei Reload

`/help reload` lädt die YAML neu und ruft `autoDeployHelpHub(guild, { editOnly: true })` auf: editiert **ausschließlich** eine bereits existierende Nachricht, postet **nie** neu (kein Überraschungs-Deploy).

### 2.3 Dropdown-Auswahl pro Modul

Listet alle Module, deren Addon-Flag nicht `false` ist. Ist kein Modul aktiv, entfällt die Dropdown-Zeile (Discord erlaubt kein Select-Menü mit 0 Optionen). Bei Auswahl: ephemere Command-Übersicht des Moduls.

- **Staff-Erkennung** (nur UX-Filter, keine echte Rechteprüfung): Nutzer mit `ModerateMembers`/`KickMembers`/`BanMembers`/`ManageMessages`/`ManageGuild` sehen zusätzliche, als "*(Staff)*" markierte Befehle.
- Zu lange Listen werden zeilenweise gekürzt.

Bekannte Module im Dropdown: welcome, mod, tickets, news, voice, customcmd, music, reactionroles, leveling, suggestions, polls, starboard.

### 2.4 Quick-Action-Buttons

Bis zu 5 Buttons aus `settings.quickActions`. Jeder hat `label`, `style`, optional `emoji`, und entweder `action` (`ticket`/`verify`, verweist auf `channels.support`/`channels.verify`) oder `jumpToChannelId`.

**Absicherung gegen "stale" Buttons:** Ändert sich die Quick-Actions-Reihenfolge/Inhalt nach dem Posten, prüft der Handler ob die im Custom-ID kodierte Aktion noch zum aktuellen Index passt — bei Abweichung wird die Aktion abgelehnt statt versehentlich die falsche auszuführen.

### 2.5 Edit-in-place-Verhalten

| Situation | Verhalten |
|---|---|
| Boot, Hub existiert im selben Channel | Editiert |
| Boot, Hub in anderem Channel | Alte gelöscht, neu gepostet |
| Boot, kein Hub getrackt | Neu gepostet |
| `/help reload` | Nur Edit, nie neuer Post |
| `/help redeploy` | Immer neuer Post, alte erst NACH Erfolg gelöscht |

## 3. Slash-Commands

| Command | Beschreibung | Berechtigung |
|---|---|---|
| `/help show` | Zeigt Embed + Dropdown ephemeral | Jeder |
| `/help module <name>` | Zeigt direkt die Befehle eines Moduls | Jeder |
| `/help redeploy` | Postet Hub zwangsweise neu | `ManageGuild` |
| `/help reload` | Lädt `help.yml` neu, aktualisiert Live-Nachricht | `ManageGuild` |

## 4. Konfiguration (`configs/modules/help.yml`)

Kein `.example.yml` im Repo — wird aus den Code-Defaults erzeugt.

```yaml
enabled: true

embed:
  author:
    name: "%guild_name%"
    iconUrl: "%guild_icon%"
  title: "📚 Befehlsübersicht"
  description: |
    **Willkommen im Help-Hub!**
    Hier findest du alle Funktionen des Bots auf einen Blick.
  color: "#5865F2"
  thumbnail: "%guild_icon%"
  footer:
    text: "%guild_name% · Help-Hub"
  timestamp: true

quickActions:
  - label: "Ticket öffnen"
    emoji: "🎫"
    style: "Primary"
    action: "ticket"
  - label: "Verifizieren"
    emoji: "✅"
    style: "Success"
    action: "verify"

moduleDescriptions: {}
```

| Feld | Typ | Default | Beschreibung |
|---|---|---|---|
| `enabled` | bool | `true` | Help-Hub-Feature an/aus |
| `embed.*` | Embed-Objekt | siehe oben | Haupt-Embed im Help-Channel |
| `quickActions[]` | Array (max. 5) | 2 Einträge | Buttons unter dem Dropdown |
| `quickActions[].action` | `ticket`/`verify` | — | Eingebaute Aktion |
| `quickActions[].jumpToChannelId` | Snowflake | — | Alternative: fester Channel-Link |
| `moduleDescriptions` | `{modul: text}` | `{}` | Zusatztext pro Modul im Detail-Embed |

Die im Dropdown angezeigten Module werden aus den globalen `addons.*`-Flags in `config.yml` abgeleitet, nicht in `help.yml` gepflegt.

## 5. Benötigter Help-Channel

```yaml
addons:
  help: true

channels:
  help: "CHANNEL_ID"
  support: "CHANNEL_ID"    # Ziel des "Ticket öffnen"-Buttons
  verify: "CHANNEL_ID"     # Ziel des "Verifizieren"-Buttons
```

## 6. Berechtigungen

| Aktion | Berechtigung |
|---|---|
| `/help show`, `/help module` | Jeder |
| Dropdown/Quick-Actions | Jeder |
| `/help redeploy`, `/help reload` | `ManageGuild` |
