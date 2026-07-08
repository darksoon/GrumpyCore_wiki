# GrumpyCore — Suggestions-Modul

## 1. Einleitung

Erlaubt Mitgliedern, Vorschläge per Slash-Command einzureichen. Jeder Vorschlag wird als Embed im Vorschlagskanal gepostet und per Emoji-Reaktion (👍/👎, konfigurierbar) bewertet — **keine Buttons für das Voting**. Optional öffnet der Bot automatisch einen Diskussions-Thread. Staff kann den Status ändern (Embed wird live aktualisiert, optionale DM an den Autor). Cooldown pro Nutzer gegen Spam.

## 2. Wie es funktioniert

### 2.1 Submit-Ablauf

1. Längenprüfung gegen `minLength`/`maxLength`.
2. Ziel-Kanal: `channelId` (Modul-Setting) → Fallback `channels.suggestions`.
3. **Race-Schutz**: In-Flight-Sperre pro Nutzer verhindert parallele Submits vor der Cooldown-Prüfung.
4. Cooldown-Prüfung.
5. Erstellung: DB-Eintrag (Status `pending`), Embed posten, Vote-Emojis anhängen, optional Thread öffnen (best-effort), `messageId` zurückschreiben. Schlägt das Posten fehl, wird der DB-Eintrag wieder gelöscht.
6. Cooldown wird **erst nach Erfolg** markiert.

### 2.2 Voting-Mechanik

Ausschließlich über Emoji-Reaktionen auf die Nachricht. Stimmen werden **nicht** in der DB gespeichert, sondern live aus den Discord-Reaktionszählern gelesen (die Bot-eigene Initialreaktion wird abgezogen).

### 2.3 Status-System

Fünf Status: `pending` (⏳ Gelb), `approved` (✅ Grün), `denied` (❌ Rot), `in-progress` (🔄 Blau), `implemented` (💡 Lila).

Beim Statuswechsel: DB-Update, Embed live neu gerendert, optional DM an den Autor (`dmOnStatusChange`, Default true). Eine mitgegebene `note` überschreibt die vorhandene; ohne `note` bleibt eine bestehende Notiz erhalten.

### 2.4 Cooldown

Konfiguriert über `cooldownMinutes` (Default 60, `0` = aus). Rein In-Memory, geht bei Neustart verloren. Wird erst nach erfolgreichem Anlegen gesetzt.

## 3. Slash-Commands

### `/suggestion submit <text>`

Text-Länge begrenzt durch `minLength`/`maxLength`. Jeder darf (mit Cooldown).

### `/suggestion status <id> <status> [note]`

Ändert den Status. Berechtigung: `ManageGuild` **oder** Rolle aus `staffRoles`.

### `/suggestion list [status]`

Letzte 15 Vorschläge, optional gefiltert. Jeder darf.

### `/suggestion top [limit]`

Rankt die letzten 50 Vorschläge (mit gesetzter `messageId`) nach Score (Up- minus Downvotes, live gelesen). `limit` (1–25, Default 10). Jeder darf.

## 4. Konfiguration (`configs/modules/suggestions.yml`)

Kein `.example.yml` im Repo.

```yaml
enabled: true
channelId: ""                    # Fallback: channels.suggestions
cooldownMinutes: 60              # 0-1440, 0 = kein Cooldown
minLength: 10                    # 1-500
maxLength: 500                   # 10-2000
voteUpEmoji: "👍"
voteDownEmoji: "👎"
dmOnStatusChange: true
staffRoles: []                   # zusätzlich zu ManageGuild
threadEnabled: true
threadAutoArchiveMinutes: 1440   # 60 | 1440 | 4320 | 10080
```

| Feld | Default | Bedeutung |
|---|---|---|
| `enabled` | `true` | Modul an/aus |
| `channelId` | `""` | Zielkanal (Vorrang vor `channels.suggestions`) |
| `cooldownMinutes` | `60` | Minuten zwischen Einreichungen |
| `minLength`/`maxLength` | 10/500 | Textlängen-Grenzen |
| `voteUpEmoji`/`voteDownEmoji` | 👍/👎 | Vote-Reaktionen |
| `dmOnStatusChange` | `true` | Autor per DM benachrichtigen |
| `staffRoles` | `[]` | Zusätzliche Rollen für `/suggestion status` |
| `threadEnabled` | `true` | Auto-Thread pro Vorschlag |
| `threadAutoArchiveMinutes` | `1440` | Archivierungsdauer |

## 5. Benötigter Vorschlagskanal

```yaml
channels:
  suggestions: "CHANNEL_ID"
```

Auflösung: `channelId` (Modul) → `channels.suggestions` → Fehlermeldung.

## 6. Berechtigungen

| Aktion | Wer darf |
|---|---|
| `submit`, `list`, `top` | Jedes Mitglied |
| `status` (inkl. Notiz) | `ManageGuild` oder Rolle aus `staffRoles` |
