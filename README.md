# GrumpyCore — Wiki

Willkommen im offiziellen Wiki für **GrumpyCore**, den selbstgehosteten Discord-Bot von Minetechworld.de.

> Entwickelt von **darksoon** · Repo: [Forgejo](https://dev.sven-neurath.de/darksoon/GrumpyCore) ([GitHub-Spiegel](https://github.com/darksoon/GrumpyCore))
> Stand dieses Wikis: main-Branch, 15 Module, TypeScript/Sapphire/discord.js v14/Prisma

---

## 📚 Inhaltsverzeichnis

### ⚙️ Setup

- [Installation & Deployment](setup/installation.md) — Pelican-Panel, Egg, Datei-Upload, erster Start
- [Konfiguration](setup/configuration.md) — `config.yml`, Modul-YAMLs, Channel-/Rollen-IDs, Env-Variablen

### 🔧 Module

**Community & Onboarding**

- [Welcome](addons/welcome.md) — Join-/Leave-Nachrichten, Banner, Verifizierung mit CAPTCHA, Raid-Schutz
- [Help-Hub](addons/help.md) — Interaktive Befehlsübersicht mit Dropdown + Quick-Actions

**Moderation**

- [Moderation](addons/moderation.md) — Warn/Kick/Ban/Tempban/Mute, Eskalationsleiter, Auto-Mod, Reports
- [Tickets](addons/tickets.md) — Multi-Panel-Support-System mit Claim, Transcript, Rating, Auto-Close

**Voice & Musik**

- [Voice (Join-to-Create)](addons/voice.md) — Automatische, benutzereigene Voice-Channels
- [Music/Radio](addons/music.md) — Internet-Radio-Streaming (Icecast/SHOUTcast), Passive-Mode

**Engagement**

- [Leveling](addons/leveling.md) — XP durch Text/Voice, Rang-Karten, Leaderboard, Rollen-Belohnungen
- [Reaction Roles](addons/reactionroles.md) — Self-Service-Rollen über Buttons/Dropdowns
- [Starboard](addons/starboard.md) — Automatisches Hervorheben beliebter Nachrichten
- [Custom Commands](addons/customcmd.md) — Admin-definierte `/cmd`-Befehle (Text/Embed/Role-Toggle)

**Feedback & Kommunikation**

- [Suggestions](addons/suggestions.md) — Vorschläge einreichen, abstimmen, Status verwalten
- [Polls](addons/polls.md) — Live-Abstimmungen mit Button-Voting
- [News](addons/news.md) — Admin-DM-Workflow für News-Posts

**Utility**

- [Reminders](addons/reminders.md) — Persönliche Erinnerungen (`/remind`)
- [Birthdays](addons/birthdays.md) — Geburtstage speichern + automatisch gratulieren

### 👥 Für Staff

- [Berechtigungen](staff/permissions.md) — Wer darf was, Rollen-Hierarchie
- [Mod-Workflow](staff/workflow.md) — Typischer Ablauf bei Regelverstößen, Tickets, Reports

---

## 🚀 Schnellstart

1. Discord-Bot-Anwendung erstellen, Token holen, Intents (Server Members + Message Content) aktivieren
2. Code auf den Pelican-Server hochladen (Tarball, Pre-Built oder Source)
3. Panel-Variablen setzen: `DISCORD_TOKEN`, `GUILD_ID`, `LANGUAGE`
4. Bot starten → `configs/config.yml` wird automatisch mit Template angelegt
5. Channel-/Rollen-IDs eintragen → siehe [Konfiguration](setup/configuration.md)
6. Bot neu starten — Verify-Panel, Ticket-Panels und Help-Hub werden automatisch gepostet

---

## Alle Module im Überblick (15)

| Modul | Addon-Key | Zweck |
|---|---|---|
| Welcome | `welcome` | Join/Leave, Verify, Raid-Schutz |
| Moderation | `mod` | Warn/Kick/Ban/Tempban, Auto-Mod, Reports |
| Tickets | `tickets` | Support-Ticket-System |
| News | `news` | DM-basierte News-Posts |
| Voice | `voice` | Join-to-Create |
| Help | `help` | Interaktives Help-Hub |
| CustomCmd | `customcmd` | Admin-definierte Befehle |
| Music | `music` | Internet-Radio |
| ReactionRoles | `reactionroles` | Self-Service-Rollen |
| Leveling | `leveling` | XP & Ränge |
| Suggestions | `suggestions` | Vorschläge & Voting |
| Polls | `polls` | Live-Abstimmungen |
| Starboard | `starboard` | Beliebte Nachrichten hervorheben |
| Reminders | `reminders` | Persönliche Erinnerungen |
| Birthdays | `birthdays` | Geburtstage & Glückwünsche |

Jedes Modul lässt sich einzeln über `addons.<key>: false` in `config.yml` deaktivieren.

---

## 📞 Support

Bei Fragen an **darksoon** wenden.
