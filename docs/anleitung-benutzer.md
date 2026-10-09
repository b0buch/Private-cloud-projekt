# Anleitung für Benutzer: Private Cloud
 
Mit dieser Anleitung richtest du den Zugang zu unserer privaten Cloud auf
deinem Smartphone und deinem Computer ein. Du brauchst etwa 15 Minuten pro
Gerät.
 
**Adresse der Cloud:** `https://privatecloud.fritz.box`
 
Die Cloud ist absichtlich **nicht** offen im Internet erreichbar. Du verbindest
dich zuerst über ein VPN (WireGuard) und öffnest danach die Cloud wie eine
normale Webseite.
 
---
 
## Was du bekommst
 
- ein eigenes Konto mit **100 GB** Speicher für deine persönlichen Dateien
- Zugriff auf die gemeinsamen Ordner **Foto-Video** und **Ski**
- Zugriff von zu Hause und von unterwegs
## Was du vom Administrator bekommst
 
| Was                              | Wie                                          |
| ---                              | ---                                          |
| Einladung per E-Mail             | an deine E-Mail-Adresse                      |
| VPN-Zugang                       | persönlich: QR-Code oder Datei auf USB-Stick |
| Zertifikat `server.crt`          | per E-Mail oder Messenger (nicht geheim)     |
 
Jedes Gerät bekommt einen **eigenen** VPN-Zugang. Bitte nie einen Zugang auf
ein zweites Gerät kopieren.
 
---
 
## Schritt 1: Konto aktivieren
 
1. Öffne die E-Mail mit der Einladung von Nextcloud.
   **Findest du sie nicht, schau im Spam-Ordner nach.**
2. Klicke auf den Link und lege ein Passwort fest.
   - mindestens 8 Zeichen (kürzere lehnt Nextcloud ab)
   - nirgends sonst verwendet
3. Das Passwort kennst nur du. Der Administrator kann es nicht sehen.

Der Link funktioniert nur mit aktivem VPN. Mach also zuerst Schritt 2 und
öffne den Link danach.
 
---
 
## Schritt 2: VPN einrichten
 
### Smartphone (Android)
 
1. Installiere die App **WireGuard** aus dem Play Store.
2. Tippe auf **+** und dann:
   - **QR-Code scannen**, wenn dir der Administrator einen QR-Code zeigt, oder
   - **Aus Datei oder Archiv importieren**, wenn du eine Datei bekommen hast.
     Lösche die Datei danach aus dem Ordner "Downloads".
3. Gib dem Tunnel den Namen `privatecloud`.
4. Schalte den Tunnel mit dem Schalter ein. Android fragt einmal nach der
   Erlaubnis für ein VPN, bestätige mit **OK**.
### Computer (Windows)
 
1. Lade WireGuard von **wireguard.com/install** herunter und installiere es.
2. Stecke den USB-Stick mit deiner Datei ein.
3. Klicke in WireGuard auf **Tunnel aus Datei importieren** und wähle **deine**
   Datei.
4. Klicke auf **Aktivieren**.
5. Lösche die Datei danach vom USB-Stick.
### Computer (macOS)
 
1. Installiere **WireGuard** aus dem App Store.
2. Klicke auf **Tunnel aus Datei importieren** und wähle deine Datei.
3. Erlaube die VPN-Konfiguration, wenn macOS danach fragt.
4. Klicke auf **Aktivieren** und lösche die Datei danach.
---
 
## Schritt 3: Zertifikat installieren
 
Ohne diesen Schritt zeigt der Browser eine Warnung "Verbindung nicht sicher".
Die Verbindung ist trotzdem verschlüsselt, das Zertifikat bestätigt nur, dass
du wirklich mit unserem Server sprichst.
 
### Android
 
1. Speichere die Datei `server.crt` auf dem Smartphone.
2. Öffne die **Einstellungen** und suche nach **Zertifikat** oder
   **CA-Zertifikat**. Der Menüpunkt heisst je nach Hersteller etwas anders.
3. Wähle **CA-Zertifikat installieren**, bestätige die Warnung und wähle die
   Datei.
4. Android verlangt dafür eine Bildschirmsperre (PIN, Muster oder
   Fingerabdruck).
### Windows
 
1. Doppelklick auf `server.crt` → **Zertifikat installieren**.
2. **Aktueller Benutzer** → **Weiter**.
3. **Alle Zertifikate in folgendem Speicher speichern** → **Durchsuchen** →
   **Vertrauenswürdige Stammzertifizierungsstellen** → **OK**.
4. **Weiter** → **Fertig** und die Warnung bestätigen.
5. Den Browser neu starten.
### macOS
 
