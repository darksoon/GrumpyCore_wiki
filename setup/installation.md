# Installation & Deployment

## Voraussetzungen

- Pelican Panel mit Node.js Egg
- Discord Bot Token ([Developer Portal](https://discord.com/developers/applications))
- Bot mit folgenden Intents: **Presence**, **Server Members**, **Message Content**

---

## Schritt 1 — Discord Bot erstellen

1. [discord.com/developers/applications](https://discord.com/developers/applications) → **New Application**
2. Links auf **Bot** → **Add Bot**
3. **Public Bot** deaktivieren
4. **Presence Intent**, **Server Members Intent**, **Message Content Intent** aktivieren
5. **Token** kopieren (wird später gebraucht)

---

## Schritt 2 — Pelican Panel

**Egg:** `egg-grumpy-core-nodejs.json` (liegt im GrumpyCore-Repo)

**Server-Einstellungen:**
```
Docker Image:  ghcr.io/goover/bots:nodemongo8
Startup:       npm install --omit=dev && node index.js
RAM:           512 MB empfohlen
```

---

## Schritt 3 — Dateien hochladen

Folgende Ordner/Dateien auf den Server hochladen:

```
build/          ← kompilierter Bot-Code (MUSS dabei sein!)
index.js        ← Root-Einstiegspunkt
package.json
```

> **Wichtig:** `configs/` und `database.sqlite` NICHT hochladen — die werden automatisch erstellt.

---

## Schritt 4 — Ersten Start

1. Bot starten
2. In der Konsole erscheint:
```
[WARN] [Startup] channels.welcome is not set in configs/config.yml
[WARN] [Startup] channels.verify is not set in configs/config.yml
...
```
3. `configs/config.yml` öffnen und alle IDs eintragen → siehe [Konfiguration](configuration.md)
4. Bot neu starten oder `reload` in der Konsole eingeben

---

## Shell-Commands (Pelican Konsole)

| Command | Funktion |
|---|---|
| `reload` | Alle Addons & Configs neu laden |
| `reload GrumpyMod` | Einzelnes Addon neu laden |
| `addons` | Geladene Addons anzeigen |
| `stop` | Bot sauber herunterfahren |
| `help` | Alle Commands anzeigen |

---

## Bot einladen

```
https://discord.com/api/oauth2/authorize?client_id=BOT_ID&permissions=8&scope=applications.commands%20bot
```
`BOT_ID` durch die Application-ID ersetzen.

> **Empfehlung:** Permission `8` = Administrator. Einfacher als einzelne Permissions zu konfigurieren.
