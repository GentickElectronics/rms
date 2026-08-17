# RMS — Manual Test Checklist

The Go API test suite (263 tests, run via `agollum/testing/golang.sh` against
real Postgres) now deterministically covers the backend-behavioural majority
of the 145-case test register — see [`README.md`](./README.md) for how to run
it and the full coverage breakdown. This document is only the remainder: the
**50 cases that a Go test cannot verify**, because the thing being checked is
what the SPA renders, an infra/external system's actual behaviour, or a
process/compliance fact. Run these by hand (or via a browser-driving agent)
after the Go suite is green.

Three further cases (TC-AUTH-08, TC-INT-14, TC-INT-23) are excluded entirely —
out of scope or N/A by design; see the README for why.

Source detail (full preconditions/steps) for every case lives in
`_docs/old/test-register.json`. "How to run" below is condensed from that;
you shouldn't need to open it to execute a case.

**Total: 50 cases.**

---

## AUTH/RBAC (4)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-AUTH-02 | Technician sees only technician-scoped navigation | Log in as a Technician; inspect the landing page and sidebar; try navigating directly to `/dashboard` and `/reports`. | Lands on `/queue` ("My queue"); sidebar under Internal-Staff shows only Technician queue (no Admin dashboard/New intake/Reports); direct nav to `/dashboard` and `/reports` redirects to `/queue`. | Sidebar visibility and route redirects are SPA (rms-frontend) rendering/routing. `authz_matrix_test.go` proves the backend role gate, not what the frontend nav shows or blocks. |
| TC-AUTH-03 | Front Desk can perform intake and dispatch | Log in as Front Desk; open New intake and complete required fields; open Customers and manage a record; open Agents and add an agent; try navigating directly to `/settings`, `/dashboard`, `/reports`, `/queue`. | Front Desk creates intake jobs, manages customers, adds agents via the UI; sidebar shows only New intake/Customers/Agents; lands on New intake; the four staff-only routes hide/redirect to New intake. | Screen access, nav scoping and route redirects are SPA behaviour; `authz_matrix_test.go` covers the API-level role gate only. |
| TC-AUTH-07 | Agent self-service password reset via email link | On login, click "Forgot password"; enter the agent's email; open the reset email (dev: Mailpit `:54324`; staging: configured SMTP inbox); follow the link, set a new password, sign in. | Generic "if an account exists" confirmation (no enumeration); reset email actually arrives with a working link; new-password page works; recovery session signed out, redirected to login with confirmation; new password logs in. | Requires reading a real inbox and following an emailed link — an end-to-end mail-delivery check outside what a Postgres-backed Go test can assert. |
| TC-AUTH-10 | Direct URL access to another role's page is blocked | Log in as an Agent; enter a staff-only URL (e.g. `/settings`) directly in the address bar. Also fire the equivalent request as a raw API call with the agent's bearer token (curl/Postman). | Browser redirects to the agent portal, no staff data exposed in the DOM. The raw API call also refuses (403/redirect) — don't rely on the SPA guard alone. | The frontend route guard/redirect is SPA behaviour; a browser-only check doesn't prove the backend refuses the equivalent direct API call (the API gate is Go-tested separately), hence the added curl step. |

