# Server und Anschluss

## Hardware

|         Komponente      |             Wert              |                         Rolle                           |
|            ---          |              ---              |                          ---                            |
| Prozessor               | Intel Core i5-11400           |                                                         |
| Arbeitsspeicher         | 16 GB, 2400 MHz               |                                                         | 
| SSD                     | Kingston SNVS250G M.2, 250 GB | Proxmox, VM, Docker, Datenbank                          |
| Festplatte              |            4 TB               | Daten von Nextcloud, Sicherung von System und Datenbank |
| Festplatte extern (USB) |            1 TB               | Sicherungsziel (rsync), ext4, /mnt/backup-extern        |

Die ursprünglich geplante zweite SSD mit 128 GB und eine zusätzliche interne
Festplatte mit 1 TB werden nicht verwendet. Das Sicherungskonzept wurde
entsprechend angepasst, siehe `backup-konzept.md`.

## Anschluss

|    Komponente    |                Wert               |
|       ---        |     ---        ---       ---      |
| Router           | FRITZ!Box 7490                    |
| Internetanbieter | InterXS                           |
| Geschwindigkeit  | 50/50 Mbit/s                      |
| Öffentliche IPv4 | vorhanden, dynamisch (kein CGNAT) |
| IPv6             | feste /64-Adresse                 |

Der Anschluss verfuegt ueber eine oeffentliche, dynamische IPv4-Adresse.
Die frueher dokumentierte Annahme von Carrier-Grade-NAT war falsch und wurde
am 04.10.2026 durch Vergleich der im Online-Monitor angezeigten Adresse mit
einer externen Abfrage widerlegt. Portweiterleitung ist damit moeglich, eine
zusaetzliche statische IPv4-Adresse ist nicht notwendig. Die wechselnde
Adresse wird ueber MyFRITZ! als Name erreichbar gemacht.

## Software

|   Komponente  |             Version            |                    Bemerkung                    |
|      ---      |               ---              |                       ---                       |
| WireGuard     | FRITZ!OS 7.62                  | VPN auf dem Router                              | 
| Proxmox VE    | 9.2.2                          | Hypervisor                                      |
| Ubuntu Server | 26.04 (VM 100)                 | VM 100                                          |
| Docker Engine | aus dem offiziellen Repository |                                                 |
| Nextcloud     | 35.0.1                         | eigenes Image `nextcloud-ffmpeg:latest`         |
| MariaDB       | LTS                            | mit angepassten Parametern, siehe datenbank.md  |
| Redis         | Alpine (Container)             | Container, Cache und Dateisperren               |
| Caddy         | Alpine                         | Container, Reverse Proxy für HTTPS              |
| ffmpeg        | 7.1.5                          | im eigenen Image, für Vorschaubilder von Videos |

## Einstellungen der VM 100

|    Einstellung   |                          Wert                              |
|        ---       |             ---           ---          ---                 |
| CPU              | 4 Kerne, Typ `host`                                        |
| Arbeitsspeicher  | 8 GB                                                       |
| Systemfestplatte | 64 GB (LVM, auf 61 GB nutzbar erweitert)                   |
| Netzwerk         | virtio, Bridge `vmbr0`                                     |
| QEMU Guest Agent | aktiviert                                                  |
| Datenfestplatte  | virtuelle Festplatte 3500 GiB, qcow2, backup=0, discard=on |
| Systemfestplatte | 64 GB (LVM, auf 61 GB erweitert), discard=on, ssd=1        |
| Autostart        | onboot=1, startup order=1                                  |
| USB              | externe 1TB-Festplatte als Sicherungsziel                  |

## Speicher der Datenfestplatte

Die Datenfestplatte ist nicht direkt formatiert, sondern über LVM eingebunden:

```
/dev/sda1 (4TB)              ext4, -T largefile, /mnt/hdd4tb   ← Host
    └── vm-100-disk-2.qcow2  3500 GiB, duenn belegt
            └── /dev/sdb     Physical Volume                   ← VM
                └── clouddata-vg          Volume Group, 3.42 TiB
                        └── clouddata-lv  ext4, -i 131072
```

Eingebunden unter `/mnt/clouddata` über die UUID in `/etc/fstab`, mit der Option
`nofail`. Der fuer root reservierte Speicher wurde beim Formatieren mit `-m 1` von 5 %
auf 1 % gesenkt. Verfuegbar sind dadurch rund 3.4 TB.

