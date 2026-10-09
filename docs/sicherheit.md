# Sicherheitskonzept
 
Stand: 08.10.2026
 
Die Sicherheit beruht nicht auf einer einzelnen Massnahme, sondern auf mehreren
unabhängigen Schichten. Fällt eine aus, schützen die übrigen weiter. Dieses
Prinzip heisst **Defense in Depth**.
 
## Übersicht der Schichten
 
|           Schicht            |        Wogegen sie schützt             |     Stand                 |
|             ---              |               ---                      |      ---                  |
| VPN                          | Zugriff aus dem Internet               | eingerichtet              |
| Netzwerktrennung (Server-VPN)| Zugriff auf fremde Geräte im Heimnetz  | eingerichtet (06.10.2026) |
| Firewall (ufw, `DOCKER-USER`)| nicht benötigte offene Ports           | eingerichtet              |
| HTTPS                        | Mitlesen des Datenverkehrs             | fertig                    |
| Zertifikat                   | gefälschter Server (Man-in-the-Middle) | fertig                    |
| Sicherheitsheader            | typische Angriffe über den Browser     | fertig                    |
| Anmeldung                    | Zugriff durch Unbefugte                | fertig                    |
| Zwei-Faktor-Authentisierung  | gestohlenes Passwort                   | fertig                    | 
| Brute-Force-Schutz           | Durchprobieren von Passwörtern         | aktiv                     |
| Benutzer und Gruppen         | Zugriff auf fremde Dateien             | fertig                    |
| Backup                       | Datenverlust                           | fertig                    |
| Überwachung der Sicherungen  | unbemerkter Ausfall einer Sicherung    | fertig (06.10.2026)       |
| Zweiter Faktor am Router     | Fernänderung der Router-Konfiguration  | aktiv                     |
 
## Zwei VPN mit unterschiedlichen Rechten
 
|                  | Router-VPN (FRITZ!Box)    | Server-VPN (VM)                       |  
| ---              | ---                       | ---                                   |
| Wer              | nur die eigene Verwaltung | die Benutzer                          |
| Erreichbar       | ganzes Heimnetz           | nur Nextcloud (TCP 443) und DNS       |
| Nicht erreichbar | –                         | SSH, Proxmox, Router, andere Geräte   |
| Durchgesetzt     | –                         | Kette `DOCKER-USER` und ufw in der VM |
 
Die FRITZ!Box kann den Zugriff einzelner Teilnehmer nicht einschränken.
Deshalb haben die Benutzer seit dem 06.10.2026 keinen Zugang zum Router-VPN
mehr, sondern ausschliesslich zum Server-VPN. Ein gestohlener Schlüssel
eines Benutzers öffnet damit nur Nextcloud, und dort verlangt die Anmeldung
weiterhin Passwort und gegebenenfalls zweiten Faktor.
 
Warum die Regeln in `DOCKER-USER` stehen und nicht in ufw, ist in `vpn.md`
erklärt: Docker leitet veröffentlichte Ports per DNAT weiter, diese Pakete
laufen durch FORWARD, und ufw sieht sie nicht.
 
## HTTPS und Zertifikat
 
### Ablauf einer Verbindung
 
1. Der Browser öffnet eine Verbindung auf Port 443 und schickt gleich seinen
   Teil des Schlüsselaustauschs mit
2. Der Server antwortet mit seinem Teil, ab hier ist die Verbindung
   verschlüsselt
3. Der **Server** sendet sein Zertifikat und beweist mit einer Signatur,
   dass er den passenden privaten Schlüssel besitzt
4. Der **Browser** prüft: passt der Name, wer hat unterschrieben, ist es noch
   gültig
5. Erst wenn die Prüfung bestanden ist, werden die eigentlichen Daten
   übertragen

So läuft es bei TLS 1.3, das Caddy und aktuelle Browser verwenden. Bei TLS 1.2
kam das Zertifikat noch vor dem Schlüsselaustausch. An seiner Rolle ändert
das nichts: Es bestätigt, mit wem der Schlüssel ausgehandelt wurde, und ohne
bestandene Prüfung fliessen keine Daten.

Das Zertifikat gehört also dem Server und wird vom Browser des Benutzers
geprüft, nicht umgekehrt.
 
