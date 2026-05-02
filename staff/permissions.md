# Berechtigungen — Wer darf was

---

## Rollen-Übersicht

| Aktion | Supporter | Moderator | Admin |
|---|---|---|---|
| `/mod warn` | ✅ | ✅ | ✅ |
| `/mod timeout` | ✅ | ✅ | ✅ |
| `/mod kick` | ❌ | ✅ | ✅ |
| `/mod ban` | ❌ | ✅ | ✅ |
| `/mod unban` | ❌ | ✅ | ✅ |
| `/mod warns @user` | ✅ | ✅ | ✅ |
| `/mod unwarn [id]` | ❌ | ✅ | ✅ |
| `/mod clearwarns` | ❌ | ❌ | ✅ |
| `/history @user` | ✅ | ✅ | ✅ |
| `/report` (einreichen) | Alle | Alle | Alle |
| Report-Buttons (Deny/Warn/Kick/Ban) | ✅ | ✅ | ✅ |
| Ticket Claim/Close | ✅ | ✅ | ✅ |
| Ticket Priorität ändern | ✅ | ✅ | ✅ |
| `/preview` | ❌ | ❌ | ✅ |
| `/botstatus` | ❌ | ❌ | ✅ |
| News posten (DM an Bot) | ❌ | ❌ | ✅* |

*Kann in der GrumpyNews-Config für bestimmte User-IDs freigegeben werden.

---

## Wichtige Regel: Rollen-Hierarchie

Du kannst nur User moderieren die einen **niedrigeren Rang** haben als du.

- Supporter kann keine Mods warnen
- Mod kann keinen Admin kicken
- Der Bot prüft das automatisch

---

## Discord Berechtigungen einstellen

**Server-Einstellungen → Integrationen → Grumpy [BOT]**

Dort kannst du für jeden Command einstellen:
- Welche Rollen ihn sehen/nutzen dürfen
- In welchen Channels er verfügbar ist

Beispiel: `/mod kick` nur für Rollen mit "Mitglieder kicken"-Berechtigung sichtbar machen.

---

## Supporter einrichten

In `configs/config.yml`:
```yaml
roles:
  support:
    - "SUPPORTER_ROLLEN_ID"
    - "MOD_ROLLEN_ID"
    - "ADMIN_ROLLEN_ID"
```

Diese Rollen können Tickets verwalten und Report-Buttons nutzen.

Discord-Berechtigung **"Threads verwalten"** im Support-Channel geben → alle Ticket-Threads automatisch sichtbar.
