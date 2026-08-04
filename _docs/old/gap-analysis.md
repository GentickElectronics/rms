# Gentick RMS — GitHub Standards Compliance Gap Analysis

**Standard:** Gentick Engineering Standards — Git & Versioning Standards v1.0 (June 2026)
**Affected deliverables:** Phase 4 (project structure, scaffolding, CI/CD pipeline, deployment runbook)
**Date:** 31 May 2026

---

## 1. Executive summary

The Phase 4 deliverables I produced predate the standard you've now shared. Some elements are already compliant by coincidence (no secrets in git, `.env.example` pattern, shared `gitignore`). Others are partially compliant (CI/CD exists but uses the wrong branching and deploy model). The biggest gaps are structural: the standard mandates a **multi-repo with submodule umbrella**, a **three-branch promotion flow** (`agentic → dev → main`), and a **two-stage deploy** (local → cloud), none of which the Phase 4 pack currently follows.

Total revision effort: about a half-day of focused rework. No requirements/schema/API decisions change — only how the code is organised, branched, versioned, and deployed.

**Before I revise anything, you need to confirm three structural decisions** — they have real cost or workflow implications (see Section 6).

## 2. Compliance scorecard

A row-by-row check of each standard requirement against current state.

| # | Standard requirement | Section | Current state | Action required |
| --- | --- | --- | --- | --- |
| 1 | One repo per component + umbrella with submodules | 2 | ❌ Monorepo `gentick-rms/` | **Decision needed** — split into 3 repos |
| 2 | Component naming `<product>-<component>` | 2.3 | ❌ Single name | Rename per the split decision |
| 3 | Reverse-proxy config in `-infra` | 3 | ⚠ Mixed — Caddyfile lives under `gentick-rms/docker/caddy/` | Move into `gentick-rms-infra` repo |
| 4 | Dockerfiles for a service in the service's own repo | 3.1 | ✓ App Dockerfile lives with the app code | Compliant |
| 5 | Compose files in `-infra` | 3.1 | ❌ docker-compose files in monorepo root | Move to `gentick-rms-infra` |
| 6 | Ingestors / daemons / scripts in `-infra` | 3.1 | ⚠ n8n workflows currently under `gentick-rms/n8n/` | Move to `gentick-rms-infra/n8n/` |
| 7 | Secrets never in git | 3 | ✓ `.env.example` pattern used | Compliant |
| 8 | Three long-lived branches `main` / `dev` / `agentic` | 4 | ❌ CI assumes `main` + feature branches | **Decision needed** + rewrite CI |
| 9 | Promotion via reviewed PR only | 4.2 | ⚠ Branch protection mentioned in CI README but configured for `main` only | Add `dev` protection; document `agentic` flow |
| 10 | Branch protection on `main` and `dev` | 4.3 | ⚠ `main` only in current README | Document `dev` protection |
| 11 | Conventional Commits | 5 | ❌ Not documented or enforced | Add commitlint + docs |
| 12 | SemVer `vA.B.C`, R&D = `v0.x.x` | 6 | ❌ CI tags by git SHA | Rewrite versioning to GitHub Releases |
| 13 | Versions = GitHub Releases | 6.3 | ❌ CI auto-builds on push to main | Build on release publish |
| 14 | Per-component version + umbrella product version | 6.4 | ❌ Single version | Per-repo CHANGELOG + Release flow |
| 15 | `ac/` folder with automated acceptance criteria | 7.1, 10 | ❌ No `ac/` folder; tests are in `tests/` | Add `ac/` per repo with README and automation reference |
| 16 | Four-eyes principle for releases | 7.2 | ❌ CI auto-deploys without separate-reviewer gate | Change CI: build on release; release is published by non-author |
| 17 | `CHANGELOG.md` with running `Unreleased` section | 7.4 | ❌ No CHANGELOG | Add `CHANGELOG.md` per repo |
| 18 | Hotfix path off `main`, then merged back into `dev`/`agentic` | 7.5 | ❌ Not documented | Document in revised CI README |
| 19 | Two-stage deploy: local → cloud, same image promoted | 8.1 | ❌ CI deploys directly from main to production | **Decision needed** — staging server |
| 20 | Build immutable container images, not source | 8.2 | ✓ Multi-stage Dockerfile builds image | Compliant |
| 21 | GHCR registry | 8.3 | ✓ CI pushes to GHCR | Compliant |
| 22 | Image tags = version (no leading `v`); `:latest` is dev-only | 8.4 | ❌ Tags by git SHA | Rewrite tagging |
| 23 | Build automatically, deploy deliberately | 8 | ❌ Deploy auto-runs on push to main | Decouple: auto-build on release; manual scripted deploy |
| 24 | Shell scripts in `-infra` (`build.sh`, `pull.sh`, `deploy-first.sh`, `deploy-fonly.sh`) | 8.5 | ❌ Current model uses GitHub Actions SSH | Add the four scripts to `gentick-rms-infra` |
| 25 | Decouple component version from infra: frontend volume-mapped, API via `.env` | 8.6 | ⚠ Next.js is full-stack so the split is different; API via `.env` is compliant; frontend bundled into app image | Adapt: serve Next.js standalone output via mount, infra serves only |
| 26 | Rollback = re-deploy previous image, not code revert | 8.7 | ✓ `rollback.yml` exists | Compliant (will need rewrite for new deploy model) |
| 27 | Submodule rules: push component before umbrella; pin to releases | 9.3 | ❌ No umbrella exists yet | Will apply once umbrella created |
| 28 | Per-folder READMEs | 10.1 | ⚠ Top-level README only; sub-folders not documented | Add per-folder READMEs |
| 29 | `.gitignore` covers build output, secrets, certs | 10.2 | ✓ Current `.gitignore` is comprehensive | Compliant |
| 30 | Git LFS for large binaries | 10.3 | n/a | No binaries in RMS — not applicable |

