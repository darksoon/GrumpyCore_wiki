# GrumpyCore — Tickets-Modul

## 1. Einleitung

Das Tickets-Modul implementiert ein vollständiges Support-Ticket-System. Nutzer öffnen Tickets über ein gepostetes **Panel** (Embed mit Auswahlmenü oder Buttons), wählen eine **Kategorie**, füllen optional ein **Formular** (Modal) aus und erhalten einen privaten Ticket-Channel. Support kann das Ticket **claimen**, kommunizieren und über einen **Close-Flow** (mit optionalem Grund) schließen. Dabei wird ein **Transcript** (HTML/Text) erzeugt, danach kann der Owner mit 1–5 Sternen **bewerten**. Zusätzlich gibt es einen **Auto-Close-Mechanismus** für inaktive Tickets sowie ein Limit für gleichzeitig offene Tickets pro Nutzer.

Mehrere Panels gleichzeitig möglich, jedes mit beliebig vielen Kategorien (eigene Rollen, Formular, Willkommensnachricht, Auto-Close-Timeout).

Zwei Command-Sets: **`/tickets`** (Administration) und **`/ticket`** (In-Ticket-Aktionen).

## 2. Wie es funktioniert

### 2.1 Ticket öffnen — kompletter Ablauf

1. **Panel-Klick**: Nutzer wählt eine Kategorie (Select-Menü oder Button).
2. **Doppelklick-Sperre**: Per-User-Lock (TTL 60s) verhindert parallele Erstellung.
3. **`maxOpenPerUser`-Prüfung** (vor dem Formular): ist das Limit in dieser Panel/Kategorie erreicht, bricht der Vorgang ab.
4. **Formular oder Direkterstellung**: Ist `form.enabled: true`, wird der Lock freigegeben (ein Modal kann ~15 Min. offen bleiben) und ein Modal mit bis zu 5 Textfeldern gezeigt. Ohne Formular wird direkt erstellt.
5. **Erstellung**: erneute `maxOpenPerUser`-Prüfung, DB-Zeile mit Platzhalter-ID anlegen, Discord-Channel erstellen (`@everyone` gesperrt, Bot + Owner + Staff-Rollen erhalten Zugriff), DB-Zeile mit echter Channel-ID aktualisieren, Willkommens-Embed + Claim/Close-Buttons posten, Staff-Rollen pingen, Log-Event schreiben.
6. Schlägt die Channel-Erstellung fehl, wird die DB-Zeile wieder gelöscht.

### 2.2 Claim-Flow

- Nur sichtbar, wenn `category.claimable: true`.
- Nur Mitglieder mit einer Rolle aus `staffRoleIds` dürfen claimen.
- Claim erfolgt **atomar** — klicken zwei Staff gleichzeitig, gewinnt nur einer.
- Unclaim nur durch den aktuellen Claimer.

### 2.3 Close-Flow

Auslöser: Close-Button (Modal mit optionalem Grund), `/ticket close [reason]`, oder `/tickets force-close <id> [reason]` (Admin, auch von außerhalb).

Ablauf:
1. **Atomarer Statuswechsel** open → closed.
2. Log-Event.
3. **Transcript** (falls aktiviert): alle Nachrichten abgeholt (max. 1000, gedeckelt), als HTML und/oder Text gerendert, in Transcript-/Log-Channel gepostet.
4. **Rating-Prompt** (falls aktiviert), asynchron, blockiert den Close nicht.
5. Hinweis im Channel, Löschung nach 5 Sekunden.

Beim Bot-Start werden Channels bereinigt, die als „closed" markiert sind, aber (z.B. durch Absturz während der Löschverzögerung) noch existieren.

### 2.4 Rating-Flow

- Nach dem Schließen (falls `rating.enabled`) bekommt der Owner ein 5-Sterne-Embed — standardmäßig per DM.
- Nur der Owner darf bewerten, Speichern erfolgt atomar.
- Rating wird ins Log gepostet.

### 2.5 Auto-Close-Mechanik

Läuft automatisch, wenn Modul + `autoClose.enabled` aktiv sind und die Kategorie `autoCloseAfterHours > 0` hat:

- **Aktivitäts-Tracking**: jede Nicht-Bot-Nachricht aktualisiert `lastActivityAt`.
- **Scan-Intervall**: `autoClose.checkIntervalMinutes` (Default 30 Min.), läuft einmal sofort beim Start.
- **Optionale Vorwarnung**: `autoClose.warnBeforeMinutes` postet eine Warnung vor dem Close.
- Ausführung über denselben Close-Flow wie manuell, Grund „auto-close", `closedBy: null`.
- Deaktivieren: global über `autoClose.enabled: false`, pro Kategorie über `autoCloseAfterHours: 0`.

### 2.6 `maxOpenPerUser`-Limit

Begrenzt gleichzeitig offene Tickets pro Nutzer in einer Panel/Kategorie-Kombination (0 = unbegrenzt, Default 1). Zweimal geprüft (beim Klick und unmittelbar vor Erstellung) gegen Race-Conditions durch lang offene Modals.

## 3. Slash-Commands

### `/tickets` (Administration, `ManageGuild`)

