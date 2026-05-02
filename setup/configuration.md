# Konfiguration

Die zentrale Konfiguration liegt in `configs/config.yml`. Hier werden **alle** wichtigen IDs eingetragen — kein Suchen in verschiedenen Addon-Configs.

---

## config.yml — Übersicht

```yaml
# ── BOT ─────────────────────────────────
token: "DEIN_BOT_TOKEN"
primary-guild: "DEINE_SERVER_ID"

error-webhook: ""    # Optional: Discord Webhook für Crash-Meldungen

# ── ADDONS — true = aktiv, false = deaktiviert
addons:
  GrumpyWelcome: true
  GrumpyMod: true
  GrumpyTickets: true
  GrumpyNews: true
  GrumpyMC: true

# ── CHANNELS — alle Channel-IDs hier eintragen
channels:
  welcome: "ID"      # Willkommens- & Leave-Nachrichten
  verify: "ID"       # Verifizierungs-Button
  mod-log: "ID"      # Mod-Log (automatisch)
  alert: "ID"        # Anti-Nuke Notfälle
  staff: "ID"        # Manuelle Mod-Aktionen (warn/kick/ban)
  support: "ID"      # Ticket-Panel
  ticket-log: "ID"   # Ticket-Transcripts
  news: "ID"         # News-Posts
  mc-status: "ID"    # Minecraft Status-Embed

# ── ROLES — alle Rollen-IDs hier eintragen
roles:
  unverified: "ID"   # Neue Member bis zur Verifizierung
  member: "ID"       # Nach erfolgreicher Verifizierung
  support:           # Wer Tickets verwalten darf (mehrere möglich)
    - "ID"           # z.B. Supporter
    - "ID"           # z.B. Moderator
  alert-ping: ""     # Rolle die bei Anti-Nuke gepingt wird (optional)

# ── MESSAGE IDs — nach erstem Start aus der Konsole kopieren
message-ids:
  verify-panel: ""   # ID der Verify-Button Nachricht
  ticket-panel: ""   # ID der Ticket-Panel Nachricht
  mc-status: ""      # ID des MC-Status Embeds
```

---

## IDs kopieren — so geht's

**Developer Mode aktivieren:**
Discord → Einstellungen → Erweitert → **Entwicklermodus** an

**IDs kopieren:**
- **Channel-ID:** Rechtsklick auf Channel → **ID kopieren**
- **Rollen-ID:** Server-Einstellungen → Rollen → Rechtsklick → **ID kopieren**
- **Server-ID:** Rechtsklick auf Server-Icon → **ID kopieren**

---

## Message-IDs eintragen

Nach dem ersten Start erscheinen in der Konsole:
```
[GrumpyWelcome] Verify message posted: 1234567890123456789 — save as message-ids.verify-panel
[GrumpyTickets] Ticket panel posted: 9876543210987654321 — save as message-ids.ticket-panel
[GrumpyMC] Status message created: 1111222233334444555 — save as message-ids.mc-status
```

Diese IDs in `configs/config.yml` unter `message-ids` eintragen, dann `reload`.

---

## Addon-spezifische Configs

Jedes Addon hat zusätzlich eine eigene Config für addon-spezifische Einstellungen:

| Datei | Inhalt |
|---|---|
| `configs/GrumpyWelcome/config.yml` | CAPTCHA-Timeout, Banner-Farben, Leave-Nachrichten |
| `configs/GrumpyMod/config.yml` | Spam-Schwellenwerte, Timeout-Dauer, Report-Cooldown |
| `configs/GrumpyTickets/config.yml` | Kategorien, Auto-Close Zeiten, Template |
| `configs/GrumpyNews/config.yml` | Admin-IDs, Reactions, Session-Timeout |
| `configs/GrumpyMC/config.yml` | MC-Host, Port, Update-Interval |

> Diese Configs werden **automatisch** beim ersten Start generiert. Einfach die Werte anpassen und `reload` eingeben.
