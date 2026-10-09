# Fernzugriff über VPN
 
Stand: 08.10.2026
 
## Grundsatz
 
Nextcloud ist aus dem Internet **nicht** erreichbar. Es gibt keine
Portweiterleitung auf Port 443 und keinen öffentlich erreichbaren Webserver.
Wer von aussen zugreifen will, baut zuerst einen VPN-Tunnel auf.
 
Es gibt zwei getrennte VPN:
 
| Verbindung              | Ort       | Port      | Zugriff auf        | Für wen                  |
| ---                     | ---       | ---       | ---                | ---                      |
| WireGuard FRITZ!Box     | Router    | UDP 51820 | ganzes Heimnetz    | nur eigene Verwaltung    |
| WireGuard Server        | Ubuntu-VM | UDP 51821 | nur Nextcloud      | Benutzer (seit 06.10.)   |
 
Nachgewiesen am 04.10.2026: über das Mobilfunknetz ohne aktives VPN ist der
Server nicht erreichbar, mit aktivem VPN sofort.
 
## Voraussetzung: der Anschluss
 
| Eigenschaft       | Wert                               |
| ---               | ---                                |
| Öffentliche IPv4  | vorhanden, dynamisch               |
| IPv6              | feste /64-Adresse                  |
| Carrier-Grade-NAT | nein                               |
 
Die frühere Annahme von Carrier-Grade-NAT war falsch. Sie stammte aus der
Planung und wurde am 04.10.2026 widerlegt: Die im Online-Monitor der FRITZ!Box
angezeigte IPv4-Adresse stimmte mit der von einem externen Dienst gemeldeten
überein. Bei Carrier-Grade-NAT läge die Adresse im Bereich `100.64.0.0/10` und
wäre mit mehreren Kunden geteilt.
 
Folge dieser Korrektur: Portweiterleitung funktioniert, und die geplante
zusätzliche statische IPv4-Adresse für 5 CHF im Monat ist nicht nötig. Weil die
Adresse wechselt, wird sie über **MyFRITZ!** als Name erreichbar gemacht — der
Router meldet seine aktuelle Adresse an den Namensdienst von AVM.
 
## Vergleich der Varianten
 
|                           | WireGuard auf der FRITZ!Box | WireGuard auf dem Server | WireGuard über IPv6 | Tailscale (laut Doku) |
|          ---              |            ---              |           ---            |        ---          |         ---           |
| Kosten                    | keine                       | keine                    | keine               | gratis bis 100 Geräte |
| Funktioniert hinter CGNAT | nein                        | nein                     | ja                  | ja                    |
| Aus IPv4-only-Netz        | ja                          | ja                       | **nein**            | ja                    |
| Zugriff einschränkbar     | **nein**                    | **ja**                   | nein                | teilweise             |
| Aufwand beim Einrichten   | klein                       | mittel                   | klein               | klein                 |
| Lerneffekt                | mittel                      | hoch                     | mittel              | gering                |
| Abhängigkeit von Dritten  | Namensdienst von AVM        | Namensdienst von AVM     | keine               | Koordinationsserver   |
 
**Entscheidung:** WireGuard auf der FRITZ!Box für das eigene Konto, weil die
Verwaltung Zugang zum ganzen Heimnetz braucht. Für die Benutzer ein zweiter
WireGuard-Server auf der VM, dessen Teilnehmer nur Nextcloud erreichen.
 
Tailscale wurde **nicht** getestet. Im Firmennetz wurde nach Absprache mit dem
Berufsbildner darauf verzichtet: Ein verschlüsselter Tunnel, der an der
Überwachung des Firmennetzes vorbeiläuft, ist aus Sicht der Netzwerksicherheit
schwer zu beurteilen. Am 05.10.2026 wurde entschieden, den Vergleich ganz
wegzulassen, weil WireGuard die Anforderung erfüllt und kein Carrier-Grade-NAT
vorliegt. Die Spalte Tailscale stützt sich auf die Dokumentation und ist als
solche gekennzeichnet.
 
