# n8n Home Server Deployment

Spin up a secure **n8n** automation server behind a **Cloudflare Tunnel** (Quick Tunnel) using Docker Compose. The tunnel prints a public `*.trycloudflare.com` URL to your logs—no domain or Cloudflare account required. Collaborate with your friends on n8n projects easily and freely by setting up this mini home server.

---

## Contents
- [Prerequisites](#prerequisites)
- [Folder structure](#folder-structure)
- [Environment variables](#environment-variables)
- [Git ignore (important)](#git-ignore-important)
- [docker-compose.yml](#docker-composeyml)
- [First-time setup](#first-time-setup)
- [Run / stop](#run--stop)
- [Get the public URL](#get-the-public-url)
- [Exporting workflows safely](#exporting-workflows-safely)
- [Backups](#backups)
- [Upgrading](#upgrading)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites
- Docker Desktop (Windows/macOS) or Docker Engine (Linux)
- Docker Compose v2 (bundled with Docker Desktop)
- (Optional) Git if you want to version-control your compose file (but **never commit secrets**).

---

## Folder structure

Create this layout in your project directory:

```
n8n-stack/
├─ docker-compose.yml
├─ .env
├─ .gitignore
├─ cloudflared_logs/           # host-mounted logs (created automatically)
└─ n8n_data/                   # persistent n8n data (Docker volume also used)
```

> `n8n_data` is a **Docker named volume** in the compose file. You can also switch it to a bind mount if you prefer (see comments in compose).

---

## Environment variables

Create a `.env` file in the project root and fill these (example values shown):

```env
# Basic auth for the n8n UI
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change-me-strong-password

# Encryption for stored credentials inside n8n (REQUIRED – keep it safe!)
N8N_ENCRYPTION_KEY=replace-with-32-characters-minimum

# Instance options
GENERIC_TIMEZONE=Asia/Karachi
N8N_RUNNERS_ENABLED=false
```

**Security note:** keep `.env` **out of Git**.

---

## Git ignore (important)

Create `.gitignore`:

```gitignore
.env
exports/
n8n_data/
cloudflared_logs/
```

> **Never** commit tokens, API keys, or n8n credential exports. If you ever leak a secret in Git history, rotate it immediately.

---

## docker-compose.yml

The stack uses a **one‑shot init job** to create/clear the log file, plus the n8n app and the Cloudflare tunnel.

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_RUNNERS_ENABLED=${N8N_RUNNERS_ENABLED}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false  # safer on Windows
    volumes:
      - n8n_data:/home/node/.n8n            # named volume for persistence
      - ./exports:/home/node/exports        # optional: host folder for exports
      - ./cloudflared_logs:/tunnel_logs:ro  # mount logs read-only here
    restart: unless-stopped

  # One-shot job to initialize/clear the tunnel log
  cloudflared_loginit:
    image: busybox:latest
    container_name: cloudflared-loginit
    command: ["sh", "-c", "mkdir -p /logs && : > /logs/tunnel.log"]
    volumes:
      - ./cloudflared_logs:/logs
    restart: "no"

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    depends_on:
      - n8n
    command:
      - tunnel
      - --no-autoupdate
      - --edge-ip-version
      - auto
      - --loglevel
      - info
      - --logfile
      - /logs/tunnel.log
      - --url
      - http://n8n:5678
    volumes:
      - ./cloudflared_logs:/logs
    restart: unless-stopped

volumes:
  n8n_data:
    # If you prefer a bind mount instead of a named volume, replace with:
    # driver: local
    # AND change the service mount to: - ./n8n_data:/home/node/.n8n
```

> **Why a one-shot job?** The official `cloudflared` image is minimal and may not have `/bin/sh`. We use `busybox` just to create/clear the log file on demand.

---

## First-time setup

```bash
# 1) Initialize/clear tunnel log (runs once)
docker compose run --rm cloudflared_loginit

# 2) Start core services
docker compose up -d n8n cloudflared

# 3) Open n8n locally
#    http://localhost:5678
```

> If you change the `.env` file, restart n8n: `docker compose restart n8n`.

---

## Run / stop

```bash
# Start all services
docker compose up -d

# Stop (keep data)
docker compose down

# View logs
docker compose logs -f n8n
docker compose logs -f cloudflared
```

---

## Get the public URL

`cloudflared` prints a line in its logs similar to:

```
INF ++++++++++++++++++++++++++++++++++++++++++++++
INF +  https://<random>.trycloudflare.com         +
INF ++++++++++++++++++++++++++++++++++++++++++++++
```

Check it with:

```bash
docker compose logs -f cloudflared
```

Use that URL to access your n8n from the internet.

---

## Exporting workflows safely

- From **n8n UI**: use *Export* for workflows; avoid exporting **credentials**.
- From CLI inside the container:
  ```bash
  docker exec -it n8n n8n export:workflow --all --output=/home/node/exports/workflows.json
  ```
- Ensure `exports/` stays **ignored** by Git (see `.gitignore`).  
- **Never** commit `exports/credentials.json` or any file containing tokens.

---

## Backups

- The `n8n_data` volume holds your database and credentials.
- Quick backup (named volume → host folder):
  ```bash
  docker run --rm -v n8n-stack_n8n_data:/data -v "$(pwd)":/backup busybox tar czf /backup/n8n_data_backup.tgz -C /data .
  ```
- Restore by reversing the process or by switching to a bind mount (`./n8n_data:/home/node/.n8n`).

---

## Upgrading

```bash
docker compose pull
docker compose up -d
```

> Read n8n release notes before major upgrades. Make a backup first.

---

## Troubleshooting

- **Port 5678 already in use**: stop other services using it or change the host port mapping.
- **No public URL appears**: check `docker compose logs -f cloudflared`. Network restrictions can block tunnels.
- **`cloudflared` shell errors**: the official image is minimal; use the provided `cloudflared_loginit` job for log file prep.
- **Windows path issues**: prefer this repo at a simple path like `C:\n8n-stack`. Avoid special characters/spaces.
- **Push protection on GitHub**: if a secret ever gets committed, rotate it, then scrub history (e.g., `git filter-repo`) before pushing again.

---

**Stay safe:** keep `.env` secret, rotate tokens periodically, and don’t export credentials unless you’re migrating securely.

---

## Quick Tunnel rotates each run (important)

**Cloudflare Quick Tunnel** generates a **new random URL each time `cloudflared` starts** (and occasionally after reconnects).  
That means your public link like `https://<random>.trycloudflare.com` **changes** whenever you restart the tunnel container.

- If you share the link with friends/teammates, they’ll need the **latest** URL after each restart.
- See the automation below to broadcast the fresh link automatically.

---

## Collaborate easily

- Share the live tunnel URL with friends so they can open your n8n UI remotely.
- Create a separate **basic-auth user/password** for shared access, or keep one strong credential and rotate it periodically.
- Prefer **sharing exported workflows** (without credentials) instead of sharing your login when possible.

---

## Beginner n8n project: auto-broadcast the new tunnel link

Goal: Every time `cloudflared` prints a new `*.trycloudflare.com` URL in the log, **detect it** and **notify your team** (Slack/Telegram/Email).  
We already mount the logs into the n8n container at **`/tunnel_logs/tunnel.log`** (read-only).

### Workflow outline
Nodes:
1. **Cron** — run every 2 minutes.
2. **Read Binary File** — path: `/tunnel_logs/tunnel.log`.
3. **Convert Binary Data** — to text (JSON field: `data`).
4. **Function** — parse the latest TryCloudflare URL and compare with the last one (stored in workflow static data).
5. **IF** — only continue if `changed == true`.
6. **Notifier** — Slack, Telegram, or Email node to broadcast.

> Tip: If you want the URL **immediately** on startup, run the workflow once manually after `cloudflared` is up, or add an extra **Execute Once** webhook/trigger button.

---

## Pro tips

- If your logs get large, rotate or truncate occasionally (the provided `cloudflared_loginit` service clears the file on demand).
- For reliability, increase Cron frequency and read only the last `N` KB with an **Execute Command** node (e.g., `tail -n 200 /tunnel_logs/tunnel.log`) if you prefer parsing only recent lines.
- Never store webhook URLs or tokens in the repo; keep them in **.env** or n8n credentials.