## INTAKE (14)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-INT-06 | Opt-out toggle immediately revokes WhatsApp consent and logs it | Open a contact/agent profile with active WhatsApp consent; switch the WhatsApp opt-out toggle on; save; inspect the consent/audit log. | Consent revoked immediately; revocation event with timestamp logged; no further WhatsApp messages sent to that party. | Feature not implemented (no opt-out toggle, no revocation write path exists) — nothing for a Go test to assert. Re-check when consent work resumes. |
| TC-INT-07 | Annual consent audit flags consents older than 12 months | Seed a consent record dated >12 months ago; run/open the annual consent audit workflow (app screen or n8n). | Aged consent flagged for renewal; consents under 12 months not flagged. | No annual consent audit workflow exists in the app or n8n — a process/feature gap, not a backend assertion. |
| TC-INT-16 | Capture all per-unit intake fields | On New intake, enter unit type, serial, fault description, accessories shipped; attach an intake photo under "Photos of condition on arrival"; save and receive. | All per-unit fields persist and are visible on the job, including the intake photo(s) captioned as condition-on-arrival. | Condition-on-arrival photo capture at intake is a form/UI + upload interaction; Go coverage (`photo_upload_test.go`, `files_test.go`) tests upload/serving mechanics generically, not this specific intake-form flow. |
| TC-INT-25 | Extra units discovered - walk-in routing by identified owner | Ensure the agent has no pending pre-registrations; as Front Desk, create jobs at intake (origin = Walk-in), identifying the owning agent; confirm the no-extra-units checkbox; save; inspect `webhook_events`. | Jobs created; `job.received` payload carries the identified agent + main customer/primary contact (correct owner routing). | Requires driving the staff intake UI end-to-end and inspecting the resulting webhook payload in context — a UI walkthrough, not a unit-level backend assertion. |
| TC-INT-26 | Confirm-no-extra-units checkbox required before receive | On New intake Step 3, leave the "no extra unannounced units" confirmation unticked; attempt "Receive shipment & notify". | Receiving blocked with an inline error until ticked; instruction reminds the user to add extras above first. | Client-side UI gate (button state, inline error). `intake_receive_gate_test.go` (T-52) covers the server-side backstop, not the UI's own blocking. |
| TC-INT-27 | Save draft preserves an in-progress intake | Start an intake, partially fill it in, click "Save draft"; navigate away and return. | Draft retained with previously entered data; no job numbers issued until receive. | Feature not implemented (no "Save draft" control or draft persistence anywhere) — nothing for a Go test to assert. |
| TC-INT-28 | Add an additional (non-primary) contact to a corporation | As Admin/Front Desk, Customers -> "Add another contact"; select the corporation; enter contact person + email, optionally phone + temp password; leave "Mark as primary" unchecked; save. | Contact created, listed without a Primary badge, contact count increments; if a temp password was given, the new contact can log in to the main-customer portal. | Pass criterion spans a UI form flow plus an actual portal login attempt with the provisioned password — an end-to-end UI+auth walkthrough. |
| TC-INT-29 | 'Add another contact' rejects missing customer and invalid email/phone | "Add another contact", click Add with no customer selected; then select a customer, enter "notanemail"/"abc"; attempt save. | "Select a customer" shown when none chosen; inline errors on malformed email/phone, all surfaced together; no contact created. | Verifies inline client-side error rendering (all errors at once, no server round-trip) — UI behaviour, not backend response content. |
| TC-INT-30 | Add agent (staff) surfaces all invalid/missing fields at once; no record saved | Agents screen, click "Add agent" with all fields blank; then enter "notanemail", "abc" phone, a 3-char password, blank Address/City; click Add agent again. | Step 1: inline errors on Main customer/Agent name/Email/Address/City all together. Step 2: inline errors for email, phone (E.164), password (min 8) all together. No record saved either step. | Verifies simultaneous inline UI validation rendering, not backend response content. |
| TC-INT-31 | Portal '+ Add an agent' surfaces all invalid/missing fields at once; no record saved | Log in as a primary main-customer contact; "Your agents" -> "+ Add an agent"; click Add with all fields blank; then enter invalid email/phone and a short password; click Add again. | All missing/invalid fields flagged simultaneously inline; no agent record created. | Portal-side inline validation UI, mirrors INT-30 in the customer portal; not a backend assertion. |
| TC-INT-32 | Agent 'Book in a repair' flags all missing unit fields on all units; blank units not silently dropped | As an Agent, open "Book in a repair" with 2+ unit rows; complete Unit 1, leave Unit 2 blank; click "Book in repair"/"Confirm & notify"; observe validation; fix/remove Unit 2, resubmit. | Submission blocked with ALL missing fields flagged inline on every incomplete unit; blank unit not silently dropped; after fixing, each unit gets its own job. | Multi-unit agent book-in is a frontend flow (dropped/not restored server-side per the audit) — inline per-row validation rendering, not a backend assertion. |
| TC-INT-33 | Staff New intake flags all missing unit fields and agent selection | Admin/Front Desk, New intake with two unit rows (one complete, one blank), no agent selected; click "Confirm & notify"; observe validation; select agent, fix/remove Unit 2, resubmit. | Blocked; agent field shows inline "select the agent" error AND all missing unit fields flagged on every incomplete unit at once. After fixing, shipment saves, each unit gets its own job. | Same inline multi-field UI validation pattern as INT-29/30/32 — verifies rendering, not backend behaviour. |
| TC-INT-34 | Edit contact details and reassign primary via Contacts table | Customers -> Contacts; edit a non-primary contact's name/phone, save; edit again, tick "Primary contact", save; edit the new primary and inspect the Primary checkbox; attempt invalid input (blank name, malformed phone) and save. | Edits persist (login email not editable); promoting demotes the previous primary in the same transaction (exactly one Primary badge); current primary's checkbox disabled with a hint; invalid input flagged inline all at once, no server call. | Combines a UI edit flow, inline validation rendering, and a UI-only disabled-checkbox affordance — not exercised by the Go suite, which drives the API directly. |
| TC-INT-35 | Remove (soft-delete) a contact; primary contact cannot be removed | Customers -> Contacts; Remove on a non-primary contact; confirm via the inline "Confirm remove" prompt; inspect the Contacts list/count; inspect the Remove button on the current primary; check `deleted_at` in the DB. | Inline two-step confirmation (no browser dialog); contact soft-deleted (list/count update, `deleted_at` set, row retained, portal login stops resolving); primary's Remove button disabled with a hint. | UI confirmation-flow and disabled-button affordance are frontend behaviour; the one-primary invariant is DB-enforced but this case's pass criteria centre on the UI interaction. |