Dieselbe Überlegung gilt für jedes VPN: Getestet wird über die
Mobilfunk-Freigabe des eigenen Smartphones, nicht aus dem Firmennetz.
 
## Wie die Verbindung zustande kommt
 
```
Schritt 1   Telefon fragt DNS: Adresse von <name>.myfritz.net?
Schritt 2   Telefon sendet UDP an diese Adresse, Port 51820 bzw. 51821
Schritt 3   FRITZ!Box nimmt an (eigener Dienst) oder leitet weiter (Server)
Schritt 4   Tunnel steht, innen gelten andere Adressen
```
 
### Zwei Adressschichten
 
```
äusserer Umschlag (für das Internet sichtbar)
   Telefon 10.231.x.x  ──UDP──►  <öffentliche IPv4>:51821
                                       │  DNAT in der FRITZ!Box
                                       ▼
                                 192.168.178.220:51821
 
innerer Inhalt (nur die beiden Endpunkte sehen ihn)
   10.9.0.2  ──HTTPS 443──►  192.168.178.220
```
 
Das Telefon kennt die interne Adresse des Servers nicht und kann sie nicht
kennen — sie existiert nur im Heimnetz. Es schreibt an die öffentliche Adresse,
und der Router tauscht den Empfänger aus. Auf dem Rückweg tauscht er den
Absender zurück.
 
Beim Server-VPN ist der Router **nur Zusteller**: Er entschlüsselt nichts und
weiss nicht, was im Paket steht. Der Tunnel wird zwischen Telefon und VM
aufgebaut.
 
**Eine Portweiterleitung ist kein VPN.** Sie bringt die Pakete nur ans Ziel.
Verschlüsselung und Prüfung der Schlüssel finden an den Endpunkten statt.
 
### Zwei Schichten Verschlüsselung
 
Innerhalb des Tunnels wird weiterhin HTTPS verwendet. Das ist keine
Verdoppelung ohne Zweck: Wer sich schon im VPN-Netz befindet, würde
unverschlüsselten Verkehr mitlesen können.
 
### Wie der Server erkennt, wer anfragt
 
Nicht an der Adresse. Die Adresse eines Smartphones im Mobilfunknetz wechselt
laufend, und der Server kennt sie vorher nicht.
 
Er entschlüsselt das eingehende Paket mit den öffentlichen Schlüsseln aller
bekannten Teilnehmer. Welcher Schlüssel passt, benennt den Absender. Erst danach
merkt sich der Server, von welcher Adresse dieser Teilnehmer gerade kommt. Das
Verfahren heisst **Cryptokey Routing**.
 
Zwei Folgen:
 
- Nur der Client braucht einen `Endpoint`, der Server nicht
- Die Verbindung übersteht einen Wechsel des Netzes, etwa von Mobilfunk auf WLAN
`PersistentKeepalive = 25` in der Konfiguration des Clients: Hinter dem NAT des
Mobilfunkanbieters wird die Zuordnung nach ein bis zwei Minuten Stille
vergessen, und der Server könnte dann nichts von sich aus senden.
 
Ein Handshake wird nur etwa alle zwei Minuten erneuert, und nur solange
Verkehr stattfindet. "Letzter Handshake vor einer Minute" bedeutet also: Der
Tunnel steht.
 
## Split Tunnel und Full Tunnel
 
|                            |          Split Tunnel             |          Full Tunnel               |
|          ---               |              ---                  |              ---                   |
| `AllowedIPs` beim Client   | nur das Ziel im Heimnetz          | `0.0.0.0/0, ::/0`                  |
| Was durch den Tunnel geht  | nur Verkehr zum Server            | **aller** Verkehr des Geräts       |
| Übriger Verkehr            | direkt über den Anbieter          | über den Hausanschluss             |
| Belastung des Anschlusses  | gering                            | der ganze Verkehr, 50 Mbit/s       |
 
Für den Zugriff auf die eigene Cloud ist Split Tunnel richtig. Beide VPN sind
so eingerichtet.
 
### Zwischenfall vom 04.10.2026
 