**Score: 7 compliant, 5 partial, 18 non-compliant** out of 30.

## 3. The three structural changes (the big ones)

### 3.1 Repository split

The standard mandates one repo per component plus an umbrella. The current monorepo `gentick-rms/` needs to become:

```
gentick-rms-app/         Next.js app (UI + API routes, since Next.js bundles both)
  app/
  components/
  lib/
  db/
  public/
  Dockerfile
  ac/
  CHANGELOG.md
  README.md
  ...

gentick-rms-infra/       Operations and deployment
  compose/
    docker-compose.yml       (dev)
    docker-compose.prod.yml  (production)
    docker-compose.local.yml (staging on local server)
  caddy/
    Caddyfile
  scripts/
    build.sh
    pull.sh
    deploy-first.sh
    deploy-fonly.sh
  n8n/
    workflows/
    whatsapp-templates.md
  ac/
  CHANGELOG.md
  README.md

gentick-rms/             Umbrella (this folder name is intentional — no suffix)
  .gitmodules            (pins gentick-rms-app + gentick-rms-infra at specific releases)
  gentick-rms-app/       (submodule)
  gentick-rms-infra/     (submodule)
  docs/
    architecture.md
    requirements-v1.0.md       (the Requirements doc)
    schema-v1.0.md             (the Schema doc)
    api-v1.0.md                (the API spec)
    wireframes-v1.1.html       (the wireframes)
    deployment-runbook.md
  CHANGELOG.md           (product-level — bumped when the validated combination changes)
  README.md              (system-level architecture, data flow)
```

**Why I'm NOT proposing splitting `app` further into `-api` + `-frontend`:**

The standard's example (`talosot-api` + `talosot-frontend`) assumes those are separate codebases — typically a Go/Laravel API + a Flutter/Angular frontend on different release cadences. Next.js is a full-stack framework: the API routes (`app/api/v1/...`) and the UI pages (`app/(staff)/...`) live in the same codebase, share types, and are built and deployed together. Forcing a split would mean replacing Next.js with something else.

If you ever build a separate mobile app (Flutter) for technicians or agents, that becomes `gentick-rms-mobile` as a new component. The Next.js app stays as one — its "API" surface is consumed by the mobile component just like Flutter would consume any other backend.

**Cost of the split:** half a day of moving files + setting up the umbrella + updating all the documentation references. No code changes. Same total LOC, just in three repos instead of one.

### 3.2 Three-branch promotion model

Currently the CI assumes `main` + feature branches. The standard requires:

- `main` — stable, released from, never pushed to directly
- `dev` — integration branch for human work
- `agentic` — isolated branch for full-AI work; nothing reaches `main` without going through `dev`

**Implication for Gentick RMS:** every repo (`-app`, `-infra`, and the umbrella) gets all three branches with the same protection rules. The CI workflows I built (`ci.yml`, `deploy.yml`, etc.) need rewriting to:

