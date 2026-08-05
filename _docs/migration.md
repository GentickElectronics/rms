# RMS Stack Migration Plan

## Status

| Phase | Status |
|---|---|
| Phase 1 — Go Backend | ✅ Complete (2026-08-04) |
| Phase 2 — Frontend | ✅ Complete (2026-08-05, `rms-frontend` agentic @ 6ef0f9a) |
| Phase 3 — Review | ✅ Complete (2026-08-05) — gaps logged below |
| Phase 4 — Addressing Review results | In progress |
| Phase 5 — Deploy & Test | Not started — IMAGES ON GHCR (rms-api + rms-frontend @ 0.0.0) |

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
├── rms-app/          ← DELETED after Phase 5
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

*Compiled 2026-08-05 from Phase 3 review (Pegasus — critical flows, Fluffy + Runner — simple parity). Severity in brackets. Deduplicated across reviewers; source in parens.*

### CRITICAL (deploy-blockers — must be resolved before cutover)
1. **[CRITICAL] n8n outbound dispatch missing entirely from Go API** (Pegasus). Old `lib/webhooks/dispatch.ts` + `sign.ts` POSTed HMAC-signed `webhook_events` envelopes to `N8N_WEBHOOK_URL` on transitions/quote/notification events. New `TransitionJob` records history but never dispatches. After cutover n8n gets ZERO outbound events → no stage-change/quote/notification dispatches.
2. **[CRITICAL] Audit log has no writer** (Fluffy). Go API reads `audit_events` (viewer works: `GET /api/v1/admin/audit`) but nothing INSERTs. Old `lib/audit/audit.ts` `recordAudit()` calls (settings/account-security/quote flow) not ported. Table not even created in Go migrations.
3. **[CRITICAL] File visibility not enforced server-side + photo `<img>` auth broken** (Fluffy, Runner). `GET /api/v1/files/{key}` and `GET /jobs/{id}/photos` apply no customer-vs-internal scoping (old used public HMAC-signed expiring URLs). `ServeFile` requires a Bearer header, but photos are rendered via `<img src>` (no header) → **portal photos return 401**. No signed-URL support; old emailed/WhatsApp links break.

### MAJOR (functional/security gaps)
4. **[MAJOR] Server-side RBAC not enforced on many admin/agent/customer endpoints** (Pegasus). `POST /admin/settings/{key}`, `POST /agents/invite`, agent/customer CRUD are `RequireAuth` only — frontend RoleGuard is the only gate. Direct API calls bypass. `InviteAgent` also lacks the old admin/front_desk/customer_contact check.
5. **[MAJOR] `CreateQuote`/`IssueQuote` have no authorization** (Pegasus). Only `RequireAuth`; a technician could issue a quote via direct API.
6. **[MAJOR] WhatsApp `classifyIntent` uses prefix match, not regex word-boundary** (Pegasus). "I don't know when it broke" classified "other" instead of "question" → missed follow-ups. `from_phone` E.164 validation also loosened.
7. **[MAJOR] Customer contact update/delete missing** (Runner). Only `GET/POST /customers/{id}/contacts`; old `updateContactAction`/`deleteContactAction` have no backend/frontend equivalent.
8. **[MAJOR] Customer contact missing update/delete**, admin-initiated password reset missing (Runner). Old `adminSetPasswordAction`/`adminSendResetEmailAction` have no endpoint.
9. **[MAJOR] Invoice PDF download broken in frontend** (Runner). `useInvoicePdfUrl` misuse (destructured as object), missing `/api/v1` prefix, `window.open` can't attach Bearer → always 401.
10. **[MAJOR] Report type regression** (Runner). Old CSV exports (aging / intake-source / technician-workload / revenue) not covered; only a flat all-jobs CSV remains.

### MINOR
11. MFA + email verification removed (Pegasus/Fluffy/Runner) — verify with Teo whether intentionally dropped.
12. Account/security page is a stub (`/profile`); admin password reset UI missing (Fluffy).
13. `GetJob` returns any job by UUID to any authenticated user (no role scoping) (Runner).
14. Photo key format drift: old `jobs/{id}/{file}` vs new `{id}/{uuid}_name` (Runner).
15. Sage export route changed: old `GET /jobs/{id}/sage-export` → `POST /invoices/sage-export`; `exported_to_sage` semantics not replicated (Runner).
16. Pre-registration flow simplified — staff-side pending queue + intake-from-pending gone (Runner).
17. No invoice auto-generation on quote approval; Sage XML unimplemented (CSV works) (Pegasus).
18. Frontend `RoleGuard` excludes customer contacts from quote page despite API supporting it (Pegasus).
19. WhatsApp `from_phone` E.164 validation dropped (Pegasus).
20. Agent pre-registration sends `originator_type`/`originator_id` the API silently overrides (Pegasus).
21. `/auth/me` shape mismatch: API returns `user_id`, frontend type expects `id` (Runner).
22. `GET /units/{id}/history` has no frontend consumer (serial-history feature unwired) (Runner).
23. `/portal/intake` has no dedicated route — only a dialog in AgentPortal (Fluffy).
24. Out of scope by design: Supabase OAuth `/auth/callback` removed (intentional).