Die Verbindung für den Laptop wurde versehentlich mit "gesamten Datenverkehr
über die FRITZ!Box senden" erstellt. In der Konfiguration stand dann:
 
```
AllowedIPs = 192.168.178.0/24, 0.0.0.0/0
```
 
Sobald der Tunnel aktiv war, verlor der Laptop jede Verbindung — auch ins
Internet.
 
Beide Einträge gelten gleichzeitig. `AllowedIPs` ist keine Liste von Filtern,
sondern eine **Routentabelle**, und entschieden wird nach dem genauesten Eintrag
(longest prefix match): Das Heimnetz läuft über die erste Zeile, alles andere
über die zweite. Das Ergebnis ist Full Tunnel.
 
Dass die Verbindung **vollständig** ausfiel und nicht nur langsam wurde, hat
eine andere Ursache: Enthält `AllowedIPs` den Eintrag `0.0.0.0/0`, aktiviert der
WireGuard-Client für Windows eine Sperre für Verkehr ausserhalb des Tunnels,
einen **Kill Switch**. Nebenwirkung: Trägt der Tunnel keinen Verkehr nach aussen,
sieht man keinen Teilausfall, sondern gar kein Netz.
 
**Merksatz für die Fehlersuche:** Bei VPN-Problemen zuerst `AllowedIPs`
ansehen, nicht das Netz.
 
## `AllowedIPs` bedeutet auf beiden Seiten etwas anderes
 
|   Seite   |                       Bedeutung                        |             Durchsetzbar?               |
|    ---    |                          ---                           |                  ---                    |
| Client    | wohin Verkehr geschickt wird (Routenhinweis)           | nein — der Client kann die Zeile ändern |
| Server    | welche Absenderadresse dieser Schlüssel verwenden darf | ja — kryptografisch geprüft             |
 
Eine Einschränkung beim Client ist deshalb **keine** Sicherheitsmassnahme.
Wer Zugriff beschränken will, muss es auf dem Server tun. Genau das macht das
Server-VPN.
 
## Zweiter WireGuard-Server auf der VM (seit 06.10.2026)
 
### Problem
 
Teilnehmer des VPN auf der FRITZ!Box erreichen das **ganze** Heimnetz: die
Weboberfläche des Routers, Proxmox, jedes andere Gerät. Die FRITZ!Box 7490 kann
den Zugriff einzelner Teilnehmer nicht einschränken.
 
### Aufbau
 
| Einstellung        | Wert                                                           |
| ---                | ---                                                            |
| Schnittstelle      | `wg0`                                                          |
| Adresse des Servers| `10.9.0.1/24`                                                  |
| Port               | UDP 51821, Portfreigabe in der FRITZ!Box auf `192.168.178.220` |
| Start              | `wg-quick@wg0`, erst nach Docker                               |
 
Die Teilnehmer landen in `10.9.0.0/24` — einem Netz, das nur innerhalb der VM
existiert. Jedes Gerät hat einen eigenen Schlüssel und eine eigene Adresse:
 
| Gerät        | Adresse     |
| ---          | ---         |
| Smartphone 1 | `10.9.0.2`  |
| Smartphone 2 | `10.9.0.3`  |
| Smartphone 3 | `10.9.0.4`  |
| Laptop 1     | `10.9.0.5`  |
| Laptop 2     | `10.9.0.6`  |
| Laptop 3     | `10.9.0.7`  |
 
Auszug aus `/etc/wireguard/wg0.conf` (Schlüssel durch Platzhalter ersetzt):
 
```ini
[Interface]
Address = 10.9.0.1/24
ListenPort = 51821
PrivateKey = <privater Schlüssel des Servers>
 
PostUp   = iptables -I DOCKER-USER -i wg0 -j DROP
PostUp   = iptables -I DOCKER-USER -i wg0 -p tcp --dport 443 -j ACCEPT
PostDown = iptables -D DOCKER-USER -i wg0 -p tcp --dport 443 -j ACCEPT
PostDown = iptables -D DOCKER-USER -i wg0 -j DROP
 
[Peer]
# Smartphone 1
PublicKey = <öffentlicher Schlüssel des Geräts>
AllowedIPs = 10.9.0.2/32
```
 
