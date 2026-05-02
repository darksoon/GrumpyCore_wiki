# GrumpyMC — Minecraft Status

Der Bot zeigt den aktuellen Status des Minecraft-Servers in einem dedizierten Channel an.

---

## Was angezeigt wird

```
🟢 Server ONLINE
Spieler:  7/50
Version:  1.21.x
MOTD:     Welcome to Minetechworld.de
Letzte Aktualisierung: 14:32 Uhr
```

Bei Offline:
```
🔴 Server OFFLINE
```

---

## Funktionsweise

- Bot pings den MC-Server alle **5 Minuten** via Standard-Protokoll (SLP)
- Das Embed wird **editiert** (nicht neu gepostet) → kein Channel-Spam
- Funktioniert mit **jedem** Minecraft-Server ohne zusätzliche Plugins

---

## Einrichtung

1. In `configs/config.yml` eintragen:
```yaml
channels:
  mc-status: "CHANNEL_ID"
```

2. In `configs/GrumpyMC/config.yml`:
```yaml
mc-host: "dein-server.de"   # Hostname oder IP
mc-port: 25565               # Port (Standard: 25565)
update-interval: 300         # Sekunden zwischen Updates
```

3. Bot starten → aus der Konsole die Message-ID kopieren:
```
[GrumpyMC] Status message created: 1234567890 — save as message-ids.mc-status
```

4. ID in `configs/config.yml` unter `message-ids.mc-status` eintragen → `reload`

---

## Chat-Bridge (DiscordSRV)

Für eine Chat-Bridge zwischen Discord und Minecraft empfehlen wir **DiscordSRV** — ein separates Plugin für den MC-Server.

DiscordSRV ist kompatibel mit:
- Paper / Spigot / Purpur
- Youer (NeoForge + Paper Hybrid) ← euer Setup

**Installation:** [docs.discordsrv.com](https://docs.discordsrv.com)
