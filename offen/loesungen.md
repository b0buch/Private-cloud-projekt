# Lösungen und Entscheidungen
 
Ideen und Entscheidungen für Punkte, die später umgesetzt werden, und
Begründungen für Abweichungen vom Plan.
 
## Mehrere Festplatten: LVM
 
Wenn später weitere Festplatten dazukommen, werden sie mit LVM zu einer Volume
Group zusammengefasst, damit Nextcloud weiterhin nur **ein** Datenverzeichnis
sieht. Ein zweites Datenverzeichnis ist nicht möglich, weil der Pfad bei der
Installation fest in der Konfiguration steht.
 
```bash
pvcreate /dev/sdX /dev/sdY
vgcreate clouddata-vg /dev/sdX /dev/sdY
lvcreate -l 100%FREE -n clouddata-lv clouddata-vg
```
 
Neue Festplatten können später mit `vgextend`, `lvextend` und `resize2fs` im
laufenden Betrieb hinzugefügt werden.
 
- [x] **Wichtig:** Die 4TB-Festplatte von Anfang an als LVM einrichten, auch
      wenn sie zuerst allein ist. Sonst müssen später alle Daten neu kopiert
      werden. (02.10.2026)
**Nachteil von LVM ohne RAID:** Fällt eine Festplatte aus, ist das ganze
Dateisystem betroffen, nicht nur der Teil auf dieser Festplatte. Ein Backup ist
deshalb Pflicht.
 
## Alternative: External Storage
 
Die App "External Storage" in Nextcloud kann eine zweite Festplatte einzelnen
Benutzern als zusätzlichen Ordner zuweisen. Die Quota von Nextcloud gilt dort
aber nicht, und für den Benutzer ist es ein separater Ordner statt einfach mehr
Platz.
 
## Quota
 
Die Quota pro Benutzer ist eine **logische** Grenze in Nextcloud und unabhängig
von den physischen Festplatten. Man kann drei Benutzern je 1 TB auf einer
4TB-Festplatte geben, solange die tatsächlich belegte Menge nicht überschritten
wird.
 
Gemeinsame Ordner (Team folders) haben eine eigene Quota. Eine Datei in einem
gemeinsamen Ordner zählt zur Quota des Ordners, nicht zu der des Benutzers, der
sie hochgeladen hat.
 
## Backup der Datenbank
 
Ein Skript, das über einen systemd-Timer läuft (bis 05.10.2026 cron):
 
1. Nextcloud in den Wartungsmodus setzen (`occ maintenance:mode --on`)
2. Datenbank mit `mysqldump` sichern
3. Wartungsmodus wieder ausschalten (`trap ... EXIT`, auch bei Abbruch)
4. Alte Sicherungen nach einer festgelegten Zeit löschen
Der Wartungsmodus ist nötig, damit Dateien und Datenbank zueinander passen.
Ohne ihn könnte während des Dumps eine Datei hochgeladen werden, die dann in der
Sicherung fehlt.

## Kein automatischer Upload

Die Nextcloud-App kann neue Fotos selbstständig hochladen. Die Funktion wird
bewusst nicht verwendet:

- Der Server läuft nicht rund um die Uhr, ein Upload im Hintergrund würde
  oft ins Leere laufen.
- Grosse Videos über das Mobilfunknetz brauchen viel Zeit (1.5 GB in rund
  10 Minuten) und belasten das Datenvolumen.
- Die Akku-Optimierung von Android stoppt Hintergrund-Uploads unzuverlässig.

Fotos und Videos werden deshalb von Hand hochgeladen, grosse Dateien im WLAN
zu Hause.
 
## Ablauf beim Wechsel auf die 4TB-Festplatte
 
Geplante Reihenfolge:
 
1. pvcreate auf der neuen Festplatte
2. vgextend: neue Festplatte zur Volume Group hinzufügen
3. pvmove: Daten von der alten auf die neue Festplatte verschieben
4. vgreduce und pvremove: alte Festplatte aus der Gruppe entfernen
5. lvextend und resize2fs: erst jetzt das Volume auf den ganzen Speicher
   erweitern
6. Die freie 1TB-Festplatte neu formatieren und als Sicherungsziel einrichten
**Abweichung vom geplanten Ablauf (02.10.2026).** pvmove wurde bewusst nicht
verwendet. Der Befehl verschiebt alle zugewiesenen Extents, nicht die belegten
Daten: 931.51 GiB waren zugewiesen, aber nur 7.8 GB belegt. Auf Festplatten mit
5400 U/min hätte das drei bis vier Stunden gedauert. Stattdessen wurden die
Daten mit rsync kopiert (1 Minute 9 Sekunden), wobei zu keinem Zeitpunkt nur
eine Kopie existierte.
 
## Zweites VPN: Abweichung vom Plan (06.10.2026)
 
Geplant war am 05.10.2026:
 
- Einschränkung mit `ufw route deny from 10.9.0.0/24`
- Zugriff über `https://10.9.0.1`, Zertifikat um `IP:10.9.0.1` erweitert,
  kein Namensdienst im Tunnel
