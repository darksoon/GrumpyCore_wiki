# GrumpyCore — Leveling-Modul

## 1. Einleitung

Vergibt XP für Text- und Sprachaktivität und leitet daraus ein Level ab. Drei Commands (`/rank`, `/leaderboard`, `/xp`), Canvas-generierte Rang-/Level-Up-Karten, optionale Rollen-Belohnungen pro Level, paginiertes Leaderboard.

> Hinweis: Es gibt aktuell **keine** rollen-/kanalbasierten XP-Multiplikatoren und **keine** Boost-Events/`/xp boost` — das existiert nur in einem separaten, nicht gemergten Feature-Branch.

## 2. Funktionsweise

### 2.1 Text-XP

Cooldown pro User (`messageXp.cooldownSeconds`, Default 60s), Menge zufällig im Bereich `[min, max]` (Default 15–25). Ignorierte Kanäle konfigurierbar. Nach Vergabe: Level-Check, ggf. `handleLevelUp()`.

### 2.2 Voice-XP

Eigener 60-Sekunden-Tick (nicht event-basiert). Pro Tick: alle Voice-Kanäle durchsucht, ausgeschlossene übersprungen. **Solo-Regel**: standardmäßig min. 2 Nicht-Bot-Mitglieder nötig (`voiceXp.soloAllowed: false`, MEE6-Vorbild als Anti-AFK-Schutz); `true` erlaubt auch Einzelpersonen. Ausgeschlossen: server-stumm, server-taub, selbst-taub. Berechtigte erhalten `voiceXp.xpPerMinute` (Default 2) pro Tick.

### 2.3 Level-Formel (MEE6-kompatibel)

```
xpForLevel(n) = 5n² + 50n + 100
```

Level 1 braucht 100 XP, Level 10 kumuliert ca. 5.500 XP. Nur `totalXp` wird gespeichert — Level wird bei jedem Zugriff neu berechnet.

### 2.4 Level-Up-Benachrichtigung

Bei Level-Anstieg: Canvas-Karte (900×300px) + Text mit Platzhaltern `{user}`/`{level}`/`{xp}`. Ziel-Kanal-Auflösung: `levelUp.channel === 'dm'` → DM; sonst gesetzter Kanal; sonst `channels.levelup`; sonst der auslösende Kanal (nur bei Text-XP vorhanden).

**Rollen-Belohnungen** (`roleRewards`): Rollen ≤ neuem Level werden vergeben (sofern Bot-Hierarchie erlaubt). `stackRoles: true` (Default) = alle behalten; `false` = nur höchste behalten, niedrigere entfernt.

### 2.5 Rang-/Level-Up-Karte

Canvas-Rendering (`@napi-rs/canvas`), dunkler Gradient-Stil. Rang-Karte: 900×280px, rundes Avatar mit Akzent-Ring, Rang `#N`, Fortschrittsbalken. Level-Up-Karte: 900×300px, "⬆ LEVEL UP!"-Badge. Akzentfarbe konfigurierbar. Avatar-Ladefehler → einfarbiger Fallback-Kreis.

### 2.6 Leaderboard mit Pagination

10 Einträge/Seite, sortiert nach `totalXp`. Medaillen-Emoji für Rang 1–3. `◀`/`▶`-Buttons, deaktiviert wenn keine weitere Seite existiert. Nicknamen Markdown-escaped (gegen Formatierungs-Injection).

## 3. Slash-Commands

### `/rank [user]`

Zeigt Rang-Karte als PNG. Optional `user` (Default: du selbst). Jeder darf. Nicht ephemeral.

### `/leaderboard [page]`

XP-Rangliste, 10/Seite. Optional `page`. Jeder darf.

### `/xp <subcommand>` — Admin-Verwaltung, `ManageGuild`

| Subcommand | Optionen | Beschreibung |
|---|---|---|
| `give` | `user`, `amount` (1–1.000.000) | XP hinzufügen |
| `take` | `user`, `amount` (1–1.000.000) | XP abziehen (min. 0) |
| `set` | `user`, `amount` (0–100.000.000) | Gesamt-XP exakt setzen |
| `reset` | `user` | XP auf 0 |

Löst bei Level-Anstieg dieselbe Level-Up-Logik aus (Karte, Kanal, Rollen-Rewards).

## 4. Konfiguration (`configs/modules/leveling.yml`)

Kein `.example.yml` im Repo — Code-Defaults dienen als Vorlage.

```yaml
enabled: true

messageXp:
  enabled: true
  min: 15
  max: 25
  cooldownSeconds: 60
  ignoredChannels: []

voiceXp:
  enabled: true
  xpPerMinute: 2
  excludedChannels: []
  soloAllowed: false

levelUp:
  channel: ""              # "" = gleicher Kanal, "dm" = DM, sonst Kanal-ID
  message: "🎉 {user} hat **Level {level}** erreicht!"

roleRewards: []             # [{ level: N, roleId: "..." }]
stackRoles: true

accentColor: "#5865F2"
```

| Feld | Default | Bedeutung |
|---|---|---|
| `enabled` | `true` | Modul komplett an/aus |
| `messageXp.min`/`max` | 15/25 | XP-Bereich pro Nachricht |
| `messageXp.cooldownSeconds` | `60` | Cooldown zwischen XP-Vergaben |
| `voiceXp.xpPerMinute` | `2` | XP pro 60s-Tick |
| `voiceXp.soloAllowed` | `false` | Anti-AFK-Regel |
| `levelUp.channel` | `""` | Ziel-Kanal |
| `stackRoles` | `true` | Rollen stapeln vs. nur höchste |
| `accentColor` | `#5865F2` | Farbe der Karten |

## 5. Level-Up-Channel (`config.yml`)

```yaml
channels:
  levelup: ""    # "" = gleicher Kanal, "dm" = DM

addons:
  leveling: true
```

Priorität: `levelUp.channel` (Modul) > `channels.levelup` (global) > auslösender Kanal.

## 6. Berechtigungen

| Command | Berechtigung |
|---|---|
| `/rank`, `/leaderboard` | Jeder |
| `/xp give`/`take`/`set`/`reset` | `ManageGuild` |

Der Bot selbst benötigt `Manage Roles` für Rollen-Belohnungen — ohne diese Berechtigung wird der Schritt stillschweigend übersprungen.
