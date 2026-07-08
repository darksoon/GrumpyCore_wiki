# GrumpyCore — Voice-Modul (Join-to-Create)

## 1. Einleitung

Das Voice-Modul ("Join-to-Create") stellt ein automatisiertes System temporärer, benutzereigener Voice-Channels bereit. Statt feste Voice-Channels vorzuhalten, betritt ein Nutzer einen zentralen **Lobby-Channel**, woraufhin der Bot einen neuen, persönlichen Voice-Channel erstellt, den Nutzer hinein verschiebt und ihn als **Owner** einträgt. Der Owner erhält erweiterte Rechte über `/voice`-Subcommands (umbenennen, sperren, Limit setzen, kicken, vertrauen, Besitz übertragen). Ist der Channel eine Zeit lang leer, wird er automatisch gelöscht.

## 2. Wie es funktioniert

### 2.1 Lobby-Join → Channel-Erstellung

Reagiert, sobald ein Nutzer den konfigurierten Lobby-Channel betritt. Ablauf:

1. **Creation-Lock** pro Nutzer verhindert doppelte Channel-Erstellung bei schnellem Doppel-Join (Sicherheitsnetz-Timer: 60s TTL).
2. **Limit-Prüfung** (`maxChannelsPerUser`): Besitzt der Nutzer schon das Maximum, wird er in seinen bestehenden Channel verschoben, oder — falls dieser nicht mehr existiert — die verwaiste DB-Zeile gelöscht und neu erstellt. Bei transienten Fehlern wird abgebrochen, ohne etwas zu löschen (verhindert Datenverlust/Doppel-Channels).
3. **Ziel-Kategorie:** konfigurierbar, sonst Parent der Lobby.
4. **Channel-Name** aus Template mit Platzhaltern.
5. **Berechtigungs-Overwrites** werden bei Erstellung gesetzt (siehe Abschnitt 6).
6. Channel wird angelegt, **DB-Eintrag zuerst** (Rollback bei DB-Fehler löscht den Discord-Channel wieder), dann wird der Nutzer verschoben.

### 2.2 Owner-Konzept

Jeder Channel wird in der Tabelle `VoiceChannel` (Owner, Trusted-IDs als JSON-Array, Lock-Status, Limit) verwaltet. Der Owner kann umbenennen, sperren/entsperren, Limit setzen, kicken, vertrauen/misstrauen und übertragen. Ein Nicht-Owner kann den Besitz **claimen**, sofern der aktuelle Owner den Channel verlassen hat (Anti-Hijacking-Schutz).

Bei `transfer`/`claim` wird der alte Owner-Overwrite **vollständig gelöscht** — sonst würde sein ursprünglicher Connect-Allow-Overwrite einen späteren Lock aushebeln.

### 2.3 Automatisches Löschen bei Leerstand

Verlässt der letzte menschliche Nutzer den Channel, startet ein Timer (`emptyDeleteSeconds`). Ein Re-Join innerhalb der Frist bricht den Timer ab. Owner-Leave triggert **keinen** automatischen Besitzwechsel — dafür ist `/voice claim` nötig.

**Restart-Cleanup:** Beim Bot-Start werden verwaiste/leere getrackte Channels bereinigt (per `fetch()`, nicht Cache, da der nach Neustart unvollständig sein kann).

### 2.4 Creation-Lock gegen Doppel-Erstellung

Pro-Nutzer-Lock mit eindeutigem Token verhindert, dass ein langsamer erster Durchlauf das Lock eines zweiten, späteren Durchlaufs löscht. Zusätzlich gibt es einen **Rename-Cooldown** (5 Minuten) als Puffer unter Discords Rate-Limit (2 Renames/10 Min pro Channel).

## 3. Slash-Commands

Alle Subcommands (außer `reload`) erfordern, dass der Nutzer **aktuell in einem verwalteten Voice-Channel** ist.

