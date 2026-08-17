# RMS Migration Audit — 2026-08-12

Consolidated findings from six independent audits of the Next.js → Go + React
migration.

> **Note (2026-08-14):** this audit critiques the original stack migration plan
> (the Phase-4 self-assessment), which now lives verbatim as **Part 1 of
> [`plan.md`](./plan.md)**. Inline `migration.md:NN` references cite that
> original document; the line numbers refer to the original file, preserved as-is
> in `plan.md` Part 1.

## Verdict

### Remediation status as at 2026-08-13

- **Findings tracked:** 74 (`B1`–`B6`, `S1`–`S17`, `D1`–`D12`, `W1`–`W12`,
  `V1`–`V18`, `I1`–`I9`).
- **Struck through (fixed — commit- or decision-verified):** 53.
- **Partial (code landed, not fully resolved):** 2 — `D1`, `D2`. Both landed
  on the backend (`aac7649`, `5c4415f`) *during this bookkeeping pass*, committed
  by a concurrent agent on the same shared `rms-api` tree; the frontend has not
  yet been migrated to consume the breaking wire-format change, so neither is
  struck through.
- **Outstanding, no commit found:** 19 — `D3`, `D4`, `D5`, `D6`, `D7`, `D8`,
  `D10`, `D11`, `D12`; `W3`, `W4`, `W6`, `W8`; `V13`; `I2`, `I3`, `I7`, `I8`, `I9`.
- **Test suite:** **4 top-level failures**, down from 16 at the Sprint 1
  baseline. The four: three migration-runner defects in `internal/db`
  belonging to `T-66` (`I7`/`I8`, not yet fixed — see below), plus
  `TestUnknownRouteAndWrongMethodUseTheStandardEnvelope`.
- **Live-server caveat:** `W9` is fixed in code but inert on the live server —
  `001_baseline.sql` has never been applied to `gentick_rms`, so `entity_id`
  is still `NOT NULL` there. See the `W9` row.
- **Moving-target note:** `rms-api`, `rms-frontend` and `rms-infra` all had
  agents actively committing during this pass (backend Sprint 3 money/VAT work
  landed mid-review — see `D1`/`D2`). This snapshot is accurate against
  `rms-api@aac7649`, `rms-frontend@cef3083`, `rms-infra@1ff1069`. Anything
  landed after those commits is not reflected here.

**The migration is not close to shippable, and `migration.md`'s Phase-4
checkmarks cannot be relied on.** The Go API cannot create a job — the core
operation of the product. `SELECT count(*) FROM jobs` on the live
`gentick_rms` database, up since 2026-08-04, returns **0**.

Six Phase-4 items marked ✔ could not be substantiated in the code: gap 1
(webhooks), gap 2 (audit), gap 3a (photo visibility), gap 4 (RBAC), gap 15
(`exported_to_sage`), gap 17 (transition-driven invoices).

> **2026-08-13 update on the six unsubstantiated items:** gap 1 → `W1`, now
> fixed (`36142fb`). gap 2 → `W9`, fixed in code but inert on live (see above).
> gap 3a → `S15`/`S16`, fixed (`bdccbc0`). gap 4 → substantiated by
> `eb3cdbc`/`T-29b`'s `TestRouterRoleGatesAdmitExactlyTheDeclaredRoles` plus the
> `S1`/`S5` wiring. gap 15 → `D5`, still open. gap 17 → `D4`, still open.

## Why the record is unreliable

The `rms-app` working tree is a **37-file husk**. Squash commit `531147f`
deleted ~150 files from `main`; the last complete tree is `9b11c8e` (186
files). Two auditors discovered this independently and re-extracted the real
tree before comparing.

If any Phase-3 review read the working tree, it compared the new stack against
a stub. That is the most plausible explanation for how twenty-odd gaps were
closed while everything below went unnoticed. **Any future parity work must
use `git archive 9b11c8e`, not the checkout.**

---

## 1. Blockers — nothing else is testable until these land

| # | Finding | Where |
|---|---|---|
| B1 | ~~`001_initial_schema.sql` exists only in `rms-app`; the Dockerfile copies only `internal/db/migrations`. A fresh database cannot migrate — `002` onward ALTERs tables nothing created, and the API crash-loops at boot~~ **FIXED — `rms-api@b35e678`** (T-10, DEC-1: `001`–`006` squashed into one idempotent `001_baseline.sql`) | `rms-api/Dockerfile:23` |
| B2 | ~~`job_number_seq` is referenced three times and created by **no** migration. Confirmed absent from the live DB~~ **FIXED — `rms-api@b35e678`** (T-10 creates the sequence; `5170084`/T-12 then removes every handler reference to it in favour of per-prefix locking — see B3. It is still read by dead code in `internal/service/jobs.go`, tracked as open ticket T-68) | `handler_jobs.go:454`, `:1050`, `service/jobs.go:46` |
| B3 | ~~Generated job numbers are `ACME-00001`; the live `chk_job_number_format` requires `^GT-[A-Z0-9]{2,10}-[0-9]{4,}$`. `GT-` appears nowhere in the Go tree~~ **FIXED — `rms-api@5170084`** (T-12, DEC-12: per-customer-prefix `MAX(job_number)` under a row lock, replacing the global sequence) | `handler_jobs.go:461`, `:1058` |
| B4 | ~~Intake does `ON CONFLICT (serial_number)`; the only unique index is on `lower(serial_number)`. Postgres raises 42P10 unconditionally~~ **FIXED — `rms-api@b35e678`** (T-10, conflict target corrected to `lower(serial_number)` to match the existing index) | `handler_jobs.go:1033` |
| B5 | ~~`INSERT INTO messages` omits `direction` and `channel` — both `NOT NULL`, no default. Every inbound WhatsApp message 500s~~ **FIXED — `rms-api@8835919`** (T-13) | `handler_misc.go:319` |
| B6 | ~~`shipmentOrigin("main_customer")` returns `"customer"`, not a label in `shipment_origin_enum`; `front_desk` → `walk_in` then violates `chk_shipment_originator`~~ **FIXED — `rms-api@8c403ec`** (T-14) | `handler_jobs.go:1136` |

