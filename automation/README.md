# Automation – rclone & Crontab

Diese beiden Crontabs erledigen den eigentlichen SharePoint-Sync, ohne dass n8n involviert ist.

## Was läuft hier?

| Job | Zweck | Frequenz |
|---|---|---|
| `rclone move` | Dateien aus dem SharePoint-Ordner `Mail Anhänge` (von n8n dort abgelegt) holen und in den Paperless Consume-Ordner schieben | jede Minute |
| `document_exporter` + `rclone copy` | Paperless macht einen vollständigen Export, der wird ins SharePoint `Archiv` gespiegelt | täglich 02:00 |

`move` (statt `copy`) für den Eingang: SharePoint wird leergeräumt, sobald die Datei im Consume-Ordner ist. Paperless nimmt sie dann auf und tagged.

## 1. rclone installieren

```bash
sudo -v ; curl https://rclone.org/install.sh | sudo bash
```

## 2. SharePoint Remote konfigurieren

```bash
rclone config
```

Schritte:

1. `n` für neuen Remote
2. Name: `sharepoint`
3. Storage: `onedrive`
4. Client ID / Secret: leer lassen für Standardwerte
5. Region: `1` (Microsoft Cloud Global) oder passend
6. **Use auto config:** Bei Headless-Server `n` und auf einem Rechner mit Browser `rclone authorize "onedrive"` ausführen, Token zurückkopieren.
7. Type: `2` (SharePoint site)
8. SharePoint site URL eingeben: `https://<tenant>.sharepoint.com/sites/<site>`
9. Drive auswählen (meist `1`)
10. Bestätigen, fertig.

`rclone.conf.example` in diesem Ordner zeigt das Schema. Die echte Datei liegt unter `~/.config/rclone/rclone.conf` und gehört **nicht** ins Repo.

### Verbindung testen

```bash
rclone lsd sharepoint:
rclone lsd "sharepoint:Mail Anhänge"
rclone lsd "sharepoint:Archiv"
```

## 3. Crontab einrichten

```bash
crontab -e
```

Inhalt aus [`crontab.example`](crontab.example) übernehmen und Pfade anpassen:

```cron
# SharePoint Mail Anhänge → Paperless Consume (alle 60 Sekunden)
* * * * * /usr/bin/rclone move "sharepoint:Mail Anhänge" /home/docker2/pngx/consume

# Paperless Export → SharePoint Archiv (täglich 02:00)
0 2 * * * docker exec paperless_web document_exporter ../export && /usr/bin/rclone copy /home/docker2/pngx/export "sharepoint:Archiv"
```

### Was passiert hier genau?

**Job 1 (jede Minute):** `rclone move` schiebt alles aus `sharepoint:Mail Anhänge` in den lokalen Consume-Ordner und löscht es im SharePoint. Paperless überwacht den Consume-Ordner und nimmt Dateien automatisch auf — OCR, Tagging, Korrespondent-Erkennung passieren dann in Paperless.

**Job 2 (nachts):**
- `docker exec paperless_web document_exporter ../export` ruft das Paperless-eigene Export-Tool auf. Es schreibt eine vollständige Sicherung (originale Dateien + Manifest mit Metadaten + DB-Inhalten) in `/home/docker2/pngx/export/`.
- `rclone copy` (nicht `move`!) kopiert das nach `sharepoint:Archiv`. So bleibt die lokale Kopie erhalten als Stand-By.

## 4. Logging und Monitoring

Cron schreibt standardmäßig per Mail an den User. Wenn kein Mailserver läuft, in den Crontab umleiten:

```cron
* * * * * /usr/bin/rclone move "sharepoint:Mail Anhänge" /home/docker2/pngx/consume >> /var/log/rclone-mail.log 2>&1
0 2 * * * (docker exec paperless_web document_exporter ../export && /usr/bin/rclone copy /home/docker2/pngx/export "sharepoint:Archiv") >> /var/log/paperless-export.log 2>&1
```

Logrotation nicht vergessen, sonst füllen die Dateien irgendwann die Disk.

## 5. Troubleshooting

| Symptom | Ursache | Fix |
|---|---|---|
| `rclone move` läuft, aber Datei taucht nicht im Consume auf | Dateirechte / Pfad falsch | `ls -la /home/docker2/pngx/consume` und prüfen, ob Cron-User Schreibrechte hat |
| Paperless verarbeitet nichts | Consume-Ordner nicht in den Container gemountet | `docker compose down && docker compose up -d` mit korrektem Mount |
| `document_exporter` Fehler `No such file or directory` | `../export` existiert im Container nicht | Im Compose ist `./export:/usr/src/paperless/export` gemappt; Pfad relativ zum WorkDir `/usr/src/paperless/src/`, also `../export` ist korrekt |
| `rclone` 401 / Token expired | OAuth Token abgelaufen | `rclone config reconnect sharepoint:` |
| SharePoint-Pfade mit Umlauten | rclone braucht UTF-8 | Path in Quotes setzen: `"sharepoint:Mail Anhänge"` |
