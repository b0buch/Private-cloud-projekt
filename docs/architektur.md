# Architektur

## Übersicht

```
                    ┌─────────────────────────────┐
                    │  Server (Hardware)          │
                    │  i5-11400, 16GB RAM         │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  Proxmox VE 9.2             │
                    │  (Virtualisierung)          │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  VM 100: Ubuntu Server      │
                    │  4 CPU-Kerne, 8 GB RAM      │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  Docker Engine              │
                    └──────────────┬──────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
  ┌──────▼──────┐          ┌───────▼──────┐          ┌───────▼──────┐
  │ nextcloud-  │          │ nextcloud-db │          │ nextcloud-   │
  │ app         │◄────────►│ MariaDB      │          │ redis        │
  │ Apache/PHP  │          │              │◄────────►│ Cache        │
  └─────────────┘          └──────────────┘          └──────────────┘
```

## Verteilung auf die Festplatten

| Festplatte    | Grösse |                  Inhalt                      |                           Begründung                                  |
| ---           | ---    |                   ---                        |                               ---                                     |
| M.2 SSD       | 250 GB | Proxmox, VM, Docker, Datenbank, Programmcode | Die Datenbank macht viele kleine Zugriffe und braucht Geschwindigkeit |
| HDD (geplant) | 4 TB   | Daten von Nextcloud (Fotos, Videos)          | Grosse Dateien, sequenzieller Zugriff, viel Platz nötig               |
| HDD intern    | 1 TB   | Backups der VM (vzdump)                      | Backup gehört auf eine andere Festplatte als das Original             |
| HDD extern    | 1 TB   | Zweite Kopie der wichtigsten Daten           | Aktuell als Testfestplatte für die Daten im Einsatz                   |

**Aktueller Zwischenstand:** Die 4TB-Festplatte ist noch nicht vorhanden.
Die externe 1TB-Festplatte wird vorübergehend als Datenfestplatte verwendet und
geht später ins Backup.

## Ordnerstruktur auf dem Server

```
/srv/nextcloud/
├── docker-compose.yml     Definition der drei Container
├── .env                   Passwörter (nicht in Git!)
├── html/                  Programmcode von Nextcloud      → SSD
└── db/                    Dateien von MariaDB             → SSD

/mnt/clouddata/            Daten von Nextcloud             → HDD
├── .ncdata                Schutzdatei von Nextcloud
├── appdata_<instanceid>/  Interne Daten der Apps
└── <benutzername>/        Persönlicher Ordner je Benutzer
```

Die Trennung ist wichtig: Beim Wechsel auf eine grössere Festplatte muss nur
**eine Zeile** in der `docker-compose.yml` geändert werden.

## Netzwerk

### Aktuell (am Arbeitsplatz)

```
Internet ──► Laptop (Windows, Internetfreigabe)
                 │  192.168.137.x
                 │  Ethernet-Kabel
                 ▼
             Proxmox ──► Ubuntu-VM ──► Nextcloud (Port 8080)
```

### Geplant (zu Hause)

```
Internet ──► FRITZ!Box 7490 ──► Server
  (VPN)      192.168.178.x        │
                                  ▼
                            Ubuntu-VM ──► Nextcloud
```

Der Zugriff von aussen erfolgt **ausschliesslich über VPN**. Nextcloud wird nicht
direkt aus dem Internet erreichbar gemacht, es gibt keine Portweiterleitung auf
Port 8080.

## Geplante Erweiterung

Wenn später weitere Festplatten dazukommen, werden sie mit LVM zu einer Volume
Group zusammengefasst, damit Nextcloud weiterhin nur ein Datenverzeichnis sieht.
Details in `offen/loesungen.md`.
