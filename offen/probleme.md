# Offene Probleme

Punkte, die noch offen sind oder erst nach dem Umzug nach Hause gelöst werden
können.

## Netzwerk

**Der Internetanbieter InterXS verwendet Carrier-Grade-NAT.** Der Anschluss hat
keine eigene öffentliche IPv4-Adresse, deshalb ist eine Portweiterleitung über
IPv4 nicht möglich. Es gibt eine feste IPv6-Adresse (/64).
Mögliche Lösungen: WireGuard über IPv6 nutzen, eine öffentliche IPv4 für
5 CHF pro Monat dazubestellen, oder einen Dienst wie Tailscale verwenden, der
ohne Portweiterleitung funktioniert.
→ Entscheidung siehe `loesungen.md`

**Die VM erhält ihre IP-Adresse aktuell über die Internetfreigabe des Laptops**
(`192.168.137.x`). Zu Hause an der FRITZ!Box gilt ein anderer Adressbereich
(`192.168.178.x`). Alle Konfigurationen mit fester IP-Adresse müssen dann
angepasst werden: `trusted_domains`, `overwrite.cli.url`, `.env`, netplan und
`/etc/network/interfaces` auf dem Proxmox-Host.

**Der Server braucht zu Hause eine feste IP-Adresse.** Wird in der FRITZ!Box
unter Heimnetz → Netzwerk zugewiesen. Mit reinem DHCP könnte sich die Adresse
ändern und der Zugriff auf die Cloud würde nicht mehr funktionieren.

**Zu prüfen:** Welches Router-Modell steht zu Hause wirklich (eigene 7490 oder
ein Gerät vom Anbieter) und welche FRITZ!OS-Version ist installiert? WireGuard
braucht mindestens FRITZ!OS 7.50.

Die Netzwerkverbindung ueber die Internetfreigabe des Laptops ist
anfaellig. Sie bietet kein IPv6 und die Konfiguration geht bei einem
Neustart des Laptops leicht verloren. Nach dem Umzug an die FRITZ!Box
sollten diese Probleme wegfallen.


## Nextcloud

**Mehrere Selbstprüfungen schlagen fehl** (WebDAV, `.well-known`, OCS,
Schriftarten, JavaScript-Module, Datenverzeichnis), weil der Webserver sich über
die Internetfreigabe des Laptops nicht selbst erreichen kann. Nach dem Umzug nach
Hause muss geprüft werden, ob die Meldungen verschwinden.

**Der E-Mail-Server ist noch nicht konfiguriert.** Ohne ihn können Benutzer ihr
Passwort nicht selbst zurücksetzen.

**Benutzerkonten und Gruppen werden neu angelegt.** Die endgültigen Konten,
Gruppen und gemeinsamen Ordner werden in einem Durchgang erstellt, sobald das
Konzept feststeht. Das Administratorkonto bleibt bis dahin unverändert.

**Trennung von Administration und täglicher Nutzung fehlt.** Geplant ist ein
eigenes Konto für die Administration, das tägliche Konto ohne
Administratorrechte.

## Speicher und Backup

**Die Datenmenge kann bis zu 4 TB betragen, für Backups steht aber nur eine
1TB-Festplatte zur Verfügung.** Es muss festgelegt werden, welche Daten gesichert
werden (wichtige Fotos und ausgewählte Videos) und welche nicht.

**Die Daten liegen aktuell auf der externen 1TB-Festplatte.** Sie muss später auf
die 4TB-Festplatte verschoben werden, dann geht die externe Festplatte ins
Backup.

## Proxmox

**Beim Aktualisieren der Paketliste erscheint ein Fehler** (`apt-get update`).
Vermutlich sind die Enterprise-Paketquellen ohne Abonnement eingetragen. Zu
prüfen unter `pve` → Updates → Repositories.

## Sicherheit

**Auch mit VPN bleiben Risiken bestehen.** Wenn ein Smartphone mit einem
WireGuard-Schlüssel verloren geht, hat der Finder Zugriff auf das Netzwerk.
Deshalb bekommt jedes Gerät eine eigene Verbindung, damit einzelne Schlüssel
gesperrt werden können.

**Die Festplatten sind nicht verschlüsselt.** Bei einem Diebstahl des Servers
könnten die Daten gelesen werden. Zu prüfen, ob eine Verschlüsselung sinnvoll
ist.

**Die Datei `.env` enthält die Passwörter der Datenbank** und darf nie in Git
aufgenommen werden. Sie steht in der `.gitignore`.

**In der öffentlichen Dokumentation werden keine echten IP-Adressen und keine
DynDNS-Namen verwendet**, sondern Platzhalter.

## Smartphones

**Die Smartphones der Benutzer (Android) haben eine aggressive
Akku-Optimierung.** Diese kann den automatischen Upload von Fotos im Hintergrund
stoppen. Für die Nextcloud-App muss die Akku-Optimierung deaktiviert werden, bei
Xiaomi zusätzlich der Autostart erlaubt werden.

## Gelöst

|                    Problem                       |                   Gelöst am                      |
|                      ---                         |                      ---                         |
| Zugriff nur über HTTP, kein HTTPS                | 30.09.2026                                       |
| Maximale Uploadgrösse zu klein für grosse Videos | 30.09.2026                                       |
| ffmpeg nur im laufenden Container installiert    | 01.10.2026, eigenes Dockerfile                   |
| Zwei-Faktor-Authentisierung nicht aktiviert      | 01.10.2026                                       |
| Backup der Datenbank fehlte                      | 29.09.2026, mysqldump mit Wiederherstellungstest |

