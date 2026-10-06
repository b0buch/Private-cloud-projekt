# Ferienprojekt: Private Cloud
 
Aufbau einer eigenen privaten Cloud auf einem Server zu Hause, mit Nextcloud,
Docker und Zugriff über VPN.
 
**Lehrjahr:** 1. Lehrjahr Applikationsentwicklung
**Zeitraum:** September / Oktober 2026 (10 Arbeitstage)
 
---
 
## Ziel
 
Fotos und Videos (besonders Ski- und GoPro-Aufnahmen) auf einem eigenen Server
speichern und von PC und Smartphone darauf zugreifen. Insgesamt drei Benutzer,
jeder mit einem eigenen Konto und eigenem Speicherbereich.
 
## Kompetenzen
 
- Linux und Serveradministration
- Virtualisierung
- Docker
- Datenbanken und SQL
- Netzwerk und VPN
- Benutzer und Berechtigungen
- Backup und Datensicherheit
---
 
## Aufbau des Repositories
 
|       Ordner / Datei         |                      Inhalt                       |
|            ---               |                       ---                         |
| `journal/`                   | Arbeitsjournal, eine Datei pro Arbeitstag         |
| `docs/architektur.md`        | Aufbau des Systems und Rolle der Festplatten      |
| `docs/backup-konzept.md`     | Sicherungsstrategie und Begründung                |
| `docs/datenbank.md`          | SQL-Abfragen und Erkenntnisse zur Datenbank       |
| `docs/server.md`             | Hardware, Anschluss, Dienste und Zeitpläne        |
| `docs/sicherheit.md`         | Sicherheitskonzept und verbleibende Risiken       |
| `docs/vpn.md`                | Fernzugriff: Router-VPN und Server-VPN            |
| `docs/umzug-checkliste.md`   | Checkliste für den Umzug des Servers nach Hause   |
| `offen/loesungen.md`         | Lösungsideen für später                           |
| `offen/probleme.md`          | Offene und gelöste Probleme                       |
| `offen/tests.md`             | Geplante und erledigte Tests                      |
| `docker-compose.yml`         | Definition der vier Container                     |
| `nextcloud-image/Dockerfile` | Eigenes Image mit ffmpeg                          |
| `caddy/Caddyfile.example`    | Konfiguration des Reverse Proxy                   |
| `.env.example`               | Vorlage für die Passwortdatei                     |
 
## Stand des Projekts
 
Stand: 06.10.2026
 
|                   Bereich                  |                         Stand                            |
|                     ---                    |                          ---                             |
| Proxmox und Ubuntu-VM                      | fertig                                                   |
| Docker und Docker Compose                  | fertig                                                   |
| Nextcloud, MariaDB, Redis                  | fertig                                                   |
| Eigenes Docker-Image mit ffmpeg            | fertig                                                   |
| 4TB-Festplatte einbauen                    | fertig (02.10.2026)                                      |
| Datenfestplatte mit LVM                    | fertig (4TB, qcow2 auf LVM)                              |
| Umzug nach Hause                           | fertig (02.10.2026)                                      |
| HTTPS mit Reverse Proxy und Zertifikat     | fertig                                                   |
| Benutzer und Gruppen                       | fertig                                                   |
| Zwei-Faktor-Authentisierung                | fertig                                                   |
| Brute-Force-Schutz                         | geprüft                                                  |
| E-Mail-Versand                             | fertig                                                   |
| VPN für die Verwaltung (FRITZ!Box)         | fertig (04.10.2026)                                      |
| VPN für die Benutzer (Server-VPN)          | fertig (06.10.2026), nur Zugriff auf Nextcloud           |
| Namensauflösung im Server-VPN (dnsmasq)    | fertig (06.10.2026)                                      |
| Zugriff mit dem Smartphone                 | fertig                                                   |
| Backup der VM (vzdump)                     | fertig, verpasste Sicherungen werden wiederholt          |
| Backup der Datenbank (mysqldump)           | fertig, systemd-Timer holt verpasste Sicherungen nach    |
| Backup der Daten (rsync)                   | fertig, systemd-Timer holt verpasste Sicherungen nach    |
| Benachrichtigung bei Fehlern               | fertig (06.10.2026), E-Mail über `OnFailure=`            |
 
---
 
## Wichtiger Hinweis zur Sicherheit
 
In diesem Repository stehen **keine** echten Passwörter, Schlüssel,
IP-Adressen aus dem Internet oder DynDNS-Namen. In der Dokumentation werden
Platzhalter verwendet.
 
Über `.gitignore` ausgeschlossen sind insbesondere:
 
- `.env` mit den Passwörtern der Datenbank
- `server.key`, der private Schlüssel des Zertifikats
- alle WireGuard-Schlüssel und Konfigurationen der Teilnehmer (`*.key`, `*.conf`)
- `/etc/msmtprc` mit dem App-Passwort für den Mailversand (liegt nur auf dem Server)