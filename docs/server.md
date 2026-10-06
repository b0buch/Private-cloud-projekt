# Server und Anschluss
 
Stand: 06.10.2026
 
## Hardware
 
|         Komponente      |             Wert              |                         Rolle                           |
|            ---          |              ---              |                          ---                            |
| Prozessor               | Intel Core i5-11400           |                                                         |
| Arbeitsspeicher         | 16 GB, 2400 MHz               |                                                         |
| SSD                     | Kingston SNVS250G M.2, 250 GB | Proxmox, VM, Docker, Datenbank                          |
| Festplatte              | 4 TB, Seagate IronWolf        | Daten von Nextcloud, Sicherung von System und Datenbank |
| Festplatte extern (USB) | 1 TB                          | Sicherungsziel (rsync), ext4, /mnt/backup-extern        |
 
Die ursprünglich geplante zweite SSD mit 128 GB und eine zusätzliche interne
Festplatte mit 1 TB werden nicht verwendet. Das Sicherungskonzept wurde
entsprechend angepasst, siehe `backup-konzept.md`.
 
## Anschluss
 
|    Komponente    |                Wert               |
|       ---        |                ---                |
| Router           | FRITZ!Box 7490, FRITZ!OS 7.62     |
| Internetanbieter | InterXS                           |
| Geschwindigkeit  | 50/50 Mbit/s                      |
| Öffentliche IPv4 | vorhanden, dynamisch (kein CGNAT) |
| IPv6             | feste /64-Adresse                 |
 
Der Anschluss verfügt über eine öffentliche, dynamische IPv4-Adresse.
Die früher dokumentierte Annahme von Carrier-Grade-NAT war falsch und wurde
am 04.10.2026 durch Vergleich der im Online-Monitor angezeigten Adresse mit
einer externen Abfrage widerlegt. Portweiterleitung ist damit möglich, eine
zusätzliche statische IPv4-Adresse ist nicht notwendig. Die wechselnde
Adresse wird über MyFRITZ! als Name erreichbar gemacht.
 
## Software
 
|   Komponente  |             Version            |                    Bemerkung                         |
|      ---      |               ---              |                       ---                            |
| Proxmox VE    | 9.2.2                          | Hypervisor, Paketquelle `no-subscription`            |
| Ubuntu Server | 26.04 (VM 100)                 | Zeitzone Europe/Zurich                               |
| Docker Engine | aus dem offiziellen Repository |                                                      |
| Nextcloud     | 35.0.1                         | eigenes Image `nextcloud-ffmpeg:latest`              |
| MariaDB       | LTS                            | mit angepassten Parametern, siehe datenbank.md       |
| Redis         | Alpine (Container)             | Cache und Dateisperren                               |
| Caddy         | Alpine (Container)             | Reverse Proxy für HTTPS                              |
| ffmpeg        | 7.1.5                          | im eigenen Image, für Vorschaubilder von Videos      |
| WireGuard     | FRITZ!OS 7.62                  | Router-VPN für die Verwaltung, UDP 51820             |
| WireGuard     | in der VM, `wg0`               | Server-VPN für die Benutzer, UDP 51821, siehe vpn.md |
| dnsmasq       | in der VM                      | Namensdienst nur für das Server-VPN, `10.9.0.1`      |
| msmtp         | in der VM                      | E-Mail bei fehlgeschlagenen Diensten                 |
 
### Paketquellen von Proxmox
 
Seit der Installation waren die Enterprise-Paketquellen eingetragen. Ohne
Abonnement schlug dadurch jedes `apt-get update` fehl (festgestellt am
29.09.2026). Am 06.10.2026 unter `pve` → Updates → Repositories die
Enterprise-Quellen deaktiviert und `No-Subscription` hinzugefügt. Danach
`apt full-upgrade` — bei Proxmox `full-upgrade`, nicht `upgrade`, weil
Paketabhängigkeiten sich ändern können.
 
Die Meldung "Keine gültige Subskription" beim Anmelden bleibt bestehen. Sie
lässt sich nicht offiziell abschalten und ändert nichts an der Funktion.
 
## Einstellungen der VM 100
 
