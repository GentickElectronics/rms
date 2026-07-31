# Changelog — rms (umbrella)

## [Unreleased]

### Added
- `_docs/migration_diffs.md` — structural mapping from RepairManagementSystem → rms.
- `_docs/migrate_plan.md` — deployment plan for gentick-infra integration.
- `rms-app` and `rms-infra` registered as git submodules with `.gitmodules`.

### Changed
- Repo renamed from RepairManagementSystem to rms.
- Architecture shifted from standalone stack to gentick-infra integration (shared Postgres, nginx, Cloudflare Tunnel).
