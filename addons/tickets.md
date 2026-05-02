# GrumpyTickets — Support-Tickets

---

## Wie es funktioniert

1. User klickt **🎫 Ticket erstellen** im Support-Channel
2. Kategorie auswählen (Support / Bug-Report / Bewerbung / Sonstiges)
3. Privater Thread öffnet sich — nur User + Support-Team sieht ihn
4. Support-Team wird gepingt und kann Claim/Close drücken

---

## Prioritäten

Jedes Ticket startet auf **🟡 Mittel**. Support-Mitarbeiter können die Priorität ändern:

| Priorität | Farbe | Bedeutung |
|---|---|---|
| 🟢 Niedrig | Grün | Kann warten |
| 🟡 Mittel | Gelb | Normal (Standard) |
| 🔴 Hoch | Rot | Dringend |

Der Thread-Name und die Embed-Farbe ändern sich mit der Priorität.

---

## Buttons im Ticket

| Button | Wer darf | Funktion |
|---|---|---|
| **📌 Claim** | Support-Mitarbeiter | Ticket übernehmen (zeigt Zuständigen an) |
| **📌 Unclaim** | Support-Mitarbeiter | Freigeben damit jemand anderes claimen kann |
| **🔒 Schließen** | Support + Ticket-Ersteller | Ticket schließen + Transcript + Rating |
| **🟢 Niedrig** | Support-Mitarbeiter | Priorität setzen |
| **🟡 Mittel** | Support-Mitarbeiter | Priorität setzen |
| **🔴 Hoch** | Support-Mitarbeiter | Priorität setzen |

---

## Transcript

Beim Schließen wird automatisch ein **HTML-Transcript** erstellt und im `ticket-log` Channel hochgeladen. Enthält alle Nachrichten aus dem Thread.

---

## Bewertung

Nach dem Schließen bekommt der Ticket-Ersteller eine DM und kann den Support mit **1-5 ⭐** bewerten.

---

## Auto-Close

Tickets die lange inaktiv sind werden automatisch geschlossen:

1. Nach **48h** Inaktivität → Warnung im Ticket + DM an User
2. Nach weiteren **24h** → automatisch geschlossen mit Transcript

Zeiten sind in `configs/GrumpyTickets/config.yml` konfigurierbar.

---

## Out-of-Service

Optional: Tickets außerhalb der Support-Zeiten bekommen eine Warnung.

```yaml
out-of-service-enabled: true
service-start-hour: 8   # 08:00 Uhr
service-end-hour: 22    # 22:00 Uhr
```

---

## /ticket Commands

| Command | Berechtigung | Funktion |
|---|---|---|
| `/ticket close [grund]` | Support / Ersteller | Ticket schließen |
| `/ticket claim` | Support | Ticket claimen |
| `/ticket add @user` | Support | User zum Thread hinzufügen |
| `/ticket remove @user` | Support | User aus Thread entfernen |
| `/ticket stats` | Support | Ticket-Statistiken anzeigen |
