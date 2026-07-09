# GrumpyCore — Twitch-Modul

## 1. Einleitung

Beobachtet Twitch-Streamer und postet eine Ankündigung, sobald einer live geht — inkl. Titel, Spiel, Zuschauerzahl und Thumbnail, optional mit Rollen-Ping. Reines DB-Modul, verwaltet über `/twitch`. Addon-Key: `twitch`.

**Erfordert Twitch-App-Zugangsdaten.** Anders als beim [YouTube-Modul](youtube.md) gibt es bei Twitch keinen anonymen API-Zugang — die Helix-API verlangt für jede Anfrage (auch nur "ist X gerade live?") ein OAuth-App-Token. Das bedeutet:

1. Kostenlos eine App unter [dev.twitch.tv/console/apps](https://dev.twitch.tv/console/apps) registrieren (nur ein Twitch-Account nötig, keine Kosten).
2. `Client ID` und `Client Secret` in `configs/config.yml` eintragen:
   ```yaml
   twitchClientId: "..."
   twitchClientSecret: "..."
   ```
3. Bot neu starten.

**Ohne diese Zugangsdaten bleibt das Modul inaktiv, aber nutzbar**: `/twitch add`/`remove`/`list` funktionieren immer (reine DB-Operationen), nur der Live-Check-Poller überspringt seine Durchläufe, solange keine Zugangsdaten hinterlegt sind. Beim Hinzufügen eines Streamers ohne konfigurierte Zugangsdaten weist der Bot in der Antwort explizit darauf hin, dass die Live-Erkennung noch inaktiv ist — der Eintrag muss danach nicht neu angelegt werden, sobald die Zugangsdaten nachgetragen werden. Im Setup-Status-Banner beim Bot-Start erscheint das Modul dann als **🟡 enabled (no credentials — inactive)** statt grün.

## 2. Wie es funktioniert

### Token-Verwaltung

`auth.ts` holt sich per `client_credentials`-Flow (`POST id.twitch.tv/oauth2/token`) ein App-Access-Token und cached es im Speicher bis kurz vor Ablauf (60s Sicherheitsabstand), danach wird automatisch erneuert. Fehlt Client-ID/Secret oder schlägt die Anfrage fehl, liefert diese Funktion `null` zurück — sie wirft nie, sodass der Poller/die Commands das sauber als "nicht konfiguriert" behandeln können.

### Poller

Prüft alle **5 Minuten** den Live-Status aller beobachteten Streamer server-übergreifend (Twitch erlaubt bis zu 800 Punkte/Minute mit App-Token, ein `Get Streams`-Aufruf kostet nur 1 Punkt egal wie viele Streamer gleichzeitig abgefragt werden — 5 Minuten sind also sehr komfortabel). Re-Entrancy-Schutz wie bei den anderen Pollern.

- Anfragen werden in Gruppen von bis zu **100 Logins** pro API-Call gebündelt (Twitch-Limit pro Anfrage).
- **Edge-Detection**: Nur der Übergang offline→live löst eine Ankündigung aus, nicht jeder Durchlauf, in dem jemand noch live ist. Der Übergang live→offline setzt den Status intern nur zurück, ohne eine weitere Nachricht zu posten (die Live-Ankündigung wird bewusst nicht nachträglich bearbeitet oder gelöscht — bleibt als Verlauf stehen).
- Ist Twitch nicht konfiguriert (siehe oben), überspringt der Poller den kompletten Durchlauf ohne Fehler.

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/twitch add <login> [channel] [ping-role]`

| Option | Beschreibung |
|---|---|
| `login` | Twitch-Benutzername (wird automatisch klein geschrieben gespeichert) |
| `channel` | Zielkanal für Live-Ankündigungen (Text/Ankündigung), Standard: aktueller Kanal |
| `ping-role` | Rolle, die beim Live-Gehen gepingt wird (optional) |

Schlägt fehl, wenn: der Streamer bereits für diesen Server beobachtet wird, oder das **Limit von 10 beobachteten Streamern pro Server** erreicht ist. Sind keine Twitch-Zugangsdaten hinterlegt, wird der Eintrag trotzdem angelegt, aber mit einem Warnhinweis in der Antwort.

### `/twitch remove <id>`

Entfernt eine Beobachtung per ID (siehe `/twitch list`). Funktioniert unabhängig davon, ob Twitch konfiguriert ist.

### `/twitch list`

Zeigt alle beobachteten Streamer des Servers inkl. aktuellem Live-Status (🔴). Ephemer, funktioniert unabhängig davon, ob Twitch konfiguriert ist.

## 4. Berechtigungen

Alle Unterbefehle erfordern standardmäßig **Server verwalten** (`ManageGuild`).