Umgesetzt wurde:
 
- Regeln in der Kette `DOCKER-USER`, weil der Verkehr zu Nextcloud per DNAT
  durch FORWARD läuft und ufw ihn nicht sieht
- Zugriff über `192.168.178.220` bzw. den Namen, Zertifikat um
  `IP:192.168.178.220` erweitert
- dnsmasq als Namensdienst nur für den Tunnel, damit die Benutzer dieselbe
  Adresse verwenden wie zu Hause
Begründung und Einzelheiten in `vpn.md`.
 
## Tailscale
 
Tailscale wurde nicht getestet. Entscheidung vom 05.10.2026: Der Vergleich
bringt für das Projektziel keinen zusätzlichen Nutzen, weil WireGuard die
Anforderung bereits erfüllt und kein Carrier-Grade-NAT vorliegt. Der Vergleich
in `vpn.md` stützt sich auf die Dokumentation beider Lösungen und ist als
solcher gekennzeichnet.
 
## Eigene Domain mit Let's Encrypt
 
Eine eigene Domain (etwa 10 bis 15 CHF im Jahr) mit einem Eintrag
`cloud.<domain>.ch → 192.168.178.220` und einem Zertifikat von Let's Encrypt
über die DNS-Prüfung würde drei Dinge gleichzeitig lösen:
 
- kein Stammzertifikat mehr auf den Geräten der Benutzer
- kein eigener Namensdienst im Server-VPN, die Geräte hängen nicht mehr am
  DNS des Servers
- kein Neuausstellen des Zertifikats von Hand, Let's Encrypt erneuert
  automatisch
Voraussetzung: ein Anbieter mit Schnittstelle für DNS-Einträge, und Caddy mit
dem passenden Modul (eigenes Image nötig). Der Server bleibt dabei weiterhin
nur über VPN erreichbar, weil die Adresse im Eintrag eine private ist.
 
## Nachtbetrieb mit rtcwake (Strom sparen)
 
Seit dem 06.10.2026 holen die systemd-Timer verpasste Sicherungen nach. Der
Nachtbetrieb ist deshalb keine Voraussetzung mehr für die Sicherungen, sondern
nur noch eine Möglichkeit, Strom zu sparen.
 
Der Server muss nicht rund um die Uhr laufen. Bei etwa 45 Watt im Leerlauf sind
das ungefähr 400 kWh im Jahr.
 
### Was nicht geht: einfach die Steckdose ausschalten
 
Eine schaltbare Steckdose trennt den Strom hart, wie ein herausgezogenes Kabel.
Das Betriebssystem kann seine Daten nicht mehr auf die Festplatte schreiben und
die Datenbank bricht mitten in einer Transaktion ab. Einmal geht das meistens
gut, täglich führt es früher oder später zu beschädigten Daten.
 
Ausserdem schliessen sich Steckdose und Fernsteuerung gegenseitig aus: Sowohl
Wake-on-LAN als auch der Weckzeitpunkt im BIOS brauchen Standby-Strom. Dieser
beträgt nur 1 bis 3 Watt, also wenige Franken im Jahr.
 
### Bessere Lösung
 
**Einschalten von unterwegs:** Die FRITZ!Box kann Geräte im Heimnetz per
Wake-on-LAN starten (Heimnetz > Netzwerk > Computer starten). Über die
MyFRITZ!App oder über VPN funktioniert das auch vom Smartphone aus.
 
**Ausschalten von unterwegs:** Mit einer SSH-App auf dem Smartphone den Befehl
`shutdown -h now` auf dem Proxmox-Host ausführen. Proxmox fährt die VM dank des
QEMU Guest Agent sauber herunter.
 
**Automatisch nachts:** Der Befehl `rtcwake` stellt einen Weckzeitpunkt in der
Hardware-Uhr des Mainboards:
 
1. Der Server startet um 02:00 durch den Weckzeitpunkt
2. Die VM startet automatisch (Option "Start at boot" in Proxmox)
3. Die Timer holen die fälligen Sicherungen nach
4. Ein Skript setzt den nächsten Weckzeitpunkt und fährt den Server herunter
```bash
rtcwake -m no -t $(date +%s -d 'tomorrow 02:00')
shutdown -h now
```
 
### Zu beachten
 
- Im BIOS muss "ErP Ready" (auch "EuP" oder "Deep Sleep") **deaktiviert** sein,
  sonst wird der Standby-Strom abgeschaltet und Wake-on-LAN funktioniert nicht.
- Wake-on-LAN muss im BIOS und in den Einstellungen der Netzwerkkarte aktiviert
  sein.
- Nicht jedes Mainboard unterstützt den Weckzeitpunkt der Hardware-Uhr
  zuverlässig. Das muss zuerst mit einem kurzen Test geprüft werden.
- Das Herunterfahren darf erst erfolgen, wenn alle Sicherungen fertig sind.
- Der automatische Upload von Fotos funktioniert nur, wenn der Server läuft.
- Zuerst den Verbrauch messen, dann entscheiden.
 