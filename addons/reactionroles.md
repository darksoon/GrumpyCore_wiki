# GrumpyCore — Reaction-Roles-Modul

## 1. Einleitung

Das Modul `reactionroles` erlaubt Servermitgliedern die **Selbst-Zuweisung von Rollen** über interaktive Panels. Trotz des Namens funktioniert es **nicht** über klassische Emoji-Reactions, sondern vollständig über Discord-Message-Components: **Buttons** oder **String-Select-Dropdowns**.

Ein Admin richtet ein Panel ein (Embed mit Buttons/Dropdowns), das im Server gepostet wird. Mitglieder klicken/wählen dort die gewünschte Rolle und erhalten sie sofort — als **Toggle**: erneuter Klick entfernt die Rolle wieder.

## 2. Wie es funktioniert

### 2.1 Hierarchie: Panel → Gruppe → Rollen-Eintrag

| Ebene | Bedeutung |
|---|---|
| **Panel** | 1 Discord-Embed-Nachricht in einem Kanal — Titel, Beschreibung, Farbe, Ziel-Kanal |
| **Gruppe** | ein Abschnitt im Panel — Darstellung als Buttons **oder** Dropdown, `maxSelections`, optionale Pflicht-Rolle |
| **Rollen-Eintrag** | eine wählbare Rolle innerhalb einer Gruppe — Rolle, Label, Emoji, Beschreibung, Button-Farbe |

Discord begrenzt eine Nachricht auf max. **5 Action Rows** — ein Panel darf daher max. **5 Gruppen** haben.

### 2.2 Exklusive vs. mehrfache Auswahl (`maxSelections`)

| Wert | Verhalten |
|---|---|
| `0` | Unbegrenzt — beliebig viele Rollen der Gruppe gleichzeitig |
| `1` | **Exklusiv/Swap** — alte Rolle der Gruppe wird automatisch entfernt, bevor die neue vergeben wird |
| `2`–`25` | **Multi-Select** bis zum Limit |

### 2.3 Voraussetzungs-Rolle

Eine Gruppe kann eine Pflicht-Rolle festlegen — ohne diese wird jede Aktion in der Gruppe abgelehnt, geprüft bei jedem Interaktions-Aufruf.

### 2.4 Ablauf bei Klick/Auswahl

1. **Interaction-Lock** pro (User, Gruppe) verhindert Race-Conditions gegen `maxSelections`.
2. Anti-Spoofing: die `roleId` aus der Custom-ID muss tatsächlich zur Gruppe gehören.
3. Pflicht-Rollen-Check.
4. Bot-Berechtigung (`ManageRoles`) + **Hierarchie-Check** (Rolle muss unter der höchsten Bot-Rolle liegen).
5. Hat das Mitglied die Rolle bereits → Toggle off.
6. Sonst: bei `maxSelections === 1` erst alte Rollen entfernen, dann neue vergeben; bei Limit-Überschreitung Ablehnung.

Beim Dropdown läuft es als Delta-Abgleich (aktuelle Auswahl vs. gehaltene Rollen der Gruppe → `toAdd`/`toRemove`).

### 2.5 Externe Löschung

Wird die Panel-Nachricht manuell gelöscht, erkennt ein Listener das und setzt `messageId` in der DB zurück — ein erneutes `panel deploy` postet automatisch neu.

## 3. Slash-Commands

Basis: `/reactionrole`, Subcommand-Gruppen `panel`, `group`, `role`.

### `panel`

| Subcommand | Beschreibung | Optionen |
|---|---|---|
| `create` | Neues Panel | `channel`, `title` (max 256), `description` (optional), `color` (optional) |
| `edit` | Titel/Beschreibung/Farbe ändern, redeployt automatisch | `panel-id`, `title`/`description`/`color` optional |
| `delete` | Panel + Nachricht löschen | `panel-id` |
| `deploy` | Posten oder aktualisieren | `panel-id` |
| `list` | Alle Panels anzeigen | — |
| `info` | Details mit Gruppen/Rollen/Status | `panel-id` |

### `group`

| Subcommand | Beschreibung | Optionen |
|---|---|---|
| `add` | Neue Gruppe (max. 5 pro Panel) | `panel-id`, `style` (`buttons`/`select`), `label`, `max` (0–25), `required-role`, `placeholder` |
| `remove` | Gruppe inkl. Rollen entfernen | `panel-id`, `group-id` |

### `role`

| Subcommand | Beschreibung | Optionen |
|---|---|---|
| `add` | Rolle zu Gruppe hinzufügen (max. 25 pro Gruppe) | `panel-id`, `group-id`, `role`, `label` (max 80), `emoji`, `description` (nur Dropdown), `style` (nur Buttons) |
| `remove` | Rolle entfernen | `panel-id`, `group-id`, `role` |

Typischer Ablauf: `panel create` → `group add` → `role add` → `panel deploy`. Nach jeder weiteren Änderung ist erneutes `deploy` nötig.

## 4. Berechtigungen

- `/reactionrole` erfordert **`ManageRoles`** (Rollen verwalten).
- Der **Bot** benötigt `ManageRoles` + ausreichende Hierarchie, sowie `ViewChannel`+`SendMessages` im Zielkanal.

## 5. Bekannte Grenzen

| Grenze | Herkunft | Verhalten |
|---|---|---|
| Max. 25 Rollen pro Gruppe | Discord-Limit | Serverseitig erzwungen bei `role add` |
| Max. 5 Gruppen pro Panel | Discord-Limit (5 Action Rows) | Bei `group add` geprüft |
| Rollen-Hierarchie | Discord-API | Geprüft bei `role add` UND bei jedem Klick/jeder Auswahl |
| Managed Roles | Bot-/Integrationsrollen | `role add` lehnt sie ab |
| Guild-Zugehörigkeit | Panel-/Gruppen-IDs sind globale Strings | Jeder Zugriffspfad prüft explizit `guildId`-Match |
| Race Conditions | Parallele Klicks | In-Memory-Lock pro (User, Gruppe) |
| Modul-Toggle | `enabled` in `reactionroles.yml` | Deaktiviert lehnt Command + Handler jede Aktion ab |
