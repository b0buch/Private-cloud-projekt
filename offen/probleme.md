# Probleme
 
Stand: 06.10.2026
 
Offene Punkte oben, gelöste in der Tabelle am Ende.
 
## Netzwerk
 
**Das Heimnetz 192.168.178.0/24 ist die Werkseinstellung der FRITZ!Box.**
Dadurch kann es mit fremden Netzen kollidieren: Ein Mobilfunk-Hotspot oder ein
Gastnetz mit demselben Bereich macht das Router-VPN unbrauchbar, weil zwei
Routen auf dasselbe Netz zeigen. Beim Server-VPN betrifft es nur die eine
Adresse `192.168.178.220`, das Risiko ist kleiner, aber nicht null. Geplant:
Umstellung auf einen seltenen Bereich wie `10.17.23.0/24`. Das erfordert neue
Reservierungen, Anpassungen an beiden VPN, am Zertifikat und an der
Dokumentation.
 
**DNS der Benutzer hängt am Server.** Mit `DNS = 10.9.0.1` gehen bei aktivem
Server-VPN alle Namensanfragen über dnsmasq. Ist der Server aus, öffnen sich
auf dem Gerät auch andere Webseiten nicht, bis das VPN ausgeschaltet wird.
Vorerst ein Hinweis in der Anleitung für die Benutzer. Dauerhafte Lösung wäre
eine eigene Domain mit einem öffentlichen DNS-Eintrag (siehe
`sicherheit.md`).
 
**Alte Einträge in `trusted_domains`.** `192.168.137.220` und `192.168.137.*`
stammen aus dem Netz des Arbeitsplatzes. Wirkungslos, aber die Konfiguration
soll den tatsächlichen Zustand zeigen. Zu entfernen.
 
## Nextcloud
 
**Reste der internen Zertifizierungsstelle von Caddy.** Unter
`caddy/data/caddy/pki/` und `caddy/data/caddy/certificates/local/` liegen
Dateien vom Versuch am 30.09.2026. Sie werden nicht verwendet und können
entfernt werden.
 
## Speicher und Backup
 
**Die Wiederherstellung aus der rsync-Sicherung ist nicht getestet.** VM und
Datenbank wurden zurückgespielt, die Daten auf der externen Festplatte noch
nicht. Geht nur zu Hause, weil die Festplatte getrennt aufbewahrt wird.
 
**Eine fehlgeschlagene Sicherung der Daten wird nicht wiederholt.** Ist die
externe Festplatte zur geplanten Zeit nicht angeschlossen, bricht das Skript
ab und es kommt eine E-Mail. `Persistent=true` holt nur **verpasste**
Ausführungen nach, nicht **fehlgeschlagene**. Nach dem Anschliessen wird die
Sicherung von Hand gestartet.
 
## Sicherheit
 
**Auch mit VPN bleiben Risiken bestehen.** Wenn ein Gerät mit einem
WireGuard-Schlüssel verloren geht, hat der Finder Zugriff auf Nextcloud (bei
einem Benutzer) bzw. auf das Heimnetz (beim eigenen Verwaltungsgerät).
Deshalb hat jedes Gerät eine eigene Verbindung, damit einzelne Schlüssel
gesperrt werden können.
 
**Das Stammzertifikat auf den Geräten der Benutzer.** Wer `server.key`
besitzt, könnte Zertifikate für beliebige Webseiten ausstellen. Siehe
`sicherheit.md`.
 
**Die Festplatten sind nicht verschlüsselt.** Bei einem Diebstahl des Servers
könnten die Daten gelesen werden. Zu prüfen, ob eine Verschlüsselung sinnvoll
ist.
 
**Die Datei `.env` enthält die Passwörter der Datenbank** und darf nie in Git
aufgenommen werden. Sie steht in der `.gitignore`, ebenso `*.key` und die
Konfigurationen der WireGuard-Teilnehmer.
 
**In der öffentlichen Dokumentation werden keine echten öffentlichen
Adressen und keine DynDNS-Namen verwendet**, sondern Platzhalter.
 
## Hinweise ohne Handlungsbedarf
 
**Abgebrochene Proxmox-Befehle hinterlassen Reste.** Ein unterbrochenes
`qm set` legt die Datei an, schreibt aber die Konfiguration nicht. `qm rescan`
findet solche Reste normalerweise — bei einem unvollständigen qcow2 nicht,
weil der Kopf fehlt und Proxmox die Datei nicht als Datenträger erkennt.
 
**Die Laufwerksbuchstaben wechseln.** Am 04.10.2026 tauschten `sdb` und `sdc`
ohne Zutun die Plätze. Deshalb stehen in `/etc/fstab` UUIDs und keine
Gerätenamen.
 
**Meldung "Keine gültige Subskription" in Proxmox.** Erscheint bei jeder
Anmeldung ohne Abonnement und lässt sich nicht offiziell abschalten. Sie hat
keinen Einfluss auf die Funktion.
 
## Gelöst
 
|                    Problem                           |                   Gelöst am                           |
|                      ---                             |                      ---                              |
| Backup der Datenbank fehlte                          | 29.09.2026, mysqldump mit Wiederherstellungstest      |
| Zugriff nur über HTTP, kein HTTPS                    | 30.09.2026                                            |
| Maximale Uploadgrösse zu klein für grosse Videos     | 30.09.2026                                            |
| ffmpeg nur im laufenden Container installiert        | 01.10.2026, eigenes Dockerfile                        |
| Zwei-Faktor-Authentisierung nicht aktiviert          | 01.10.2026                                            |
| Daten lagen auf der externen 1TB-Festplatte          | 02.10.2026, Umzug auf 4TB                             |
| Server war nur über die Internetfreigabe erreichbar  | 02.10.2026, Umzug nach Hause                          |
| Docker startete vor dem Einbinden der Datenfestplatte| 04.10.2026, `RequiresMountsFor=/mnt/clouddata`        |
| Annahme Carrier-Grade-NAT war falsch                 | 04.10.2026, öffentliche IPv4 vorhanden                |
| Selbstprüfungen von Nextcloud schlugen fehl          | 05.10.2026, Ursache DNS der VM                        |
| E-Mail-Server nicht konfiguriert                     | 05.10.2026, SMTP über Gmail                           |
| Benutzerkonten und Gruppen provisorisch              | 05.10.2026                                            |
| Trennung von Administration und Alltag fehlte        | 05.10.2026                                            |
| 4 TB Daten, aber nur 1 TB für die Sicherung          | 06.10.2026, Regel: nur der Ordner Foto-Video          |
| VPN-Teilnehmer der FRITZ!Box erreichen das Heimnetz  | 06.10.2026, Server-VPN für die Benutzer               |
| Geplante Aufgaben liefen nur im Betrieb              | 06.10.2026, systemd-Timer mit `Persistent=true`       |
| Dumps der Datenbank für alle Benutzer lesbar         | 06.10.2026, `chmod 600` und `UMask=0077`              |
| Fehler einer Sicherung fiel niemandem auf            | 06.10.2026, E-Mail über `OnFailure=`                  |
| `apt-get update` auf Proxmox schlug fehl             | 06.10.2026, Paketquelle `no-subscription`             |
 