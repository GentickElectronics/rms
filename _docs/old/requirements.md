# Gentick Repair Management System (RMS)

## Requirements Document

**Version:** 1.1
**Date:** 24 June 2026
**Owner:** Lucia, Gentick Electronics
**Status:** Active — authentication & customer-management enhancements from build/test feedback

---

## Changelog

**v1.1 (24 June 2026)** — Authentication and customer-management enhancements arising from implementation and testing:

- **Primary contact captured at customer registration (FR-INT-01 revised).** Corporation Name, Contact Person, Contact Email, and Contact Phone are captured in a single registration step; the corporation and its primary contact are created together. All four fields are mandatory and validated inline.
- **Email verification for external users (new FR-INT-14).** Customer contacts and agents must confirm their email address with a 6-digit code before their first sign-in; login is blocked until the email is verified. Staff accounts remain admin-provisioned and pre-confirmed.
- **Multiple contacts per corporation (new FR-INT-15).** A main customer may have several portal contacts, exactly one of which is the primary. Admin/Front Desk can add additional (non-primary) contacts; only the primary contact may invite agents.
- **Form input validation (new FR-INT-16).** Email format and E.164 phone validation, with all missing/invalid required fields surfaced inline before save.
- **NFR-04 (Security) updated** to include external-user email verification.

**v1.0 (31 May 2026)** — Locked baseline. Final operational defaults confirmed:

- **Job number format updated** to per-customer prefix: `GT-{CUSTOMER_PREFIX}-NNNN` (e.g., `GT-ACME-0001`). Walk-in / unlinked jobs use `GT-GEN-NNNN`. Schema gains a `customer_prefix` field on MainCustomer.
- **WhatsApp escalation contact** added as a configurable system setting. Default: Michael (082 374 9842).
- **Internal champion / first-line support** added as a configurable system setting. Default: Michael.
- All other Section 14 defaults from v0.3 accepted as proposed.
- Section 14 renamed "Resolved Operational Defaults" — items are now decisions of record.

**v0.3 (31 May 2026)** — Incorporates Lucia's v0.2 review:

- **Assessment fee narrowed.** FR-QUO-09 now triggers only on Not Repairable. Rejected quotes incur no fee. State machine (Section 10) updated to match.
- **Gentick company details locked.** Legal name, address, and VAT number captured. New FR-ADM-08 makes these a system setting used on all customer-facing documents.
- **Agent password reset.** FR-INT-13 added: self-service via Supabase Auth email link.
- **Phase 2 hook for threshold-based agent quote approval** added to Section 12.2.
- **Agent visibility of pricing.** Agents see full quote details, including R-value (no price-opacity logic — main customer forwards the quote anyway).
- Open questions 4, 6, 7, 8, 9 closed.

**v0.2 (31 May 2026)** — Added agent-initiated intake, multi-currency (ZAR + USD), VAT zero-rating for exports, assessment fee for non-repairable units, Hetzner Cape Town hosting, Setting entity.

**v0.1 (16 May 2026)** — Initial draft.

---

## 1. Executive Summary

Gentick Electronics designs and manufactures telemetry units and electronic sensors. The business currently has no formal system for tracking repair jobs, customer communications, or repair-related reporting. This document specifies the requirements for a custom web-based Repair Management System (RMS) that will:

1. Capture and track repair jobs from intake through dispatch.
2. Manage customer and agent records with POPIA-compliant consent.
3. Notify customers and agents at key workflow milestones via WhatsApp and email.
4. Generate quotes and proforma invoices in ZAR or USD, and feed billing data into Sage Pastel Online.
5. Surface operational and product-improvement insights through reporting.

The system will be self-hosted by Gentick on Hetzner Cape Town infrastructure, built using Next.js, PostgreSQL, and supporting services. The first usable version (MVP) is targeted at 4–6 weeks of development effort, with phased enhancements thereafter.

## 2. Project Background and Goals

### 2.1 Business context

Gentick repairs its own telemetry units and sensors. Repairs typically flow as follows:

- A **main customer** (a corporation that resells or services Gentick equipment) **or** one of their **agents** (end users or field operators) initiates a repair.
- The broken unit is shipped to Gentick, and the repaired unit is dispatched directly back to the agent.
- Gentick communicates with both the main customer (for approvals and quoting) and the agent (for shipping and collection).

