# Backup-Konzept
 
Stand: 08.10.2026
 
## Konzept
 
Grundsatz: Eine Sicherung muss auf einer anderen physischen Festplatte liegen
als das Original. Deshalb kann die grosse Datenfestplatte gleichzeitig die
Sicherung des Systems aufnehmen, obwohl sie selbst Originaldaten enthält.
 
|    Festplatte    |         Original          |                    Sicherung von                           |
|       ---        |            ---            |                         ---                                |
| SSD 250 GB (M.2) | System, Docker, Datenbank | –                                                          |
| HDD 4 TB         | Fotos und Videos          | System (vzdump) und Datenbank (ca. 1 MB pro Dump)          |
| HDD 1 TB extern  | –                         | 500 GB wichtige Fotos und Videos (Ordner Foto-Video)       |
 
## Ablauf
 
|            Was            | Werkzeug  |      Ziel              |          Zeitpunkt               |       Ausgelöst durch          |
|            ---            |   ---     |      ---               |             ---                  |             ---                |
| Virtuelle Maschine        | vzdump    | HDD 4 TB               | Mo–Fr 00:00                      | Sicherungsauftrag von Proxmox  |
| Datenbank und config.php  | mysqldump | HDD 4 TB (über vzdump) | täglich 01:30                    | systemd-Timer                  |
| Wichtige Fotos und Videos | rsync     | HDD 1 TB extern        | Samstag 02:30                    | systemd-Timer                  |
| Konfigurationsdateien     | von Hand  | USB-Stick 64 GB        | bei Änderungen                   | –                              |
 
Alle Zeiten in Schweizer Zeit (Zeitzone der VM seit 06.10.2026
Europe/Zurich).
 
### Ausschlüsse beim rsync
 
Nicht gesichert werden die Vorschaubilder (`appdata_*/preview/`) und der
Papierkorb (`*/files_trashbin/`). Vorschaubilder erzeugt Nextcloud bei Bedarf
aus den Originalen neu und sie brauchen viel Platz. Der Papierkorb enthält
vom Benutzer bereits gelöschte Dateien. Beides ist ein bewusst akzeptiertes
Risiko: Fällt die 4TB-Festplatte aus, ist der Papierkorb verloren.
 
### Aufbewahrung
 
- vzdump: **wirksam ist `keep-last=3`** aus dem Sicherungsauftrag. In der
  Konfiguration des Speichers `hdd4tb` steht `keep-last=5`, die Einstellung
  im Auftrag hat aber Vorrang. Festgestellt am 06.10.2026 im Protokoll von
  vzdump (`prune older backups with retention: keep-last=3`).
- Datenbank: `find /srv/backup/db -name '*.sql.gz' -mtime +30 -delete`
- rsync: Spiegel mit `--delete`, keine Versionen
### Schutz der Sicherungen
 
Ein Dump der Datenbank enthält Passwort-Hashes, E-Mail-Adressen und die ganze
Benutzerstruktur, die gesicherte `config.php` zusätzlich `secret`,
`passwordsalt` und das SMTP-Passwort. Sicherungen müssen deshalb genauso
geschützt sein wie das Original.
 
Bis zum 06.10.2026 waren die Dumps mit `-rw-r--r--` für jeden Benutzer der VM
lesbar. Korrigiert mit `chmod 600`, und `UMask=0077` in den Diensten sorgt
dafür, dass neue Dateien von Anfang an nur für root lesbar sind.
 
## Der Server läuft nicht rund um die Uhr
 
### Problem (festgestellt am 05.10.2026)
 
Cron führt eine Aufgabe nur aus, wenn der Rechner zu diesem Zeitpunkt läuft,
und holt sie nicht nach. Deshalb wurden die geplanten Sicherungen bis zum
05.10.2026 nie automatisch ausgeführt, obwohl die Zeiten seit dem 02.10.2026
eingetragen waren. Festgestellt durch Kontrolle von `/srv/backup/db/` und
`/var/log/nextcloud-data-backup.log`: Dort standen nur die von Hand
gestarteten Durchgänge.
 
### Lösung (umgesetzt am 06.10.2026)
 
Die Sicherungen in der VM laufen über **systemd-Timer** mit
`Persistent=true`. systemd merkt sich den Zeitpunkt der letzten Ausführung.
War der Server zur geplanten Zeit aus, wird die Sicherung direkt nach dem
nächsten Start nachgeholt.
 
Eine nachgeholte Sicherung läuft sehr früh im Startvorgang, wenn die
Container möglicherweise noch nicht bereit sind. Der Dienst wartet deshalb
zuerst auf die Datenbank und auf Nextcloud:
 
```ini
ExecStartPre=/bin/sh -c 'until docker exec nextcloud-db mariadb-admin ping --silent && docker exec -u www-data nextcloud-app php occ status >/dev/null 2>&1; do sleep 10; done'
TimeoutStartSec=15min
```
 
Für vzdump auf dem Proxmox-Host gilt dasselbe über die Option
"Verpasste wiederholen" im Sicherungsauftrag.
 
**Test vom 06.10.2026:** Der Zeitstempel des Timers wurde um zwei Tage
zurückgesetzt und die VM neu gestartet:
 
