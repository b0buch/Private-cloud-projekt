# Fernzugriff über VPN

Stand: 05.10.2026

## Grundsatz

Nextcloud ist aus dem Internet **nicht** erreichbar. Es gibt keine
Portweiterleitung auf Port 443 und keinen öffentlich erreichbaren Webserver.
Wer von aussen zugreifen will, baut zuerst einen VPN-Tunnel auf und befindet
sich danach logisch im Heimnetz.

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

|                           | WireGuard auf der FRITZ!Box | WireGuard auf dem Server | WireGuard über IPv6 |       Tailscale       |
|          ---              |            ---              |           ---            |        ---          |         ---           |
| Kosten                    | keine                       | keine                    | keine               | gratis bis 100 Geräte |
| Funktioniert hinter CGNAT | nein                        | nein                     | ja                  | ja                    |
| Aus IPv4-only-Netz        | ja                          | ja                       | **nein**            | ja                    |
| Zugriff einschränkbar     | **nein**                    | **ja**                   | nein                | teilweise             |
| Aufwand beim Einrichten   | klein                       | mittel                   | klein               | klein                 |
| Lerneffekt                | mittel                      | hoch                     | mittel              | gering                |
| Abhängigkeit von Dritten  | Namensdienst von AVM        | Namensdienst von AVM     | keine               | Koordinationsserver   |

**Entscheidung:** WireGuard auf der FRITZ!Box für das eigene Konto, weil es
Zugang zum ganzen Heimnetz für die Verwaltung braucht. Für weitere Benutzer ein
zweiter WireGuard-Server auf der VM, dessen Teilnehmer nur Nextcloud erreichen.
Tailscale wird zum Vergleich getestet, aber nicht im Betrieb verwendet, weil es
einen fremden Koordinationsserver voraussetzt.

Tailscale wurde im Firmennetz nach Absprache mit dem Berufsbildner **nicht**
installiert. Ein verschlüsselter Tunnel, der an der Überwachung des Firmennetzes
vorbeiläuft, ist aus Sicht der Netzwerksicherheit schwer zu beurteilen, auch
wenn er nur ein einzelnes Gerät verbindet. Dieselbe Überlegung gilt für
WireGuard.

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
   10.9.0.2  ──HTTPS 443──►  10.9.0.1
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
Verdoppelung ohne Zweck: Wer sich schon im VPN-Netz befindet, würde unverschlüsselten
Verkehr mitlesen können.

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

## Split Tunnel und Full Tunnel

|                            |          Split Tunnel             |          Full Tunnel               |
|          ---               |              ---                  |              ---                   |
| `AllowedIPs` beim Client   | nur das Heimnetz                  | `0.0.0.0/0, ::/0`                  |
| Was durch den Tunnel geht  | nur Verkehr zum Server            | **aller** Verkehr des Geräts       |
| Übriger Verkehr            | direkt über den Anbieter          | über den Hausanschluss             |
| Belastung des Anschlusses  | gering                            | der ganze Verkehr, 50 Mbit/s       |

Für den Zugriff auf die eigene Cloud ist Split Tunnel richtig.

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
über die zweite. Die erste Zeile ist überflüssig, weil `0.0.0.0/0` sie
einschliesst, aber nicht widersprüchlich. Das Ergebnis ist Full Tunnel.

Dass die Verbindung **vollständig** ausfiel und nicht nur langsam wurde, hat
eine andere Ursache: Enthält `AllowedIPs` den Eintrag `0.0.0.0/0`, aktiviert der
WireGuard-Client für Windows eine Sperre für Verkehr ausserhalb des Tunnels,
einen **Kill Switch**. Er soll verhindern, dass bei einem Abbruch Daten am VPN
vorbei abfliessen. Nebenwirkung: Trägt der Tunnel keinen Verkehr nach aussen,
sieht man keinen Teilausfall, sondern gar kein Netz — und kommt vom Symptom
nicht auf die Ursache.

**Merksatz für die Fehlersuche:** Bei VPN-Problemen zuerst `AllowedIPs`
ansehen, nicht das Netz.

## `AllowedIPs` bedeutet auf beiden Seiten etwas anderes