1. Doppelklick auf `server.crt`, die Schlüsselbundverwaltung öffnet sich.
2. Das Zertifikat doppelt anklicken → **Vertrauen** →
   "Bei Verwendung dieses Zertifikats": **Immer vertrauen**.

**Firefox** hat einen eigenen Speicher: Einstellungen → Datenschutz &
Sicherheit → Zertifikate anzeigen → Zertifizierungsstellen → **Importieren**.
 
---
 
## Schritt 4: Cloud öffnen
 
### Im Browser (Smartphone und Computer)
 
1. VPN einschalten.
2. `https://privatecloud.fritz.box` öffnen.
3. Mit deinem Benutzernamen und Passwort anmelden.
4. Tipp: Lege ein Lesezeichen an.
### Mit der App (Smartphone)
 
1. Installiere die App **Nextcloud** aus dem Play Store.
2. VPN einschalten.
3. Als Serveradresse `https://privatecloud.fritz.box` eingeben.
4. Die App öffnet eine Anmeldeseite im Browser. Melde dich dort an und
   erlaube den Zugriff.
5. Fragt die App, ob sie dem Zertifikat vertrauen soll, bestätige das einmal.
### Mit dem Desktop-Programm (Computer, freiwillig)
 
Wer Dateien automatisch mit einem Ordner auf dem Computer abgleichen will,
installiert **Nextcloud Desktop** von **nextcloud.com/install**. Serveradresse
wie oben. Für gelegentliches Hoch- und Herunterladen reicht der Browser.
 
---
 
## Im Alltag
 
### VPN ein- und ausschalten
 
- **Zum Benutzen der Cloud:** VPN einschalten.
- **Wenn du die Cloud nicht brauchst:** VPN ausschalten.

**Wichtig:** Der Server läuft nicht immer. Ist er ausgeschaltet und dein VPN
eingeschaltet, öffnen sich auf deinem Gerät **auch andere Webseiten nicht**.
In diesem Fall einfach das VPN ausschalten, dann geht alles wieder.

### Wo lege ich meine Dateien ab?

| Ordner                | Wer sieht ihn | Gesichert?          |
| ---                   | ---           | ---                 |
| **Foto-Video**        | alle Benutzer | **ja**, wöchentlich |
| **Ski**               | alle Benutzer | nein                |
| Deine eigenen Dateien | nur du        | **nein**            |

**Nur der Ordner Foto-Video wird gesichert.** Fällt die Festplatte aus, sind
alle anderen Dateien verloren. Was dir wichtig ist, gehört in **Foto-Video**.

### Grosse Videos
 
Über das Mobilfunknetz dauert eine Datei mit 1.5 GB etwa 10 Minuten, ein
GoPro-Video mit 4 GB etwa eine halbe Stunde. Grosse Dateien deshalb lieber
zu Hause im WLAN hochladen.
 
### Passwort vergessen
 
Auf der Anmeldeseite auf **Passwort vergessen?** klicken. Die E-Mail landet
oft im **Spam-Ordner**.
 
### Zwei-Faktor-Authentisierung (empfohlen)
 
Unter **Persönliche Einstellungen → Sicherheit** kannst du einen zweiten
Faktor mit einer Authenticator-App einrichten. Dann reicht ein gestohlenes
Passwort nicht mehr. Bewahre die Wiederherstellungscodes **nicht** auf dem
Smartphone auf.
 
---
 
## Wenn etwas nicht funktioniert
 
|                 Problem                       |                            Lösung                                      |
|                  ---                          |                              ---                                       |
| Cloud öffnet sich nicht                       | Ist das VPN eingeschaltet? Läuft der Server? Den Administrator fragen. |
| Andere Webseiten öffnen sich nicht            | VPN ausschalten. Der Server ist vermutlich aus.                        |
| Warnung "Verbindung nicht sicher"             | Zertifikat aus Schritt 3 installieren.                                 |
| "Zugriff über nicht vertrauenswürdige Domain" | Adresse genau so eingeben: `https://privatecloud.fritz.box`            |
| E-Mail von Nextcloud kommt nicht an           | Spam-Ordner prüfen.                                                    |
| Upload sehr langsam                           | Im Mobilfunknetz normal, besser zu Hause im WLAN.                      |
 
## Gerät verloren oder verkauft?
 
**Sofort dem Administrator Bescheid geben.** Er sperrt den VPN-Zugang genau
dieses Geräts, deine anderen Geräte funktionieren weiter. Ändere danach dein
Passwort in der Cloud.
 
## Was du nie tun solltest
 
- VPN-Datei oder QR-Code an andere weitergeben oder per Messenger verschicken
- denselben VPN-Zugang auf zwei Geräten verwenden
- dein Cloud-Passwort für andere Dienste verwenden