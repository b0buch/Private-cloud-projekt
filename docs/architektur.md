# Architektur

Stand: 05.10.2026

## Übersicht

```
                    ┌─────────────────────────────┐
                    │  Internet                   │
                    └──────────────┬──────────────┘
                                   │ nur über VPN
                    ┌──────────────▼──────────────┐
                    │  FRITZ!Box 7490             │
                    │  192.168.178.1              │
                    │  WireGuard-Server, UDP 51820│
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  Server (Hardware)          │
                    │  i5-11400, 16GB RAM         │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  Proxmox VE 9.2.2           │
                    │  (Virtualisierung)          │
                    │  192.168.178.100            │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  VM 100: Ubuntu Server      │
                    │  4 CPU-Kerne, 8 GB RAM      │
                    │  192.168.178.220            │
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

Aus dem Internet ist nichts davon erreichbar. Es gibt keine Portweiterleitung
auf Port 443. Wer von aussen zugreifen will, baut zuerst einen VPN-Tunnel auf.
Details in `vpn.md`.

Jede Schicht hat einen eigenen Grund:

|   Schicht   |                         Warum sie da ist                                          |
|     ---     |                               ---                                                 |
| Proxmox     | Snapshots vor jeder Änderung, Sicherung der ganzen VM mit einem Befehl            |
| Ubuntu-VM   | Trennung: Ein Fehler in der Cloud betrifft den Hypervisor nicht                   |
| Docker      | Jeder Dienst in eigener Umgebung, austauschbar, ohne Abhängigkeit zum Wirtssystem |

## Die vier Container

|     Container     |                  Image                    |                                        Aufgabe                                                |
|       ---         |                  ---                      |                                          ---                                                  |
| `nextcloud-caddy` | `caddy:alpine`                            | Nimmt HTTPS-Anfragen entgegen, leitet sie intern per HTTP weiter, setzt die Sicherheitsheader |
| `nextcloud-app`   | `nextcloud-ffmpeg:latest` (eigenes Image) | Nextcloud mit Apache und PHP                                                                  |
| `nextcloud-db`    | `mariadb:lts`                             | Datenbank                                                                                     |
| `nextcloud-redis` | `redis:alpine`                            | Cache, Dateisperren, Brute-Force-Zähler                                                       |

Die Container finden sich im internen Docker-Netzwerk über den **Servicenamen**,
nicht über IP-Adressen. Deshalb steht in der Konfiguration `MYSQL_HOST=db` und
im Caddyfile `reverse_proxy app:80`. Möglich ist das, weil Docker einen eigenen
DNS-Dienst unter `127.0.0.11` betreibt.

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

Die Zeile mit `ForceIPv4` war nötig, weil der Paketserver von Debian über die
damalige Verbindung nur IPv6-Adressen zurückgab, während keine IPv6-Route
vorhanden war. `apt` blieb dann ohne Fehlermeldung stehen.

### Angepasste Parameter der Datenbank

Der Dienst `db` wird nicht mit den Voreinstellungen gestartet:

```yaml
command: >
  --transaction-isolation=READ-COMMITTED
  --log-bin=binlog
  --binlog-format=ROW
  --innodb-log-file-size=256M
  --slow-query-log=ON
  --slow-query-log-file=/var/lib/mysql/slow.log
  --long-query-time=2
```

Die ersten drei Parameter verlangt Nextcloud. Die übrigen drei stammen aus den
Empfehlungen der Selbstprüfung vom 05.10.2026. Begründung je Parameter in
`datenbank.md`.

## Verteilung auf die Festplatten

| Festplatte     | Grösse |                     Inhalt                      |                           Begründung                                  |
| ---            | ---    |                      ---                        |                               ---                                     |
| M.2 SSD        | 250 GB | Proxmox, VM, Docker, Datenbank, Programmcode    | Die Datenbank macht viele kleine Zugriffe und braucht Geschwindigkeit |
| HDD intern     | 4 TB   | Daten von Nextcloud, Archive von vzdump         | Grosse Dateien, sequenzieller Zugriff, viel Platz nötig               |
| HDD extern USB | 1 TB   | Sicherungsziel für die Daten (rsync)            | Kann abgezogen und an einem anderen Ort aufbewahrt werden             |

### LVM und die Kette der Speicherschichten

Die Datenfestplatte ist nicht direkt formatiert. Vom Blech bis zur Datei liegen
fünf Schichten übereinander:

```
/dev/sda                     4TB-Festplatte, GPT
  └── /dev/sda1              ext4, -m 1 -T largefile, /mnt/hdd4tb   ← Host
        ├── images/100/vm-100-disk-2.qcow2   3500 GiB, dünn belegt
        │     └── /dev/sdb   Physical Volume (PV)                   ← VM
        │           └── clouddata-vg          Volume Group (VG)
        │                 └── clouddata-lv    Logical Volume (LV), ext4
        │                       → /mnt/clouddata
        └── dump/            Archive von vzdump