|   Seite   |                       Bedeutung                        |             Durchsetzbar?               |
|    ---    |                          ---                           |                  ---                    |
| Client    | wohin Verkehr geschickt wird (Routenhinweis)           | nein — der Client kann die Zeile ändern |
| Server    | welche Absenderadresse dieser Schlüssel verwenden darf | ja — kryptografisch geprüft             |

Das ist der Grund, warum eine Einschränkung beim Client **keine**
Sicherheitsmassnahme ist. Wer Zugriff beschränken will, muss es auf dem Server
tun.

## Kollision der Subnetze

Am 04.10.2026 fiel die Verbindung über einen Mobilfunk-Hotspot zunächst aus.
Vermutet wurde eine Kollision: Wenn der Hotspot Adressen aus `192.168.178.0/24`
vergibt, zeigen zwei Routen auf dasselbe Netz, und der eigene Gateway wird in
den Tunnel geschickt. Die Prüfung mit `ipconfig` ergab allerdings `10.231.x.x` —
die Vermutung war falsch, die Ursache war der Kill Switch.

Das Risiko bleibt trotzdem bestehen und ist in `probleme.md` vermerkt:
`192.168.178.0/24` ist die Werkseinstellung jeder FRITZ!Box und kollidiert
deshalb häufig mit fremden Netzen. Geplant ist die Umstellung auf einen seltenen
Bereich.

## Schlüsselverwaltung

**Ein eigener Schlüssel je Gerät.** Geht ein Gerät verloren, wird genau dieser
Schlüssel gesperrt, die übrigen laufen weiter. Zwei Geräte mit demselben
Schlüssel würden sich gegenseitig verdrängen, weil der Server sie für dasselbe
Gerät hält.

**Was in einer Konfiguration geheim ist:**

|      Feld      |   Geheim? |          Wirkung bei Offenlegung               |
|      ---       |    ---    |                   ---                          |
| `PrivateKey`   | ja        | Fremder kann sich als dieses Gerät ausgeben    |
| `PresharedKey` | ja        | zusätzlicher Schutz entfällt                   |
| `Endpoint`     | teilweise | verrät die öffentliche Adresse des Anschlusses |
| `AllowedIPs`   | nein      | nur ein Routenhinweis                          |
| `Address`      | nein      | Adresse im Tunnel                              |
 
Ein QR-Code enthält die vollständige Konfiguration samt privatem Schlüssel.

**Vorgehen bei Verdacht auf Offenlegung:** Die Verbindung in der FRITZ!Box
löschen und neu anlegen, danach die neue Konfiguration auf dem Gerät
importieren und die alte entfernen. Der alte Schlüssel wird damit wertlos. Am
05.10.2026 für den Laptop durchgeführt, nachdem die Konfiguration
möglicherweise in einen Chatverlauf geraten war. Grundregel: Ein Geheimnis, das
den eigenen Einflussbereich verlassen haben **könnte**, gilt als offengelegt.

## Zweiter Faktor der FRITZ!Box

Sicherheitsrelevante Änderungen an der FRITZ!Box verlangen eine Bestätigung
**aussserhalb** der Weboberfläche: entweder ein Tastendruck am Gerät oder eine
Ziffernfolge, die an einem an der FRITZ!Box angeschlossenen Telefon gewählt
wird.

Damit genügt ein gestohlenes Router-Passwort nicht, um aus der Ferne eine
VPN-Verbindung anzulegen oder einen Port zu öffnen. Praktische Folge: Solche
Änderungen sind nur zu Hause möglich.

Zusammen mit TOTP in Nextcloud gibt es damit zwei unabhängige
Zweitfaktor-Verfahren auf verschiedenen Ebenen — eines über eine App, eines
über die Telefonie.

## Geplant: zweiter WireGuard-Server auf der VM

### Problem

Teilnehmer des VPN auf der FRITZ!Box erreichen das **ganze** Heimnetz: die
Weboberfläche des Routers, Proxmox, jedes andere Gerät. Die FRITZ!Box 7490 kann
den Zugriff einzelner Teilnehmer nicht einschränken — es gibt keine Regeln je
Teilnehmer und keine Möglichkeit, sie in ein getrenntes Netzsegment zu legen.

### Lösung