All six Sprint 1 blockers are closed and each has its own commit — checked
individually against the actual commits rather than assumed.

### On B1 — the fix is not "copy the file across"

Dropping `001` in verbatim breaks the **existing** server: it would run
`CREATE TYPE job_stage_enum` on a type that already exists and exit 1. The
live `schema_migrations` has no `001` row because `rms-app` created those 28
tables out of band.

**Recommended:** port `001` into `rms-api` rewritten idempotent — guarded
`CREATE TYPE`, `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`,
`DROP TRIGGER IF EXISTS` before `CREATE TRIGGER`, and `ON CONFLICT DO NOTHING`
on the seeds. Valid conflict targets already exist (`uq_fault_reasons_code`,
`uq_settings_key_version`); without them a re-run double-seeds `fault_reasons`
and violates `uq_settings_key_version`.

Then a new `007` for the genuinely-missing objects: `job_number_seq` (B2), the
`units` conflict target (B4), and the three `webhook_events` indexes that `005`
omits. **Do not edit `001`–`006` after this ships** — their checksums are
already recorded.

This is a schema decision, so it has deliberately not been taken.

> **Note (2026-08-13):** superseded by DEC-1 — the squash-into-one-baseline
> path was chosen over this idempotent-`001`-plus-`007` path. See B1's status
> above.

---

## 2. Security

### Authorization

The old app enforced authorization in `lib/`, next to the data access, so every
path through an operation got the same check. The Go port moved it to coarse
router middleware. That is the root cause of most of this section — and it is
why bolting `authz.Assert*` onto individual handlers will keep producing gaps
(`CreateQuote` checks role but not job ownership).

**Recommended structural fix:** a `requireJobAccess(ctx, user, jobID)` helper
called by every `/jobs/{id}/*` handler. It closes S1–S3 at once and makes the
next omission visible by its absence.

