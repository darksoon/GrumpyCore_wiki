# GrumpyCore — RSS-Modul

## 1. Einleitung

Abonniert RSS/Atom-Feeds für einen Server und postet neue Einträge automatisch in einen Kanal, optional mit Rollen-Ping. Reines DB-Modul, verwaltet über `/rss`. Addon-Key: `rss`.

## 2. Wie es funktioniert

### Poller

Ein Hintergrund-Timer prüft alle **10 Minuten** alle abonnierten Feeds server-übergreifend auf neue Einträge. Erster Durchlauf sofort beim Start des Bots. Ein Re-Entrancy-Schutz sorgt dafür, dass ein noch laufender Durchlauf (z.B. bei vielen Feeds oder langsamen Hosts) nicht mit dem nächsten geplanten Durchlauf überlappt — sonst könnten Einträge doppelt gepostet werden.

### Ablauf pro Feed

1. Feed-XML wird über einen eigenen, manuell gesteuerten Redirect-Loop abgerufen (siehe SSRF-Schutz unten), mit 10s Timeout und einer Obergrenze von 5MB Antwortgröße (wird beim Streamen geprüft, nicht erst danach).
2. Der Parser unterstützt sowohl RSS 2.0 (`<channel><item>`) als auch Atom (`<feed><entry>`) und extrahiert GUID, Titel, Link und Datum je Eintrag.
3. **Erstes Abonnement**: Beim allerersten Check eines neu hinzugefügten Feeds wird nur der aktuell neueste Eintrag als Referenzpunkt (`lastGuid`) gespeichert — **es wird nichts gepostet**. Das verhindert, dass beim Abonnieren eines alten, aktiven Blogs dessen komplette Historie in den Kanal geflutet wird.
4. Bei folgenden Durchläufen werden alle Einträge gepostet, die neuer sind als `lastGuid` (in der Feed-Reihenfolge vor dem bekannten Eintrag gefunden). Ist der bekannte Eintrag nicht mehr im Feed-Fenster vorhanden (z.B. nach langer Downtime oder GUID-Reset des Feeds), gelten vorsichtshalber alle aktuellen Einträge als neu.
5. Pro Feed und Durchlauf werden maximal **5 Einträge** gepostet (älteste zuerst), um bei einem plötzlichen Nachrichten-Stau nicht Dutzende Nachrichten auf einmal in den Kanal zu kippen.
6. `lastGuid` wird nach jedem Durchlauf aktualisiert — auch wenn der Post in den Kanal fehlschlägt (z.B. fehlende Berechtigung), damit sich der Bot nicht endlos an denselben Einträgen festbeißt. Admins müssen den Kanal/die Berechtigung reparieren, damit neue Einträge wieder ankommen.
7. Ist der Zielkanal nicht erreichbar, wird das nur geloggt.

### SSRF-Schutz (URL-Sicherheitsprüfung)

Da der Bot serverseitig beliebige URLs abruft, die von Mitgliedern mit `ManageGuild` angegeben werden, prüft `urlSafety.ts` jede URL, bevor sie abgerufen wird:

- Nur `http://` und `https://` sind erlaubt.
- `localhost`/`*.localhost` wird abgelehnt.
- Ist der Host bereits eine IP-Adresse, wird sie direkt geprüft.
- Andernfalls wird der Hostname per DNS aufgelöst (**alle** zurückgegebenen A/AAAA-Records, nicht nur der erste — ein Hostname mit mehreren Adressen ist bereits unsicher, wenn nur eine davon intern ist).
- Abgelehnt werden u.a.: `10.0.0.0/8`, `127.0.0.0/8` (Loopback), `169.254.0.0/16` (Link-Local, z.B. Cloud-Metadaten-Endpunkte wie `169.254.169.254`), `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10` (CGNAT), `0.0.0.0/8`, Broadcast, sowie die entsprechenden IPv6-Äquivalente (`::1`, `fe80::/10`, `fc00::/7`, IPv4-gemappte Adressen).
- Schlägt die DNS-Auflösung fehl oder ist die Adresse nicht eindeutig einzuordnen, gilt sie als **unsicher** (fail-closed, kein Fail-Open).

Die Prüfung läuft an **zwei Stellen**:

1. **Bei `/rss add`**: schnelle Ablehnung offensichtlich unsicherer URLs direkt bei der Eingabe.
2. **Im Poller, bei jedem Fetch und jedem einzelnen Redirect-Hop** (bis zu 5 Hops): Der Bot folgt Redirects nicht automatisch (`redirect: 'manual'`), sondern validiert jede neue Ziel-URL erneut, bevor er ihr folgt. Das ist notwendig, weil ein Hostname zum Zeitpunkt des Abonnierens öffentlich auflösen kann (DNS-Rebinding), später aber auf eine interne Adresse zeigt — oder weil ein kompromittierter/böswilliger Feed per 301-Redirect auf einen internen Endpunkt umleitet.

Für Admins bedeutet das: Wird eine URL bei `/rss add` mit "zeigt auf eine interne/private Adresse" abgelehnt, oder postet ein bereits abonnierter Feed dauerhaft keine neuen Einträge mehr, liegt das in der Regel an dieser Schutzmaßnahme — nicht an einem Bug.

## 3. Slash-Commands

Nicht in DMs verfügbar.

### `/rss add <url> [channel] [ping-role]`

| Option | Beschreibung |
|---|---|
| `url` | Feed-URL (RSS oder Atom), muss mit `http://` oder `https://` beginnen |
| `channel` | Zielkanal für neue Einträge (Text/Ankündigung), Standard: aktueller Kanal |
| `ping-role` | Rolle, die bei neuen Einträgen gepingt wird (optional) |

Schlägt fehl, wenn: die URL ungültig oder unsicher ist (siehe SSRF-Schutz), der Feed bereits für diesen Server abonniert ist, oder das **Limit von 20 Feeds pro Server** erreicht ist.

### `/rss remove <id>`

| Option | Beschreibung |
|---|---|
| `id` | Feed-ID (siehe `/rss list`) |

Entfernt ein Abonnement des Servers per ID.

### `/rss list`

Zeigt alle Feed-Abonnements des Servers (ID, Zielkanal, URL, ggf. Ping-Rolle). Ephemer.

## 4. Berechtigungen

Alle Unterbefehle erfordern standardmäßig **Server verwalten** (`ManageGuild`), serverseitig konfigurierbar über Discords Befehlsberechtigungen.