## WORKFLOW (2)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-WF-13 | Stage timeline highlights done/current/future stages | Open Job detail for a job in Quoted stage; inspect the stage timeline pips. | Completed stages show "done", current stage highlighted "now", later stages "future". | Pure visual/CSS state rendering in the SPA — a Go backend test has no notion of pip colour or highlight state. |
| TC-WF-14 | Print jobcard from Job detail | Open Job detail; click "Print jobcard". | A single-page A4 jobcard renders with the Gentick header sourced from company-details settings (not hardcoded). | Print layout and settings-sourced header rendering are frontend/CSS concerns; print itself works but the settings-sourced header currently doesn't — needs a human to inspect the printed output. |

## COMMS (2)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-COM-02 | WhatsApp primary when consent given; email always sent | With a party who has WhatsApp consent, trigger an "item received" event (e.g. via intake) and check outgoing messages. | WhatsApp message sent (primary) and email also sent as fallback/record. | Blocked in this environment: no consent-state schema to branch on (D12) and delivery routing happens inside a live n8n workflow — the Go suite (Postgres-only, no n8n/WhatsApp sandbox) can't reach it. Needs n8n imported/active and a Meta sandbox number. |
| TC-COM-03 | Email-only when WhatsApp consent is absent | With a party who has opted out of WhatsApp, trigger a milestone event and check outgoing messages. | Only email is sent; no WhatsApp message reaches the opted-out party. | Same as COM-02 — consent-state + live n8n routing, outside the Go suite's reach. |

## QUOTING (2)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-QUO-04 | Agent sees full quote details (incl. R-value) after approval | Get a quote approved by the main customer; log in as the agent and open the job. | Agent now sees the full quote including price/R-value. | Price-gating/redaction is rendered client-side in the agent portal; the money math is Go-tested (`money_test.go`, `decimal_test.go`) but what the SPA actually displays to the agent is not. |
| TC-QUO-05 | Agent does NOT see price before approval | With a quoted job not yet approved, log in as the agent and open the job. | Agent sees "Quoted/awaiting approval" status but no price is exposed. | Same as QUO-04 — verifies what the SPA renders/withholds, not backend data. |

## REPORTING (2)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-REP-01 | Active jobs dashboard shows per-stage counts and aging | Open the Admin dashboard with jobs distributed across stages. | KPI tiles show active jobs, aged >14 days, avg turnaround, revenue; stage strip shows per-stage counts; aged jobs visually highlighted. | The underlying counts are mostly API-verifiable, but the aged->14-day KPI tile and its visual highlight live only in the SPA — `/dashboard/kpis` has no aged-count field to assert against. |
| TC-REP-08 | Today's priorities surfaces approvals, boxed units, and Sage batch | With pending approvals, boxed units, and a ready Sage batch, open the Admin dashboard. | "Today's priorities" lists quotes awaiting approval, boxed units awaiting collection notification, and the Sage export ready for download. | A composite UI panel assembled from several backend pieces; no single endpoint to assert against — needs a human to check the panel renders and links correctly. |

