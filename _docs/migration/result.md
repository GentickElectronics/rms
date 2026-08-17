# RMS Migration Remediation — Result

Outcome of the remediation of the Go rewrite (restoring parity with the retired
`rms-app`). The plan and the forward backlog are in [`plan.md`](./plan.md); the
findings ledger in [`audit.md`](./audit.md); the full test evidence in
[`../testing/results/2026-08-14.md`](../testing/results/2026-08-14.md).

## What was done

Sprints 0–6 all landed on `agentic` across `rms-api`, `rms-frontend`, `rms-infra`:

| Sprint | Theme | Result |
|---|---|---|
| 0 | Test harness | ~4,500 lines of DB-backed test infra committed; baseline captured |
| 1 | Make it work | Job creation unblocked (the deferred-FK + numbering blockers), messages insert, shipment origin, webhook dispatch, signed URLs |
| 2 | Make it safe | `requireJobAccess`, list scoping, client-ownership, financial gating, role policy, session/refresh rotation, signed-URL secret, upload validation, CORS, portal leakage, login timing oracle, verification-gate bypass |
| 3 | Make it correct | Money → `shopspring/decimal`, VAT by currency, proforma PDF, audit writer, template versioning, transactional quote approval, invoice idempotency, assessment-fee flow |
| 4 | Make it complete | Frontend batches, onboarding (invites/resend/`invited_at`), agent intake, login provisioning, POPIA consent, the T-55/56/57/58 validation batch |
| 5 | Infrastructure | Migration runner hardened (the deploy gate), `rms-app` removed, frontend delivery re-scoped, shadowing guard, uploads, `supabase_user_id` dropped, dead code deleted |
| 6 | Visual restoration | Brand palette `#72C044`/`#413F42`, wordmark, stage→colour map, aged escalation, fonts, density — restored to the signed-off wireframes |

## Deploy

Deployed to `gentickit.local` off `agentic`: `rms-api:0.0.0-staging` (multi-arch,
GHCR) + the remediated frontend bundle into `gentick-infra`. Migrations applied
clean on boot (idempotent baseline, no checksum clash). The DB was a clean
staging slate (seed data + staff, no business data). **Not merged to `main`.**

## Test run — 2026-08-14

145 documented cases executed against the live deploy:

**98 PASS · 28 FAIL · 16 BLOCKED · 3 N/A · zero regressions.**

Pre-remediation the plan predicted only 7 passes; the remediation flipped the
majority to green. The Go API suite (263 tests) covers the backend-behavioural
majority; the FAILs are overwhelmingly features the original audit flagged as
never-in-scope. Full breakdown in the test results file above.

## Fixed mid-run

- **F1** — `email.IsConfigured()` wrongly required SMTP auth, disabling all mail
  on the auth-less Mailpit relay. Fixed (`rms-api@e62aa56`), redeployed, verified.

## Open defects (carried into the backlog — see `plan.md` + `audit.md`)

| | Defect | Severity |
|---|---|---|
| F7 | WhatsApp auto-approve doesn't check `party_type` — an agent can approve a quote (the HTTP path is gated; this channel isn't) | HIGH (security) |
| F8 | Quote responses embed a base64 `agent_snapshot` decoding to `password_hash` + token hashes | HIGH (security) |
| F2 | One-primary-contact invariant unprotected (can reach zero primaries) | MEDIUM |
| F9 | Thread-less (first-contact) inbound WhatsApp 500s and drops the message | MEDIUM |
| F10 | Login + consent changes not audited (only 2 of 4 categories) | MEDIUM |
| I1 | n8n workflows not active server-side → notification delivery dead (addressed by the next scrum) | infra |

## State at close

- All code on `agentic`, unpushed to `main`; submodule pointers not bumped.
- `_docs` restructured; not yet committed to the parent repo.
- Test fixtures (`RMSTEST*`) left live on the staging DB.
