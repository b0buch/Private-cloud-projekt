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

|       Ordner / Datei         |                      Inhalt                     |
|      ---          ---        |    ---                ---                ---    |
| `journal/`                   | Arbeitsjournal, eine Datei pro Arbeitstag       |
| `docs/architektur.md`        | Aufbau des Systems und Rolle der Festplatten    |
| `docs/backup-konzept.md`     | Sicherungsstrategie und Begründung              |
| `docs/datenbank.md`          | SQL-Abfragen und Erkenntnisse zur Datenbank     |
| `docs/server.md`             | Hardware und Anschluss                          |
| `docs/sicherheit.md`         | Sicherheitskonzept und verbleibende Risiken     |
| `docs/umzug-checkliste.md`   | Checkliste für den Umzug des Servers nach Hause |
| `offen/loesungen.md`         | Lösungsideen für später                         |
| `offen/probleme.md`          | Offene Probleme und Punkte für später           |
| `offen/tests.md`             | Geplante und erledigte Tests                    |
| `docker-compose.yml`         | Definition der vier Container                   |
| `nextcloud-image/Dockerfile` | Eigenes Image mit ffmpeg                        |
| `caddy/Caddyfile.example`    | Konfiguration des Reverse Proxy                 |
| `.env.example`               | Vorlage für die Passwortdatei                   |


## Stand des Projekts

|                   Bereich                  |                    Stand                  |
|    ---              ---             ---    |     ---             ---            ---    |
| Proxmox und Ubuntu-VM                      | fertig                                    |
| Docker und Docker Compose                  | fertig                                    |
| Nextcloud, MariaDB, Redis                  | fertig                                    |
| Datenfestplatte und Verschiebung der Daten | fertig (4TB, qcow2 auf LVM)               |
| Backup der VM (vzdump)                     | fertig                                    |
| Benutzer und Gruppen                       | fertig                                    |
| Backup der Datenbank (mysqldump)           | fertig                                    |
| VPN (WireGuard)                            | Router fertig, Server-VPN geplant         |
| Zugriff mit dem Smartphone                 | fertig                                    |
| Eigenes Docker-Image mit ffmpeg            | fertig                                    |
| Datenfestplatte mit LVM                    | fertig                                    |
| HTTPS mit Reverse Proxy und Zertifikat     | fertig                                    |
| Zwei-Faktor-Authentisierung                | fertig                                    |
| Brute-Force-Schutz                         | geprüft                                   |
| 4TB-Festplatte einbauen                    | fertig (02.10.2026)                       |
| Umzug nach Hause                           | fertig (02.10.2026)                       |
| E-Mail-Versand                             | fertig                                    |  
| Geplante Aufgaben (cron)                   | eingerichtet, laufen nur im Betrieb       |

---

## Wichtiger Hinweis zur Sicherheit

In diesem Repository stehen **keine** echten Passwörter, IP-Adressen aus dem
Internet oder DynDNS-Namen. Die Passwörter der Datenbank liegen auf dem Server
in der Datei `.env`, die über `.gitignore` ausgeschlossen ist.
In der Dokumentation werden Platzhalter verwendet.