## ADMIN (1)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-ADM-04 | Audit log viewer shows consent, logins, role and data changes | Generate activity (a consent change, a login, a role change, a data edit); open the Admin audit log viewer. | All four event categories are visible in the viewer with actor and timestamp. | About what the viewer screen renders, not just whether `audit_events` rows exist (`audit_writer_test.go` covers the writer). Known gap: login and consent events are never written, so 2 of 4 categories are structurally missing — re-check once that's fixed. |

## UI/WIREFRAME (10)

No case in this category has any Go coverage — every one is a rendered SPA behaviour.

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-UI-01 | Technician 'Today's focus' shows prioritised cards | Log in as Technician with an aged job and parts arrivals; open "My queue". | Focus cards show e.g. an aged job to escalate, a parts-arrived resume prompt, and a ready-for-QC card, each with a primary action. | Prioritised card layout/content is SPA presentation logic. |
| TC-UI-02 | Technician full queue sorting | As a Technician with 12 jobs, sort the queue by oldest first, by stage, and by customer. | Queue reorders correctly for each sort option. | Client-side sort/UI interaction. Known gap: no sort controls currently exist. |
| TC-UI-03 | Technician quick-log note/part/photo against active job | As Technician working a job, use Quick log to select the job, add a note, a part, and a photo, then save. | Note, part, and photo all recorded against the selected job. | A composite UI workflow across three input types in one panel; each underlying write is separately Go-tested, but the panel's combined UX is not. |
| TC-UI-04 | Agent portal KPIs reflect agent's own jobs | As an Agent with 3 active / 1 awaiting approval / 1 ready-for-collection job, open the agent portal. | KPI tiles match the agent's actual job counts and statuses. | KPI tile rendering in the SPA. |
| TC-UI-05 | Agent 'My units' pre-fills a new repair request | As an Agent with prior units, in "My units" pick a known serial to start a new repair request. | New repair form pre-fills that unit's type/serial and prior context. | Form pre-fill is frontend behaviour. Known gap: no My-units prefill currently exists. |
| TC-UI-06 | Agent active/closed repairs toggle | As an Agent with active and closed jobs, toggle "Active only" vs "Include closed". | List updates correctly for each toggle state. | Client-side filter toggle. Known gap: no toggle exists (stage dropdown only). |
| TC-UI-07 | Main customer 'Action required' approve/reject controls | As a Main Customer Contact with 3 quotes awaiting approval, use "View quote PDF", "Approve", and "Reject" on a quote in the portal. | PDF opens; Approve transitions the job to Approved; Reject moves it to Rejected with no fee. | The approve/reject actions themselves are Go-tested (quote transition side-effects), but this verifies the portal's UI controls are actually wired up and reachable — a known prior gap (W5) was exactly that nothing linked to them. |
| TC-UI-08 | Main customer spend-this-quarter and CSV export | As a Main Customer Contact with invoiced jobs, open the portal and click "Export spend report (CSV)". | Spend summary shown; CSV export downloads the spend data. | Portal widget rendering + browser download. Known gap: no portal spend widget/CSV currently exists. |
| TC-UI-09 | Main customer all-active-jobs agent filter | As a Main Customer Contact with jobs across several agents, filter the all-active-jobs table by a single agent. | Only that agent's jobs are listed. | Client-side table filter UI. Known gap: no agent filter exists (stage filter only). |
| TC-UI-10 | Branding/header renders Gentick identity | Load any portal page and inspect the header/sidebar. | Gentick Electronics branding and "Experts in Telemetry Systems" tagline render correctly. | Purely visual branding check. |

