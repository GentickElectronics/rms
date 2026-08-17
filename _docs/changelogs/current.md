# Changelog — rms (umbrella)

## [Unreleased]

> **Versioning reset.** `_docs/VERSION` is `0.0.0-staging` and every git tag
> has been deleted, locally and on origin. Nothing here has been released, so
> the former `v0.0.0.md` archive is folded back in below. The first release
> will be v0.0.0, cut from this file.

### Changed
- **n8n decoupling scrum (N-1..N-6) complete.** Outbound email, the
  daily-summary cron and webhook retry/dead-letter moved out of n8n into the
  rms-api `internal/worker` package; n8n is parked behind a `whatsapp` compose
  profile (WhatsApp-only, not yet built). Closes finding I1 — a silently-dead
  n8n workflow can no longer take the whole notification path down with it.
  Per-repo detail in the rms-api and rms-infra changelogs; outcome and the
  default choices (SMTP transport, quote-decision emails, 7-day aging boundary,
  plain-text mail) in `_docs/migration/plan.md` Part 2.
- **`_docs/` reorganized.** Consolidated into `_docs/migration/`
  (`plan`, `audit`, `result`), `_docs/testing/` (procedure, manual checklist,
  dated results) and a top-level `_docs/scrum.md` pointer. Superseded material
  under `_docs/old/` pruned (api-spec, deployment-*, gap-analysis, the old
  migration diffs/plan, the sprint-1 test baseline); the QA test register
  (`_docs/old/test-register.{md,json}`, `test-cases.xlsx`) is now tracked.
- **rms-api schema re-baselined before release** — the `000_init_tracking` +
  `001_baseline` + `002`..`008` migration chain collapsed into a single
  `000_baseline.sql` (generated from the release schema). The staging database
  was rebuilt from it with all data preserved. Detail in the rms-api changelog.
- **Submodule pointers bumped to their `agentic` HEADs** — rms-api `2938499`,
  rms-infra `cad1b71`, rms-app `c282472`, rms-frontend `6ae7d33`.

### rms-api

#### Added
- **Password reset flow** (`auth/service.go`, `handler/handler_auth.go`):
  `RequestPasswordReset` generates a 32-byte hex token (1-hour expiry), stored
  in new `password_reset_tokens` table. `ResetPassword` validates the token and
  updates the appropriate user table (staff, contacts, agents). Handlers return
  user-enumeration-safe responses (always 200 for forgot-password).
- **Migration `002_messages_columns.sql`**: adds `from_phone`, `received_at`,
  `in_reply_to_external_id` to `messages` to bridge Lucia's Drizzle schema with
  the Go API. API INSERT switched from `intent` → `classified_intent` to
  eliminate a duplicate column.
- **Migration `004_password_reset.sql`**: creates `password_reset_tokens` table
  with `user_id`, `user_type`, `token_hash`, `expires_at`, `used_at`.
- **PDF invoice generation** (`handler/pdf.go`): uses `gofpdf` to render A4 tax
  invoices with company header, line-items table, subtotal/VAT/total, and footer.
  `InvoicePDF` handler (`GET /api/v1/invoices/{id}/pdf`) returns a proper PDF
  download with `Content-Disposition`.
- `go.mod`: added `github.com/jung-kurt/gofpdf v1.16.2`.
- `extractNameFromSnapshot` helper for parsing JSONB snapshots.

#### Fixed
- **Schema alignment with `rms-app` `001_initial_schema.sql`**:
  - `quote_approvals`: renamed `action`→`decision`, `approved_by_user_id`→
    `decided_by_user_id`, `approved_by_actor_type`→`decided_by_actor_type`,
    `created_at`→`decided_at`; added `source_message_id`.
  - `sage_exports`: removed `status`/`created_at`; renamed `exported_by_user_id`
    →`generated_by_user_id`, `exported_at`→`generated_at`; added `filename`,
    `file_path`, `downloaded_at`.
  - `insertQuoteApproval` now calls `mapActorType(user.EntityType)` to convert
    `main_customer_contact`→`main_customer` for the `actor_type_enum` constraint.
  - `job_status_history`: 4 INSERTs now include `transitioned_by_actor_type`
    (was `NOT NULL` in Lucia's schema but missing from API inserts).
  - `SageExport` handler generates `filename`/`file_path` automatically
    (format: `sage-export-YYYYMMDD-HHMMSS.csv`).
  - `ListSageExports` SELECT/Scan updated for new column set.

## Earlier work — was `v0.0.0.md`

Folded in when versioning was reset. Never released under a tag that still exists.

### Added
- `_docs/migration_diffs.md` — structural mapping from RepairManagementSystem → rms.
- `_docs/migrate_plan.md` — deployment plan for gentick-infra integration.
- `rms-app` and `rms-infra` registered as git submodules with `.gitmodules`.

### Changed
- Repo renamed from RepairManagementSystem to rms.
- Architecture shifted from standalone stack to gentick-infra integration (shared Postgres, nginx, Cloudflare Tunnel).