|    Einstellung   |                          Wert                              |
|        ---       |                          ---                               |
| CPU              | 4 Kerne, Typ `host`                                        |
| Arbeitsspeicher  | 8 GB                                                       |
| Systemfestplatte | 64 GB (LVM, auf 61 GB erweitert), discard=on, ssd=1        |
| Datenfestplatte  | virtuelle Festplatte 3500 GiB, qcow2, backup=0, discard=on |
| Netzwerk         | virtio, Bridge `vmbr0`                                     |
| QEMU Guest Agent | aktiviert                                                  |
| Autostart        | onboot=1, startup order=1                                  |
| USB              | externe 1TB-Festplatte als Sicherungsziel                  |
 
## Speicher der Datenfestplatte
 
Die Datenfestplatte ist nicht direkt formatiert, sondern über LVM eingebunden:
 
```
/dev/sda1 (4TB)              ext4, -T largefile, /mnt/hdd4tb   ← Host
    └── vm-100-disk-2.qcow2  3500 GiB, dünn belegt
            └── /dev/sdb     Physical Volume                   ← VM
                └── clouddata-vg          Volume Group, 3.42 TiB
                        └── clouddata-lv  ext4, -i 131072
```
 
Eingebunden unter `/mnt/clouddata` über die UUID in `/etc/fstab`, mit der Option
`nofail`. Der für root reservierte Speicher wurde beim Formatieren mit `-m 1`
von 5 % auf 1 % gesenkt. Verfügbar sind dadurch rund 3.4 TB.
 
Grund für LVM: Weitere Festplatten lassen sich im laufenden Betrieb in
dieselbe Volume Group aufnehmen, ohne dass Nextcloud etwas davon merkt oder
Daten kopiert werden müssen.
 
Die beiden Dateisysteme sind absichtlich unterschiedlich formatiert. Auf dem
Host liegen nur wenige, sehr grosse Dateien (ein Festplattenabbild, die
Archive von vzdump), deshalb `-T largefile`: 3 815 552 Inodes statt rund
244 Millionen, Inode-Tabelle etwa 0.9 GiB statt 58 GiB. In der VM liegen
viele Dateien, weil Nextcloud zu jedem Foto mehrere Vorschaubilder ablegt,
deshalb `-i 131072`: 28 672 000 Inodes. Die Anzahl der Inodes wird beim
Formatieren festgelegt und lässt sich später nicht erhöhen. Kontrolle mit
`df -i`.
 
## Netzwerk
 
|           |                   zu Hause                       |
|    ---    |                     ---                          |
| Gateway   | 192.168.178.1 (FRITZ!Box)                        |
| Proxmox   | 192.168.178.100 (Reservierung)                   |
| Ubuntu-VM | 192.168.178.220 (Reservierung)                   |
| DNS       | FRITZ!Box, per DHCP zugewiesen                   |
| VPN-Netz  | 10.9.0.0/24 (`wg0`, nur innerhalb der VM)        |
| Zugriff   | `https://privatecloud.fritz.box` über VPN        |
 
Der Port 8080 wird nicht mehr veröffentlicht. Nextcloud ist nur noch über den
Reverse Proxy erreichbar.
 
Offene Ports der VM:
 
| Port      | Schnittstelle | Dienst                         |
| ---       | ---           | ---                            |
| TCP 22    | `ens18`       | SSH, nur aus dem Heimnetz      |
| TCP 80    | alle          | Weiterleitung auf HTTPS        |
| TCP 443   | alle          | Nextcloud über Caddy           |
| UDP 51821 | `ens18`       | WireGuard (Server-VPN)         |
| UDP/TCP 53| `wg0`         | dnsmasq, nur für das VPN       |
 
## Zeit
 
Die VM läuft seit dem 06.10.2026 in der Zeitzone **Europe/Zurich**, vorher in
UTC. Alle Zeitpläne der systemd-Timer gelten damit in Schweizer Zeit.
 
**Ausnahme:** Das Wartungsfenster von Nextcloud
(`maintenance_window_start = 2`) wird immer in **UTC** angegeben, unabhängig
von der Zeitzone des Systems. Es beginnt also um 04:00 Sommerzeit bzw. 03:00
Winterzeit und überschneidet sich nicht mit den Sicherungen.
 
## Snapshots und Backups
 
|         Name       |                     Zweck                       |   Stand  |
|         ---        |                      ---                        |    ---   |
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
 
## Geplante Aufgaben
 
