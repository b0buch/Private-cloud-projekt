# Abschlussbericht: Ferienprojekt Private Cloud
 
**Lehrjahr:** 1. Lehrjahr Applikationsentwicklung
**Zeitraum:** 18.09.2026 bis 07.10.2026
**Stand:** 08.10.2026
 
---
 
## 1. Ausgangslage und Ziel
 
Ziel des Ferienprojekts war eine eigene private Cloud auf dem eigenen Server
zu Hause. Darin sollten Fotos und Videos, besonders Ski- und GoPro-Aufnahmen,
gespeichert werden, mit Zugriff vom PC und vom Smartphone. Geplant war
zusätzlich ein zweiter Benutzer, damit eine weitere Person eigene Videos
speichern kann.
 
Dabei sollten Kenntnisse in sieben Bereichen erweitert werden: Linux und
Serveradministration, Virtualisierung, Docker, Datenbanken und SQL, Netzwerk
und VPN, Benutzer und Berechtigungen sowie Backup und Datensicherheit.
 
## 2. Ergebnis
 
Die Cloud läuft und wird genutzt. Drei Benutzer greifen von Smartphones und
Laptops darauf zu, von zu Hause und von unterwegs, ausschliesslich über VPN.
 
### Soll-Ist-Vergleich
 
| Geplant                                   | Umgesetzt                                                                                              |
| ---                                       | ---                                                                                                    |
| Server vorbereiten                        | Proxmox auf eigener Hardware, Ubuntu Server als VM                                                     |
| Linux und virtuelle Maschine einrichten   | erledigt, mit Snapshots vor jeder grösseren Änderung                                                   |
| Docker installieren                       | erledigt, vier Container mit Docker Compose                                                            |
| Nextcloud installieren                    | erledigt, mit eigenem Image (ffmpeg für Vorschaubilder)                                                |
| Datenbank einrichten                      | MariaDB, mit SQL untersucht und nach Empfehlungen angepasst                                            |
| Speicher für Fotos und Videos             | 4TB-Festplatte, qcow2 auf LVM, im laufenden Betrieb erweiterbar                                        |
| Benutzer und Berechtigungen               | drei Benutzer, Gruppen, zwei gemeinsame Ordner mit Quota, getrenntes Administratorkonto                |
| Backup einrichten                         | VM, Datenbank und Daten, automatisch, mit Nachholen und E-Mail bei Fehlern; Wiederherstellung von VM, Datenbank und Dateien getestet |
| Zugriff mit dem Smartphone                | erledigt, per Browser und App                                                                          |
| Fernzugriff über VPN                      | zwei VPN: eines für die Verwaltung, eines für die Benutzer mit Zugriff nur auf Nextcloud               |
| Sicherheit überprüfen                     | HTTPS, Zwei-Faktor-Authentisierung, Brute-Force-Schutz, Firewall, Sicherheitskonzept                   |
| Funktion testen und dokumentieren         | über 25 Tests, Arbeitsjournal für jeden Tag, Dokumentation auf GitHub                                  |
| **ein** zweiter Benutzer                  | **drei** Benutzer mit sechs Geräten                                                                    |
  
Alle geplanten Punkte sind erreicht. Über den Plan hinaus kamen dazu: HTTPS
über einen Reverse Proxy, Zwei-Faktor-Authentisierung, ein zweites VPN mit
eingeschränktem Zugriff, E-Mail-Versand und eine Benachrichtigung bei
fehlgeschlagenen Sicherungen.
 
## 3. Aufbau in Kürze
 
```
Internet ──VPN──► FRITZ!Box ──► Server ──► Proxmox ──► Ubuntu-VM ──► Docker
                                                                       ├── Caddy (HTTPS)
                                                                       ├── Nextcloud
                                                                       ├── MariaDB
                                                                       └── Redis
```
 
| Festplatte      | Aufgabe                                        |
| ---             | ---                                            |
| SSD 250 GB      | System, VM, Docker, Datenbank                  |
| HDD 4 TB        | Daten von Nextcloud, Sicherungen der VM        |
| HDD 1 TB extern | Sicherung des Ordners Foto-Video               |
 
Einzelheiten in `architektur.md`, `vpn.md`, `backup-konzept.md` und
`sicherheit.md`.
 
## 4. Ablauf
 
