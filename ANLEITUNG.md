# Installationsanleitung für Einsteiger

Diese Anleitung führt dich Schritt für Schritt durch die Einrichtung. Bei jedem Schritt steht, **wo** du gerade bist (PC, Pi oder Browser), **was** du tust und **was du sehen solltest**. Technische Details und die Fehlersuche stehen in der [README](README.md).

Die gezeigten Ausgaben sind sinngemäß. Versionsnummern, Zeiten und Namen sehen bei dir etwas anders aus.

**Dauer:** etwa eine Stunde, das meiste davon ist Warten.

**Du brauchst:**
- einen Raspberry Pi 4 oder 5 mit Netzteil und SD-Karte (mind. 16 GB), am besten mit LAN-Kabel
- einen PC mit SD-Kartenleser
- Konten bei Garmin Connect, Yazio, [Tailscale](https://tailscale.com/) (kostenlos) und Claude

---

## Teil A – Am PC: Pi vorbereiten (≈ 15 Min.)

### 1. SD-Karte beschreiben

Installiere den [Raspberry Pi Imager](https://www.raspberrypi.com/software/) und starte ihn.

1. **Gerät:** dein Pi-Modell, z. B. *Raspberry Pi 5*
2. **Betriebssystem:** *Raspberry Pi OS (other)* → **Raspberry Pi OS Lite (64-bit)**
3. **SD-Karte:** deine Karte
4. Klicke auf *Weiter*. Bei der Frage nach den OS-Anpassungen wählst du **Einstellungen bearbeiten**:
   - **Hostname:** z. B. `raspberrypi`
   - **Benutzername und Passwort:** frei wählen und gut merken
   - **WLAN:** nur eintragen, wenn du kein Kabel nutzt
   - Reiter **Dienste:** *SSH aktivieren*, mit Passwort-Authentifizierung
5. Klicke auf *Speichern*, dann *Ja*. Das Beschreiben dauert ein paar Minuten.

### 2. Pi starten

Steck die Karte in den Pi, schließ das LAN-Kabel an und dann den Strom. Warte 2–3 Minuten. Einen Bildschirm brauchst du nicht.

### 3. Mit dem Pi verbinden

Öffne am PC **PowerShell** (Windows) oder das **Terminal** (Mac/Linux) und gib ein, mit deinem Benutzernamen und Hostnamen:

```bash
ssh <benutzer>@raspberrypi.local
```

Beim ersten Mal kommt eine Rückfrage:

```
The authenticity of host 'raspberrypi.local' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Tippe `yes` und Enter, dann dein Passwort. **Beim Tippen des Passworts erscheinen keine Zeichen, das ist normal.** Danach siehst du:

```
<benutzer>@raspberrypi:~ $
```

Ab jetzt tippst du alles **auf dem Pi**, bis Teil D.

> Kommt die Warnung `REMOTE HOST IDENTIFICATION HAS CHANGED` (weil der Pi schon einmal eingerichtet war), führst du am PC `ssh-keygen -R raspberrypi.local` aus und versuchst es noch einmal.

---

## Teil B – Auf dem Pi: Software installieren (≈ 15 Min.)

### 4. System aktualisieren und Docker installieren

Kopier die Befehle einzeln hinein. In PowerShell fügst du mit **Rechtsklick** ein.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
```

Es läuft viel Text durch, das dauert einige Minuten. Zur Kontrolle:

```bash
sudo docker compose version
```

Du siehst `Docker Compose version v2.…`.

### 5. Projekt herunterladen

```bash
git clone https://github.com/dawidmalas1407/fitness-mcp.git ~/fitness-mcp
cd ~/fitness-mcp
```

Du siehst `Cloning into '…/fitness-mcp'...` und danach `done.`.

---

## Teil C – Auf dem Pi: Einrichten (≈ 20 Min.)

### 6. Zugangsdaten eintragen

Zuerst erzeugst du deinen **MCP-Token**. Das ist ein selbst ausgedachtes Passwort, mit dem Claude später auf deine Server zugreift:

```bash
cp .env.example .env
chmod 600 .env
openssl rand -hex 32
```

Du siehst eine lange Zeichenkette mit 64 Zeichen, z. B. `3f9a1c…e72b`. **Markiere und kopiere sie.**

Dann öffnest du die Datei:

```bash
nano .env
```

Ein Editor öffnet sich:

```
YAZIO_USERNAME=
YAZIO_PASSWORD=
MCP_TOKEN=
```

Geh mit den Pfeiltasten hinter die `=` und trag ein:
- `YAZIO_USERNAME=` deine Yazio-E-Mail
- `YAZIO_PASSWORD=` dein Yazio-Passwort
- `MCP_TOKEN=` den kopierten Token, mit Rechtsklick einfügen

Ohne Leerzeichen und ohne Anführungszeichen. Lässt du einen Wert leer, starten die Server später absichtlich nicht. Speichern: **Strg+O**, dann **Enter**. Beenden: **Strg+X**.

> Diese Datei ist geheim. Zeig sie niemandem und füg sie nirgends ein.

### 7. Server bauen

```bash
sudo docker compose build
```

Beim ersten Mal dauert das 5–10 Minuten. Am Ende steht ohne rote Fehlermeldung so etwas wie `✔ Service garmin Built` und `✔ Service yazio Built`.

### 8. Bei Garmin anmelden

```bash
mkdir -p garmin-tokens
sudo docker compose run --rm -it garmin python garmin_auth.py
```

Du siehst:

```
    ║   GARMIN CONNECT AUTHENTICATION SETUP             ║
Enter your Garmin email:
Enter your Garmin password:
```

Gib deine Garmin-E-Mail und dein Passwort ein. Auch das Passwort bleibt unsichtbar. Hast du die Zwei-Faktor-Anmeldung an, bekommst du eine SMS und gibst den Code bei `Enter the MFA code:` ein.

Bei Erfolg:

```
Authentication successful!
Logged in as: <dein Name>
Tokens saved to: /root/.garminconnect
```

Die Anmeldung hält etwa ein Jahr. Danach wiederholst du diesen Schritt.

> **`429 Too Many Requests`?** Dann blockt Garmin gerade automatische Logins. Warte eine Weile oder nimm die Browser-Variante aus der [README, Schritt 6](README.md#6-garmin-einmalig-anmelden-tokens-erzeugen).

### 9. Server starten

```bash
sudo docker compose up -d
sudo docker compose ps
```

Du siehst drei Zeilen, und bei allen steht **Up**:

```
NAME                   ...   STATUS
fitness-mcp-caddy-1    ...   Up 5 seconds
fitness-mcp-garmin-1   ...   Up 5 seconds
fitness-mcp-yazio-1    ...   Up 5 seconds
```

Zur Kontrolle:

```bash
curl -i http://localhost:8080
```

Du siehst `HTTP/1.1 401 Unauthorized`. **Das ist richtig**, denn ohne Token kommt niemand rein.

### 10. Server aus dem Internet erreichbar machen

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Du siehst:

```
To authenticate, visit:

        https://login.tailscale.com/a/xxxxxxxxxxxx
```

Öffne den Link **am PC**, melde dich bei Tailscale an (zum Beispiel mit Google) und klicke auf *Connect*. Im Terminal erscheint `Success.`.

Jetzt gibst du den Pi frei:

```bash
sudo tailscale funnel --bg 8080
```

Beim ersten Mal meldet der Befehl eventuell, dass Funnel nicht aktiviert ist, und zeigt einen Link. Öffne ihn am PC, klicke auf *Enable* und führe den Befehl noch einmal aus.

Kontrolle:

```bash
sudo tailscale funnel status
```

Du siehst:

```
https://raspberrypi.tailXXXXXX.ts.net (Funnel on)
|-- / proxy http://127.0.0.1:8080
```

**Notiere dir diese Adresse.** Hol dir außerdem deinen Token noch einmal:

```bash
grep MCP_TOKEN .env
```

### 11. Autostart prüfen

```bash
sudo systemctl is-enabled docker tailscaled
```

Du siehst zweimal `enabled`. Damit kommt nach einem Stromausfall alles von selbst wieder hoch.

---

## Teil D – Im Browser: Claude verbinden (≈ 5 Min.)

### 12. Garmin-Connector anlegen

Öffne <https://claude.ai/customize/connectors> und klicke auf **Add custom connector**. Füll den Dialog so aus:

| Feld | Eintrag |
|---|---|
| **Name** | `Garmin` |
| **Remote MCP server URL** | `https://raspberrypi.tailXXXXXX.ts.net/garmin/mcp` (deine Adresse aus Schritt 10 + `/garmin/mcp`) |
| **Authentication** | **No sign-in** |
| **Request headers** → Header | `authorization` |
| **Request headers** → Value | `Bearer 3f9a1c…e72b`, also `Bearer`, **ein Leerzeichen**, dann dein Token |
| **Request headers** → Required | an |

Klicke auf **Add**.

> **Das `Bearer ` nicht vergessen.** Claude schickt den Wert genau so, wie du ihn eingibst. Fehlt es, funktioniert der Connector nicht. Nachträglich ändern geht nicht; du müsstest den Connector löschen und neu anlegen.

> **Kein Abschnitt „Request headers“ zu sehen?** Der ist bei Claude noch eine Beta und nicht in jedem Konto freigeschaltet. Mehr dazu in der [README, Schritt 9.2](README.md#92-wenn-request-headers-fehlt).

### 13. Yazio-Connector anlegen

Genau wie Schritt 12, nur mit:
- **Name:** `Yazio`
- **URL:** `https://raspberrypi.tailXXXXXX.ts.net/yazio/mcp`
- **Token:** derselbe

### 14. Ausprobieren

Öffne einen neuen Chat. Unter **+ → Connectors** müssen *Garmin* und *Yazio* eingeschaltet sein. Schreib:

> Welche Garmin-Geräte habe ich?

Claude fragt, ob es das Tool `get_devices` benutzen darf. Klick auf *Erlauben*, und Claude nennt deine Uhr. Dann:

> Was sind meine Yazio-Ziele?

Claude nennt deine Kalorien- und Makroziele.

> Wenn der allererste Aufruf mit „Error dialing …“ scheitert, versuch es einfach noch einmal.

**Fertig!** Du kannst das SSH-Fenster schließen, der Pi läuft weiter. Die Connectoren funktionieren auch in der Claude-App am Handy mit demselben Konto.

---

## Danach

| Wenn du … | dann … |
|---|---|
| sehen willst, ob alles läuft | per SSH verbinden, `cd ~/fitness-mcp` und `sudo docker compose ps` |
| ein Update einspielen willst | `cd ~/fitness-mcp && git pull && sudo docker compose up -d --build` |
| Garmin nach etwa einem Jahr nicht mehr geht | Schritt 8 wiederholen, dann `sudo docker compose restart garmin` |
| ein Problem hast | die Tabelle [Fehlersuche in der README](README.md#fehlersuche) ansehen |