### Durchsetzung: die Kette `DOCKER-USER`
 
Geplant war am 05.10.2026 eine Einschränkung mit ufw (`ufw route deny`). Bei
der Umsetzung zeigte sich, dass das für Nextcloud nicht greift:
 
Eine Anfrage an Port 443 leitet Docker per **DNAT** an den Container von Caddy
weiter. Das Paket ist damit nicht mehr für die VM selbst bestimmt und läuft
durch die Kette **FORWARD**, nicht durch INPUT. Docker hängt seine eigenen
Ketten in FORWARD **vor** die Regeln von ufw — ufw sieht diese Pakete nicht.
 
Die von Docker vorgesehene Stelle für eigene Regeln ist die Kette
`DOCKER-USER`. Dort stehen zwei Regeln:
 
1. `ACCEPT` für TCP 443 aus `wg0`
2. `DROP` für alles andere aus `wg0`
Die Reihenfolge entscheidet: `-I` fügt am Anfang ein, deshalb wird zuerst das
Verbot und danach die Erlaubnis eingefügt, damit die Erlaubnis oben steht.
 
Für den Zugriff auf die VM **selbst** (SSH und alle anderen Dienste) gilt ufw:
 
```bash
ufw allow in on ens18 to any port 51821 proto udp
ufw insert 1 deny in on wg0
ufw insert 1 allow in on wg0 to 10.9.0.1 port 53 proto udp
ufw insert 1 allow in on wg0 to 10.9.0.1 port 53 proto tcp
```
 
Ergebnis: Über das Server-VPN sind Nextcloud (443) und der Namensdienst (53)
erreichbar, sonst nichts — kein SSH, kein Proxmox, kein Router, kein anderes
Gerät im Heimnetz.
 
Die Paketweiterleitung im Kernel abzuschalten wäre keine Alternative: Docker
setzt `net.ipv4.ip_forward=1` selbst, sonst hätten die Container kein Netz.
 
### Startreihenfolge
 
Die Kette `DOCKER-USER` existiert erst, wenn Docker läuft. Startet WireGuard
vorher, schlägt `PostUp` fehl und die Schnittstelle kommt nicht hoch. Deshalb:
 
```ini
# systemctl edit wg-quick@wg0
[Unit]
After=docker.service
Requires=docker.service
OnFailure=notify-failure@%n.service
```
 
`OnFailure` verschickt eine E-Mail, falls der Tunnel nicht startet (siehe
`server.md`).
 
### Konfiguration der Teilnehmer
 
```ini
[Interface]
PrivateKey = <privater Schlüssel des Geräts>
Address = 10.9.0.2/32
DNS = 10.9.0.1
 
[Peer]
PublicKey = <öffentlicher Schlüssel des Servers>
Endpoint = <name>.myfritz.net:51821
AllowedIPs = 192.168.178.220/32, 10.9.0.1/32
PersistentKeepalive = 25
```
 
`AllowedIPs` enthält nur den Server und den Namensdienst. Das übrige Internet
des Geräts läuft weiter direkt über den Anbieter.
 
### Namensauflösung im Tunnel (dnsmasq)
 
Geplant war, die Teilnehmer über eine Adresse zugreifen zu lassen und keinen
Namensdienst zu betreiben. Im Betrieb ist das unpraktisch: Die Benutzer sollen
dieselbe Adresse verwenden wie zu Hause, `https://privatecloud.fritz.box`.
 
Ohne Namensdienst im Tunnel kann ein Gerät diesen Namen nicht auflösen. Die
FRITZ!Box kann den Teilnehmern auch nicht antworten, weil sie das Netz
`10.9.0.0/24` nicht kennt.
 
Gelöst mit dnsmasq in der VM, das **nur** auf `10.9.0.1` antwortet:
 
```ini
# /etc/dnsmasq.d/wireguard.conf
interface=wg0
except-interface=lo
bind-dynamic
listen-address=10.9.0.1
no-resolv
server=192.168.178.1
address=/privatecloud.fritz.box/192.168.178.220
address=/cloud.fritz.box/192.168.178.220
```
 
