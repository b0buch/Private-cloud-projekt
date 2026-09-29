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

## Nextcloud

**Mehrere Selbstprüfungen schlagen fehl** (WebDAV, `.well-known`, OCS,
Schriftarten, JavaScript-Module, Datenverzeichnis), weil der Webserver sich über
die Internetfreigabe des Laptops nicht selbst erreichen kann. Nach dem Umzug nach
Hause muss geprüft werden, ob die Meldungen verschwinden.

**Der Zugriff läuft aktuell nur über HTTP.** Ohne HTTPS funktionieren einige
Funktionen von Nextcloud nicht ("copy to clipboard", "service workers").

**Die Zwei-Faktor-Authentisierung ist noch nicht aktiviert.** Sie wird wichtig,
sobald die Cloud von aussen erreichbar ist.

**Der E-Mail-Server ist noch nicht konfiguriert.** Ohne ihn können Benutzer ihr
Passwort nicht selbst zurücksetzen.

**Die maximale Grösse für den Upload ist noch nicht angepasst.** Die
Standardwerte von PHP reichen für grosse GoPro-Videos wahrscheinlich nicht.

## Speicher und Backup

**Die Datenmenge kann bis zu 4 TB betragen, für Backups steht aber nur eine
1TB-Festplatte zur Verfügung.** Es muss festgelegt werden, welche Daten gesichert
werden (wichtige Fotos und ausgewählte Videos) und welche nicht.

**Die Daten liegen aktuell auf der externen 1TB-Festplatte.** Sie muss später auf
die 4TB-Festplatte verschoben werden, dann geht die externe Festplatte ins
Backup.

**Ein Backup der Datenbank fehlt noch.** Das Backup der VM mit vzdump enthält die
Daten auf der externen Festplatte nicht, weil sie als USB-Gerät durchgereicht
wird. Für Daten und Datenbank ist ein eigenes Backup mit `mysqldump` nötig.

Die Sicherungen der Datenbank liegen aktuell auf derselben SSD wie die
Datenbank selbst. Das ist nur eine Zwischenloesung zum Testen des
Ablaufs. Sobald die zusaetzliche 128GB-SSD eingebaut ist, wird der
Zielordner dorthin verlegt.

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

