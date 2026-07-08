# Konfiguration

GrumpyCore hat zwei Konfigurationsebenen:

1. **`configs/config.yml`** — zentrale Bot-Konfiguration: Token, Guild-ID, Sprache, Addon-Toggles, alle Channel-/Rollen-IDs, Logging, Bot-Presence.
2. **`configs/modules/<modul>.yml`** — modul-spezifische Feineinstellungen (Eskalationsschwellen, Cooldowns, Embed-Texte, Presets, ...). Wird pro Modul beim ersten Start automatisch mit Defaults angelegt.

---

## `config.yml` — vollständige Struktur

```yaml
token: "DEIN_BOT_TOKEN"
guildId: "DEINE_SERVER_ID"
language: "de"              # de | en
ownerIds: []                # Discord-User-IDs mit voller Kontrolle
errorWebhook: ""            # Optional: Discord-Webhook-URL für Crash-Meldungen

# ── Addons — true = aktiv, false = deaktiviert ──────────────────────────────
addons:
  welcome: true
  mod: true
  tickets: true
  news: true
  voice: true
  help: true
  customcmd: true
  music: true
  reactionroles: true
  leveling: true
  suggestions: true
  polls: true
  starboard: true
  reminders: true
  birthdays: true

# ── Channels — alle Channel-IDs an einem Platz ──────────────────────────────
channels:
  welcome: "CHANNEL_ID"       # Welcome- & Leave-Nachrichten
  verify: "CHANNEL_ID"        # Verifizierungs-Panel
  mod-log: "CHANNEL_ID"       # Automatisches Mod-Log
  alert: "CHANNEL_ID"         # Anti-Nuke-Alarme
  staff: "CHANNEL_ID"         # Bevorzugtes Ziel für /report
  support: "CHANNEL_ID"       # Ticket-Panel-Standardkanal
  ticket-log: "CHANNEL_ID"    # Ticket-Transcripts & Logs
  news: "CHANNEL_ID"          # News-Posts
  voice-lobby: "CHANNEL_ID"   # Join-to-Create-Lobby
  help: "CHANNEL_ID"          # Help-Hub-Embed
  radio: "CHANNEL_ID"         # Passive-Radio-Fallback-Channel
  suggestions: "CHANNEL_ID"   # Vorschläge-Board-Fallback
  starboard: "CHANNEL_ID"     # Starboard-Fallback
  levelup: ""                 # "" = gleicher Kanal, "dm" = Direktnachricht

# ── Rollen ───────────────────────────────────────────────────────────────────
roles:
  unverified: "ROLE_ID"       # Neue Member bis zur Verifizierung
  member: "ROLE_ID"           # Nach erfolgreicher Verifizierung
  support:                    # Dürfen Tickets verwalten + Report-Buttons nutzen (mehrere möglich)
    - "ROLE_ID"
  alert-ping: ""              # Optional bei Anti-Nuke gepingt

# ── Logging ──────────────────────────────────────────────────────────────────
logging:
  enabled: true
  dir: "logs"
  retentionDays: 7            # 0 = für immer
  captureStderr: true

# ── Bot-Presence (Status unter dem Bot-Namen) ───────────────────────────────
presence:
  enabled: true
  status: "online"            # online | idle | dnd | invisible
  rotateSeconds: 60
  activities:
    - "Watching Minetechworld.de"
    - "Listening to TechnoBase.FM"
    - "Playing /help"
```

**Auto-Heal:** Beim Start ergänzt der Bot automatisch fehlende `addons`-/`channels`-Felder (z.B. nach einem Update mit neuen Modulen) und entfernt veraltete Top-Level-Keys. Bestehende Werte und YAML-Kommentare bleiben erhalten.

---

## IDs kopieren

**Entwicklermodus aktivieren:** Discord → Einstellungen → Erweitert → **Entwicklermodus** an.

- **Channel-ID:** Rechtsklick auf Channel → **ID kopieren**
- **Rollen-ID:** Server-Einstellungen → Rollen → Rechtsklick → **ID kopieren**
- **Server-ID:** Rechtsklick auf Server-Icon → **ID kopieren**

---

## Modul-spezifische Configs (`configs/modules/`)

Jedes Modul hat eine eigene YAML-Datei mit feature-spezifischen Einstellungen. Wird beim ersten Start (oder erstmaligen Zugriff) automatisch mit Defaults angelegt.

| Datei | Modul | Details siehe |
|---|---|---|
| `welcome.yml` | Willkommen, Verify, Raid-Schutz | [Welcome-Modul](../addons/welcome.md) |
| `mod.yml` | Moderation, Auto-Mod, Eskalation | [Moderation-Modul](../addons/moderation.md) |
| `tickets.yml` | Ticket-Panels, Kategorien, Auto-Close | [Tickets-Modul](../addons/tickets.md) |
| `news.yml` | DM-Admin-Liste, Reaktionen | [News-Modul](../addons/news.md) |
| `voice.yml` | Join-to-Create-Verhalten | [Voice-Modul](../addons/voice.md) |
| `music.yml` | Radio-Presets, Passive-Mode | [Music-Modul](../addons/music.md) |
| `help.yml` | Help-Hub-Embed, Quick-Actions | [Help-Modul](../addons/help.md) |
| `reactionroles.yml` | Reaction-Role-Panels | [Reaction-Roles](../addons/reactionroles.md) |
| `leveling.yml` | XP-Bereich, Voice-XP, Rollen-Belohnungen | [Leveling](../addons/leveling.md) |
| `suggestions.yml` | Cooldown, Staff-Rollen, Threads | [Suggestions](../addons/suggestions.md) |
| `polls.yml` | Max-Laufzeit | [Polls](../addons/polls.md) |
| `starboard.yml` | Emoji, Threshold | [Starboard](../addons/starboard.md) |
| `birthdays.yml` | Congrats-Kanal, Rolle | [Birthdays](../addons/birthdays.md) |

`customcmd` und `reminders` sind reine DB-Module ohne eigene YAML.

> **Fallback-Kette:** Ist ein `channelId` in einer Modul-YAML leer, fällt der Bot auf das entsprechende zentrale `channels.*` in `config.yml` zurück (z.B. `suggestions.yml → channelId` → `channels.suggestions`). So reicht meist eine zentrale Stelle.

Nach dem Ändern einer Modul-YAML: `/‹modul› reload` (falls vorhanden) oder Bot-Neustart.

---

## Umgebungsvariablen (`.env` / Pelican-Panel-Variablen)

| Variable | Bedeutung | Default |
|---|---|---|
| `DISCORD_TOKEN` | Überschreibt `token` aus `config.yml` | — |
| `GUILD_ID` | Überschreibt `guildId` | — |
| `DATABASE_PROVIDER` | `sqlite` \| `mysql` \| `mariadb` \| `postgres` | `sqlite` |
| `DATABASE_URL` | Verbindungsstring | `file:./database.sqlite` |
| `NODE_ENV` | `production` \| `development` | `production` |
| `CONFIG_DIR` | Alternativer Pfad für `configs/` | `configs` |

**Wichtig für eigene Deployments:** Ist `NODE_ENV=production` gesetzt, installiert `npm ci` standardmäßig **keine** `devDependencies` — das bricht den TypeScript-Build (`tsc-alias` fehlt). Für einen Source-Build (kein `build/`-Ordner vorhanden) muss `npm ci --include=dev` verwendet werden, danach kann mit `npm prune --omit=dev` wieder aufgeräumt werden.