| Datum      | Schwerpunkt                                                      |
| ---        | ---                                                              |
| 18.09.2026 | Hardware geprüft, Proxmox installiert, Ubuntu-VM erstellt        |
| 28.09.2026 | Docker, Nextcloud, MariaDB und Redis, erste SQL-Abfragen         |
| 29.09.2026 | Datenfestplatte, Benutzer und Gruppen, Backups und erste Tests   |
| 30.09.2026 | Grosse Uploads, Vorschaubilder, HTTPS mit Caddy, Netzwerkprobleme|
| 01.10.2026 | Eigenes Docker-Image, Zwei-Faktor-Authentisierung, LVM geübt     |
| 02.10.2026 | Einbau der 4TB-Festplatte, Umzug der Daten, Umzug nach Hause     |
| 04.10.2026 | VPN auf der FRITZ!Box, Test von aussen (Sonntag, freiwillig)     |
| 05.10.2026 | DNS, Datenbank, E-Mail, endgültige Benutzer, Dokumentation       |
| 06.10.2026 | Zweites VPN, systemd-Timer, Benachrichtigung, Dokumentation      |
| 07.10.2026 | Tests der Sicherung und Wiederherstellung, Präsentation          |
 
Bis zum 02.10.2026 stand der Server am Arbeitsplatz und war über die
Internetfreigabe eines Laptops angebunden. Diese Zwischenlösung war die
Ursache für mehrere Probleme in der ersten Woche.
 
## 5. Die wichtigsten Probleme und was ich daraus gelernt habe
 
### Die Sicherungen liefen nie
 
Am 02.10.2026 waren die Zeiten für die Sicherungen eingetragen, und in der
Dokumentation stand "täglich 01:30". Am 05.10.2026 zeigte eine Kontrolle:
Keine einzige Sicherung war automatisch gelaufen. Der Server war nachts aus,
und cron holt verpasste Aufgaben nicht nach.
 
Gelöst mit systemd-Timern, die verpasste Ausführungen nach dem Start
nachholen, und einer E-Mail bei Fehlern. Das Nachholen wurde getestet, indem
der Zeitstempel des Timers zurückgesetzt wurde.
 
**Gelernt:** Eine eingerichtete Aufgabe und eine ausgeführte Aufgabe sind zwei
verschiedene Dinge. Eine Automatisierung muss man nachweisen, nicht nur
einrichten.
 
### Eine falsche Annahme hat drei Wochen überlebt
 
Seit der Planung stand in der Dokumentation, der Anschluss habe keine eigene
öffentliche IPv4-Adresse (Carrier-Grade-NAT). Darauf aufbauend war eine
zusätzliche Adresse für 5 CHF im Monat geplant. Am 04.10.2026 hat ein
Vergleich von zwei Minuten gezeigt, dass die Annahme falsch war.
 
**Gelernt:** Annahmen aus der eigenen Dokumentation prüfen, statt sie
weiterzutragen.
 
### Die Firewall sah die Pakete nicht
 
Für das zweite VPN war geplant, den Zugriff mit ufw einzuschränken. Bei der
Umsetzung zeigte sich, dass Docker veröffentlichte Ports per DNAT
weiterleitet und diese Pakete an ufw vorbeilaufen. Die Regeln mussten in die
von Docker vorgesehene Kette `DOCKER-USER`.
 
**Gelernt:** Zu verstehen, welchen Weg ein Paket durch das System nimmt, ist
wichtiger, als die Befehle eines Werkzeugs zu kennen.
 
### Ein Container vergisst alles
 
ffmpeg wurde direkt im laufenden Container installiert und war am nächsten
Morgen verschwunden, weil Docker den Container neu erstellt hatte.
 
**Gelernt:** Änderungen gehören in ein Image (Dockerfile), nicht in einen
laufenden Container.
 
### Docker startete vor der Festplatte
 
Nach dem ersten selbstständigen Neustart zu Hause meldete Nextcloud ein
ungültiges Datenverzeichnis. Docker hatte ein leeres Verzeichnis eingebunden,
weil die Festplatte noch nicht bereit war.
 
**Gelernt:** Die Reihenfolge beim Hochfahren ist Teil der Konfiguration. Seit
diesem Tag ist sie für Docker und WireGuard ausdrücklich festgelegt.
 
### Fehlersuche Schicht für Schicht
 
Am 30.09.2026 blieben Installationen ohne Fehlermeldung hängen. Die Ursache
lag nicht in der Anwendung, sondern in der Namensauflösung über IPv6 ohne
IPv6-Route.
 
