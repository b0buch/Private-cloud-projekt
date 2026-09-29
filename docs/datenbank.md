# Die Datenbank von Nextcloud

Untersuchung der MariaDB-Datenbank von Nextcloud mit SQL. Passt zum Modul
Datenbanken / SQL.

## Verbindung

```bash
cd /srv/nextcloud
docker compose exec db sh -c 'mariadb -u nextcloud -p"$MARIADB_PASSWORD" nextcloud'
```

Das Passwort wird aus der Umgebungsvariable im Container gelesen, damit es nicht
von Hand kopiert werden muss.

## Umfang der Datenbank

```sql
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'nextcloud';
```

Ergebnis bei einer neuen Installation: **138 Tabellen**. Alle mit dem Präfix
`oc_` (aus der Zeit, als Nextcloud noch ownCloud hiess).

Interessant: `information_schema` ist eine Systemdatenbank, in der die Datenbank
Informationen über sich selbst speichert. Metadaten lassen sich also mit ganz
normalen SELECT-Abfragen auslesen.

## Wichtige Tabellen

|      Tabelle       |                       Inhalt                      |
|        ---         |       ---              ---              ---       |
| `oc_users`         | Benutzerkonten                                    |
| `oc_groups`        | Gruppen                                           |
| `oc_group_user`    | Verknüpfung Benutzer ↔ Gruppe (c:m)               |
| `oc_preferences`   | Einstellungen je Benutzer, zum Beispiel die Quota |
| `oc_filecache`     | Index aller Dateien und Ordner                    |
| `oc_storages`      | Speicherorte                                      |
| `oc_mimetypes`     | Dateitypen                                        |
| `oc_share`         | Freigaben                                         |
| `oc_group_folders` | Gemeinsame Ordner (App Team folders)              |

## Benutzer

```sql
DESCRIBE oc_users;
```

```
+-------------+--------------+------+-----+---------+
| Field       | Type         | Null | Key | Default |
+-------------+--------------+------+-----+---------+
| uid         | varchar(64)  | NO   | PRI |         |
| displayname | varchar(64)  | YES  |     | NULL    |
| password    | varchar(255) | NO   |     |         |
| uid_lower   | varchar(64)  | YES  | MUL |         |
+-------------+--------------+------+-----+---------+
```

Beobachtungen:

- `uid` ist der Primärschlüssel. Deshalb kann ein Benutzername nur einmal
  vorkommen und lässt sich später nicht ändern.
- `displayname` darf NULL sein. NULL bedeutet "kein Wert vorhanden", nicht
  "leerer Text".
- `uid_lower` hat einen normalen Index (`MUL`). Der Benutzername wird ein
  zweites Mal in Kleinbuchstaben gespeichert, damit die Suche schnell und ohne
  Rücksicht auf Gross- und Kleinschreibung funktioniert. Eine bewusste
  Redundanz zugunsten der Geschwindigkeit.
- In `password` steht kein Passwort, sondern ein Hash.

## Benutzer und Gruppen zusammen

```sql
SELECT u.uid, u.displayname, g.gid
FROM oc_users u
LEFT JOIN oc_group_user g ON u.uid = g.uid
ORDER BY u.uid;
```

`LEFT JOIN` ist hier wichtig: Mit einem normalen `JOIN` würde ein Benutzer ohne
Gruppe komplett aus dem Ergebnis verschwinden.

Anzahl Mitglieder je Gruppe:

```sql
SELECT gid, COUNT(*) AS anzahl_benutzer
FROM oc_group_user
GROUP BY gid;
```

`oc_group_user` hat keinen eigenen ID-Schlüssel. Der Primärschlüssel ist
zusammengesetzt aus `gid` und `uid`. Dadurch kann ein Benutzer nicht zweimal in
derselben Gruppe stehen.

## Quota

```sql
SELECT userid, configkey, configvalue
FROM oc_preferences
WHERE appid = 'files' AND configkey = 'quota';
```

Die Quota steht nicht in `oc_users`, sondern in einer allgemeinen
Einstellungstabelle mit den Spalten Benutzer, App, Schlüssel und Wert. So lassen
sich neue Einstellungen hinzufügen, ohne die Struktur der Datenbank zu ändern
(Entity-Attribute-Value).

## Dateien

```sql
SELECT fileid, path, parent, size FROM oc_filecache LIMIT 10;
```

Die Spalte `parent` verweist auf die `fileid` des übergeordneten Ordners — die
Tabelle hat also einen Fremdschlüssel **auf sich selbst**. So wird eine
Baumstruktur in einer flachen Tabelle abgebildet.