```

Jede Schicht hat einen Zweck.

**ext4 auf dem Host, statt die Festplatte der VM direkt durchzureichen.** Eine
durchgereichte Festplatte würde die Snapshot-Funktion von Proxmox für die ganze
VM unmöglich machen. Als Datei im Dateisystem bleiben Snapshots erhalten, und
Daten und Sicherungen teilen dieselbe Partition.

**qcow2 statt raw.** Nur qcow2 unterstützt Snapshots. Die Datei ist dünn belegt:
Sie meldet 3500 GiB und belegte direkt nach dem Erstellen 548 MiB. Angeforderte
Grösse und belegter Platz sind zwei verschiedene Dinge. Damit gelöschte Dateien
den Platz wieder freigeben, ist `discard=on` gesetzt.

**LVM in der VM.** Weitere Festplatten lassen sich im laufenden Betrieb in
dieselbe Volume Group aufnehmen, ohne dass Nextcloud etwas davon merkt:

```bash
pvcreate /dev/sdX
vgextend clouddata-vg /dev/sdX
lvextend -l +100%FREE /dev/clouddata-vg/clouddata-lv
resize2fs /dev/clouddata-vg/clouddata-lv
```

Ein zweites Datenverzeichnis wäre keine Alternative, weil der Pfad bei der
Installation fest in der Konfiguration steht.

**Nachteil von LVM ohne RAID:** Fällt eine Festplatte aus, ist das ganze
Dateisystem betroffen, nicht nur der Teil auf dieser Festplatte. Ein Backup ist
deshalb nicht optional.

### Zwei Dateisysteme, absichtlich entgegengesetzt formatiert

Die Anzahl der Inodes wird beim Formatieren festgelegt und lässt sich später
nicht erhöhen. Sind sie aufgebraucht, meldet das System "kein Platz", während
`df -h` freien Speicher zeigt. Kontrolle mit `df -i`.

|      Dateisystem       |    Parameter   |    Inodes   | Inode-Tabelle |              Begründung                |
|          ---           |      ---       |     ---     |      ---      |                 ---                    |
| `/mnt/hdd4tb` (Host)   | `-T largefile` |   3 815 552 |   ca. 0.9 GiB | Drei Objekte: ein Abbild, zwei Archive |
| `/mnt/clouddata` (VM)  | `-i 131072`    |  28 672 000 |   ca. 7 GiB   | Viele Dateien samt Vorschaubildern     |
| Voreinstellung wäre    | `-i 16384`     | 229 376 000 |  ca. 55 GiB   | Auf Dateien von 16 KiB ausgelegt       |

Beide Male zusätzlich `-m 1`: Der für root reservierte Anteil sinkt von 5 % auf
1 %. Bei 3.64 TiB sind das 37 GiB statt 186 GiB.

Gemessen beim Umzug am 02.10.2026: 1315 Objekte für 7.8 GB Daten, also rund
6 MB je Datei. Hochgerechnet auf 3.4 TB ergibt das etwa 570 000 Dateien
zuzüglich Vorschaubilder — bei 28.7 Millionen Inodes ein Spielraum von etwa
Faktor 50.

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

/mnt/clouddata/             Daten von Nextcloud             → 4TB (qcow2, LVM)
├── .ncdata                 Schutzdatei von Nextcloud
├── appdata_<instanceid>/   Interne Daten der Apps, Vorschaubilder
├── __groupfolders/         Inhalt der gemeinsamen Ordner
└── <benutzername>/         Persönlicher Ordner je Benutzer

/mnt/backup-extern/         Sicherungsziel                  → 1TB extern
└── clouddata/              Spiegel der Daten (rsync)

/srv/backup/
├── db/                     Sicherungen der Datenbank und von config.php
├── nextcloud-db-backup.sh
└── nextcloud-data-backup.sh

/mnt/hdd4tb/                                                → 4TB (Host)
├── images/100/             virtuelle Festplatte der VM
└── dump/                   Archive von vzdump
```

Die Trennung ist wichtig: Beim Wechsel auf eine grössere Festplatte muss nur der
Mountpunkt geändert werden, Programmcode und Datenbank bleiben unberührt. Beim
Umzug auf die 4TB-Festplatte am 02.10.2026 hat das die Unterbrechung auf etwa
zehn Minuten begrenzt.

### Einbinden über UUID

In `/etc/fstab` stehen UUIDs und keine Gerätenamen, jeweils mit `nofail`:

```
UUID=46cc1783-... /mnt/clouddata     ext4 defaults,nofail 0 2
UUID=3c3621b6-... /mnt/backup-extern ext4 defaults,nofail,x-systemd.device-timeout=10 0 2
```

Der Grund zeigte sich am 04.10.2026: Nach einem Neustart hatten `sdb` und `sdc`
ohne Zutun die Plätze getauscht. Mit Gerätenamen wäre die Sicherungsfestplatte
als Datenverzeichnis eingebunden worden.

`x-systemd.device-timeout=10` bei der externen Festplatte: Sie ist laut Konzept
meist abgezogen, und ohne diese Option wartet systemd bei jedem Start 90
Sekunden auf ein Gerät, das nicht da ist.

### Docker muss auf das Dateisystem warten

Am 04.10.2026 startete Docker beim Hochfahren, bevor `/mnt/clouddata`
eingebunden war. Docker band das leere Verzeichnis als Volume ein, und Nextcloud
meldete, die Datei `.ncdata` fehle. Die Daten waren unversehrt — der Container
hing am leeren Verzeichnis, nicht am Pfad.

Gelöst mit einer Ergänzung für den Dienst:

```
# /etc/systemd/system/docker.service.d/override.conf
[Unit]
RequiresMountsFor=/mnt/clouddata
```

`/mnt/backup-extern` steht dort absichtlich **nicht**: Diese Festplatte ist
zeitweise abgezogen, und Docker würde dann überhaupt nicht starten.

## Netzwerk

### Zu Hause (seit 02.10.2026)

```
Internet ──► FRITZ!Box 7490 ──► Server
  (VPN)      192.168.178.1        │
                                  ▼
                            Proxmox 192.168.178.100
                                 └── Ubuntu-VM 192.168.178.220
                                         ├── Docker-Netz 172.18.0.0/16
                                         └── Nextcloud über Caddy, Port 443
```

|  Komponente  |                Wert                  |
|     ---      |                 ---                  |
| Gateway      | 192.168.178.1 (FRITZ!Box)            |
| Proxmox      | 192.168.178.100, Adresse reserviert  |
| Ubuntu-VM    | 192.168.178.220, Adresse reserviert  |
| Docker-Netz  | 172.18.0.0/16                        |
| DNS          | FRITZ!Box, über DHCP zugewiesen      |
| Zugriff      | `https://privatecloud.fritz.box`     |

Beide Adressen kommen über DHCP mit einer Reservierung in der FRITZ!Box, nicht
als feste Konfiguration im Betriebssystem. Vorteil: Ein Wechsel des Netzes
erfordert keine Änderung am Server.

Der DNS-Server muss die FRITZ!Box sein und kein öffentlicher. Nur sie kennt die
Namen im Heimnetz unter `.fritz.box`. Mit `1.1.1.1` konnte der Webserver sich
nicht selbst erreichen, und sechs Selbstprüfungen von Nextcloud schlugen fehl —
behoben am 05.10.2026 durch Entfernen von `dhcp4-overrides: use-dns: false` aus
der netplan-Konfiguration.

Der Zugriff von aussen erfolgt **ausschliesslich über VPN**. Es gibt keine
Portweiterleitung auf Nextcloud. Details in `vpn.md`.

### Früher: Zwischenlösung am Arbeitsplatz (18.09. bis 02.10.2026)

```
Internet ──► Laptop (Windows, Internetfreigabe)
                 │  192.168.137.1
                 │  Ethernet-Kabel
                 ▼
             Proxmox 192.168.137.2
                 └── Ubuntu-VM 192.168.137.220
```

Diese Lösung war nötig, weil der Server bis zum Umzug am Arbeitsplatz stand.
Ihre Einschränkungen sind der Grund für mehrere Probleme im Arbeitsjournal:

- kein IPv6
- das Gateway stellt keinen DNS-Dienst bereit, deshalb waren öffentliche
  DNS-Server eingetragen — und genau das liess die Selbstprüfungen scheitern
- die Freigabe wurde durch Windows-Updates zurückgesetzt
- gegen Ende antwortete der DHCP-Dienst der Freigabe nicht mehr
- kein Zugriff von anderen Geräten im Firmennetz

Der Abschnitt bleibt als Begründung für die damaligen Entscheidungen erhalten.

## HTTPS

|                | Wert                                             |
| ---            | ---                                              |
| Reverse Proxy  | Caddy                                            |
| Zertifikat     | selbst signiert, 825 Tage                        |
| Schlüssellänge | RSA 2048                                         |
| Gültig für     | `DNS:privatecloud.fritz.box` im `subjectAltName` |
| Weiterleitung  | http wird auf https umgeleitet                   |

