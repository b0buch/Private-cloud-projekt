# Architektur
 
Stand: 08.10.2026
 
## Übersicht
 
```
                    ┌─────────────────────────────────┐
                    │  Internet                       │
                    └────────────────┬────────────────┘
                                     │ nur über VPN
                    ┌────────────────▼────────────────┐
                    │  FRITZ!Box 7490, 192.168.178.1  │
                    │  WireGuard (Verwaltung) 51820   │
                    │  Portfreigabe UDP 51821 → VM    │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │  Server (Hardware)              │
                    │  i5-11400, 16GB RAM             │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │  Proxmox VE 9.2.2               │
                    │  192.168.178.100                │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │  VM 100: Ubuntu Server          │
                    │  4 CPU-Kerne, 8 GB RAM          │
                    │  192.168.178.220                │
                    │  wg0 10.9.0.1 (Server-VPN)      │
                    │  dnsmasq nur auf 10.9.0.1       │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │  Docker Engine                  │
                    └────────────────┬────────────────┘
                                     │
                         ┌───────────▼─────────┐
                         │   nextcloud-caddy   │  ← Port 443 / 80
                         │   Reverse Proxy     │
                         └───────────┬─────────┘
                                     │ HTTP intern
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
  ┌──────▼──────┐            ┌───────▼──────┐            ┌───────▼──────┐
  │ nextcloud-  │            │ nextcloud-db │            │ nextcloud-   │
  │ app         │◄──────────►│ MariaDB      │            │ redis        │
  │ Apache/PHP  │            │              │◄──────────►│ Cache        │
  └─────────────┘            └──────────────┘            └──────────────┘
```
 
Nur der Reverse Proxy ist erreichbar. Die Portfreigabe von Nextcloud wurde
entfernt, der Zugriff erfolgt ausschliesslich über Caddy.
 
Aus dem Internet ist nichts davon direkt erreichbar. Es gibt keine
Portweiterleitung auf Port 443. Wer von aussen zugreifen will, baut zuerst
einen VPN-Tunnel auf:
 
| VPN          | Endpunkt   | Reichweite        | Für wen       |
| ---          | ---        | ---               | ---           |
| Router-VPN   | FRITZ!Box  | ganzes Heimnetz   | Verwaltung    |
| Server-VPN   | VM, `wg0`  | nur Nextcloud     | Benutzer      |
 
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
 
**ext4 auf dem Host, statt die Festplatte der VM direkt durchzureichen.** Eine
durchgereichte Festplatte würde die Snapshot-Funktion von Proxmox für die ganze
VM unmöglich machen.
 
**qcow2 statt raw.** Nur qcow2 unterstützt Snapshots. Die Datei ist dünn belegt:
Sie meldet 3500 GiB und belegte direkt nach dem Erstellen 548 MiB. Damit
gelöschte Dateien den Platz wieder freigeben, ist `discard=on` gesetzt.
 
**LVM in der VM.** Weitere Festplatten lassen sich im laufenden Betrieb in
dieselbe Volume Group aufnehmen, ohne dass Nextcloud etwas davon merkt:
 
```bash
pvcreate /dev/sdX
vgextend clouddata-vg /dev/sdX
lvextend -l +100%FREE /dev/clouddata-vg/clouddata-lv
resize2fs /dev/clouddata-vg/clouddata-lv
```
 
**Nachteil von LVM ohne RAID:** Fällt eine Festplatte aus, ist das ganze
Dateisystem betroffen. Ein Backup ist deshalb nicht optional.
 
### Zwei Dateisysteme, absichtlich entgegengesetzt formatiert
 
