# Berechtigungen — Wer darf was

GrumpyCore prüft Berechtigungen auf zwei Ebenen: native **Discord-Permissions** (z.B. `Moderate Members`) und — bei einigen Modulen zusätzlich — konfigurierbare **Staff-Rollen** (z.B. `roles.support`, `suggestions.yml → staffRoles`).

---

## Übersicht nach Modul

| Command / Aktion | Erforderliche Berechtigung |
|---|---|
| `/mod warn` | `Moderate Members` |
| `/mod kick` | `Moderate Members` + `Kick Members` |
| `/mod ban` / `tempban` / `softban` / `unban` | `Moderate Members` + `Ban Members` |
| `/mod mute` / `unmute` / `unwarn` / `note` / `history` | `Moderate Members` |
| `/mod clear` / `purge` | `Moderate Members` + `Manage Messages` |
| `/mod slowmode` | `Moderate Members` + `Manage Channels` |
| `/mod reload` | `Manage Guild` |
| `/report` (einreichen) | Jeder |
| Report-Buttons „Deny"/„Warn" | `Moderate Members` **oder** Rolle aus `roles.support` |
| Report-Buttons „Timeout"/„Kick"/„Ban" | zusätzlich die jeweils passende Permission |
| `/welcome *` (alle Subcommands) | `Manage Guild` |
| `/tickets *` (Administration) | `Manage Guild` |
| `/ticket claim`/`unclaim`/`transfer` | Rolle aus der Kategorie-`staffRoleIds` |
| `/ticket close` | Ticket-Owner oder Kategorie-Staff |
| `/voice rename`/`lock`/`limit`/`kick`/`trust`/`transfer` | Channel-Owner |
| `/voice claim` | Jeder im Channel, nur wenn Owner abwesend |
| `/voice reload` | `Manage Guild` |
| `/radio start`/`stop`/`info`/`list` | Jeder (in Voice) |
| `/radio reload`, `/radio passive *` | `Manage Guild` |
| `/help redeploy`/`reload` | `Manage Guild` |
| `/cc add`/`edit`/`delete` | `Manage Guild` |
| `/cmd <name>` | Jeder, außer rollenbeschränkt |
| `/reactionrole *` | `Manage Roles` |
| `/xp give`/`take`/`set`/`reset` | `Manage Guild` |
| `/rank`, `/leaderboard` | Jeder |
| `/suggestion submit`/`list`/`top` | Jeder |
| `/suggestion status` | `Manage Guild` oder Rolle aus `staffRoles` |
| `/poll create`/`end` | `Manage Messages` |
| `/remind add`/`list`/`cancel` | Jeder (nur eigene Erinnerungen) |
| `/birthday set`/`remove`/`upcoming` | Jeder (nur eigener Geburtstag) |
| `/birthday reload` | `Manage Guild` |

Details je Modul stehen auf der jeweiligen Modul-Wiki-Seite unter `addons/`.

---

## Wichtige Regel: Rollen-Hierarchie (Mod-Modul)

Beim Moderieren (`/mod`, Report-Buttons) gilt immer:

- Man kann nur Mitglieder moderieren, deren höchste Rolle **niedriger** ist als die eigene.
- Der **Bot** muss ebenfalls über der Zielrolle stehen — sonst schlägt die Aktion fehl, unabhängig von der eigenen Berechtigung.
- Der **Server-Owner** ist von der eigenen Rollen-Hierarchie-Prüfung ausgenommen (kann jeden moderieren außer sich selbst/Bots), muss aber trotzdem über der Bot-Hierarchie liegen.

Das wird vom Bot automatisch geprüft — es gibt keine Möglichkeit, das über Discord-Permissions zu umgehen.

---

## Discord-Integrationseinstellungen

**Server-Einstellungen → Integrationen → GrumpyCore**

Dort lässt sich pro Command zusätzlich einstellen:
- Welche Rollen ihn sehen/nutzen dürfen (zusätzliche Einschränkung über die im Bot fest programmierten Mindest-Permissions hinaus)
- In welchen Channels er verfügbar ist

Das ist eine reine Discord-seitige Sichtbarkeits-/Zugriffssteuerung, unabhängig von den im Bot-Code hinterlegten Mindestanforderungen.

---

## Support-Rollen einrichten

In `configs/config.yml`:

```yaml
roles:
  support:
    - "SUPPORTER_ROLLEN_ID"
    - "MOD_ROLLEN_ID"
    - "ADMIN_ROLLEN_ID"
```

Diese Rollen dürfen:
- Tickets claimen/schließen (sofern zusätzlich in der jeweiligen Ticket-Kategorie unter `staffRoleIds` gelistet — die globale `roles.support` gilt **nicht automatisch** für Tickets, siehe [Tickets-Modul](../addons/tickets.md))
- Report-Buttons „Deny"/„Warn" nutzen (ohne `Moderate Members` nötig zu haben)

Für zusätzliche modul-eigene Staff-Rollen (z.B. Suggestions) siehe die jeweilige Modul-YAML (`staffRoles`).
