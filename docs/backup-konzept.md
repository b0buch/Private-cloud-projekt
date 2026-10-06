# Backup-Konzept

## Konzept (Stand 30.09.2026)

Grundsatz: Eine Sicherung muss auf einer anderen physischen Festplatte liegen
als das Original. Deshalb kann die grosse Datenfestplatte gleichzeitig die
Sicherung des Systems aufnehmen, obwohl sie selbst Originaldaten enthaelt.

|    Festplatte    |         Original          |                    Sicherung von                           | 
|       ---        |            ---            |      ---                ---               ---              |
| SSD 250 GB (M.2) | System, Docker, Datenbank | –                                                          |
| HDD 4 TB         | Fotos und Videos          | System (vzdump) und Datenbank (ca. 1 MB pro Dump)          |
| HDD 1 TB extern  | –                         | 500 GB wichtige Fotos und Videos, zweite Kopie des Systems |

### Ablauf

|            Was            | Werkzeug  |      Ziel       |   Haeufigkeit   |
|   ---      ---    ---     |   ---     |      ---        |       ---       |
| Virtuelle Maschine        | vzdump    | HDD 4 TB        | taeglich Mo–Fr  |
| Datenbank                 | mysqldump | HDD 4 TB        | taeglich        |
| Wichtige Fotos und Videos | rsync     | HDD 1 TB extern | woechentlich    |
| Konfigurationsdateien     | von Hand  | USB-Stick 64 GB | bei Aenderungen |

### Ausschluesse beim rsync

Nicht gesichert werden die Vorschaubilder (`appdata_*/preview/`) und der
Papierkorb (`*/files_trashbin/`). Vorschaubilder erzeugt Nextcloud bei Bedarf
aus den Originalen neu und sie brauchen viel Platz. Der Papierkorb enthaelt
vom Benutzer bereits geloeschte Dateien. Beides ist ein bewusst akzeptiertes
Risiko: Faellt die 4TB-Festplatte aus, ist der Papierkorb verloren.

### Aufbewahrung

- vzdump: `keep-last=5` in der Speicherkonfiguration von Proxmox
- Datenbank: `find /srv/backup/db -name '*.sql.gz' -mtime +30 -delete`
- rsync: Spiegel mit `--delete`, keine Versionen

### Einschraenkung: der Server laeuft nicht rund um die Uhr

Cron fuehrt eine Aufgabe nur aus, wenn der Rechner zu diesem Zeitpunkt
laeuft, und holt sie nicht nach. Deshalb wurden die geplanten Sicherungen bis
zum 05.10.2026 nie automatisch ausgefuehrt, obwohl die Zeiten seit dem
02.10.2026 eingetragen waren. Festgestellt durch Kontrolle von
`/srv/backup/db/` und `/var/log/nextcloud-data-backup.log`: dort standen nur
die von Hand gestarteten Durchgaenge.

Geplante Loesung: Der Server weckt sich nachts ueber die Hardware-Uhr
(`rtcwake`), fuehrt die Sicherungen der Reihe nach aus und schaltet sich
wieder ab. Zusaetzlich systemd-Timer mit `Persistent=true`, die eine
verpasste Ausfuehrung nach dem naechsten Start nachholen — etwas, das cron
nicht kann.

### Warum kein USB-Stick fuer die taeglichen Sicherungen

Ein taegliches Backup der VM schreibt etwa 4.4 GB, im Jahr also rund 1.5 TB.
Guenstige USB-Sticks haben eine einfache Speicherverwaltung und halten diese
Schreiblast nur ein bis zwei Jahre aus. Sie fallen zudem meist ohne Vorwarnung
aus, was erst im Ernstfall auffaellt.

Der Stick wird stattdessen fuer die Konfigurationsdateien verwendet
(docker-compose.yml, .env, config.php, Backup-Skripte, Notizen mit den UUIDs
der Festplatten). Diese sind nur wenige Megabyte gross, werden selten
geschrieben und waeren am schwersten wiederherzustellen.

### Bewusst akzeptiertes Risiko

Von den bis zu 4 TB Daten werden nur die 500 GB gesichert, die als wichtig
eingestuft sind. Faellt die 4TB-Festplatte aus, gehen die uebrigen Videos
verloren. Diese Entscheidung ist bewusst getroffen: Eine vollstaendige
Sicherung wuerde eine weitere Festplatte mit 4 TB erfordern.

Eine Erweiterung ist in ein bis zwei Jahren vorgesehen, falls die Cloud
regelmaessig genutzt wird und die Datenmenge tatsaechlich waechst.

### Regel fuer "wichtig" (festgelegt am 06.10.2026)

Gesichert wird ausschliesslich der gemeinsame Ordner **Foto-Video**. Die
persoenlichen Ordner der Benutzer werden **nicht** gesichert.

Begruendung: Die Sicherungsfestplatte hat 1 TB, die Datenfestplatte 4 TB. Eine
vollstaendige Sicherung ist damit unmoeglich, und eine Auswahl nach Gefuehl
waere nicht nachvollziehbar. Ein gemeinsamer Ordner als Kriterium ist eindeutig:
Wer etwas gesichert haben will, legt es dorthin.

Folge fuer die Benutzer: Dateien in den persoenlichen Ordnern existieren nur
einmal. Faellt die 4TB-Festplatte aus, sind sie verloren. Darauf werden die
Benutzer in der Anleitung ausdruecklich hingewiesen.

### Aufbewahrungsort

Die externe Festplatte sollte nicht neben dem Server liegen. Bei Feuer,
Diebstahl oder Ueberspannung waeren sonst alle Kopien gleichzeitig betroffen.
Sie wird an einem anderen Ort aufbewahrt und in regelmaessigen Abstaenden zum
Aktualisieren angeschlossen.