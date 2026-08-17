# RMS — Testing Procedure

How to run a full test cycle. Read [`README.md`](./README.md) first for what's
covered where. The short version: **the Go suite covers the backend; a small
manual checklist covers the rest.**

## 1. Backend regression — the Go suite (do this first, every time)

Against real Postgres, ~263 tests:

```bash
agollum/testing/golang.sh --app C:/dev/rms/rms-api --infra C:/dev/rms/rms-infra --defaults
```

Green here means the backend-behavioural majority of the register holds. A
failure is a real backend regression — fix before going further. Load the
`agollum` skill for the script's flags. This needs no deploy; it stands up its
own throwaway test stack.

## 2. Deploy (only for an end-to-end / release cycle)

Build + deploy `agentic` to `gentickit.local` (see the `dev-environment` skill
and `migration/result.md` for the exact steps already used once). Skip this step
if you only need the backend regression check.

## 3. Manual / UI cases — the checklist

Work through [`manual-checklist.md`](./manual-checklist.md): the 50 cases a Go
test can't see (SPA rendering, infra, external systems). Each row has a condensed
how-to-run + pass criteria. Drive them by hand or with a browser-driving agent.
**Route this to a Sonnet/runner agent, not Opus** — it's execution, not
reasoning; reserve an Opus pass for judging the findings.

## 4. Record the results

Create `results/<yyyy-mm-dd>.md` for the run. Capture, per case, the actual
verdict (PASS / FAIL / BLOCKED / N/A), the evidence, and — for a FAIL — actual
vs expected. Keep a findings section for genuine defects, separated from
already-known / never-in-scope gaps. The 2026-08-14 run is the template.

## 5. Before filing a defect

Check the latest `results/*.md` findings section and `migration/audit.md` — a
manual case failing may be a real regression **or** a known, logged gap. Don't
re-file something already on record.

## Efficiency notes (learned 2026-08-14)

- The Go suite is the cheap deterministic check; don't re-verify Go-covered cases
  by hand (~zero tokens vs a fleet of agents).
- When using agents: **Sonnet/runner** for execution, one **Opus/pegasus** pass to
  judge findings. Give each agent a scratchpad for working notes and a single
  shared deliverable, and tell it to **save incrementally**.
- Namespace any test data each agent creates (`RMSTEST<batch>`), and let only one
  agent drive the shared browser at a time.
