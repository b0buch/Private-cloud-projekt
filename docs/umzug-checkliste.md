# Checkliste: Umzug des Servers nach Hause
 
Erstellt: 28.09.2026
 
---
 
**Durchgefuehrt am 02.10.2026.** Die Abschnitte bleiben als Nachweis des
Ablaufs erhalten. Korrekturen und tatsaechliche Werte sind unten eingetragen.
 
## Teil A — Vorbereitung (ohne Server, zu Hause am Abend)
 
### A1. Anschluss und Router kontrollieren
 
- [x] Welches Gerät hängt direkt an der Internetleitung? (eigene FRITZ!Box 7490
      oder ein Gerät vom Anbieter)
- [x] Wenn zwei Router hintereinander: Double NAT vermeiden. Ein Gerät muss als
      Modem / Bridge arbeiten.
- [x] FRITZ!OS-Version notieren (System → Update). Für WireGuard nötig:
      mindestens 7.50
- [x] Kontrollieren, ob der Menüpunkt Internet → Freigaben → VPN (WireGuard)
      vorhanden ist
- [x] Unter Internet → Online-Monitor die öffentliche IP ansehen. Eine Adresse
      aus `100.64.x.x` bis `100.127.x.x` bedeutet Carrier-Grade-NAT.
Ergebnis (04.10.2026): FRITZ!Box direkt am Anschluss, FRITZ!OS 7.62, kein
doppeltes NAT, öffentliche IPv4-Adresse.
 
### A2. Öffentliche IPv4-Adresse bestellen
 
Nicht notwendig. Am 04.10.2026 wurde festgestellt, dass der Anschluss bereits
eine oeffentliche IPv4-Adresse hat. Die Annahme von Carrier-Grade-NAT war
falsch. Die wechselnde Adresse wird ueber MyFRITZ! als Name erreichbar
gemacht, Kosten entstehen keine.
 
### A3. Netzwerk vorbereiten
 
- [x] MyFRITZ einrichten (Internet → MyFRITZ!-Konto)
- [x] Name notieren: `<name>.myfritz.net` (in der Dokumentation Platzhalter)
- [x] Feste IP-Adresse für den Server festlegen: `192.168.178.220`
- [x] Aktuellen Adressbereich der FRITZ!Box notieren: `192.168.178.0/24`
### A4. Smartphones vorbereiten (alle sechs Geräte)
 
- [ ] Nextcloud-App installieren
- [ ] WireGuard-App installieren
- [ ] Freien Speicherplatz auf den Geräten prüfen
### A5. Benutzerkonzept festlegen
 
- [x] Benutzer festgelegt (05.10.2026, Namen nicht in der Dokumentation)
- [x] Gemeinsame Ordner: `Foto-Video` (500 GB) und `Ski` (50 GB)
- [x] Quota pro Benutzer: 100 GB für das Alltagskonto
- [x] Nur der Ordner `Foto-Video` wird gesichert (siehe `backup-konzept.md`)
---
 
## Teil B — Aufbau zu Hause (mit Server)
 
### B1. Hardware anschliessen
 
- [x] Server aufstellen (Belüftung beachten, nicht im Schrank)
- [x] Netzwerkkabel zur FRITZ!Box anschliessen
- [x] Server einschalten und im Netzwerk suchen
### B2. Netzwerk von Proxmox anpassen
 
> **Wichtig:** Die aktuelle Konfiguration ist für die Internetfreigabe des
> Laptops eingerichtet (`192.168.137.x`). Zu Hause gilt ein anderer
> Adressbereich (`192.168.178.x`).
 
- [x] Vorher eine Kopie der Datei erstellen:
      `cp /etc/network/interfaces /etc/network/interfaces.backup`
- [x] `/etc/network/interfaces` auf dem Proxmox-Host anpassen
- [x] Netzwerkdienst neu starten oder Server neu starten
- [x] Proxmox-Weboberfläche vom PC aus öffnen
- [x] Kontrollieren, ob die VM startet und Internet hat
- [x] Auch die Adresse der VM in `/etc/netplan/` anpassen (jetzt DHCP)
### B3. Feste IP-Adressen vergeben
 
- [x] In der FRITZ!Box unter Heimnetz → Netzwerk den Server suchen
- [x] Option aktivieren: "Diesem Netzwerkgerät immer die gleiche IPv4-Adresse
      zuweisen"
- [x] Dasselbe für die Ubuntu-VM
- [x] Adressen notieren — Proxmox: `192.168.178.100` VM: `192.168.178.220`
### B4. Nextcloud auf die neuen Adressen umstellen
 
```bash
cd /srv/nextcloud
 
docker compose exec -u www-data app php occ \
  config:system:set trusted_domains 0 --value=192.168.178.XX
 
docker compose exec -u www-data app php occ \
  config:system:set overwrite.cli.url --value=https://privatecloud.fritz.box
 
docker compose exec -u www-data app php occ \
  config:system:set overwriteprotocol --value=https
```
 
- [x] Befehle ausführen
- [x] In der Datei `.env` den Wert `TRUSTED_DOMAINS` anpassen
- [x] Container neu starten: `docker compose up -d`
- [x] Nextcloud im Browser öffnen und Anmeldung testen
- [x] Alte Einträge `192.168.137.*` aus `trusted_domains` entfernen
### B5. Selbstprüfungen kontrollieren
 