```bash
touch -d "2 days ago" /var/lib/systemd/timers/stamp-nextcloud-db-backup.timer
reboot
```
 
Die Sicherung startete um 09:31:39, wartete etwa elf Sekunden auf die
Container und war um 09:31:51 fertig. Bestanden.
 
Der Nachtbetrieb mit `rtcwake` (siehe `loesungen.md`) ist damit keine
Voraussetzung mehr für die Sicherungen, sondern nur noch eine Möglichkeit,
Strom zu sparen.
 
### Benachrichtigung bei Fehlern
 
Eine Automatisierung muss man nicht nur einrichten, sondern auch nachweisen.
Damit ein Fehler nicht erst beim nächsten manuellen Blick auffällt, schickt
systemd bei einem fehlgeschlagenen Sicherungsdienst eine E-Mail
(`OnFailure=`, Einzelheiten in `server.md`).
 
Voraussetzung: Die Skripte müssen bei einem Fehler auch wirklich mit einem
Fehlercode enden. Deshalb beginnen beide mit `set -euo pipefail` (kontrolliert am 07.10.2026). Ohne
`pipefail` würde in der Kette `mysqldump | gzip` nur der Rückgabewert von
`gzip` zählen, und ein abgebrochener Dump bliebe unbemerkt.
 
## Warum kein USB-Stick für die täglichen Sicherungen
 
Ein tägliches Backup der VM schreibt etwa 4 bis 5 GB, im Jahr also rund
1.5 TB. Günstige USB-Sticks haben eine einfache Speicherverwaltung und halten
diese Schreiblast nur ein bis zwei Jahre aus. Sie fallen zudem meist ohne
Vorwarnung aus, was erst im Ernstfall auffällt.
 
Der Stick wird stattdessen für die Konfigurationsdateien verwendet
(docker-compose.yml, .env, config.php, Backup-Skripte, Notizen mit den UUIDs
der Festplatten). Diese sind nur wenige Megabyte gross, werden selten
geschrieben und wären am schwersten wiederherzustellen.
 
## Bewusst akzeptiertes Risiko
 
Von den bis zu 4 TB Daten werden nur die 500 GB gesichert, die als wichtig
eingestuft sind. Fällt die 4TB-Festplatte aus, gehen die übrigen Videos
verloren. Diese Entscheidung ist bewusst getroffen: Eine vollständige
Sicherung würde eine weitere Festplatte mit 4 TB erfordern.
 
Eine Erweiterung ist in ein bis zwei Jahren vorgesehen, falls die Cloud
regelmässig genutzt wird und die Datenmenge tatsächlich wächst.
 
### Regel für "wichtig" (festgelegt am 06.10.2026)
 
Gesichert wird ausschliesslich der gemeinsame Ordner **Foto-Video**. Die
persönlichen Ordner der Benutzer werden **nicht** gesichert.
 
Begründung: Die Sicherungsfestplatte hat 1 TB, die Datenfestplatte 4 TB. Eine
vollständige Sicherung ist damit unmöglich, und eine Auswahl nach Gefühl
wäre nicht nachvollziehbar. Ein gemeinsamer Ordner als Kriterium ist eindeutig:
Wer etwas gesichert haben will, legt es dorthin.
 
Folge für die Benutzer: Dateien in den persönlichen Ordnern existieren nur
einmal. Fällt die 4TB-Festplatte aus, sind sie verloren. Darauf werden die
Benutzer in der Anleitung ausdrücklich hingewiesen.
 
## Aufbewahrungsort
 
Die externe Festplatte sollte nicht neben dem Server liegen. Bei Feuer,
Diebstahl oder Überspannung wären sonst alle Kopien gleichzeitig betroffen.
Sie wird an einem anderen Ort aufbewahrt und in regelmässigen Abständen zum
Aktualisieren angeschlossen.
 
Ist sie zur geplanten Zeit nicht angeschlossen, bricht das Skript an der
Prüfung mit `findmnt` ab, und es kommt eine E-Mail. `Persistent=true` hilft
hier nicht: Der Timer gilt als ausgeführt, auch wenn der Dienst fehlschlug,
und läuft erst zum nächsten geplanten Zeitpunkt wieder. Nach dem Anschliessen
der Festplatte wird die Sicherung deshalb von Hand gestartet:
 
```bash
systemctl start nextcloud-data-backup.service
```

## Nachgewiesen

| Test                                              | Datum      | Ergebnis                                   |
| ---                                               | ---        | ---                                        |
| Wiederherstellung der VM als VM 999               | 29.09.2026 | bestanden, wiederholt am 02.10.2026        |
| Wiederherstellung der Datenbank                   | 29.09.2026 | bestanden                                  |
| Erste automatische Sicherung von VM und Datenbank | 06.10.2026 | bestanden                                  |
| Nachholen einer verpassten Sicherung              | 06.10.2026 | bestanden                                  |
| Sicherung ohne angeschlossene externe Festplatte  | 07.10.2026 | Abbruch erkannt, E-Mail angekommen         |
| Wiederherstellung aus der rsync-Sicherung         | 07.10.2026 | bestanden, `diff -rq` ohne Unterschiede    |
