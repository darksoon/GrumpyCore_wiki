# GrumpyCore — Leveling-Modul

## 1. Einleitung

Vergibt XP für Text- und Sprachaktivität und leitet daraus ein Level ab. Drei Commands (`/rank`, `/leaderboard`, `/xp`), Canvas-generierte Rang-/Level-Up-Karten, optionale Rollen-Belohnungen pro Level, paginiertes Leaderboard mit Zeitraum-Filter (gesamt/Woche/Monat). Zusätzlich: rollen-/kanalbasierte XP-Multiplikatoren und ein server-weiter temporärer XP-Boost (`/xp boost`).

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

10 Einträge/Seite, sortiert nach `totalXp` (bei Zeitraum-Filter nach `periodXp`, siehe 2.8). Medaillen-Emoji für Rang 1–3. `◀`/`▶`-Buttons, deaktiviert wenn keine weitere Seite existiert. Nicknamen Markdown-escaped (gegen Formatierungs-Injection).

### 2.7 XP-Multiplikatoren (Rollen/Kanäle) und Boost

- **`roleMultipliers`**: Pro Rolle konfigurierbarer Faktor (0–10). Hat ein Mitglied mehrere passende Rollen, gilt der **höchste** Multiplikator (nicht gestapelt).
- **`channelMultipliers`**: Pro Kanal konfigurierbarer Faktor (0–10), wirkt als vollständiger Override für diesen Kanal — `0` bedeutet immer "keine XP hier", unabhängig von Rollen-Multiplikator oder aktivem Boost.
- **`/xp boost`**: Server-weiter temporärer XP-Multiplikator (0,1–10×, Dauer frei wählbar z.B. `2h`, `1d`, bis zu 1 Jahr). In der DB persistiert (übersteht Neustart), Hot-Path liest aus einem In-Memory-Cache mit exakt getimtem Ablauf-Timer. Ankündigung erfolgt **nicht ephemeral** — ein aktiver Boost ist für die Community sichtbar. Multiplikatoren aus Rolle/Kanal/Boost wirken multiplikativ zusammen.
- **`/xp boost-status`**: Zeigt aktiven Boost (Faktor, Ablaufzeitpunkt) oder Hinweis, dass keiner aktiv ist. Ephemeral.
- **`/xp boost-stop`**: Beendet einen aktiven Boost vorzeitig. Ephemeral.

### 2.8 Zeitraum-Leaderboards und Season-Reset

`/leaderboard timeframe:week|month` berechnet ein separates Ranking anhand eines pro Periode gespeicherten XP-Snapshots (`periodXp = totalXp - baselineXp`), vollständig im Speicher sortiert. `/xp season-reset timeframe:week|month` postet vorher eine Abschluss-Top-10 als Embed und setzt danach den Zeitraum-Snapshot zurück (Admin, `ManageGuild`).

## 3. Slash-Commands

### `/rank [user]`

Zeigt Rang-Karte als PNG. Optional `user` (Default: du selbst). Jeder darf. Nicht ephemeral.

### `/leaderboard [timeframe] [page]`

XP-Rangliste, 10/Seite. `timeframe`: `Gesamt` (Default), `Diese Woche`, `Dieser Monat`. Optional `page`. Jeder darf.

### `/xp <subcommand>` — Admin-Verwaltung, `ManageGuild`

| Subcommand | Optionen | Beschreibung |
|---|---|---|
| `give` | `user`, `amount` (1–1.000.000) | XP hinzufügen |
| `take` | `user`, `amount` (1–1.000.000) | XP abziehen (min. 0) |
| `set` | `user`, `amount` (0–100.000.000) | Gesamt-XP exakt setzen |
| `reset` | `user` | XP auf 0 |
| `boost` | `multiplier` (0,1–10), `duration` (z.B. `2h`, `1d`) | Server-weiten temporären XP-Boost starten (nicht ephemeral) |
| `boost-status` | — | Aktiven Boost anzeigen |
| `boost-stop` | — | Aktiven Boost vorzeitig beenden |
| `season-reset` | `timeframe` (Woche/Monat) | Zeitraum-Rangliste zurücksetzen (postet vorher Abschluss-Top-10) |

`give`/`take`/`set`/`reset` lösen bei Level-Anstieg dieselbe Level-Up-Logik aus (Karte, Kanal, Rollen-Rewards).

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

roleMultipliers: []         # [{ roleId: "...", multiplier: 1.5 }]
channelMultipliers: []      # [{ channelId: "...", multiplier: 2 }]

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
| `roleMultipliers[].multiplier` | — | XP-Faktor pro Rolle (0–10), höchster gewinnt bei mehreren Treffern |
| `channelMultipliers[].multiplier` | — | XP-Faktor pro Kanal (0–10), `0` = keine XP; überschreibt Rollen-Multiplikator für diesen Kanal |
| `accentColor` | `#5865F2` | Farbe der Karten |

Der aktive `/xp boost` wird separat in der DB verwaltet (kein YAML-Feld) und wirkt multiplikativ zusätzlich zu `roleMultipliers`/`channelMultipliers`.

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
| `/xp give`/`take`/`set`/`reset`/`boost`/`boost-status`/`boost-stop`/`season-reset` | `ManageGuild` |

Der Bot selbst benötigt `Manage Roles` für Rollen-Belohnungen — ohne diese Berechtigung wird der Schritt stillschweigend übersprungen.
