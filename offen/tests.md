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

- [x] Die Abfrage mit GROUP BY über `oc_mimetypes` wiederholen, nachdem Fotos
      und Videos hochgeladen wurden, und mit dem Ergebnis vom 29.09.2026
      vergleichen.

- [x] **Brute-Force-Schutz.** Getestet, funktioniert. Die richtige Adresse des
      Clients wird erkannt, damit ist trusted_proxies bestaetigt. (01.10.2026)

- [x] **Zwei-Faktor-Authentisierung.** Eingerichtet und getestet. (01.10.2026)

- [x] **Maximale Uploadgrösse.** Limit auf 16 GB gesetzt, mit einer 6 GB grossen
      Datei getestet: rund 1 Minute, etwa 100 MB/s. (30.09.2026)

- [x] **Vorschaubilder für Videos.** Funktionieren mit ffmpeg im eigenen Image.
      (30.09.2026, nach dem Neuaufbau bestätigt am 01.10.2026)

- [x] **Zwei-Faktor-Authentisierung mit der Nextcloud-App.** Getestet am
      05.10.2026 ueber VPN. Die Annahme, die App umgehe 2FA ueber
      App-Passwoerter, war falsch: Die App verwendet die browserbasierte
      Anmeldung (Login Flow v2), und dabei wird der zweite Faktor verlangt.

- [x] Ist die zusätzliche IPv4-Adresse bei InterXS monatlich kündbar und wie
      lange dauert die Aktivierung?
- [x] Welcher Router hängt zu Hause direkt am Anschluss? Bei zwei Routern
      hintereinander (Double NAT) funktioniert WireGuard nicht.
- [x] Ändert die zusätzliche IPv4-Adresse etwas an der Telefonie im Haushalt?

- [x] **Zugriff mit dem Smartphone im lokalen Netzwerk.** (02.10.2026)
- [x] **Selbstpruefungen nach dem Umzug.** Alle sechs netzwerkbezogenen
      Pruefungen sind verschwunden, nachdem der DNS-Server der VM von 1.1.1.1
      auf die FRITZ!Box umgestellt wurde. (05.10.2026)
- [x] **VPN ueber das Mobilfunknetz.** Zugriff auf Proxmox und Nextcloud per
      IP und per Name, WLAN ausgeschaltet. (04.10.2026)
- [x] **Nextcloud ist ohne VPN nicht erreichbar.** Ueber Mobilfunk ohne
      aktives VPN keine Verbindung. Es existiert keine Portweiterleitung auf
      443. (04.10.2026)
- [x] **Cronjob laeuft alle fuenf Minuten.** Bestaetigt ueber
      `occ config:app:get core lastcron` und die Selbstpruefung
      "Cron last run". (05.10.2026)
- [x] **Sicherung der Daten mit rsync.** Erster Durchgang: 93 Dateien,
      1.5 GiB. Zweiter Durchgang unmittelbar danach: 0 Dateien uebertragen,
      7 598 Byte gesendet. rsync vergleicht Groesse und Aenderungszeit und
      ueberspringt unveraenderte Dateien — eine taegliche Sicherung kostet
      deshalb kaum etwas. (05.10.2026, Testdateien)

## Offen — erst nach dem Umzug nach Hause

- [ ] Automatischen Upload von Fotos auf allen drei Smartphones testen,
      besonders im Hintergrund über mehrere Stunden (Akku-Optimierung).
- [ ] Den Upload eines grossen Videos über VPN testen (Zeit messen, der
      Anschluss hat 50 Mbit/s im Upload).
