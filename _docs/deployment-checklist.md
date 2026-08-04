# Gentick RMS — Production Deployment Checklist

**Target:** gentickit server, alongside gentick-infra (shared Postgres, nginx, Cloudflare Tunnel)
**Based on:** `rms/_docs/migrate_plan.md` (2026-07-31)
**RMS footprint:** 2 containers only — `gentick-app` + `gentick-n8n`
**Date prepared:** 2026-08-04

Suggested owners are marked **[Lucia]** / **[Teo]** — swap as you see fit.

---

## Phase 0 — Decisions (before anything else)

- [ ] **Supabase project** — dedicated `gentick-rms-prod` project, or share the existing TalosOT one? *(Recommended: dedicated — keeps RMS users, MFA settings and email templates fully separate.)* **[Lucia + Teo]**
- [ ] **Image registry namespace** — `ghcr.io/gentickelectronics/...` (org standard) vs personal. *(Recommended: the org, per Gentick standards. This checklist assumes `ghcr.io/gentickelectronics/rms-app`.)* **[Teo]**
- [ ] **n8n** — confirm RMS keeps its own n8n container (the plan recommends yes; its 5 workflows carry WhatsApp + Postmark credentials). **[Teo]**
- [ ] Confirm which branch/commit is being deployed — `rms-app` `main` (now the full codebase, HEAD `9b11c8e` or later). **[Lucia]**

## Phase 1 — External accounts (start early — WhatsApp is the long pole)

- [ ] **Meta WhatsApp Business** — start business verification NOW (1–10 business days). Needed for customer notifications. **[Lucia]**
- [ ] **Supabase** — create the production project; note the `SUPABASE_URL`, `ANON_KEY`, `SERVICE_ROLE_KEY` from the dashboard. Configure: email templates, MFA enabled, redirect URL `https://rms.gentick.co.za/auth/callback`. **[Lucia]**
- [ ] **Postmark** — create account, verify the `repairs@gentick.co.za` sender signature/domain (SPF + DKIM DNS records). **[Lucia]**
- [ ] **GHCR** — confirm push rights to `ghcr.io/gentickelectronics` for whoever builds. **[Teo]**

## Phase 2 — Build & push the app image (on the build machine)

Prereqs: Docker Desktop, Node.js 20 LTS, pnpm 9+, `docker login ghcr.io`.

- [ ] Pull latest `rms-app` `main`; confirm it builds clean: `pnpm install && pnpm build`. **[Teo]**
- [ ] Build the production image:
      `docker build -t ghcr.io/gentickelectronics/rms-app:1.0.0 .`
- [ ] Push: `docker push ghcr.io/gentickelectronics/rms-app:1.0.0`
- [ ] Tag the release in git (`v1.0.0`) and update `_docs/VERSION` + changelog. **[Teo]**

## Phase 3 — Server one-time setup (on gentickit, via SSH)

- [ ] Create the RMS database on the shared Postgres:
      `docker exec -it gentick-postgres psql -U gentick -c "CREATE DATABASE gentick_rms;"`
- [ ] Create the n8n database the same way (`CREATE DATABASE n8n;`) if it doesn't exist yet.
- [ ] Add the two nginx routes to gentick-infra's `nginx.conf` (`rms.gentick.co.za` → `gentick-app:3000`, `automation.gentick.co.za` → `gentick-n8n:5678`) and reload nginx. **[Teo]**
- [ ] Add both hostnames to the Cloudflare Tunnel / DNS. **[Teo]**
- [ ] Create directories:
      `sudo mkdir -p /opt/gentick-rms /data/rms-uploads && sudo chown -R deploy:deploy /opt/gentick-rms /data/rms-uploads`
- [ ] Place `compose.yml` (from `rms-infra`) at `/opt/gentick-rms/`.
- [ ] Create `/opt/gentick-rms/.env` with **mode 600** (`chmod 600 .env`):
  - `POSTGRES_USERNAME` / `POSTGRES_PASSWORD` — from gentick-infra's .env
  - `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` — from Phase 1
  - `WEBHOOK_SECRET`, `STORAGE_URL_SECRET`, `N8N_ENCRYPTION_KEY` — generate each with `openssl rand -hex 32`
  - `N8N_BASIC_AUTH_USER=lucia`, `N8N_BASIC_AUTH_PASSWORD` — generate with `openssl rand -base64 16`
- [ ] ⚠️ The `SERVICE_ROLE_KEY` lives ONLY in this file — never in git, chat, or a browser.

## Phase 4 — Deploy

- [ ] `docker pull ghcr.io/gentickelectronics/rms-app:1.0.0`
- [ ] `cd /opt/gentick-rms && docker compose up -d`
- [ ] Run migrations: `docker compose exec app pnpm db:migrate`
- [ ] Create the first admin user: add Lucia in Supabase Auth dashboard, then insert the matching `staff_users` row (script `scripts/create-portal-users.ts` / SQL per runbook).
- [ ] Import the 5 n8n workflows and connect WhatsApp + Postmark credentials inside n8n. **[Teo/Lucia]**

## Phase 5 — Verify

- [ ] `docker compose ps` — both containers healthy.
- [ ] Health check: `curl http://gentick-app:3000/api/v1/health` → 200 (from the server).
- [ ] `https://rms.gentick.co.za` shows the login screen; log in as admin (with MFA).
- [ ] `https://automation.gentick.co.za` prompts for the n8n basic-auth login.
- [ ] End-to-end smoke test: create a test intake → move it through a stage transition → confirm the WhatsApp/email notification fires → generate a quote PDF/print view → check the audit log recorded it all.
- [ ] Portal check: log in as a test agent and as the main customer; confirm they see only their own jobs.

## Phase 6 — After go-live

- [ ] **Backups** — confirm gentick-infra's Postgres backup job includes the new `gentick_rms` database, and that `/data/rms-uploads` (photos) is backed up too. ⚠️ Not covered in the migration plan — worth an explicit check. **[Teo]**
- [ ] Rollback plan noted: `docker compose down`, re-point compose at the previous image tag, `docker compose up -d`.
- [ ] Record the deployed version + date in `rms/_docs/changelogs/current.md`.
- [ ] Decommission/archive the old standalone runbook assumptions (own Postgres/Caddy) so nobody follows them by mistake.
- [ ] Diarise Supabase/Postmark/Meta credential renewal + billing checks.

---

*Nothing is installed on staff or agent computers — the RMS is a web app; a browser is all they need.*
