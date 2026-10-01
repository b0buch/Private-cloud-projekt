# Checkliste: Umzug des Servers nach Hause

Erstellt: 28.09.2026

---

## Teil A — Vorbereitung (ohne Server, zu Hause am Abend)

### A1. Anschluss und Router kontrollieren

- [ ] Welches Gerät hängt direkt an der Internetleitung? (eigene FRITZ!Box 7490
      oder ein Gerät vom Anbieter)
- [ ] Wenn zwei Router hintereinander: Double NAT vermeiden. Ein Gerät muss als
      Modem / Bridge arbeiten.
- [ ] FRITZ!OS-Version notieren (System → Update). Für WireGuard nötig:
      mindestens 7.50
- [ ] Kontrollieren, ob der Menüpunkt Internet → Freigaben → VPN (WireGuard)
      vorhanden ist
- [ ] Unter Internet → Online-Monitor die öffentliche IP ansehen. Eine Adresse
      aus `100.64.x.x` bis `100.127.x.x` bedeutet Carrier-Grade-NAT.

### A2. Öffentliche IPv4-Adresse bestellen

- [ ] Bei InterXS die zusätzliche statische IPv4-Adresse bestellen (5 CHF/Monat)
- [ ] Fragen: Ist die Option monatlich kündbar?
- [ ] Fragen: Wie lange dauert die Aktivierung?
- [ ] Datum der Bestellung notieren: `________________`

### A3. Netzwerk vorbereiten

- [ ] DynDNS oder MyFRITZ einrichten (Internet → Freigaben → DynDNS bzw.
      MyFRITZ!-Konto)
- [ ] Name notieren: `________________________________`
- [ ] Feste IP-Adresse für den Server festlegen: `192.168.178.____`
- [ ] Aktuellen Adressbereich der FRITZ!Box notieren: `____________________`

### A4. Smartphones vorbereiten (alle drei Geräte)

- [ ] Nextcloud-App installieren
- [ ] WireGuard-App installieren
- [ ] Akku-Optimierung für die Nextcloud-App ausschalten
      (Xiaomi: zusätzlich Autostart in den MIUI-Einstellungen erlauben)
- [ ] Freien Speicherplatz auf den Geräten prüfen

### A5. Benutzerkonzept festlegen

- [ ] Benutzer 1: `____________________________`
- [ ] Benutzer 2: `____________________________`
- [ ] Benutzer 3: `____________________________`
- [ ] Gibt es einen gemeinsamen Ordner für alle?
- [ ] Welche Quota pro Benutzer?
- [ ] Wer darf Dateien mit anderen teilen?

---

## Teil B — Aufbau zu Hause (mit Server)

### B1. Hardware anschliessen

- [ ] Server aufstellen (Belüftung beachten, nicht im Schrank)
- [ ] Netzwerkkabel zur FRITZ!Box anschliessen
- [ ] Server einschalten und im Netzwerk suchen

### B2. Netzwerk von Proxmox anpassen

> **Wichtig:** Die aktuelle Konfiguration ist für die Internetfreigabe des
> Laptops eingerichtet (`192.168.137.x`). Zu Hause gilt ein anderer
> Adressbereich (`192.168.178.x`).

- [ ] Vorher eine Kopie der Datei erstellen:
      `cp /etc/network/interfaces /etc/network/interfaces.backup`
- [ ] `/etc/network/interfaces` auf dem Proxmox-Host anpassen
- [ ] Netzwerkdienst neu starten oder Server neu starten
- [ ] Proxmox-Weboberfläche vom PC aus öffnen
- [ ] Kontrollieren, ob die VM startet und Internet hat
- [ ] Auch die feste Adresse der VM in `/etc/netplan/` anpassen

### B3. Feste IP-Adressen vergeben

- [ ] In der FRITZ!Box unter Heimnetz → Netzwerk den Server suchen
- [ ] Option aktivieren: "Diesem Netzwerkgerät immer die gleiche IPv4-Adresse
      zuweisen"
- [ ] Dasselbe für die Ubuntu-VM
- [ ] Adressen notieren — Proxmox: `____________` VM: `____________`

### B4. Nextcloud auf die neuen Adressen umstellen

```bash
cd /srv/nextcloud

docker compose exec -u www-data app php occ \
  config:system:set trusted_domains 0 --value=192.168.178.XX

docker compose exec -u www-data app php occ \
  config:system:set overwrite.cli.url --value=https://192.168.178.XX

docker compose exec -u www-data app php occ \
  config:system:set overwriteprotocol --value=https
```