---

## Phase 4 — Addressing Review results

**Deliverable:** Resolve every actionable gap from Phase 3 before Deploy & Test. All gaps tracked below; each is assigned to a worker agent. Items marked **DECISION** are not implementation tasks — they need Teo.

### Backend — `rms-api` (Pegasus)

- [x] **1. [CRITICAL]** n8n outbound dispatch missing entirely — port old `lib/webhooks/dispatch.ts` + `sign.ts` HMAC-signed `webhook_events` envelopes to `N8N_WEBHOOK_URL` on transition / quote / notification events. ✔ Pegasus 2026-08-05
- [x] **2. [CRITICAL]** Audit log has no writer — port `recordAudit()` writes; create the table in a new Go migration (e.g. 005). ✔ Pegasus 2026-08-05
- [x] **3a. [CRITICAL]** File visibility — enforce customer-vs-internal scoping on `GET /api/v1/files/{key}` and `GET /jobs/{id}/photos` server-side; support public HMAC-signed expiring URLs (so emails/WhatsApp/`<img>` links work without a Bearer header). ✔ Pegasus 2026-08-05
- [x] **4. [MAJOR]** Server-side RBAC on admin/agent/customer endpoints currently `RequireAuth`-only; add role checks (admin / front_desk / customer_contact as in old flows) to `POST /admin/settings/{key}`, `POST /agents/invite`, agent/customer CRUD, etc. ✔ Pegasus 2026-08-05
- [x] **5. [MAJOR]** `CreateQuote` / `IssueQuote` authorization — must not be issuable by a technician via direct API. ✔ Pegasus 2026-08-05
- [x] **6. [MAJOR]** WhatsApp `classifyIntent` — regex word-boundary matching, not prefix; restore strict E.164 `from_phone` validation. ✔ Pegasus 2026-08-05
- [x] **13. [MINOR]** `GetJob` — restrict UUID access by role (customer/agent scoping). ✔ Pegasus 2026-08-05
- [x] **14. [MINOR]** Photo key format drift — reconcile old `jobs/{id}/{file}` vs new `{id}/{uuid}_name`. ✔ Pegasus 2026-08-05
- [x] **15. [MINOR]** Sage export semantics — reconcile old `GET /jobs/{id}/sage-export` vs new `POST /invoices/sage-export`; replicate `exported_to_sage`. ✔ Pegasus 2026-08-05
- [x] **17. [MINOR]** Invoice auto-generation on quote approval; Sage XML export (CSV already works). Invoice auto-gen done; **Sage XML still unimplemented** (CSV works for now) — carried into Phase 5. ✔ Pegasus 2026-08-05 (auto-gen) / ⚠️ Sage XML open
- [x] **20. [MINOR]** Agent pre-registration — stop silently overriding `originator_type` / `originator_id`. ✔ Pegasus 2026-08-05

### Full-stack — customer/agent contacts (Runner)

- [x] **7. [MAJOR]** Customer contact **update/delete** — backend endpoint + frontend UI (only GET/POST `/customers/{id}/contacts` exists). PATCH/DELETE `/customers/{id}/contacts/{contactId}` (Pegasus, rms-api 0b0bb9f) + edit/delete UI in CustomersPage contact rows (Runner). ✔ Runner 2026-08-05
- [x] **8. [MAJOR]** Customer contact update/delete + admin-initiated password reset (`adminSetPasswordAction` / `adminSendResetEmailAction`) — add API + UI. API: `POST /admin/users/{entity_type}/{id}/send-reset-email` + `/set-password` + `POST /auth/change-password` (Pegasus, rms-api 0b0bb9f). UI: `UserResetActions` dropdown on Staff/Agents/Customers-contact rows + admin reset panel on /profile (Runner). ✔ Runner 2026-08-05

### Frontend — `rms-frontend` (Runner)

> ⚠️ **BUILD-UNVERIFIED:** All frontend gaps below are implemented and committed on `agentic`, but **none have been compiled**. No agent host (Windows, WSL) has Node.js, so `npm run build` / `tsc` has **not** been run on the Phase-4 frontend changes. **This is the gating blocker for Phase 5** — the frontend must build cleanly and be smoke-tested before cutover. Enable Node on a host (or run the Phase 5 Docker build) to verify.
>
> 🔎 **Peer review (Pegasus 2026-08-05):** Runner's frontend (commits `b40f117`+`b749802`) independently reviewed against backend contracts. Verdict: **6 gaps VERIFIED clean** (3b, 7/8, 12, 18, 21, 23), **2 NEEDS-FIX (9, 22)**, 0 regressions, build risk low. 3 action items:
> 1. **Gap 9** — `api.get<Blob>` + `responseType:"blob"` in `useInvoices.ts` TS type risk → **fixed** (`AxiosResponse<Blob>` cast, `e078d72`).
> 2. **Gap 22** — `useUnit()`/`useUnitHistory()` fired wasted calls on every job load → **fixed** (gated, +Skeleton loading, `e078d72`).
> 3. **UX (7/8)** — `UserResetActions` shown to `front_desk` → **fixed** (admin-only guard in component, `e078d72`).

