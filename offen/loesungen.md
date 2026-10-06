# Lösungen und Entscheidungen

Ideen und Entscheidungen für Punkte, die später umgesetzt werden.

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

- [x] **Wichtig:** Die 4TB-Festplatte von Anfang an als LVM einrichten, auch wenn sie
      zuerst allein ist. Sonst müssen später alle Daten neu kopiert werden. (02.10.2026)

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

Ein Skript, das per Cron läuft:

1. Nextcloud in den Wartungsmodus setzen (`occ maintenance:mode --on`)
2. Datenbank mit `mysqldump` sichern
3. Wartungsmodus wieder ausschalten
4. Alte Sicherungen nach einer festgelegten Zeit löschen

Der Wartungsmodus ist nötig, damit Dateien und Datenbank zueinander passen.
Ohne ihn könnte während des Dumps eine Datei hochgeladen werden, die dann in der
Sicherung fehlt.

## Ablauf beim Wechsel auf die 4TB-Festplatte

Wichtig: Der Befehl "lvextend" darf erst **nach** dem Entfernen der alten
Festplatte ausgefuehrt werden. Wird das Logical Volume vorher auf den ganzen
Speicher erweitert, belegt es beide Festplatten und die alte laesst sich nicht
mehr herausnehmen, ohne das Dateisystem zu verkleinern.

Reihenfolge:

1. pvcreate auf der neuen Festplatte
2. vgextend: neue Festplatte zur Volume Group hinzufuegen
3. pvmove: Daten von der alten auf die neue Festplatte verschieben
   (im laufenden Betrieb, Nextcloud bleibt erreichbar)
4. vgreduce und pvremove: alte Festplatte aus der Gruppe entfernen
5. lvextend und resize2fs: erst jetzt das Volume auf den ganzen Speicher
   erweitern
6. Die freie 1TB-Festplatte neu formatieren und als Sicherungsziel einrichten

**Abweichung vom geplanten Ablauf.** pvmove wurde bewusst nicht verwendet.
Der Befehl verschiebt alle zugewiesenen Extents, nicht die belegten Daten:
931.51 GiB waren zugewiesen, aber nur 7.8 GB belegt. Auf Festplatten mit
5400 U/min haette das drei bis vier Stunden gedauert. Stattdessen wurden die
Daten mit rsync kopiert (1 Minute 9 Sekunden), wobei zu keinem Zeitpunkt nur
eine Kopie existierte.

## Nachtbetrieb mit rtcwake (notwendig fuer die Sicherungen)

Der Server muss nicht rund um die Uhr laufen. Bei etwa 45 Watt im Leerlauf sind
das ungefaehr 400 kWh im Jahr. Wenn er nur wenige Stunden taeglich laeuft, sinkt
der Verbrauch deutlich.

### Was nicht geht: einfach die Steckdose ausschalten

Eine schaltbare Steckdose trennt den Strom hart, wie ein herausgezogenes Kabel.
Das Betriebssystem kann seine Daten nicht mehr auf die Festplatte schreiben und
die Datenbank bricht mitten in einer Transaktion ab. Einmal geht das meistens
gut, taeglich fuehrt es frueher oder spaeter zu beschaedigten Daten.

Ausserdem schliessen sich Steckdose und Fernsteuerung gegenseitig aus: Sowohl
Wake-on-LAN als auch der Weckzeitpunkt im BIOS brauchen Standby-Strom. Dieser
betraegt nur 1 bis 3 Watt, also wenige Franken im Jahr.

### Bessere Loesung

**Einschalten von unterwegs:** Die FRITZ!Box kann Geraete im Heimnetz per
Wake-on-LAN starten (Heimnetz > Netzwerk > Computer starten). Ueber die
MyFRITZ!App oder ueber VPN funktioniert das auch vom Smartphone aus. Es gibt
zusaetzlich die Option, den Computer automatisch zu starten, sobald aus dem
Internet darauf zugegriffen wird.

**Ausschalten von unterwegs:** Mit einer SSH-App auf dem Smartphone
(zum Beispiel Termux oder JuiceSSH) den Befehl "shutdown -h now" auf dem
Proxmox-Host ausfuehren. Proxmox faehrt die VM dank des QEMU Guest Agent
sauber herunter.

**Automatisch nachts:** Der Befehl "rtcwake" stellt einen Weckzeitpunkt in der
Hardware-Uhr des Mainboards. Der Server kann also selbst festlegen, wann er das
naechste Mal startet. Ablauf:

1. Der Server startet um 02:00 durch den Weckzeitpunkt
2. Die VM startet automatisch (Option "Start at boot" in Proxmox)
3. Ein Skript fuehrt das Datenbank-Backup und danach vzdump aus
4. Das Skript setzt den naechsten Weckzeitpunkt und faehrt den Server herunter

Beispiel:

    rtcwake -m no -t $(date +%s -d 'tomorrow 02:00')
    shutdown -h now



Tailscale wurde nicht getestet. Entscheidung vom 05.10.2026: Der Vergleich
bringt fuer das Projektziel keinen zusaetzlichen Nutzen, weil WireGuard die
Anforderung bereits erfuellt und kein Carrier-Grade-NAT vorliegt. Der Vergleich
in vpn.md stuetzt sich auf die Dokumentation beider Loesungen und ist als
solcher gekennzeichnet.

### Zu beachten

- Im BIOS muss "ErP Ready" (auch "EuP" oder "Deep Sleep") **deaktiviert** sein,
  sonst wird der Standby-Strom abgeschaltet und Wake-on-LAN funktioniert nicht.
- Wake-on-LAN muss im BIOS und in den Einstellungen der Netzwerkkarte aktiviert
  sein.
- Nicht jedes Mainboard unterstuetzt den Weckzeitpunkt der Hardware-Uhr
  zuverlaessig. Das muss zuerst mit einem kurzen Test geprueft werden, bevor
  darauf aufgebaut wird.
- Fuer den Aufruf des Skripts in der VM braucht es einen SSH-Schluessel ohne
  Passwort, sonst bleibt das Skript bei der Passwortabfrage stehen.
- Der automatische Upload von Fotos funktioniert nur, wenn der Server laeuft.
  Eventuell sollte er zusaetzlich abends kurz starten, damit die Smartphones
  ihre Aufnahmen hochladen koennen.
- Zuerst den Verbrauch messen, dann entscheiden. Viele schaltbare Steckdosen
  zeigen den Verbrauch an.

Diese Erweiterung wird erst nach dem Umzug nach Hause umgesetzt, wenn das
Grundsystem laeuft.