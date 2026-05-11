---
last-updated: '"2026-05-08"'
---
# lpg-store

---
project: lpg-store
status: active
last-updated: 2026-05-08
---

## Description

Backend, frontend dashboard, and chat bot for a small-city LPG (Liquefied Petroleum Gas) tank delivery business. The system tracks the chain of custody from inventory assignment to order fulfillment: which delivery user has which tanks each day, what was sold to whom, when each delivery happened, and how unpaid orders accumulate. **Primary value: traceability**, not analytics or accounting.

The product is pre-production. The backend is currently being rebuilt as v2 (clean reset, with v1 archived under `legacy/`). The frontend and bot are in earlier stages.

## Repositories

| Repo | Path | Stack | Role |
|------|------|-------|------|
| lpg-backend | /home/diegomh/dev/personal/freelance/lpg-store/lpg-backend | Node 22 / Express 5 / Drizzle / Postgres / Redis / TypeScript | backend (v2 in progress) |
| lpg-frontend-vue | /home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue | Vue 3 / Vite / Tailwind / Chart.js / Leaflet / PWA | dashboard frontend |
| lpg-bot | /home/diegomh/dev/personal/freelance/lpg-store/lpg-bot | Node / Express / axios / commander | chat bot |

## Domains

- [[eng/index]] — architecture, decisions, code patterns
- [[product/index]] — what the product does, user roles, requirements
- [[ops/index]] — deployment, CI/CD, runbooks
- [[specs/index]] — feature specs (draft, approved, in-progress, done)

## Active Work

| Status | Spec | Summary | Repos Affected |
|--------|------|---------|----------------|
| _none_ | — | — | — |

## Recent Changes
| Date | Domain | Summary |
|------|--------|---------|
| 2026-05-08 | specs/frontend-bootstrap | `v2-skeleton` done — v1 archived to `legacy/` (230 files via git mv), fresh src/ with module-by-domain layout, shadcn-vue UI library, single-guard router, auth vertical slice + role-stub homes. 38 KB gzipped main bundle. |
| 2026-05-08 | eng + specs | Frontend documented in vault: `architecture.md` Frontend section filled in, `eng/frontend-bloat-analysis.md` added, `specs/frontend-bootstrap/v2-skeleton` drafted then implemented |
| 2026-05-08 | specs/auth | `auth-foundation` done — login, JWT, role middleware, invitations, BOOTSTRAP_TOKEN-gated developer creation, Redis logout blocklist (32 tests passing) |
| 2026-05-07 | eng | Vault initialized; backend v2 skeleton in place; v1 archived to `legacy/` |