**Gelernt:** Ein Netzwerkproblem von unten nach oben eingrenzen: Schnittstelle,
Route, Verbindung nach aussen, Namensauflösung, erst zum Schluss die
Anwendung.
 
## 6. Reflexion nach Kompetenzen
 
| Kompetenz                     | Was ich jetzt kann                                                                                     |
| ---                           | ---                                                                                                    |
| Linux und Serveradministration| Dienste mit systemd steuern, Timer und Vorlagen schreiben, Protokolle lesen, Rechte und `umask` setzen |
| Virtualisierung               | VMs in Proxmox erstellen, Snapshots und Backups, Wiederherstellung als neue VM, Festplatten als qcow2  |
| Docker                        | Compose-Dateien schreiben, eigene Images bauen, Volumes und Netzwerke verstehen, Startreihenfolge      |
| Datenbanken und SQL           | JOIN, Self-Join, GROUP BY, Bitmasken, Indizes; Dump und Wiederherstellung mit Wartungsmodus            |
| Netzwerk und VPN              | DHCP, DNS, NAT, Portfreigaben, zwei WireGuard-VPN, Split Tunnel, Firewall-Regeln mit iptables und ufw  |
| Benutzer und Berechtigungen   | Benutzer, Gruppen, Quotas, gemeinsame Ordner, Trennung von Verwaltung und Alltag                       |
| Backup und Datensicherheit    | Sicherungskonzept mit begründeten Lücken, getestete Wiederherstellung, HTTPS, Zertifikate, 2FA         |
 
Am meisten gelernt habe ich nicht beim Einrichten, sondern bei der
Fehlersuche. Fast jeder Tag hatte ein Problem, dessen Ursache an einer
anderen Stelle lag, als ich zuerst vermutet hatte.
 
Wichtig war für mich auch das Gespräch mit meinem Berufsbildner über
Tailscale. Ich habe dabei gelernt, dass eine technische Lösung nicht nur
funktionieren muss, sondern in einem fremden Netz auch erlaubt und
nachvollziehbar sein muss.
 
## 7. Was ich beim nächsten Mal anders machen würde
 
- **Annahmen sofort prüfen.** Die falsche Annahme zum Anschluss hätte ich am
  ersten Tag in zwei Minuten widerlegen können.
- **Automatisierung sofort nachweisen.** Nach dem Einrichten einer geplanten
  Aufgabe am nächsten Tag kontrollieren, ob sie wirklich lief.
- **Den Server früher am endgültigen Ort aufstellen.** Die Zwischenlösung über
  den Laptop hat in der ersten Woche viel Zeit gekostet.
- **Dokumentation laufend korrigieren.** Am 05.10.2026 bestand mehr als die
  Hälfte der Dokumentationsarbeit aus Korrekturen von Angaben, die nicht mehr
  stimmten.
## 8. Bewusst offene Punkte und Ausblick
 
| Punkt                                   | Begründung                                               |
| ---                                     | ---                                                      |
| Nur der Ordner Foto-Video wird gesichert| Sicherungsfestplatte mit 1 TB, Daten bis 4 TB            |
| Kein automatischer Upload vom Smartphone| Server läuft nicht immer, grosse Dateien über Mobilfunk  |
| Festplatten nicht verschlüsselt         | Risiko bekannt, in `sicherheit.md` festgehalten          |
| Selbst signiertes Stammzertifikat       | Risiko bekannt, Lösung über eigene Domain vorgesehen     |
 
Mögliche nächste Schritte:
 
- eigene Domain mit Zertifikat von Let's Encrypt über die DNS-Prüfung
- Nachtbetrieb mit `rtcwake`, um Strom zu sparen
- Umstellung des Heimnetzes auf einen seltenen Adressbereich
- zweite Festplatte für eine vollständige Sicherung, falls die Datenmenge
  wächst
## 9. Fazit
 
Das Projektziel ist erreicht und in mehreren Punkten übertroffen. Die Cloud
ist kein Versuchsaufbau mehr, sondern wird von drei Personen genutzt. Damit
gelten für sie dieselben Anforderungen wie für einen echten Dienst:
Sicherungen, die nachweislich laufen, ein Zugriff, der nur so weit reicht wie
nötig, und eine Dokumentation, die den tatsächlichen Zustand beschreibt.