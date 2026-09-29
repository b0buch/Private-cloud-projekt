# Tests

## Erledigt

- [x] **Automatischer Start der Container nach einem Neustart.**
      Funktioniert. Die externe Festplatte wird über `/etc/fstab` eingebunden,
      die Container starten durch `restart: unless-stopped` von selbst.
      (29.09.2026)

- [x] **Verhalten beim Ausfall der Datenfestplatte.**
      Nextcloud meldet den Fehler und verweigert den Zugriff (HTTP 503), ohne
      Daten zu beschädigen. Nach dem Wiederanschliessen funktioniert alles
      normal. (29.09.2026)

- [x] **Backup der VM mit vzdump.**
      Erstellt, 4.39 GiB, Modus Snapshot, Komprimierung ZSTD. (29.09.2026)
     
- [x] **Wiederherstellung der VM aus dem vzdump-Archiv.**
      Als neue VM mit der ID 999 wiederhergestellt, gestartet, geprueft und
      danach geloescht. Funktioniert. (29.09.2026)

- [x] Ein Backup der Datenbank mit `mysqldump` erstellen und die
      **Wiederherstellung** testen. Ein Backup, das nie zurückgespielt wurde,
      ist kein Backup.

## Offen — kann am Arbeitsplatz gemacht werden

- [ ] Testen, wie gross eine hochgeladene Videodatei maximal sein darf
      (Einstellungen von PHP).
- [ ] Prüfen, ob der Cronjob wirklich alle fünf Minuten läuft (Zeitpunkt der
      letzten Ausführung in den Administrationseinstellungen).
- [ ] Die Abfrage mit GROUP BY über `oc_mimetypes` wiederholen, nachdem Fotos
      und Videos hochgeladen wurden, und mit dem Ergebnis vom 29.09.2026
      vergleichen.
- [ ] Prüfen, ob Vorschaubilder für Videos funktionieren (braucht ffmpeg).

## Offen — erst nach dem Umzug nach Hause

- [ ] Zugriff mit dem Smartphone im lokalen Netzwerk testen.
- [ ] Automatischen Upload von Fotos auf allen drei Smartphones testen,
      besonders im Hintergrund über mehrere Stunden (Akku-Optimierung).
- [ ] Prüfen, ob die fehlgeschlagenen Selbstprüfungen von Nextcloud nach dem
      Umzug an die FRITZ!Box verschwinden.
- [ ] VPN über das Mobilfunknetz testen, nicht nur im eigenen WLAN.
- [ ] Prüfen, dass Nextcloud **nicht** ohne VPN aus dem Internet erreichbar ist.
- [ ] Den Upload eines grossen Videos über VPN testen (Zeit messen, der
      Anschluss hat 50 Mbit/s im Upload).

## Abzuklären

- [ ] Ist die zusätzliche IPv4-Adresse bei InterXS monatlich kündbar und wie
      lange dauert die Aktivierung?
- [ ] Welcher Router hängt zu Hause direkt am Anschluss? Bei zwei Routern
      hintereinander (Double NAT) funktioniert WireGuard nicht.
- [ ] Ändert die zusätzliche IPv4-Adresse etwas an der Telefonie im Haushalt?
