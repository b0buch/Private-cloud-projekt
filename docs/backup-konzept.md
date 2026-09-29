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