- `bind-dynamic`: `wg0` kann nach dnsmasq entstehen
- `except-interface=lo`: kein Konflikt mit `systemd-resolved` der VM
- alle anderen Anfragen gehen an die FRITZ!Box weiter
**Nebenwirkung:** Mit `DNS = 10.9.0.1` laufen bei aktivem Tunnel **alle**
DNS-Anfragen des Geräts über den Server. Ist der Server ausgeschaltet,
funktionieren bei aktivem VPN auch andere Webseiten nicht. Die Benutzer müssen
das VPN dann ausschalten. In der Anleitung für die Benutzer festgehalten.
 
### Zertifikat
 
Das Zertifikat enthielt bis zum 06.10.2026 nur Namen. Mit demselben Schlüssel
neu ausgestellt, damit auch der Aufruf über die Adresse ohne Warnung
funktioniert:
 
```
subjectAltName = DNS:privatecloud.fritz.box, DNS:cloud.fritz.box, DNS:localhost, IP:192.168.178.220
```
 
Die ursprünglich geplante Adresse `10.9.0.1` ist nicht nötig, weil die
Teilnehmer Nextcloud über `192.168.178.220` bzw. den Namen erreichen.
 
## Schlüsselverwaltung
 
**Ein eigener Schlüssel je Gerät.** Geht ein Gerät verloren, wird genau dieser
Schlüssel gesperrt, die übrigen laufen weiter. Zwei Geräte mit demselben
Schlüssel würden sich gegenseitig verdrängen.
 
**Was in einer Konfiguration geheim ist:**
 
|      Feld      |   Geheim? |          Wirkung bei Offenlegung               |
|      ---       |    ---    |                   ---                          |
| `PrivateKey`   | ja        | Fremder kann sich als dieses Gerät ausgeben    |
| `PresharedKey` | ja        | zusätzlicher Schutz entfällt                   |
| `Endpoint`     | teilweise | verrät die öffentliche Adresse des Anschlusses |
| `AllowedIPs`   | nein      | nur ein Routenhinweis                          |
| `Address`      | nein      | Adresse im Tunnel                              |
 
Ein QR-Code enthält die vollständige Konfiguration samt privatem Schlüssel.
 
### Verteilung an die Geräte
 
|                     Weg                          |                Bewertung                        |
|                     ---                          |                   ---                           |
| Schlüssel auf dem Gerät erzeugen, nur den öffentlichen Schlüssel an den Server geben | **bevorzugt**: der private Schlüssel verlässt das Gerät nie |
| QR-Code persönlich vom Bildschirm scannen        | gut, kein Übertragungsweg                        |
| Datei per USB-Kabel oder USB-Stick               | in Ordnung, Datei danach löschen                 |
| Datei per Messenger oder E-Mail                  | **nicht** zulässig                               |
 
Beim ersten Weg wird in der WireGuard-App "Von Grund auf erstellen" gewählt
und der Schlüssel dort erzeugt. Der öffentliche Schlüssel ist nicht geheim und
darf übermittelt werden. Es ist derselbe Gedanke wie bei der Einladung per
E-Mail in Nextcloud: Das Geheimnis kennt nur der Benutzer.
 
Wurden Schlüssel auf dem Server erzeugt, werden die Dateien nach dem Import
auf allen Geräten gelöscht (Server, USB-Stick, eigener Laptop). Die
öffentlichen Schlüssel (`*.pub`) dürfen bleiben.
 
**Vorgehen bei Verdacht auf Offenlegung:** Die Verbindung löschen und neu
anlegen, danach die neue Konfiguration auf dem Gerät importieren und die alte
entfernen. Am 05./06.10.2026 für den Laptop im Router-VPN durchgeführt,
nachdem die Konfiguration möglicherweise in einen Chatverlauf geraten war.
Grundregel: Ein Geheimnis, das den eigenen Einflussbereich verlassen haben
**könnte**, gilt als offengelegt.
 
