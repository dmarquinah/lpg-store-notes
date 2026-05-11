# Frontend Bootstrap — lpg-store

---
project: lpg-store
domain: specs
category: frontend-bootstrap
last-updated: 2026-05-08
---

## Context Documents

Read these before adding frontend-bootstrap specs:

- [[../../eng/architecture]] — Frontend section (v2 layout, current state)
- [[../../eng/frontend-bloat-analysis]] — diagnosis of v1 inflation drivers
- [[../../eng/patterns/frontend-module-template]] — vertical-module convention for new feature ports
- [[../../product/overview]] — what features are MVP vs deferred
- [[../../eng/decisions]] — backend ADRs that frontend types must follow (ADR-005 signed-delta API, ADR-008 three-state inventory workflow)
- [[../bootstrap/v2-skeleton]] — backend reset playbook (`legacy/` archive + clean reset), used as the model for the frontend reset

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[v2-skeleton]] | done | Archived v1 frontend, bootstrapped v2 (Vue 3 + Vite + Pinia, module-by-domain, shadcn-vue, drop i18n + FCM + datepicker). 38 KB gzipped main bundle, login flow live, 3 role-stub homes. |

## Out of scope for this category

Module ports (auth UI features, users UI, inventory UI, orders UI, etc.) are **not** bootstrap specs. Each feature module gets its own spec under `specs/<feature>/` once the backend module is ready.
