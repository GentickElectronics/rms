# Gentick RMS — Deployment Runbook

**Version:** 1.0
**Date:** 31 May 2026
**Audience:** Whoever provisions, deploys, operates, or recovers the production system
**Target environment:** Hetzner Cloud, Cape Town region, Ubuntu 24.04 LTS

---

## 1. What this document is

The complete operations manual for the Gentick RMS production system. Read it top-to-bottom before first deployment; refer to specific sections during operations.

Every command in this doc is copy-pasteable. Where a value needs to be replaced (e.g., your domain, your email), it's shown like `<gentick.co.za>` — replace with the actual value.

The runbook is divided into:

| Part | When you use it |
| --- | --- |
| **Part A — One-time provisioning** | First-ever deployment (sections 2–10) |
| **Part B — First deploy** | Once provisioning is done (sections 11–14) |
| **Part C — Ongoing operations** | Daily / weekly / when-needed (sections 15–18) |
| **Part D — Disaster recovery** | When something goes wrong (sections 19–22) |

---

## PART A — One-time provisioning

## 2. Provision the Hetzner VPS

### 2.1 Server specifications

For Gentick's MVP scale (hundreds of jobs/month, 1–2 concurrent staff, occasional customer portal traffic):

| Spec | Choice | Why |
| --- | --- | --- |
| Provider | Hetzner Cloud | Per requirements; Cape Town datacenter; competitive pricing |
| Plan | **CX22** (€4.51/mo) | 2 vCPU, 4 GB RAM, 40 GB SSD. Enough for app + Postgres + n8n + Caddy with headroom |
| Region | **Cape Town (cpt1)** | Lowest latency for SA customers; data stays in SA (POPIA-friendly) |
| OS | Ubuntu 24.04 LTS | Long-term support; matches the Docker base images |
| Backup | Enable Hetzner snapshots | Adds 20% to cost; pays for itself the first time you need it |
| IPv4 | Yes | Required |
| Firewall | Hetzner Cloud Firewall (configured below) | Layer in front of UFW |

If you grow beyond ~1000 jobs/month, jump to CX32 (4 vCPU / 8 GB RAM) — same operating procedure, just a bigger box.

### 2.2 Create the server

In the Hetzner Cloud Console (`console.hetzner.cloud`):

