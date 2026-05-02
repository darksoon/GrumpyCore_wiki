# Mod-Workflow

Typische Abläufe im Moderations-Alltag.

---

## Regelverstoß — manuell

```
1. /mod warn @User [Regelverstoß als Grund]
   → User bekommt DM + Eintrag im Staff-Channel (P#ID)
   → Mod-Log wird automatisch aktualisiert

2. Wiederholt? → /mod warn erneut → Eskalation automatisch
   Stufe 3 → Timeout (10 Min)
   Stufe 4 → Kick
   Stufe 5+ → Ban
```

---

## Report bearbeiten

```
1. User meldet jemanden mit /report @User [Grund]
2. Report erscheint im Staff-Channel mit Buttons
3. Mod prüft den Fall
4. Aktion wählen:
   ❌ Ablehnen → Report-Embed wird als "Abgelehnt" markiert
   ⚠️ Warn     → Verwarnung wird erteilt (P#ID)
   ⏱️ Timeout  → 10 Minuten Timeout
   👢 Kick     → Sofort-Kick
   🔨 Ban      → Sofort-Ban
```

---

## Ticket bearbeiten

```
1. Neues Ticket öffnet sich → Support-Rollen werden gepingt
2. Verfügbarer Supporter klickt 📌 Claim → Thread wird zugewiesen
3. Problem lösen
4. Ticket schließen → 🔒 Schließen klicken
   → Transcript wird automatisch in #ticket-log gepostet
   → User bekommt Rating-DM (1-5 ⭐)
5. Falls weg müssen → Ticket an Kollegen: 📌 Unclaim → anderer claimen
```

---

## Inaktives Ticket

```
Bot macht das automatisch:
- 48h keine Aktivität → Warnung im Ticket + DM an User
- Weitere 24h keine Reaktion → automatisch geschlossen mit Transcript
```

---

## Punishment-History checken

```
/history @User
→ Tabs: ⚠️ Warns | 👢 Kicks | 🔨 Bans | ⏱️ Timeouts
→ Zeigt alle P#IDs mit Datum und Moderator
```

---

## Verwarnung korrigieren

```
Falsche Verwarnung ausgestellt?
→ /mod warns @User → P#ID der falschen Verwarnung notieren
→ /mod unwarn [ID] → nur diese Verwarnung löschen
  (Berechtigung: Server verwalten)

Alle Verwarnungen eines Users löschen:
→ /mod clearwarns @User
  (Berechtigung: Administrator)
```

---

## Ban aufheben

```
/mod unban [User-ID]
→ User-ID nötig (nicht Username) — aus dem Mod-Log oder Audit-Log kopieren
```

---

## Tipps

- **Immer einen Grund angeben** — erscheint im Mod-Log und in der DM an den User
- **P#IDs notieren** wenn Fälle dokumentiert werden müssen
- **Rollen-Hierarchie** beachten — Bot blockiert automatisch wenn du höherrangige User moderieren willst
