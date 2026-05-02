# GrumpyMod — Moderation

GrumpyMod ist das Moderations-Addon. Es übernimmt automatische Schutzmaßnahmen und stellt manuelle Mod-Commands bereit.

---

## Automatischer Schutz

### Anti-Spam
Zu viele Nachrichten in kurzer Zeit → automatische Verwarnung + Nachrichten löschen.

**Standard:** 5 Nachrichten in 5 Sekunden

### Anti-Flood
Dieselbe Nachricht mehrfach hintereinander → Verwarnung.

**Standard:** 3 identische Nachrichten in Folge

### Anti-Advertising
Discord-Invite-Links und externe URLs in nicht-erlaubten Channels werden automatisch gelöscht.

**Erlaubte Channels** können in der Config definiert werden (`allowed-link-channels`).

### Anti-Nuke
Massenhafte Channel-Löschungen oder Bans innerhalb kurzer Zeit → sofortiger Alert im Alert-Channel + Rollen-Entzug beim Verursacher.

---

## Warn-Eskalation

| Stufe | Aktion |
|---|---|
| 1 | DM-Warnung |
| 2 | DM-Warnung #2 |
| 3 | Timeout (Standard: 10 Min) |
| 4 | Kick |
| 5+ | Ban |

Verwarnungen **resetten** nach 24 Stunden (konfigurierbar).

---

## Mod-Log

Alle Ereignisse werden automatisch im `mod-log` Channel geloggt:

| Event | Farbe |
|---|---|
| 👋 Join | Grün |
| 🚪 Leave/Kick | Orange/Grau |
| 🔨 Ban | Rot |
| ✅ Unban | Grün |
| ⚠️ Warn | Gelb |
| ⏱️ Timeout | Lila |
| ✏️ Nick-Änderung | Blau |
| 🟣 Rolle vergeben | Lila |
| ⚫ Rolle entfernt | Dunkel |
| 📢 Channel erstellt | Türkis |
| ⚙️ Server-Settings | Orange |

---

## Report-System

Jeder User kann mit `/report @user [grund]` einen anderen User melden.

- Optionaler Screenshot als Beweis
- Cooldown zwischen Reports (Standard: 5 Min)
- Max. Reports pro Tag (Standard: 3)
- Mods sehen den Report im Staff-Channel mit Aktions-Buttons: **Ablehnen / Warn / Timeout / Kick / Ban**

---

## Punishment-IDs

Jede Verwarnung bekommt eine eindeutige ID (z.B. `P#42`). Diese ID kann genutzt werden um:
- Eine spezifische Verwarnung zu löschen: `/mod unwarn 42`
- In der History nachzuschauen: `/history @user`

---

## Konfiguration (`configs/GrumpyMod/config.yml`)

```yaml
spam-messages: 5          # Nachrichten bevor Spam erkannt wird
spam-interval: 5          # Zeitfenster in Sekunden
flood-duplicates: 3       # Identische Nachrichten in Folge
allowed-link-channels: [] # Channel-IDs wo Links erlaubt sind
nuke-channel-threshold: 3 # Channel-Löschungen für Nuke-Alert
nuke-ban-threshold: 5     # Bans für Nuke-Alert
nuke-time-window: 10      # Zeitfenster in Sekunden
warn-reset-hours: 24      # Stunden bis Verwarnungen resetten
timeout-duration: 600     # Timeout in Sekunden (Standard: 10 Min)
report-cooldown: 300      # Sekunden zwischen Reports
report-max-per-day: 3     # Max Reports pro User/Tag
```
