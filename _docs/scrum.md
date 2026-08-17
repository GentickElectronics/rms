# RMS — Active Scrum

**No active scrum.** The n8n-decoupling scrum is **complete (2026-08-17)**.

**Done:** outbound email, the daily-summary cron, and webhook retry/dead-letter
now live in the Go API's new `internal/worker` package; n8n is parked behind a
`whatsapp` compose profile (WhatsApp-only, not yet built). This closes finding
I1 — a silently-dead n8n workflow can no longer take the whole notification path
down with it. Committed on `agentic`: rms-api `2ba1159` (N-1..N-4), rms-infra
`cad1b71` (N-5/N-6). **Not deployed** until merged to `main` + server pull, and
the new env (`ESCALATION_EMAIL`, `MORNING_SUMMARY_RECIPIENTS`, …) is filled in.

Ticket-by-ticket outcome, and the handful of default choices worth a look
(SMTP-not-Postmark, quote.approved/rejected now email the customer, 7-day aging
boundary, plain-text mail): [`migration/plan.md`](./migration/plan.md) Part 2.

**Candidate next tracks** (not started): the live defects **F8** (credential-hash
leak in quote responses) and **F2** (one-primary invariant), each with a Go
regression test; then the WhatsApp build itself, which carries F7/F9.

---

_Prior scrums, both complete: the Go-rewrite parity remediation (sprints 0–6),
outcome [`migration/result.md`](./migration/result.md); and n8n decoupling
(N-1…N-6), outcome in [`migration/plan.md`](./migration/plan.md) Part 2._