This two-tier relationship is the single most important shape of the business and drives the data model, authentication design, and notification routing. Because agents can initiate intake themselves, the agent portal must be intake-capable, not read-only.

### 2.2 Goals

- Eliminate paper- and spreadsheet-based repair tracking.
- Give customers and agents self-service visibility AND self-service intake for their repair jobs.
- Reduce the manual effort of sending status updates.
- Generate quotes and proforma invoices in seconds rather than hours.
- Provide management with reliable operational reporting.
- Surface common faults to drive product improvement.

### 2.3 Success measures

- Average turnaround time per repair, trended monthly.
- Percentage of jobs with all status milestones recorded.
- Percentage of customers and agents with valid WhatsApp opt-in.
- Volume of customer support phone calls / "where is my repair?" enquiries (expected to drop).
- Percentage of jobs that come in through the self-service intake (agent or main customer) vs Gentick front desk.
- Number of common-fault patterns identified per quarter and fed back into engineering.

### 2.4 Gentick company details (used on customer-facing documents)

| Field | Value |
| --- | --- |
| Legal name | Gentick Electronics |
| Address | 169 Haupt Street, Sidwell, Port Elizabeth, 6001, South Africa |
| VAT number | 4840244414 |

These values are stored as a system setting (FR-ADM-08) and printed on every quote, proforma invoice, and jobcard.

## 3. Scope

### 3.1 In scope (MVP)

