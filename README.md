# paperless-m365-stack

Self-hosted Paperless-NGX mit Microsoft 365 Anbindung (Outlook + SharePoint), reverse-proxied über Nginx Proxy Manager und automatisiert mit n8n + Claude AI für intelligente Dokumentenbenennung.

## Was macht das Setup?

- **Paperless-NGX** als zentrales DMS auf einer Debian VM, erreichbar unter einer eigenen Domain mit Let's Encrypt SSL über NPM.
- **Outlook OAuth2** Integration: Paperless holt Mail-Anhänge direkt aus M365.
- **SharePoint Sync** über `rclone`: Eingehende Dokumente aus `Mail Anhänge` werden in den Consume-Ordner geschoben, der nächtliche Paperless-Export wird ins SharePoint-Archiv gespiegelt.
- **n8n + Claude** als parallele Pipeline: Mail-Anhänge werden per KI analysiert, sinnvoll umbenannt (z.B. `2025-09-Telekom-Rechnung.pdf`) und in kategorisierte SharePoint-Ordner einsortiert (Rechnungen, Verträge, Behörden, ...).
- Telegram-Notifications am Ende jedes Workflow-Laufs.

## Architektur (Kurzfassung)

```
M365 Outlook ─┐
              ├──► n8n (KI-Rename) ──► SharePoint /DMS/Mail Anhänge/<Kategorie>
              │                              │
              │                              ▼ (rclone, jede Minute)
              └──► Paperless OAuth2 ────► Paperless Consume ──► Paperless DMS
                                                                       │
                                                                       ▼ (cron, 02:00)
                                                            SharePoint /DMS/Archiv
```

Details und Diagramm siehe [`docs/architecture.md`](docs/architecture.md).

## Repo-Struktur

| Ordner | Inhalt |
|---|---|
| [`setup/`](setup/) | Installation: Docker Compose, NPM, M365 OAuth2, PiHole DNS |
| [`automation/`](automation/) | rclone Config + Crontabs für SharePoint-Sync und nächtlichen Export |
| [`flows/`](flows/) | n8n Workflows (importierbare JSONs + READMEs) |
| [`docs/`](docs/) | Architektur und weiterführende Notizen |

## Quick Start

1. Voraussetzungen prüfen und Stack hochziehen → [`setup/README.md`](setup/README.md)
2. rclone und Crontab einrichten → [`automation/README.md`](automation/README.md)
3. n8n Workflows importieren → [`flows/README.md`](flows/README.md)

## Stack

- Debian 13 VM, Docker Compose
- Paperless-NGX (PostgreSQL + Redis)
- Nginx Proxy Manager (eigener LE Cert pro Host)
- PiHole + unbound (lokales DNS)
- n8n (self-hosted)
- Stirling-PDF (für PDF-Manipulation)
- rclone (SharePoint via OneDrive Backend)
- Anthropic Claude API (über `@n8n/n8n-nodes-langchain.anthropic`)

## Lizenz

MIT — siehe [LICENSE](LICENSE).

## Hinweis

Dieses Repo dokumentiert ein konkretes Homelab-Setup. Alle Domains, Tenant-IDs, Telegram Chat-IDs und Outlook Folder-IDs in den Beispieldateien sind Platzhalter und müssen vor dem Import an die eigene Umgebung angepasst werden — siehe die jeweiligen READMEs.
