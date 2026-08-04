# Gentick RMS — API Specification

**Version:** 1.0
**Date:** 31 May 2026
**Base URL:** `https://rms.gentick.co.za/api/v1`
**Companion:** `Gentick_RMS_OpenAPI_v1.0.yaml` (machine-readable spec)

This document is the canonical reference for the Gentick RMS HTTP API. It is intended to be implemented as Next.js API routes (`app/api/v1/...` route handlers) with PostgreSQL persistence and Supabase Auth.

---

## 1. Overview

The API is REST-ish JSON, versioned by URL prefix (`/api/v1`). It is consumed by:

- **The Gentick web app** itself (staff portal, agent portal, main customer portal — same backend).
- **n8n workflows** that bridge to WhatsApp (Meta Cloud API) and email.
- **Future integrations** (Sage Pastel Online, technician mobile app, etc.).

There are three classes of caller, distinguishable by their JWT:

| Caller | How it authenticates | Typical operations |
| --- | --- | --- |
| Staff user | Supabase Auth session (browser cookie or `Authorization: Bearer`) | All operations within their role |
| External user (main customer contact / agent) | Supabase Auth session | Their scoped resources only |
| n8n / system | A signed service-account JWT issued by the RMS for n8n's use | Webhook ingestion, sending messages, scheduled jobs |

## 2. Conventions

### 2.1 Endpoints and verbs

| Verb | Used for |
| --- | --- |
| `GET` | Read a resource or list |
| `POST` | Create a resource OR trigger a non-idempotent action |
| `PATCH` | Partial update of a resource |
| `PUT` | Full replace (rare — used for settings) |
| `DELETE` | Soft-delete a resource |

Action endpoints (e.g., `POST /jobs/{id}/transition`) are preferred over overloaded `PATCH` for anything that has side effects (firing webhooks, sending emails).

### 2.2 IDs, timestamps, money

- **IDs:** UUID v4 strings, e.g., `"3e72a8c1-d4b1-4d3f-9f9e-7f8b0c2a1d8e"`.
- **Timestamps:** ISO 8601 with timezone, always UTC, e.g., `"2026-05-31T11:30:00Z"`.
- **Money:** decimal number `"3553.50"` (serialised as string to avoid float drift) plus a separate `currency_code` field, e.g., `"ZAR"`.
- **Phone numbers:** E.164 format, e.g., `"+27821234567"`.
- **Job numbers:** human-readable strings, e.g., `"GT-ACME-0184"`.

### 2.3 Request / response content types

- Requests: `application/json` for everything except photo uploads (`multipart/form-data`).
- Responses: `application/json` for everything except PDF endpoints (`application/pdf`) and CSV exports (`text/csv`).
- All requests must include `Accept: application/json` (or `application/pdf` for PDF endpoints).

### 2.4 Standard response envelope

Single-resource responses return the resource object directly. List responses always use this envelope:

```json
{
  "data": [ /* array of resources */ ],
  "pagination": {
    "cursor": "opaque-string",
    "next_cursor": "opaque-string-or-null",
    "total": 42
  }
}
```

The `total` field is best-effort and may be omitted for very large collections where exact counts are expensive.

### 2.5 Pagination

Two styles:

- **Cursor pagination** (default for `/jobs`, `/messages`, `/audit-events`, `/webhook-events` — anything that grows indefinitely). Query params: `cursor` (opaque, from previous response), `limit` (max 100, default 25).
- **Offset pagination** for small collections (`/main-customers`, `/agents`, `/parts`, `/repair-types`). Query params: `page` (1-based), `limit` (max 100, default 25).

### 2.6 Filtering and sorting

List endpoints accept:

- Field filters as query params: `?current_stage=quoted&main_customer_id={uuid}`.
- Range filters with `_from` / `_to` suffixes: `?intake_at_from=2026-01-01&intake_at_to=2026-03-31`.
- Sort: `?sort=intake_at:desc,current_stage:asc`.

Each endpoint's "Filters" section documents which fields are supported.

### 2.7 Soft delete

Most resources support soft delete. `DELETE /resource/{id}` sets `deleted_at`. Soft-deleted items are excluded from list endpoints unless `?include_deleted=true` is set (admin only).

A subsequent `POST /resource/{id}/restore` clears the `deleted_at`.

POPIA right-to-erasure is a separate action: `POST /main-customer-contacts/{id}/erase` or `POST /agents/{id}/erase`. Erasure cascades and scrubs PII — see Section 6.4.

## 3. Authentication and Authorization

### 3.1 Authentication

Supabase Auth is the identity provider. Two delivery mechanisms:

- **Browser:** Supabase sets HTTP-only secure cookies (`sb-access-token`, `sb-refresh-token`). The Next.js middleware reads them server-side via `@supabase/ssr`. No bearer header needed.
- **API consumers (n8n, integrations):** `Authorization: Bearer <jwt>`. JWT signed by Supabase project; the API verifies signature and extracts `sub` (Supabase user id), `aud`, and `exp`.

A user's RMS role is resolved by looking up their Supabase `user_id` in `staff_users`, `main_customer_contacts`, or `agents` (a single user record exists in exactly one of these three).

### 3.2 Authorization model

Authorization is enforced at the handler level. Every endpoint section below lists "Auth" — the roles permitted. The matrix below is the canonical reference:

| Role | Read jobs | Create jobs | Approve quotes | Manage settings | Manage users |
| --- | --- | --- | --- | --- | --- |
| Gentick Admin | All | Yes (on behalf) | All | Yes | Yes |
| Gentick Technician | All | No | No | No | No |
| Gentick Front Desk | All | Yes | No | No | No |
| Main Customer Contact | Own organisation only | Yes (own corp's agents) | Yes (own corp's jobs) | No | Invite own agents |
| Agent | Own jobs only | Yes (for own units) | No | No | No |
| n8n / system | All (service) | No | No (only records inbound approvals) | No | No |