- Customer and agent registration with POPIA opt-in capture.
- Main-customer-led agent onboarding (main customer invites their agents).
- Agent self-service password reset via email link.
- Job intake from three sources: main customer, agent, Gentick front desk.
- Multi-unit shipments — single shipment can contain multiple jobs.
- Job workflow tracking through 10 defined stages.
- Photo uploads per job (fault evidence, before/after).
- Quote generation and email delivery (always to the main customer).
- Multi-currency support — ZAR and USD for MVP, with VAT rules.
- Quote approval capture (via email reply or WhatsApp YES).
- Proforma invoice generation as PDF + CSV/XML export for Sage Pastel Online.
- Assessment fee for Not Repairable units — configurable system setting, default R350.
- WhatsApp notifications at key milestones (received, quote ready, ready for collection, not repairable).
- Customer portal with split visibility (main customer sees all their agents' jobs; agent sees their own jobs including full quote details).
- Internal staff portal with role-based access (admin, technician, front desk).
- Active jobs dashboard.
- Basic reporting (turnaround time, technician workload, revenue, common faults, intake source).
- Parts/inventory tracking per job (consumption only — not procurement).
- Audit log of consent events and significant data changes.
- Admin-managed system settings (company details, assessment fee, VAT default, currency list, message templates).

### 3.2 In scope (Phase 2)

- Direct Sage Pastel Online API integration if/when API maturity supports it.
- Dedicated technician mobile app (Flutter or React Native).
- Customer feedback / NPS capture post-completion.
- Knowledge base of common faults and recommended fixes.
- Parts procurement and stock level tracking.
- Multi-warehouse / multi-site support.
- FX-rate feed + ZAR-converted reporting.
- Additional foreign currencies beyond USD.
- **Threshold-based agent quote approval** — agents can approve quotes below a configurable R-value threshold (e.g., under R1,000) without main-customer involvement. The MVP schema must include the fields and audit hooks to support this so it is not a rewrite later.

### 3.3 Out of scope

- Payment processing (Sage Pastel Online owns this).
- Accounting ledgers / general ledger functions.
- Marketing automation beyond transactional repair notifications.
- Customer billing collections (handled in Sage Pastel Online).
- Field service dispatch / on-site repair scheduling.

## 4. User Roles and Personas

### 4.1 Roles

| Role | Description | Portal | Can initiate repairs? |
| --- | --- | --- | --- |
| **Gentick Admin** | Full system access, manages users and settings | Internal | Yes (on behalf) |
| **Gentick Technician** | Updates jobs assigned to them, logs parts used | Internal | No |
| **Gentick Front Desk** | Intake and dispatch, manages customer records | Internal | Yes |
| **Main Customer Contact** | Sees all jobs across their agents, approves all quotes, invites their own agents | External | Yes |
| **Agent** | Sees own jobs (including full quote details), can initiate intake for their own units, ships units directly to Gentick | External | Yes |
| **System / Automation** | n8n workflows, scheduled jobs — service account, no human login | (none) | — |

**Key authority rule:** quote approval is always the main customer's responsibility, regardless of who initiated the repair. (Phase 2 may introduce a threshold below which agents can self-approve — see Section 12.2.)

**Contact authority:** a main customer may have multiple portal contacts but exactly one **primary** contact. Only the primary contact may invite/add agents for the corporation; non-primary contacts have visibility but cannot invite agents (see FR-INT-15).

### 4.2 Personas

**Pieter — Gentick Front Desk.** Receives shipments. Needs a fast intake flow that handles multi-unit boxes (whether pre-registered by the agent/main customer or not), prints a jobcard, and triggers customer acknowledgement automatically. Pain point today: lost paperwork and uncertainty about which agent a unit belongs to.

**Naledi — Gentick Technician.** Diagnoses and repairs units. Needs to see her own queue, log diagnosis notes, attach photos, record parts used, and update status. Pain point today: she has to ask the front desk for unit history every time.

**Lucia — Gentick Admin / Owner.** Wants operational visibility (active jobs, turnaround) and product-improvement signal (common faults). Approves new staff users, tweaks templates, sets the assessment fee and company details. Pain point today: cannot answer "how are we doing?" without a manual data pull.

**Johan — Main Customer Contact.** Works at the corporate that resells Gentick equipment. Invites his agents into the system, sees all their jobs, approves all quotes, and forwards approved quote details to his agents for their own records. Wants confidence that no spend slips by without his sign-off. Pain point today: chases Gentick by phone for status updates.

**Sipho — Agent.** Field technician at one of the corporate's sites. When a unit fails, he logs into his portal, books it in, ships it directly to Gentick, and tracks the repair end-to-end. Sees the full quote (including price) once Johan has approved it. Approves nothing financial — Johan handles the quote. Pain point today: opaque turnaround, has to go via Johan even to log a job.

## 5. Functional Requirements

### 5.1 Intake and Customer Management

**FR-INT-01** — The system shall capture the following at main customer registration, in a single step: Corporation Name (with a 2–10 character customer prefix), Contact Person, Contact Email, and Contact Phone Number. Registering a main customer creates the corporation together with its **primary** contact. All four fields are mandatory and validated inline (field-level), with every missing or invalid field surfaced at once before the record can be saved.

**FR-INT-02** — The system shall capture the following agent fields, linked to a main customer: Agent Name, Agent Phone Number, Agent Email, Agent Address.

**FR-INT-03** — The system shall capture WhatsApp opt-in consent from both the main customer contact and the agent at first registration, with timestamp, IP address (if collected via web), and the wording version they consented to.

**FR-INT-04** — The system shall provide an opt-out toggle in the customer/agent profile that immediately revokes WhatsApp consent and logs the revocation.

**FR-INT-05** — The system shall provide an annual consent audit workflow that flags consents older than 12 months for renewal.

**FR-INT-06** — Job intake shall support three origin paths: (a) main customer creates the job and ships the unit to Gentick, (b) agent creates the job and ships the unit directly to Gentick, (c) Gentick front desk creates the job on receipt of an unannounced unit. The data captured is identical in all three paths.

**FR-INT-07** — Job intake shall support registering multiple units in a single shipment. Each unit gets its own job number but shares a shipment identifier for traceability.

**FR-INT-08** — Each job shall be assigned a unique, human-readable job number in the format `GT-{CUSTOMER_PREFIX}-NNNN`, where `CUSTOMER_PREFIX` is the prefix configured on the main customer record (e.g., `GT-ACME-0001`). Sequence is per-customer and zero-padded to 4 digits, expanding as needed. Walk-in / unlinked jobs use the fallback prefix `GT-GEN-NNNN`.

**FR-INT-09** — Per unit, the system shall capture: unit type, serial number, fault description (free text), accessories shipped with the unit, intake photos.

**FR-INT-10** — Jobcards shall be emailable to the originator at the point of intake. Both the main customer AND the agent shall be able to view all their relevant jobcards (current and historical) through the portal.

**FR-INT-11** — Agent onboarding shall be initiated by the main customer through their portal. The main customer adds the agent's details, the system emails an invitation link, and the agent completes their own profile (including WhatsApp opt-in) on first login.

**FR-INT-12** — If the front desk discovers additional units in a shipment beyond what was registered:
- (a) When the originator was the main customer: notify both main customer and agent.
- (b) When the originator was the agent: notify the agent and copy the main customer for visibility.
- (c) When there was no pre-registration (walk-in / unannounced): create the jobs at intake and route the notification based on whichever party the unit is identified to belong to (front desk selects).

**FR-INT-13** — Password reset for all external users (main customer contacts and agents) shall be self-service via a Supabase Auth email link. No staff intervention required.

**FR-INT-14** — Email verification for external users. When a login is provisioned for a main customer contact or an agent, the system shall create the account in an unverified state and email the user a 6-digit verification code. The user must enter the code on a verification page to confirm their email address before their first sign-in; sign-in is blocked until the email is verified. The code shall be re-sendable, and incorrect or expired codes shall be rejected with a clear message. Staff accounts (Admin, Technician, Front Desk) are provisioned by an administrator and are exempt from this self-verification step.

**FR-INT-15** — Multiple contacts per main customer. A main customer may have more than one portal contact, exactly one of which is designated the **primary** contact. Admin and Front Desk users may add additional (non-primary) contacts to a corporation. Only the primary contact may invite agents for that corporation (see §4.1); non-primary contacts have portal visibility but cannot invite agents.

**FR-INT-16** — Form input validation. Customer, contact, and agent forms shall validate required fields, email address format, and phone numbers (E.164, e.g. +27821234567) before submission. Validation errors shall be shown inline at field level, all missing or invalid fields surfaced together, and invalid records shall be rejected (no partial/invalid save).

### 5.2 Workflow Tracking

**FR-WF-01** — Jobs shall move through the following defined stages: Received → Diagnosed → Quoted → Approved → In Progress → QC → Boxed → Ready for Collection → Collected → Closed. Refer to Section 10 for the full state machine.

**FR-WF-02** — Each stage transition shall record: timestamp, the staff user who triggered it, the previous and new stage, optional notes.

**FR-WF-03** — Technicians can be assigned to jobs. Reassignment is permitted and logged.

**FR-WF-04** — Jobs shall capture diagnosis notes (technician-only visibility) and customer-facing notes (visible in the portal).

**FR-WF-05** — Multiple photos can be attached per job at any stage with optional captions and a "visibility" flag (internal-only or customer-facing).

**FR-WF-06** — Where a unit is deemed not repairable, the system shall require a structured reason from the following options: lightning damage, secondary lightning damage, water damage, component failure, power supply failure, chip-to-probe communication failure, other (free text).

**FR-WF-07** — Jobs in the Boxed stage shall trigger a "ready for collection" notification automatically to the agent.

### 5.3 Customer Communication

**FR-COM-01** — The system shall send communication to the relevant party (main customer or agent — see Section 8) at the following milestones: item received, quote ready, item ready for collection, item not repairable.

**FR-COM-02** — Communication channels are WhatsApp (primary, if consent given) and email (fallback / always).

**FR-COM-03** — Quote approvals shall be capturable via two channels: (a) a "YES" reply on WhatsApp, (b) an email reply or web-portal click-to-approve action. Only the main customer can approve a quote, even when the repair was initiated by an agent.

**FR-COM-04** — Every outbound and inbound communication shall be logged against the job, with channel, timestamp, sender/recipient, and content.

**FR-COM-05** — Outbound message templates shall be configurable by an admin user, with versioning so historical messages retain the wording sent at the time.

### 5.4 Quoting and Billing

**FR-QUO-01** — Quotes shall be generated within a job and include: labour estimate, parts cost (line items), notes, validity period, total inclusive of VAT, currency code.

**FR-QUO-02** — Quotes shall be issued by email to the main customer, regardless of who initiated the repair. The agent is informed but not asked to approve. Once the main customer has approved, the agent sees the full quote (including R-value) in their portal.

**FR-QUO-03** — Quote approval shall be captured per FR-COM-03 and shall transition the job to the Approved stage automatically.

**FR-QUO-04** — On completion (job closed), the system shall generate a proforma invoice as a downloadable PDF.

**FR-QUO-05** — The system shall export proforma invoice data in a CSV/XML format compatible with Sage Pastel Online's import format. Direct API integration is deferred to Phase 2 pending Sage API maturity.

**FR-QUO-06** — Payment status is tracked in Sage Pastel Online, not in the RMS. The RMS shall display "Invoiced" / "Not invoiced" based on whether an export has been generated; settlement status is not mirrored.

**FR-QUO-07** — The system shall support quoting and invoicing in multiple currencies. MVP currencies are ZAR (default) and USD. Each quote and invoice stays in its native currency end-to-end; no conversion is performed for reporting in MVP.

**FR-QUO-08** — VAT default rules:
- ZAR quotes default to 15% VAT.
- Non-ZAR quotes default to 0% (SA export zero-rating).
- An admin or quoting user can override the VAT rate per quote with a captured reason.

**FR-QUO-09** — When a unit is moved to the Not Repairable terminal state, the system shall automatically generate a proforma invoice for the assessment fee (default R350, taken from system settings; configurable per job; waivable by admin with a captured reason). **A Rejected quote (main customer declines) does NOT trigger an assessment fee.**

**FR-QUO-10** — All quotes and proforma invoices shall include Gentick's legal name, physical address, and VAT number, sourced from the company-details system setting (FR-ADM-08).

### 5.5 Reporting and Operations

**FR-REP-01** — The system shall provide an active jobs dashboard showing job counts per stage, jobs older than X days, and an aging breakdown.

**FR-REP-02** — Technician workload report — open jobs per technician with average time-in-stage.

**FR-REP-03** — Turnaround time report — average days from Received to Collected, sliceable by month and repair type.

**FR-REP-04** — Revenue report — invoiced value by period, by repair type, by main customer. Currency is shown alongside each value; no ZAR conversion in MVP.

**FR-REP-05** — Customer history report — per customer (main or agent), all historical jobs with quick-view to common fault patterns.

**FR-REP-06** — Common faults report — frequency of each fault classification, trended over time, to feed engineering reviews.

**FR-REP-07** — All reports shall be exportable to CSV.

**FR-REP-08** — Intake source report — split of jobs by origin (main customer, agent, front desk) to track self-service adoption.

### 5.6 Admin

**FR-ADM-01** — Staff user management with the three roles defined in Section 4.1.

**FR-ADM-02** — Parts catalogue — admin-maintained list of parts with code, description, cost, and currency. Technicians select from this list when logging usage.

**FR-ADM-03** — Per-job parts consumption — quantity used per part, linked to the job and feeding the quote.

**FR-ADM-04** — Repair type taxonomy — admin-maintained list of repair categories used for reporting.

**FR-ADM-05** — Message template management with versioning per FR-COM-05.

**FR-ADM-06** — Audit log viewer for consent events, user logins, role changes, and significant data edits.

**FR-ADM-07** — System settings management — admin-only UI for configurable values: default assessment fee, supported currencies, default VAT rates per currency, jobcard print template, quote validity period, photo upload limits.

**FR-ADM-08** — Company details system setting — admin-only UI to manage Gentick's legal name, physical address, and VAT number. These values are sourced by all quote, invoice, and jobcard generators per FR-QUO-10. Settings are versioned so historical documents always reflect the company details in effect at the time of issue.

**FR-ADM-09** — Operational contacts system setting — admin-only UI to manage:
- **WhatsApp escalation contact** — the person (name + mobile number) who receives unclassified WhatsApp inbound messages and serves as the manual escalation point. Default: Michael (082 374 9842).
- **Internal champion / first-line support contact** — the person who acts as the in-business product owner and first-line user support after launch. Default: Michael.

Both values can be reassigned at any time by an admin without code changes.

## 6. Non-Functional Requirements

**NFR-01 — Performance.** Page loads under 2 seconds on a 4G connection for staff users; under 3 seconds for customer portal users. Job list page should support 1,000 jobs without pagination performance issues.

**NFR-02 — Availability.** Target 99% uptime measured monthly. Planned maintenance windows announced 48 hours in advance.

**NFR-03 — Backup.** Daily automated backups of the PostgreSQL database and the file storage area. Off-site backup copy retained 30 days minimum.

**NFR-04 — Security.** All traffic over HTTPS. Passwords hashed with bcrypt or equivalent (handled by Supabase Auth). Multi-factor authentication available for staff users. External users (main customer contacts and agents) must verify their email address via a 6-digit code before first sign-in (see FR-INT-14). Secrets stored in n8n's encrypted credentials store, never in code or workflow JSON.

**NFR-05 — POPIA compliance.** Lawful basis for processing recorded; consent revocable; subject access requests answerable within 30 days; data hosted in Hetzner Cape Town (SA region) — no cross-border data transfer.

**NFR-06 — Browser support.** Latest 2 versions of Chrome, Edge, Firefox, Safari. Mobile-responsive (especially the customer portal — agents will primarily use phones, including for self-service intake).

**NFR-07 — Accessibility.** WCAG 2.1 AA for the customer portal at minimum.

**NFR-08 — Scalability.** Designed for hundreds of jobs per month. Should handle 10x growth without architectural change.

**NFR-09 — Maintainability.** TypeScript throughout. Code conventions defined in a project README. Database schema versioned via migrations.

**NFR-10 — Observability.** Application logs centralised. Error alerts to admin email. n8n error workflow capturing automation failures.

## 7. Integrations

### 7.1 WhatsApp via Meta Cloud API and n8n

The RMS does not call WhatsApp directly. The flow is:

1. RMS emits an event (e.g., `job.status_changed`, `quote.ready`) to an n8n webhook.
2. n8n decides which template to send and to which phone number based on the event type and customer/agent.
3. n8n calls the Meta Cloud API.
4. n8n logs the outbound message back to the RMS via API.

Inbound replies follow the reverse path: Meta webhook → n8n → classified (YES approval, question, other) → either auto-responded or routed to staff and logged against the job.

### 7.2 Sage Pastel Online (proforma invoice export)

Sage Pastel Online's API is limited. The integration approach for MVP:

- Proforma invoices generated in the RMS as PDF.
- Customer and invoice data exported in CSV/XML matching Sage Pastel Online's import format.
- Staff member imports the file into Sage Pastel Online manually on a daily/weekly cadence.
- Payment status is tracked exclusively in Sage Pastel Online.

Phase 2 may revisit direct API integration if Sage Pastel Online's API matures or a partner connector is available.

### 7.3 Email

Transactional emails (jobcards, quotes, status updates, proforma invoices, agent invitations, password resets) sent via n8n. Recommended provider is a transactional ESP such as Postmark, SendGrid, or Mailgun. Domain authentication (SPF, DKIM, DMARC) configured on `gentick.co.za` to maintain deliverability.

## 8. Notification Routing Matrix

| Event | Main Customer | Agent | Channel(s) |
| --- | --- | --- | --- |
| Agent invited (welcome) | No | Yes | Email |
| Item received at Gentick (any origin) | Yes | Yes | WhatsApp + email |
| Extra units discovered | Yes | Yes | WhatsApp + email |
| Quote ready | Yes | No (until approved) | Email primary + WhatsApp to main customer for approval |
| Quote approved (confirmation) | Yes | Yes (sees full price) | WhatsApp + email |
| Item not repairable | Yes | Yes | WhatsApp + email (with reason report + assessment-fee invoice) |
| Item boxed / ready for collection | No | Yes | WhatsApp + email |
| Item collected | Yes | Yes | Email |
| Job closed | Yes | No | Email (with proforma invoice) |

## 9. Technical Architecture (high level)

```
[Customer browser] ─┐
[Agent browser]    ─┼─► Next.js (web + API routes)
[Staff browser]    ─┘            │
                                 │
                            ┌────┴────┐
                            │         │
                    [PostgreSQL]  [Local file storage]
                            │         │
                            └────┬────┘
                                 │
                          n8n (Docker) ──► Meta Cloud API ──► Customer's phone
                                 │
                          Email ESP (Postmark/SendGrid)
                                 │
                          CSV/XML export ──► Sage Pastel Online (manual import)
```

All components run via Docker Compose on a single Hetzner Cape Town VPS for MVP. PostgreSQL, n8n, and the Next.js app are separate containers. Photo uploads stored under `/uploads/jobs/{job_id}/` and abstracted behind a storage interface so a swap to Cloudflare R2 is a one-class change.

A detailed architecture diagram is delivered in Phase 2.

## 10. Repair Workflow State Machine

Allowed transitions:

```
Received ──► Diagnosed ──► Quoted ──► Approved ──► In Progress ──► QC ──► Boxed ──► Ready for Collection ──► Collected ──► Closed
   │            │             │
   │            │             └─► Rejected (terminal — NO assessment fee)
   │            └──────────────► Not Repairable (terminal, with reason + assessment fee)
   └─────────────────────────► Cancelled (terminal — admin only, no fee)
```

Notes:

- **Not Repairable** is a structured terminal state captured per FR-WF-06 with a defined reason set. Triggers automatic assessment-fee proforma invoice per FR-QUO-09.
- **Rejected** = main customer declined the quote. No fee. Unit is returned to the agent or scrapped per main customer instructions.
- **Cancelled** = administrative cancel before any work began. No fee.
- Each transition records actor, timestamp, and optional notes.
- The transition from Quoted → Approved is the only one that can be triggered by the customer (the main customer specifically, via WhatsApp YES or email approval); all others are staff-triggered.

## 11. Data Model Overview

Detailed schema and ERD are delivered in Phase 2. At the requirements level, the core entities are:

- **MainCustomer** — corporation record. Includes a unique `customer_prefix` field (uppercase short code, e.g., `ACME`) used to generate job numbers per FR-INT-08.
- **MainCustomerContact** — person at the corporation (one-to-many to MainCustomer).
- **Agent** — downstream customer linked to a MainCustomer. Has login credentials and intake permissions.
- **AgentInvitation** — pending invite issued by a main customer, with token and expiry.
- **ConsentRecord** — consent events per contact/agent with version, channel, granted/revoked timestamps.
- **Shipment** — physical batch of units received (groups Jobs). Has an origin field: `main_customer` | `agent` | `walk_in`.
- **Job** — single repair task for a single unit. Has an originator field linking back to whoever initiated it.
- **JobStage** / **JobStatusHistory** — current stage and full transition log.
- **JobPhoto** — file references with visibility flag.
- **JobNote** — internal vs customer-facing notes.
- **PartCatalog** / **JobPart** — parts master and per-job consumption (with currency).
- **Quote** / **QuoteLineItem** / **QuoteApproval** — with currency and VAT rate per quote. (Phase 2 hook: an `approval_authority` field anticipates agent-self-approval below threshold.)
- **Invoice** (proforma) — including assessment-fee invoices.
- **Message** / **MessageThread** — communication log across channels.
- **StaffUser** — Gentick staff with role.
- **AuditEvent** — security and compliance audit log.
- **Setting** — system settings (company details, operational contacts, assessment fee, supported currencies, default VAT rates per currency, quote validity, etc.). Versioned so historical jobs retain the setting that was in effect at the time.

## 12. Phasing and MVP Definition

### 12.1 Phase 1 — MVP (4–6 weeks)

- Customer/agent registration with consent
- Main-customer-led agent invitation flow
- Agent self-service password reset
- Job intake (multi-unit supported) from three origin paths
- Full workflow tracking with all 10 stages
- Photo uploads
- Quote generation + email + approval capture (main-customer-only)
- Multi-currency (ZAR + USD) with VAT default rules
- Assessment fee on Not Repairable
- Proforma invoice PDF + CSV export, with Gentick company details
- WhatsApp notifications at the 4 key milestones
- Customer portal (main + agent split with agent intake capability; agent sees full quote details after approval)
- Staff portal with 3 roles
- Active jobs dashboard
- 4 core reports (turnaround, workload, revenue, intake source)
- Parts catalogue + per-job parts log
- Audit log
- System settings UI (including company details)

### 12.2 Phase 2 — Enhancements (post-MVP)

- Common faults report and product-improvement dashboard
- Customer history report
- Direct Sage Pastel Online API (if viable)
- Technician mobile app
- Customer feedback capture
- Multi-warehouse support
- FX rate feed + ZAR-normalised reporting
- Additional foreign currencies beyond USD
- **Threshold-based agent quote approval** — agents approve quotes below a configurable R-value threshold without main-customer involvement. MVP data model includes the fields and audit hooks to support this.

### 12.3 Phase 3 — Optional

- Parts procurement and stock tracking
- Knowledge base for common faults
- Field service / on-site repair scheduling

## 13. Compliance and Risk

**POPIA.** Gentick is the responsible party. Consent is captured per FR-INT-03 and revocable per FR-INT-04. Annual audit per FR-INT-05. Data hosted in Hetzner Cape Town (SA) — no cross-border data transfer.

**Meta Cloud API compliance.** WhatsApp business messages require pre-approved templates for transactional notifications. Templates will be drafted and submitted as part of the Phase 2 deliverables.

**SARS / VAT compliance.** Gentick is VAT-registered (VAT number 4840244414). All quotes and proforma invoices include the legal name, physical address (169 Haupt Street, Sidwell, Port Elizabeth, 6001), and VAT number, per FR-ADM-08 and FR-QUO-10. Foreign-currency quotes default to 0% VAT under SA export zero-rating. Admin can override per quote.

**Data retention.** Job records and communication history retained for 7 years post-closure (aligned with SARS record-keeping requirements). Photos retained 2 years post-closure unless required for a longer-running dispute.

**Operational risk.** Self-hosting means Gentick is responsible for uptime. Mitigations: reputable provider (Hetzner Cape Town), UptimeRobot monitoring, daily backups, documented runbooks (Phase 2).

## 14. Resolved Operational Defaults

All operational defaults are confirmed and form part of the v1.0 baseline. Items marked as system settings can be changed in-app by an admin without code changes.

| # | Item | Decision | Type |
| --- | --- | --- | --- |
| 1 | Job number format | `GT-{CUSTOMER_PREFIX}-NNNN` per FR-INT-08; fallback `GT-GEN-NNNN` for walk-ins | Hardcoded |
| 2 | Customer prefix | Per-main-customer short code (e.g., `ACME`), set on customer creation | Per-customer field |
| 3 | Photo upload limits | 20 photos per job, 5 MB each | System setting |
| 4 | Quote validity period | 14 days from issue, then auto-expires | System setting |
| 5 | Multi-currency dashboard totals | Currencies shown separately (no rolled total) for MVP | Hardcoded |
| 6 | Jobcard print format | Single-page A4 with Gentick header (sourced from FR-ADM-08) | System setting |
| 7 | Email-from address | `repairs@gentick.co.za` | System setting |
| 8 | WhatsApp display name | "Gentick Electronics" | System setting |
| 9 | WhatsApp escalation contact | Michael, 082 374 9842 — receives unclassified inbound | System setting (FR-ADM-09) |
| 10 | Internal champion / first-line support | Michael | System setting (FR-ADM-09) |
| 11 | Backup retention | Daily backups 30 days; monthly snapshots 12 months; off-site copy in a second Hetzner region | Infrastructure config |
| 12 | Staff training plan | One-day handover with Lucia plus a written runbook produced in Phase 4 | Process |

## 15. Glossary

- **Agent** — The end user of a Gentick unit; can initiate repair jobs themselves through the agent portal and ships units directly to Gentick.
- **Assessment fee** — A charge (default R350) raised only when a unit is determined to be Not Repairable. Configurable in system settings; waivable per job by admin. Rejected quotes do NOT incur this fee.
- **Main Customer** — The corporation that intermediates between Gentick and the agents. Invites agents into the system and approves all quotes.
- **MVP** — Minimum Viable Product. The smallest version of the RMS that delivers business value.
- **n8n** — Workflow automation platform used as the glue between the RMS, WhatsApp, and email.
- **POPIA** — Protection of Personal Information Act (South Africa).
- **Proforma invoice** — A pre-invoice document used to record the cost of a completed repair; the formal tax invoice is issued from Sage Pastel Online.
- **RMS** — Repair Management System (this product).
- **Shipment** — A physical batch of units received in a single delivery, possibly containing multiple jobs.
- **Stage** — One of the 10 defined points in the repair workflow.
- **VAT zero-rating** — SA tax treatment for exports of goods/services to non-residents, charged at 0% VAT rather than 15%.

## 16. Sign-off

This requirements document v1.0 is the locked baseline for the Gentick RMS build. It serves as the basis for:

- Database schema design with full ERD (Phase 2)
- Architecture diagram (Phase 2)
- UI wireframes (Phase 3)
- API specification (Phase 3)
- Implementation guidance and code scaffolding (Phase 4)

| Role | Name | Signature | Date |
| --- | --- | --- | --- |
| Business owner | Lucia | | |
| Technical lead | | | |