- [x] **3b. [CRITICAL]** Photos — switch portal/message `<img>` to signed URLs (or otherwise) so they don't 401 (no Bearer header on `<img>`). JobDetailPage + PortalJobDetailPage now render `photo.signed_url` from ListJobPhotos via `getPhotoUrl()` helper; also fixed staff img URL missing the `files/` segment. ✔ Runner 2026-08-05
- [x] **9. [MAJOR]** Invoice PDF download broken — fix `useInvoicePdfUrl`, add `/api/v1` prefix, handle auth for `window.open` / download. Replaced with `downloadInvoicePdf()` (api client attaches Bearer + auto-refreshes, blob download from `content-disposition` filename). ✔ Runner 2026-08-05
- [x] **12. [MINOR]** Account/security page (`/profile`) is a stub; add admin password reset UI. /profile is now a real Account & Security page: self-service change password + admin-only reset panel (staff/agents). MFA NOT implemented (per decision gap 11). ✔ Runner 2026-08-05
- [x] **18. [MINOR]** `RoleGuard` excludes customer contacts from quote page although API supports it. Added `main_customer_contact` to the quotes route; also fixed route param (`/jobs/:jobId/quotes`) so QuotesPage's `useParams<{jobId}>` actually resolves. ✔ Runner 2026-08-05
- [x] **21. [MINOR]** `/auth/me` shape mismatch — API returns `user_id`, frontend type expects `id`. `meToUser()` normalizes `user_id` → `id` in auth.tsx (bootstrap + refreshUser). ✔ Runner 2026-08-05
- [x] **22. [MINOR]** `GET /units/{id}/history` unwired — serialize-history feature has no frontend consumer. Wired via `useUnitHistory` + `useUnit` into JobDetailPage details tab ("Unit Repair History" card). ✔ Runner 2026-08-05
- [x] **23. [MINOR]** `/portal/intake` needs a dedicated route (currently only a dialog in AgentPortal). Added `/portal/intake` (PortalIntakePage); pre-registration form extracted to shared `PreRegisterForm.tsx`; AgentPortal now links to the route. ✔ Runner 2026-08-05

### Flows — reporting + pre-registration (Fluffy)

- [x] **10. [MAJOR]** Report type regression — restore CSV exports: aging / intake-source / technician-workload / revenue (only a flat all-jobs CSV remains). Restored — backend: `GET /reports/aging|intake-source|technician-workload|revenue` (JSON) + `GET /reports/export?type=aging|intake-source|workload|technician-workload|revenue` (CSV, same columns/filenames as old app); frontend: 4 "Report Exports" cards on ReportsPage with CSV buttons. ✔ Fluffy 2026-08-05
- [x] **16. [MINOR]** Pre-registration flow — restore staff-side pending queue + intake-from-pending. Restored — backend: `GET /jobs/pre-registrations` (admin/front_desk, agent filter) + JobIntake auto-links matching serial to the new shipment (no duplicate jobs); frontend: pending pre-registrations panel on IntakePage with "Add to intake" prefill. ✔ Fluffy 2026-08-05

### Decision needed (Teo)

- [ ] **11. [DECISION]** MFA + email verification removed — confirm whether intentionally dropped.
- [ ] **24. [INTENTIONAL]** Supabase OAuth `/auth/callback` removed by design — no action.

---

## Phase 5 — Deploy & Test

**Deliverable:** Full stack (Go + React) running in compose. rms-app shut down.

- [ ] Add `rms-frontend` to compose.yml
- [ ] Nginx: `/api/*` → rms-api, `/*` → rms-frontend
- [x] Dockerfile for rms-frontend (multi-stage node→nginx, commit 3679a23) — image on ghcr
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

## Phase 5 — Image build status (Oda, 2026-08-05)

Both app images built multi-arch (linux/amd64 + linux/arm64) via the `img-build.sh` flow (docker buildx build --platform linux/amd64,linux/arm64 --push) and pushed to GHCR at version `0.0.0` (VERSION untouched):

| Image | GHCR ref | Digest |
|---|---|---|
| rms-api | `ghcr.io/gentickelectronics/rms-api:0.0.0` | `sha256:2436b7b7…` |
| rms-frontend | `ghcr.io/gentickelectronics/rms-frontend:0.0.0` | `sha256:2788a238…` |

Notes:
- `img-build.sh` is interactive (readline `read -e`); driving it non-interactively yields an empty tag, so its exact buildx command was run directly per image. Script + flow otherwise unchanged.
- First-ever compile of rms-frontend happened here — surfaced then fixed all TypeScript build errors (commits 38c3338, 11008d2). Confirms the SPA now builds cleanly.

---

## Key Risks

| Risk | Mitigation |
|---|---|
| Auth cutover: existing users need new passwords | Password reset flow is live. Send reset emails at cutover |
| n8n webhook format mismatch | n8n calls HTTP endpoints — unchanged. Verify payload signatures |
| Frontend regressions | Archive rms-app, don't delete. Can roll back by switching nginx |
| DB migration failure | Run on staging first. All migrations are additive |