1. **Project:** create one called `Gentick Production` (a single project per environment is cleanest).
2. **Add server:**
   - Location: Cape Town
   - Image: Ubuntu 24.04
   - Type: CX22
   - SSH key: add your local public key from `~/.ssh/id_ed25519.pub` (generate one first if you don't have it: `ssh-keygen -t ed25519`)
   - Name: `gentick-rms-prod-01`
   - Networking: standard IPv4 + IPv6 enabled
   - Backups: Yes (enable)
3. **Note the IPv4 address** — you'll need it for DNS in section 3.

### 2.3 Cloud Firewall

In the same console, **Firewalls → Create**:

| Rule | Direction | Protocol | Port | Source |
| --- | --- | --- | --- | --- |
| SSH | Inbound | TCP | 22 | Your office IP / VPN range only (NOT 0.0.0.0/0) |
| HTTP | Inbound | TCP | 80 | 0.0.0.0/0 |
| HTTPS | Inbound | TCP | 443 | 0.0.0.0/0 |

Apply this firewall to `gentick-rms-prod-01`. SSH from elsewhere is now blocked — if you ever lock yourself out, edit the source range in the console.

## 3. DNS

Point three subdomains at the new IPv4:

| Subdomain | Type | Value | Used for |
| --- | --- | --- | --- |
| `rms.gentick.co.za` | A | `<server-ipv4>` | The web app |
| `automation.gentick.co.za` | A | `<server-ipv4>` | n8n |
| `repairs.gentick.co.za` (optional) | CNAME | `rms.gentick.co.za` | Friendly alias if you want |

Wait until DNS propagates (`dig rms.gentick.co.za` returns the right IP — usually 1–30 minutes) before continuing to section 7, otherwise Let's Encrypt cert issuance will fail.

While you're in DNS, also add the Postmark records for transactional email deliverability — Postmark's dashboard generates the exact SPF, DKIM, and DMARC records to add. Without these, your emails will land in spam.

## 4. Initial server hardening

SSH in as root for the first and only time:

```bash
ssh root@<server-ipv4>
```

### 4.1 System update + base tools

```bash
apt update && apt upgrade -y
apt install -y ufw fail2ban unattended-upgrades curl wget git restic htop ncdu vim
```

### 4.2 Create a deploy user

```bash
adduser --disabled-password --gecos "" deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# Allow passwordless sudo for service operations only (no full shell sudo).
# Adjust this if your team needs broader access.
echo "deploy ALL=(ALL) NOPASSWD: /usr/bin/docker, /usr/bin/docker-compose, /usr/bin/systemctl" > /etc/sudoers.d/deploy
```

### 4.3 Lock down SSH

```bash
# Disable root login + password auth; enforce key-only access
sed -i 's/^#*PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl reload ssh
```

**Test in a SECOND terminal** before closing this one:
```bash
ssh deploy@<server-ipv4>
```
If that works, exit the root session. From now on you always use `deploy@`.

### 4.4 UFW firewall (defence in depth alongside Hetzner's)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### 4.5 Fail2ban + automatic security updates

```bash
sudo systemctl enable --now fail2ban
sudo dpkg-reconfigure -plow unattended-upgrades  # accept defaults
```

### 4.6 Timezone + swap

```bash
sudo timedatectl set-timezone Africa/Johannesburg

# 2GB swap file (helps when Docker builds peak memory)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 5. Install Docker

```bash
# Official Docker repo
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Allow deploy user to run docker without sudo
sudo usermod -aG docker deploy
newgrp docker  # apply now without re-login
docker --version  # smoke test
```

## 6. Lay out the deploy directory

```bash
sudo mkdir -p /opt/gentick-rms /data/uploads /data/backups
sudo chown -R deploy:deploy /opt/gentick-rms /data
cd /opt/gentick-rms
git init  # we'll populate from CI on first deploy
```

This is where the production `docker-compose.prod.yml` and `.env.prod` will live.

## 7. Create secrets

All secrets live in `/opt/gentick-rms/.env.prod` (mode `600`, owner `deploy`). NEVER commit this file.

Generate the cryptographic secrets first:

```bash
echo "WEBHOOK_SECRET=$(openssl rand -hex 32)"
echo "STORAGE_URL_SECRET=$(openssl rand -hex 32)"
echo "N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)"
echo "JWT_SIGNING_SECRET=$(openssl rand -hex 32)"
echo "POSTGRES_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=' | head -c 24)"
echo "N8N_BASIC_AUTH_PASSWORD=$(openssl rand -base64 16 | tr -d '/+=' | head -c 16)"
```

Save those values somewhere safe (1Password, your KMS, an encrypted file). You'll need them in `.env.prod` and to issue the n8n service JWT.

Now create `.env.prod`:

```bash
nano /opt/gentick-rms/.env.prod
```

Paste this skeleton and fill in the values:

```bash
# ===== Postgres =====
POSTGRES_USER=gentick
POSTGRES_PASSWORD=<from above>
POSTGRES_DB=gentick_rms

# ===== App =====
DATABASE_URL=postgres://gentick:<POSTGRES_PASSWORD>@postgres:5432/gentick_rms
NEXT_PUBLIC_APP_URL=https://rms.gentick.co.za
LOG_LEVEL=info

# ===== Supabase =====
SUPABASE_URL=https://<your-project>.supabase.co
SUPABASE_ANON_KEY=eyJhbG...
SUPABASE_SERVICE_ROLE_KEY=eyJhbG...

# ===== n8n integration =====
N8N_WEBHOOK_URL=http://n8n:5678/webhook/gentick-events
WEBHOOK_SECRET=<from above>
N8N_SERVICE_JWT=<issued in section 7.1 below>

# ===== Storage =====
STORAGE_URL_SECRET=<from above>

# ===== Email =====
EMAIL_FROM_ADDRESS=repairs@gentick.co.za
EMAIL_FROM_NAME=Gentick Electronics

# ===== n8n =====
N8N_ENCRYPTION_KEY=<from above>
N8N_BASIC_AUTH_USER=lucia
N8N_BASIC_AUTH_PASSWORD=<from above>

# ===== Optional =====
SENTRY_DSN=
RMS_IMAGE_TAG=latest
```

Lock the file:

```bash
chmod 600 /opt/gentick-rms/.env.prod
```

### 7.1 Issue the n8n service JWT

n8n authenticates to the RMS using a long-lived JWT. Generate once:

```bash
# On the server, with Node already installed (any version)
JWT_SIGNING_SECRET=<from above> node -e "
const { SignJWT } = require('jose');
const secret = new TextEncoder().encode(process.env.JWT_SIGNING_SECRET);
new SignJWT({ sub: 'n8n-service', role: 'system' })
  .setProtectedHeader({ alg: 'HS256' })
  .setIssuedAt()
  .setIssuer('rms.gentick.co.za')
  .sign(secret)
  .then(console.log);
"
```

If `jose` isn't installed, `npm install -g jose` first. Paste the output into `.env.prod` as `N8N_SERVICE_JWT=...` and ALSO save it in n8n's credential store later (section 13).

## 8. Postmark account setup

1. Sign up at `postmarkapp.com` (free for the first 100 emails/month).
2. **Sender Signatures → Add Sender Signature** → `repairs@gentick.co.za`. Click the verification link Postmark emails.
3. **Domain → Add Domain** → `gentick.co.za`. Postmark gives you SPF, DKIM, and DMARC DNS records — add them in your DNS host (Cloudflare or wherever).
4. Wait ~30 minutes for DNS to propagate, then click "Verify" in Postmark.
5. **Servers → Create Server** → name it `gentick-rms-prod`. Get the **Server API Token** — paste this into n8n's `Postmark Server Token` credential (section 13).

## 9. Meta WhatsApp Business setup

This is the longest-lead-time step — start it early (it can take 1–10 business days for Meta to verify your business).

1. Meta Business Manager → **Add WhatsApp Business Account**.
2. **Phone Numbers → Add Phone Number** — register the phone number customers will see messages from.
3. **Business Verification** → submit Gentick's business details. Meta verifies via documents (CIPC registration, utility bill at your registered address). Allow 1–10 business days.
4. While verification is pending, generate a **System User** in Business Settings → Users → System Users → Add. Assign WhatsApp Business Management permission. Generate a **permanent access token** (never expires).
5. Note the **phone number ID** (15-digit number, NOT the phone number itself) — find it in WhatsApp Manager → Phone Numbers → API setup → "Phone Number ID".
6. While waiting for business verification, **submit the WhatsApp templates** from `n8n/whatsapp-templates.md`. They'll be in pending status until verification clears.

## 10. Supabase Auth setup

1. Sign up at `supabase.com` (free tier is fine).
2. Create a new project — name `gentick-rms-prod`. Pick a region (Cape Town isn't available yet for Supabase; Frankfurt is the closest with ~150ms latency to SA — acceptable since auth calls are infrequent).
3. From the **API** settings, copy:
   - Project URL (`SUPABASE_URL`)
   - `anon` key (`SUPABASE_ANON_KEY`)
   - `service_role` key (`SUPABASE_SERVICE_ROLE_KEY` — this is sensitive, never expose to browser)
4. From **Authentication → URL Configuration**, add `https://rms.gentick.co.za` to "Site URL" and to the allowed redirect URLs list.
5. Paste the keys into `.env.prod`.

---

## PART B — First deploy

## 11. Build the image (from your dev laptop, OR rely on CI)

The cleanest path is to let GitHub Actions build and push the image (see CI/CD setup in `.github/workflows/`). But for a first deploy without CI you can do it manually:

```bash
# On your dev laptop, inside the gentick-rms/ folder
docker build -t ghcr.io/<gh-org>/gentick-rms:latest -f docker/app.Dockerfile .
docker login ghcr.io -u <gh-username>
docker push ghcr.io/<gh-org>/gentick-rms:latest
```

If you're using CI, just push to `main` and CI does the build + push for you.

## 12. Copy compose files to the server

```bash
# From your dev laptop, in gentick-rms/
scp docker-compose.prod.yml deploy@<server-ipv4>:/opt/gentick-rms/
scp -r docker/ deploy@<server-ipv4>:/opt/gentick-rms/
scp -r db/migrations/ deploy@<server-ipv4>:/opt/gentick-rms/db/migrations/
scp -r n8n/workflows/ deploy@<server-ipv4>:/opt/gentick-rms/n8n/workflows/
```

(CI/CD automates this step — see `.github/workflows/deploy.yml`.)

## 13. First start

```bash
ssh deploy@<server-ipv4>
cd /opt/gentick-rms

# Pull the image (requires registry login on the server too — see below)
docker login ghcr.io -u <gh-username>  # one-time
docker compose -f docker-compose.prod.yml --env-file .env.prod pull
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d

# Watch the logs until everything is healthy
docker compose -f docker-compose.prod.yml logs -f --tail=50
```

Expected sequence:
1. Postgres starts, runs `postgres -D /var/lib/postgresql/data`, waits for healthcheck.
2. App container starts, connects to Postgres.
3. Caddy starts, attempts Let's Encrypt cert issuance for both subdomains. This takes 30–60 seconds on first run. If it fails, check DNS is pointing at this server.
4. n8n starts and is ready on port 5678 (internal, behind Caddy).

When `docker compose ps` shows all four services as `running (healthy)`, proceed.

### 13.1 Run database migrations

```bash
# Migrations are in db/migrations/. Apply 001_initial_schema.sql:
docker compose -f docker-compose.prod.yml --env-file .env.prod exec -T postgres \
  psql -U gentick -d gentick_rms -f /docker-entrypoint-initdb.d/001_initial_schema.sql

# OR — recommended for future migrations — run via the app container's drizzle-kit:
docker compose -f docker-compose.prod.yml --env-file .env.prod exec app pnpm db:migrate
```

### 13.2 Seed the initial staff user

Right now there are no users. Create your own Supabase Auth account first (sign up via the app login page once Caddy serves it), then insert your `staff_users` row:

```bash
docker compose -f docker-compose.prod.yml --env-file .env.prod exec postgres \
  psql -U gentick -d gentick_rms -c "
INSERT INTO staff_users (supabase_user_id, full_name, email, role)
VALUES ('<your-supabase-user-id>', 'Lucia M', 'lucia@gentick.co.za', 'admin');
"
```

(Get your Supabase user ID from the Supabase dashboard → Authentication → Users → click your email.)

### 13.3 Import n8n workflows + credentials

1. Browse to `https://automation.gentick.co.za` — log in with the basic auth credentials from `.env.prod`.
2. Workflows → **Import from File** → import each JSON in `/opt/gentick-rms/n8n/workflows/` in order (01 through 05).
3. **Credentials → New** → create:
   - `Meta WhatsApp Bearer Auth` (HTTP Header Auth): name `Authorization`, value `Bearer <META_PERMANENT_TOKEN>`
   - `Postmark Server Token`: paste the Postmark token from section 8
   - `RMS Service Account JWT` (HTTP Header Auth): name `Authorization`, value `Bearer <N8N_SERVICE_JWT>`
4. On each imported workflow, open every node that uses a credential and select the matching credential from the dropdown.
5. Set the n8n environment variables in `docker-compose.prod.yml` (or extra env in the compose file) — see `n8n/workflows/README.md` for the full list.
6. **Activate** each workflow.

### 13.4 Configure Meta webhook URLs

In Meta Business Manager → WhatsApp Manager → Configuration:

- Callback URL: `https://automation.gentick.co.za/webhook/meta-whatsapp-inbound`
- Verify token: same value as `META_VERIFY_TOKEN` env var
- Subscribe to: `messages`, `message_statuses`, `message_template_status_update`

Click "Verify and save". Meta does a GET handshake — the second n8n workflow you set up for verification (per templates doc section 1) responds with the challenge.

## 14. Smoke test

End-to-end test:

1. Browse to `https://rms.gentick.co.za`. You should see the login screen with the Gentick logo.
2. Log in as yourself.
3. Create a test main customer with prefix `TEST`.
4. Create a test agent under that customer.
5. Create a test job via the intake form.
6. Verify: did `job.received` fire? Check n8n's executions log AND the customer's WhatsApp (if their phone is opted in).
7. Walk through the full workflow: diagnose → quote → approve (reply YES on WhatsApp) → in progress → boxed → ready for collection → collected → closed.
8. Confirm each stage transition fires the appropriate notification.
9. Generate a proforma invoice and a Sage CSV export. Verify it imports into Sage Pastel Online without errors.

If anything fails, the logs are your friend:
```bash
docker compose -f docker-compose.prod.yml logs -f app | grep ERROR
docker compose -f docker-compose.prod.yml logs -f n8n
```

---

## PART C — Ongoing operations

## 15. Deploying an update

With CI/CD wired up (next deliverable), this happens automatically when you push to `main`. Manual procedure:

```bash
ssh deploy@<server-ipv4>
cd /opt/gentick-rms

# Pull the new image
docker compose -f docker-compose.prod.yml --env-file .env.prod pull app

# Run any new migrations FIRST (before restarting the app)
docker compose -f docker-compose.prod.yml --env-file .env.prod exec app pnpm db:migrate

# Restart just the app — Postgres and n8n keep running
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps app

# Watch logs
docker compose -f docker-compose.prod.yml logs -f app
```

The app container has a health check — if the new image fails to start cleanly, Docker keeps the old one running on the same port (zero-downtime deploys aren't automatic in single-instance Compose — see Phase 2 enhancements).

## 16. Backups

Two things need backing up: PostgreSQL and the uploads directory. n8n stores its workflows in Postgres so it's covered.

### 16.1 Backup destination

The recommended setup is **Hetzner Storage Box** (separate from the VPS, EU region). 1TB Storage Box is ~€4/month and accessible via SFTP.

Alternative: another Hetzner Cloud server in Frankfurt running a `restic` repository, accessed over the Hetzner private network. Slightly more setup, slightly cheaper.

### 16.2 Configure restic

```bash
# One-time
sudo nano /opt/gentick-rms/restic.env
```
```bash
export RESTIC_REPOSITORY=sftp:u123456@u123456.your-storagebox.de:/backups/gentick-rms
export RESTIC_PASSWORD=<long random string — store in 1Password too>
```
```bash
chmod 600 /opt/gentick-rms/restic.env
source /opt/gentick-rms/restic.env

# Initialize the repo (one-time)
restic init
```

### 16.3 Nightly backup script

```bash
sudo nano /opt/gentick-rms/scripts/backup.sh
```
```bash
#!/usr/bin/env bash
set -euo pipefail
source /opt/gentick-rms/restic.env

cd /opt/gentick-rms

# 1. Dump Postgres
docker compose -f docker-compose.prod.yml --env-file .env.prod exec -T postgres \
  pg_dumpall -U gentick > /tmp/gentick_$(date +%F).sql

# 2. Restic snapshot: SQL dump + uploads + n8n volume
restic backup \
  /tmp/gentick_$(date +%F).sql \
  /data/uploads \
  --tag nightly \
  --host gentick-rms-prod-01

rm /tmp/gentick_*.sql

# 3. Prune to retain 30 daily + 12 monthly + 7 yearly snapshots
restic forget --keep-daily 30 --keep-monthly 12 --keep-yearly 7 --prune

# 4. Sanity-check the backup integrity
restic check --read-data-subset=1G  # spot check 1 GB
```
```bash
chmod +x /opt/gentick-rms/scripts/backup.sh
```

### 16.4 Schedule

```bash
sudo crontab -u deploy -e
```
Add:
```cron
# Nightly backup at 02:30 SAST
30 2 * * * /opt/gentick-rms/scripts/backup.sh >> /var/log/gentick-backup.log 2>&1
```

### 16.5 Monitor backup health

A failed backup that nobody notices is the same as no backup. Set up UptimeRobot's "Heartbeat" (free) and have the backup script ping it on success:

```bash
# Append to backup.sh
curl -fsS --max-time 10 --retry 3 -o /dev/null https://heartbeat.uptimerobot.com/<your-id>
```

If the heartbeat doesn't ping every 24 hours, UptimeRobot emails you.

## 17. Monitoring

Three layers:

| Layer | Tool | What it watches |
| --- | --- | --- |
| Uptime | UptimeRobot (free) | `https://rms.gentick.co.za/api/v1/health` every 5 min; SMS alert on failure |
| Logs | `docker compose logs` + journald | Per-container application logs |
| n8n executions | n8n built-in | Per-workflow execution history with errors |

### 17.1 UptimeRobot setup

1. Sign up at `uptimerobot.com`.
2. **Add New Monitor → HTTP(s)**:
   - URL: `https://rms.gentick.co.za/api/v1/health`
   - Interval: 5 minutes
   - Alert contacts: your phone (SMS) + email
3. Add a second monitor for n8n: `https://automation.gentick.co.za/healthz`.
4. Add a third "Heartbeat" monitor for the backup script (section 16.5).

### 17.2 Viewing logs

```bash
# Tail everything
docker compose -f docker-compose.prod.yml logs -f --tail=100

# Just the app, filter for errors
docker compose -f docker-compose.prod.yml logs -f app | grep -i error

# n8n executions
docker compose -f docker-compose.prod.yml logs -f n8n
```

The structured JSON logs from the app include `request_id` — search for a specific failed request by ID:
```bash
docker compose -f docker-compose.prod.yml logs app | grep "req_2026053111300012ab"
```

## 18. Routine ops tasks

### Restart a service

```bash
docker compose -f docker-compose.prod.yml --env-file .env.prod restart app
```

### Stop everything

```bash
docker compose -f docker-compose.prod.yml stop
```

### Apply a system OS update

```bash
sudo apt update && sudo apt upgrade -y
# If kernel updated:
sudo reboot
```
(Unattended-upgrades from section 4.5 handles security patches automatically; this is for everything else.)

### Rotate the n8n service JWT (yearly)

1. Generate a new JWT (section 7.1).
2. Update `.env.prod` → `N8N_SERVICE_JWT=...`.
3. Update the n8n credential `RMS Service Account JWT` with the new value.
4. Restart the app: `docker compose ... restart app`.
5. The old JWT is still valid until the secret is rotated — see "secret rotation" below.

### Rotate a secret (e.g., DB password)

This is delicate. Sketch:

1. Change `POSTGRES_PASSWORD` in `.env.prod`.
2. Change the password inside Postgres: `ALTER USER gentick WITH PASSWORD '<new>';`
3. Restart the app (it'll pick up the new env var and reconnect).
4. Test.

For `WEBHOOK_SECRET` (HMAC signing for n8n), update both `.env.prod` AND n8n's `WEBHOOK_SECRET` env at the same moment — there will be a few seconds where in-flight webhooks fail HMAC verification. Schedule for low-traffic time.

---

## PART D — Disaster recovery

## 19. Restoring from backup

```bash
# 1. SSH to a fresh (or recovered) server
ssh deploy@<server-ipv4>

# 2. Pull restic env
source /opt/gentick-rms/restic.env

# 3. List available snapshots
restic snapshots --tag nightly

# 4. Pick the snapshot ID and restore
restic restore <snapshot-id> --target /tmp/restore

# 5. Restore Postgres
docker compose -f docker-compose.prod.yml --env-file .env.prod stop app n8n
docker compose -f docker-compose.prod.yml --env-file .env.prod exec -T postgres \
  psql -U gentick < /tmp/restore/tmp/gentick_<date>.sql

# 6. Restore uploads
rsync -av /tmp/restore/data/uploads/ /data/uploads/

# 7. Restart
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d

# 8. Smoke test (section 14)
```

## 20. Rollback a bad deploy

If a new app version is broken and you need to roll back:

```bash
cd /opt/gentick-rms

# Find the previous good image tag (CI/CD tags every image with the git SHA)
docker images ghcr.io/<gh-org>/gentick-rms

# Set the env var and restart
sed -i "s/^RMS_IMAGE_TAG=.*/RMS_IMAGE_TAG=<previous-sha>/" .env.prod
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps app
```

**If the bad deploy ran a database migration that's not backwards-compatible**, rolling back the app isn't enough — restore the DB from the most recent pre-deploy backup (section 19). This is why every migration must be backwards-compatible with the immediately previous app version (Phase 2 ops convention to enforce in CI).

## 21. Disaster scenarios

| Scenario | Action |
| --- | --- |
| App container won't start | Check `docker compose logs app` → fix the issue, re-deploy. If urgent, roll back to previous tag. |
| Postgres won't start (corrupted) | Restore from last nightly backup (section 19). Lose ≤ 24h of data. |
| VPS unreachable (Hetzner outage) | Wait — Hetzner publishes status at `status.hetzner.com`. SLA credit if it exceeds 99% monthly. |
| VPS hardware failure | Provision a new VPS (sections 2–10), restore from backup (section 19). Repoint DNS. Recovery time ~1 hour. |
| Lost SSH access (firewall lock-out) | Hetzner Cloud Console → server → "Console" tab → log in as root via the web console → fix firewall rules. |
| Domain hijacked / DNS broken | Lock domain at registrar (set "registrar lock"). Contact registrar support. While broken: app is unreachable but data is safe. |
| WhatsApp templates rejected en masse | Customer notifications fall back to email-only (Workflow 01 design). System is degraded but not broken. Resubmit templates. |
| Data breach / suspected compromise | Rotate ALL secrets (section 18), invalidate all Supabase sessions (Supabase dashboard → Sessions → Revoke all), audit recent activity in `audit_events` table, notify customers per POPIA breach-notification rules within 72 hours if PII exposed. |

## 22. Contact tree

When something goes wrong, who do you call?

| Vendor | What for | How |
| --- | --- | --- |
| Hetzner | VPS issues | `support@hetzner.com` / console support ticket |
| Supabase | Auth issues | Dashboard support (free tier: community Discord; paid: email) |
| Postmark | Email deliverability | `support@postmarkapp.com` |
| Meta | WhatsApp template / API issues | Meta Business Help Centre (slow — allow days) |
| Cloudflare or your DNS registrar | DNS issues | Their support |

Internally:

| Role | Person | Reachable via |
| --- | --- | --- |
| Business owner | Lucia | lucia@gentick.co.za |
| First-line support / escalation | Michael | +27 82 374 9842 |
| Technical owner (developer) | TBC | TBC |

Keep this table updated. If your developer changes, this is one of the first things to revise.

---

## Appendix A — Useful one-liners

```bash
# How big is each Docker volume?
docker system df -v

# Free disk
df -h /

# Free memory
free -h

# Active database connections
docker compose -f docker-compose.prod.yml --env-file .env.prod exec postgres \
  psql -U gentick -d gentick_rms -c "SELECT count(*) FROM pg_stat_activity;"

# Largest tables
docker compose -f docker-compose.prod.yml --env-file .env.prod exec postgres \
  psql -U gentick -d gentick_rms -c "
SELECT schemaname, tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname='public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;"

# Recent failed webhook events
docker compose -f docker-compose.prod.yml --env-file .env.prod exec postgres \
  psql -U gentick -d gentick_rms -c "
SELECT event_type, last_error, delivery_attempts, emitted_at
FROM webhook_events
WHERE delivered_at IS NULL AND emitted_at > now() - interval '24 hours'
ORDER BY emitted_at DESC LIMIT 20;"

# Force-redispatch a stuck webhook
docker compose -f docker-compose.prod.yml --env-file .env.prod exec app \
  pnpm tsx scripts/redispatch-webhook.ts <event-id>

# Restart all services
docker compose -f docker-compose.prod.yml --env-file .env.prod restart
```

## Appendix B — Cost estimate (monthly, ZAR ex VAT, 2026 rates)

| Item | Cost |
| --- | --- |
| Hetzner CX22 (Cape Town) | R 100 |
| Hetzner snapshots (20% of server) | R 20 |
| Hetzner Storage Box 1TB (backups) | R 80 |
| Supabase (free tier — fine until ~500MB) | R 0 |
| Postmark (10k emails/mo plan) | R 280 |
| Meta WhatsApp Cloud API (utility messages, ZA rates ~R 0.05 / msg, est. 2k/mo) | R 100 |
| UptimeRobot (free tier) | R 0 |
| Domain rego (annualised) | R 20 |
| **Total monthly** | **~R 600** |

Foreign-currency line items priced at R18 = €1 / $1. Costs scale with volume — at 10× growth, expect Postmark and WhatsApp to be the main movers.
