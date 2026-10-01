# Architektur

Stand: 01.10.2026

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
                         ┌─────────▼─────────┐
                         │   nextcloud-caddy │  ← Port 443 / 80
                         │   Reverse Proxy   │
                         └─────────┬─────────┘
                                   │ HTTP intern
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
  ┌──────▼──────┐          ┌───────▼──────┐          ┌───────▼──────┐
  │ nextcloud-  │          │ nextcloud-db │          │ nextcloud-   │
  │ app         │◄────────►│ MariaDB      │          │ redis        │
  │ Apache/PHP  │          │              │◄────────►│ Cache        │
  └─────────────┘          └──────────────┘          └──────────────┘
```

Nur der Reverse Proxy ist von aussen erreichbar. Die Portfreigabe von Nextcloud
wurde entfernt, der Zugriff erfolgt ausschliesslich über Caddy.

## Die vier Container

|     Container     |                  Image                    |                                        Aufgabe                                                |
|       ---         |                  ---                      |                                          ---                                                  |
| `nextcloud-caddy` | `caddy:alpine`                            | Nimmt HTTPS-Anfragen entgegen, leitet sie intern per HTTP weiter, setzt die Sicherheitsheader |
| `nextcloud-app`   | `nextcloud-ffmpeg:latest` (eigenes Image) | Nextcloud mit Apache und PHP                                                                  |
| `nextcloud-db`    | `mariadb:lts`                             | Datenbank                                                                                     |
| `nextcloud-redis` | `redis:alpine`                            | Cache, Dateisperren, Brute-Force-Zähler                                                       |

Die Container finden sich im internen Docker-Netzwerk über den **Servicenamen**,
nicht über IP-Adressen. Deshalb steht in der Konfiguration `MYSQL_HOST=db` und
im Caddyfile `reverse_proxy app:80`.

### Eigenes Image

Für Nextcloud wird kein fertiges Image verwendet, sondern ein eigenes auf Basis
des offiziellen:

```dockerfile
FROM nextcloud:apache
RUN echo 'Acquire::ForceIPv4 "true";' > /etc/apt/apt.conf.d/99force-ipv4
RUN apt-get update \
 && apt-get install -y --no-install-recommends ffmpeg \
 && rm -rf /var/lib/apt/lists/*
```

Grund: ffmpeg wird für die Vorschaubilder von Videos gebraucht. Eine
Installation im laufenden Container geht verloren, sobald der Container neu
erstellt wird.

## Verteilung auf die Festplatten

| Festplatte    | Grösse |                  Inhalt                                       |                           Begründung                                  |
| ---           | ---    |                   ---                                         |                               ---                                     |
| M.2 SSD       | 250 GB | Proxmox, VM, Docker, Datenbank, Programmcode                  | Die Datenbank macht viele kleine Zugriffe und braucht Geschwindigkeit |
| HDD (geplant) | 4 TB   | Daten von Nextcloud, Sicherung von System und Datenbank       | Grosse Dateien, sequenzieller Zugriff, viel Platz nötig               |
| HDD extern    | 1 TB   | aktuell Datenfestplatte, später Sicherung der wichtigen Daten | Zwischenlösung, bis die 4TB-Festplatte eintrifft                      |

### LVM

Die Datenfestplatte ist nicht direkt formatiert, sondern über LVM eingebunden:

```
/dev/sdb                      Physical Volume (PV)
    └── clouddata-vg          Volume Group (VG)
            └── clouddata-lv  Logical Volume (LV), ext4
```

Grund: Weitere Festplatten lassen sich später im laufenden Betrieb hinzufügen,
ohne dass Nextcloud etwas davon merkt:

```bash
pvcreate /dev/sdX
vgextend clouddata-vg /dev/sdX
lvextend -l +100%FREE /dev/clouddata-vg/clouddata-lv
resize2fs /dev/clouddata-vg/clouddata-lv
```

Ohne LVM müssten alle Daten auf die neue Festplatte kopiert werden.

## Ordnerstruktur auf dem Server

```
/srv/nextcloud/
├── docker-compose.yml      Definition der vier Container
├── .env                    Passwörter (nicht in Git!)
├── nextcloud-image/
│   └── Dockerfile          eigenes Image mit ffmpeg
├── caddy/
│   ├── Caddyfile           Konfiguration des Reverse Proxy
│   ├── certs/              Zertifikat und privater Schlüssel
│   ├── data/               Daten von Caddy
│   └── config/             Konfiguration von Caddy
├── html/                   Programmcode von Nextcloud      → SSD
└── db/                     Dateien von MariaDB             → SSD

/mnt/clouddata/             Daten von Nextcloud             → HDD (LVM)
├── .ncdata                 Schutzdatei von Nextcloud
├── appdata_<instanceid>/   Interne Daten der Apps
└── <benutzername>/         Persönlicher Ordner je Benutzer

/srv/backup/db/             Sicherungen der Datenbank
```

Die Trennung ist wichtig: Beim Wechsel auf eine grössere Festplatte muss nur der
Mountpunkt geändert werden, Programmcode und Datenbank bleiben unberührt.

## Netzwerk

### Aktuell (am Arbeitsplatz)

```
Internet ──► Laptop (Windows, Internetfreigabe)
                 │  192.168.137.1
                 │  Ethernet-Kabel
                 ▼
             Proxmox 192.168.137.2
                 └── Ubuntu-VM 192.168.137.220
                         └── Nextcloud über Caddy, Port 443
```

Einschränkungen dieser Zwischenlösung:

- Kein IPv6
- Das Gateway stellt keinen DNS-Dienst bereit, deshalb sind öffentliche
  DNS-Server eingetragen
- Die Konfiguration geht bei Windows-Updates verloren
- Kein Zugriff von anderen Geräten im Firmennetz

### Geplant (zu Hause)

```
Internet ──► FRITZ!Box 7490 ──► Server
  (VPN)      192.168.178.1        │
                                  ▼
                            Ubuntu-VM ──► Caddy ──► Nextcloud
```

Der Zugriff von aussen erfolgt **ausschliesslich über VPN**. Es gibt keine
Portweiterleitung auf Nextcloud.

## HTTPS

|                | Wert                                |
| ---            | ---                                 |
| Reverse Proxy  | Caddy                               |
| Zertifikat     | selbst signiert, 825 Tage           |
| Schlüssellänge | RSA 2048                            |
| Gültig für     | IP-Adresse im Feld `subjectAltName` |
| Weiterleitung  | http wird auf https umgeleitet      |

Das Caddyfile definiert den Dienst über den **Port**, nicht über einen Namen:

```
:443 {
    tls /certs/server.crt /certs/server.key
    reverse_proxy app:80
}
```

Grund: Beim Aufruf über eine IP-Adresse sendet der Browser kein SNI (Server Name
Indication), weil der Standard dort nur Domainnamen erlaubt. Ein Block mit einem
Namen könnte deshalb nie zugeordnet werden.

Zu Hause mit einem lokalen Namen kann wieder die normale Schreibweise verwendet
werden, und es können mehrere Dienste über denselben Proxy laufen.

## Einstellungen für den Betrieb hinter einem Proxy

|    Einstellung      |        Wert         |                            Zweck                               |
|        ---          |        ---          |                             ---                                |
| `trusted_proxies`   | `172.18.0.0/16`     | Nur dem Docker-Netz wird der Header `X-Forwarded-For` geglaubt |
| `overwriteprotocol` | `https`             | Erzeugte Links verwenden https                                 |
| `overwrite.cli.url` | `https://<adresse>` | Basisadresse für Hintergrundaufgaben                           |
| `trusted_domains`   | `<adresse>`         | Erlaubte Adressen für den Zugriff                              |

Ohne `trusted_proxies` würde Nextcloud alle Anfragen dem Proxy zuschreiben. Der
Brute-Force-Schutz würde dann alle Benutzer gleichzeitig verzögern.