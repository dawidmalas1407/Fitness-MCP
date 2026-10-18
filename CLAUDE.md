# CLAUDE.md – Self-hosted Garmin- & Yazio-MCP-Server auf einem Raspberry Pi

Kontext für Claude Code: Dieses Repo betreibt zwei MCP-Server (Garmin, Yazio) in Docker auf einem Raspberry Pi. Sie werden über Tailscale Funnel öffentlich erreichbar gemacht und in Claude als Custom Connectors eingebunden.

Platzhalter in spitzen Klammern ersetzen: `<PI_USER>`, `<PI_HOST>`, `<TAILNET_DOMAIN>`, `<TOKEN>`.

## Regeln für Claude Code

- Keine Geheimnisse ausgeben oder committen (`.env`, `garmin-tokens/`, Bearer-Token).
- Vor Änderungen an `docker-compose.yml` / `Caddyfile` eine `.bak`-Kopie anlegen.
- Befehle als kurze, kopierbare Codeblöcke angeben und die erwartete Ausgabe dazuschreiben.
- Docker-Befehle auf dem Pi mit `sudo`, sofern der Benutzer nicht in der Gruppe `docker` ist.

## Architektur

```
Claude (Cloud) ─HTTPS─> Tailscale Funnel ─> 127.0.0.1:8080 (Caddy, Bearer-Token-Check)
                                              ├─> garmin (fastmcp, :8000)   Pfad /garmin/mcp
                                              └─> yazio  (Node,    :8001)   Pfad /yazio/mcp
```

- Öffentliche URL: `https://<PI_HOST>.<TAILNET_DOMAIN>.ts.net`
- Connector-URL Garmin: `https://<PI_HOST>.<TAILNET_DOMAIN>.ts.net/garmin/mcp` 
- Connector-URL Yazio: `https://<PI_HOST>.<TAILNET_DOMAIN>.ts.net/yazio/mcp` (beide Pfade in der Praxis bestätigt)
- Caddy lauscht nur auf `127.0.0.1:8080`; auf die Wurzel-URL (`curl -i http://localhost:8080`) antwortet er mit `401`.
- Zugriffsschutz: Die Connectoren in Claude senden den Header `Authorization: Bearer <TOKEN>`; Caddy lehnt Anfragen ohne gültigen Token mit `401` ab. Kontrolle: `curl -i http://localhost:8080/garmin/mcp` (ohne Header) muss `401` liefern.

## Voraussetzungen

- Raspberry Pi (getestet auf Pi 5) mit Raspberry Pi OS Lite 64-bit, SSH aktiv
- Docker + Docker Compose
- Tailscale-Account mit aktiviertem Funnel
- Garmin-Connect-Konto und Yazio-Konto

## Projektstruktur

| Pfad | Zweck |
|---|---|
| `docker-compose.yml` | Dienste `garmin`, `yazio`, `caddy` (`caddy:2`) |
| `Caddyfile` | Routing und Bearer-Token-Prüfung |
| `.env` | Geheimnisse (Yazio-Zugangsdaten, Bearer-Token) – nicht committen |
| `garmin-mcp-server/` | Garmin-MCP-Server (`garmin_mcp_server.py`, `garmin_auth.py`, Dockerfile) |
| `garmin-tokens/` | Garmin-OAuth-Tokens (`oauth1_token.json`, `oauth2_token.json`), gemountet nach `/root/.garminconnect` – nicht committen |
| `yazio/` | Dockerfile des Yazio-MCP-Servers (Node) |

`.gitignore`:
```
.env
garmin-tokens/
*.tgz
*.bak
```

## Einrichtung

1. **Pi vorbereiten:** Raspberry Pi OS Lite 64-bit flashen (Imager: Hostname, Benutzer, WLAN/LAN, SSH aktivieren). Verbinden: `ssh <PI_USER>@<PI_HOST>.local`.
2. **System aktualisieren und Docker installieren:**
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   curl -fsSL https://get.docker.com | sudo sh
   sudo systemctl enable --now docker
   sudo docker compose version
   ```
3. **Tailscale installieren und anmelden:**
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
4. **Projekt bereitstellen** (Repo klonen bzw. Backup einspielen, siehe unten) und `.env` mit Zugangsdaten und Token anlegen.
5. **Garmin-Tokens erzeugen:** Das Auth-Skript (`garmin_auth.py`) schreibt im Container nach `/root/.garminconnect`; durch den Mount landen die Tokens in `garmin-tokens/`. (Dieser Weg ist noch nicht praktisch getestet.)
6. **Container bauen und starten:**
   ```bash
   cd <PROJEKTORDNER>
   sudo docker compose up -d --build
   sudo docker compose ps
   ```
7. **Funnel einrichten:**
   ```bash
   sudo tailscale funnel --bg 8080
   sudo tailscale funnel status
   ```
8. **In Claude als Custom Connector eintragen** (Einstellungen -> Connectoren -> benutzerdefinierten Connector hinzufügen): je eine URL pro Server (siehe Architektur). Einstellungen im Dialog:
   - Anmeldung: **keine Anmeldung** (kein OAuth).
   - Authentifizierung: **Bearer Token**. Ins Feld kommt der Wert von `MCP_TOKEN` aus der `.env`, ohne `Bearer`-Präfix. Claude sendet ihn als `Authorization: Bearer <TOKEN>`, so wie es das Caddyfile prüft.
9. **Test:** `curl -i http://localhost:8080` -> `401`; in Claude Garmin `get_devices` und Yazio `get_user_goals` aufrufen.

