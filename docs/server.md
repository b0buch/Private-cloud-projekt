# Server und Anschluss

## Hardware

|         Komponente      |             Wert              |
|            ---          |              ---              |
| Prozessor               | Intel Core i5-11400           |
| Arbeitsspeicher         | 16 GB, 2400 MHz               |
| SSD                     | Kingston SNVS250G M.2, 250 GB |
| Festplatte (geplant)    |            4 TB               |
| Festplatte intern       |            1 TB               |
| Festplatte extern (USB) |            1 TB               |

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

|   Komponente  |       Version      |
|      ---      |         ---        |
| Proxmox VE    | 9.2.2              |
| Ubuntu Server | 26.04 (VM 100)     |
| Nextcloud     | 35.0.1             |
| MariaDB       | LTS (Container)    |
| Redis         | Alpine (Container) |

## Einstellungen der VM

|    Einstellung   |                    Wert                     |
|        ---       |       ---           ---          ---        |
| CPU              | 4 Kerne, Typ `host`                         |
| Arbeitsspeicher  | 8 GB                                        |
| Systemfestplatte | 64 GB (LVM, auf 61 GB nutzbar erweitert)    |
| Netzwerk         | virtio, Bridge `vmbr0`                      |
| QEMU Guest Agent | aktiviert                                   |
| Datenfestplatte  | externe 1TB-Festplatte über USB-Passthrough |

## Snapshots und Backups

|         Name       |                     Zweck                       |
|         ---        |        ---           ---           ---          |
| `vor-docker`       | Zustand vor der Docker-Installation             |
| `nextcloud-laeuft` | Nextcloud läuft, vor der Verschiebung der Daten |
| `vor-datenumzug`   | Vor der Verschiebung auf die externe Festplatte |

Automatisches Backup (vzdump): Montag bis Freitag, 00:00 Uhr, Modus Snapshot,
Komprimierung ZSTD. Grösse des ersten Backups: 4.39 GiB.

**Wichtig:** Ein Snapshot ist kein Backup. Er liegt auf derselben Festplatte wie
die VM und hilft nicht, wenn die SSD ausfällt.
