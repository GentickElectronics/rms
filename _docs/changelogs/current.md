# Changelog — rms (umbrella)

## [Unreleased]

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
