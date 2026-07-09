# GrumpyCore — YouTube-Modul

## 1. Einleitung

Beobachtet YouTube-Kanäle und postet neue Uploads automatisch in einen Kanal, optional mit Rollen-Ping. Reines DB-Modul, verwaltet über `/youtube`. Addon-Key: `youtube`.

**Kein API-Key nötig.** Statt der (kostenpflichtig limitierten) YouTube Data API nutzt das Modul den öffentlichen, unauthentifizierten Atom-Feed, den YouTube für jeden Kanal bereitstellt (`https://www.youtube.com/feeds/videos.xml?channel_id=...`). Der Feed-Parser ist derselbe, den auch das [RSS-Modul](rss.md) verwendet (Atom wird generisch unterstützt) — es gibt also keinerlei Setup-Voraussetzung, das Modul funktioniert sofort nach Deploy.

## 2. Wie es funktioniert

### Poller

Prüft alle **10 Minuten** alle beobachteten Kanäle server-übergreifend auf neue Uploads. Erster Durchlauf sofort beim Start. Re-Entrancy-Schutz wie beim RSS-Poller: ein noch laufender Durchlauf überlappt nicht mit dem nächsten geplanten.

### Ablauf pro beobachtetem Kanal

1. Feed wird direkt per `fetch()` mit 10s-Timeout abgerufen — anders als beim RSS-Modul ist hier **kein SSRF-Schutz mit Redirect-Validierung nötig**, weil die URL immer selbst aus einer festen, vertrauenswürdigen `youtube.com`-Domain gebaut wird und nicht aus einer von Nutzern eingegebenen beliebigen URL besteht.
2. **Erste Beobachtung**: Beim allerersten Check wird nur das aktuell neueste Video als Referenzpunkt (`lastVideoId`) gespeichert — **es wird nichts gepostet**. Verhindert, dass beim Hinzufügen eines etablierten Kanals dessen komplette Upload-Historie geflutet wird.
3. Bei folgenden Durchläufen werden alle Videos gepostet, die neuer sind als `lastVideoId`. Fällt der bekannte Eintrag aus dem Feed-Fenster (lange Downtime, Feed-Reset), gelten vorsichtshalber alle aktuellen Einträge als neu.
4. Pro Kanal und Durchlauf werden maximal **3 Videos** gepostet (älteste zuerst).
5. `lastVideoId` wird immer aktualisiert, auch wenn der Post fehlschlägt — ein dauerhaft kaputter Zielkanal blockiert nicht den Fortschritt, der Admin muss den Kanal/die Berechtigung reparieren.
6. Postet Titel + Link als Embed (kein Thumbnail — der wiederverwendete generische Atom-Parser extrahiert es nicht, bewusst nicht extra dafür erweitert).

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/youtube add <channel-id> [channel] [ping-role] [label]`

| Option | Beschreibung |
|---|---|
| `channel-id` | YouTube-Kanal-ID (beginnt mit `UC`, 24 Zeichen) — **nicht** der Handle (`@name`) oder Kanalname |
| `channel` | Zielkanal für neue Video-Posts (Text/Ankündigung), Standard: aktueller Kanal |
| `ping-role` | Rolle, die bei neuen Uploads gepingt wird (optional) |
| `label` | Anzeigename für den Kanal in Posts/Listen (optional) |

Die Kanal-ID findet man z.B. über die Kanal-Startseite → "Kanal teilen" → "Kanal-ID kopieren", oder im Quelltext der Kanalseite.

Schlägt fehl, wenn: die Kanal-ID ungültig aussieht, der Kanal bereits für diesen Server beobachtet wird, oder das **Limit von 10 beobachteten Kanälen pro Server** erreicht ist.

### `/youtube remove <id>`

Entfernt eine Beobachtung per ID (siehe `/youtube list`).

### `/youtube list`

Zeigt alle beobachteten YouTube-Kanäle des Servers. Ephemer.

## 4. Berechtigungen

Alle Unterbefehle erfordern standardmäßig **Server verwalten** (`ManageGuild`).
