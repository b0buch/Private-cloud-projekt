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

|       Ordner / Datei       |                      Inhalt                     |
|      ---          ---      |    ---                ---                ---    |
| `journal/`                 | Arbeitsjournal, eine Datei pro Arbeitstag       |
| `docs/architektur.md`      | Aufbau des Systems und Rolle der Festplatten    |
| `docs/server.md`           | Hardware und Anschluss                          |
| `docs/umzug-checkliste.md` | Checkliste für den Umzug des Servers nach Hause |
| `docs/datenbank.md`        | SQL-Abfragen und Erkenntnisse zur Datenbank     |
| `offen/probleme.md`        | Offene Probleme und Punkte für später           |
| `offen/loesungen.md`       | Lösungsideen für später                         |
| `offen/tests.md`           | Geplante und erledigte Tests                    |

## Stand des Projekts

|                   Bereich                  |                    Stand                  |
|    ---              ---             ---    |     ---             ---            ---    |
| Proxmox und Ubuntu-VM                      |                   fertig                  |
| Docker und Docker Compose                  |                   fertig                  |
| Nextcloud, MariaDB, Redis                  |                   fertig                  |
| Datenfestplatte und Verschiebung der Daten | fertig (Test mit externer 1TB-Festplatte) |
| Backup der VM (vzdump)                     |                   fertig                  |
| Benutzer und Gruppen                       |                   in Arbeit               |
| Backup der Datenbank (mysqldump)           |                   offen                   |
| HTTPS                                      |                   offen                   |
| VPN (WireGuard)                            | offen, erst nach dem Umzug nach Hause     |
| Zugriff mit dem Smartphone                 | offen, erst nach dem Umzug nach Hause     |

---

## Wichtiger Hinweis zur Sicherheit

In diesem Repository stehen **keine** echten Passwörter, IP-Adressen aus dem
Internet oder DynDNS-Namen. Die Passwörter der Datenbank liegen auf dem Server
in der Datei `.env`, die über `.gitignore` ausgeschlossen ist.
In der Dokumentation werden Platzhalter verwendet.