### Zwei getrennte Aufgaben
 
**Verschlüsselung.** Der Datenverkehr kann unterwegs nicht mitgelesen werden.
Das funktioniert unabhängig davon, wer das Zertifikat unterschrieben hat.
 
**Nachweis der Identität.** Die Unterschrift einer vertrauenswürdigen Stelle
beweist, dass der Server wirklich der ist, für den er sich ausgibt.
 
Ein selbst signiertes Zertifikat verschlüsselt vollständig. Die Warnung des
Browsers bezieht sich ausschliesslich auf die zweite Aufgabe.
 
### Wogegen der Identitätsnachweis schützt
 
Bei einem **Man-in-the-Middle-Angriff** gibt sich ein Angreifer als Server aus.
Er legt sein eigenes Zertifikat vor, die Verbindung wird aufgebaut und
verschlüsselt — aber mit ihm. Er liest das Passwort mit und leitet die Anfrage
danach an den echten Server weiter, damit nichts auffällt.
 
> Das Schloss im Browser bedeutet "die Verbindung ist verschlüsselt", nicht
> "der Server ist echt".
 
### Umsetzung im Projekt
 
Das Zertifikat wurde mit openssl selbst erstellt und am 06.10.2026 mit
**demselben Schlüssel** neu ausgestellt, um die IP-Adresse zu ergänzen:
 
```bash
openssl req -x509 -new -key server.key -sha256 -days 825 \
  -out server.crt \
  -subj "/CN=privatecloud.fritz.box" \
  -addext "subjectAltName=DNS:privatecloud.fritz.box,DNS:cloud.fritz.box,DNS:localhost,IP:192.168.178.220" \
  -addext "basicConstraints=critical,CA:TRUE"
```
 
Wichtig ist das Feld `subjectAltName`. Moderne Browser werten nur dieses Feld
aus, das ältere Feld `CN` wird ignoriert. Das Zertifikat muss für **jede**
Adresse gelten, über die der Dienst aufgerufen wird, also für Namen und
IP-Adressen.
 
Weil das Zertifikat die Eigenschaft `Basic Constraints: CA:TRUE` besitzt, kann
es auf den Geräten als Stammzertifikat installiert werden. Danach bestätigt der
Browser die Identität des Servers, und die Warnung verschwindet.
 
Praktisch nachgewiesen am 01.10.2026: Die Verbindung war vorher und nachher
gleich stark verschlüsselt, der Unterschied lag allein im Vertrauen.
 
### Installation auf den Geräten
 
| Gerät    | Ort                                                                     |
| ---      | ---                                                                     |
| Windows  | Zertifikat installieren → Vertrauenswürdige Stammzertifizierungsstellen |
| macOS    | Schlüsselbundverwaltung → Vertrauen → Immer vertrauen                   |
| Android  | Einstellungen → Suche "CA-Zertifikat", Bildschirmsperre nötig           |
| Firefox  | eigener Speicher: Zertifikate anzeigen → Zertifizierungsstellen         |
 
Die Datei `server.crt` ist **nicht** geheim und darf verschickt werden.
Geheim ist nur `server.key`.
 
### Warum kein Let's Encrypt
 
Ein Zertifikat von Let's Encrypt wäre auf allen Geräten ohne Installation
vertrauenswürdig. Die übliche Überprüfung verlangt aber, dass der Server aus
dem Internet über die Ports 80 und 443 erreichbar ist. Genau das widerspricht
dem Konzept, dass die Cloud ausschliesslich über VPN zugänglich sein soll.
 
Die Alternative wäre eine Überprüfung über einen DNS-Eintrag mit einer eigenen
Domain (etwa 10 bis 15 CHF im Jahr). Ein Eintrag wie
`cloud.<domain>.ch → 192.168.178.220` würde zusätzlich den eigenen
Namensdienst im VPN und die Installation des Stammzertifikats überflüssig
machen. Diese Möglichkeit bleibt für später offen.
 
## Reverse Proxy
 
Caddy nimmt alle Anfragen entgegen und leitet sie intern weiter. Das bringt
drei Vorteile für die Sicherheit:
 
- Die Verschlüsselung ist an einer einzigen Stelle konfiguriert
- Nextcloud veröffentlicht keinen eigenen Port mehr und ist nur noch über den
  Proxy erreichbar
