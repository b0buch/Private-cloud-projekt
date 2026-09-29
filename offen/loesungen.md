# Lösungen und Entscheidungen

Ideen und Entscheidungen für Punkte, die später umgesetzt werden.

## Fernzugriff: zuerst WireGuard, später Tailscale

**Entscheidung:** Für die Projektphase wird beim Anbieter InterXS für einen Monat
eine öffentliche statische IPv4-Adresse dazubestellt (5 CHF pro Monat). Damit
funktioniert WireGuard direkt auf der FRITZ!Box 7490 mit Portweiterleitung.

Nach dem Projekt wird auf **Tailscale** gewechselt, damit keine monatlichen
Kosten mehr entstehen. Tailscale funktioniert auch hinter Carrier-Grade-NAT, weil
beide Geräte eine ausgehende Verbindung zu einem Koordinationsserver aufbauen und
keine Portweiterleitung nötig ist.

**Vorteil dieser Lösung:** Im Projekt wird das VPN selbst konfiguriert und
verstanden, im Betrieb wird danach die günstigere Variante genutzt. In der
Dokumentation lassen sich beide Varianten vergleichen.

|                          | WireGuard auf der FRITZ!Box |              Tailscale            |
|           ---            |             ---             |                ---                |
| Kosten                   |   5 CHF/Monat für die IPv4  |       gratis bis 100 Geräte       |
| Funktioniert hinter CGNAT|            nein             |               ja                  |
| Aufwand beim Einrichten  |            mittel           |               klein               |
| Lerneffekt               |            hoch             |               gerin               |
| Abhängigkeit von Dritten |            keine            | Koordinationsserver von Tailscale |

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

> **Wichtig:** Die 4TB-Festplatte von Anfang an als LVM einrichten, auch wenn sie
> zuerst allein ist. Sonst müssen später alle Daten neu kopiert werden.

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

## Backup der Datenbank (geplant)

Ein Skript, das per Cron läuft:

1. Nextcloud in den Wartungsmodus setzen (`occ maintenance:mode --on`)
2. Datenbank mit `mysqldump` sichern
3. Wartungsmodus wieder ausschalten
4. Alte Sicherungen nach einer festgelegten Zeit löschen

Der Wartungsmodus ist nötig, damit Dateien und Datenbank zueinander passen.
Ohne ihn könnte während des Dumps eine Datei hochgeladen werden, die dann in der
Sicherung fehlt.


## Stromverbrauch senken (Idee fuer spaeter)

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