Seit dem 06.10.2026 als **systemd-Timer** statt cron. Grund: Der Server läuft
nicht rund um die Uhr, und cron holt verpasste Ausführungen nicht nach. Die
alte Datei `/etc/cron.d/nextcloud` wurde nach `/root/nextcloud.cron.bak`
verschoben.
 
|              Timer            |       Zeitpunkt       | Nachholen   |          Bei Fehler         |
|               ---             |          ---          |    ---      |             ---             |
| `nextcloud-cron.timer`        | alle 5 Minuten        | nicht nötig | keine E-Mail (würde fluten) |
| `nextcloud-db-backup.timer`   | täglich 01:30         | ja          | E-Mail                      |
| `nextcloud-data-backup.timer` | Samstag 02:30         | ja          | E-Mail                      |
 
Auf dem Proxmox-Host läuft zusätzlich der Sicherungsauftrag von vzdump
(Mo–Fr 00:00) mit der Option "Verpasste wiederholen".
 
Übersicht und nächste Ausführung:
 
```bash
systemctl list-timers 'nextcloud*'
```
 
### Benachrichtigung bei Fehlern
 
Schlägt eine Sicherung oder der Start von WireGuard fehl, verschickt systemd
eine E-Mail mit Status und den letzten 30 Zeilen des Protokolls.
 
|            Teil                               |                        Aufgabe                              |
|             ---                               |                          ---                                |
| `OnFailure=notify-failure@%n.service`         | in der Unit, die überwacht wird                             |
| `/etc/systemd/system/notify-failure@.service` | Vorlage, `%i` wird zum Namen der fehlgeschlagenen Unit      |
| `/usr/local/bin/notify-failure.sh`            | sammelt Status und Protokoll, übergibt an msmtp             |
| `/etc/msmtprc` (chmod 600)                    | Zugang zu Gmail, eigenes App-Passwort `privatecloud-system` |
 
Für den Mailversand des Systems wird ein **eigenes** App-Passwort verwendet,
nicht dasjenige von Nextcloud. So lassen sich beide unabhängig widerrufen.
 
## Sicherungen
 
|        Was         |         Werkzeug              |            Zeitpunkt               |                 Ziel                   |
|        ---         |           ---                 |               ---                  |                 ---                    |
| Virtuelle Maschine | vzdump, Modus Snapshot, ZSTD  | Mo–Fr 00:00, verpasste wiederholen | hdd4tb (4TB), `keep-last=3`            |
| Datenbank          | `mysqldump` mit Wartungsmodus | täglich 01:30, Persistent          | `/srv/backup/db` (in vzdump enthalten) |
| Daten              | rsync                         | Samstag 02:30, Persistent          | `/mnt/backup-extern` (1TB)             |
 
Die erste automatische Ausführung ist am 06.10.2026 nachgewiesen: vzdump um
00:00 (40 Sekunden, 5.27 GB) und die Sicherung der Datenbank um 01:30.
 
Neue Sicherungen sind nur für root lesbar (`UMask=0077` in den Diensten).
 
## Getestete Wiederherstellungen
 
|                  Test                        |   Datum    |                    Ergebnis                         |
|                  ---                         |    ---     |                      ---                            |
| VM aus vzdump als neue VM 999                | 29.09.2026 | bestanden                                           |
| Datenbank aus mysqldump                      | 29.09.2026 | bestanden                                           |
| Ausfall der Datenfestplatte im Betrieb       | 29.09.2026 | Nextcloud meldet den Fehler, keine Daten beschädigt |
| Automatischer Start nach Neustart            | 29.09.2026 | bestanden                                           |
| VM aus vzdump auf hdd4tb als VM 999          | 02.10.2026 | bestanden                                           |
| Nachholen einer verpassten Sicherung         | 06.10.2026 | bestanden, 12 Sekunden nach dem Start               |
| E-Mail bei fehlgeschlagenem Dienst           | 06.10.2026 | bestanden                                           |
 
## Gemessene Grösse der VM-Sicherung
 
|                Zeitpunkt                 |  Grösse   |
|                   ---                    |    ---    |
| 29.09.2026, ohne discard                 | 4.39 GiB  |
| 02.10.2026, vor fstrim                   | 15.40 GiB |
| 02.10.2026, mit discard=on und fstrim    | 4.05 GiB  |
| 06.10.2026, erste automatische Sicherung | 5.27 GB   |