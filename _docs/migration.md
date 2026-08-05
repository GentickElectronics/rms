# RMS Stack Migration Plan

## Status

| Phase | Status |
|---|---|
| Phase 1 — Go Backend | ✅ Complete (2026-08-04) |
| Phase 2 — Frontend | ⬜ Pending |
| Phase 3 — Review | ⬜ Pending |
| Phase 4 — Deploy & Test | ⬜ Pending |

## Goal

Full cutover from Next.js monolith + Supabase → Go REST API + standalone React SPA.
No parallel auth — a single cutover with a test phase before production switch.

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

**Repo structure after migration:**

```
rms (umbrella)
├── rms-api/          ← Go backend (new)
├── rms-frontend/     ← React SPA (new)
├── rms-infra/        ← compose, nginx, n8n (updated)
├── rms-app/          ← DELETED after Phase 4
└── _docs/
```

---

## Phase 1 — Go Backend (`rms-api`) ✅ Complete

**Deliverable:** Go REST API serving every endpoint. Custom JWT auth — Supabase eliminated.

### What was built

- **Auth** — JWT issue/validate/refresh, bcrypt password hashing, login/logout/invitation flows, password reset. Refresh tokens stored in DB (revocable, rotated on use).
- **Jobs** — CRUD, transitions (stage state machine), job numbering, intake (multi-unit booking), technician queue
- **People** — customers, agents, staff. CRUD + role management + invitations
- **Quotes & Invoices** — quote creation/approval/rejection, invoice generation, Sage export, PDF invoice generation
- **Parts, Photos, Notes** — per-job logging, file upload/serve with visibility rules
- **Reports** — operational rollups, CSV export, dashboard KPIs
- **Admin** — company settings, repair taxonomy, message templates, audit log
- **Webhooks** — WhatsApp inbound, outbound dispatch to n8n
- **Email** — SMTP sender for password resets (graceful degradation if not configured)

### Migrations applied

| Migration | Description |
|---|---|
| 001 | Initial schema (Lucia, rms-app) |
| 002 | Messages columns for API compatibility |
| 003 | Auth columns (password_hash, refresh_token_hash, refresh_tokens) |
| 004 | Password reset tokens |

### Project Layout

```
rms-api/
├── cmd/api/          ← entry point, router, middleware stack
├── internal/
│   ├── auth/         ← JWT, bcrypt, middleware
│   ├── db/           ← pgxpool, migrations
│   ├── handler/      ← HTTP handlers per domain
│   ├── middleware/   ← CORS, logging, recovery, request ID
│   ├── model/        ← domain types
│   ├── config/       ← env config
│   ├── email/        ← SMTP sender
│   └── service/      ← business logic
├── Dockerfile
└── go.mod
```

---

## Phase 2 — Frontend Migration (`rms-frontend`)

**Deliverable:** Vite + React SPA replacing every Next.js page. Uses Go API for all data.
No Supabase. No Drizzle. No direct Postgres connection.

**Tech:** Vite, React 19, React Router v7, TanStack Query, Tailwind v4 + shadcn/ui.

### Phase 2a — Foundation + Auth

- Project scaffolding: Vite, TypeScript, Tailwind, shadcn/ui, build config
- UI kit: all `components/ui/*` ported (Button, Card, Table, Sidebar, Topbar, Dialog, etc.)
- API client: Axios/fetch wrapper with JWT injection, auto-refresh on 401, error handling
- Auth context: user state, login/logout/refresh, role-based guards
- React Router: route tree, layout shell, protected route wrappers
- Auth pages: login, accept-invitation, forgot-password, reset-password
- Root redirect: role-based (admin → dashboard, technician → queue, etc.)

### Phase 2b — Staff Core

- Dashboard: KPI grid, active jobs table, priorities, fault chart
- Technician queue
- Job detail: stage timeline, transition panel, notes, photos, parts
- Intake form: multi-unit booking with photo upload

### Phase 2c — Financial + Admin

- Quote creation, approval, rejection
- Invoice view + PDF download
- Customer management (CRUD)
- Agent management (CRUD + invitations)
- Staff management (CRUD + role assignment)
- Parts catalogue
- Repair types taxonomy
- Message templates
- Company settings
- Audit log viewer
- Sage export

### Phase 2d — Reports + Portal

- Operational reports with CSV export
- Customer portal (job tracking, read-only)
- Agent portal (pre-registration, job tracking)

---

## Phase 3 — Review

**Deliverable:** Side-by-side comparison confirming rms-api + rms-frontend cover everything rms-app did.

### Checklist

- [ ] Every rms-app API route has a matching rms-api endpoint
- [ ] Every rms-app page has a matching rms-frontend route
- [ ] Auth flows: login, logout, refresh, invite, password reset
- [ ] All role-based permissions match (admin, technician, front_desk, customer, agent)
- [ ] Job lifecycle: intake → all transitions → close
- [ ] Quote lifecycle: create → issue → approve/reject
- [ ] Invoice: generation, PDF download, Sage export
- [ ] File upload/view with visibility rules
- [ ] WhatsApp webhooks: inbound → intent classification → auto-approve
- [ ] n8n dispatch: outbound events (quote issued, stage change, etc.)
- [ ] Dashboard KPIs and technician queue
- [ ] Audit log: creation + viewer
- [ ] Settings CRUD, message templates, repair types
- [ ] Customer/agent portal flows

### Gaps found

*(Populated during review)*

---

## Phase 4 — Deploy & Test

**Deliverable:** Full stack (Go + React) running in compose. rms-app shut down.

- [ ] Add `rms-frontend` to compose.yml
- [ ] Nginx: `/api/*` → rms-api, `/*` → rms-frontend
- [ ] Dockerfile for rms-frontend
- [ ] CI: add rms-frontend to img-build.sh flow
- [ ] DB migration run on staging
- [ ] Integration test: full user flows
  - Login → dashboard → intake → transition → quote → invoice
  - Agent invitation → accept → login → view jobs
  - Admin: staff CRUD, settings, audit log
- [ ] Smoke test all screens
- [ ] Archive rms-app (keep repo, remove from compose)
- [ ] Drop `supabase_user_id` columns
- [ ] Remove Supabase env vars

---

## Key Risks

| Risk | Mitigation |
|---|---|
| Auth cutover: existing users need new passwords | Password reset flow is live. Send reset emails at cutover |
| n8n webhook format mismatch | n8n calls HTTP endpoints — unchanged. Verify payload signatures |
| Frontend regressions | Archive rms-app, don't delete. Can roll back by switching nginx |
| DB migration failure | Run on staging first. All migrations are additive |
