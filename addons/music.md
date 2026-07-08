# GrumpyCore — Music-Modul (Internet-Radio)

## 1. Einleitung

Das `music`-Modul ist **kein klassischer Musik-Bot** — keine YouTube/Spotify-Suche, keine Warteschlange, kein Skip. Stattdessen streamt der Bot ausschließlich **vordefinierte Icecast/SHOUTcast-Streams** ("Presets", feste Radiosender wie TechnoBase.FM) dauerhaft in einen Voice-Channel.

Der HTTP-Fetch des Streams ist selbst implementiert (nicht über ffmpeg direkt), weil viele Radio-CDNs den Standard-User-Agent von ffmpeg/prism-media ablehnen. Der Bot schickt einen Browser-User-Agent und reicht den Stream per `StreamType.Arbitrary` an ffmpeg weiter.

## 2. Wie es funktioniert

### 2.1 Presets statt Suche

Alle Sender liegen fest in `configs/modules/music.yml → presets`. Jedes Preset: `id`, `label`, `url`, `description`, `color`. IDs werden lowercased normalisiert, Duplikate beim Laden abgelehnt.

### 2.2 Zwei Modi

**Interactive** (Standard bei `/radio start`): Bot joint den Channel des Aufrufers. Verlässt der letzte Mensch den Channel, trennt der Bot nach `autoLeaveSeconds` (Default 60s) komplett.

**Passive** (`/radio passive enable`, Admin): Bot lebt dauerhaft in einem festen Channel. Bei leerem Channel wird pausiert statt getrennt (`pauseWhenEmpty`, Default true). **Autostart beim Boot**, falls `passive.enabled: true`. `/radio stop` in einer passiven Session stoppt nur temporär — beim nächsten Neustart startet sie automatisch wieder; nur `/radio passive disable` deaktiviert dauerhaft.

### 2.3 Reconnect/Restart bei Fehlern

Zwei Fehlerebenen:
1. **Voice-Verbindung getrennt**: bis zu 5s Warten auf automatische Wiederherstellung, sonst Session-Stop.
2. **Audio-Stream stirbt**: automatischer Neustart mit exponentiellem Backoff (1s, 2s, 4s, 8s, 16s), max. **5 Versuche in 60 Sekunden**. Schlägt auch der letzte Retry fehl, wird die nächste Backoff-Runde automatisch angestoßen (nicht nur einmal versucht). Werden alle Versuche verbraucht, gibt der Bot auf und beendet die Session.

### 2.4 Voice-Channel-Status mit Songtitel

Icecast-Metadaten (`Icy-MetaData: 1`) werden aus dem Audiostream demuxt und bei Titeländerung als Voice-Channel-Status gesetzt (`🎵 Sender · Titel`). Benötigt Bot-Permission `Set Voice Channel Status` — fehlt sie, wird das 5 Minuten pro Guild gecacht (kein Rate-Limit-Spam).

### 2.5 Pause bei leerem Channel

Interactive: kein Pausieren, nur Auto-Leave-Timer. Passive: HTTP-Verbindung wird komplett geschlossen (nicht nur `player.pause()`), damit beim Fortsetzen der aktuelle Live-Punkt statt alter gepufferter Daten kommt.

## 3. Slash-Commands

Basis: `/radio`. Jeder darf starten/stoppen (solange in Voice); Stop erfordert denselben Channel wie der Bot oder `Manage Server`.

| Command | Beschreibung | Optionen | Berechtigung |
|---|---|---|---|
| `/radio start <station>` | Startet Stream im aktuellen Voice-Channel | `station` (Autocomplete) | Jeder, muss in Voice sein |
| `/radio stop` | Stoppt, trennt Verbindung | — | Selbes Voice-Channel oder `ManageGuild` |
| `/radio info` | Sender, Hörerzahl, Status, Uptime | — | Jeder |
| `/radio list` | Alle Presets | — | Jeder |
| `/radio reload` | Lädt `music.yml` neu | — | `ManageGuild` |
| `/radio passive enable <channel> <station>` | Aktiviert Passive-Modus dauerhaft | `channel`, `station` | `ManageGuild` |
| `/radio passive disable` | Deaktiviert dauerhaft | — | `ManageGuild` |
| `/radio passive status` | Zeigt Konfiguration/Live-Status | — | `ManageGuild` |

`/radio start` lehnt Stage-Channels ab, prüft Bot-Permissions (View/Connect/Speak), verhindert Channel-Jacking und wartet bis zu 15s auf `Playing`, bevor Erfolg gemeldet wird.

## 4. Konfiguration (`configs/modules/music.yml`)

Kein `.example.yml` im Repo.

```yaml
enabled: true
autoLeaveSeconds: 60      # 10-3600
defaultVolume: 80         # 0-200, aktuell nicht per Command änderbar

presets:
  - id: technobase
    label: "TechnoBase.FM"
    url: "https://listen.technobase.fm/tunein-mp3"
    description: "Hands Up & Dance"
    color: "#FF1F4A"
  # ... weitere Presets

passive:
  enabled: false
  channelId: "0"          # "0" = Fallback auf channels.radio
  stationId: "technobase"
  pauseWhenEmpty: true
```

| Feld | Default | Bedeutung |
|---|---|---|
| `enabled` | `true` | Modul global an/aus |
| `autoLeaveSeconds` | `60` | Disconnect-Wartezeit im interactive-Modus |
| `defaultVolume` | `80` | Discord-Lautstärkeskala |
| `presets[]` | 7 Sender | Abspielbare Radiosender |
| `passive.enabled` | `false` | Dauerhafter Passive-Modus + Autostart |
| `passive.channelId` | `"0"` | Fallback auf `channels.radio` |
| `passive.pauseWhenEmpty` | `true` | Pausieren statt Dauerlauf bei Leerstand |

## 5. Benötigter radio-Channel

```yaml
addons:
  music: true

channels:
  radio: "CHANNEL_ID"    # Fallback für Passive-Modus, falls music.yml keinen setzt
```

## 6. Bekannte Grenzen

- **Cooldown**: 10s pro Nutzer, gilt für ALLE Subcommands (auch `/radio info`), bewusst nicht guild-weit.
- Kein Skip, keine Warteschlange, kein Crossfade.
- Stage-Channels nicht unterstützt.
- Ein gemeinsamer Preset-Katalog pro Bot-Instanz (nicht pro Guild).
- Max. 5 Restart-Versuche in 60s, danach gibt der Bot auf.
- Songtitel hängt von Sender-Metadaten und Bot-Rechten ab.
- `defaultVolume` aktuell nicht per Command änderbar.
- In-Memory-State: interaktive Sessions gehen bei Neustart verloren (Passive wird aus der Config wiederhergestellt).
