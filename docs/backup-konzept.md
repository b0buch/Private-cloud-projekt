# Backup-Konzept

## Grundsatz

Ein Backup muss auf einer anderen physischen Festplatte liegen als das Original.
Ein Snapshot in Proxmox ist kein Backup, weil er auf derselben Festplatte liegt
wie die virtuelle Maschine.

Die Daten des Projekts sind in drei Gruppen aufgeteilt, die unterschiedlich
gesichert werden:

|            Gruppe             |         Grösse         | Wiederherstellbar ohne Backup?        |
|             ---               |          ---           |                 ---                   |
| System, Docker, Konfiguration | ca. 5 GB               | ja, aber mit mehreren Stunden Aufwand |
| Datenbank                     | klein, waechst langsam |                 nein                  |
| Fotos und Videos              | bis mehrere TB         |                 nein                  |

Deshalb werden drei getrennte Sicherungen gemacht, statt einer grossen.

## Stufe 1 — aktueller Stand

|        Was         |      Werkzeug       |    Ziel    |  Haeufigkeit   |
|        ---         |        ---          |    ---     |      ---       |
| Virtuelle Maschine | vzdump (Proxmox)    | SSD 128 GB | taeglich Mo-Fr |
| Datenbank          | mysqldump           | SSD 128 GB | taeglich       |
| Fotos und Videos   | rsync (ausgewaehlt) | HDD 1 TB   | woechentlich   |

Die 128-GB-SSD ist bereits vorhanden und wird nur fuer Sicherungen verwendet.
Ein Backup der VM belegt komprimiert etwa 5 GB, es haben also rund 20 Versionen
Platz. Die Schreiblast ist gering und fuer eine SSD unproblematisch.

**Wichtig:** Die Datenfestplatte darf nicht Teil des VM-Backups sein. In Proxmox
wird bei dieser Festplatte die Option "Backup" deaktiviert. Sonst waere das
Archiv mehrere Terabyte gross und wuerde auf keine Sicherungsfestplatte passen.

## Stufe 2 — geplante Erweiterung

Wenn mehr Daten vorhanden sind, werden zwei Festplatten mit je 1 TB fuer die
Sicherung der Fotos und Videos verwendet, eine intern und eine extern.

Die Aufteilung je Festplatte:

|  Bereich   | Groesse |               Inhalt                   |
|    ---     |   ---   |                ---                     |
| `wichtig/` | 500 GB  | dieselben Daten auf beiden Festplatten |
| `unique/`  | 500 GB  | unterschiedliche Daten je Festplatte   |

Rechnung: 500 GB (wichtig, doppelt) + 500 GB (unique Platte 1)
+ 500 GB (unique Platte 2) = **1500 GB gesicherte Daten** bei 2 TB Kapazitaet.
Die 500 GB der wichtigsten Daten liegen in zwei Exemplaren.

Vorteil: Eine der beiden Festplatten ist extern und wird ausserhalb des
Serverraums aufbewahrt. Damit sind die wichtigsten Daten auch bei Feuer,
Diebstahl oder Ueberspannung geschuetzt, weil diese Ereignisse alle Festplatten
im Gehaeuse gleichzeitig treffen wuerden.

Nachteil: Die Zuteilung muss von Hand festgelegt werden. Damit das
nachvollziehbar bleibt, wird eine feste Regel definiert statt nach Gefuehl
entschieden, zum Beispiel:

> "wichtig" sind alle Fotos und Videos der letzten zwei Jahre sowie der Ordner
> "Favoriten". Alles andere ist "unique" und wird nur einfach gesichert.

Beispiel fuer die Umsetzung mit rsync:

    # Festplatte 1 (intern)
    rsync -a --delete /mnt/clouddata/<benutzer>/files/Wichtig/ /mnt/backup1/wichtig/
    rsync -a --delete /mnt/clouddata/<benutzer>/files/2024/    /mnt/backup1/unique/

    # Festplatte 2 (extern)
    rsync -a --delete /mnt/clouddata/<benutzer>/files/Wichtig/ /mnt/backup2/wichtig/
    rsync -a --delete /mnt/clouddata/<benutzer>/files/2023/    /mnt/backup2/unique/

## Bekannte Grenze

Bei einer Datenfestplatte von 4 TB und 1.5 TB Sicherungskapazitaet sind rund
37 Prozent der moeglichen Datenmenge gesichert. Solange wenig Daten vorhanden
sind, reicht das. Wenn die 4TB-Festplatte voller wird, muss entweder eine
weitere Sicherungsfestplatte gekauft oder bewusst festgelegt werden, welche
Daten nicht gesichert werden.

Diese Entscheidung ist bewusst getroffen und kein Versehen.

## Wiederherstellung

Ein Backup, das nie zurueckgespielt wurde, ist kein Backup. Deshalb wird jede
Sicherungsart einmal getestet:

- Virtuelle Maschine: Das Archiv wird als neue VM mit einer anderen ID
  wiederhergestellt (nicht ueber die laufende VM!), gestartet und geprueft.
  Danach wird die Test-VM geloescht.
- Datenbank: Ein Testbenutzer wird geloescht, der Dump zurueckgespielt und
  geprueft, ob der Benutzer wieder vorhanden ist.
- Dateien: Eine Datei wird aus der Sicherung zurueckkopiert und geoeffnet.













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
| Virtuelle Maschine        | vzdump    | HDD 4 TB        | taeglich        |
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