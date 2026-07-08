# Mod-Workflow

Typische Abläufe im Moderations-Alltag mit GrumpyCore.

---

## Regelverstoß — manuell

```
1. /mod warn user:@User reason:"Regelverstoß als Grund"
   → Eintrag P-# wird angelegt, zählt zur Eskalation

2. Wiederholt? → erneut /mod warn → Eskalation läuft automatisch
   (Standard-Leiter: 3 Warns → Mute 60min · 5 Warns → Kick · 7 Warns → Ban
    — konfigurierbar in mod.yml → escalation)
```

Für einen befristeten Ban direkt: `/mod tempban user:@User duration:3d reason:"..."` — wird nach Ablauf automatisch per Auto-Unban-Runner aufgehoben (Prüfung alle 60 Sekunden).

---

## Report bearbeiten

```
1. User meldet jemanden: /report user:@User reason:"Grund" [proof: Screenshot]
2. Report erscheint im konfigurierten Kanal (channels.staff → mod.yml reportChannelId → channels.mod-log)
   mit Buttons: Deny / Warn / Timeout / Kick / Ban
3. Mod/Support prüft den Fall, wählt eine Aktion:
   ❌ Deny     → nur Support-Rolle oder Moderate Members nötig
   ⚠️ Warn     → nur Support-Rolle oder Moderate Members nötig
   ⏱️ Timeout  → zusätzlich Moderate Members nötig (fest 10 Min.)
   👢 Kick     → zusätzlich Kick Members nötig
   🔨 Ban      → zusätzlich Ban Members nötig
4. Buttons werden nach der Aktion deaktiviert, Status wird im Embed vermerkt
```

---

## Ticket bearbeiten

```
1. Neues Ticket öffnet über ein Panel → Staff-Rollen der Kategorie werden gepingt
2. Verfügbarer Supporter klickt "Claim" → Ticket wird zugewiesen (atomar, kein Doppel-Claim)
3. Problem lösen
4. Ticket schließen → Close-Button (Modal mit optionalem Grund) oder /ticket close
   → Transcript wird automatisch generiert und in Log-/Transcript-Kanal gepostet
   → Owner bekommt Rating-Prompt (1-5 ⭐, standardmäßig per DM)
   → Channel wird nach 5 Sekunden gelöscht
5. Falls nötig: /ticket transfer category:<id> verschiebt in eine andere Kategorie
```

**Inaktive Tickets** werden automatisch geschlossen, wenn die Kategorie `autoCloseAfterHours > 0` hat und seit `lastActivityAt` entsprechend viel Zeit vergangen ist (Scan-Intervall Standard 30 Min.).

---

## Punishment-History checken

```
/mod history user:@User
→ Zeigt bis zu 25 Einträge (🟢 aktiv / ⚫ inaktiv), neueste zuerst
→ Jeder Eintrag mit P-#-ID, Moderator, Zeitstempel, ggf. Ablaufzeit
```

---

## Verwarnung korrigieren

```
Falsche Verwarnung ausgestellt?
→ /mod history user:@User → P-#-ID der falschen Verwarnung notieren
→ /mod unwarn id:<ID> → markiert nur diese Verwarnung als inaktiv
  (zählt danach nicht mehr zur Eskalation, bleibt aber in der History sichtbar)
```

Es gibt **keinen** globalen "alle Verwarnungen löschen"-Befehl — nur einzelne, gezielte `unwarn`.

---

## Ban aufheben

```
/mod unban user-id:<Discord-User-ID>
→ User-ID nötig (nicht Username) — z.B. aus dem Mod-Log oder /mod history kopieren
```

Bei einem `/mod tempban` ist kein manuelles Unban nötig — der Auto-Unban-Runner hebt ihn automatisch zum konfigurierten Zeitpunkt auf.

---

## Tipps

- **Immer einen Grund angeben** (`reason`) — erscheint im Mod-Log und (bei Warn/Kick/Ban) in der Aktion selbst.
- **P-#-IDs notieren**, wenn Fälle dokumentiert werden müssen.
- **Rollen-Hierarchie beachten** — der Bot blockiert automatisch, wenn versucht wird, einen gleich- oder höherrangigen User zu moderieren.
- Bei Auto-Mod-Treffern (Spam/Flood/Phishing/etc.) läuft die Eskalation identisch zu manuellen Warns — Auto-Mod-Warns zählen mit.
