# Tests
 
Stand: 06.10.2026
 
## Erledigt
 
### System und Speicher
 
- [x] **Automatischer Start der Container nach einem Neustart.**
      Die Festplatte wird über `/etc/fstab` eingebunden, die Container starten
      durch `restart: unless-stopped` von selbst. (29.09.2026)
- [x] **Verhalten beim Ausfall der Datenfestplatte.**
      Nextcloud meldet den Fehler und verweigert den Zugriff (HTTP 503), ohne
      Daten zu beschädigen. Nach dem Wiederanschliessen funktioniert alles
      normal. (29.09.2026)
- [x] **Maximale Uploadgrösse.** Limit auf 16 GB gesetzt, mit einer 6 GB grossen
      Datei getestet: rund 1 Minute, etwa 100 MB/s. (30.09.2026)
- [x] **Vorschaubilder für Videos.** Funktionieren mit ffmpeg im eigenen Image.
      (30.09.2026, nach dem Neuaufbau bestätigt am 01.10.2026)
- [x] Die Abfrage mit GROUP BY über `oc_mimetypes` wiederholen, nachdem Fotos
      und Videos hochgeladen wurden, und mit dem Ergebnis vom 29.09.2026
      vergleichen.
- [x] **Selbstprüfungen nach dem Umzug.** Alle sechs netzwerkbezogenen
      Prüfungen sind verschwunden, nachdem der DNS-Server der VM von 1.1.1.1
      auf die FRITZ!Box umgestellt wurde. (05.10.2026)
- [x] **Neustart von WireGuard nach einem Neustart der VM.** Prüfen, dass
      `wg-quick@wg0` nach Docker startet und die Regeln in `DOCKER-USER`
      oben stehen.
### Sicherungen
 
- [x] **Backup der VM mit vzdump.**
      Erstellt, 4.39 GiB, Modus Snapshot, Komprimierung ZSTD. (29.09.2026)
- [x] **Wiederherstellung der VM aus dem vzdump-Archiv.**
      Als neue VM mit der ID 999 wiederhergestellt, gestartet, geprüft und
      danach gelöscht. (29.09.2026, wiederholt vom neuen Ziel am 02.10.2026)
- [x] **Backup und Wiederherstellung der Datenbank** mit `mysqldump`. Ein
      Backup, das nie zurückgespielt wurde, ist kein Backup. (29.09.2026)
- [x] **Sicherung der Daten mit rsync.** Erster Durchgang: 93 Dateien,
      1.5 GiB. Zweiter Durchgang unmittelbar danach: 0 Dateien übertragen,
      7 598 Byte gesendet. rsync vergleicht Grösse und Änderungszeit und
      überspringt unveränderte Dateien. (05.10.2026, Testdateien)
- [x] **Erste automatische Sicherungen.** In der Nacht auf den 06.10.2026 lief
      der Server durch: vzdump um 00:00 (40 Sekunden, 5.27 GB), Datenbank um
      01:30. Zum ersten Mal ohne Zutun. (06.10.2026)
- [x] **Nachholen einer verpassten Sicherung.** Zeitstempel des Timers um zwei
      Tage zurückgesetzt, VM neu gestartet. Die Sicherung startete direkt nach
      dem Start, wartete etwa elf Sekunden auf die Container und war nach
      zwölf Sekunden fertig. Neue Datei mit `-rw-------`. (06.10.2026)
- [x] **E-Mail bei einem fehlgeschlagenen Dienst.** Mit
      `systemd-run --unit=test-fehler -p OnFailure=... /bin/false` einen
      Fehler erzeugt. Die E-Mail mit Status und Protokoll kam an. (06.10.2026)
- [x] **Hintergrundaufgaben über den systemd-Timer.** `occ setupchecks`
      meldet "Last background job execution ran seconds ago". (06.10.2026)
### Sicherheit und Anmeldung
 
- [x] **Brute-Force-Schutz.** Die richtige Adresse des Clients wird erkannt,
      damit ist `trusted_proxies` bestätigt. (01.10.2026)
- [x] **Zwei-Faktor-Authentisierung.** Eingerichtet und getestet. (01.10.2026)
- [x] **Zwei-Faktor-Authentisierung mit der Nextcloud-App.** Die Annahme, die
      App umgehe 2FA über App-Passwörter, war falsch: Die App verwendet die
      browserbasierte Anmeldung (Login Flow v2), und dabei wird der zweite
      Faktor verlangt. (05.10.2026)
### Netzwerk und VPN
 
- [x] **Zugriff mit dem Smartphone im lokalen Netzwerk.** (02.10.2026)
- [x] **Router-VPN über das Mobilfunknetz.** Zugriff auf Proxmox und Nextcloud
      per IP und per Name, WLAN ausgeschaltet. (04.10.2026)
- [x] **Nextcloud ist ohne VPN nicht erreichbar.** Über Mobilfunk ohne aktives
      VPN keine Verbindung. Es existiert keine Portweiterleitung auf 443.
      (04.10.2026)
- [x] **Server-VPN über das Mobilfunknetz.** Smartphone über Mobilfunk und
      Laptop über die Mobilfunk-Freigabe, Nextcloud über
      `https://192.168.178.220` erreichbar. (06.10.2026)
- [x] **Namen im Server-VPN.** Nach der Einrichtung von dnsmasq ist
      `https://privatecloud.fritz.box` auch über das Server-VPN erreichbar.
      Kontrolle mit `nslookup`: Antwort von `10.9.0.1`. (06.10.2026)
- [x] **Server-VPN erreicht Proxmox nicht.** Über den Tunnel des Laptops ist
      die Weboberfläche von Proxmox nicht erreichbar, wie gewollt.
      (06.10.2026)
### Klärungen
 
- [x] Ist die zusätzliche IPv4-Adresse bei InterXS monatlich kündbar und wie
      lange dauert die Aktivierung? (nicht mehr nötig, öffentliche IPv4
      vorhanden)
- [x] Welcher Router hängt zu Hause direkt am Anschluss? FRITZ!Box, kein
      doppeltes NAT.
- [x] Ändert die zusätzliche IPv4-Adresse etwas an der Telefonie im Haushalt?
      (entfällt)
## Offen
 
- [ ] **Vollständige Zugriffsmatrix des Server-VPN.** Über den Tunnel eines
      Benutzers dürfen nicht erreichbar sein: SSH auf die VM
      (`ssh <benutzer>@192.168.178.220`), Weboberfläche der FRITZ!Box,
      andere Geräte im Heimnetz.
- [ ] **Alle Geräte der Benutzer verbinden** und jeweils mit ausgeschaltetem
      WLAN prüfen.
- [ ] **Realistischer Test der Benachrichtigung.** Die Sicherung der Daten
      ohne angeschlossene externe Festplatte starten. Erwartet: Abbruch an der
      Prüfung mit `findmnt` und eine E-Mail.
- [ ] **Wiederherstellung einzelner Dateien aus der rsync-Sicherung.**
- [ ] **Automatischen Upload von Fotos** auf allen drei Smartphones testen,
      besonders im Hintergrund über mehrere Stunden (Akku-Optimierung).
- [ ] **Upload eines grossen Videos über VPN** (Zeit messen, der Anschluss hat
      50 Mbit/s im Upload).