- Run CI on PRs into `dev` (from `agentic` and `feature/*` branches) AND on PRs into `main` (from `dev`)
- Stop auto-deploying on push to `main` — instead, build & push images when a GitHub Release is published
- Add `dev` to branch protection

**Question for you:** will the `agentic` branch actually be used? It only makes sense if your dev does serious AI-pair-programming and wants to keep the autonomous work isolated. If they're using AI as an everyday assistant (Copilot-style), all the AI work lives on `dev` and `agentic` stays empty. I'll keep the branch in the scaffolding either way since the standard mandates it, but the documented workflow shifts.

### 3.3 Two-stage deploy (local → cloud)

Currently I designed CI to auto-deploy to the Hetzner Cape Town production VPS on every push to `main`. The standard requires:

1. CI builds the image and pushes to GHCR (automated).
2. A human runs `deploy-fonly.sh` to deploy that image to the **local server** (a separate, cloud-connected staging environment).
3. A different human runs the automated acceptance test suite on the local server.
4. If everything passes, that same person promotes the **exact same image** to the cloud server with `deploy-fonly.sh` against the prod environment.

**Implication for Gentick RMS:**
- You need a second Hetzner Cape Town VPS as the "local server" (staging). A CX11 (~R80/month, 1 vCPU / 2 GB RAM) is enough since it only handles internal testing traffic.
- OR you can use a developer's machine as the local server during early R&D, and only spin up the second VPS when you go live with real customers.
- The `deploy.yml` GitHub Action becomes a build-and-push workflow only. The actual deploy becomes a shell script someone runs from their laptop.

**Question for you:** are you OK with adding a small staging VPS (~R80/month), or do you want me to design the workflow so the developer's laptop serves as the "local server" until you're closer to launch?

## 4. The medium changes (technical, no decisions needed)

These I can just do — flagging them so you know what's happening:

- **Image tags become version numbers, no `v` prefix.** A release tagged `v0.3.0` produces images tagged `gentick-rms-app:0.3.0`. The `:latest` tag is reserved for the developer's local testing — never deployed to a server.
- **Build trigger changes.** Currently CI builds on every push to `main`. The standard requires CI to build only when a GitHub Release is published. The pre-release auto-builds will still run for `v0.x.x` Pre-releases during R&D.
- **Conventional Commits enforcement.** Add `commitlint` to CI and a `commitlint.config.js`. Commit messages that don't follow the format fail CI.
- **`ac/` folder per repo.** A new folder containing the acceptance test plan, with its own README explaining what the criteria are and the single command that runs them all (`pnpm test:acceptance` or similar).
- **`CHANGELOG.md` per repo** with a running `Unreleased` section. Updated as commits land. Cleared on release.
- **Per-folder READMEs.** Each significant folder (`app/`, `lib/`, `components/`, `db/migrations/`, etc.) gets a short README explaining what lives there.
- **Frontend serving via volume mount.** Next.js standalone output is volume-mounted from the app image into an infra-managed nginx/Caddy container so that infra version doesn't bump on every frontend change. (Note: this is a small architectural shift — currently Next.js serves itself on port 3000. The standard pattern requires nginx/Caddy to serve the built static frontend assets directly while proxying `/api/*` to the Next.js server. We can do this or document why we'd stay with Next.js serving itself — see Section 7.)
- **Shell scripts.** Add `build.sh`, `pull.sh`, `deploy-first.sh`, `deploy-fonly.sh` to `gentick-rms-infra/scripts/`. These become the canonical deploy mechanism; GitHub Actions deploy job is replaced by a scripted local invocation.

## 5. The small changes (purely additive)

- Add `dev` branch protection to the GitHub setup documentation.
- Update the deployment runbook to reference the two-server model and the shell scripts instead of GitHub-Actions-driven deploys.
- Add the standard's submodule rules + the `git config push.recurseSubmodules check` configuration to the contributor docs.
- Add the standard's hotfix flow (off `main` → patch release → merge back into `dev` and `agentic`).
- Replace `pull_request_template.md` references to `main` with the dev/main flow.

## 6. Three decisions needed from you before I revise

1. **Repo split — confirm 3 repos (`-app`, `-infra`, umbrella `gentick-rms`)?** Or do you want me to interpret the standard differently (e.g., keep the umbrella + just split off infra; or split Next.js into `-api` + `-frontend` despite the framework friction)?