**Quote approval rule (FR-COM-03):** only the main customer contact can approve a quote in MVP, even when the repair was initiated by an agent. A Phase 2 hook (`approval_authority = 'agent_threshold'` on the quote) is in the schema but no endpoint exposes it yet.

### 3.3 Authorization errors

- `401 Unauthorized` — no valid token or expired token.
- `403 Forbidden` — token is valid but the user's role doesn't allow the operation, OR the user is trying to access a resource outside their scope (e.g., an agent reading another agent's job).

## 4. Errors

Every error response uses this shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid job stage transition",
    "details": [
      { "field": "to_stage", "issue": "cannot move from 'received' to 'closed' directly" }
    ],
    "request_id": "req_2026053111300012ab"
  }
}
```

| HTTP | `error.code` | Meaning |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | Malformed body or invalid field values |
| 400 | `INVALID_STAGE_TRANSITION` | Job stage change not allowed from current stage |
| 400 | `QUOTE_EXPIRED` | Cannot approve an expired quote |
| 401 | `UNAUTHENTICATED` | Missing or invalid auth |
| 403 | `FORBIDDEN` | Authenticated but not permitted |
| 404 | `NOT_FOUND` | Resource does not exist or is soft-deleted |
| 409 | `CONFLICT` | Duplicate, e.g., customer_prefix already in use |
| 410 | `GONE` | Resource was hard-deleted under SARS retention purge |
| 422 | `BUSINESS_RULE_VIOLATION` | Allowed by schema but violates business logic, e.g., assessment fee on a Rejected job |
| 429 | `RATE_LIMITED` | Too many requests — see `Retry-After` header |
| 500 | `INTERNAL_ERROR` | Unhandled server error — `request_id` is the log correlation key |

All responses include an `X-Request-Id` header with the same `request_id` value, for log correlation.

## 5. Rate limiting

Default rate limits (per authenticated user):

- Staff & external users: 600 requests per minute.
- n8n service account: 6,000 requests per minute (for webhook ingestion bursts).

Exceeded → `429 RATE_LIMITED`. The response includes `Retry-After: <seconds>` and `X-RateLimit-Remaining: 0`.

## 6. Endpoints

Endpoint sections follow this format:

> #### `METHOD /path`
> **Auth:** roles permitted.
> **Description:** what it does and any side effects.
> **Request:** body fields.
> **Response 200/201:** body shape.
> **Errors:** non-default error codes worth flagging.

### 6.1 Auth and current user

#### `GET /api/v1/me`
**Auth:** any authenticated user.
**Description:** returns the current user's RMS profile, role, and permissions. Frontend calls this on app load to drive nav + role-based UI.

**Response 200:**
```json
{
  "user_id": "auth-uuid",
  "role": "main_customer_contact",
  "rms_entity": {
    "type": "main_customer_contact",
    "id": "uuid",
    "main_customer_id": "uuid",
    "contact_person": "Johan Kruger",
    "email": "johan@acme.co.za",
    "is_primary": true
  },
  "permissions": {
    "can_approve_quotes": true,
    "can_invite_agents": true,
    "can_create_jobs": true,
    "scopes": ["main_customer:uuid"]
  }
}
```

#### `POST /api/v1/auth/password-reset-request`
**Auth:** none (public).
**Description:** triggers a Supabase password reset email. No-op if email is unknown.
**Request:** `{ "email": "string" }`
**Response 204** (always, to avoid email enumeration).

### 6.2 Main customers and contacts

#### `GET /api/v1/main-customers`
**Auth:** staff (admin, front_desk).
**Filters:** `q` (search by name or prefix), `include_deleted`.
**Response 200:** `{ data: MainCustomer[], pagination: {...} }`

#### `POST /api/v1/main-customers`
**Auth:** staff (admin, front_desk).
**Request:**
```json
{
  "corporation_name": "Acme Agritech",
  "customer_prefix": "ACME",
  "notes": "Primary partner — agricultural sensors"
}
```
**Response 201:** `MainCustomer`
**Errors:** `409 CONFLICT` if prefix is taken.

#### `GET /api/v1/main-customers/{id}`
**Auth:** staff, OR the main customer's own contacts.
**Response 200:** `MainCustomer` with `contacts[]` and `agents_summary` embedded.

#### `PATCH /api/v1/main-customers/{id}`
**Auth:** staff (admin).
**Request:** any subset of `corporation_name`, `notes`. Note: `customer_prefix` is NOT updatable after creation (job numbers depend on it).

#### `DELETE /api/v1/main-customers/{id}`
**Auth:** staff (admin).
**Description:** soft-deletes the customer and cascades soft-delete to contacts and agents. Active jobs prevent deletion (`409 CONFLICT`).

#### `GET /api/v1/main-customers/{id}/contacts`
**Auth:** staff, OR contacts of that main customer.
**Response 200:** `{ data: MainCustomerContact[] }`

#### `POST /api/v1/main-customers/{id}/contacts`
**Auth:** staff (admin), OR primary contact of that main customer.
**Request:**
```json
{
  "contact_person": "Naledi Smith",
  "contact_email": "naledi@acme.co.za",
  "contact_phone": "+27821234567",
  "is_primary": false
}
```
**Response 201:** `MainCustomerContact`. Side effect: sends invitation email if `contact_email` doesn't yet have a Supabase user.

### 6.3 Agents

#### `GET /api/v1/agents`
**Auth:** staff; OR main customer contact (returns only their own corp's agents); OR agent (returns just themselves).
**Filters:** `main_customer_id`, `q`, `active`.

#### `POST /api/v1/main-customers/{id}/agents/invite`
**Auth:** staff (admin, front_desk), OR the main customer's primary contact (per FR-INT-11).
**Description:** creates an `agent_invitations` row and emails the invite link.
**Request:**
```json
{
  "agent_name": "Sipho Mthembu",
  "agent_email": "sipho@karoofarms.co.za"
}
```
**Response 201:** `AgentInvitation`. The token is NOT returned in the response — it's emailed directly.

#### `POST /api/v1/agents/accept-invitation`
**Auth:** none (public — token-bearing).
**Request:**
```json
{
  "token": "from-email-link",
  "supabase_user_id": "uuid-from-fresh-supabase-signup",
  "agent_phone": "+27821234567",
  "address_line1": "Plot 12, Karoo Road",
  "city": "Graaff-Reinet",
  "province": "Eastern Cape",
  "postal_code": "6280",
  "consent_whatsapp": true
}
```
**Response 201:** `Agent` (with `supabase_user_id` linked, invitation marked accepted, consent row written).
**Errors:** `400 VALIDATION_ERROR` (expired/used token), `409 CONFLICT` (email already has an agent).

#### `GET /api/v1/agents/{id}`
**Auth:** staff; main customer contact (own corp's agents); the agent themselves.
**Response 200:** `Agent` with `units_summary` and `active_jobs_count`.

#### `PATCH /api/v1/agents/{id}`
**Auth:** staff (admin); the agent themselves (limited fields).
Self-edit allows phone, address, language preference. Email change goes via Supabase Auth.

#### `POST /api/v1/agents/{id}/erase`
**Auth:** staff (admin only).
**Description:** POPIA right-to-erasure. Soft-deletes the agent, scrubs PII columns to `[REDACTED]`, writes an `audit_events` row of type `agent.erased`. Does NOT delete linked jobs / quotes / invoices — those retain their snapshots per SARS retention.
**Request:** `{ "reason": "Customer request 2026-05-30" }`
**Response 200:** `{ "agent_id": "uuid", "erased_at": "...", "audit_event_id": "uuid" }`

### 6.4 Consent

#### `GET /api/v1/consent`
**Auth:** staff (admin); the subject themselves.
**Filters:** `subject_type`, `subject_id`, `channel`.
**Response 200:** `{ data: ConsentRecord[] }` — full history.

#### `POST /api/v1/consent`
**Auth:** staff (any), OR the subject themselves (via their profile page).
**Request:**
```json
{
  "subject_type": "agent",
  "subject_id": "uuid",
  "channel": "whatsapp",
  "action": "revoked",
  "wording_version": "v2",
  "recorded_via": "web_form"
}
```
**Response 201:** `ConsentRecord`. Side effect: if `revoked`, all in-flight WhatsApp queues to that subject are flushed.

#### `GET /api/v1/consent/current/{subject_type}/{subject_id}`
**Auth:** staff; the subject themselves.
**Description:** convenience endpoint returning the current state per channel.
**Response 200:**
```json
{
  "whatsapp": { "granted": true, "since": "2026-01-15T08:00:00Z", "wording_version": "v2" },
  "email":    { "granted": true, "since": "2026-01-15T08:00:00Z" }
}
```

### 6.5 Units

#### `GET /api/v1/units`
**Auth:** staff; main customer contact (own corp's units, via the agents that own them); agent (own units only).
**Filters:** `serial_number` (exact), `q`, `agent_id`, `unit_type`.

#### `GET /api/v1/units/{id}`
**Auth:** same as above.
**Response 200:** `Unit` plus `repair_history[]` (summary of every job that touched this unit).

#### `POST /api/v1/units`
**Auth:** staff (admin, front_desk).
**Description:** rarely called directly — units are usually auto-created during intake. Use this to register a unit ahead of receiving it.

#### `PATCH /api/v1/units/{id}`
**Auth:** staff (admin).
Allowed updates: `unit_type`, `current_agent_id` (for ownership transfers), `notes`.

### 6.6 Shipments and intake

#### `POST /api/v1/shipments`
**Auth:** staff (admin, front_desk), OR main customer contact (origin = `main_customer`), OR agent (origin = `agent`).
**Description:** the canonical intake action. Creates a shipment AND its jobs in a single transactional call. Units are upserted by serial number (existing units link automatically).
**Request:**
```json
{
  "origin": "agent",
  "originator_agent_id": "uuid",
  "tracking_number": "AB12384",
  "received_at": "2026-05-31T11:05:00Z",
  "notes": "Box intact",
  "units": [
    {
      "serial_number": "SN-21A847",
      "unit_type": "4Gee Telemetry Unit",
      "fault_description": "Unit stopped sending readings...",
      "accessories": "Power adapter, antenna, SIM holder cover",
      "repair_type_id": "uuid",
      "intake_photo_ids": ["photo-uuid-1", "photo-uuid-2"]
    },
    {
      "serial_number": "SN-22F019",
      "unit_type": "Soil Probe v2",
      "fault_description": "Probe readings drifted...",
      "accessories": "Cable only"
    }
  ]
}
```
**Response 201:**
```json
{
  "shipment_id": "uuid",
  "jobs": [
    { "id": "uuid", "job_number": "GT-ACME-0185", "current_stage": "received" },
    { "id": "uuid", "job_number": "GT-ACME-0186", "current_stage": "received" }
  ],
  "units_created": 1,
  "units_linked": 1
}
```
Side effects: fires one `job.received` webhook per new job; emails the jobcard PDFs to the originator and the agent.

**Errors:** `422 BUSINESS_RULE_VIOLATION` if walk-in origin lacks an originator selection at intake time.

#### `POST /api/v1/shipments/{id}/discover-extra-units`
**Auth:** staff (front_desk, admin).
**Description:** front-desk-only endpoint for FR-INT-12. Adds extra units found in the shipment beyond what was registered. Routes notifications per the origin rules in FR-INT-12.
**Request:** `{ "units": [ ... ] }` (same shape as `/shipments` units).
**Response 201:** `{ "jobs": [...] }`. Side effects: fires `job.received` AND `job.extra_unit_discovered` webhooks.

#### `GET /api/v1/shipments`
**Auth:** staff; main customer contact (origin = own corp); agent (origin = self).
**Filters:** `origin`, `received_from`, `received_to`, `originator_main_customer_id`, `originator_agent_id`.

#### `GET /api/v1/shipments/{id}`
**Auth:** same as list.
**Response 200:** `Shipment` with `jobs[]` summary.

### 6.7 Jobs — read

#### `GET /api/v1/jobs`
**Auth:** staff (sees all); main customer contact (sees own corp's jobs); agent (sees own jobs).
**Filters:**
- `current_stage` (single value or comma-separated list)
- `main_customer_id`, `agent_id`, `assigned_technician_id`, `unit_id`, `repair_type_id`
- `intake_from`, `intake_to`, `closed_from`, `closed_to`
- `q` (free-text search across job number, fault description, customer/agent name)
- `include_deleted` (admin only)
- `include_closed` (defaults to false — pass `true` to include closed/cancelled/rejected/not_repairable)

**Sort:** `intake_at:desc` (default), `current_stage_changed_at:desc`, `job_number:asc`.

**Cursor-paginated.**

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "job_number": "GT-ACME-0184",
      "main_customer": { "id": "uuid", "corporation_name": "Acme Agritech", "customer_prefix": "ACME" },
      "agent": { "id": "uuid", "agent_name": "Sipho Mthembu" },
      "unit": { "id": "uuid", "serial_number": "SN-21A847", "unit_type": "4Gee Telemetry Unit", "total_repair_count": 3 },
      "current_stage": "quoted",
      "current_stage_changed_at": "2026-05-31T09:14:00Z",
      "assigned_technician": { "id": "uuid", "full_name": "Naledi Dube" },
      "intake_at": "2026-05-29T11:05:00Z",
      "age_days": 3
    }
  ],
  "pagination": { "cursor": "...", "next_cursor": "...", "total": 47 }
}
```