Das Caddyfile definiert den Dienst über den **Port**, nicht über einen Namen:

```
:443 {
    tls /certs/server.crt /certs/server.key
    reverse_proxy app:80
}

:80 {
    redir https://{host}{uri} permanent
}
```

Grund: Beim Aufruf über eine IP-Adresse sendet der Browser kein SNI (Server Name
Indication), weil der Standard dort nur Domainnamen erlaubt. Ein Block mit einem
Namen könnte deshalb nie zugeordnet werden. Der Zugriff über eine Adresse muss
weiterhin möglich bleiben, etwa zur Fehlersuche oder für das geplante
Server-VPN, deshalb bleibt die Schreibweise mit dem Port.

In der Weiterleitung steht `{host}` und keine feste Adresse. `{host}` ist der
Name, den der Client selbst angefragt hat. Vorher stand dort die Adresse des
Arbeitsplatzes, und nach dem Umzug schickte jede Anfrage ohne `https://` den
Browser auf eine Adresse, die es nicht mehr gab.

**Das Zertifikat ist auf einen Namen ausgestellt, nicht auf eine Adresse.** Das
ist die wichtigste Entscheidung in diesem Abschnitt: Ein Wechsel des Netzes oder
eine neue Adresse aus dem DHCP machen es nicht ungültig. Beim ersten Umzug war
noch ein Neuausstellen nötig, künftig nicht mehr.

## Einstellungen für den Betrieb hinter einem Proxy

|    Einstellung      |                 Wert                  |                            Zweck                               |
|        ---          |                 ---                   |                             ---                                |
| `trusted_proxies`   | `172.18.0.0/16`                       | Nur dem Docker-Netz wird der Header `X-Forwarded-For` geglaubt |
| `overwriteprotocol` | `https`                               | Erzeugte Links verwenden https                                 |
| `overwrite.cli.url` | `https://privatecloud.fritz.box`      | Basisadresse für Hintergrundaufgaben und Selbstprüfungen       |
| `trusted_domains`   | Name und Adressbereiche mit `*`       | Erlaubte Adressen für den Zugriff                              |

Ohne `trusted_proxies` würde Nextcloud alle Anfragen dem Proxy zuschreiben. Der
Brute-Force-Schutz würde dann alle Benutzer gleichzeitig verzögern.

In `trusted_domains` sind Platzhalter mit `*` eingetragen. Das ist nötig, weil
die Adressen über DHCP kommen: Ohne Platzhalter erscheint bei einer neuen
Adresse die Meldung "Access through untrusted domain".

## Geplante Aufgaben

Alle an einem Ort, in `/etc/cron.d/nextcloud`, ausgeführt als root:

```
*/5 *  * * *  root  docker exec -u www-data nextcloud-app php /var/www/html/cron.php
30  1  * * *  root  /srv/backup/nextcloud-db-backup.sh
30  2  * * 6  root  /srv/backup/nextcloud-data-backup.sh
```

Vorher lagen sie in den crontabs zweier verschiedener Benutzer, und für die
Sicherung der Datenbank gab es überhaupt keinen Eintrag. Dateien in
`/etc/cron.d/` haben ein zusätzliches Feld mit dem Benutzernamen und müssen mit
einem Zeilenumbruch enden — bei einem Fehler wird die Datei ohne Warnung
ignoriert.

Einschränkung: Diese Zeiten gelten nur, wenn der Server zu diesem Zeitpunkt
läuft. Siehe `backup-konzept.md`.

## Was absichtlich anders gebaut wurde

|      Naheliegend        |            Gewählt           |                        Grund                                  |
|          ---            |             ---              |                         ---                                   |
| Festplatte durchreichen | qcow2 im Dateisystem         | Sonst keine Snapshots der VM mehr möglich                     |
| RAID für die Daten      | LVM ohne RAID                | RAID braucht gleich grosse Festplatten; Backup ist wichtiger  |
| Let's Encrypt           | selbst signiertes Zertifikat | Let's Encrypt verlangt Erreichbarkeit aus dem Internet        |
| Zertifikat auf die IP   | Zertifikat auf den Namen     | Überlebt einen Wechsel der Adresse                            |
| `pvmove` beim Umzug     | rsync                        | `pvmove` verschiebt zugewiesene Extents, nicht belegte Daten  |
| Nextcloud direkt        | Reverse Proxy davor          | Verschlüsselung an einer Stelle, Header für alle Dienste      |
| feste IP im System      | DHCP mit Reservierung        | Ein Netzwechsel erfordert keine Änderung am Server            |
| cron                    | cron, systemd-Timer geplant  | cron holt verpasste Ausführungen nicht nach                   |