# GrumpyNews — News posten

Admins können News direkt per DM an den Bot schicken. Der Bot postet sie als schön formatierten Embed im News-Channel.

---

## Ablauf

1. Admin schreibt dem Bot **eine beliebige DM** (triggert die Session)
2. Bot fragt nach dem **Titel**
3. Admin antwortet mit dem Titel
4. Bot fragt nach dem **Inhalt**
5. Admin antwortet mit dem Inhalt
6. Bot fragt nach einem optionalen **Bild-URL** (oder `skip`)
7. Bot zeigt eine **Vorschau** und fragt "So posten? (ja / nein)"
8. Admin bestätigt mit `ja` → News wird gepostet + Reactions hinzugefügt

---

## Bilder hinzufügen

Nur direkte Bild-Links funktionieren (endet auf `.png`, `.jpg`, `.gif`, `.webp`).

**Empfohlen:** Bild in irgendeinen Discord-Channel hochladen → Rechtsklick → **Link kopieren** → im Bot einfügen.

Vecteezy, Pinterest und ähnliche Seiten blockieren das Laden durch Discord (Hotlink-Schutz).

---

## Session-Timeout

Wenn du 10 Minuten nichts antwortest, wird die Session automatisch abgebrochen. Einfach erneut eine DM schicken um neu zu starten.

---

## Konfiguration

In `configs/GrumpyNews/config.yml`:

```yaml
admin-ids:
  - "DEINE_USER_ID"    # Wer News posten darf
  - "WEITERE_USER_ID"  # Mehrere erlaubt

reactions:
  - "👍"
  - "❤️"
  - "🎉"

session-timeout: 600   # Sekunden bis Session abläuft
```

**User-ID kopieren:** Developer Mode an → Rechtsklick auf deinen Namen → **ID kopieren**
