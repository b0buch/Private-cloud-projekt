# Server und Anschluss

## Hardware

|         Komponente      |             Wert              |                           Rolle                               |
|            ---          |              ---              |                            ---                                |
| Prozessor               | Intel Core i5-11400           |                                                               |
| Arbeitsspeicher         | 16 GB, 2400 MHz               |                                                               | 
| SSD                     | Kingston SNVS250G M.2, 250 GB | Proxmox, VM, Docker, Datenbank                                |
| Festplatte (geplant)    |            4 TB               | Daten von Nextcloud, Sicherung von System und Datenbank       |
| Festplatte extern (USB) |            1 TB               | zurzeit Datenfestplatte, später Sicherung der wichtigen Daten |

Die ursprünglich geplante zweite SSD mit 128 GB und eine zusätzliche interne
Festplatte mit 1 TB werden nicht verwendet. Das Sicherungskonzept wurde
entsprechend angepasst, siehe `backup-konzept.md`.

## Anschluss

|    Komponente    |                Wert              |
|       ---        |     ---        ---       ---     |
| Router           | FRITZ!Box 7490                   |
| Internetanbieter | InterXS                          |
| Geschwindigkeit  | 50/50 Mbit/s                     |
| Öffentliche IPv4 | keine eigene (Carrier-Grade-NAT) |
| IPv6             | feste /64-Adresse                |

Wegen Carrier-Grade-NAT ist eine Portweiterleitung über IPv4 nicht möglich.
Für das Projekt wird eine zusätzliche öffentliche IPv4-Adresse dazugebucht.
Details in `offen/probleme.md`.

## Software

|   Komponente  |             Version            |                    Bemerkung                    |
|      ---      |               ---              |                       ---                       |
| Proxmox VE    | 9.2.2                          | Hypervisor                                      |
| Ubuntu Server | 26.04 (VM 100)                 | VM 100                                          |
| Docker Engine | aus dem offiziellen Repository |                                                 |
| Nextcloud     | 35.0.1                         | eigenes Image `nextcloud-ffmpeg:latest`         |
| MariaDB       | LTS (Container)                | Container                                       |
| Redis         | Alpine (Container)             | Container, Cache und Dateisperren               |
| Caddy         | Alpine                         | Container, Reverse Proxy für HTTPS              |
| ffmpeg        | 7.1.5                          | im eigenen Image, für Vorschaubilder von Videos |

## Einstellungen der VM 100

|    Einstellung   |                    Wert                      |
|        ---       |       ---           ---          ---         |
| CPU              | 4 Kerne, Typ `host`                          |
| Arbeitsspeicher  | 8 GB                                         |
| Systemfestplatte | 64 GB (LVM, auf 61 GB nutzbar erweitert)     |
| Netzwerk         | virtio, Bridge `vmbr0`                       |
| QEMU Guest Agent | aktiviert                                    |
| Datenfestplatte  | externe 1TB-Festplatte über LVM, siehe unten |

## Speicher der Datenfestplatte

Die Datenfestplatte ist nicht direkt formatiert, sondern über LVM eingebunden:

```
/dev/sdb                      Physical Volume
    └── clouddata-vg          Volume Group, 931.51 GiB
            └── clouddata-lv  Logical Volume, ext4, Label nextcloud-data
```

Eingebunden unter `/mnt/clouddata` über die UUID in `/etc/fstab`, mit der Option
`nofail`. Der für root reservierte Speicher wurde mit `tune2fs -m 1` von 5 % auf
1 % gesenkt, verfügbar sind dadurch rund 907 GB.

Grund für LVM: Die 4TB-Festplatte kann später im laufenden Betrieb in dieselbe
Volume Group aufgenommen werden, ohne dass Daten kopiert werden müssen.

## Netzwerk

|           |        Aktuell (Arbeitsplatz)            |      Geplant (zu Hause)       |
|    ---    |                 ---                      |            ---                |
| Gateway   | 192.168.137.1 (Laptop, Internetfreigabe) | 192.168.178.1 (FRITZ!Box)     |
| Proxmox   | 192.168.137.2                            | noch festzulegen              |
| Ubuntu-VM | 192.168.137.220 (fest in netplan)        | Reservierung in der FRITZ!Box |
| DNS       | 1.1.1.1 und 8.8.8.8                      | FRITZ!Box                     |
| Zugriff   | `https://<adresse>` über Caddy, Port 443 | zusätzlich über VPN           |

Der Port 8080 wird nicht mehr veröffentlicht. Nextcloud ist nur noch über den
Reverse Proxy erreichbar.

## Snapshots und Backups

|         Name       |                     Zweck                       |   Stand   |
|         ---        |        ---           ---           ---          |    ---    |
| `vor-docker`       | Zustand vor der Docker-Installation             | gelöscht  |
| `nextcloud-laeuft` | Nextcloud läuft, vor der Verschiebung der Daten | gelöscht  |
| `vor-datenumzug`   | Vor der Verschiebung auf die externe Festplatte | gelöscht  |
| `vor-https`        | vor der Einrichtung des Reverse Proxy           | gelöscht  |
| `vor-lvm`          | vor dem Umbau der Datenfestplatte auf LVM       | vorhanden |

**Wichtig:** Ein Snapshot ist kein Backup. Er liegt auf derselben Festplatte wie
die VM und hilft nicht, wenn die SSD ausfällt. Snapshots mit gesichertem
Arbeitsspeicher belegen zusätzlich so viel Platz, wie der VM Arbeitsspeicher
zugewiesen ist.

## Sicherungen

|        Was         |         Werkzeug              |        Zeitpunkt        |                 Ziel                   |
|        ---         |           ---                 |           ---           |                 ---                    |
| Virtuelle Maschine | vzdump, Modus Snapshot, ZSTD  | Mo–Fr 00:00             | `local` (SSD), Zwischenlösung          |
| Datenbank          | `mysqldump` mit Wartungsmodus | täglich 01:30           | `/srv/backup/db` (SSD), Zwischenlösung |
| Daten              | rsync                         | noch nicht eingerichtet | externe Festplatte, geplant            | 

Grösse des ersten VM-Backups: 4.39 GiB.

> **Zwischenlösung:** Die Sicherungen liegen zurzeit auf derselben SSD wie das
> Original und erfüllen damit die Grundregel noch nicht. Sobald die
> 4TB-Festplatte eingebaut ist, werden sie dorthin verlegt. Siehe
> `backup-konzept.md`.

## Getestete Wiederherstellungen

| Test                                   | Datum      | Ergebnis                                            |
| ---                                    | ---        | ---                                                 |
| VM aus vzdump als neue VM 999          | 29.09.2026 | bestanden                                           |
| Datenbank aus mysqldump                | 29.09.2026 | bestanden                                           |
| Ausfall der Datenfestplatte im Betrieb | 29.09.2026 | Nextcloud meldet den Fehler, keine Daten beschädigt |
| Automatischer Start nach Neustart      | 29.09.2026 | bestanden                                           |
