# VPS Resource and Transfer Plan

Status: Sizing- und Uebertragungsentscheidung vor VPS-Deployment.

## 1. Aktueller GitHub-Stand

Das Repo ist die richtige Uebertragungsquelle. Keine ZIP-Datei als Primaerweg verwenden.

Warum nicht ZIP:

- `.env`, lokale Caches, Datenbanken, node_modules oder Secrets koennen versehentlich mitwandern.
- Git-History, Branch-Stand und spaetere Updates werden unklar.
- Deployment und Rollback sind schlechter reproduzierbar.

Empfohlen:

1. Repo auf GitHub aktuell halten.
2. Auf dem VPS per `git clone` oder `git pull` deployen.
3. Nur `.env`, persistente Volumes und ggf. Backups separat behandeln.

## 2. Was muss auf den VPS?

Per Git:

- `docker-compose.yml`
- `docker-compose.prod.yml`
- `backend/`
- `odoo/`
- `pwa/`
- `n8n/workflows/`
- `infrastructure/`
- `docs/`

Nicht per Git/ZIP:

- `.env`
- echte Secrets/API-Keys
- lokale Browserprofile
- lokale Datenbanken
- `node_modules/`
- Python `.deps/`
- alte n8n Backup-Tarballs, sofern nicht bewusst benoetigt

Separat sichern/migrieren:

- Docker volumes: `pg_data`, `odoo_data`, `n8n_data`, `caddy_data`
- n8n Credentials/Workflows via n8n Export, falls bereits live genutzt
- Odoo/Postgres Dump, falls produktive Daten existieren

## 3. Aktuelle Services im Compose

Produktionspfad mit `docker-compose.yml + docker-compose.prod.yml`:

- `caddy` - Reverse Proxy/TLS
- `db` - PostgreSQL 16 Alpine
- `odoo` - Odoo 18 Community + Custom Addon
- `backend` - FastAPI
- `whisper` - OpenAI Whisper ASR Webservice, Modell `small`
- `n8n` - n8n 2.13.3, Postgres-backed, `mem_limit: 2g`, `cpus: 1.5`
- `tunnel` - Cloudflare Tunnel
- `pwa` - statischer Caddy
- `mailpit` - im Prod-Overlay nur Profil `dev`, also normalerweise aus

## 4. Ressourcenrisiko

Der schwere Teil ist nicht n8n. Laut n8n-Doku kann n8n idle sehr klein sein (~100 MB), aber Workflows mit Code Nodes und Binary/Image-Daten koennen deutlich mehr brauchen. In diesem Projekt ist n8n auf 2 GB begrenzt.

Die schweren Komponenten sind:

1. Odoo + PostgreSQL
2. Whisper ASR mit Modell `small`
3. n8n mit Binary-Daten/Fotos
4. Docker Build-Spitzen beim Image-Build

Grobe Schaetzung im Betrieb:

| Service | Erwarteter RAM grob |
| --- | ---: |
| PostgreSQL | 200-700 MB |
| Odoo | 700 MB - 1.5 GB |
| Backend | 150-400 MB |
| n8n | 300 MB - 2 GB Limit |
| Whisper small | 1.0-2.5 GB, je nach Modell/Last |
| Caddy/PWA/Tunnel/Mailpit | <300 MB zusammen |

Realistische Gesamtspanne:

- Minimal idle ohne Whisper-Last: ca. 2.5-4 GB
- Mit Whisper und n8n-Binary-Verarbeitung: ca. 5-7 GB
- Build/Peaks: 7-10 GB moeglich

## 5. Empfehlung nach VPS-Groesse

### 2 GB RAM

Nicht empfehlenswert fuer den Vollstack. Nur moeglich, wenn stark reduziert:

- kein Whisper lokal
- kein Odoo oder externes Odoo
- n8n klein halten
- Swap zwingend

### 4 GB RAM

Grenzwertig. Nur fuer Demo/PoC, wenn wir abspecken:

