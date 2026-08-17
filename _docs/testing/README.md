# RMS Testing — Orientation

Two documents, two jobs. Read this first if you're about to test RMS.

## 1. Backend/API behaviour → run the Go suite

RMS has a Go API test suite: **263 test functions across ~50 files**, run
against a real Postgres instance (not mocks). It is the regression harness for
`rms-api` and covers the behavioural majority of what the 145-case manual test
register used to check by hand — auth/RBAC, intake validation and numbering,
the job state machine and its side-effects, money/VAT/PDF/Sage, WhatsApp/email
emission, reporting exports, and admin settings gating.

Run it with:

```bash
agollum/testing/golang.sh --app C:/dev/rms/rms-api --infra C:/dev/rms/rms-infra --defaults
```

Load the `agollum` skill first if you haven't already — it explains the
script's flags and defaults. This is the harness to run before and after any
backend change; a green run means the backend-behavioural majority of the
register still holds without anyone clicking through the UI.

**Do not re-verify Go-covered cases by hand.** If a Go test asserts it, running
it again manually is wasted effort and duplicate signal.

## 2. Everything else → `manual-checklist.md`

**50 cases** need a human (or a browser-driving agent) because they check
something a Postgres-backed Go test structurally cannot see: what the SPA
renders (sidebar visibility, inline validation, KPI tiles, print layout,
branding), or a real external system's behaviour (n8n delivery, SMTP/DNS
mail authentication, hosting region, backups, uptime).

See [`manual-checklist.md`](./manual-checklist.md) for the full list, grouped
by category, each with a condensed "how to run" so you don't need to open the
source register to execute it.

**3 cases are excluded entirely** (neither Go-covered nor on the manual list):

| Case | Why excluded |
|---|---|
| TC-AUTH-08 | MFA — out of scope per DEC-8 (TOTP deliberately not built for this stack) |
| TC-INT-14 | N/A by design — every job requires an identified customer/agent before save (TC-INT-11), so the `GT-GEN` fallback prefix path is unreachable through the app |
| TC-INT-23 | N/A by design — main customers cannot pre-register units, so there's no announced baseline against which an "extra" unit could be detected |

## Coverage summary

145 register cases total: **92 Go-covered (drop from manual runs) · 50 manual
(run by hand) · 3 excluded (N/A/out of scope)**.

| Category | Total | Go-covered | Manual | Excluded | Covering Go test files |
|---|---|---|---|---|---|
| AUTH/RBAC | 13 | 8 | 4 | 1 | `authflow_test.go`, `authz_matrix_test.go`, `jwt_test.go`, `bcrypt_test.go`, `password_policy_test.go`, `session_family_test.go`, `login_timing_test.go`, `provision_login_test.go`, `verification_resend_test.go`, `invitation_accept_test.go`, `invitation_email_test.go` |
| INTAKE | 35 | 19 | 14 | 2 | `job_numbering_test.go`, `agent_intake_test.go`, `intake_receive_gate_test.go`, `shipment_origin_test.go`, `input_validation_test.go`, `validation_test.go`, `pg_error_test.go`, `consent_record_test.go`, `list_agents_customer_test.go`, `list_agents_filters_test.go` |
| WORKFLOW | 14 | 12 | 2 | 0 | `transition_side_effects_test.go`, `audit_writer_test.go`, `portal_scoping_test.go`, `files_test.go`, `photo_upload_test.go` |
| QUOTING | 19 | 17 | 2 | 0 | `money_test.go`, `decimal_test.go`, `vat_test.go`, `pdf_test.go`, `pdf_route_test.go`, `sage_export_test.go`, `template_versioning_test.go` |
| COMMS | 15 | 13 | 2 | 0 | `whatsapp_inbound_test.go`, `webhook_dispatch_test.go`, `template_versioning_test.go` |
| REPORTING | 10 | 8 | 2 | 0 | `handler_report_exports_test.go`, `money_test.go` |
| ADMIN | 10 | 9 | 1 | 0 | `authz_matrix_test.go`, `photo_upload_test.go`, `audit_writer_test.go` |
| UI/WIREFRAME | 10 | 0 | 10 | 0 | none — every case in this category is browser/visual |
| NFR | 14 | 2 | 12 | 0 | `cors_test.go`, `ratelimit_test.go`, `sigurl_test.go`, `password_policy_test.go`, `session_family_test.go`, `login_timing_test.go`, `photo_upload_test.go` |
| INTEGRATIONS | 5 | 4 | 1 | 0 | `webhook_dispatch_test.go`, `sage_export_test.go`, `sigurl_test.go` |
| **Total** | **145** | **92** | **50** | **3** | |

Classification rule: a case drops to Go-covered when a Go test in the file
list above asserts the behaviour the case checks; a case stays manual when
what it actually verifies is SPA rendering, an infra/external system, or a
process/compliance fact no Go test can reach. Where a category is *mostly*
Go-covered but one specific case is about how the SPA renders that data (e.g.
TC-QUO-04/05 — the money math is Go-tested, but whether the agent portal
shows or hides the price is not), the case stays on the manual list. See
`manual-checklist.md`'s "Why not automatable" column for the reasoning on
each of the 50.

## Where the evidence lives

- [`results/2026-08-14.md`](./results/2026-08-14.md) — the last full 145-case run
  (98 PASS / 28 FAIL / 16 BLOCKED / 3 N/A): the results table (with the `Layer`
  column that drove the classification above), the findings/defect ledger, and
  the per-batch detail — one dated record. New runs get a new dated file in
  `results/`.
- [`../migration/audit.md`](../migration/audit.md) and
  [`../migration/result.md`](../migration/result.md) — the migration findings
  ledger and remediation outcome.
- `../old/test-register.json` / `../old/test-cases.xlsx` — the full historical
  145-case specs (Preconditions + Test Steps). The manual checklist condenses
  what you need; go to the source only when a case needs more context.

## A note on regressions

The Go suite is the fast, cheap, deterministic check. A case failing there is
a real backend regression. A case on the manual checklist failing on a given
run may be a real regression *or* a known, already-logged gap (several manual
cases test features that are simply not implemented yet — see the "Why not
automatable" notes for which). Check the latest `results/*.md` findings section
(and `../migration/audit.md`) before filing a new defect for something already
on record.
