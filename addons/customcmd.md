# GrumpyCore — Custom Commands

## 1. Einleitung

Custom Commands erlauben es Server-Admins, eigene Slash-Befehle zu definieren, ohne den Bot-Code zu ändern. Ein Admin legt mit `/cc add` einen Befehl mit Namen, Typ und frei konfigurierbarer Antwort an; Nutzer rufen ihn über `/cmd <name>` auf.

Das Modul ist **DB-only** (keine YAML-Konfiguration), pro Guild getrennt. Drei Command-Typen:

| Typ | Zweck |
|---|---|
| `text` | Sendet einen Text mit Platzhalter-Ersetzung |
| `embed` | Sendet ein Embed, definiert über ein JSON-Objekt |
| `role-toggle` | Nutzer schaltet sich selbst eine Rolle an/aus |

Jeder Custom Command kann optional auf eine einzelne Rolle beschränkt werden (`allowed-role`).

## 2. Funktionsweise

### 2.1 Datenmodell

| Feld | Beschreibung |
|---|---|
| `guildId` / `name` | Eindeutiger Schlüssel (lowercase, `a-z0-9_-`, max. 32 Zeichen) |
| `type` | `text` \| `embed` \| `role-toggle` |
| `response` | Inhalt — abhängig vom Typ |
| `description` | Optional, erscheint in Autocomplete/`/cc list` |
| `permissions` | `{"roles": [roleId, ...]}` oder `null` (= für alle) |
| `createdById` | Ersteller |
| `uses` | Nutzungszähler |

Namen/Beschreibungen werden vor dem Speichern von unsichtbaren/RTL-Override-Zeichen befreit (Spoofing-Schutz).

### 2.2 Typ `text`

Max. 2000 Zeichen, wird durch das Platzhalter-System gejagt, gesendet mit `allowedMentions: { parse: [] }` (kein `@everyone`-Ping möglich).

```
/cc add name: rules type: text response: "%user_mention%, lies bitte die Regeln in #regeln!"
```

### 2.3 Typ `embed`

`response` muss valides JSON sein:

```json
{"title": "Discord", "description": "Wir sind hier!", "color": "#5865F2"}
```

Mindestens `title` oder `description` als String erforderlich. Ist ein Feld gesetzt aber kein String, wird die Erstellung abgelehnt (verhindert Laufzeit-Crash).

### 2.4 Typ `role-toggle`

`response` = Rollen-ID. Toggle-Verhalten (Rolle an/aus). Sicherheitsprüfungen — sowohl bei Erstellung als auch bei jeder Ausführung (Defense-in-Depth):

| Prüfung | Ablehnung bei |
|---|---|
| Rolle existiert nicht mehr | ja |
| Rolle ist „managed" (Bot/Integration) | ja |
| Rollenposition ≥ höchste Bot-Rolle | ja |
| Rolle hat „gefährliche" Berechtigung | ja |
| Rollenposition ≥ höchste Rolle des Erstellers (nur bei Erstellung) | ja, außer Server-Owner |

„Gefährliche" Berechtigungen, die niemals vergeben werden dürfen: `Administrator`, `ManageGuild`, `ManageRoles`, `ManageChannels`, `ManageWebhooks`, `BanMembers`, `KickMembers`, `ModerateMembers`, `ManageMessages`, `MentionEveryone`.

**Owner-Ausnahme:** Die Hierarchie-Prüfung gegen den Ersteller gilt nicht für den Server-Owner (der steht außerhalb der Rollenhierarchie). Die Bot-Positionsprüfung und Dangerous-Permission-Prüfung gelten aber unverändert.

```
/cc add name: notify type: role-toggle response: "1234567890123456" description: "Toggle Notify-Pings"
```

### 2.5 Platzhalter-System

Format `%name%`, case-insensitive, nur in `text`/`embed`-Feldern. Eingebaut u.a.: `%user_id%`, `%user_name%`, `%user_mention%`, `%user_avatar%`, `%guild_name%`, `%guild_members%`, `%now_date%`, `%now_relative%`. Nicht auflösbare Platzhalter bleiben unverändert stehen.

### 2.6 Sichtbarkeits- und Berechtigungslogik

**Verwaltung** (`/cc add`, `edit`, `delete`): erfordert `ManageGuild`.

**Ausführung** (`/cmd <name>`): ohne Rollenbeschränkung → jeder; mit Beschränkung → nur Mitglieder mit passender Rolle. Korrupte Permission-Einträge → **fail closed** (Ausführung verweigert statt versehentlich offen).

**Autocomplete:** rollenbeschränkte Commands ohne passende Rolle des Nutzers werden ausgeblendet.

**Sichtbarkeit in `/cc list`/`/cc info`:** Admins (`ManageGuild`) sehen alles; andere nur Commands ohne Beschränkung oder mit eigener passender Rolle. Ein für den Aufrufer unsichtbarer Command wird bei `/cc info` als „nicht gefunden" gemeldet (verschleiert auch die Existenz).

## 3. Slash-Commands

### `/cc add <name> <type> <response> [description] [allowed-role]`

| Option | Pflicht | Beschreibung |
|---|---|---|
| `name` | ja | `a-z0-9_-`, max. 32 Zeichen |
| `type` | ja | `text` / `embed (JSON)` / `role-toggle` |
| `response` | ja | Inhalt je nach Typ, max. 2000 Zeichen |
| `description` | nein | max. 80 Zeichen |
| `allowed-role` | nein | Rollenbeschränkung |

### `/cc edit <name> [response] [description] [allowed-role] [clear-permissions]`

Mind. eine Option (außer `name`) nötig. `clear-permissions: true` + `allowed-role` gleichzeitig = Konflikt, wird abgelehnt.

### `/cc delete <name>`

Löscht endgültig. Echte DB-Fehler werden geloggt und gemeldet statt verschluckt.

### `/cc list`

Ephemere, alphabetisch sortierte Liste (gefiltert nach Sichtbarkeit), 🔒-Symbol bei Rollenbeschränkung, zeilenweise Kürzung bei Überlänge.

### `/cc info <name>`

Ephemeres Detail-Embed inkl. vollständigem Response-Inhalt (Code-Block, 900 Zeichen, Backticks entschärft gegen Code-Block-Ausbruch).

### `/cmd <name>`

Führt den Command aus. Autocomplete zeigt nur ausführbare Commands. `uses`-Zähler wird fire-and-forget erhöht. Laufzeitfehler → generische Fehlermeldung ohne Interna.

## 4. Berechtigungen — Zusammenfassung

| Aktion | Erforderliche Berechtigung |
|---|---|
| `/cc add`/`edit`/`delete` | `ManageGuild` |
| `/cc list`/`info` | Jeder — Inhalt pro Nutzer gefiltert |
| `/cmd <name>` | Jeder, sofern keine Rollenbeschränkung; sonst passende Rolle nötig |
| Role-Toggle-Ausführung | Zusätzlich serverseitig blockiert bei gefährlichen Rechten/managed/zu hoher Position |
| Role-Toggle anlegen mit zu hoher Rolle | Blockiert außer für den Server-Owner |

Alle `/cmd`-Antworten nutzen `allowedMentions: { parse: [] }` — kein `@everyone`-Ping möglich, egal was im Response steht.
