# Backup-Konzept

## Konzept (Stand 30.09.2026)

Grundsatz: Eine Sicherung muss auf einer anderen physischen Festplatte liegen
als das Original. Deshalb kann die grosse Datenfestplatte gleichzeitig die
Sicherung des Systems aufnehmen, obwohl sie selbst Originaldaten enthaelt.

|    Festplatte    |         Original          |                    Sicherung von                           | 
|       ---        |            ---            |      ---                ---               ---              |
| SSD 250 GB (M.2) | System, Docker, Datenbank | –                                                          |
| HDD 4 TB         | Fotos und Videos          | System (vzdump) und Datenbank (ca. 20 GB)                  |
| HDD 1 TB extern  | –                         | 500 GB wichtige Fotos und Videos, zweite Kopie des Systems |

### Ablauf

|            Was            | Werkzeug  |      Ziel       |   Haeufigkeit   |
|   ---      ---    ---     |   ---     |      ---        |       ---       |
| Virtuelle Maschine        | vzdump    | HDD 4 TB        | taeglich Mo–Fr  |
| Datenbank                 | mysqldump | HDD 4 TB        | taeglich        |
| Wichtige Fotos und Videos | rsync     | HDD 1 TB extern | woechentlich    |
| Kopie des Systems         | rsync     | HDD 1 TB extern | woechentlich    |
| Konfigurationsdateien     | von Hand  | USB-Stick 64 GB | bei Aenderungen |

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

### Regel fuer "wichtig"

Damit die Auswahl nachvollziehbar bleibt, wird eine feste Regel definiert statt
nach Gefuehl entschieden, zum Beispiel: Alle Dateien im Ordner "Wichtig" sowie
alle Fotos der letzten zwei Jahre. Alles andere wird nicht gesichert.

### Aufbewahrungsort

Die externe Festplatte sollte nicht neben dem Server liegen. Bei Feuer,
Diebstahl oder Ueberspannung waeren sonst alle Kopien gleichzeitig betroffen.
Sie wird an einem anderen Ort aufbewahrt und in regelmaessigen Abstaenden zum
Aktualisieren angeschlossen.