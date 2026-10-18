# Fitness-MCP

Selbst gehostete [MCP](https://modelcontextprotocol.io/)-Server für **Garmin Connect** und **Yazio**, gedacht für einen Raspberry Pi (oder jeden anderen Linux-Rechner mit Docker). Über einen Tailscale Funnel sind sie per HTTPS aus dem Internet erreichbar und lassen sich so als Custom Connectors in Claude (oder einem anderen MCP-Client) einbinden.

```
MCP-Client (Cloud) ─HTTPS─> Tailscale Funnel ─> 127.0.0.1:8080  Caddy (prüft Bearer-Token)
                                                  ├─ /garmin/* ─> garmin (fastmcp, :8000)
                                                  └─ /yazio/*  ─> yazio  (Node,    :8001)
```

| Pfad | Zweck |
|---|---|
| `docker-compose.yml` | Dienste `garmin`, `yazio`, `caddy`. Caddy lauscht nur auf `127.0.0.1:8080` |
| `Caddyfile` | Routing und Token-Prüfung (ohne gültigen Token kommt `401`) |
| `.env.example` | Vorlage für die `.env` mit den Geheimnissen |
| `garmin-mcp-server/` | Garmin-Server, basiert auf [bmccarn/garmin-mcp-server](https://github.com/bmccarn/garmin-mcp-server) |
| `yazio/` | Dockerfile für den Yazio-Server ([`yazio-mcp`](https://www.npmjs.com/package/yazio-mcp) über `supergateway`) |
| `garmin-tokens/` | *(nicht im Repo)* Garmin-OAuth-Tokens, wird beim Einrichten erzeugt |
| `.env` | *(nicht im Repo)* Yazio-Login und MCP-Token |

> **Geheimnisse:** `.env` und `garmin-tokens/` gehören nie ins Repo und nie in einen Chat. Beide stehen in der `.gitignore`.

---

## Voraussetzungen

- Raspberry Pi 4/5 mit **Raspberry Pi OS Lite 64-bit** (oder ein anderer Linux-Rechner, arm64 oder amd64)
- Ein Garmin-Connect-Konto und ein Yazio-Konto
- Ein kostenloses [Tailscale](https://tailscale.com/)-Konto
- Am besten ein **LAN-Kabel**: Bei schwachem WLAN sind die Connectoren immer wieder unerreichbar (siehe [Fehlersuche](#fehlersuche)).

In der Anleitung stehen Platzhalter:

| Platzhalter | Bedeutung |
|---|---|
| `<user>` | Benutzername auf dem Pi |
| `<hostname>` | Hostname des Pi, z. B. `raspberrypi` |
| `<tailnet>` | Name deines Tailnets, z. B. `tailXXXXXX.ts.net` |

---

## Einrichtung

### 1. Pi vorbereiten

Flashe mit dem **Raspberry Pi Imager** *Raspberry Pi OS Lite (64-bit)*. Lege in den Einstellungen des Imagers Hostname, Benutzer und Passwort fest, aktiviere SSH und trage gegebenenfalls das WLAN ein.

Vom PC aus verbinden (PowerShell, Terminal):

```bash
ssh <user>@<hostname>.local
```

Wenn der Pi neu geflasht wurde und eine Warnung `REMOTE HOST IDENTIFICATION HAS CHANGED` erscheint, entferne zuerst den alten Schlüssel:

```bash
ssh-keygen -R <hostname>.local
```

### 2. System aktualisieren und Docker installieren (auf dem Pi)

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
sudo docker --version
sudo docker compose version
```

Alle Docker-Befehle in dieser Anleitung laufen mit `sudo`.

### 3. Repository klonen

```bash
cd ~
git clone https://github.com/dawidmalas1407/fitness-mcp.git
cd ~/fitness-mcp
```

### 4. `.env` anlegen

```bash
cp .env.example .env
chmod 600 .env
openssl rand -hex 32        # Ausgabe als MCP_TOKEN verwenden
nano .env
```

Inhalt:

```ini
YAZIO_USERNAME=deine-yazio-email@example.com
YAZIO_PASSWORD=dein-yazio-passwort
MCP_TOKEN=<die 64 Zeichen aus openssl rand>
```

Der `MCP_TOKEN` schützt beide Server. Wer ihn kennt, kann deine Gesundheitsdaten lesen und in Yazio Einträge anlegen. Behandle ihn deshalb wie ein Passwort.

### 5. Container bauen

```bash
sudo docker compose build
```

Auf einem Pi dauert der erste Build einige Minuten.

### 6. Garmin einmalig anmelden (Tokens erzeugen)

Der Garmin-Server braucht OAuth-Tokens in `garmin-tokens/`. Erzeuge sie einmalig mit dem mitgelieferten Skript, das in einem temporären Container läuft:

```bash
mkdir -p garmin-tokens
sudo docker compose run --rm -it garmin python garmin_auth.py
```

Das Skript fragt nach Garmin-E-Mail und Passwort und gegebenenfalls nach dem MFA-Code per SMS. Danach liegen dort `oauth1_token.json` und `oauth2_token.json`:

```bash
ls -la garmin-tokens
```

Die Tokens halten etwa ein Jahr. Danach führst du den Befehl einfach noch einmal aus.

**Falls der Login mit `429 Too Many Requests` scheitert**, blockt Cloudflare den automatischen Login. Dann nimmst du die Browser-Variante auf einem PC mit Browser und Python:

```bash
pip install -r garmin-mcp-server/requirements.txt
python garmin-mcp-server/garmin_browser_auth.py
```

Das Skript speichert die Tokens unter `~/.garminconnect/` auf dem PC. Kopiere sie anschließend auf den Pi:

```bash
scp ~/.garminconnect/oauth*_token.json <user>@<hostname>.local:~/fitness-mcp/garmin-tokens/
```

Weitere Details stehen in [`garmin-mcp-server/README.md`](garmin-mcp-server/README.md#browser-assisted-login-cloudflare-fallback).

### 7. Stack starten

```bash
sudo docker compose up -d
sudo docker compose ps
curl -i http://localhost:8080
```

Erwartete Ausgabe:
- `caddy`, `garmin` und `yazio` sind alle `Up`.
- `curl` liefert `HTTP/1.1 401 Unauthorized`. Das ist richtig, denn die Anfrage hatte keinen Token.

Mit Token kannst du lokal so testen:

```bash
source .env
curl -i -H "Authorization: Bearer $MCP_TOKEN" http://localhost:8080/garmin/mcp
```

Hier sollte kein `401` mehr kommen. Eine Fehlermeldung vom MCP-Server selbst, etwa `406` oder `400`, ist normal, weil `curl` kein MCP spricht.

### 8. Tailscale und Funnel einrichten

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up            # angezeigten Link im Browser öffnen und anmelden
sudo tailscale funnel --bg 8080
sudo tailscale funnel status
```

Erwartete Ausgabe:

```
https://<hostname>.<tailnet> (Funnel on)
|-- / proxy http://127.0.0.1:8080
```

Falls Tailscale Funnel noch nicht freigeschaltet ist, zeigt der Befehl einen Link, über den du es in der Tailscale-Admin-Konsole aktivierst.

### 9. In Claude einbinden

Lege zwei Custom Connectors an:

| Connector | URL |
|---|---|
| Garmin | `https://<hostname>.<tailnet>/garmin/mcp` |
| Yazio | `https://<hostname>.<tailnet>/yazio/mcp` |

Beide brauchen den `MCP_TOKEN` aus der `.env`, der bei jeder Anfrage als `Authorization: Bearer <MCP_TOKEN>` mitgeschickt werden muss.

Für einen schnellen Funktionstest frage Claude nach deinen Garmin-Geräten (Tool `get_devices`) oder deinen Yazio-Zielen (Tool `get_user_goals`).

### 10. Autostart prüfen

Alle Dienste haben `restart: unless-stopped`. Nach einem Neustart oder Stromausfall kommt der Stack also von selbst wieder hoch, solange Docker und Tailscale beim Booten starten:

```bash
sudo systemctl is-enabled docker tailscaled    # Erwartung: zweimal "enabled"
```

---

## Betrieb

Alle Befehle laufen in `~/fitness-mcp`:

```bash
sudo docker compose ps                 # Status
sudo docker compose logs -f caddy      # kommen Anfragen an?
sudo docker compose logs -f garmin
sudo docker compose restart            # alles neu starten
sudo docker compose up -d --build      # nach Änderungen neu bauen
curl -i http://localhost:8080          # Erwartung: 401
sudo tailscale funnel status           # Erwartung: Funnel on
```

**Gesund heißt:** Alle drei Container sind `Up`, `curl` liefert `401`, und der Funnel steht auf `on`.

### Update

```bash
cd ~/fitness-mcp
git pull
sudo docker compose up -d --build
```

### Backup

Geheimnisse liegen nur in `.env` und `garmin-tokens/`, es gibt keine Docker-Volumes. Ein Backup des Projektordners reicht daher:

```bash
sudo tar czf ~/fitness-mcp-backup.tgz -C ~ fitness-mcp
```

Das Archiv enthält Zugangsdaten. Bewahre es also nur privat auf und lade es nirgends hoch. Zum Wiederherstellen auf einem neuen System nach Schritt 2:

```bash
tar xzf ~/fitness-mcp-backup.tgz -C ~
cd ~/fitness-mcp && sudo docker compose up -d --build
```

Bei einem Neuaufbau solltest du die alte Maschine vorher in der Tailscale-Admin-Konsole (*Machines*) löschen. Dann bekommt der neue Pi denselben Hostnamen, und die Connector-URLs bleiben gleich.

---

## Fehlersuche

| Symptom | Ursache / Lösung |
|---|---|
| `permission denied … docker.sock` | Docker-Befehle mit `sudo` ausführen. |
| Build-Fehler `additional privileges requested: pass "--allow=network.host"` | Im `build:`-Abschnitt steht `network: host`. Entferne die Zeile, in diesem Repo ist sie bereits entfernt. `COMPOSE_BAKE=false` hilft nicht. |
| `curl: (7) Failed to connect to localhost port 8080` | Caddy läuft nicht. Prüfe `sudo docker compose ps` und die Logs. |
| Connectoren zeitweise unerreichbar, obwohl der Pi läuft | Häufig liegt es an instabilem WLAN. Prüfe mit `nmcli device status` und `iw dev wlan0 link`. Bei einem Signal unter etwa -75 dBm hilft ein LAN-Kabel oder ein kürzerer Abstand zum Router. |
| Test vom Handy klappt, Claude erreicht den Server aber nicht | Wenn das Handy selbst im Tailnet ist, läuft der Test auch ohne Funnel. Prüfe deshalb `sudo tailscale funnel status` und in `logs caddy`, ob Anfragen von Claude ankommen. |
| Erster Garmin-Aufruf liefert kurz „Error dialing …“ | Kommt gelegentlich vor, ein zweiter Versuch klappt meist. |
| Garmin-Auth-Fehler | Tokens sind abgelaufen oder ungültig. Wiederhole Schritt 6 und führe dann `sudo docker compose restart garmin` aus. |
| Yazio meldet einen Login-Fehler | Prüfe die Zugangsdaten in `.env` und führe dann `sudo docker compose up -d yazio` aus. Eventuell musst du den Connector in Claude neu verbinden. |

### Bekannte Eigenheiten

- **Garmin-Schlafzeiten** können um die Zeitzonen-Differenz verschoben sein. Das ist ein Zeitzonenfehler im Server, die Dauer selbst stimmt.
- `get_devices` liefert `last_synced: null`. Das ist normal und kein Fehler.
- Der Garmin-Server rechnet Distanzen in Meilen um (Upstream-Verhalten).
- Die Yazio-Schreib-Tools (`add_user_consumed_item`, `add_user_water_intake`, `remove_user_consumed_item`) ändern echte Daten in deinem Konto.