#### `GET /api/v1/jobs/{id}`
**Auth:** staff; main customer contact (own corp); agent (own jobs).
**Response 200:** full `Job` including embedded `unit`, `agent`, `main_customer`, `shipment`, `repair_type`, `assigned_technician`, `fault_description`, `accessories`, `not_repairable_reason`, `assessment_fee_*`, `intake_at`, `closed_at`.

### 6.8 Jobs — workflow actions

#### `POST /api/v1/jobs/{id}/transition`
**Auth:** depends on the transition.
- All stages except `Approved`: staff (any role appropriate to the stage; technicians can drive most repair-stage transitions).
- `quoted -> approved`: only n8n service (driven by WhatsApp YES) OR staff (admin, when manual override) OR main customer contact (via portal `POST /quotes/{id}/approve`).
- Terminal states (`closed`, `not_repairable`, `rejected`, `cancelled`): staff (admin).

**Description:** transitions the job. Writes a `job_status_history` row, fires the appropriate webhook(s) — see Section 7.

**Request:**
```json
{
  "to_stage": "in_progress",
  "notes": "Parts arrived, resuming",
  "fields": {
    "assigned_technician_id": "uuid"
  }
}
```
For the `not_repairable` transition specifically:
```json
{
  "to_stage": "not_repairable",
  "fields": {
    "not_repairable_reason_id": "uuid",
    "not_repairable_reason_other": "Optional free text when reason = 'other'",
    "assessment_fee_waived": false,
    "assessment_fee_waiver_reason": null
  }
}
```
Side effect: automatically generates the assessment-fee proforma invoice per FR-QUO-09 unless `assessment_fee_waived = true`.

