# RMS Migration & Deployment Plan

**Date:** 2026-07-31
**Goal:** Migrate RemoteManagementSystems from flat workspace to GitHub repos,
then deploy on gentickit alongside gentick-infra.

---

## 1. Repo Structure (target)

Three repos under the `GentickElectronics` GitHub org, following the TalosOT
pattern:

```
gentick-rms-app/          ← Next.js 15 app (frontend + API)
  app/  components/  lib/  db/  public/  docker/
  package.json  tsconfig.json  next.config.ts  ...
  _docs/
    ├── VERSION
    └── changelogs/
        ├── current.md
        └── ...

gentick-rms-infra/        ← ops: compose, Caddy, n8n, deploy scripts
  compose/  caddy/  n8n/  scripts/
  env.example
  _docs/
    ├── VERSION
    └── changelogs/
        ├── current.md
        └── ...

gentick-rms/              ← umbrella (submodules + product docs)
  gentick-rms-app/        (submodule)
  gentick-rms-infra/      (submodule)
  _docs/
    ├── VERSION
    ├── changelogs/
    ├── requirements.md   ← from Gentick_RMS_Requirements_v1.0.md
    ├── api-spec.md       ← from Gentick_RMS_API_Spec_v1.0.md
    ├── schema.md         ← from Gentick_RMS_Schema_Design_v1.0.md
    ├── erd.mmd           ← from Gentick_RMS_ERD_v1.0.mmd
    ├── erd.svg           ← from Gentick_RMS_ERD_v1.0.svg
    ├── wireframes.html   ← from Gentick_RMS_Wireframes_v1.1.html
    ├── openapi.yaml      ← from Gentick_RMS_OpenAPI_v1.0.yaml
    ├── deployment-runbook.md  ← from Gentick_RMS_Deployment_Runbook_v1.0.md
    └── gap-analysis.md   ← from Gentick_RMS_GitHub_Standards_Gap_Analysis.md
  README.md               ← signpost pointing to _docs/
```

---

## 2. Migration Steps

### Step 1 — Rename & clean up source folders

The current workspace at `C:\dev\RepairManagementSystem\`:

```
RepairManagementSystem/
├── gentick-rms/          ← will become gentick-rms-app/
├── gentick-rms-app/      ← DELETE — skeleton placeholder
├── gentick-rms-infra/    ← stays, needs _docs/ standard applied
└── 19 loose files        ← .md, .yaml, .sql, .html, .png, .svg — move to umbrella _docs/
```

Actions:
1. Delete `gentick-rms-app/` (the skeleton — empty except for README stubs).
2. Rename `gentick-rms/` → `gentick-rms-app/`.
3. Move all 19 loose root files into `gentick-rms/_docs/` (umbrella).
4. Add `_docs/VERSION` and `_docs/changelogs/current.md` to each repo.

### Step 2 — Git init each repo

```bash
# gentick-rms-app
cd C:\dev\rms\gentick-rms-app
git init
git checkout -b main   # if not default
# Add _docs/VERSION (v0.0.0), _docs/changelogs/current.md
# Add root README.md signpost
git add -A
git commit -m "chore: initial commit — Next.js 15 RMS app scaffolding"

# gentick-rms-infra
cd C:\dev\rms\gentick-rms-infra
git init
git checkout -b main
# Apply _docs/ standard: add VERSION, changelogs/current.md
# Add root README.md (already has one)
git add -A
git commit -m "chore: initial commit — RMS infra: compose, Caddy, n8n, deploy scripts"

# gentick-rms (umbrella)
cd C:\dev\rms
git init
git checkout -b main
# Add _docs/ with all design artifacts
# Add root README.md signpost
git add -A
git commit -m "chore: initial commit — RMS umbrella with design docs"
```

### Step 3 — Push to GitHub

```bash
# Create repos on GitHub (GentickElectronics org):
#   gentick-rms-app
#   gentick-rms-infra
#   gentick-rms

cd C:\dev\rms\gentick-rms-app
git remote add origin git@github.com:GentickElectronics/gentick-rms-app.git
git branch -M main
git push -u origin main

