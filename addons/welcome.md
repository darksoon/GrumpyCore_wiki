# GrumpyWelcome — Willkommen & Verifizierung

---

## Welcome-Banner

Wenn ein neuer Member joint, postet der Bot ein **dynamisches Bild** im Welcome-Channel:

- Runder Avatar des Users
- Username (Schriftgröße passt sich automatisch an)
- Mitgliedsnummer (#247)
- Progress-Bar
- Server-Name

**Banner anpassen** in `configs/GrumpyWelcome/config.yml`:
```yaml
banner-enabled: true
banner-background-color: "#1a1a2e"  # Hintergrundfarbe
banner-accent-color: "#ff6b35"      # Avatar-Ring, Bar, Labels
banner-text-color: "#ffffff"        # Username-Farbe
banner-background-url: ""           # Eigenes Hintergrundbild (https://...)
```

---

## Verifizierung

**Ablauf:**
1. Neuer Member joined → bekommt `Unverified`-Rolle (sieht nur #verify)
2. Bot pingt ihn kurz in #verify (Ping löscht sich nach 10 Sekunden)
3. User klickt **📋 Regeln akzeptieren & Verifizieren**
4. Bot schickt CAPTCHA per DM (Matheaufgabe)
5. User beantwortet per DM
6. Richtig → `Unverified` weg, `Member`-Rolle drauf

**Bei gesperrten DMs:** Bot zeigt Anleitung wie man DMs aktiviert.

**3 Fehlversuche** → User wird automatisch gekickt (kann rejoin und erneut versuchen).

---

## Leave-Nachrichten

Wenn ein Member den Server verlässt, postet der Bot eine zufällige Nachricht im Welcome-Channel.

Nachrichten anpassen in `configs/GrumpyWelcome/config.yml`:
```yaml
leave-messages:
  - "😤 **{user}** ist davongeflogen!"
  - "💨 **{user}** hat das Weite gesucht."
  - "🚪 **{user}** hat die Tür hinter sich zugeknallt."
```
`{user}` wird durch den Username ersetzt.

---

## /preview Commands

Vorschau ohne dass jemand joinen muss:

| Command | Funktion |
|---|---|
| `/preview welcome` | Welcome-Banner mit deinem eigenen Avatar |
| `/preview welcome @user` | Banner mit einem anderen User simulieren |
| `/preview leave` | Alle Leave-Nachrichten anzeigen |
| `/preview verify` | Verify-Button Vorschau |

**Berechtigung:** Server verwalten