## Zweiter Faktor der FRITZ!Box
 
Sicherheitsrelevante Änderungen an der FRITZ!Box verlangen eine Bestätigung
**ausserhalb** der Weboberfläche: entweder ein Tastendruck am Gerät oder eine
Ziffernfolge an einem angeschlossenen Telefon.
 
Damit genügt ein gestohlenes Router-Passwort nicht, um aus der Ferne eine
VPN-Verbindung anzulegen oder einen Port zu öffnen. Praktische Folge: Solche
Änderungen sind nur zu Hause möglich. Die Portfreigabe für UDP 51821 wurde
deshalb am Morgen des 06.10.2026 zu Hause angelegt, die Einrichtung in der VM
erfolgte danach aus der Ferne.
 
Dabei wurde zuerst versehentlich "MyFRITZ!-Freigabe" gewählt. Diese Art ist
für Webdienste über TCP gedacht. WireGuard braucht eine normale
"Portfreigabe" mit dem Protokoll UDP.
 
## Tests
 
|                        Test                            |    Datum   |            Ergebnis             |
|                        ---                             |    ---     |              ---                |
| Router-VPN: Zugriff über Mobilfunk, per Adresse        | 04.10.2026 | bestanden                       |
| Router-VPN: Zugriff über Mobilfunk, per Name           | 04.10.2026 | bestanden                       |
| Nextcloud-App über VPN mit Zwei-Faktor-Authentisierung | 05.10.2026 | bestanden                       |
| Erreichbarkeit ohne VPN über Mobilfunk                 | 04.10.2026 | nicht erreichbar, wie gewollt   |
| Weboberfläche der FRITZ!Box aus dem Internet           | 04.10.2026 | nicht erreichbar, wie gewollt   |
| Router-VPN über einen Mobilfunk-Hotspot auf dem Laptop | 04.10.2026 | nach Korrektur der `AllowedIPs` |
| Server-VPN: Smartphone über Mobilfunk, per Adresse     | 06.10.2026 | bestanden                       |
| Server-VPN: Laptop über Mobilfunk-Freigabe, per Adresse| 06.10.2026 | bestanden                       |
| Server-VPN: Zugriff per Name über dnsmasq              | 06.10.2026 | bestanden                       |
| Server-VPN: Proxmox über den Tunnel                    | 06.10.2026 | nicht erreichbar, wie gewollt   |
| Server-VPN: SSH auf die VM, Weboberfläche der FRITZ!Box| 06.10.2026 | nicht erreichbar, wie gewollt   |
| Server-VPN: Upload 1.5 GB über Mobilfunk               | 06.10.2026 | 8 bis 10 Minuten, ca. 2.5 MB/s  |
| Server-VPN: alle Geräte der Benutzer verbunden         | 06.10.2026 | bestanden                       |
 
Beim Upload ist der Upload des Mobilfunknetzes die Grenze, nicht der Server.
 
## Verbleibende Risiken
 
**Verlorenes Gerät mit Schlüssel.** Deshalb ein Schlüssel je Gerät und ein
festgelegtes Vorgehen zur Sperrung.
 
**Teilnehmer des Router-VPN erreichen das ganze Heimnetz.** Seit dem
06.10.2026 hat nur noch das eigene Verwaltungsgerät einen Zugang dort. Die
Benutzer verwenden ausschliesslich das Server-VPN.
 
**DNS über den Server.** Ist der Server aus, fällt bei aktivem Server-VPN die
Namensauflösung des ganzen Geräts aus.
 
**Der Inhalt des Tunnels ist für ein überwachtes Netz nicht einsehbar.** Aus
Sicht des Betreibers eines Firmennetzes ist das ein Nachteil, kein Vorteil —
siehe die Absprache zu Tailscale.
 
**Die Weiterleitung lebt auf dem Router.** Wird die FRITZ!Box zurückgesetzt oder
ersetzt, verschwindet die Portfreigabe für UDP 51821, obwohl am Server nichts
geändert wurde. Gehört in eine Checkliste für den Routerwechsel.