# Mod-Commands

Alle manuellen Moderations-Commands. Nur für User mit den entsprechenden Discord-Berechtigungen sichtbar.

---

## /mod — Übersicht

| Command | Berechtigung | Funktion |
|---|---|---|
| `/mod warn @user [grund]` | Mitglieder moderieren | User verwarnen (P#ID) |
| `/mod timeout @user [dauer] [grund]` | Mitglieder moderieren | User timeoutten (10m, 1h, 7d) |
| `/mod untimeout @user` | Mitglieder moderieren | Timeout aufheben |
| `/mod kick @user [grund]` | Mitglieder kicken | User kicken |
| `/mod ban @user [grund] [dauer]` | Mitglieder bannen | User bannen (permanent oder zeitlich) |
| `/mod unban [user-id]` | Mitglieder bannen | User entbannen |
| `/mod warns @user` | Mitglieder moderieren | Aktive Verwarnungen anzeigen |
| `/mod unwarn [id]` | Server verwalten | Einzelne Verwarnung per P#ID löschen |
| `/mod clearwarns @user` | Administrator | Alle Verwarnungen eines Users löschen |

> **Rollen-Hierarchie:** Du kannst nur User moderieren die einen **niedrigeren Rang** haben als du.

---

## /history @user

Zeigt die komplette Punishment-History eines Users mit Tabs:

- ⚠️ **Warns** — alle Verwarnungen mit P#ID
- 👢 **Kicks** — alle Kicks
- 🔨 **Bans** — alle Bans
- ⏱️ **Timeouts** — alle Timeouts

Tabs sind klickbar um zwischen Kategorien zu wechseln.

**Berechtigung:** Mitglieder moderieren

---

## /report @user [grund]

Meldet einen User an das Staff-Team.

- Optionaler Screenshot als Beweis anhängen
- Das Team sieht die Meldung im Staff-Channel mit Aktions-Buttons
- Cooldown: 5 Minuten zwischen Reports
- Limit: 3 Reports pro Tag

**Berechtigung:** Alle Member

---

## /botstatus

Zeigt Status aller geladenen Addons, Uptime, Memory, Ping.

**Berechtigung:** Administrator

---

## Dauer-Format

| Eingabe | Bedeutung |
|---|---|
| `10s` | 10 Sekunden |
| `5m` | 5 Minuten |
| `2h` | 2 Stunden |
| `7d` | 7 Tage |