**Response 200:**
```json
{
  "id": "uuid",
  "current_stage": "in_progress",
  "current_stage_changed_at": "2026-05-31T11:30:00Z",
  "side_effects": ["webhook_fired:job.in_progress", "audit_event_recorded"]
}
```
**Errors:** `400 INVALID_STAGE_TRANSITION` if the requested transition is not allowed from the current stage; `403 FORBIDDEN` if the caller's role can't drive this transition.

#### `PATCH /api/v1/jobs/{id}`
**Auth:** staff. Limited fields: `assigned_technician_id`, `repair_type_id`, `accessories`, `fault_description` (with audit).
Stage cannot be changed via PATCH — use `/transition`.

#### `DELETE /api/v1/jobs/{id}`
**Auth:** staff (admin). Only allowed if `current_stage` is in `('received', 'cancelled')`. Use `/transition` to `cancelled` first if you need to cancel an active job.

### 6.9 Jobs — notes, photos, parts

#### `GET /api/v1/jobs/{id}/notes`
**Auth:** staff (sees all); main customer contact + agent (sees `visibility = customer` only).
**Response 200:** `{ data: JobNote[] }`

#### `POST /api/v1/jobs/{id}/notes`
**Auth:** staff (any).
**Request:** `{ "body": "string", "visibility": "internal" | "customer" }`
**Response 201:** `JobNote`

