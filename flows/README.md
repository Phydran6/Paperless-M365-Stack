# n8n Workflows

Hier liegen die importierbaren n8n Workflows als JSON. Jeder Flow hat einen eigenen Unterordner mit README und der `workflow.json`.

## Aktuelle Flows

| Flow | Zweck |
|---|---|
| [`mail-attachments-to-sharepoint/`](mail-attachments-to-sharepoint/) | Holt Mail-Anhänge aus M365 Outlook, lässt Claude einen sinnvollen Dateinamen + Kategorie bestimmen, lädt das Ergebnis in den passenden SharePoint-Ordner und benachrichtigt per Telegram. |

## Import in n8n

1. In n8n links oben **Workflows → Import from File**.
2. Die `workflow.json` aus dem jeweiligen Unterordner auswählen.
3. **Vor der ersten Ausführung** alle Platzhalter ersetzen — siehe README im Flow-Ordner. Der Import schlägt nicht fehl, aber der Flow läuft nicht durch, solange Tenant, Folder-IDs, Chat-IDs und Credentials nicht stimmen.
4. Credentials in n8n anlegen (Microsoft Outlook OAuth2, Microsoft SharePoint OAuth2, Anthropic API, Telegram).
5. Workflow aktivieren.

## Benötigte n8n Credentials

| Credential-Typ | Wofür |
|---|---|
| Microsoft Outlook OAuth2 | Mails + Anhänge holen, Mails in Erfolgs-/Fehler-Folder verschieben |
| Microsoft SharePoint OAuth2 | Ordner anlegen + Files hochladen via REST API |
| Anthropic API | Claude für Dokumenten-Analyse |
| Telegram API | Statusmeldungen |

Alle vier müssen vor dem Import bzw. direkt nach dem Import in n8n unter **Credentials** hinterlegt werden. Die Credential-IDs in der JSON werden beim Import automatisch durch die eigenen ersetzt, sobald man die Credentials in den jeweiligen Nodes auswählt.