## Betrieb

```bash
sudo docker compose ps                # alle drei Container "Up"
sudo docker compose logs -f caddy     # eingehende Anfragen beobachten
sudo docker compose up -d --build     # neu bauen/starten
sudo docker compose restart
sudo tailscale funnel status          # "Funnel on", Proxy -> 127.0.0.1:8080
```

Gesund = drei Container `Up`, `curl` auf `localhost:8080` liefert `401`, Funnel ist `on`.

Autostart nach Stromausfall: `restart: unless-stopped` für alle Compose-Dienste, `docker` und `tailscaled` per systemd aktiviert (`sudo systemctl is-enabled docker tailscaled`).

## Backup und Neuaufbau

- Backup: `sudo tar czf ~/pi-backup.tgz <PROJEKTORDNER>`. Das Archiv enthält Geheimnisse und gehört nicht ins Git. Es sind keine Docker-Volumes nötig.
- Neuaufbau: Pi neu flashen (gleicher Benutzer, damit die Pfade passen), Docker und Tailscale installieren, Backup mit `sudo tar xzf ~/pi-backup.tgz -C /` einspielen, `sudo tailscale funnel --bg 8080`, `sudo docker compose up -d --build`.
- Soll die öffentliche URL gleich bleiben, die alte Maschine vorher im Tailscale-Admin (Machines) entfernen, damit der Hostname nicht `-1` bekommt.
- Nach dem Neuflashen ändert sich der SSH-Host-Key: auf dem PC `ssh-keygen -R <PI_HOST>.local`.

## Bekannte Stolpersteine

1. **Build-Fehler `additional privileges requested: pass "--allow=network.host"`:** `network: host` in einem `build:`-Abschnitt löst die Buildx/Bake-Freigabeabfrage aus; `COMPOSE_BAKE=false` hilft nicht zuverlässig. Lösung: die Zeile `network: host` aus den `build:`-Abschnitten der `docker-compose.yml` entfernen (im Repo bereits geschehen).
2. **Connectoren unerreichbar, Pi aber „an“:** Häufig instabiles WLAN. Prüfen mit `nmcli device status` und `iw dev wlan0 link`; ein LAN-Kabel behebt das dauerhaft. Im Caddy-Log sieht man, ob Anfragen von Claude ankommen: `401` = Pfad stimmt, Token fehlt oder ist falsch; keine Zeilen = falsche URL oder Funnel/Netz down.
3. **Tailscale-Test täuscht:** Ein Gerät im selben Tailnet erreicht den Pi auch ohne Funnel. Zum Testen der öffentlichen Erreichbarkeit ein Gerät außerhalb des Tailnets verwenden.
4. **Docker „permission denied … docker.sock“:** Befehl mit `sudo` ausführen oder den Benutzer zur Gruppe `docker` hinzufügen.
5. **Garmin-Auth-Fehler:** `garmin-tokens/` und `garmin_auth.py` prüfen, danach `sudo docker compose restart garmin`.
6. **Yazio „needs reconnect“:** In den Claude-Connector-Einstellungen neu verbinden.
7. **Garmin-Schlafzeiten** können um +2 Stunden verschoben geliefert werden (Zeitzonenfehler im Server); Auswertungen sollten das berücksichtigen.
8. Ein kurzer „Error dialing …“-Fehler beim ersten Tool-Aufruf kommt vor; ein erneuter Versuch hilft meist.

## Tools

**Garmin:** `get_devices`, `get_todays_summary`, `get_daily_stats`, `get_heart_rate`, `get_stress`, `get_all_day_stress`, `get_body_battery`, `get_hrv`, `get_spo2`, `get_respiration`, `get_sleep`, `get_sleep_quality_trends`, `get_recovery_metrics`, `get_training_readiness`, `get_training_status`, `get_recent_activities`, `get_last_activity`, `get_activities_by_type`, `search_activities`, `get_activity_details`, `get_personal_records`, `get_body_composition`, `get_weight_history`, `get_weekly_summary`, `get_weekly_health_report`, `get_health_data_for_date_range`, `compare_periods` u. a. Datumsformat `YYYY-MM-DD`, `today` ist erlaubt.

**Yazio:** Lesen: `get_user`, `get_user_goals`, `get_user_settings`, `get_user_dietary_preferences`, `get_user_daily_summary`, `get_user_consumed_items`, `get_user_exercises`, `get_user_water_intake`, `get_user_weight`, `get_user_suggested_products`, `get_product`, `search_products`. Schreiben: `add_user_consumed_item`, `add_user_water_intake`, `remove_user_consumed_item` (nur auf ausdrücklichen Wunsch verwenden).

## Absicherung

Der Schutz besteht aus dem Bearer-Token (Caddyfile + Header im Connector). Hinweise:
- Token geheim halten, nicht in Chats einfügen; bei Verdacht auf Leck in `.env` bzw. Caddyfile ändern, den Header im Connector anpassen und `sudo docker compose up -d` ausführen.
- Regelmäßig prüfen: `curl -i http://localhost:8080/garmin/mcp` und `.../yazio/mcp` ohne Header müssen `401` liefern.
