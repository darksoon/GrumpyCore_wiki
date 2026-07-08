# Installation & Deployment

## Voraussetzungen

- Pelican Panel mit Node.js-Egg (`nodejs_24` oder `nodejs_22`)
- Discord Bot Token ([Developer Portal](https://discord.com/developers/applications))
- Bot mit folgenden **privilegierten Intents** aktiviert: **Server Members**, **Message Content**
  - (Der frühere Presence-Intent wird nicht mehr benötigt und wurde entfernt.)

---

## Schritt 1 — Discord Bot erstellen

1. [discord.com/developers/applications](https://discord.com/developers/applications) → **New Application**
2. Links auf **Bot** → **Add Bot**
3. **Public Bot** deaktivieren (sofern der Bot nur für den eigenen Server laufen soll)
4. Unter **Privileged Gateway Intents**: **Server Members Intent** und **Message Content Intent** aktivieren
5. **Token** kopieren (wird für die Pelican-Server-Variable `DISCORD_TOKEN` gebraucht)

---

## Schritt 2 — Pelican Panel: Server anlegen

**Egg:** `pelican-egg.json` (liegt im Repo-Root von GrumpyCore) — in Pelican unter Admin → Eggs importieren.

**Docker-Image:** `ghcr.io/parkervcp/yolks:nodejs_24` (oder `nodejs_22`)

**Wichtige Server-Variablen (Panel):**

| Variable | Bedeutung |
|---|---|
| `DISCORD_TOKEN` | Bot-Token aus Schritt 1 |
| `GUILD_ID` | Server-ID, auf der der Bot läuft |
| `LANGUAGE` | `de` oder `en` |
| `DATABASE_PROVIDER` | `sqlite` (Default), `mysql`, `mariadb` oder `postgres` |
| `DATABASE_URL` | Nur bei nicht-sqlite nötig, z.B. `mysql://user:pass@host:3306/dbname` |
| `NODE_ENV` | `production` (Default) |

---

## Schritt 3 — Code hochladen

Drei unterstützte Upload-Varianten (das Install-Script erkennt automatisch, was vorliegt):

1. **Release-Tarball** (`grumpy-bot-*.tar.gz`, z.B. aus einer GitHub-Actions-Release) — wird automatisch entpackt.
2. **Pre-Built-Upload** — `build/`-Ordner + `package.json` bereits vorhanden.
3. **Source-Upload** — `src/`, `package.json`, `prisma/`, `scripts/`, `configs/config.example.yml` (ohne fertigen `build/`-Ordner) — wird beim Install automatisch gebaut.

> **Wichtig:** `configs/config.yml`, `configs/modules/*.yml` (ohne `.example`) und `database.sqlite` NICHT hochladen — die werden automatisch erstellt bzw. sollten die Live-Werte des Servers enthalten und nicht überschrieben werden.

---

## Schritt 4 — Installation ausführen

Das Pelican-Install-Script (`pelican-egg.json`) macht automatisch:

1. Build-Tools installieren (`build-essential`, `python3`, `ca-certificates`)
2. `npm ci` (bzw. `npm install`) — bei Source-Upload inkl. Dev-Dependencies für den Build-Schritt
3. `npm run build` (TypeScript → JavaScript, Prisma-Client generieren)
4. `configs/config.yml` aus den Panel-Variablen erzeugen (nur falls noch nicht vorhanden — bestehende Datei wird nie überschrieben)
5. `.env` mit `DATABASE_PROVIDER`/`DATABASE_URL`/`NODE_ENV` schreiben
6. `npm run db:push` — Datenbankschema anwenden

---

## Schritt 5 — Erster Start

1. Server in Pelican starten.
2. In der Konsole erscheinen ggf. Warnungen zu fehlenden Channel-/Rollen-IDs.
3. `configs/config.yml` öffnen und Channel-/Rollen-IDs eintragen → siehe [Konfiguration](configuration.md).
4. Bot neu starten (oder die jeweiligen `/… reload`-Commands nutzen, wo verfügbar).
5. Beim Start postet der Bot automatisch: Verify-Panel (falls `welcome`-Modul aktiv), Ticket-Panels, Help-Hub.
6. Alle Modul-YAMLs unter `configs/modules/` werden beim ersten Start automatisch mit Defaults angelegt.

---

## Console-Commands (Pelican-Konsole, nicht Discord!)

| Command | Wirkung |
|---|---|
| `stop` / `exit` / `quit` | Graceful Shutdown (wie Pelican-„Stop"-Button / SIGTERM) |
| `help` / `?` | Liste der Console-Commands |

Pelican „Stop" (SIGTERM) und Ctrl-C (SIGINT) lösen denselben sauberen Shutdown-Flow aus (10s-Timeout-Fallback, zweites SIGINT/SIGTERM erzwingt sofortigen Exit).

---

## Bot einladen

```
https://discord.com/api/oauth2/authorize?client_id=BOT_ID&permissions=8&scope=applications.commands%20bot
```

`BOT_ID` durch die Application-ID ersetzen. `permissions=8` = Administrator (einfachste Option). Wer feingranularer einladen möchte, braucht mindestens: Manage Roles, Manage Channels, Kick Members, Ban Members, Moderate Members, Manage Messages, Manage Webhooks, Connect, Speak, Send Messages, Embed Links, Attach Files, Read Message History, Use External Emojis, Manage Nicknames.

---

## Datenbank-Backends

| Provider | `DATABASE_URL`-Beispiel |
|---|---|
| `sqlite` (Default) | `file:./database.sqlite` — kein Setup nötig |
| `mysql`/`mariadb` | `mysql://user:pass@host:3306/dbname` |
| `postgres` | `postgresql://user:pass@host:5432/dbname` |

Das Schema wird aus `prisma/schema.template.prisma` je nach `DATABASE_PROVIDER` generiert (`npm run db:push` nach jeder Schema-Änderung).

> Es gibt derzeit kein automatisiertes Backup der Bot-Datenbank — vor größeren Updates empfiehlt sich ein manuelles Backup von `database.sqlite` (bzw. der externen DB).