|      Dateisystem       |    Parameter   |    Inodes   | Inode-Tabelle |              Begründung                |
|          ---           |      ---       |     ---     |      ---      |                 ---                    |
| `/mnt/hdd4tb` (Host)   | `-T largefile` |   3 815 552 |   ca. 0.9 GiB | Wenige Objekte: ein Abbild, Archive    |
| `/mnt/clouddata` (VM)  | `-i 131072`    |  28 672 000 |   ca. 7 GiB   | Viele Dateien samt Vorschaubildern     |
| Voreinstellung wäre    | `-i 16384`     | 229 376 000 |  ca. 55 GiB   | Auf Dateien von 16 KiB ausgelegt       |
 
Beide Male zusätzlich `-m 1`: Der für root reservierte Anteil sinkt von 5 % auf
1 %. Die Anzahl der Inodes wird beim Formatieren festgelegt und lässt sich
später nicht erhöhen. Kontrolle mit `df -i`.
 
## Ordnerstruktur auf dem Server
 
```
/srv/nextcloud/
├── docker-compose.yml      Definition der vier Container
├── .env                    Passwörter (nicht in Git!)
├── nextcloud-image/
│   └── Dockerfile          eigenes Image mit ffmpeg
├── caddy/
│   ├── Caddyfile           Konfiguration des Reverse Proxy
│   ├── certs/              Zertifikat und privater Schlüssel (nicht in Git!)
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
└── Foto-Video/             Spiegel des Ordners Foto-Video (rsync --delete)
 
/srv/backup/
├── db/                     Sicherungen der Datenbank und von config.php
├── nextcloud-db-backup.sh
└── nextcloud-data-backup.sh
 
/etc/wireguard/
└── wg0.conf                Server-VPN (Schlüssel, Teilnehmer, Regeln)
/etc/dnsmasq.d/
└── wireguard.conf          Namensdienst für das Server-VPN
/etc/systemd/system/
├── nextcloud-*.service     Hintergrundaufgaben und Sicherungen
├── nextcloud-*.timer       Zeitpläne
└── notify-failure@.service Vorlage für E-Mails bei Fehlern
/usr/local/bin/
└── notify-failure.sh
/etc/msmtprc                Zugang für den Mailversand (chmod 600)
 
/mnt/hdd4tb/                                                → 4TB (Host)
├── images/100/             virtuelle Festplatte der VM
└── dump/                   Archive von vzdump
```
 
Beim Wechsel auf eine grössere Festplatte muss nur der Mountpunkt geändert
werden, Programmcode und Datenbank bleiben unberührt. Beim Umzug auf die
4TB-Festplatte am 02.10.2026 hat das die Unterbrechung auf etwa zehn Minuten
begrenzt.
 
### Einbinden über UUID
 
```
UUID=46cc1783-... /mnt/clouddata     ext4 defaults,nofail 0 2
UUID=3c3621b6-... /mnt/backup-extern ext4 defaults,nofail,x-systemd.device-timeout=10 0 2
```
 
Am 04.10.2026 tauschten `sdb` und `sdc` nach einem Neustart ohne Zutun die
Plätze. Mit Gerätenamen wäre die Sicherungsfestplatte als Datenverzeichnis
eingebunden worden.
 
### Startreihenfolge
 
Zwei Dienste hängen von anderen ab, und beides ist ausdrücklich festgelegt:
 
| Dienst          | wartet auf          | Grund                                                     |
| ---             | ---                 | ---                                                       |
| Docker          | `/mnt/clouddata`    | Sonst bindet Docker das leere Verzeichnis ein (04.10.2026)|
| `wg-quick@wg0`  | Docker              | Die Kette `DOCKER-USER` existiert erst nach Docker        |
 
```
# /etc/systemd/system/docker.service.d/override.conf
[Unit]
RequiresMountsFor=/mnt/clouddata
 
# /etc/systemd/system/wg-quick@wg0.service.d/override.conf
[Unit]
After=docker.service
Requires=docker.service
OnFailure=notify-failure@%n.service
```
 
`/mnt/backup-extern` steht absichtlich **nicht** bei Docker: Diese Festplatte
ist zeitweise abgezogen, und Docker würde dann überhaupt nicht starten.
 