## NFR (12)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-NFR-01 | Staff page loads under 2s on 4G; portal under 3s | On a throttled 4G connection (browser devtools throttling), load the staff dashboard and customer portal pages; measure load time. | Staff pages < 2s; portal pages < 3s. | Real-world page-load timing under network throttling; backend response time alone (<300ms) doesn't capture SPA bundle/render time. |
| TC-NFR-04 | Secrets are not exposed in code or workflow JSON | Inspect rms-api/rms-frontend/rms-infra source and exported n8n workflow JSON for hardcoded credentials. | No secrets found in code or workflow JSON; credentials live in n8n's encrypted store / env secrets. | A manual code/config audit across infra repos, not an assertion the app makes about itself at runtime. |
| TC-NFR-05 | POPIA data residency - no cross-border transfer | Verify the DB/file-storage hosting region and the backup region for the production deployment. | Data and backups stay within the agreed region; no cross-border transfer. | Infrastructure/hosting fact, not application behaviour. |
| TC-NFR-06 | Subject access request answerable within 30 days | Attempt to export all personal data held for a given contact/agent. | A complete personal-data export can be produced within 30 days of a request. | Process/compliance capability; no personal-data export feature currently exists to test. |
| TC-NFR-07 | Customer portal is mobile-responsive | Resize the browser (or device emulation) to a phone-width viewport (<=1100px); open the agent portal and attempt to book a repair. | Layout reflows to a single column; intake is fully usable on mobile. | Responsive layout is a visual/CSS check requiring a human or visual regression tool. |
| TC-NFR-08 | Supported browsers render correctly | Open key screens in the latest 2 versions of Chrome, Edge, Firefox, and Safari. | Screens render and function correctly with no browser-specific breakage. | Cross-browser rendering requires an actual browser matrix, not something a Go backend test can assert. |
| TC-NFR-09 | Customer portal meets WCAG 2.1 AA basics | Run an accessibility audit (e.g. axe, Lighthouse) against customer portal pages: contrast, form labels, keyboard nav, focus order. | Portal passes WCAG 2.1 AA checks for contrast, labels, and keyboard nav. | Rendered contrast/keyboard-nav/focus order are visual and interaction properties of the SPA; needs an accessibility tool or human review. *(Flagged for review — see report; this was tested via "API" layer in the last run, which is unusual for a visual check.)* |
| TC-NFR-10 | Daily backups exist and are restorable | Verify a recent daily DB + file backup exists; perform a test restore. | Daily backups exist (30-day retention, monthly snapshots 12 months); a restore succeeds. | Infrastructure/ops procedure, not app behaviour. |
| TC-NFR-11 | Automation failures raise error alerts to admin | Force an n8n workflow to fail (e.g. a bad WhatsApp send); check for an admin error alert. | The n8n error workflow captures the failure and emails an alert to admin. | Requires a live n8n instance and its error-workflow config; not reachable by a Postgres-only Go test. |
| TC-NFR-12 | Availability target and maintenance notice | Review uptime monitoring (e.g. UptimeRobot) over a measured month; verify planned maintenance windows are announced 48h ahead. | Monthly uptime >= 99%; maintenance windows announced >=48h in advance. | Ops/process metric measured over real time, not a single test run. |
| TC-NFR-13 | Scalability headroom for 10x growth | Load-test the system with ~10x expected monthly job volume; measure response times and resource use. | System handles 10x volume without architectural change and within acceptable performance. | A dedicated load-test exercise, not a functional regression check. |
| TC-NFR-14 | Maintainability standards in place | Review the codebase and migration history: confirm documented conventions (README) and that DB schema changes are tracked as migrations. | Conventions documented; schema changes versioned via migrations. Note: reword the stale "TypeScript throughout" step — the backend is now Go, frontend is TypeScript. | A repo/process review, not a runtime assertion. |

## INTEGRATIONS (1)

| Case | Title | How to run | Pass criteria | Why not automatable |
|---|---|---|---|---|
| TC-INTG-05 | Transactional emails authenticated (SPF/DKIM/DMARC) | Send a jobcard/quote email from the production mail path and inspect headers (e.g. via mail-tester.com or the receiving mailbox's raw headers). | Email passes SPF, DKIM, and DMARC; from-address is `repairs@gentick.co.za` (or the configured production sender). | DNS/mail-authentication verification against a real mail provider — the dev environment uses Mailpit, which doesn't perform real SPF/DKIM/DMARC checks. |

---

**Totals: 50 manual cases** across all 10 categories — AUTH 4, INTAKE 14, WORKFLOW 2,
COMMS 2, QUOTING 2, REPORTING 2, ADMIN 1, UI/WIREFRAME 10, NFR 12, INTEGRATIONS 1.
The other 92 cases in the 145-case register are already covered by the Go suite
and 3 more are excluded as N/A/out-of-scope — see [`README.md`](./README.md) for
the full per-category breakdown and the covering Go test files.
