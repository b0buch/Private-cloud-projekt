# Offene Probleme

Punkte, die noch offen sind oder erst nach dem Umzug nach Hause gelöst werden
können.

## Netzwerk

**Die Heimnetz-Subnetz 192.168.178.0/24 ist die Werkseinstellung der
FRITZ!Box.** Dadurch kollidiert sie mit fremden Netzen: Ein Mobilfunk-Hotspot
oder ein Gastnetz mit demselben Bereich macht die VPN-Verbindung unbrauchbar,
weil zwei Routen auf dieselbe Subnetz zeigen. Geplant: Umstellung auf einen
seltenen Bereich wie 10.17.23.0/24. Das erfordert neue Reservierungen, neue
WireGuard-Verbindungen und Anpassung der Dokumentation.

**VPN-Teilnehmer der FRITZ!Box erreichen das ganze Heimnetz.** Der Router
kann den Zugriff einzelner Teilnehmer nicht einschraenken. Geplant: ein
zweiter WireGuard-Server auf der VM, dessen Teilnehmer ueber Firewall-Regeln
nur auf Nextcloud zugreifen koennen.

**Geplante Aufgaben laufen nur im Betrieb.** Siehe server.md.

## Nextcloud

**Mehrere Selbstprüfungen schlagen fehl** (WebDAV, `.well-known`, OCS,
Schriftarten, JavaScript-Module, Datenverzeichnis), weil der Webserver sich über
die Internetfreigabe des Laptops nicht selbst erreichen kann. Nach dem Umzug nach
Hause muss geprüft werden, ob die Meldungen verschwinden.

**Der E-Mail-Server ist noch nicht konfiguriert.** Ohne ihn können Benutzer ihr
Passwort nicht selbst zurücksetzen.

**Benutzerkonten und Gruppen werden neu angelegt.** Die endgültigen Konten,
Gruppen und gemeinsamen Ordner werden in einem Durchgang erstellt, sobald das
Konzept feststeht. Das Administratorkonto bleibt bis dahin unverändert.

**Trennung von Administration und täglicher Nutzung fehlt.** Geplant ist ein
eigenes Konto für die Administration, das tägliche Konto ohne
Administratorrechte.

## Speicher und Backup

**Die Datenmenge kann bis zu 4 TB betragen, für Backups steht aber nur eine
1TB-Festplatte zur Verfügung.** Es muss festgelegt werden, welche Daten gesichert
werden (wichtige Fotos und ausgewählte Videos) und welche nicht.

**Die Daten liegen aktuell auf der externen 1TB-Festplatte.** Sie muss später auf
die 4TB-Festplatte verschoben werden, dann geht die externe Festplatte ins
Backup.

**Docker startet vor dem Einbinden der Datenfestplatte.** Beim Hochfahren war
`/mnt/clouddata` noch nicht eingebunden, Docker band das leere Verzeichnis in
den Container ein, und Nextcloud meldete `.ncdata` fehle. Die Daten waren
unversehrt. Geloest mit einer Ergaenzung fuer den Dienst:
`RequiresMountsFor=/mnt/clouddata`. Geloest am 04.10.2026.

**Die Laufwerksbuchstaben wechseln.** Am 04.10.2026 tauschten `sdb` und `sdc`
ohne Zutun die Plaetze. Deshalb stehen in `/etc/fstab` UUIDs und keine
Gerätenamen — mit Buchstaben waere die Sicherungsfestplatte als
Datenverzeichnis eingebunden worden.

**Abgebrochene Proxmox-Befehle hinterlassen Reste.** Ein unterbrochenes
`qm set` legt die Datei an, schreibt aber die Konfiguration nicht. `qm rescan`
findet solche Reste normalerweise — bei einem unvollstaendigen qcow2 nicht,
weil der Kopf fehlt und Proxmox die Datei nicht als Datentraeger erkennt.

## Proxmox

**Beim Aktualisieren der Paketliste erscheint ein Fehler** (`apt-get update`).
Vermutlich sind die Enterprise-Paketquellen ohne Abonnement eingetragen. Zu
prüfen unter `pve` → Updates → Repositories.

## Sicherheit

**Auch mit VPN bleiben Risiken bestehen.** Wenn ein Smartphone mit einem
WireGuard-Schlüssel verloren geht, hat der Finder Zugriff auf das Netzwerk.
Deshalb bekommt jedes Gerät eine eigene Verbindung, damit einzelne Schlüssel
gesperrt werden können.

**Die Festplatten sind nicht verschlüsselt.** Bei einem Diebstahl des Servers
könnten die Daten gelesen werden. Zu prüfen, ob eine Verschlüsselung sinnvoll
ist.

**Die Datei `.env` enthält die Passwörter der Datenbank** und darf nie in Git
aufgenommen werden. Sie steht in der `.gitignore`.

**In der öffentlichen Dokumentation werden keine echten IP-Adressen und keine
DynDNS-Namen verwendet**, sondern Platzhalter.

## Smartphones

**Die Smartphones der Benutzer (Android) haben eine aggressive
Akku-Optimierung.** Diese kann den automatischen Upload von Fotos im Hintergrund
stoppen. Für die Nextcloud-App muss die Akku-Optimierung deaktiviert werden, bei
Xiaomi zusätzlich der Autostart erlaubt werden.

## Gelöst

|                    Problem                           |                   Gelöst am                      |
|                      ---                             |                      ---                         |
| Zugriff nur über HTTP, kein HTTPS                    | 30.09.2026                                       |
| Maximale Uploadgrösse zu klein für grosse Videos     | 30.09.2026                                       |
| ffmpeg nur im laufenden Container installiert        | 01.10.2026, eigenes Dockerfile                   |
| Zwei-Faktor-Authentisierung nicht aktiviert          | 01.10.2026                                       |
| Backup der Datenbank fehlte                          | 29.09.2026, mysqldump mit Wiederherstellungstest |
| Annahme Carrier-Grade-NAT war falsch                 | 04.10.2026, oeffentliche IPv4 vorhanden          |
| Selbstpruefungen von Nextcloud schlugen fehl         | 05.10.2026, Ursache DNS der VM                   |
| E-Mail-Server nicht konfiguriert                     | 05.10.2026, SMTP ueber Gmail                     |
| Benutzerkonten und Gruppen provisorisch              | 05.10.2026                                       |
| Trennung von Administration und Alltag fehlte        | 05.10.2026                                       |
| Daten lagen auf der externen 1TB-Festplatte          | 02.10.2026                                       |
| apt-get update auf Proxmox schlug fehl               | 02.10.2026, no-subscription                      |
| Server war nur ueber die Internetfreigabe erreichbar | 02.10.2026, Umzug nach Hause                     | 

