# GrumpyCore — Welcome & Verifizierung (Modul `welcome`)

---

## Welcome-Banner

Wenn ein neuer Member joint, postet der Bot je nach `join.mode` entweder ein **Embed** oder ein **dynamisches Canvas-Bild** (Banner) im Welcome-Channel (`channels.welcome` in `configs/config.yml`):

- Runder Avatar des Users
- Username (Schriftgröße passt sich automatisch an)
- Mitgliedsnummer
- Progress-Bar
- Server-Name

**Banner anpassen** in `configs/modules/welcome.yml`:
```yaml
join:
  enabled: true
  mode: banner              # "embed" oder "banner"
  contentMessage: "%user_mention% Willkommen auf **%guild_name%**!"
  banner:
    backgroundColor: "#1a1a2e"   # Hintergrundfarbe
    accentColor: "#ff6b35"       # Avatar-Ring, Bar, Labels
    textColor: "#ffffff"         # Username-Farbe
    backgroundUrl: ""            # Eigenes Hintergrundbild (https://...)
    label: "· WELCOME ·"
```

---

## Verifizierung

**Ablauf:**
1. Neuer Member joined → bekommt die konfigurierte `unverified`-Rolle (`roles.unverified` in `configs/config.yml`), sofern `verification.enabled` und die Rolle gesetzt ist.
2. Im Verify-Channel (`channels.verify`) hängt ein **dauerhaftes** Panel (per `/welcome setup-verify` gepostet, wird beim Bot-Start automatisch neu bereitgestellt, falls die gespeicherte Nachricht fehlt) mit Button **"Regeln akzeptieren & Verifizieren"**.
3. User klickt den Button.
4. Bot schickt CAPTCHA per DM — Standard-Typ ist ein **Bild-CAPTCHA** (`verification.captcha.type: image`), alternativ `math` (Matheaufgabe als Text).
5. User beantwortet per DM.
6. Richtig → `unverified`-Rolle weg, `roles.member`-Rolle drauf.

**Bei gesperrten DMs:** Bot zeigt Anleitung wie man DMs aktiviert.

**Fehlversuche/Timeout:** `verification.captcha.maxAttempts` (Default 3) bzw. `timeoutSeconds` (Default 300). Was danach passiert, ist konfigurierbar über `verification.failAction`: `kick` (Default), `ban` oder `nothing`.

**Zusätzliche Schutzmechanismen** (`verification.protection`):
- `minAccountAgeDays` (Default 7), `requireAvatar`, `blockedUsernamePatterns` — blockieren den Verify-Start schon vor dem CAPTCHA.
- **Raid-Schutz** (`protection.raid`): löst bei zu vielen Joins in kurzer Zeit (`joinThreshold`/`windowSeconds`, Default 5 Joins / 30s) eine Lockdown-Phase aus (`lockdownSeconds`, Default 600s), während der Verifizierung blockiert wird; `onJoinDuringLockdown` steuert, was mit während der Lockdown joinenden Mitgliedern passiert (`nothing`/`kick`/`ban`). Manuell aufhebbar per `/welcome clear-lockdown`.

---

## Leave-Nachrichten

Wenn ein Member den Server verlässt, postet der Bot je nach `leave.mode` (`text`/`embed`/`banner`) eine zufällige Text-Nachricht, ein Embed oder ein Canvas-Banner im Welcome-Channel.

Nachrichten anpassen in `configs/modules/welcome.yml`:
```yaml
leave:
  enabled: true
  mode: text
  messages:
    - "👋 **%user_name%** hat den Server verlassen."
    - "👋 **%user_name%** ist gegangen."
    - "👋 Auf Wiedersehen **%user_name%**."
```
`%user_name%` (und weitere Platzhalter wie `%user_mention%`, `%guild_name%`) werden ersetzt.

---

## /welcome — Admin-Commands (`ManageGuild`)

| Command | Funktion |
|---|---|
| `/welcome preview` | Welcome-Embed/-Banner mit deinem eigenen Avatar |
| `/welcome preview-leave` | Zufällige Leave-Nachricht (Text/Embed/Banner je nach `leave.mode`) |
| `/welcome preview-verify` | Verify-Panel-Vorschau |
| `/welcome preview-captcha` | Beispiel-Bild-CAPTCHA (nur im Bild-Modus) |
| `/welcome setup-verify` | Postet das Verify-Panel im konfigurierten Channel und speichert die Nachrichten-ID |
| `/welcome force-verify <user>` | Überspringt das CAPTCHA, verifiziert sofort |
| `/welcome reset <user>` | Setzt aktive CAPTCHA-Session/Cooldown eines Users zurück |
| `/welcome clear-lockdown` | Beendet eine aktive Raid-Lockdown manuell |
| `/welcome set-channel feature:verify-log channel:<#channel>` | Setzt den Verify-Log-Channel (Join/Leave/Verify-Channels selbst liegen in `configs/config.yml`) |
| `/welcome toggle feature:<join\|leave\|verification> enabled:<bool>` | Schaltet ein Feature an/aus |
| `/welcome show` | Zeigt eine Zusammenfassung der aktuellen Welcome-Einstellungen |
| `/welcome reload` | Lädt `configs/modules/welcome.yml` neu |

Es gibt keinen separaten `/preview`-Command mehr — alle Vorschauen sind Subcommands von `/welcome`.

**Berechtigung:** Für den gesamten `/welcome`-Command ist `Manage Guild` (Server verwalten) erforderlich.