#### `GET /api/v1/jobs/{id}/photos`
**Auth:** staff (all); main customer contact + agent (customer-visibility only).
**Response 200:** `{ data: JobPhoto[] }` (file URLs are time-limited signed URLs)

#### `POST /api/v1/jobs/{id}/photos`
**Auth:** staff; main customer contact (own corp); agent (own jobs).
**Content-Type:** `multipart/form-data`
**Form fields:** `file` (binary), `caption` (string, optional), `visibility` (`internal`|`customer`, default `internal` for staff and `customer` for external uploaders).
**Response 201:**
```json
{
  "id": "uuid",
  "file_path": "/uploads/jobs/{job_id}/abc123.jpg",
  "signed_url": "https://...",
  "mime_type": "image/jpeg",
  "file_size_bytes": 1840234,
  "visibility": "customer"
}
```
**Errors:** `400 VALIDATION_ERROR` for unsupported mime types (allowlist: jpg, jpeg, png, webp, heic); `413 PAYLOAD_TOO_LARGE` over 5 MB (configurable in settings).

#### `DELETE /api/v1/jobs/{id}/photos/{photo_id}`
**Auth:** staff (admin, technician — the technician can only delete their own uploads).

#### `GET /api/v1/jobs/{id}/parts`
**Auth:** staff.

#### `POST /api/v1/jobs/{id}/parts`
**Auth:** staff (technician, admin).
**Description:** log a part used against the job. Snapshots `unit_cost` and `currency_code` from the parts catalog at the moment of logging.
**Request:**
```json
{ "part_id": "uuid", "quantity": "1" }
```
**Response 201:** `JobPart` (with snapshot fields populated).

#### `DELETE /api/v1/jobs/{id}/parts/{job_part_id}`
**Auth:** staff (admin, technician — own entries only, within 24h of logging).

#### `GET /api/v1/jobs/{id}/status-history`
**Auth:** staff; main customer (own corp); agent (own jobs — public-facing audit only, no internal notes).
**Response 200:** `{ data: JobStatusHistory[] }` newest-first.

### 6.10 Quotes

#### `POST /api/v1/jobs/{id}/quotes`
**Auth:** staff (admin, technician).
**Description:** creates a new quote in `draft` status (no snapshot fields yet; not visible to customers).
**Request:**
```json
{
  "currency_code": "ZAR",
  "vat_rate": "15.00",
  "validity_days": 14,
  "notes": "Includes recalibration",
  "line_items": [
    { "description": "Labour · diagnosis + repair", "is_labour": true, "quantity": "2.5", "unit_price": "650.00" },
    { "description": "PSU board assembly", "part_id": "uuid", "quantity": "1", "unit_price": "1140.00" }
  ]
}
```
Server computes `subtotal`, `vat_amount`, `total`, and `line_total` per row.
**Response 201:** `Quote` (status = `draft`).

#### `GET /api/v1/quotes/{id}`
**Auth:** staff; main customer contact (own corp); agent (own jobs — only after status = `approved`, per FR-QUO-02 visibility rule).
**Response 200:** full `Quote` including `line_items[]` and snapshot fields.

#### `PATCH /api/v1/quotes/{id}`
**Auth:** staff (admin, technician). Allowed only while `status = 'draft'`.

#### `POST /api/v1/quotes/{id}/issue`
**Auth:** staff (admin, technician).
**Description:** moves draft → issued. Captures snapshots of customer, agent, unit, and Gentick company details. Computes `expires_at = issued_at + validity_days`. Fires `quote.ready` webhook (drives the email + WhatsApp to main customer).
**Request:** `{}` (no body — all data already on the quote)
**Response 200:** `Quote` (status = `issued`, snapshots populated).
**Errors:** `422` if quote is empty (no line items) or status is not `draft`.

#### `POST /api/v1/quotes/{id}/approve`
**Auth:** main customer contact (own corp), OR n8n service (when recording WhatsApp YES), OR staff (admin, manual override).
**Request:**
```json
{
  "channel": "portal",
  "notes": null,
  "source_message_id": null
}
```
For n8n recording WhatsApp YES: `{ "channel": "whatsapp", "source_message_id": "uuid" }`.
**Response 200:**
```json
{
  "quote_id": "uuid",
  "status": "approved",
  "approval_id": "uuid",
  "job_transitioned_to": "approved"
}
```
Side effects: writes `quote_approvals` row, transitions job to `approved` stage, fires `quote.approved` webhook (drives confirmation messages).
**Errors:** `400 QUOTE_EXPIRED` if past `expires_at`; `409 CONFLICT` if quote already decided.

#### `POST /api/v1/quotes/{id}/reject`
**Auth:** main customer contact (own corp), OR n8n service.
**Request:** `{ "channel": "portal", "notes": "Customer declined — too expensive" }`
**Response 200:** job is NOT auto-transitioned; staff must manually move it to `rejected` (per state machine). No assessment fee is charged (Section 5.4 of requirements, v1.0). Fires `quote.rejected` webhook.

#### `POST /api/v1/quotes/{id}/supersede`
**Auth:** staff (admin, technician).
**Description:** marks the current quote `superseded` and creates a new draft quote (next version) on the same job.
**Response 201:** new `Quote` (status = `draft`, version incremented).

#### `GET /api/v1/quotes/{id}/pdf`
**Auth:** staff; main customer contact (own corp); agent (own jobs — quote must be `approved`).
**Response 200:** `application/pdf` binary. Always renders from the snapshot — same byte output every time once issued.

### 6.11 Invoices and Sage exports