## Netzwerk
 
```
Internet ──► FRITZ!Box 7490 ──► Server
  (VPN)      192.168.178.1        │
                                  ▼
                            Proxmox 192.168.178.100
                                 └── Ubuntu-VM 192.168.178.220
                                         ├── wg0 10.9.0.1/24 (Server-VPN)
                                         ├── Docker-Netz 172.18.0.0/16
                                         └── Nextcloud über Caddy, Port 443
```
 
|  Komponente  |                Wert                        |
|     ---      |                 ---                        |
| Gateway      | 192.168.178.1 (FRITZ!Box)                  |
| Proxmox      | 192.168.178.100, Adresse reserviert        |
| Ubuntu-VM    | 192.168.178.220, Adresse reserviert        |
| VPN-Netz     | 10.9.0.0/24, nur in der VM                 |
| Docker-Netz  | 172.18.0.0/16                              |
| DNS          | FRITZ!Box über DHCP, im Server-VPN dnsmasq |
| Zugriff      | `https://privatecloud.fritz.box`           |
 
Beide Adressen kommen über DHCP mit einer Reservierung in der FRITZ!Box, nicht
als feste Konfiguration im Betriebssystem. Vorteil: Ein Wechsel des Netzes
erfordert keine Änderung am Server.
 
Der DNS-Server der VM muss die FRITZ!Box sein und kein öffentlicher. Nur sie
kennt die Namen im Heimnetz unter `.fritz.box`. Mit `1.1.1.1` konnte der
Webserver sich nicht selbst erreichen, und sechs Selbstprüfungen von Nextcloud
schlugen fehl — behoben am 05.10.2026.
 
Die Teilnehmer des Server-VPN erreicht die FRITZ!Box nicht, weil sie das Netz
`10.9.0.0/24` nicht kennt. Für sie beantwortet dnsmasq in der VM die Namen und
leitet alles Übrige an die FRITZ!Box weiter.
 
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
## HTTPS
 
|                | Wert                                             |
| ---            | ---                                              |
| Reverse Proxy  | Caddy                                            |
| Zertifikat     | selbst signiert, 825 Tage, `CA:TRUE`             |
| Schlüssellänge | RSA 2048                                         |
| Gültig für     | `DNS:privatecloud.fritz.box`, `DNS:cloud.fritz.box`, `DNS:localhost`, `IP:192.168.178.220` |
| Weiterleitung  | http wird auf https umgeleitet                   |
 
```
:443 {
    tls /certs/server.crt /certs/server.key
    reverse_proxy app:80
}
 
:80 {
    redir https://{host}{uri} permanent
}
```
 
Das Caddyfile definiert den Dienst über den **Port**, nicht über einen Namen:
Beim Aufruf über eine IP-Adresse sendet der Browser kein SNI, weil der
Standard dort nur Domainnamen erlaubt. Ein Block mit einem Namen könnte dann
nie zugeordnet werden. Der Aufruf über die Adresse wird im Server-VPN
tatsächlich verwendet.
 
In der Weiterleitung steht `{host}` und keine feste Adresse. `{host}` ist der
Name, den der Client selbst angefragt hat.
 
**Der Name ist die Hauptadresse.** Das Zertifikat wurde am 02.10.2026 auf den
Namen ausgestellt, damit ein Wechsel des Netzes es nicht ungültig macht. Am
06.10.2026 wurde zusätzlich `IP:192.168.178.220` aufgenommen, weil Geräte im
Server-VPN zuerst nur über die Adresse zugreifen konnten. Ändert sich die
Adresse der VM, muss diese eine Zeile angepasst werden, der Zugriff über den
Namen funktioniert unabhängig davon.
 
## Einstellungen für den Betrieb hinter einem Proxy
 