2. **Local-server staging — separate Hetzner VPS, or developer laptop?** A second CX11 VPS at ~R80/mo gives you a real cloud-connected staging environment from day one. The alternative is "local server = the dev's laptop running docker-compose" until you're closer to launch — cheaper for the first few months but means staging integrations like Postmark and Meta have to point at the dev's machine over a tunnel.

3. **`agentic` branch — is your developer doing autonomous AI development that warrants isolation?** Always-on AI assistants (Copilot, Cursor) live happily on `dev`. The `agentic` branch is for "I gave the AI a big task and let it run for hours" workflows. Either way it'll exist (the standard requires it) — the question is whether anyone will actually use it.

## 7. Open architectural question (lower priority — for your dev to weigh in on)

The standard's "frontend volume-mapped, served from nginx" pattern was designed for an SPA + REST API split. Next.js full-stack has two ways to do this:

**Option A — Next.js serves itself.** Caddy reverse-proxies everything to the Next.js container on port 3000. Standalone Next.js handles its own static asset serving and routing. Simpler, matches current scaffolding. Slightly bumps `gentick-rms-infra` on app structural changes (rare).

**Option B — Caddy serves the static frontend; proxies `/api/*` to Next.js.** Closer to the standard's intent. Requires exporting Next.js as a static build PLUS a server build, mounting the static into Caddy. More complex. Truly decouples infra from frontend version.

Recommendation: **Option A** for MVP since Gentick's RMS is one Next.js app, not the firmware/API/frontend product the standard imagines. Document the deviation in the umbrella repo's architecture docs as "this is a full-stack framework so the volume-mapped frontend rule applies differently". Your dev can revisit if/when the mobile app is added.

## 8. Suggested revision plan

Once you've answered Section 6, I propose:

**Step 1 — Restructure (45 min):** Split `gentick-rms/` into the three repos. Move files. Update internal path references. Add submodule wiring on the umbrella.

**Step 2 — Add the missing repo hygiene (45 min):** `CHANGELOG.md`, `ac/` folder, per-folder READMEs, `commitlint` config. Same pattern in each repo.

**Step 3 — Rewrite branching docs (15 min):** Update README + the new `CONTRIBUTING.md` to describe the three-branch flow, the four-eyes release rule, the hotfix path. Document branch protection settings.

**Step 4 — Rewrite CI/CD (60 min):** New workflow files split by concern:
- `gentick-rms-app/.github/workflows/ci.yml` — runs on PRs to `dev` and `main`. Lint, typecheck, test.
- `gentick-rms-app/.github/workflows/build-on-release.yml` — runs when a GitHub Release is published. Builds image tagged with the version, pushes to GHCR.
- `gentick-rms-infra/.github/workflows/ci.yml` — lint shell scripts, validate Compose files.
- `gentick-rms-infra/scripts/{build,pull,deploy-first,deploy-fonly}.sh` — the actual deploy mechanism.

**Step 5 — Update the deployment runbook (30 min):** Reflect the two-server model, the scripted deploy, the four-eyes release flow.

**Step 6 — Verify (15 min):** Cross-check the standard's cheat sheet against the revised pack item-by-item.

**Total revision effort: ~3.5 hours of focused work.** Producible in one continuous session once you've answered Section 6.

---

## Appendix: standard compliance, side-by-side reference

| Standard section | Where it's now satisfied | Reference doc / file |
| --- | --- | --- |
| §2 Repository structure | After Step 1 of revision | New repo layout |
| §3 What goes in which repo | After Step 1 | `gentick-rms-infra/` |
| §4 Branching strategy | After Step 3 | `CONTRIBUTING.md` |
| §5 Commit conventions | After Step 4 (commitlint) | `commitlint.config.js`, README |
| §6 Versioning | After Step 4 | `CHANGELOG.md`, GitHub Releases |
| §7 Testing & release | After Step 2 + Step 3 | `ac/README.md`, `CONTRIBUTING.md` |
| §8 Deployment & infrastructure | After Step 4 + Step 5 | `gentick-rms-infra/scripts/`, runbook |
| §9 Submodules | After Step 1 + Step 3 | Umbrella README |
| §10 Repository hygiene | After Step 2 | Per-folder READMEs, `.gitignore` |
| §11 Cheat sheet | Already covered by above | All of the above |