Grund fuer LVM: Weitere Festplatten lassen sich im laufenden Betrieb in
dieselbe Volume Group aufnehmen, ohne dass Nextcloud etwas davon merkt oder
Daten kopiert werden muessen.

Die beiden Dateisysteme sind absichtlich unterschiedlich formatiert. Auf dem
Host liegen nur wenige, sehr grosse Dateien (ein Festplattenabbild, die
Archive von vzdump), deshalb `-T largefile`: 3 815 552 Inodes statt rund
244 Millionen, Inode-Tabelle etwa 0.9 GiB statt 58 GiB. In der VM liegen
viele Dateien, weil Nextcloud zu jedem Foto mehrere Vorschaubilder ablegt,
deshalb `-i 131072`: 28 672 000 Inodes. Die Anzahl der Inodes wird beim
Formatieren festgelegt und laesst sich spaeter nicht erhoehen. Kontrolle mit
`df -i`.

## Netzwerk

|           |                 zu Hause                   |
|    ---    |                   ---                      |
| Gateway   | 192.168.178.1 (FRITZ!Box)                  |
| Proxmox   | 192.168.178.100 (Reservierung)             |
| Ubuntu-VM | 192.168.178.220 (Reservierung)             |
| DNS       | FRITZ!Box, per DHCP zugewiesen             |
| Zugriff   | `https://privatecloud.fritz.box` ueber VPN |

Der Port 8080 wird nicht mehr veröffentlicht. Nextcloud ist nur noch über den
Reverse Proxy erreichbar.

## Snapshots und Backups

|         Name       |                     Zweck                       |   Stand  |
|         ---        |        ---           ---           ---          |    ---   |
| `vor-docker`       | Zustand vor der Docker-Installation             | gelöscht |
| `nextcloud-laeuft` | Nextcloud läuft, vor der Verschiebung der Daten | gelöscht |
| `vor-datenumzug`   | Vor der Verschiebung auf die externe Festplatte | gelöscht |
| `vor-https`        | vor der Einrichtung des Reverse Proxy           | gelöscht |
| `vor-lvm`          | vor dem Umbau der Datenfestplatte auf LVM       | gelöscht |
| `vor-4tb`          | vor dem Umbau der Datenfestplatte               | gelöscht |

**Wichtig:** Ein Snapshot ist kein Backup. Er liegt auf derselben Festplatte wie
die VM und hilft nicht, wenn die SSD ausfällt. Snapshots mit gesichertem
Arbeitsspeicher belegen zusätzlich so viel Platz, wie der VM Arbeitsspeicher
zugewiesen ist.

## Sicherungen

|        Was         |         Werkzeug              |        Zeitpunkt        |                 Ziel                   |
|        ---         |           ---                 |           ---           |                 ---                    |
| Virtuelle Maschine | vzdump, Modus Snapshot, ZSTD  | Mo–Fr 00:00             | hdd4tb (4TB)                           |
| Datenbank          | `mysqldump` mit Wartungsmodus | täglich 01:30           | `/srv/backup/db` (in vzdump enthalten) |
| Daten              | rsync                         |  Samstag 02:30          | `/mnt/backup-extern` (1TB)             | 


> **Einschraenkung:** Die geplanten Zeiten gelten nur, wenn der Server laeuft.
> Da er nicht rund um die Uhr in Betrieb ist, wurden die Aufgaben bis zum
> 05.10.2026 nie automatisch ausgefuehrt. Geplante Loesung: Der Server weckt
> sich nachts selbst (rtcwake), fuehrt die Sicherungen der Reihe nach aus und
> schaltet sich wieder ab.

## Getestete Wiederherstellungen

|                  Test                  |   Datum    |                    Ergebnis                         |
|                  ---                   |    ---     |                      ---                            |
| VM aus vzdump als neue VM 999          | 29.09.2026 | bestanden                                           |
| Datenbank aus mysqldump                | 29.09.2026 | bestanden                                           |
| Ausfall der Datenfestplatte im Betrieb | 29.09.2026 | Nextcloud meldet den Fehler, keine Daten beschädigt |
| Automatischer Start nach Neustart      | 29.09.2026 | bestanden                                           |
| VM aus vzdump auf hdd4tb als VM 999    | 02.10.2026 | bestanden                                           |

## Gemessene Groesse der VM-Sicherung

|                Zeitpunkt               |  Groesse  |
|                   ---                  |    ---    |
| 29.09.2026, ohne discard               | 4.39 GiB  |
| 02.10.2026, vor fstrim                 | 15.40 GiB |
| 02.10.2026, mit discard=on und fstrim  | 4.05 GiB  |