Ein eigener WireGuard-Server in der Ubuntu-VM, auf einem anderen Port. Die
FRITZ!Box leitet UDP 51821 nur weiter.

| Verbindung              | Ort       | Port  | Zugriff auf        | Für wen          |
| ---                     | ---       | ---   | ---                | ---              |
| WireGuard FRITZ!Box     | Router    | 51820 | ganzes Heimnetz    | nur Verwaltung   |
| WireGuard Server        | Ubuntu-VM | 51821 | nur Nextcloud      | weitere Benutzer |

Die Teilnehmer landen in `10.9.0.0/24` — einem Netz, das nur innerhalb der VM
existiert. Das Heimnetz gibt es für sie nicht.

### Durchsetzung über Regeln, nicht über fehlende Weiterleitung

Naheliegend wäre, die Paketweiterleitung im Kernel abzuschalten. Das geht hier
nicht: **Docker setzt `net.ipv4.ip_forward=1` selbst**, sonst hätten die
Container kein Netz. Die Einschränkung muss deshalb über die Firewall erfolgen:

```bash
ufw allow 51821/udp
ufw route deny from 10.9.0.0/24
ufw insert 1 deny in on wg0 to any port 22 proto tcp
```

- die erste Regel öffnet den Port des Tunnels
- die zweite verbietet jede **Weiterleitung** aus dem VPN-Netz, womit Proxmox,
  Router und alle anderen Geräte unerreichbar werden
- die dritte schliesst SSH auf der VM selbst; `insert 1` setzt das Verbot
  **vor** die erlaubenden Regeln, denn in ufw entscheidet die Reihenfolge

Das ist nachvollziehbarer als eine abgeschaltete Einstellung: Die Regeln lassen
sich zeigen, erklären und prüfen.

### Zertifikat

Die Teilnehmer greifen über `https://10.9.0.1` zu. Damit kein Zertifikatsfehler
erscheint, wird das Zertifikat um die Adresse im Feld `subjectAltName` erweitert:

```
subjectAltName = DNS:privatecloud.fritz.box, IP:10.9.0.1, IP:192.168.178.220, DNS:localhost
```

Ein Namensdienst innerhalb des Tunnels ist dadurch nicht nötig.

## Tests

|                        Test                            |    Datum   |            Ergebnis             |
|                        ---                             |    ---     |              ---                |
| Zugriff über Mobilfunk mit VPN, per Adresse            | 04.10.2026 | bestanden                       |
| Zugriff über Mobilfunk mit VPN, per Name               | 04.10.2026 | bestanden                       |
| Nextcloud-App über VPN mit Zwei-Faktor-Authentisierung | 05.10.2026 | bestanden                       |
| Erreichbarkeit ohne VPN über Mobilfunk                 | 04.10.2026 | nicht erreichbar, wie gewollt   |
| Weboberfläche der FRITZ!Box aus dem Internet           | 04.10.2026 | nicht erreichbar, wie gewollt   |
| Zugriff über einen Mobilfunk-Hotspot auf dem Laptop    | 04.10.2026 | nach Korrektur der `AllowedIPs` |

Noch offen: Zugriffsmatrix für das Server-VPN (Nextcloud erreichbar, Proxmox und
Router nicht), Geschwindigkeit beim Hochladen eines grossen Videos über VPN,
Vergleich mit Tailscale.

## Verbleibende Risiken

**Verlorenes Gerät mit Schlüssel.** Deshalb ein Schlüssel je Gerät und ein
festgelegtes Vorgehen zur Sperrung.

**Teilnehmer des Router-VPN erreichen das ganze Heimnetz.** Zugang erhält
deshalb nur, wem auch physischer Zugang zur Wohnung zugestanden wird. Für
weitere Benutzer ist der Server-VPN vorgesehen.

**Der Inhalt des Tunnels ist für ein überwachtes Netz nicht einsehbar.** Aus
Sicht des Betreibers eines Firmennetzes ist das ein Nachteil, kein Vorteil —
siehe die Absprache zu Tailscale.

**Die Weiterleitung lebt auf dem Router.** Wird die FRITZ!Box zurückgesetzt oder
ersetzt, verschwindet sie, obwohl am Server nichts geändert wurde. Gehört in
eine Checkliste für den Routerwechsel.