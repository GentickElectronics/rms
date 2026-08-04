# RMS Migration — What Changed

**Source:** `RepairManagementSystem/` (Lucia's original)
**Target:** `rms/` (Teo's cleaned migration)
**Date:** 2026-07-31

---

## Structural Changes

| Original | Migrated | Notes |
|---|---|---|
| `gentick-rms/` | `rms-app/` | App code only — Dockerfile moved to root |
| `gentick-rms-infra/` | `rms-infra/` | Ops: compose, Caddy, n8n workflows |
| `gentick-rms-app/` | **dropped** | Skeleton with README stubs, empty dirs, CI templates only |
| 19 loose root files (.md, .yaml, .sql, .html, .svg) | umbrella `_docs/` | Design artifacts: requirements, API spec, ERD, wireframes |

## Dropped (Intentionally)

- **`components/ui/`** — empty directory. Zero imports in the codebase.
- **`gentick-rms-app/`** — skeleton repo with only placeholder READMEs and CI config templates. No actual app code.
- **`scripts/`** (build.sh, deploy-*.sh, pull.sh) — old standalone deploy scripts. The new compose integrates with gentick-infra; rebuild these when needed.
- **`ac/`** (acceptance criteria), `.github/` (CI), `CHANGELOG.md`, `CONTRIBUTING.md` — process artifacts. Recreate from gentick standards when CI is set up.
- **App-level compose files** (`docker-compose.yml`, `docker-compose.prod.yml`) — replaced by `rms-infra/compose.yml`.
- **`gentick-rms-app/.github/`** — CI workflows, CODEOWNERS, PR template. Not ported.

## Key Architectural Shift

The original ran as a **standalone** stack (own Postgres, Caddy for TLS). The migration follows the gentick-infra pattern: RMS runs alongside gentick-infra on gentickit, **sharing** Postgres, Nginx, and Cloudflare Tunnel.

**RMS now deploys only 2 containers:** `app` + `n8n`. Everything else is gentick-infra's.

## compose.yml Changes (This Session)

- Postgres + Caddy services **removed** (provided by gentick-infra)
- Joined `gentick-infra_backend-net` (external network)
- Added `logging` (local driver, 1 MB / 3 files — matches gentick-infra)
- Added `healthcheck` blocks with `start_period`
- `container_name` prefixed `gentick-` for consistency
- `DATABASE_URL` now points to gentick-infra's Postgres via env vars

## env Changes

- `rms-infra/.env.example` — infra-side vars only (Postgres creds, Supabase, n8n secrets). Old copy was the app-level .env.example; now matches the new compose.
- `rms-app/.env.example` — **new**, for local dev. Was missing from the migration.

## Still Present (Not Yet Used)

- `rms-infra/caddy/Caddyfile` — kept for reference. Not used (nginx handles routing now).
- `rms-app/Dockerfile` — verified paths work from new root location. `public/` kept (was empty; `.gitkeep` added so Docker COPY doesn't fail).
