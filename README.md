# rms

Umbrella repo for the Gentick Electronics Repair Management System — app and infra submodules plus product and design docs

## Structure

- `rms-app` - Next.js 15 application, submodule
- `rms-infra` - deployment and ops, submodule
- `_docs/` - product and design documentation

## Documentation

- `_docs/requirements.md` - product requirements, v1.1
- `_docs/api-spec.md` and `_docs/openapi.yaml` - API specification
- `_docs/schema.md`, `_docs/erd.svg`, `_docs/erd.mmd`, `_docs/erd.png` - database design
- `_docs/wireframes.html` - UI wireframes, v1.1
- `_docs/deployment-runbook.md` - deployment runbook
- `_docs/gap-analysis.md` - GitHub standards gap analysis
- `_docs/migrate_plan.md` and `_docs/migration_diffs.md` - migration notes

## Cloning

    git clone --recurse-submodules https://github.com/GentickElectronics/rms.git