cd C:\dev\rms\gentick-rms-infra
git remote add origin git@github.com:GentickElectronics/gentick-rms-infra.git
git branch -M main
git push -u origin main

cd C:\dev\rms
git remote add origin git@github.com:GentickElectronics/gentick-rms.git
git branch -M main
git push -u origin main
```

### Step 4 — Wire submodules in umbrella

```bash
cd C:\dev\rms
git submodule add git@github.com:GentickElectronics/gentick-rms-app.git
git submodule add git@github.com:GentickElectronics/gentick-rms-infra.git
git add .gitmodules gentick-rms-app gentick-rms-infra
git commit -m "chore: pin gentick-rms-app + gentick-rms-infra as submodules"
git push
```

### Step 5 — Apply git standards

Per the Gentick engineering standards:
- Create `agentic` branch on each repo from `main`.
- Add branch protection on GitHub: require PR for `main`, block direct pushes.
- Add `CHANGELOG.md` → `_docs/changelogs/current.md` with `## [Unreleased]` header.
- Set `_docs/VERSION` to `v0.0.0` (R&D pre-release).
- Add `.gitignore` (already present in app, add to infra if missing).

---

## 3. Gentick-Infra Integration

RMS will follow the exact TalosOT pattern: **private containers only** on
gentickit, sharing gentick-infra's Postgres and nginx.

### 3.1 Database

```bash
# On gentickit — one-time setup
docker exec -it gentick-postgres psql -U gentick -c "CREATE DATABASE gentick_rms;"
```

### 3.2 Nginx route

Add to gentick-infra's `nginx/nginx.conf`:

```nginx
# RMS — Next.js app + n8n
server {
    listen 80;
    server_name rms.gentick.co.za;

    location / {
        proxy_pass http://gentick-app:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

server {
    listen 80;
    server_name automation.gentick.co.za;

    location / {
        proxy_pass http://gentick-n8n:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### 3.3 RMS compose (minimal — no Caddy, no Postgres)

```yaml
# compose.yml — deployed at /opt/gentick-rms on gentickit
name: gentick-rms

networks:
  backend-net:
    external: true
    name: gentick-infra_backend-net

services:
  app:
    image: ghcr.io/gentickelectronics/gentick-rms-app:0.0.0
    container_name: gentick-app
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://${POSTGRES_USERNAME}:${POSTGRES_PASSWORD}@postgres:5432/gentick_rms
      SUPABASE_URL: ${SUPABASE_URL}
      SUPABASE_ANON_KEY: ${SUPABASE_ANON_KEY}
      SUPABASE_SERVICE_ROLE_KEY: ${SUPABASE_SERVICE_ROLE_KEY}
      N8N_WEBHOOK_URL: http://gentick-n8n:5678/webhook/gentick-events
      WEBHOOK_SECRET: ${WEBHOOK_SECRET}
      STORAGE_URL_SECRET: ${STORAGE_URL_SECRET}
      NEXT_PUBLIC_APP_URL: https://rms.gentick.co.za
      EMAIL_FROM_ADDRESS: repairs@gentick.co.za
      EMAIL_FROM_NAME: Gentick Electronics
    volumes:
      - uploads:/data/uploads
    networks:
      - backend-net

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: gentick-n8n
    restart: unless-stopped
    environment:
      N8N_HOST: automation.gentick.co.za
      N8N_PROTOCOL: https
      WEBHOOK_URL: https://automation.gentick.co.za/
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: ${POSTGRES_USERNAME}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_BASIC_AUTH_ACTIVE: "true"
      N8N_BASIC_AUTH_USER: ${N8N_BASIC_AUTH_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_BASIC_AUTH_PASSWORD}
      GENERIC_TIMEZONE: Africa/Johannesburg
    volumes:
      - n8n-data:/home/node/.n8n
    networks:
      - backend-net

volumes:
  uploads:
  n8n-data:
```

### 3.4 Server env vars

Create `/opt/gentick-rms/.env` (mode 600):

```bash
POSTGRES_USERNAME=<from gentick-infra .env>
POSTGRES_PASSWORD=<from gentick-infra .env>
SUPABASE_URL=https://<project>.supabase.co
SUPABASE_ANON_KEY=<from Supabase dashboard>
SUPABASE_SERVICE_ROLE_KEY=<from Supabase dashboard>
WEBHOOK_SECRET=<openssl rand -hex 32>
STORAGE_URL_SECRET=<openssl rand -hex 32>
N8N_ENCRYPTION_KEY=<openssl rand -hex 32>
N8N_BASIC_AUTH_USER=lucia
N8N_BASIC_AUTH_PASSWORD=<openssl rand -base64 16>
```

---

## 4. Deploy Flow

### 4.1 Build & push image

```bash
cd gentick-rms-app
docker build -t ghcr.io/gentickelectronics/gentick-rms-app:0.0.0 -f docker/app.Dockerfile .
docker push ghcr.io/gentickelectronics/gentick-rms-app:0.0.0
```

### 4.2 Deploy on gentickit

```bash
ssh deploy@gentickit

# Pull image
docker pull ghcr.io/gentickelectronics/gentick-rms-app:0.0.0

# Ensure directory exists
sudo mkdir -p /opt/gentick-rms /data/rms-uploads
sudo chown -R deploy:deploy /opt/gentick-rms /data/rms-uploads

# Place compose.yml + .env
cd /opt/gentick-rms

# Start
docker compose up -d

# Run DB migrations
docker compose exec app pnpm db:migrate

# Seed initial admin user (via Supabase Auth first, then SQL)
# docker compose exec postgres psql -U gentick -d gentick_rms \
#   -c "INSERT INTO staff_users ..."
```

### 4.3 Verify

- `docker compose ps` — both `gentick-app` and `gentick-n8n` healthy
- `curl http://gentick-app:3000/api/v1/health` → 200
- Browse `https://rms.gentick.co.za` → login screen
- n8n at `https://automation.gentick.co.za` → login prompt

---

## 5. What RMS Does NOT Need (because gentick-infra provides it)

| Service | Provided by |
|---------|------------|
| Postgres | gentick-infra (`gentick_rms` database added) |
| TLS/HTTPS | Cloudflare Tunnel (terminates at edge) |
| Reverse proxy | gentick-infra nginx |
| MQTT | Not needed by RMS |
| QuestDB | Not needed by RMS |
| Redis | Not needed by RMS |
| MariaDB | Not needed by RMS |

RMS brings only two containers to the server: `app` + `n8n`.

---

## 6. Dependencies & Prerequisites

### External SaaS (needs accounts):
- **Supabase** — Auth provider (free tier sufficient)
- **Postmark** — Transactional email (free for first 100/month)
- **Meta WhatsApp Business** — Customer notifications (requires business verification, 1–10 business days lead time)
- **GitHub Container Registry** — Image hosting (included with GitHub)

### Server-side (one-time):
- `CREATE DATABASE gentick_rms` on shared Postgres
- DNS: `rms.gentick.co.za` + `automation.gentick.co.za` → Cloudflare Tunnel
- nginx config update (routes above)
- `.env` file at `/opt/gentick-rms/.env`

### Dev laptop:
- Docker Desktop running
- Node.js 20 LTS + pnpm 9+
- SSH access to gentickit as `deploy`
- GHCR login: `docker login ghcr.io -u teojordaangentick`

---

## 7. Open Decisions

1. **Supabase project** — use the existing TalosOT Supabase project (shared auth across products) or create a dedicated `gentick-rms-prod` project?
2. **n8n — shared or dedicated?** Currently RMS gets its own n8n container. Could consolidate with a future shared n8n instance on gentick-infra, but RMS's 5 workflows are heavily customized (WhatsApp + Postmark credentials). Keeping separate for now is safer.
3. **Image registry** — use `ghcr.io/gentickelectronics/*` (same as TalosOT) or `ghcr.io/teojordaangentick/*`? Standard says `GentickElectronics` org.