- Whisper deaktivieren oder kleineres Modell verwenden
- n8n Concurrency niedrig
- Odoo Worker minimal
- Swap 4 GB aktivieren
- keine parallelen Builds auf dem VPS; besser Images vorher bauen oder Build nacheinander

### 8 GB RAM

Empfohlene Mindestgroesse fuer diesen Stack als Bachelor-/Demo-System.

- Odoo + Postgres + Backend + n8n + PWA stabiler
- Whisper small wahrscheinlich nutzbar
- Vision-Workflow ruft externe API auf, kein lokales Vision-Modell
- Swap trotzdem sinnvoll

### 16 GB RAM

Komfortabel.

- Mehr Reserve fuer Builds, Logs, n8n-Binary-Daten, Odoo, Tests
- weniger Risiko bei parallelen Prozessen

## 6. Was wir zuerst rauswerfen oder optional machen sollten

Fuer einen stabilen ersten VPS-Stand:

1. `mailpit` nur Dev-Profil - ist bereits vorbereitet.
2. `whisper` optional machen, falls RAM klein ist.
3. `tunnel` nur nutzen, wenn keine direkte Domain/TLS ueber Caddy verwendet wird.
4. Alte P1-Telegram/Gmail/Knowledge Workflows nicht produktiv aktivieren, wenn sie nicht zur Bachelor-Demo gehoeren.
5. n8n Execution Retention niedrig halten - ist bereits eingeschraenkt.
6. Bilddaten nicht dauerhaft in n8n aufblasen; Binary filesystem + Pruning nutzen.

## 7. VPS-Readiness-Check

Auf dem VPS ausfuehren und Ergebnis pruefen:

```bash
set -e
printf '== host ==\n'
hostname
whoami
uname -a
cat /etc/os-release | sed -n '1,8p'

printf '\n== cpu/mem/disk ==\n'
nproc
free -h
df -h /
lsblk

printf '\n== docker ==\n'
command -v docker || true
command -v docker-compose || true
docker --version || true
docker compose version || true
systemctl is-active docker || true

printf '\n== ports ==\n'
ss -tulpn | grep -E ':(22|80|443|8069|5678|5432|5433|8025)\b' || true

printf '\n== firewall ==\n'
sudo ufw status verbose || true
```

## 8. Docker-Installation, falls fehlt

Nur auf Ubuntu 20.04/22.04/24.04 und nur mit Root/Sudo ausfuehren.

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
docker --version
docker compose version
```

Optional User fuer Docker freischalten:

```bash
sudo usermod -aG docker "$USER"
# danach neu einloggen
```

## 9. Swap fuer kleine VPS

Bei 4-8 GB RAM sinnvoll, bei 2 GB zwingend. Beispiel 4 GB Swap:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

## 10. Deployment-Transfer

Empfohlen:

```bash
mkdir -p ~/apps
cd ~/apps
git clone https://github.com/endritmurati99/mobile-picking-and-voice-assistant.git
cd mobile-picking-and-voice-assistant/'Mobile Picking und Voice Assistant'
cp .env.example .env
chmod 600 .env
nano .env
```

Dann Compose rendern:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml config > /tmp/mobile-picking-compose.rendered.yml
grep -nE '5678:|8069:|5432:|5433:|8025:|--reload|/mnt/c|C:' /tmp/mobile-picking-compose.rendered.yml || true
```

Start erst nach bestandener Pruefung:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml build
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps
```

## 11. Entscheidung

Vor dem Vollstack-Start muss bekannt sein:

- RAM des VPS
- CPU/vCPU
- freier Diskspace
- ob Domain direkt auf VPS zeigt oder Cloudflare Tunnel genutzt werden soll
- ob Whisper lokal gebraucht wird oder spaeter aktiviert werden kann

Technische Empfehlung: Wenn der VPS weniger als 8 GB RAM hat, zuerst reduzierten Stack ohne lokalen Whisper starten und n8n/Odoo stabil bekommen. Danach Features einzeln aktivieren.