- Die Sicherheitsheader gelten automatisch für alle künftigen Dienste
|             Header                |                        Wirkung                              |
|              ---                  |                          ---                                |
| `Strict-Transport-Security`       | Der Browser verwendet künftig nur noch HTTPS                |
| `X-Content-Type-Options: nosniff` | Der Browser rät den Dateityp nicht selbst                   |
| `Referrer-Policy: no-referrer`    | Beim Folgen eines Links wird die Herkunft nicht übermittelt |
 
## Zwei-Faktor-Authentisierung
 
### Die drei Faktoren
 
|    Faktor   |          Beispiel                |
|     ---     |            ---                   |
| Wissen      | Passwort                         |
| Besitz      | Smartphone, Sicherheitsschlüssel |
| Eigenschaft | Fingerabdruck, Gesicht           |
 
Eine Zwei-Faktor-Authentisierung verlangt zwei **verschiedene** Faktoren. Zwei
Passwörter wären kein zweiter Faktor.
 
### TOTP
 
Verwendet wird TOTP (Time-based One-Time Password). Server und App auf dem
Smartphone kennen ein gemeinsames Geheimnis und berechnen daraus unabhängig
voneinander einen Code, der von der aktuellen Uhrzeit abhängt. Der Code
wechselt alle 30 Sekunden. Bei der Anmeldung wird nur dieser Code übertragen,
der nach kurzer Zeit wertlos ist. Das gemeinsame Geheimnis selbst geht nur
einmal beim Einrichten (QR-Code) über das Netz, danach nie mehr.
 
### Wiederherstellungscodes
 
Beim Einrichten werden einmalige Codes erzeugt. Sie müssen **ausserhalb des
Smartphones** aufbewahrt werden. Geht das Gerät verloren, sind sie der einzige
Weg zurück ins eigene Konto.
 
Für Administratoren gibt es zusätzlich den Weg über die Konsole:
 
```bash
occ twofactorauth:disable <benutzer>
```
 
## Brute-Force-Schutz
 
Nextcloud sperrt ein Konto nach falschen Anmeldeversuchen **nicht**, sondern
verzögert die Antworten zunehmend, bis zu 25 Sekunden. Nach zehn
Fehlversuchen innerhalb von 30 Minuten antwortet Nextcloud zusätzlich mit
`429 Too Many Requests`. Beides gilt für die Adresse des Angreifers, nicht für
das Konto.
 
Eine echte Sperrung könnte ein Angreifer missbrauchen, um fremde Konten
absichtlich unbrauchbar zu machen. Ein Mensch, der sich zweimal vertippt, merkt
von der Verzögerung nichts, ein Programm mit tausenden Versuchen schon.
 
Die Daten werden je nach Konfiguration in der Datenbank oder im Cache abgelegt.
In diesem Projekt ist Redis als verteilter Cache eingerichtet, deshalb verwendet
Nextcloud den `MemoryCacheBackend`.
 
### Zusammenhang mit dem Proxy
 
Ohne die Einstellung `trusted_proxies` würde Nextcloud alle Anfragen der Adresse
des Proxy-Containers zuordnen. Ein einziger Angreifer würde dann die Antworten
für **alle** Benutzer verlangsamen. Mit der richtigen Einstellung wertet
Nextcloud den Header `X-Forwarded-For` aus. Am 01.10.2026 überprüft.
 
Über das Server-VPN erscheinen die Benutzer mit ihrer Tunneladresse
(`10.9.0.x`). Jedes Gerät wird damit einzeln verzögert.
 
## Benutzer und Berechtigungen
 
- Jeder Benutzer erhält einen eigenen, isolierten Bereich
- Gruppen dienen als Grundlage für gemeinsame Ordner
- Gemeinsame Ordner (Team folders) gehören keinem Benutzer und besitzen eine
  eigene Speicherbegrenzung
- Passwörter werden als Hash gespeichert, nicht im Klartext
- Benutzer werden ohne Passwort angelegt und erhalten eine Einladung per
  E-Mail. Das Passwort kennt damit nur der Benutzer.