|    Einstellung      |                 Wert                  |                            Zweck                               |
|        ---          |                 ---                   |                             ---                                |
| `trusted_proxies`   | `172.18.0.0/16`                       | Nur dem Docker-Netz wird der Header `X-Forwarded-For` geglaubt |
| `overwriteprotocol` | `https`                               | Erzeugte Links verwenden https                                 |
| `overwrite.cli.url` | `https://privatecloud.fritz.box`      | Basisadresse für Hintergrundaufgaben und Selbstprüfungen       |
| `trusted_domains`   | Name und Adressbereiche mit `*`       | Erlaubte Adressen für den Zugriff                              |
 
Die beiden Einträge aus dem Netz des Arbeitsplatzes (`192.168.137.*`) wurden
nach dem Umzug aus `trusted_domains` entfernt.
 
## Geplante Aufgaben
 
Seit dem 06.10.2026 als **systemd-Timer**, ausgeführt als root:
 
| Unit                          | Zeitplan        | `Persistent` | `OnFailure` |
| ---                           | ---             | ---          | ---         |
| `nextcloud-cron`              | alle 5 Minuten  | –            | nein        |
| `nextcloud-db-backup`         | täglich 01:30   | ja           | E-Mail      |
| `nextcloud-data-backup`       | Samstag 02:30   | ja           | E-Mail      |
 
Zuvor lagen die Aufgaben in `/etc/cron.d/nextcloud`. cron holt eine
verpasste Ausführung aber nicht nach, und der Server läuft nicht rund um die
Uhr. Am 05.10.2026 zeigte sich, dass deshalb keine einzige Sicherung
automatisch gelaufen war.
 
Drei Einzelheiten machen die Timer zuverlässig:
 
- `Persistent=true` holt eine verpasste Ausführung nach dem Start nach
- `ExecStartPre` wartet, bis Datenbank und Nextcloud bereit sind, weil eine
  nachgeholte Sicherung sehr früh im Startvorgang läuft
- `UMask=0077` macht neue Sicherungen nur für root lesbar
Die Hintergrundaufgaben von Nextcloud lösen bei einem Fehler absichtlich
**keine** E-Mail aus: Sie laufen alle fünf Minuten, und bei einer Störung
kämen zwölf Nachrichten pro Stunde. Ihre Fehler meldet Nextcloud selbst in
den Selbstprüfungen.
 
## Was absichtlich anders gebaut wurde
 
|      Naheliegend          |            Gewählt                        |                        Grund                                        |
|          ---              |             ---                           |                         ---                                         |
| Festplatte durchreichen   | qcow2 im Dateisystem                      | Sonst keine Snapshots der VM mehr möglich                           |
| RAID für die Daten        | LVM ohne RAID                             | RAID braucht gleich grosse Festplatten; Backup ist wichtiger        |
| Let's Encrypt             | selbst signiertes Zertifikat              | Let's Encrypt verlangt Erreichbarkeit aus dem Internet              |
| Zertifikat nur auf die IP | Zertifikat auf den Namen, IP zusätzlich   | Überlebt einen Wechsel der Adresse                                  |
| `pvmove` beim Umzug       | rsync                                     | `pvmove` verschiebt zugewiesene Extents, nicht belegte Daten        |
| Nextcloud direkt          | Reverse Proxy davor                       | Verschlüsselung an einer Stelle, Header für alle Dienste            |
| feste IP im System        | DHCP mit Reservierung                     | Ein Netzwechsel erfordert keine Änderung am Server                  |
| cron                      | systemd-Timer                             | cron holt verpasste Ausführungen nicht nach                         |
| Router-VPN für alle       | zweites VPN in der VM                     | Der Router kann den Zugriff einzelner Teilnehmer nicht einschränken |
| Regeln in ufw             | Kette `DOCKER-USER`                       | Docker-Ports laufen durch FORWARD, ufw sieht sie nicht              |
| Kontrolle von Hand        | E-Mail über `OnFailure=`                  | Ein Fehler soll auffallen, ohne dass jemand nachsieht               |