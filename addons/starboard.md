# GrumpyCore — Starboard-Modul

## 1. Einleitung

Automatisches "Best-of"-Board: sammelt eine Nachricht genug Reaktionen mit einem konfigurierten Emoji (Standard ⭐), postet der Bot sie als Embed in einem Starboard-Channel. **Keine Slash-Commands** — vollständig reaktionsgetrieben.

> Im main-Branch existieren keine `/starboard`-Commands. Ein separater, nicht gemergter Feature-Branch enthält möglicherweise Arbeiten an `/starboard top`/`stats`/`remove` — nicht Teil dieses Standes.

## 2. Wie es funktioniert

### Trigger und Events

- `MessageReactionAdd`/`Remove` — Standardfall.
- `MessageReactionRemoveAll` — ein Mod entfernt alle Reaktionen auf einmal (kein per-User-Event) → Eintrag wird direkt gelöscht.
- `MessageReactionRemoveEmoji` — gezieltes Entfernen des Starboard-Emojis.
- `MessageDelete` — Original gelöscht → verwaister Starboard-Post wird aufgeräumt.

### Schwellenwert-Logik

Bei jedem Event: Modul aktiv? Ziel-Channel bekannt (`channelId` → Fallback `channels.starboard`)? Ursprungs-Channel nicht in `ignoredChannels`? Emoji korrekt? Autor kein Bot (`ignoreBots`)? Self-Star (`selfStarAllowed: false`) wird automatisch entfernt und zählt nicht. Erreicht/überschreitet die Zahl `threshold`, wird gepostet/aktualisiert; fällt sie darunter, wird der Post gelöscht (DB-Zeile bleibt mit `starboardMsgId: null`).

### Duplikat-Schutz

In-Memory-Lock pro Nachrichten-ID verhindert doppelte Posts bei praktisch gleichzeitigen Reaktions-Events.

### Verhalten bei Update-Fehlern

- **Discord-Fehler 10008 (Unknown Message)**: Post wurde extern gelöscht → automatisch neu gepostet, neue ID gespeichert.
- **Alle anderen Fehler** (Rechte, transiente 5xx): **kein** Neu-Post (Duplikat-Vermeidung), nächster Reaktions-Event versucht erneut.

### Starboard-Embed

Autor-Avatar/-Name, Original-Text (2048 Zeichen), erstes Bild-Attachment, Link zur Original-Nachricht, Footer mit Emoji/Zähler/Channel.

## 3. Slash-Commands

**Keine im main-Branch.** Einzige indirekte Erwähnung: `/botstatus` zeigt die Gesamtzahl der Starboard-Einträge als generische Statistik.

## 4. Konfiguration (`configs/modules/starboard.yml`)

Kein `.example.yml` im Repo.

| Setting | Default | Beschreibung |
|---|---|---|
| `enabled` | `true` | Modul an/aus |
| `channelId` | `""` | Ziel-Channel, Fallback `channels.starboard` |
| `emoji` | `⭐` | Auslösendes Emoji (Unicode oder Custom, inkl. animiert) |
| `threshold` | `3` | Mindest-Reaktionen (1–100) |
| `ignoredChannels` | `[]` | Ausgeschlossene Kanäle |
| `ignoreBots` | `true` | Bot-Autoren ignorieren |
| `selfStarAllowed` | `false` | Eigene ⭐-Reaktion zählt nicht (wird entfernt) |

## 5. Benötigter Starboard-Channel

```yaml
addons:
  starboard: true

channels:
  starboard: "CHANNEL_ID"
```

## 6. Berechtigungen

Keine Slash-Commands, also keine Command-Permissions. Der **Bot** benötigt: `View Channel` + `Read Message History` in überwachten Kanälen, `Send Messages` + `Embed Links` im Starboard-Channel, `Manage Messages` in überwachten Kanälen (für automatisches Entfernen der Self-Star-Reaktion — ohne diese Berechtigung bleibt der Self-Star sichtbar, zählt aber weiterhin nicht).