#### `POST /api/v1/jobs/{id}/invoices`
**Auth:** staff (admin). Usually system-triggered (on Closed or Not Repairable transitions) — manual creation is for edge cases.
**Request:**
```json
{
  "type": "repair",
  "currency_code": "ZAR",
  "vat_rate": "15.00",
  "line_items": [ ... ]
}
```
For type `assessment_fee` the system fills line items from the assessment-fee setting; you can override `amount` per-job in the call.

**Response 201:** `Invoice` (status = `issued`, snapshots taken).

#### `GET /api/v1/invoices`
**Auth:** staff. Filters: `status`, `type`, `issued_from`, `issued_to`, `currency_code`, `main_customer_id`.

#### `GET /api/v1/invoices/{id}`
**Auth:** staff; main customer contact (own corp's invoices).
**Response 200:** full `Invoice` with `line_items[]` and snapshot fields.

#### `GET /api/v1/invoices/{id}/pdf`
**Auth:** staff; main customer contact (own corp).

#### `POST /api/v1/sage-exports`
**Auth:** staff (admin).
**Description:** generates a CSV or XML batch export of one or more `issued` invoices in Sage Pastel Online's import format. Marks each invoice `status = exported_to_sage` with the export's `id`.
**Request:**
```json
{
  "format": "csv",
  "invoice_ids": ["uuid", "uuid"]
}
```
**Response 201:**
```json
{
  "id": "uuid",
  "format": "csv",
  "filename": "gentick_sage_export_2026-05-31_001.csv",
  "invoice_count": 12,
  "download_url": "https://.../download (24h signed)"
}
```

#### `GET /api/v1/sage-exports`
**Auth:** staff. Lists generated exports.

#### `GET /api/v1/sage-exports/{id}/download`
**Auth:** staff (admin).
**Response 200:** `text/csv` or `application/xml`.

### 6.12 Messages

#### `GET /api/v1/jobs/{id}/messages`
**Auth:** staff; main customer contact (own corp); agent (own jobs).
**Filters:** `channel`, `direction`, `intent`.
**Cursor-paginated** (messages volume can be large).
**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "thread_id": "uuid",
      "direction": "outbound",
      "channel": "whatsapp",
      "template": { "name": "quote_ready", "version": 3 },
      "body": "Hi Johan, your repair quote ...",
      "sent_at": "...",
      "delivered_at": "...",
      "read_at": null,
      "classified_intent": null
    }
  ]
}
```

#### `POST /api/v1/jobs/{id}/messages`
**Auth:** staff (admin). Manual outbound message — bypasses templates for one-off cases. Goes through n8n via webhook.
**Request:** `{ "to_party_type": "main_customer", "channel": "email", "body": "..." }`

### 6.13 Parts catalog

#### `GET /api/v1/parts`
**Auth:** staff. Filters: `q`, `active`.

#### `POST /api/v1/parts`
**Auth:** staff (admin).
**Request:** `{ "code": "PSU-4G-A", "description": "4Gee PSU board assembly v1", "unit_cost": "1140.00", "currency_code": "ZAR" }`
**Response 201:** `Part`. **Errors:** `409` if `code` taken.

#### `PATCH /api/v1/parts/{id}`
**Auth:** staff (admin).
Allowed: `description`, `unit_cost`, `currency_code`, `active`. Code is immutable.

#### `DELETE /api/v1/parts/{id}`
**Auth:** staff (admin). Soft-delete only. Existing `job_parts` rows retain their snapshot cost.

### 6.14 Repair types and fault reasons

#### `GET /api/v1/repair-types`
**Auth:** any authenticated user.
**Response 200:** `{ data: RepairType[] }` (flat list, with `parent_id` for future hierarchy).

#### `POST /api/v1/repair-types`
**Auth:** staff (admin).

#### `GET /api/v1/fault-reasons`
**Auth:** any authenticated user.
**Response 200:** `{ data: FaultReason[] }` — used by the Not Repairable transition form.

### 6.15 Reports

All report endpoints accept `format=json` (default) or `format=csv`.

#### `GET /api/v1/reports/active-jobs`
**Auth:** staff (all roles).
**Description:** powers the dashboard active-jobs panel and stages strip.
**Query:** `as_of` (default now), `main_customer_id`, `assigned_technician_id`.
**Response 200:**
```json
{
  "as_of": "2026-05-31T11:30:00Z",
  "total_active": 47,
  "by_stage": [
    { "stage": "received", "count": 5 },
    { "stage": "quoted", "count": 9 },
    { "stage": "in_progress", "count": 12 }
  ],
  "aged_buckets": [
    { "bucket": "0-7d", "count": 28 },
    { "bucket": "8-14d", "count": 13 },
    { "bucket": "15-30d", "count": 5 },
    { "bucket": "30d+", "count": 1 }
  ]
}
```

#### `GET /api/v1/reports/turnaround`
**Auth:** staff.
**Query:** `period=daily|weekly|monthly|quarterly`, `from`, `to`, `repair_type_id`.
**Response 200:** `[{ period: "2026-04", avg_days: 7.2, p50_days: 6.0, p90_days: 13.5, count: 38 }, ...]`

#### `GET /api/v1/reports/technician-workload`
**Auth:** staff.
**Response 200:** per-technician open job count, avg time-in-current-stage, aged-job count.

#### `GET /api/v1/reports/revenue`
**Auth:** staff (admin).
**Query:** `period`, `from`, `to`, `currency_code`, `main_customer_id`.
**Response 200:** invoiced totals per period per currency (no auto-conversion).

#### `GET /api/v1/reports/common-faults`
**Auth:** staff.
**Query:** `from`, `to`, `unit_type`.
**Response 200:** fault counts per reason, trended monthly.

#### `GET /api/v1/reports/customer-history/{main_customer_id}`
**Auth:** staff; the main customer contact themselves.

#### `GET /api/v1/reports/intake-source`
**Auth:** staff.
**Response 200:** job count split by `originator_type` (main_customer / agent / front_desk) over time, for tracking self-service adoption.

### 6.16 Settings

#### `GET /api/v1/settings`
**Auth:** staff. Returns current value for every setting key.
**Response 200:**
```json
{
  "company_details": { "value": {...}, "version": 1, "valid_from": "..." },
  "assessment_fee": { "value": { "amount": "350.00", "currency_code": "ZAR" }, ... },
  "escalation_contact": { ... },
  "internal_champion": { ... },
  "quote_validity_days": { "value": 14, ... },
  "photo_upload_limits": { ... },
  "supported_currencies": { "value": ["ZAR", "USD"], ... },
  "vat_defaults": { ... }
}
```

#### `GET /api/v1/settings/{key}`
**Auth:** staff.

#### `GET /api/v1/settings/{key}/history`
**Auth:** staff (admin).

#### `PUT /api/v1/settings/{key}`
**Auth:** staff (admin).
**Description:** creates a new version of the setting. Previous version's `valid_to` is set to now.
**Request:** `{ "value": <any JSON>, "change_reason": "Optional explanation" }`
**Response 200:** new `Setting` row.

### 6.17 Message templates

#### `GET /api/v1/message-templates`
**Auth:** staff. Filters: `channel`, `name`, `include_retired`.

#### `POST /api/v1/message-templates`
**Auth:** staff (admin).
**Request:** `{ "name": "quote_ready", "channel": "whatsapp", "language": "en", "body": "Hi {{contact_name}}, your repair quote ..." }`
**Response 201:** `MessageTemplate` (auto-incremented version).

#### `POST /api/v1/message-templates/{id}/retire`
**Auth:** staff (admin). Sets `retired_at`. Existing messages retain reference.

### 6.18 Audit events

#### `GET /api/v1/audit-events`
**Auth:** staff (admin).
**Filters:** `entity_type`, `entity_id`, `event_type`, `actor_user_id`, `occurred_from`, `occurred_to`.
**Cursor-paginated.**

### 6.19 Dashboards

Convenience aggregated endpoints — return everything one screen needs in a single call.

#### `GET /api/v1/dashboards/staff`
**Auth:** staff. Returns KPIs (active jobs, aged, avg turnaround, monthly revenue), `by_stage[]`, `active_jobs_preview[]` (first 10), `priorities[]`, `common_faults[]`.

#### `GET /api/v1/dashboards/technician/{user_id}`
**Auth:** staff. The technician themselves, or admin.
Returns their queue and KPIs.

#### `GET /api/v1/dashboards/main-customer/{main_customer_id}`
**Auth:** staff; main customer contact of that org.

#### `GET /api/v1/dashboards/agent/{agent_id}`
**Auth:** staff; the agent themselves.

## 7. Webhook contracts

The webhook layer is the integration boundary with **n8n**. The RMS knows nothing about WhatsApp directly — it just emits events that n8n consumes and transforms.

### 7.1 Outbound — events the RMS fires to n8n

The RMS POSTs to a single n8n webhook URL configured in `settings.n8n_webhook_url`. n8n dispatches by `event_type`.

**Common envelope:**
```json
{
  "event_id": "uuid",
  "event_type": "job.received",
  "occurred_at": "2026-05-31T11:05:00Z",
  "version": "1.0",
  "payload": { ... event-specific ... }
}
```

**Signing:** every outbound webhook includes `X-Gentick-Signature: sha256=<hmac>` computed over the raw body using `settings.n8n_webhook_secret`. n8n verifies before processing.

**Retry policy:** if n8n returns non-2xx, the RMS retries with exponential backoff up to 6 attempts. Failed events accumulate in `webhook_events.last_error`.

**Event catalogue:**

| Event type | Triggered by | Payload includes |
| --- | --- | --- |
| `job.received` | Shipment received | job + customer + agent + jobcard PDF URL |
| `job.extra_unit_discovered` | Discover-extras endpoint | job + originator + notify-both flag |
| `quote.ready` | Quote issued | quote + customer + agent + quote PDF URL + total + currency |
| `quote.approved` | Quote approval recorded | quote + decision channel |
| `quote.rejected` | Quote rejection recorded | quote + decision channel |
| `job.in_progress` | Transition to in_progress | job + technician |
| `job.qc` | Transition to QC | job |
| `job.boxed` | Transition to boxed | job |
| `job.ready_for_collection` | Transition to ready | job + agent (the notify target) |
| `job.collected` | Transition to collected | job |
| `job.closed` | Transition to closed | job + invoice PDF URL |
| `job.not_repairable` | Transition to not_repairable | job + reason + assessment fee invoice URL |
| `job.cancelled` | Transition to cancelled | job + reason |
| `agent.invited` | Agent invitation issued | invitation + invite link |

**Example payload:**
```json
{
  "event_id": "9c4...e1",
  "event_type": "quote.ready",
  "occurred_at": "2026-05-31T09:14:00Z",
  "version": "1.0",
  "payload": {
    "quote": {
      "id": "uuid", "quote_number": "Q-GT-ACME-0184-1", "total": "3553.50", "currency_code": "ZAR",
      "expires_at": "2026-06-14T09:14:00Z", "pdf_url": "https://.../quote.pdf (24h signed)"
    },
    "job": { "id": "uuid", "job_number": "GT-ACME-0184" },
    "main_customer": {
      "corporation_name": "Acme Agritech",
      "primary_contact": { "name": "Johan Kruger", "email": "johan@acme.co.za", "phone": "+27821111111",
                           "whatsapp_consent": true }
    },
    "agent": {
      "agent_name": "Sipho Mthembu", "email": "sipho@karoofarms.co.za", "phone": "+27822222222",
      "whatsapp_consent": true
    }
  }
}
```

### 7.2 Inbound — n8n posts to the RMS

n8n authenticates as the service account via `Authorization: Bearer <service_jwt>`.

#### `POST /api/v1/webhooks/whatsapp/inbound`
**Auth:** n8n service account.
**Description:** records an inbound WhatsApp message and runs classification (YES / NO / question / other). If YES on a pending quote, auto-records the quote approval. Otherwise routes to staff per `settings.escalation_contact`.
**Request:**
```json
{
  "external_id": "wamid.HBgL...",
  "from_phone": "+27821111111",
  "received_at": "2026-05-31T09:31:00Z",
  "body": "YES — approve please",
  "context": {
    "in_reply_to_external_id": "wamid.HBgK..."
  }
}
```
**Response 200:**
```json
{
  "message_id": "uuid",
  "classified_intent": "approval_yes",
  "matched_quote_id": "uuid",
  "action_taken": "quote_approved",
  "escalated_to": null
}
```

#### `POST /api/v1/webhooks/whatsapp/delivery`
**Auth:** n8n service.
**Description:** delivery / read receipts for outbound messages.
**Request:**
```json
{
  "external_id": "wamid.HBgK...",
  "status": "delivered" | "read" | "failed",
  "occurred_at": "...",
  "error": "optional error string"
}
```

#### `POST /api/v1/webhooks/email/inbound`
**Auth:** n8n service.
**Description:** inbound email replies (e.g., quote approval via email). Same classification logic as WhatsApp.

#### `POST /api/v1/webhooks/email/delivery`
**Auth:** n8n service.
**Description:** SES / SendGrid / Postmark delivery + bounce reporting.

## 8. File uploads

Photos are the only file type currently uploaded by users. Sage exports and PDFs are server-generated and served on-demand.

### Limits

- Max size: 5 MB per photo (from `settings.photo_upload_limits`).
- Max count: 20 photos per job (from same setting).
- Allowed mime types: `image/jpeg`, `image/png`, `image/webp`, `image/heic`.

### Storage

Photos are stored under `/uploads/jobs/{job_id}/{uuid}.{ext}` on the Hetzner VPS local filesystem during MVP. A `getFile(path) -> Buffer` and `saveFile(buffer, path) -> void` abstraction lives in `lib/storage/`. Swap implementation to Cloudflare R2 by changing the abstraction body — no other code changes.

### Signed URLs

Photos are served via signed URLs valid for 24 hours: `GET /api/v1/files/{photo_id}?token=...`. The signed URL is the `signed_url` returned by photo endpoints. Direct filesystem path is never exposed.

## 9. PDF generation

Quotes, invoices, and jobcards are rendered server-side using HTML-to-PDF (recommended: `puppeteer` against a templated React component, OR `@react-pdf/renderer` for simpler templates).

For quotes and invoices, **always render from the snapshot JSON, not the live database**. This is the entire point of immutable snapshots. A regression test should assert: "re-issuing a 2-year-old invoice produces the exact same bytes."

## 10. Versioning and deprecation policy

- Breaking changes go in a new major version: `/api/v2/`. The old version is supported in parallel for at least 6 months.
- Additive changes (new fields in responses, new optional request fields, new endpoints) do not bump the version.
- Deprecated endpoints return `Sunset: <RFC3339-date>` headers per RFC 8594.

## 11. Resource quick-reference

The resource shapes returned by endpoints map 1:1 to the schema entities documented in `Gentick_RMS_Schema_Design_v1.0.md`. The OpenAPI spec (`Gentick_RMS_OpenAPI_v1.0.yaml`) carries the precise field-level types.

## Appendix A: HTTP status codes used

| Code | Meaning in this API |
| --- | --- |
| 200 | OK — read success or action success without new resource |
| 201 | Created — new resource created |
| 202 | Accepted — async action queued (used for n8n webhook ingestion ack) |
| 204 | No Content — successful action with no body |
| 400 | Validation or business-rule error |
| 401 | Not authenticated |
| 403 | Forbidden (authenticated but not permitted) |
| 404 | Not found |
| 409 | Conflict (duplicates, concurrent edits) |
| 410 | Gone (hard-deleted under retention purge) |
| 413 | Payload too large |
| 422 | Business rule violation |
| 429 | Rate limited |
| 500 | Internal server error |
| 503 | Service unavailable (planned maintenance) |

## Appendix B: cURL examples

**Login (browser flow uses cookies, but for n8n / scripting):**
```bash
curl -X POST https://rms.gentick.co.za/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{ "email": "lucia@gentick.co.za", "password": "..." }'
```

**Receive a shipment (front desk):**
```bash
curl -X POST https://rms.gentick.co.za/api/v1/shipments \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d @shipment.json
```

**Transition a job:**
```bash
curl -X POST https://rms.gentick.co.za/api/v1/jobs/$JOB_ID/transition \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{ "to_stage": "in_progress", "notes": "Parts arrived" }'
```

**Approve a quote (main customer):**
```bash
curl -X POST https://rms.gentick.co.za/api/v1/quotes/$QUOTE_ID/approve \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{ "channel": "portal" }'
```

**Generate a Sage export:**
```bash
curl -X POST https://rms.gentick.co.za/api/v1/sage-exports \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{ "format": "csv", "invoice_ids": ["uuid1", "uuid2"] }'
```