- [x] Administrationseinstellungen → Übersicht öffnen
- [x] Kontrollieren, ob die Fehler verschwunden sind, die vorher wegen der
      Internetfreigabe entstanden sind (WebDAV, `.well-known`, OCS,
      Schriftarten, Datenverzeichnis) — verschwunden am 05.10.2026, nachdem
      der DNS-Server der VM auf die FRITZ!Box umgestellt wurde
- [x] Verbleibende Warnungen im Journal notieren
### B6. Zertifikat neu ausstellen
 
Das Zertifikat wurde am 02.10.2026 neu ausgestellt, diesmal auf den **Namen**
`privatecloud.fritz.box` statt auf eine IP-Adresse. Damit ist es von der
Adresse unabhängig.
 
Am 06.10.2026 kam für das Server-VPN zusätzlich `IP:192.168.178.220` dazu. Bei
einem künftigen Wechsel der Adresse muss nur diese Angabe angepasst werden,
der Zugriff über den Namen funktioniert weiter.
 
---
 
## Teil C — VPN einrichten
 
### C1. Voraussetzungen prüfen
 
- [x] Ist die öffentliche IPv4-Adresse aktiv? Kontrolle unter Internet →
      Online-Monitor: Die Adresse darf nicht aus dem Bereich `100.64.x.x`
      stammen.
- [x] Kontrolle von aussen: Die Adresse über das Mobilfunknetz (ohne WLAN)
      prüfen
### C2. WireGuard auf der FRITZ!Box einrichten
 
- [x] Internet → Freigaben → VPN (WireGuard) → Verbindung hinzufügen
- [x] Typ: **Einzelnes Gerät** — für jedes Gerät eine eigene Verbindung,
      damit einzelne Schlüssel gesperrt werden können
- [x] Name der Verbindung sprechend wählen
- [x] QR-Code mit der WireGuard-App auf dem Handy scannen
- [x] Konfigurationsdatei sicher aufbewahren
Das Router-VPN wird seit dem 06.10.2026 nur noch für die eigene Verwaltung
verwendet. Die Benutzer erhalten Zugang über das Server-VPN (C5).
 
### C3. VPN testen
 
- [x] Test 1: Verbindung im eigenen WLAN aufbauen (sagt wenig aus)
- [x] Test 2: WLAN am Handy ausschalten und über das Mobilfunknetz verbinden —
      **das ist der richtige Test**
- [x] Über VPN die Nextcloud-Adresse im Browser öffnen
- [x] Über VPN in der Nextcloud-App anmelden
- [x] Ein Foto hochladen und die Geschwindigkeit messen
- [ ] Verbindung für alle drei Benutzer einrichten und testen
      (zwei von drei erledigt am 06.10.2026)
### C4. Sicherheit
 
- [x] Zwei-Faktor-Authentisierung für die Gruppe `admin` verpflichtend
- [x] Kontrollieren, dass Nextcloud **nicht** direkt aus dem Internet erreichbar
      ist (keine Portweiterleitung auf Port 443)
- [x] Starke Passwörter für alle drei Benutzer
- [x] Brute-Force-Schutz von Nextcloud kontrollieren
### C5. Server-VPN für die Benutzer (06.10.2026)
 
- [x] Portfreigabe UDP 51821 auf die VM in der FRITZ!Box (Bestätigung am Gerät)
- [x] WireGuard in der VM, `wg0`, `10.9.0.1/24`
- [x] Regeln in `DOCKER-USER`: nur TCP 443 aus `wg0`
- [x] ufw: Zugriff auf die VM selbst über `wg0` gesperrt, ausser DNS
- [x] dnsmasq für die Namen im Tunnel
- [x] Zertifikat um die IP-Adresse ergänzt
- [x] Sechs Teilnehmer vorbereitet (drei Smartphones, drei Laptops)
- [ ] Alle Geräte verbunden und die Schlüsseldateien danach gelöscht
---
 
## Was beim Umzug tatsaechlich schiefging
 
**netplan: `set-name` wirkt nicht im Betrieb.** Die Umstellung auf DHCP wurde
mit `netplan try` versucht. Die Verbindung brach ab, weil die Maschine eine
andere Adresse erhielt — und da die Bestaetigung nicht moeglich war, wurde
nach 120 Sekunden alles zurueckgesetzt. Richtig ist: Netzwerkaenderungen an
einer VM ueber die Konsole von Proxmox vornehmen, nicht ueber SSH.
 
**Die Internetfreigabe von Windows antwortete nicht mehr auf DHCP.** Deshalb
wurde die DHCP-Konfiguration nur in die Datei geschrieben und nicht
angewendet. Zu Hause an der FRITZ!Box griff sie beim ersten Start sofort.
 
**Docker startete vor dem Einbinden der Datenfestplatte.** Siehe
`probleme.md`.
 
## Notizen
 
**Werte am Arbeitsplatz (bis 02.10.2026):**
 
|               |                                                                      |
|      ---      |                                ---                                   |
| Projektordner | `/srv/nextcloud`                                                     |
| Datenordner   | `/mnt/clouddata`                                                     |
| Container     | `nextcloud-caddy`, `nextcloud-app`, `nextcloud-db`, `nextcloud-redis`|
| Port          | 443 (HTTPS), 80 leitet weiter                                        |
 
**Werte zu Hause:**
 
|             |                                    |
|     ---     |               ---                  |
| Proxmox     | `192.168.178.100`                  |
| Ubuntu-VM   | `192.168.178.220`                  |
| Nextcloud   | `https://privatecloud.fritz.box`   |
| MyFRITZ!    | `<name>.myfritz.net` (Platzhalter) |
| Server-VPN  | `10.9.0.0/24`, UDP 51821           |