**Umgesetzt am 05.10.2026.** Es gibt ein Konto in der Gruppe `admin`
ausschliesslich für die Verwaltung und ein Konto ohne Sonderrechte für den
Alltag und für die Smartphone-App. Ein kompromittiertes Alltagskonto hat damit
keine Administratorrechte.
 
Die Zwei-Faktor-Authentisierung ist nur für die Gruppe `admin` verpflichtend.
Für die übrigen Benutzer wird sie empfohlen, aber nicht erzwungen — eine
bewusste Entscheidung bei drei Benutzern aus einem Haushalt.
 
## Geheimnisse auf dem Server
 
|            Datei                        |          Inhalt                              |   Schutz                |
|             ---                         |            ---                               |    ---                  |
| `/srv/nextcloud/.env`                   | Passwörter der Datenbank                     | chmod 600, nicht in Git |
| `/srv/nextcloud/caddy/certs/server.key` | privater Schlüssel des Zertifikats           | chmod 600, nicht in Git |
| `/etc/wireguard/server.key`             | privater Schlüssel des Server-VPN            | chmod 600 (umask 077)   |
| `/etc/msmtprc`                          | App-Passwort für den Mailversand des Systems | chmod 600               |
| `config.php` von Nextcloud              | `secret`, `passwordsalt`, SMTP-Passwort      | im Container            |
| `/srv/backup/db/*`                      | Dumps und Kopien von `config.php`            | chmod 600, `UMask=0077` |
 
Für den Mailversand gibt es **zwei getrennte** App-Passwörter von Google: eines
für Nextcloud und eines für das System (`privatecloud-system`). Beide lassen
sich einzeln widerrufen.
 
## Verbleibende Risiken
 
Diese Punkte sind bekannt und bewusst in Kauf genommen oder für später geplant:
 
**Das Stammzertifikat vertraut mehr als nötig.** Das Zertifikat hat
`CA:TRUE` und ist auf den Geräten der Benutzer als **Stammzertifikat**
installiert. Wer den Schlüssel `server.key` besitzt, könnte damit
Zertifikate für **beliebige** Webseiten ausstellen — auch für eine Bank —, und
die Geräte der Benutzer würden ihnen vertrauen. Solange der Schlüssel nur auf
dem Server liegt, ist das tragbar. Er darf deshalb nie in Git, in ein Backup
ausserhalb des Servers oder in eine Nachricht gelangen. Eine eigene Domain mit
Let's Encrypt würde dieses Risiko beseitigen.
 
**DNS der Benutzer läuft über den Server.** Bei aktivem Server-VPN gehen alle
Namensanfragen der Geräte über dnsmasq. Damit wären sie auf dem Server
einsehbar, und bei ausgeschaltetem Server fällt die Namensauflösung aus.
dnsmasq protokolliert keine Anfragen.
 
**Verlorenes Smartphone mit VPN-Schlüssel.** Jedes Gerät hat eine eigene
Verbindung, damit einzelne Schlüssel gesperrt werden können. Über das
Server-VPN ist zudem nur Nextcloud erreichbar.
 
**Keine Verschlüsselung der Festplatten.** Bei einem Diebstahl des Servers
liessen sich die Daten an einem anderen Computer lesen.
 
**Schwache Passwörter der Benutzer.** Innerhalb des VPN wäre das bereits
gefährlich. Die Benutzer müssen entsprechend angewiesen werden.
 
**Mitgliedschaft in der Gruppe `docker`.** Sie entspricht faktisch
Root-Rechten, weil über einen Container auf das ganze System zugegriffen werden
kann.
 
**Eigene Fehler.** Eine versehentlich offen gelassene Portweiterleitung ist die
häufigste Ursache für Probleme bei selbst betriebenen Servern. In der
FRITZ!Box besteht nur die Freigabe UDP 51821 auf die VM.
 
**Unvollständiges Backup.** Von den Daten werden nur die als wichtig
eingestuften gesichert. Die Entscheidung ist bewusst getroffen und in
`backup-konzept.md` begründet.
 
**Die Sicherung der Konfiguration enthält ein Passwort.** Das Backup-Skript
kopiert `config.php`, und darin steht seit dem 05.10.2026 das SMTP-Passwort im
Klartext — Nextcloud kann es nicht anders speichern. Deshalb wird ein
App-Passwort von Google verwendet und nicht das Kontopasswort.