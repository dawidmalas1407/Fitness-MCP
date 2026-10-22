# Fitness-MCP

Selbst gehostete [MCP](https://modelcontextprotocol.io/)-Server für **Garmin Connect** und **Yazio**, gedacht für einen Raspberry Pi (oder jeden anderen Linux-Rechner mit Docker). Über einen Tailscale Funnel sind sie per HTTPS aus dem Internet erreichbar und lassen sich so als Custom Connectors in Claude (oder einem anderen MCP-Client) einbinden.

> **Neu hier?** Die [Installationsanleitung für Einsteiger](ANLEITUNG.md) führt dich Klick für Klick durch die Einrichtung und zeigt bei jedem Schritt, was du sehen solltest. Diese README ist die technische Referenz.

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
| `garmin-mcp-server/` | Garmin-Server, basiert auf [bmccarn/garmin-mcp-server](https://github.com/bmccarn/garmin-mcp-server). Die dortige `README.md` beschreibt den Standalone-Betrieb, hier startet ihn die `docker-compose.yml` im Hauptordner |
| `yazio/` | Dockerfile für den Yazio-Server ([`yazio-mcp`](https://www.npmjs.com/package/yazio-mcp) 0.0.14 über [`supergateway`](https://www.npmjs.com/package/supergateway) 4.2.0, Versionen fest eingetragen) |
| `garmin-tokens/` | *(nicht im Repo)* Garmin-OAuth-Tokens, wird beim Einrichten erzeugt |
| `.env` | *(nicht im Repo)* Yazio-Login und MCP-Token |

> **Geheimnisse:** `.env` und `garmin-tokens/` gehören nie ins Repo und nie in einen Chat. Beide stehen in der `.gitignore`.

---

## Voraussetzungen

- Raspberry Pi 4/5 mit **Raspberry Pi OS Lite 64-bit** (oder ein anderer Linux-Rechner, arm64 oder amd64)
- Ein Garmin-Connect-Konto und ein Yazio-Konto
- Ein kostenloses [Tailscale](https://tailscale.com/)-Konto
- Am besten ein **LAN-Kabel**: Bei schwachem WLAN sind die Connectoren immer wieder unerreichbar (siehe [Fehlersuche](#fehlersuche)).

### Welche Zugangsdaten du wann brauchst

| Schritt | Was | Woher | Wo es landet |
|---|---|---|---|
| 3 | GitHub-Login, nur bei privatem Repo | `gh auth login` oder Personal Access Token | in `gh` auf dem Pi |
| 4 | **MCP-Token** (Bearer-Token) | **selbst erzeugt** mit `openssl rand -hex 32` | `.env` |
| 4 | Yazio-E-Mail und -Passwort | dein normales Yazio-Konto | `.env` |
| 6 | **Garmin-Tokens** | einmaliger Login mit Garmin-E-Mail, Passwort und ggf. SMS-Code über `garmin_auth.py` | `garmin-tokens/` |
| 8 | Tailscale-Anmeldung | Link aus `sudo tailscale up` im Browser öffnen | intern bei Tailscale |
| 9 | derselbe MCP-Token | aus der `.env`, als `Bearer <MCP_TOKEN>` | in beiden Claude-Connectoren |

Den MCP-Token stellt dir niemand aus. Du erzeugst ihn selbst, und Caddy und Claude kennen danach denselben Wert. Das Garmin-Passwort wird nirgends gespeichert, nur die Tokens, die etwa ein Jahr halten. Yazio meldet sich bei jedem Start selbst mit den Daten aus der `.env` an.

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

Ist das Repository **privat**, fragt `git` nach Benutzername und Passwort. Ein GitHub-Passwort funktioniert dort nicht. Am einfachsten meldest du dich vorher mit der GitHub-CLI an:

```bash
sudo apt install -y gh
gh auth login          # GitHub.com → HTTPS → Login with a web browser
gh repo clone dawidmalas1407/fitness-mcp ~/fitness-mcp
cd ~/fitness-mcp
```

Alternativ legst du auf GitHub unter *Settings → Developer settings → Personal access tokens* einen Token mit Lesezugriff auf das Repository an und gibst ihn bei `git clone` als Passwort ein.

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

Bevor du anfängst, halte den Token bereit. Du brauchst ihn gleich, und Claude zeigt ihn nach dem Speichern nicht mehr an:

```bash
grep MCP_TOKEN ~/fitness-mcp/.env
```

Du legst zwei Connectoren an, einen pro Server:

| Name | Remote MCP server URL |
|---|---|
| `Garmin` | `https://<hostname>.<tailnet>/garmin/mcp` |
| `Yazio` | `https://<hostname>.<tailnet>/yazio/mcp` |

#### 9.1 Connector anlegen (Free, Pro, Max)

Am einfachsten geht das im Browser auf [claude.ai](https://claude.ai) oder in der Desktop-App. Für jeden der beiden Connectoren:

1. Öffne [**Customize → Connectors**](https://claude.ai/customize/connectors).
2. Klicke auf **Add custom connector**.
3. Trag **Name** und **Remote MCP server URL** aus der Tabelle oben ein.
4. Wähle bei **Authentication** die Option **No sign-in**. Die Server haben kein OAuth, der Schutz läuft über den Token.
5. Öffne **Request headers** und lege einen Header an:
   - **Header name:** `authorization` aus der Liste
   - **Value:** `Bearer <MCP_TOKEN>`, also das Wort `Bearer`, ein Leerzeichen und dann der Token
   - **Required:** an
6. Klicke auf **Add**.

> **Wichtig:** Claude schickt den Wert genau so, wie du ihn eingibst. Das `Bearer ` davor musst du deshalb selbst mit eintragen. Fehlt es, antwortet Caddy mit `401` und der Connector funktioniert nicht.

Je nach Konto sieht der Dialog leicht anders aus: Manchmal hat er zwei Schritte, manchmal stehen die Optionen unter **Advanced settings**. Die Feldnamen sind dieselben.

Bei **Team und Enterprise** legt ein Owner die Connectoren unter [**Organization settings → Connectors**](https://claude.ai/admin-settings/connectors) an, über **Add → Custom → Web**, mit denselben Angaben. Mitglieder finden sie danach unter *Customize → Connectors* mit dem Label **Custom** und klicken auf **Connect**. Alle Mitglieder nutzen dabei denselben Token und sehen damit dieselben Garmin- und Yazio-Daten.

#### 9.2 Wenn „Request headers“ fehlt

Der Abschnitt **Request headers** ist laut [Claude-Doku](https://claude.com/docs/connectors/custom/remote-mcp#authenticate-with-request-headers) eine Beta und noch nicht für alle Konten freigeschaltet. Fehlt er bei dir, gibt es mit diesem Setup keinen sicheren Weg, den Connector einzurichten. Lass in dem Fall den Caddy-Schutz **nicht** weg, denn sonst kann jeder mit der URL deine Gesundheitsdaten lesen. Warte, bis die Funktion freigeschaltet ist, oder nutze vorerst Claude Code (siehe 9.5).

#### 9.3 Nachträglich ändern

Die Authentifizierung, also auch den Token, kannst du nach dem Anlegen nicht mehr bearbeiten. Hast du dich vertippt oder einen neuen Token erzeugt, entferne den Connector unter *Customize → Connectors* mit **Remove** und lege ihn neu an.

#### 9.4 Im Chat benutzen

- Die Connectoren gelten für dein ganzes Claude-Konto, also auch für die Desktop-App und die Handy-App mit demselben Login.
- Pro Chat schaltest du sie über **+ → Connectors** ein oder aus.
- Beim ersten Tool-Aufruf fragt Claude nach einer Freigabe. Wähle **Always allow** nur, wenn du dem Server vertraust. Hier ist es dein eigener.
- Unter *Customize → Connectors → Yazio* kannst du einzelne Tools auf **Blocked** setzen, zum Beispiel die Schreib-Tools `add_user_consumed_item`, `add_user_water_intake` und `remove_user_consumed_item`, wenn Claude nichts in Yazio eintragen soll.

**Test:** Schreib in einem neuen Chat zum Beispiel *„Welche Garmin-Geräte habe ich?“* (Tool `get_devices`) und *„Was sind meine Yazio-Ziele?“* (Tool `get_user_goals`). Wenn der allererste Aufruf mit „Error dialing …“ scheitert, versuch es einfach noch einmal.

#### 9.5 Optional: Claude Code

Für die Kommandozeile von Claude Code fügst du die Server so hinzu:

```bash
claude mcp add --transport http garmin https://<hostname>.<tailnet>/garmin/mcp --header "Authorization: Bearer <MCP_TOKEN>"
claude mcp add --transport http yazio  https://<hostname>.<tailnet>/yazio/mcp  --header "Authorization: Bearer <MCP_TOKEN>"
```

#### 9.6 Prüfen, dass der Schutz greift

Ohne Token müssen beide Pfade gesperrt sein:

```bash
curl -i http://localhost:8080/garmin/mcp     # Erwartung: 401
curl -i http://localhost:8080/yazio/mcp      # Erwartung: 401
```

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

Die npm-Pakete für Yazio sind in `yazio/Dockerfile` auf feste Versionen gesetzt, damit ein neues Release nicht unbemerkt etwas kaputt macht. Für ein Update der Pakete passt du die Versionsnummern dort an (aktuelle Versionen zeigt `npm view yazio-mcp version` bzw. `npm view supergateway version`) und baust neu. Beim Garmin-Server legt das `garmin-mcp-server/requirements.txt` die Versionen fest.

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
| Im Caddy-Log stehen `401` für Anfragen von Claude | Der Pfad stimmt, aber der Token fehlt oder ist falsch. Meist fehlt im Header-Wert das `Bearer ` davor. Entferne den Connector und leg ihn neu an (siehe 9.3). Stehen gar keine Zeilen im Log, ist die URL falsch oder Funnel bzw. Netz sind down. |
| Token womöglich geleakt | Erzeuge einen neuen `MCP_TOKEN` in `.env`, führe `sudo docker compose up -d` aus und leg beide Connectoren mit dem neuen Token neu an (siehe 9.3). |
| Yazio meldet einen Login-Fehler | Prüfe die Zugangsdaten in `.env` und führe dann `sudo docker compose up -d yazio` aus. Eventuell musst du den Connector in Claude neu verbinden. |

### Bekannte Eigenheiten

- **Garmin-Schlafzeiten** können um die Zeitzonen-Differenz verschoben sein. Das ist ein Zeitzonenfehler im Server, die Dauer selbst stimmt.
- `get_devices` liefert `last_synced: null`. Das ist normal und kein Fehler.
- Der Garmin-Server rechnet Distanzen in Meilen um (Upstream-Verhalten).
- Die Yazio-Schreib-Tools (`add_user_consumed_item`, `add_user_water_intake`, `remove_user_consumed_item`) ändern echte Daten in deinem Konto.
