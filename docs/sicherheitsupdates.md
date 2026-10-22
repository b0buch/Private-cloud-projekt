# Sicherheitsupdates

Liste der Updates, die wegen einer Sicherheitslücke durchgeführt wurden.

| Datum      | Lücke                                              | Betroffen       | Vorher | Nachher | Stand        |
| ---        | ---                                                | ---             | ---    | ---     | ---          |
| 22.10.2026 | PHP2Shell (CVE-2026-109618, CVE-2026-109619), RCE | PHP im Container `nextcloud-app` | 8.5.11 | 8.5.12 | geschlossen |

## 22.10.2026 — PHP2Shell

- Zwei kritische Lücken in PHP (CVSS 9.2), Ausführen von Code ohne Anmeldung.
- Risiko bei uns geringer, weil Nextcloud nur über VPN erreichbar ist.
- Update durch Neubau des eigenen Images (`build --pull --no-cache`), dabei
  auch Nextcloud 35.0.1 → 35.0.2.
- Kontrolle: `docker exec nextcloud-app php -v` zeigt PHP 8.5.12.
- Details im Arbeitsjournal: `journal/2026-10-22.md`.