Inhalt eines Ordners über einen Self-Join:

```sql
SELECT k.fileid, k.path, k.size
FROM oc_filecache k
JOIN oc_filecache p ON k.parent = p.fileid
WHERE p.path = 'files/Photos';
```

Die Grösse eines Ordners ist bereits die Summe seines Inhalts. Nextcloud
speichert diesen Wert im Voraus, damit er beim Öffnen nicht jedes Mal neu
berechnet werden muss.

## Auswertung nach Dateityp

```sql
SELECT m.mimetype, COUNT(*) AS anzahl, SUM(f.size) AS gesamtgroesse
FROM oc_filecache f
JOIN oc_mimetypes m ON f.mimetype = m.id
GROUP BY m.mimetype
ORDER BY gesamtgroesse DESC;
```

Ergebnis bei der leeren Installation (29.09.2026, nur Beispieldateien):

|                      mimetype                   | anzahl | gesamtgroesse |
|       ---              ---             ---      |  ---   |      ---      |
| httpd/unix-directory                            |   17   | 161 536 062   |
| application/vnd.oasis.opendocument.presentation |   11   | 16 105 234    |
| application/vnd.oasis.opendocument.text         |   11   | 6 570 308     |
| image/jpeg                                      |   8    | 5 656 313     |
| application/pdf                                 |   3    | 32 472 900    |
| video/mp4                                       |   1    | 3 963 036     |

Diese Abfrage wird nach dem Hochladen der eigenen Fotos und Videos wiederholt,
um den Unterschied zu zeigen.

## Berechtigungen als Bitmaske

Die Spalte `permissions` in `oc_group_folders_groups` ist eine einzige Zahl.
Jedes Bit steht für ein Recht:

| Bit | Wert |   Recht   |
| --- |  --- |    ---    |
| 1   | 1    | lesen     |
| 2   | 2    | ändern    |
| 3   | 4    | erstellen |
| 4   | 8    | löschen   |
| 5   | 16   | teilen    |

Der Wert **31** bedeutet alle Rechte (1 + 2 + 4 + 8 + 16). Wird das Teilen
deaktiviert, steht dort 15.

Entschlüsseln mit dem bitweisen UND-Operator:

```sql
SELECT folder_id, group_id, permissions,
       IF(permissions & 1,  'ja', 'nein') AS lesen,
       IF(permissions & 2,  'ja', 'nein') AS aendern,
       IF(permissions & 4,  'ja', 'nein') AS erstellen,
       IF(permissions & 8,  'ja', 'nein') AS loeschen,
       IF(permissions & 16, 'ja', 'nein') AS teilen
FROM oc_group_folders_groups;
```

So werden fünf Wahrheitswerte in einer einzigen Spalte gespeichert, statt in
fünf einzelnen Spalten.

## Speichergrössen

Quota und Grösse werden in **Byte** gespeichert, nicht als Text wie "10 GB".
Mit Zahlen kann man rechnen und vergleichen, mit Text nicht.

```sql
SELECT mount_point,
       ROUND(quota / 1024 / 1024 / 1024, 2) AS quota_gb,
       ROUND(size / 1024 / 1024, 2) AS belegt_mb
FROM oc_group_folders;
```

Der Wert `-1` bedeutet "unbegrenzt". Statt NULL wird ein Wert verwendet, der
natürlicherweise nicht vorkommen kann.

## Wer hat Zugriff auf welchen gemeinsamen Ordner

```sql
SELECT gu.uid AS benutzer, f.mount_point AS ordner, gfg.permissions
FROM oc_group_user gu
JOIN oc_group_folders_groups gfg ON gu.gid = gfg.group_id
JOIN oc_group_folders f ON gfg.folder_id = f.folder_id
ORDER BY gu.uid;
```

Drei Tabellen in einer Abfrage. Das Ergebnis zeigt direkt, ob die Berechtigungen
richtig gesetzt sind — zuverlässiger als das Durchklicken der Weboberfläche.

## Wartung der Datenbank

Nextcloud meldete in den Administrationseinstellungen vier fehlgeschlagene
Prüfungen der Datenbank. Behoben mit:

```bash
docker compose exec -u www-data app php occ db:add-missing-indices
docker compose exec -u www-data app php occ db:add-missing-columns
docker compose exec -u www-data app php occ db:add-missing-primary-keys
```

Ein Index kostet Speicherplatz und macht das Schreiben etwas langsamer, dafür
wird das Lesen deutlich schneller. Bei wenigen Dateien fällt das nicht auf, bei
zehntausenden Videos sehr wohl.
