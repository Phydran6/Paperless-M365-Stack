# Setup

Schritt-für-Schritt Installation des Stacks. Voraussetzung: eine Debian 13 VM mit Docker + Docker Compose und Root/sudo-Zugriff. Nginx Proxy Manager und PiHole laufen bereits (in dieser Anleitung nicht enthalten).

## 0. Voraussetzungen

- [x] Debian 13 VM mit Docker und Docker Compose v2
- [x] Nginx Proxy Manager läuft im selben Docker-Netzwerk oder erreichbar
- [x] PiHole + DynDNS-Eintrag für die gewünschte Domain (`dms.example.com`)
- [x] M365 Account mit Berechtigung für Entra App Registrations
- [x] SharePoint Site eingerichtet, in der die Ordner-Struktur angelegt werden darf

## 1. DNS und Reverse Proxy

### PiHole DNS Rewrite

Lokal auflösen, ohne Hairpin über den Router:

```
dms.phytech.de   →   192.168.179.<vm-ip>
ocr.phytech.de   →   192.168.179.<vm-ip>
```

In PiHole: **Local DNS → DNS Records**.

### NPM Proxy Host

Im Nginx Proxy Manager einen neuen Proxy Host anlegen:

- **Domain Names:** `dms.phytech.de`
- **Forward Hostname / IP:** `paperless_web` (Container-Name) oder die interne VM-IP
- **Forward Port:** `8000`
- **Block Common Exploits:** ✓
- **Websockets Support:** ✓
- **SSL → Request a new SSL Certificate** mit Let's Encrypt
- **Force SSL:** ✓
- **HTTP/2 Support:** ✓

> Wichtig: NPM und Paperless müssen im selben Docker-Netzwerk hängen, damit der Container-Name aufgelöst wird. Falls separate Netze, vorher mit `docker network connect <npm-net> paperless_web` verbinden.

## 2. Paperless-NGX

`docker-compose.example.yml` und `.env.example` aus diesem Ordner als Vorlage nehmen.

```bash
cd ~/pngx
cp /pfad/zum/repo/setup/docker-compose.example.yml docker-compose.yml
cp /pfad/zum/repo/setup/.env.example .env
# .env editieren – Secrets, Admin-Passwort, Zeitzone, etc.
docker compose up -d
```

Erste Anmeldung unter `https://dms.phytech.de` mit den Credentials aus `.env`.

## 3. M365 OAuth2 für Paperless

Damit Paperless direkt aus M365 Mails liest, ohne IMAP-Passwort.

### Entra App Registration

1. Im **Microsoft Entra Admin Center** → **App registrations** → **New registration**.
2. Name: `Paperless DMS Mail Reader` (oder beliebig).
3. **Supported account types:** Single tenant.
4. **Redirect URI:** `Web` → `https://dms.phytech.de/accounts/microsoft/login/callback/`
5. Nach dem Anlegen:
   - **Client ID** notieren
   - **Endpoints → OAuth 2.0 authorization endpoint (v2)** notieren (Tenant ID enthalten)
   - **Certificates & secrets → New client secret** → Wert sofort kopieren (wird nur einmal angezeigt)
6. **API permissions:**
   - `Microsoft Graph` → `Delegated`:
     - `User.Read`
     - `Mail.Read`
     - `Mail.ReadWrite` (falls Paperless Mails verschieben/markieren soll)
     - `offline_access`
   - **Grant admin consent** klicken (sonst muss jeder Nutzer selbst zustimmen).

### Paperless konfigurieren

In `docker-compose.env` (oder `.env`) ergänzen:

```env
PAPERLESS_APPS=allauth.socialaccount.providers.microsoft
PAPERLESS_SOCIALACCOUNT_PROVIDERS={"microsoft":{"APPS":[{"provider_id":"microsoft","name":"Microsoft","client_id":"<CLIENT_ID>","secret":"<CLIENT_SECRET>","settings":{"tenant":"<TENANT_ID>"}}],"SCOPE":["User.Read","Mail.Read","offline_access"]}}
PAPERLESS_URL=https://dms.phytech.de
PAPERLESS_CSRF_TRUSTED_ORIGINS=https://dms.phytech.de
```

Container neu starten:

```bash
docker compose down && docker compose up -d
```

In Paperless dann unter **Settings → Mail** eine neue Mail-Account-Verbindung anlegen, OAuth-Login durchklicken, Mail Rules definieren (Folder, Filter, Action).

## 4. Service-Übersicht

Nach erfolgreichem Setup laufen auf der VM:

```
docker ps
```

sollte mindestens zeigen:

- `paperless_web`, `paperless_db` (postgres), `paperless_broker` (redis)
- (optional `paperless_gotenberg`, `paperless_tika` für Office-Files)
- `nginx-proxy-manager`
- `n8n` (siehe eigene Doku, hier nicht enthalten)
- `stirling-pdf` (siehe eigene Doku, hier nicht enthalten)

## 5. Nächster Schritt

→ [`../automation/README.md`](../automation/README.md) für rclone und Crontabs
→ [`../flows/README.md`](../flows/README.md) für die n8n Workflows