| Command | Beschreibung | Optionen | Berechtigung |
|---|---|---|---|
| `/voice rename <name>` | Benennt den eigenen Channel um | `name` (1–100 Zeichen) | Owner |
| `/voice lock` | Sperrt den Channel (`@everyone` → Connect: false); Trusted bleiben zugelassen | — | Owner |
| `/voice unlock` | Entsperrt den Channel | — | Owner |
| `/voice limit <count>` | Setzt das Nutzerlimit | `count` (0–99, 0 = kein Limit) | Owner |
| `/voice kick <user>` | Wirft einen Nutzer aus dem Channel | `user` | Owner |
| `/voice trust <user>` | Erlaubt Beitritt trotz Lock | `user` | Owner |
| `/voice untrust <user>` | Entzieht Trust wieder | `user` | Owner |
| `/voice transfer <user>` | Überträgt Ownership | `user` (muss im Channel sein) | Owner |
| `/voice claim` | Übernimmt einen Channel ohne anwesenden Owner | — | Jedes Mitglied im Channel |
| `/voice info` | Zeigt Owner, Mitglieder, Limit, Lock-Status, Bitrate, Trust-Liste | — | Jedes Mitglied |
| `/voice reload` | Lädt `voice.yml` neu | — | `ManageGuild` |

Details: `rename` ist deferred (Rate-Limit-Queue kann Minuten dauern); `trust`/`untrust` laufen atomar über eine Prisma-Transaktion (Lost-Update-Schutz); `transfer`/`claim` setzen denselben vollen Owner-Overwrite-Satz.

## 4. Konfiguration (`configs/modules/voice.yml`)

Kein `.example.yml` im Repository — wird beim ersten Start automatisch mit Defaults erzeugt.

```yaml
enabled: true
lobbyChannelId: "0"     # "0" = Fallback auf channels.voice-lobby
categoryId: "0"         # "0" = Fallback auf Parent-Kategorie der Lobby
nameTemplate: "🎮 %user%"
defaultLimit: 0         # 0 = unlimitiert, sonst 1-99
emptyDeleteSeconds: 30  # 5-600
maxChannelsPerUser: 1   # 1-5
defaultBitrate: 64      # kbps, 8-384 (begrenzt durch Guild-Boost-Level)
```

| Schlüssel | Default | Beschreibung |
|---|---|---|
| `enabled` | `true` | Modul aktiv/inaktiv |
| `lobbyChannelId` | `"0"` | Lobby-Channel-ID; Fallback auf `channels.voice-lobby` |
| `categoryId` | `"0"` | Ziel-Kategorie; Fallback auf Lobby-Parent |
| `nameTemplate` | `"🎮 %user%"` | Namensvorlage mit Platzhaltern |
| `defaultLimit` | `0` | Standard-Nutzerlimit |
| `emptyDeleteSeconds` | `30` | Wartezeit vor Auto-Löschung |
| `maxChannelsPerUser` | `1` | Max. gleichzeitig besessene Channels |
| `defaultBitrate` | `64` | Standard-Bitrate in kbps |

## 5. Benötigter Lobby-Channel

```yaml
addons:
  voice: true

channels:
  voice-lobby: "CHANNEL_ID"
```

`channels.voice-lobby` ist der Fallback, falls `voice.yml → lobbyChannelId` auf `"0"` steht. Eine explizit gesetzte `lobbyChannelId` hat Vorrang.

## 6. Berechtigungen

### Discord-Permission-Overwrites (technisch)

| Rolle/Nutzer | Overwrite |
|---|---|
| Owner | Allow: ViewChannel, Connect, Speak, ManageChannels, MoveMembers, MuteMembers, DeafenMembers, PrioritySpeaker, Stream, UseVAD |
| Bot | Allow: ViewChannel, Connect, ManageChannels, MoveMembers |
| Trusted | Allow: ViewChannel, Connect |
| `@everyone` bei Lock | Deny: Connect |

### Funktionale Matrix

| Aktion | Owner | Trusted | Normaler Nutzer |
|---|---|---|---|
| Beitritt bei Lock | ja | ja | nein |
| rename/lock/limit/kick/trust/transfer | ja | nein | nein |
| claim | nein (bereits Owner) | ja (falls Owner abwesend) | ja (falls Owner abwesend) |
| info | ja | ja | ja |
| reload | nur mit `ManageGuild` | — | — |

Der Bot selbst benötigt `Manage Channels` und `Move Members` in der Zielkategorie/Guild.