| Subcommand | Optionen | Beschreibung |
|---|---|---|
| `setup-panel` | `panel-id` | Postet ein Panel erneut, speichert Message-ID |
| `history` | `user` | Zeigt kompletten Ticket-Verlauf eines Nutzers |
| `force-close` | `id`, `reason` optional | Schließt von überall aus |
| `list-panels` | — | Alle Panels mit Status/Channel/Layout |
| `set-channel` | `feature` (Transcript channel), `channel` | Setzt Transcript-Channel |
| `reopen` | `id` | Erstellt für geschlossenes Ticket neuen Channel |
| `show` | — | Übersicht aller Ticket-Einstellungen |
| `reload` | — | Lädt `tickets.yml` neu, startet Auto-Close-Runner neu |

### `/ticket` (In-Ticket-Aktionen)

| Subcommand | Optionen | Berechtigung |
|---|---|---|
| `close` | `reason` optional | Owner oder Kategorie-Staff |
| `claim` | — | Kategorie-Staff, nur wenn `claimable: true` |
| `unclaim` | — | Nur der aktuelle Claimer |
| `info` | — | Jeder im Ticket |
| `transfer` | `category` | Kategorie-Staff |

### Ticket-Buttons (im Ticket-Channel)

Claim/Unclaim (nur bei `claimable: true`) und Close (öffnet Modal mit optionalem Grund).

## 4. Panel-Setup

Panels sind vollständig in `configs/modules/tickets.yml` definiert — kein interaktiver Wizard:

1. Unter `panels:` einen Eintrag anlegen: `id`, `channelId` (`"0"` = Fallback auf `channels.support`), Embed, `layout` (`select-menu`/`buttons`).
2. Unter `categories:` mindestens eine Kategorie: `id`, `label`, `parentChannelId` (Pflicht!), `staffRoleIds`, `pingRoleIds`, `maxOpenPerUser`, `claimable`, `welcomeMessage`, `autoCloseAfterHours`, optional `form`.
3. `/tickets reload` ausführen.
4. Beim Boot werden alle aktivierten Panels automatisch gepostet. `/tickets setup-panel <id>` für manuelles Re-Deploy.

Grenzen: max. **25 Kategorien pro Panel** (Discord-Limit), max. **5 Formularfelder pro Kategorie**.

## 5. Konfiguration (`configs/modules/tickets.yml`)

```yaml
enabled: true
transcriptChannelId: "0"       # "0" = Fallback auf channels.ticket-log
namingFormat: "ticket-{user}-{id}"

transcripts:
  enabled: true
  format: html                 # html | text | both

autoClose:
  enabled: true
  checkIntervalMinutes: 30     # 5-1440
  warnBeforeMinutes: 0         # 0-1440, 0 = keine Warnung

rating:
  enabled: true
  dmOwner: true
  promptTitle: "How was your support experience?"
  promptMessage: "Your ticket **#{{id}}** in **{{guild}}** was just closed..."
  thanksMessage: "Thanks for your feedback! ⭐"

panels:
  - id: support
    enabled: true
    channelId: "0"
    embed:
      title: "🎫 Need help?"
      description: "Click the menu below to open a ticket."
      color: "#5865F2"
    layout: select-menu
    placeholder: "Choose a category..."
    categories:
      - id: general
        label: "General Support"
        emoji: "🎫"
        description: "General questions or help"
        parentChannelId: "0"     # Pflicht für echten Betrieb!
        staffRoleIds: []
        pingRoleIds: []
        maxOpenPerUser: 1
        claimable: true
        welcomeMessage: "Hello %user_mention%! Staff will be with you shortly."
        autoCloseAfterHours: 0
        form:
          enabled: true
          title: "Open General Support"
          fields:
            - id: issue
              label: "Describe your issue"
              style: paragraph
              required: true
              minLength: 10
              maxLength: 1000
              placeholder: "What's going on?"
```

Wichtige Felder je Kategorie: `parentChannelId` ist Pflicht (Discord-Kategorie für neue Channels), `staffRoleIds` bestimmt wer sehen/claimen/schließen darf, `pingRoleIds` wer beim Öffnen benachrichtigt wird.

## 6. Benötigte Channels/Rollen

- `configs/config.yml → channels.support` — Standard-Panel-Ziel (Fallback bei `channelId: "0"`)
- `configs/config.yml → channels.ticket-log` — zentrales Log (open/close/claim/transfer, Rating), Fallback für Transcripts
- `configs/modules/tickets.yml → transcriptChannelId` — eigener Transcript-Channel (optional)
- `configs/modules/tickets.yml → panels[].categories[].parentChannelId` — **Pflicht** pro Kategorie

Rollen sind ausschließlich pro Kategorie definiert (`staffRoleIds`, `pingRoleIds`) — es gibt keine globale „Support-Rolle" auf Modul-Ebene.

## 7. Berechtigungen — Übersicht

| Aktion | Wer darf es |
|---|---|
| Panel posten, force-close, Verlauf, reopen, Settings | `ManageGuild` (Server verwalten) |
| Claim/Unclaim | `staffRoleIds` der Kategorie, nur wenn `claimable: true` |
| Ticket schließen | Owner oder `staffRoleIds` der Kategorie |
| Transfer | `staffRoleIds` der **aktuellen** Kategorie |
| Info | Jeder mit Zugriff auf den Channel |
| Bewerten | Ausschließlich der Ticket-Owner |
