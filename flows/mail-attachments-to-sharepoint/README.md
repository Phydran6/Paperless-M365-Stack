# Flow: Mail Anhänge → SharePoint mit KI-Rename

## Was macht der Flow?

Pro Lauf wird eine Mail mit Anhängen aus einem definierten Outlook-Ordner gezogen und Anhang für Anhang verarbeitet:

1. **Anhang prüfen.** PDF? Wenn ja, kurzer Encrypt-Check (sucht `/Encrypt` Marker), damit passwortgeschützte PDFs nicht den Claude-Call sprengen.
2. **Claude analysiert das Dokument** (über den nativen `@n8n/n8n-nodes-langchain.anthropic` Document-Node, Modell `claude-sonnet-4-20250514`). Antwort hat zwei Zeilen:
   ```
   NAME: 2025-09-Telekom-Rechnung
   KATEGORIE: Rechnungen
   ```
3. **SharePoint-Ordner anlegen** (HTTP REST API, da kein nativer Folder-Create Node).
4. **Datei hochladen** (HTTP REST API, mit `overwrite=true`).
5. **Ergebnisse sammeln**, Telegram-Zusammenfassung schicken, Mail je nach Erfolg in `n8n-erfolgreich` oder `n8n-fehler` verschieben.

Bei Fehlern in der KI-Stufe fällt der Flow auf `Sonstiges` + Originalname zurück, damit nichts verloren geht.

## Voraussetzungen

- n8n läuft (self-hosted)
- Vier Credentials in n8n hinterlegt:
  - **Microsoft Outlook OAuth2**
  - **Microsoft SharePoint OAuth2**
  - **Anthropic API** (Bezahl-API, separat von Claude Pro)
  - **Telegram Bot API**
- SharePoint Site und Basis-Pfad `/<site>/DMS/Mail Anhänge/` müssen existieren. Die Kategorie-Unterordner werden automatisch angelegt, falls sie fehlen.
- Drei Outlook-Folder eingerichtet: Eingang (zu verarbeiten), `n8n-erfolgreich`, `n8n-fehler`.

## Import und Anpassung

1. Import via **Workflows → Import from File** → `workflow.json`.
2. In der JSON sind die folgenden **Platzhalter** zu ersetzen (entweder vor dem Import per Editor oder nach dem Import in den jeweiligen Nodes):

| Platzhalter | Wo | Was reinschreiben |
|---|---|---|
| `<TENANT>` | URLs in den HTTP-Nodes "Sharepoint Ordner erstellen…" und "SharePoint Upload" | Eigener M365 Tenant, z.B. `contoso` aus `contoso.sharepoint.com` |
| `<SITE>` | gleiche zwei Nodes | Site-Name, z.B. `MeineSite` aus `/sites/MeineSite` |
| `<TELEGRAM_CHAT_ID>` | Node "Send a text message" | Chat-ID des Empfängers, z.B. von `@RawDataBot` |
| `<OUTLOOK_INBOX_FOLDER_ID>` | Node "Mails mit Anhängen holen" | Folder-ID des zu pollenden Eingangs |
| `<OUTLOOK_ERROR_FOLDER_ID>` | Node "Mail -> n8n-fehler" | Folder-ID für Fehler-Mails |
| `<OUTLOOK_SUCCESS_FOLDER_ID>` | Node "Mail -> n8n-erfolgreich" | Folder-ID für erfolgreich verarbeitete Mails |
| `<ERROR_WORKFLOW_ID>` | Settings → `errorWorkflow` | (Optional) ID eines Error-Handler-Workflows, sonst Feld entfernen |
| `<INSTANCE_ID>` | `meta.instanceId` | n8n setzt das beim Import selber, einfach lassen |

> **Outlook Folder-IDs finden:** Im Outlook-Node `messageFolder → getAll` ausführen, dann liefert n8n die IDs als JSON. Oder in der M365 Admin Graph Explorer GUI.

3. In jedem Node die hinterlegten Credentials neu auswählen (nach Import zeigen sie auf nicht-existierende IDs).

4. Trigger: Standardmäßig schaltet der Schedule-Trigger jede Sekunde — das ist für Tests gedacht. **Vor dem produktiven Aktivieren auf einen vernünftigen Wert setzen** (z.B. alle 5 Minuten), sonst hämmerst du sowohl Outlook als auch die Anthropic API.

## Kategorien

Fest in den Claude-Prompt geschrieben:

```
Rechnungen, Verträge, Angebote, Lieferscheine,
Kontoauszüge, Versicherungen, Behörden, Sonstiges
```

Wenn Claude eine offensichtlich passende andere Kategorie erkennt, darf er sie verwenden — der entsprechende Ordner wird in SharePoint dann automatisch erzeugt. Wenn das nicht erwünscht ist, im Prompt-String des Nodes "Dokument analysieren (Claude)" das `wenn du selbst eine erkennst diese nehmen` rauslöschen.

## Bekannte Schwächen / Roadmap

- **Multi-Attachment-Mails**: Aktuell wird pro Lauf nur die erste Mail (`limit: 1`) und davon alle Anhänge nacheinander via SplitInBatches verarbeitet. Wenn mehrere Mails parallel kommen, dauern die Durchläufe. Skalierung über die Schedule-Frequenz oder durch Erhöhen des Limits.
- **PDF-Encrypt-Check** ist nur eine String-Suche. Echte PDFs ohne `/Encrypt` aber mit ungewöhnlicher Struktur könnten fälschlich als offen erkannt werden — Claude würde dann mit gutem Buffer trotzdem laufen oder sauber failen.
- **Kosten**: Sonnet ist nicht billig. Für Standard-Rechnungen reicht Haiku problemlos — im Node "Dokument analysieren (Claude)" das Modell auf `claude-haiku-4-5-20251001` umstellen.

## Node-Übersicht

```
Schedule / Manueller Start
        │
        ▼
Mails mit Anhängen holen ──► Alle Attachments extrahieren ──► SplitInBatches
                                                                     │
                                                                     ▼
                                                                 Ist PDF?
                                                                 ├─ ja ──► PDF öffenbar? ──► PDF offen? ──┐
                                                                 │                                          │
                                                                 └─ nein ──► Kein PDF durchreichen ─────────┤
                                                                                                            ▼
                                                                                              Dokument analysieren (Claude)
                                                                                                            │
                                                                                                            ▼
                                                                                          KI erfolgreich / KI fehlgeschlagen
                                                                                                            │
                                                                                                            ▼
                                                                                              SharePoint Folder anlegen
                                                                                                            │
                                                                                                            ▼
                                                                                                  Upload vorbereiten
                                                                                                            │
                                                                                                            ▼
                                                                                                   SharePoint Upload
                                                                                                            │
                                                                                              ┌─────────────┴────────────┐
                                                                                              ▼                          ▼
                                                                                         Upload ok               Upload fehlgeschlagen
                                                                                              │                          │
                                                                                              └────► SplitInBatches ◄────┘
                                                                                                            │
                                                                                                  (nach letztem Item)
                                                                                                            ▼
                                                                                              Ergebnisse sammeln
                                                                                                            │
                                                                                                            ▼
                                                                                              📋 Zusammenfassung
                                                                                                            │
                                                                                              ┌─────────────┴───────────────┐
                                                                                              ▼                              ▼
                                                                                       Kritischer Fehler?           Telegram senden
                                                                                              │
                                                                                  ┌───────────┴────────────┐
                                                                                  ▼                        ▼
                                                                          Mail → n8n-fehler        Mail → n8n-erfolgreich
```
