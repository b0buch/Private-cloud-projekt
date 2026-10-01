# Sicherheitskonzept

Stand: 01.10.2026

Die Sicherheit beruht nicht auf einer einzelnen Massnahme, sondern auf mehreren
unabhängigen Schichten. Fällt eine aus, schützen die übrigen weiter. Dieses
Prinzip heisst **Defense in Depth**.

## Übersicht der Schichten

|           Schicht           |        Wogegen sie schützt             |     Stand    |
|             ---             |               ---                      |      ---     |
| VPN                         | Zugriff aus dem Internet               | geplant      |
| Firewall (ufw)              | nicht benötigte offene Ports           | eingerichtet |
| HTTPS                       | Mitlesen des Datenverkehrs             | fertig       |
| Zertifikat                  | gefälschter Server (Man-in-the-Middle) | fertig       |
| Sicherheitsheader           | typische Angriffe über den Browser     | fertig       |
| Anmeldung                   | Zugriff durch Unbefugte                | fertig       |
| Zwei-Faktor-Authentisierung | gestohlenes Passwort                   | fertig       |
| Brute-Force-Schutz          | Durchprobieren von Passwörtern         | aktiv        |
| Benutzer und Gruppen        | Zugriff auf fremde Dateien             | in Arbeit    |
| Backup                      | Datenverlust                           | teilweise    |

## HTTPS und Zertifikat

### Ablauf einer Verbindung

1. Der Browser öffnet eine Verbindung auf Port 443
2. Der **Server** sendet sein Zertifikat
3. Der **Browser** prüft: passt der Name, wer hat unterschrieben, ist es noch
   gültig
4. Beide Seiten handeln die Schlüssel für die Verschlüsselung aus
5. Erst danach werden die eigentlichen Daten übertragen

Das Zertifikat gehört also dem Server und wird vom Browser des Benutzers
geprüft, nicht umgekehrt.

### Zwei getrennte Aufgaben

Ein Zertifikat erfüllt zwei Aufgaben, die oft verwechselt werden:

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

Der Benutzer bemerkt nichts: Das Schloss im Browser ist sichtbar, die
Verschlüsselung funktioniert. Nur die fehlende Unterschrift einer bekannten
Zertifizierungsstelle verrät den Angriff.

> Das Schloss im Browser bedeutet "die Verbindung ist verschlüsselt", nicht
> "der Server ist echt".

### Umsetzung im Projekt

Das Zertifikat wurde mit openssl selbst erstellt:

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout server.key -out server.crt -days 825 \
  -subj "/C=CH/ST=Bern/L=Bern/O=Private Cloud/CN=<adresse>" \
  -addext "subjectAltName=IP:<adresse>,DNS:localhost"
```

Wichtig ist das Feld `subjectAltName`. Moderne Browser werten nur dieses Feld
aus, das ältere Feld `CN` wird ignoriert.

Weil das Zertifikat die Eigenschaft `Basic Constraints: CA:TRUE` besitzt, kann
es auf den Geräten als Stammzertifikat installiert werden. Danach bestätigt der
Browser die Identität des Servers, und die Warnung verschwindet.

Praktisch nachgewiesen am 01.10.2026: Die Verbindung war vorher und nachher
gleich stark verschlüsselt, der Unterschied lag allein im Vertrauen.

### Warum kein Let's Encrypt

Ein Zertifikat von Let's Encrypt wäre auf allen Geräten ohne Installation
vertrauenswürdig. Die übliche Überprüfung verlangt aber, dass der Server aus
dem Internet über die Ports 80 und 443 erreichbar ist. Genau das widerspricht
dem Konzept, dass die Cloud ausschliesslich über VPN zugänglich sein soll.

Die Alternative wäre eine Überprüfung über einen DNS-Eintrag. Dafür wird eine
eigene Domain mit passender Schnittstelle benötigt. Diese Möglichkeit bleibt
für später offen.

## Reverse Proxy

Caddy nimmt alle Anfragen entgegen und leitet sie intern weiter. Das bringt
drei Vorteile für die Sicherheit:

- Die Verschlüsselung ist an einer einzigen Stelle konfiguriert
- Nextcloud veröffentlicht keinen eigenen Port mehr und ist nur noch über den
  Proxy erreichbar
- Die Sicherheitsheader gelten automatisch für alle künftigen Dienste

Gesetzte Header:

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
wechselt alle 30 Sekunden und wird nie über das Netz übertragen.

Wer das Passwort stiehlt, kommt ohne das Smartphone nicht hinein.

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
verzögert die Antworten zunehmend, bis zu 30 Sekunden.

Das ist bewusst so gelöst: Eine echte Sperrung könnte ein Angreifer missbrauchen,
um fremde Konten absichtlich unbrauchbar zu machen. Ein Mensch, der sich zweimal
vertippt, merkt von der Verzögerung nichts, ein Programm mit tausenden Versuchen
schon.

Die Daten werden je nach Konfiguration in der Datenbank oder im Cache abgelegt.
In diesem Projekt ist Redis als verteilter Cache eingerichtet, deshalb verwendet
Nextcloud den `MemoryCacheBackend`.

### Zusammenhang mit dem Proxy

Ohne die Einstellung `trusted_proxies` würde Nextcloud alle Anfragen der Adresse
des Proxy-Containers zuordnen. Die Folge: Ein einziger Angreifer würde die
Antworten für **alle** Benutzer verlangsamen.

Mit der richtigen Einstellung wertet Nextcloud den Header `X-Forwarded-For` aus
und erkennt die tatsächliche Adresse. Am 01.10.2026 wurde das überprüft:
Nextcloud meldete die Adresse des Laptops, nicht die des Containers.

## Benutzer und Berechtigungen

- Jeder Benutzer erhält einen eigenen, isolierten Bereich
- Gruppen dienen als Grundlage für gemeinsame Ordner
- Gemeinsame Ordner (Team folders) gehören keinem Benutzer und besitzen eine
  eigene Speicherbegrenzung
- Passwörter werden als Hash gespeichert, nicht im Klartext

**Geplant:** Trennung von Administration und täglicher Nutzung. Das Konto für
den Alltag erhält keine Administratorrechte, damit ein kompromittiertes Konto
nicht gleich die ganze Installation gefährdet.

## Verbleibende Risiken

Diese Punkte sind bekannt und bewusst in Kauf genommen oder für später geplant:

**Verlorenes Smartphone mit VPN-Schlüssel.** Deshalb erhält jedes Gerät eine
eigene Verbindung, damit einzelne Schlüssel gesperrt werden können, ohne die
anderen zu beeinträchtigen.

**Keine Verschlüsselung der Festplatten.** Bei einem Diebstahl des Servers
liessen sich die Daten an einem anderen Computer lesen.

**Schwache Passwörter der Benutzer.** Innerhalb des VPN wäre das bereits
gefährlich. Die Benutzer müssen entsprechend angewiesen werden.

**Mitgliedschaft in der Gruppe `docker`.** Sie entspricht faktisch
Root-Rechten, weil über einen Container auf das ganze System zugegriffen werden
kann.

**Eigene Fehler.** Eine versehentlich offen gelassene Portweiterleitung ist die
häufigste Ursache für Probleme bei selbst betriebenen Servern.

**Unvollständiges Backup.** Von den Daten werden nur die als wichtig
eingestuften gesichert. Die Entscheidung ist bewusst getroffen und in
`backup-konzept.md` begründet.
