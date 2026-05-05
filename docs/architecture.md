# Architektur

## Übersicht

Eine Debian 13 VM (`docker2`) im LAN `192.168.179.0/25` hostet alle Container über Docker Compose. Externer Zugriff läuft ausschließlich über **Nginx Proxy Manager**, der pro Host ein eigenes Let's Encrypt Zertifikat ausstellt. DNS-Auflösung im LAN erledigt **PiHole + unbound** (recursive resolver), die externen DNS-Einträge (`*.phytech.de`) zeigen via DynDNS auf die Heim-IP.

## Komponenten auf der VM

| Service | Zweck | Domain |
|---|---|---|
| Nginx Proxy Manager | Reverse Proxy + LE Certs | (Admin-Port intern) |
| Paperless-NGX | DMS | `dms.phytech.de` |
| n8n | Workflow-Automation | (intern, nur LAN) |
| Stirling-PDF | PDF-Manipulation | `ocr.phytech.de` |
| Portainer + Agent | Container-Mgmt | (intern) |
| TacticalRMM | Endpoint-Mgmt | (intern) |

## Datenfluss: Mail → DMS

```
                ┌──────────────────────────────────────────────┐
                │              M365 Mailbox                    │
                └──────────────┬───────────────────────────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
   ┌─────────────────────┐         ┌──────────────────────┐
   │  Paperless OAuth2   │         │   n8n Workflow       │
   │  (Mail Rules in     │         │   (KI-Rename + Sort) │
   │   Paperless)        │         │                      │
   └─────────┬───────────┘         └──────────┬───────────┘
             │                                │
             │                                ▼
             │                    ┌────────────────────────┐
             │                    │  SharePoint            │
             │                    │  /DMS/Mail Anhänge/    │
             │                    │   ├─ Rechnungen/       │
             │                    │   ├─ Verträge/         │
             │                    │   ├─ Behörden/         │
             │                    │   └─ ...               │
             │                    └──────────┬─────────────┘
             │                               │
             │                               │ rclone move
             │                               │ (cron, jede Minute)
             │                               ▼
             │                    ┌────────────────────────┐
             ├───────────────────►│  Paperless Consume     │
             │                    │  /home/docker2/pngx/   │
             │                    │   consume/             │
             │                    └──────────┬─────────────┘
             │                               │
             ▼                               ▼
   ┌──────────────────────────────────────────────────────┐
   │              Paperless-NGX DMS                        │
   │              (OCR, Tagging, Volltext-Suche)           │
   └──────────────────────┬───────────────────────────────┘
                          │
                          │ document_exporter + rclone copy
                          │ (cron, täglich 02:00)
                          ▼
              ┌────────────────────────┐
              │  SharePoint            │
              │  /DMS/Archiv/          │
              └────────────────────────┘
```

## Warum zwei Eingänge (Paperless OAuth2 + n8n)?

- **Paperless OAuth2** erfasst alles, was direkt ins DMS soll (OCR, Tagging, Volltext).
- **n8n** kümmert sich um Anhänge, die im SharePoint nach Kategorie strukturiert abgelegt werden sollen, mit besseren Dateinamen — Dinge, die nicht zwingend ins DMS, aber im Archiv menschlich auffindbar sein müssen.

Beide Pipelines sind unabhängig und greifen auf separate Mail-Folder/Regeln zu.

## Warum rclone + Cron statt n8n für den Sync?

Frühere Versuche, den Paperless→SharePoint Export in einem n8n-Workflow zu bauen, hatten zu viel Reibung (Binary-Daten zwischen Nodes, fehlende native Folder-Create, Auth-Tokens). Ein kleiner Crontab mit `rclone move` und `rclone copy` ist robuster, transparenter und erfordert keine Workflow-Wartung.

n8n bleibt für die KI-Schritte zuständig, wo es seine Stärken (LangChain Anthropic Node, IF-Nodes, SplitInBatches) ausspielt.

## Netzwerk-Layout

```
Internet ──► Strato DynDNS (*.phytech.de) ──► Heim-Router (Port 80/443)
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │  Debian VM       │
                                          │  192.168.179.x   │
                                          │                  │
                                          │  ┌────────────┐  │
                                          │  │    NPM     │  │ ◄── LE Cert pro Host
                                          │  └─────┬──────┘  │
                                          │        │         │
                                          │  ┌─────▼──────┐  │
                                          │  │ Container  │  │
                                          │  │ via Proxy  │  │
                                          │  └────────────┘  │
                                          └──────────────────┘

LAN-Clients ──► PiHole DNS ──► lokale A-Records für *.phytech.de ──► VM intern
```

PiHole hat lokale DNS-Rewrites, damit `dms.phytech.de` & Co. im LAN direkt auf die VM zeigen, statt über den Router-Hairpin zu laufen.