| # | Finding |
|---|---|
| S1 | ~~~30 routes are `RequireAuth`-only with no object-level scoping. Any authenticated agent or customer contact can read: every customer and contact (name/email/phone), every agent (email/phone/address), every job's timeline, notes **including internal**, photos, parts, quotes with full JSONB snapshots, invoices, invoice PDFs, every unit's serial and repair history, dashboard KPIs, the technician queue, company settings and message templates~~ **FIXED — `rms-api@c4c35a1` + `rms-api@36f36f3`** (T-17: `c4c35a1` adds `requireJobAccess`/`handler.RequireJobAccess`; the actual wiring onto all fourteen `/jobs/{id}/*` routes, plus staff-gating the dashboard KPIs, is in `36f36f3` — whose message says only "CORS". See the contamination note below this table.) |
| S2 | ~~`POST /jobs` and `POST /jobs/intake` trust client-supplied `agent_id`, `main_customer_id`, `originator_type`, `originator_id`, `received_by_staff_id`. The old code derived all of them from the session and said so in a comment. `jobs.originator_id` has no FK, so forged values persist~~ **FIXED — `rms-api@426233c`** (T-19) |
| S3 | ~~`PATCH /jobs/{id}` has no authorization at all — any authenticated user can rewrite any job's fault description and reassign its technician~~ **FIXED — `rms-api@426233c`** (T-19; ownership is enforced via the S1 job-access guard, PATCH itself is further restricted to staff) |
| S4 | ~~`POST /invoices/sage-export` with an empty body selects **every** unexported invoice in the system, streams a cross-tenant CSV, and marks them all exported. Behind `RequireAuth` alone. `GET /jobs/{id}/sage-export` mutates on a GET with no ownership check~~ **FIXED — `rms-api@36f36f3`** (T-20; no dedicated commit exists — the `staffMw` gate on the sage-export routes is in the same router.go hunk mislabeled "CORS") |
| S5 | ~~`staffMw` on customer/agent/parts/repair-type mutations admits **technicians**, where the old policy was admin-or-front-desk (customers/agents) or admin-only (parts, repair types). A technician can change part unit costs, which flow into quote lines~~ **FIXED — `rms-api@eb3cdbc` + `rms-api@36f36f3`** (T-21, DEC-5. `eb3cdbc`'s own message discloses this: "the router.go edits this describes were swept into 36f36f3 by another agent's `git add` on the same shared working tree" — the `deskMw`/`adminMw` policy change is actually in `36f36f3`; `eb3cdbc` adds `authz.go` support plus the test matrix.) |
| S6 | ~~`GET /customers` and `GET /agents` apply no role predicate — one authenticated GET enumerates the whole customer base~~ **FIXED — `rms-api@528d602`** (T-18; `e061e6f` is a same-day follow-up fixing a `$1`-numbering regression `528d602` introduced in `ListAgents`, which had been 500ing under every filter) |

> **Contamination note (2026-08-13):** `36f36f3`'s subject line and prose are
> entirely about CORS (T-26/S13), but its `internal/router/router.go` hunk
> also carries: the `RequireJobAccess` wiring onto all fourteen
> `/jobs/{id}/*` routes (T-17/S1), the `staffMw` gate added to the three
> Sage-export routes (T-20/S4), and the full `deskMw`/`adminMw` role-policy
> rewrite on customer/agent/parts/repair-type routes (T-21/S5, DEC-5).
> Confirmed by diffing `router.go` inside that commit, and independently by
> `eb3cdbc`'s own commit message disclaiming the router edits it describes.
> Cite `36f36f3` for S1, S4 and S5's code, not only S13's.

### Auth implementation

| # | Finding |
|---|---|
| S7 | ~~`ResetPassword` does not revoke sessions. `ChangePassword` and `AdminSetPassword` both do. A stolen refresh token survives the victim's password reset — the one thing a reset must prevent~~ **FIXED — `rms-api@d49383d`** (T-22) |
| S8 | ~~Refresh tokens INSERT a row per login but only the newest is ever read, and logout deletes **all** rows. Second device silently kills the first; logout anywhere logs you out everywhere. No reuse detection, which is the entire point of rotation~~ **FIXED — `rms-api@d49383d`** (T-22: session families with `family_id`/`rotated_at`/`revoked_at`, reuse revokes the whole family) |
| S9 | ~~`WEBHOOK_SECRET` defaults to `""` and doubles as the signed-URL HMAC key. `ServeFile` validates signatures without checking the secret is non-empty → unauthenticated read of any file under `STORAGE_ROOT`. The old app used a separate `STORAGE_URL_SECRET`; collapsing them also means a leaked n8n secret grants file access~~ **FIXED — `rms-api@1324f74`** (T-23). Note: this commit also carries an unrelated `internal/dbtest/dbtest.go` hunk from another agent's concurrent work on the same shared tree — harmless, but explains the otherwise-unrelated file in the diff. |
| S10 | ~~Photo upload has no MIME allowlist and no size cap; `http.ServeFile` sets Content-Type by sniffing, not from the stored `mime_type`. Upload `x.html` → persistent XSS on the API origin. Old app allowlisted four image types at 5 MB~~ **FIXED — `rms-api@ca863b2`** (T-24) |
| S11 | ~~`AcceptInvitation` and `CreateStaff`/`UpdateStaff` accept any password length. Every other path enforces ≥ 8. Staff creation UI asks for 6~~ **FIXED — `rms-api@3b33970` + `rms-api@8877b20`** (T-25: `3b33970` moves the ≥8 check into `HashPassword`, the single point every password passes through; `8877b20` is a same-day follow-up turning the resulting 500 into a proper 400 for `CreateStaff`/`UpdateStaff`) |
| S12 | ~~`AcceptInvitation` returns a working token pair for an account with `email_verified = false`, while `Login` refuses the same account with 403. `RequireAuth` never checks the flag, so the gate is bypassable by anyone holding an invitation link~~ **FIXED — `rms-api@3b33970`** (T-25, DEC-8: an `evf` claim is now carried and enforced in `RequireAuth`). **Not covered:** T-28's separate ask for a rate-limited **resend** endpoint has not been implemented — no such route exists in `router.go` as of this pass, so losing the original verification email still locks an account out permanently. |
| S13 | ~~CORS is `Access-Control-Allow-Origin: *` on every route, with a comment saying "tighten for production". No `nosniff`, no HSTS, no CSP anywhere~~ **FIXED — `rms-api@36f36f3`** (T-26: configurable origin allowlist, `SecurityHeaders` middleware adds `nosniff`/CSP/`Referrer-Policy`, opt-in HSTS) |
| S14 | ~~Login is a bcrypt timing oracle — unknown email ≈ 1 ms, known email ≈ 250 ms. No dummy-hash on the miss path~~ **FIXED — `rms-api@ca863b2`** (T-29). **Contamination:** this is the commit whose subject says only "photo uploads" (T-24/S10) — the entire timing-oracle fix (`internal/auth/bcrypt.go` dummy-hash constant, the `Login` single-comparison rewrite in `service.go`, `login_timing_test.go`) is bundled inside it. |

### Data exposure

| # | Finding |
|---|---|
| S15 | ~~Internal notes and photos are sent to portal browsers and hidden by a client-side `.filter()`. The API supports a `visibility` param but defaults to returning everything, and the hooks never pass it~~ **FIXED — `rms-api@bdccbc0`** (T-27) |
| S16 | ~~`ListJobPhotos` mints a 24-hour HMAC-signed **public** URL for every photo it returns, internal ones included — and signed URLs bypass all scoping by design~~ **FIXED — `rms-api@bdccbc0`** (T-27) |
| S17 | ~~A new portal Parts tab exposes `unit_cost_at_use` to customers and agents. The old portal had no parts view; parts were staff-only~~ **FIXED — `rms-api@bdccbc0`** (T-27, DEC-9: keep the tab, strip the cost fields server-side; `rms-frontend@48dadc4`/T-59a strips it from the render as well) |

---

## 3. Broken workflows

| # | Finding |
|---|---|
| W1 | ~~All seven webhook call sites use `go webhook.EmitEvent(r.Context(), …)`. `net/http` cancels that context when the handler returns. **`webhook_events` has 0 rows.** Gap 1's entire fix has never fired. One-line remedy: `context.WithoutCancel`~~ **FIXED — `rms-api@36142fb`** (T-15: `context.WithoutCancel` plus a 30s ceiling; also wires the previously-dead `job.received` and `job.extra_unit_discovered` events) |
| W2 | ~~Signed photo URLs 404. Keys are `{job_id}/{uuid}_{name}` — two path segments — the route is `{key}`, and Go's `ServeMux` matches one. Generation also doesn't escape the key. Old app used `encodeURIComponent`~~ **FIXED — `rms-api@1144214`** (T-16: route changed to `{key...}`, generation escapes per-segment) |
| W3 | Invited agents never receive a link. `InviteAgent` sends no email; the token is returned only in the HTTP response body, and the webhook payload omits it. **OPEN — no commit found for T-48.** |
| W4 | Agents cannot book in a unit that isn't already in the system — the pre-registration form requires selecting an existing unit and no unit-creation endpoint exists. **OPEN — no commit found for T-49.** DEC-11 (agent-callable intake variant) is settled but not implemented. |
| W5 | ~~Portal quote approve/reject is unreachable. The hooks, dialog, route guard and API endpoints all exist; nothing links to them~~ **FIXED — `rms-frontend@9b1576b`** (T-45; also restores the agent price-gating rule from `rms-app@9b11c8e`) |
| W6 | `invited_at` is never written by any code path, so the Agents page shows "Not invited" forever, Edit never renders, and Invite always does — minting duplicate invitations that 500 on acceptance against `uq_agents_email_live`. **OPEN — no commit found for T-46.** |
| W7 | ~~The Invoices page can never load — the route declares `:id`, the page reads `useParams<{jobId}>`. The same bug was found and fixed for quotes fourteen lines above~~ **FIXED — `rms-frontend@704d219`** (T-40) |
| W8 | `POST /parts` always 400s: the form sends `active`, the Go struct lacks it, and `DecodeJSON` sets `DisallowUnknownFields`. Editing works, creating doesn't. **OPEN — no commit found for T-41 in either repo.** |
| W9 | ~~The audit log will be permanently empty. One `RecordAudit` call site, passing `EntityID: nil` into a `NOT NULL` column — and `005` uses `CREATE TABLE IF NOT EXISTS`, so it is a no-op wherever `001` already ran. Failures are swallowed as warnings~~ **FIXED IN CODE — `rms-api@0996820`** (T-35). **⚠ Inert on the live server:** `001_baseline.sql` (`b35e678`) has never been *applied* to `gentick_rms` — live still carries `entity_id NOT NULL` because the baseline migration has never run there. The fix does nothing until that deploy happens. |
| W10 | ~~Quotes and invoices have no entry point. `JobDetailPage` has zero references to either; the sidebar has neither~~ **FIXED — `rms-frontend@bd4f3f4`** (T-44) |
| W11 | ~~Job detail Status History renders one hardcoded row for the current stage. The timeline is fetched and used only for dot colouring — every historical transition, its notes and its actor are invisible~~ **FIXED — `rms-frontend@3a0f8a0`** (T-43) |
| W12 | ~~Reports page likely white-screens: two `SelectItem value=""` against Radix ≥2, which throws, and there is no ErrorBoundary anywhere in the SPA~~ **FIXED — `rms-frontend@1905f1b`** (T-42) |

---

## 4. Data integrity

| # | Finding |
|---|---|
| D1 | ~~Money is `float64` end-to-end with no rounding at any step, against `NUMERIC(12,2)` columns under `CHECK (total = subtotal + vat_amount)`. Postgres rounds each field independently, so fractional quantities can break the equality → 500. This gets more expensive with every invoice written.~~ **FIXED — `rms-api@aac7649` + `rms-frontend@7cde78e`** (T-30, DEC-2: `shopspring/decimal` throughout `internal/model` and the quote/invoice handlers, JSON-encoded as strings; the frontend now types every wire decimal field as `string` and renders via a float-free `formatMoney` helper, keeping create-quote money/rate inputs as strings end-to-end). The backend landed by a concurrent agent during this pass; the frontend half completes the explicitly breaking wire change. |
| D2 | VAT is hardcoded 15% regardless of currency. The old rule was ZAR 15%, everything else 0% (SA export zero-rating) — and the currency picker *gained* EUR and GBP. Every foreign quote is charged SA VAT. **PARTIAL — `rms-api@5c4415f`** (T-31: ZAR 15% / everything else 0%, restored server-side). Same caveat as D1: landed moments before this pass by a concurrent agent, and its wiring depends on D1's decimal plumbing, which is itself not yet consumed by the frontend. |
| D3 | Assessment-fee invoicing (FR-QUO-09) does not exist. The columns, the settings key and the revenue report's label branch all exist; no code path can create one. **OPEN — no commit found for T-33.** |
| D4 | Transitions no longer create invoices. Closing a job raises no proforma; not-repairable raises no assessment fee. Gap 17 was closed for quote approval only — a different trigger. **OPEN — no commit found for T-33.** |
| D5 | `exported_to_sage` is never written. Both export paths set only `sage_export_id`, so the enum value is unreachable and the "Invoiced" state can never be shown. **OPEN — no commit found for T-36.** Confirmed still current: `handler_quotes_invoices.go`'s Sage-export path still only inserts into `sage_exports` and never updates invoice `status`. |
| D6 | Invoice numbering lost its idempotency guard and changed format to include a date. Two approvals on the same job, same day → unique violation, invoice not created, failure only logged while the quote is already approved. Different day → silent duplicate. **OPEN — no commit found for T-34.** |
| D7 | Every transition writes **two** `job_status_history` rows — the DB trigger and an explicit INSERT — one attributed to `system`. Quote-driven stage changes write only the trigger's row, with a NULL actor. **OPEN — no commit found for T-38.** |
| D8 | `not_repairable` no longer requires a fault reason, on either layer. The old app enforced it in the client and threw `VALIDATION_ERROR` server-side. **OPEN — no commit found for T-39b.** `TC-WF-06` remains red. |
| D9 | ~~Message templates changed from immutable versioning to in-place `UPDATE`. Prior wording is destroyed; `retired_at` is never written. Messages already sent no longer keep the wording in effect at the time~~ **FIXED — `rms-api@5b414d5`** (T-39: retire-and-insert under a row lock, partial unique index enforces one live version per name/channel/language) |
| D10 | Neither quote approval nor rejection is transactional — four independent statements, each failing with a log line. The old app wrapped all of it. **OPEN — no commit found for T-38.** |
| D11 | XML Sage export marks every selected invoice exported, commits, *then* returns a JSON note saying XML isn't implemented — delivered as a `.xml` download. No way to un-flag from the UI. **OPEN — no commit found for T-37.** |
| D12 | `consent_records` is never referenced. The old app wrote POPIA consent rows on agent invitation acceptance. **Compliance question, needs a decision.** DEC-6 settled the decision (restore the write) but it has not been implemented — **OPEN, no commit found for T-59b.** `consent_records` still appears only in the migration schema (`001_baseline.sql`), referenced by no Go code. |

> **In progress, uncommitted (as of this pass):** T-32 (the PDF fix — "TAX
> INVOICE" wording, hardcoded company details, `%.0f` quantity truncation) has
> code sitting modified/untracked in the `rms-api` working tree
> (`internal/handler/pdf.go`, `handler_quotes_invoices.go`, plus new
> `pdf_test.go`/`pdf_route_test.go`) but **not committed**. Not struck through
> — no commit exists to cite. `T-32` corresponds to "D-new" in `scrum.md` and
> has no lettered row of its own in this table.

---

## 5. Infrastructure

The live ingress **works** and needs no change: `gentick-infra/nginx/nginx.conf`
routes `/api/*` → `rms-api:8080` and `/*` → the SPA with history fallback,
correct cache headers on both. The deployed bundle is current.

| # | Finding |
|---|---|
| I1 | ~~**`rms-app` is still running on the server** — the Next.js image with all three Supabase keys, sharing the same Postgres *and the same uploads volume* as `rms-api`. Two applications, two auth models, one set of tables~~ **FIXED — `rms-infra@ca71712`** (T-60: `rms-app` dropped from the deployment) |
| I2 | Three competing frontend-delivery mechanisms. Only the static sync into `gentick-infra` is live. `rms-infra/scripts/deploy-frontend.sh` targets a gitignored directory; `rms-frontend`'s image is in no compose file and its `nginx.conf` has no `/api` proxy (latent critical if ever deployed); `rms-infra/public/nginx/web-apps/rms/` is a committed duplicate build mounted by nothing. **OPEN.** DEC-7 settled the decision (keep the static sync, delete the other two) but the deletions haven't happened: `rms-infra/caddy/`, `rms-infra/public/nginx/web-apps/rms/`, and `rms-frontend/Dockerfile` + `nginx.conf` are all still present as of this pass. |
| I3 | `rms-infra/caddy/Caddyfile` reverse-proxies to `app:3000` — there is no caddy service and no caddy container. Dead config that documents the wrong ingress. **OPEN.** `rms-infra`'s own changelog (from `4a24e19`) confirms it directly: *"`caddy/Caddyfile` is now referenced by nothing. Left in place pending a decision — nginx replaced it."* The file itself has not been removed. |
| I4 | ~~`svc-start.sh` references a `caddy` service that doesn't exist and omits `api` and `nginx` entirely. `--public` and `--all` fail; `--private` starts the Next.js app, not the Go API. `boot.sh` defaults to `--all` under systemd → fails~~ **FIXED — `rms-infra@4a24e19`** ("adopt the shared infra standard" — replaced the hand-maintained service-name arrays with a compose-file-driven `svc-start.sh`; no `caddy` reference remains, `--scope private` now starts `api`). Refined by `169035a` (svc-scripts → 0.1.0). |
| I5 | ~~`rms-infra/compose.yml` defines its own `postgres` and `nginx`. On gentickit.local those names belong to gentick-infra. Bring rms-infra's up and `postgres` resolves on `rms-net` first — rms-api silently connects to an empty database, migrates into it, and reports healthy. Exactly the trap documented for talosot's `mqtt`, with no warning comment~~ **FIXED — `rms-infra@4a24e19`** (split into `public/compose.yml`, only for a host providing none of its own; the private `compose.yml` now defines only `api` and `n8n` — confirmed against the current tree) |
| I6 | ~~Uploads live in a Docker named volume on the root filesystem, not under `HOST_DATA_ROOT`. Invisible to any backup targeting `/mnt/main-drive`, and shared between `api` and `app` which use different key formats~~ **FIXED — `rms-infra@b2c9ee3`** (T-65: bind mount at `${HOST_DATA_ROOT}/rms/uploads`, confirmed in the current `compose.yml`) |
| I7 | `MIGRATIONS_DIR` unset ⇒ migrations are **silently skipped** and the API starts against whatever schema exists. Appears in no `.env.example`. Should be an opt-out, not an opt-in. **OPEN — no commit found for T-66.** Consistent with the current test baseline: 3 of the 4 remaining top-level failures are migration-runner defects in `internal/db` attributed to T-66. |
| I8 | `RunMigrations` skips a checksum-mismatched migration with a warning and continues. Two environments, one repo, different schemas, no alarm — and no supported way to re-apply. **OPEN — same T-66, no commit.** |
| I9 | All three submodules are on `main`, not `agentic`. **Resolved operationally, not by commit.** `rms-api`, `rms-frontend`, `rms-infra` (and `rms-app`) are all currently checked out on `agentic` locally — verified via `git branch -a` during this pass. This was Sprint 0's `T-00`, a manual branch-hygiene task with no fixing commit to cite, so it is left unstruck per the "only mark done against a commit" rule rather than silently dropped. |

---

## 6. Visual and design system

Audited separately after the functional passes. Both apps are Tailwind v4 with
CSS-first `@theme` — the old `tailwind.config.ts` was dead code (no `@config`
directive), so the live token source was `app/globals.css`. The old token block
matches `_docs/old/wireframes.html` value-for-value: the old app was a faithful
implementation of the signed-off wireframe. **The divergence below is not a
migration constraint. Both sides use the same mechanism; the tokens were
rewritten.**

| # | Finding |
|---|---|
| V1 | ~~**Job stage colour semantics destroyed.** 13 stages across 5 colour variants (amber `quoted`, blue `approved`/`in_progress`/`qc`, red `not_repairable`/`rejected`/`cancelled`, green, grey) collapsed to **one hardcoded green tint**, repeated byte-identically at 11 call sites. No stage→colour map exists anywhere in `rms-frontend` — `STAGE_LABELS` was ported, the variant map was not. A job list is now a column of identical green pills; "Cancelled" and "Ready for Collection" are visually the same chip~~ **FIXED — `rms-frontend@d025fca` + `rms-frontend@c8337bd`** (T-72) |
| V2 | ~~**Brand colour replaced.** `#72C044` → `#00563f`. The old config annotates it *"Gentick brand palette (locked in v1.0)"*; the wireframe and the wordmark SVG both hardcode it. `#00563f` appears in no doc, asset or wireframe. Bright lime → dark forest green~~ **FIXED — `rms-frontend@391f717`** (T-70, DEC-3: reverted to `#72C044`/`#413F42`) |
| V3 | ~~**The Gentick wordmark was not ported.** `components/ui/Logo.tsx` has no successor. Login now shows a lucide **wrench icon** in a green square; the sidebar shows the literal text "RMS"; the customer-facing portal has no branded header at all~~ **FIXED — `rms-frontend@ba5930e`** (T-71, at sidebar/login/portal) |
| V4 | ~~**Aged >14 days escalation removed wholesale** — the KPI tile, the stage-badge label override (`Aged · 18d`) and the amber queue-card treatment. `grep "aged\|escalate"` over the new `src/` returns nothing. The new "High" indicator is stage-position-based, not age-based — it answers a different question~~ **FIXED — `rms-frontend@21fd473`** (T-73) |
| V5 | ~~**The Vite logo ships as the favicon.** `public/` contains exactly one file: `vite.svg`. Another product's mark in the browser tab of a customer-facing portal~~ **FIXED — `rms-frontend@ba5930e`** (T-71, same commit as V3 — replaces the favicon with the Gentick mark) |
| V6 | ~~**Root font-size 14px → 16px.** The old app pinned `html { font-size: 14px }`, scaling every rem utility by 0.875×, and pinned most sizing in px besides. The new app is entirely rem-based on a 16px root. Net: the UI is 15–25% larger. Table rows 42px → 53px — roughly **19% fewer rows per screen** for staff doing high-volume intake~~ **FIXED — `rms-frontend@391f717`** (T-75, DEC-10) |
| V7 | ~~**`text-muted-foreground` used 50 times, generates zero CSS.** It is a shadcn convention token, undefined in `index.css` and not a Tailwind default. Verified against the shipped bundle: 50 source occurrences, 0 matching rules. Every "Subtotal:/VAT:/Total:" label on Quotes, Invoices and Parts renders at full body colour — the label/value hierarchy is flat~~ **FIXED — `rms-frontend@391f717`** (T-74: `--color-muted-foreground` defined) |
| V8 | ~~**Sidebar inverted dark → light.** A `#413f42` rail with a solid `#72C044` active pill became a white rail with a pale mint pill. Section headers and the footer user block removed. Sidebar, topbar and page ground are now all near-white — the structural frame separating navigation from content is gone~~ **FIXED — `rms-frontend@92bd42f`** (T-78) |
| V9 | ~~**Status palette handed back to Tailwind defaults.** The old `@theme` overrode `amber-*`, `red-*`, `blue-*` so those class names produced wireframe colours. The new one defines none of them. `blue-500` went `#2563a6` (navy) → `#2b7fff` (electric)~~ **FIXED — `rms-frontend@ba5930e`** (T-76) |
| V10 | ~~**Topbar is empty** — the left side is a literal `<div />`. Breadcrumb, page title and contextual actions all gone, and `<title>` is static with zero `document.title` writes. No persistent "where am I"~~ **FIXED — `rms-frontend@92bd42f`** (T-78) |
| V11 | ~~**Six components not ported**: `Kpi`/`KpiGrid` (the colour-accent bar was the point), `Timeline`, `Message`, `StageBadge`, `Field`/`FormGrid`, and the `UserChip` avatar tone system (green=agent / blue=customer / grey=staff → always green). `migration.md:98` claims all of `components/ui/*` was ported, and also lists `Dialog`, which never existed in the old app~~ **FIXED — `rms-frontend@cab64e4`** (T-79) |
| V12 | ~~**Print support removed** — `.no-print`, the print media block, `print:hidden` on the sidebar and `PrintButton`. Job sheets and invoices now print with full chrome~~ **FIXED — `rms-frontend@cab64e4`** (T-79; refined by `ca28176`, keeping the invoice-dialog title visible when printing) |
| V13 | **Untracked — no ticket exists for this finding.** `max-w-[1320px]` page cap removed — tables span the full viewport on wide monitors. **OPEN.** Sprint 6 (T-70–T-80) never assigned this a ticket number, so it was never picked up; confirmed still absent from `src/` as of this pass. |
| V14 | ~~Quote status `issued` went amber → blue, losing its "waiting on the customer" urgency. `expired` and `superseded` have no map entries and land on grey only via the `\|\| map.draft` fallback~~ **FIXED — `rms-frontend@c1b992b`** (T-80) |
| V15 | ~~Two grey scales (`grey-*` brand, `gray-*` Tailwind) and four greens (`#00563f`, `#003d2b`, `#008236`, focus ring `#289066`) now coexist~~ **FIXED — `rms-frontend@ba5930e` + `rms-frontend@391f717`** (T-76 collapses the grey scales and status greens; T-70 restores the brand pair) |
| V16 | ~~Inter and JetBrains Mono are declared and never loaded — no `<link>`, no `@font-face`, not npm deps. Silent fallback to `system-ui`. The old app used a deliberate system stack, so nothing was lost in rendering, but what ships is not what the tokens declare~~ **FIXED — `rms-frontend@ba5930e`** (T-80, font-loading half only, per that commit's own message — self-hosted via `@fontsource-variable`) |
| V17 | ~~`tailwindcss-animate` is not installed, so `animate-in`/`fade-in-0`/`zoom-in-95` in `dialog.tsx` and `select.tsx` emit no CSS. Dialogs and popovers appear with no transition~~ **FIXED — `rms-frontend@391f717`** (T-77) |
| V18 | ~~Smaller drift: brand-tinted shadows → neutral black; warm grey ramp → pure neutrals; card header hairline removed; card padding 18→24px; table headers lost their tint, uppercase and tracking; buttons dropped `font-semibold` → `font-medium`; `Badge` `dot` prop removed; sidebar 248→224px~~ **FIXED (mostly) — `rms-frontend@c1b992b`** (T-80: card header hairline, table header treatment, button weight restored). Not independently verified: brand-tinted shadows, the warm-grey-ramp claim, `Badge`'s `dot` prop, and the exact sidebar width — not re-audited in this pass. |

**Contrast is the one dimension that broadly improved** — every changed status
badge came out equal or better, and the old amber button (2.56:1, a clear AA
failure) is gone with the variant. Two live failures remain: `text-amber-600`
on white for the dashboard "High" chip (3.20:1) and the `grey-400` placeholder
(3.23:1, pre-existing). The irony is that the old amber and red badges were
themselves marginal (4.33, 4.15) — the migration fixed them by deleting the
distinction that made them useful.

> **DEC-13 (2026-08-13):** the brand palette's AA shortfall (`#72C044` 4.33:1,
> `#413F42` 4.15:1 against the 4.5:1 normal-text threshold) is an **accepted,
> documented accessibility exception**, not a defect to remediate. `TC-NFR-09`
> is being rewritten to assert the exception rather than fail on it. This is
> a decision, not a code fix — no commit closes it.

**Genuine improvements, keep them:** lucide icons replacing Unicode/emoji nav
glyphs, skeleton loading states, icon+message empty states, sonner toasts (no
toast system existed before), Radix primitives for Select/Dialog/Dropdown,
`overflow-auto` on tables, fixed button heights, five-state invoice status
replacing a binary, `Eye`/`EyeOff` on visibility badges.

**Verified faithfully ported:** radius scale, dark mode (absent on both sides,
cleanly — no orphaned tokens or toggle), table row hover, job-number mono +
brand-green-deep treatment, card structure and shadow level, input border
colour, all 13 `STAGE_LABELS` (capitalisation aside). No `next/font` existed in
the old app, so nothing was lost there.

## 7. Systematic regressions

- **Email format validation is gone at every layer** — client, Go handler and
  DB. Email is the login identity and the reset target; malformed addresses
  create unrecoverable accounts. **OPEN — no commit found for T-55.**
- **E.164 phone validation is gone on both sides, but the DB CHECK
  constraints survive.** The new placeholders (`+27 11 123 4567`) demonstrate
  a format that violates them, so users following the hint get a 500.
  **OPEN — no commit found for T-56.**
- ~~**Every zod `max()` is gone.** Serials, unit types, accessories, notes,
  descriptions and shipment sizes are all unbounded client- and server-side~~
  **FIXED — `rms-frontend@488bcab`** (T-57)
- **No pgx error classification anywhere.** Duplicate emails, duplicate
  prefixes, FK violations and CHECK violations all surface as generic 500s.
  `ErrorDetail.Details []FieldError` exists and is never populated, so clients
  get one opaque message where zod returned per-field issues.
  **OPEN — no commit found for T-58.**
- ~~**Role access widened across the board** — dashboard, jobs list, reports and
  parts are now reachable by technicians and front desk, where the old app was
  deliberately restrictive~~ **FIXED — see S5** (`eb3cdbc` + `36f36f3`, T-21,
  DEC-5)
- **Intake lost**: condition-on-arrival photos, the "no extra unannounced
  units" gate, the shipment-origin field (every staff intake is now recorded
  `walk_in`), the serial repair-history banner, multi-unit booking, waybill
  number, and both post-intake webhooks. **Status by item:**
  - ~~condition-on-arrival photos~~ **FIXED — `rms-frontend@e659e35`** (T-51)
  - "no extra unannounced units" gate — ~~server OPEN, no commit found for
    T-52~~ **backend FIXED — `rms-api@790d447`** (T-52 server half:
    `POST /jobs/intake` now requires `confirmed_no_extra_units: true` and
    answers 400 `VALIDATION_ERROR` without it — TC-INT-26 backstop). Frontend
    confirmation UI still **OPEN**.
  - ~~shipment-origin field~~ **FIXED — `rms-frontend@71a3c47`** (T-53
    frontend) **+ `rms-api@8c403ec`** (B6/T-14 backend)
  - ~~serial repair-history banner, multi-unit booking, waybill number~~
    **FIXED — `rms-frontend@cef3083`** (T-54)
  - post-intake webhooks — **FIXED — see W1** (`36142fb`, T-15, `job.received`
    / `job.extra_unit_discovered` now wired)
- `internal/service/` is **dead code** — nothing imports it. It duplicates
  handler logic including the same broken `nextval('job_number_seq')`.
  **OPEN — no commit found for T-68; the directory still exists
  (`jobs.go`, `people.go`, `quotes.go`, `service.go`) as of this pass.**

---

## 8. Decisions — all settled 2026-08-12

**Answered by Teo; recorded in [`scrum.md`](./scrum.md) as DEC-1 … DEC-13.**
Summary: squash `001`–`006` into one idempotent baseline · `shopspring/decimal`
for money · revert the brand to `#72C044` · assessment-fee invoicing in scope
for cutover · restore the old role policy · restore the POPIA consent write ·
keep the static frontend sync into `gentick-infra` · TOTP stays out and email
verification is account-verification only (add the missing resend, close the
bypass) · keep the portal Parts tab without the cost column · restore the 14px
root · agent-callable intake variant for unit pre-registration · per-customer-
prefix job numbering · brand palette AA shortfall accepted as a documented
exception.

The original open questions, for the record:

1. **B1 remedy** — idempotent `001` + `007`, or squash `001`–`006` into a new
   baseline? Affects every existing environment.
2. **D1 money representation** — decimal, or integer cents? Decide before more
   invoices are written.
3. **D3 assessment fee** — in scope for cutover, or fast-follow?
4. **D12 POPIA consent** — was dropping `consent_records` intentional?
5. **S5 role widening** — deliberate product change, or `staffMw` copy-pasted
   from the catalogue routes?
6. **I2 delivery model** — confirm the static sync into `gentick-infra` is the
   intended shape, and delete the other two.
7. **MFA** — `migration.md` records it as TBD. NFR-04 states it as a
   requirement and it is absent from both repos.
8. **S17 part costs** — should customers and agents see them?
9. **V2 brand colour** — was `#72C044` → `#00563f` a rebrand you signed off,
   or drift? Nothing in the repo says so, and it contradicts three sources
   that all name `#72C044` as locked. Everything downstream (V3 wordmark, the
   green-tint stage badges, the light sidebar) follows from this answer.
10. **V1 stage colours** — restore the five-variant map, or accept a
    single-colour stage badge? This is the one visual finding with an
    operational cost rather than an aesthetic one.
11. **V6 density** — was the 14px root deliberate in the old app (it traced a
    px-specified wireframe) and is the larger new UI intended?

---

## 9. What the test suite now pins

`rms-api` has a database-backed suite modelled on `saamsaam-api`: 8 files,
72 test functions, plus `internal/testenv` and `internal/dbtest`. Written and
compile-verified; **not yet run against a database.**

The design decision that matters: an unset `TEST_DATABASE_URL` calls
`t.Fatal`, not `t.Skip`. A skip prints "ok", and "ok" for a test that never
executed is worse than no test at all. Only an explicit `RMS_SKIP_DB_TESTS=1`
skips, and it says loudly that nothing was verified. The DSN is also refused
if it is not loopback, because these suites open with `DROP SCHEMA`.

The route table is parsed out of `router.go` with `go/ast` rather than listed
in the test — `net/http.ServeMux` has no introspection API, and a hand-kept
list drifts. It resolves 84 of 84 routes and fatals on anything it cannot
parse. One hand-maintained list survives: the nine intentionally-public
routes, which *should* need a decision to change.

Two defects are already caught with no database attached:
`TestMigrationNumberingHasNoGaps` (B1) and
`TestAnEmptySecretCannotSignOrValidate` (S9). Verified failing.

Suites pin the correct behaviour, not the current behaviour, so most of
section 2 is expected to fail on first run. That is the point.

> **2026-08-13 update:** the suite has since been run. Both defects named
> above are fixed (see B1, S9). The baseline of 16 top-level failures is now
> down to **4** — see the "Remediation status" block at the top of this
> document.

---

## 10. Not verified

- **No test has been run against a database.** Every runtime claim is derived
  from source plus live read-only inspection of `gentickit.local`.
- The `jobs = 0`, `webhook_events = 0` and empty-sequence-catalogue facts *are*
  from the live database. The causal chain to them is inference.
- n8n workflows were not inspected, so the webhook payload-shape break
  (flattened scalars vs whole objects) is asserted from the Go structs only.
- Which commit built the running `rms-api:0.0.1` image is unknown, so whether
  the deployed API contains the Phase-4 work is unverified.
- No exploit was attempted. Every authorization finding is from reading the
  router and the complete set of 13 in-handler checks.
- Nothing was modified in `rms-app`, `rms-frontend`, `rms-infra` or on the
  server. The only changes are the new test files in `rms-api`, uncommitted.

> **2026-08-13 update:** most of the above is now stale — the DB-backed suite
> has run (see section 9), and `rms-api`, `rms-frontend` and `rms-infra` have
> all had substantial remediation commits since. `migration.md`'s Phase-4
> checkmarks remain untouched deliberately (see `scrum.md`'s tracking note);
> this document has not been re-verified against a fresh live-database read
> since 2026-08-13 morning.
