# RMS Stack Migration Plan

## Status

| Phase | Status |
|---|---|
| Phase 1 — Go Backend | ✅ Complete |
| Phase 2a — Foundation + Auth | ⬜ Pending |
| Phase 2b — Staff Core | ⬜ Pending |
| Phase 2c — Financial + Admin | ⬜ Pending |
| Phase 2d — Reports + Portal | ⬜ Pending |
| Phase 3 — Deploy & Test | ⬜ Pending |
| Phase 4 — Purge | ⬜ Pending |

## Goal

Decompose the Next.js monolith into a Go REST API + standalone React frontend.
Eliminate Supabase — auth moves to Go with JWT + bcrypt.
Four phases, each producing a working system. No big-bang rewrites.

## Target Architecture

```
                    ┌── nginx ──┐
   Browser ────────>│  /api/* ───> rms-api (Go)
                    │  /* ───────> rms-frontend (React SPA)
                    └────────────┘
                            │
                      PostgreSQL 16
                       (unchanged)
```

No separate nexus/traffiker. RMS has no MQTT devices — one Go binary handles all REST.

**Repo structure after migration:**

```
rms (umbrella)
├── rms-api/          ← Go backend (new)
├── rms-frontend/     ← React SPA (new)
├── rms-infra/        ← compose, nginx, n8n (updated)
├── rms-app/          ← DELETED in Phase 4
└── _docs/
```

---

## Phase 1 — Go Backend (`rms-api`)

**Deliverable:** Go REST API serving every endpoint the Next.js app currently provides.
Deployed alongside Next.js. Nginx routes `/api/*` → Go, everything else → Next.js.
Zero frontend changes.

### What Moves

The backend is split into logical domains. Each becomes a Go package:

- **Auth** — JWT issue/validate/refresh, bcrypt password hashing, login/logout/invitation flows. Replaces Supabase entirely. Refresh tokens stored in DB (revocable, rotated on use).
- **Jobs** — CRUD, transitions (stage state machine), job numbering, intake (multi-unit booking), technician queue
- **People** — customers, agents, staff. CRUD + role management + invitations
- **Quotes & Invoices** — quote creation/approval/rejection, proforma invoice generation, Sage export
- **Parts, Photos, Notes** — per-job logging, file upload/serve with visibility rules
- **Reports** — operational rollups, CSV export, dashboard KPIs
- **Admin** — company settings, repair taxonomy, message templates, audit log
- **Webhooks** — WhatsApp inbound, outbound dispatch to n8n
- **Communications** — email/WhatsApp delivery (proxied through n8n, unchanged)

### Auth Migration

Supabase's `auth.users` table disappears. Auth moves into existing tables:

- `staff_users` gains `password_hash`, `refresh_token_hash`
- `customer_contacts` gains the same
- `agent_invitations` gains `password_hash`
- New `refresh_tokens` table for session management

A migration script hashes passwords for existing users. Production cutover forces a password reset.

### Schema

The existing Postgres schema stays. Drizzle migrations → Go migrations (same DDL, different tool).
Auth columns are additive — nothing breaks existing queries. The `supabase_user_id` FK is dropped last,
after both auth systems have run in parallel.

### Project Layout

```
rms-api/
├── cmd/api/          ← entry point, router, middleware stack
├── internal/
│   ├── auth/         ← JWT, bcrypt, middleware
│   ├── db/           ← sqlc queries, migrations
│   ├── handler/      ← HTTP handlers per domain
│   └── service/      ← business logic
└── Dockerfile
```

---

## Phase 2 — Frontend Migration (`rms-frontend`)

**Deliverable:** Vite + React SPA replacing every Next.js page. Uses Go API for all data.
Split into sub-phases ordered by dependency.

**Tech:** Vite, React 19, React Router v7, TanStack Query, Tailwind v4 + shadcn/ui (all port directly).

### Phase 2a — Foundation + Auth

- Project scaffolding, build config, CSS
- UI kit: all `components/ui/*` port as-is (Button, Card, Table, Sidebar, Topbar, etc.)
- API client with JWT injection, auto-refresh, error handling
- Auth context: user state, login/logout/refresh
- React Router: route tree, layout shell, role-based guards
- Auth pages: login, callback, accept-invitation, password reset, MFA
- Root redirect (role-based)

### Phase 2b — Staff Core

- Dashboard: KPI grid, active jobs table, priorities, fault chart
- Technician queue
- Job detail: stage timeline, transition panel, notes, photos, parts
- Intake form: multi-unit booking with photo upload (most complex UI in the app)

### Phase 2c — Financial + Admin

- Quote creation/approval/rejection
- Proforma invoice (printable)
- Customer management (CRUD)
- Agent management (CRUD + invitations)
- Staff management (CRUD + role assignment)
- Parts catalogue
- Repair types taxonomy
- Message templates
- Company settings
- Audit log viewer
- MFA setup + admin password reset

### Phase 2d — Reports + Portal

- Operational reports with CSV export
- Customer portal (job tracking, read-only)
- Agent portal (pre-registration, job tracking)

---

## Phase 3 — Deploy & Test

**Deliverable:** Full stack (Go + React) deployed in compose. Next.js still running as fallback.

- Add `rms-api` and `rms-frontend` to compose.yml
- Nginx: `/api/*` → Go, `/*` → React SPA
- Dockerfiles for both new services
- CI: add to `img-build.sh` flow
- DB migration run on staging
- Seed data with hashed passwords
- Integration test: full user flows (login → dashboard → intake → transition → quote → invoice)
- Smoke test all ~20 screens

**Rollback:** Keep Next.js on a backup port. Flip nginx back if anything breaks. Zero downtime risk.

---

## Phase 4 — Purge

**Deliverable:** Next.js and Supabase fully decommissioned.

- Remove `rms-app` from compose, umbrella, and GitHub (archive, don't delete)
- Remove `SUPABASE_*` env vars
- Drop `staff_users.supabase_user_id` column
- Delete Supabase cloud project (or archive)
- Update docs: README, architecture diagrams, runbooks

---

## Key Risks

| Risk | Mitigation |
|---|---|
| Auth migration breaks existing sessions | Run both auth systems in parallel. Force password reset at production cutover |
| API parity gaps | Automated comparison tests against both backends during Phase 1 |
| n8n webhook format mismatch | n8n calls HTTP endpoints — unchanged. Verify payload signatures |
| Frontend regressions | Keep Next.js running as rollback target throughout Phase 3 |
| DB migration failure | Run on staging first. Nullable columns, drop FKs last |