- [ ] Befehle ausführen
- [ ] In der Datei `.env` den Wert `TRUSTED_DOMAINS` anpassen
- [ ] Container neu starten: `docker compose up -d`
- [ ] Nextcloud im Browser öffnen und Anmeldung testen

### B5. Selbstprüfungen kontrollieren

- [ ] Administrationseinstellungen → Übersicht öffnen
- [ ] Kontrollieren, ob die Fehler verschwunden sind, die vorher wegen der
      Internetfreigabe entstanden sind (WebDAV, `.well-known`, OCS,
      Schriftarten, Datenverzeichnis)
- [ ] Verbleibende Warnungen im Journal notieren

### B6. Zertifikat neu ausstellen

> Das aktuelle Zertifikat gilt nur für die Adresse am Arbeitsplatz. Mit einer
> neuen Adresse oder einem lokalen Namen meldet der Browser sonst einen Fehler.

- [ ] Neues Zertifikat mit der neuen Adresse im Feld `subjectAltName` erstellen
- [ ] Caddy neu starten: `docker compose restart caddy`
- [ ] Das neue Zertifikat auf allen Geräten als vertrauenswürdig installieren
- [ ] Prüfen, dass der Browser keine Warnung mehr anzeigt

---

## Teil C — VPN einrichten

### C1. Voraussetzungen prüfen

- [ ] Ist die öffentliche IPv4-Adresse aktiv? Kontrolle unter Internet →
      Online-Monitor: Die Adresse darf nicht mehr aus dem Bereich `100.64.x.x`
      stammen.
- [ ] Kontrolle von aussen: Die Adresse über das Mobilfunknetz (ohne WLAN)
      prüfen

### C2. WireGuard auf der FRITZ!Box einrichten

- [ ] Internet → Freigaben → VPN (WireGuard) → Verbindung hinzufügen
- [ ] Typ: **Einzelnes Gerät** — für jedes Smartphone eine eigene Verbindung,
      damit einzelne Schlüssel gesperrt werden können
- [ ] Name der Verbindung sprechend wählen
- [ ] QR-Code mit der WireGuard-App auf dem Handy scannen
- [ ] Konfigurationsdatei sicher aufbewahren

### C3. VPN testen

- [ ] Test 1: Verbindung im eigenen WLAN aufbauen (sagt wenig aus)
- [ ] Test 2: WLAN am Handy ausschalten und über das Mobilfunknetz verbinden —
      **das ist der richtige Test**
- [ ] Über VPN die Nextcloud-Adresse im Browser öffnen
- [ ] Über VPN in der Nextcloud-App anmelden
- [ ] Ein Foto hochladen und die Geschwindigkeit messen
- [ ] Verbindung für alle drei Benutzer einrichten und testen

### C4. Sicherheit

- [ ] Zwei-Faktor-Authentisierung für die beiden neuen Benutzer einrichten
      (für das eigene Konto am 01.10.2026 bereits erledigt)
- [ ] Kontrollieren, dass Nextcloud **nicht** direkt aus dem Internet erreichbar
      ist (keine Portweiterleitung auf Port 443)
- [ ] Starke Passwörter für alle drei Benutzer
- [ ] Brute-Force-Schutz von Nextcloud kontrollieren

---

## Teil D — Nach dem Projekt

- [ ] Tailscale auf dem Server installieren und testen
- [ ] Zugriff über Tailscale von aussen prüfen
- [ ] **Erst danach** die statische IPv4-Adresse bei InterXS kündigen
- [ ] Vergleich der beiden Lösungen in der Dokumentation festhalten (Kosten,
      Aufwand, Sicherheit, Geschwindigkeit)

---

## Notizen

**Aktuelle Werte (Arbeitsplatz):**

|               |                                                                      |
|      ---      |        ---            ---             ---                            |
| Projektordner | `/srv/nextcloud`                                                     |
| Datenordner   | `/mnt/clouddata`                                                     |
| Container     |`nextcloud-caddy`, `nextcloud-app`, `nextcloud-db`, `nextcloud-redis` |
| Port          |443 (HTTPS), 80 leitet weiter                                         |

**Neue Werte (zu Hause):**

|             |                          |
|     ---     |            ---           |
| Proxmox     | `______________________` |
| Ubuntu-VM   | `______________________` |
| Nextcloud   | `______________________` |
| DynDNS-Name | `______________________` |
