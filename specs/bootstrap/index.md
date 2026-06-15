---
last-updated: 2026-06-04
---
# Bootstrap — lpg-store

---
project: lpg-store
domain: specs
category: bootstrap
last-updated: 2026-05-07
---

## Context Documents

Read these before adding bootstrap specs:

- [[../../eng/architecture]] — backend + frontend v2 architecture overview
- [[../../eng/decisions]] — ADRs
- [[../../eng/legacy-bloat-analysis]] — backend v1 inflation drivers the reset avoids
- [[../../eng/frontend-bloat-analysis]] — frontend v1 inflation drivers the reset avoids
- [[../../eng/patterns/module-template]] / [[../../eng/patterns/frontend-module-template]] — vertical-module conventions per repo

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[v2-skeleton/index\|v2-skeleton]] | done · backend ✓ frontend ✓ | Reset both repos to clean v2 skeletons. Backend: Express + Drizzle + Docker + GHCR + VPS pipeline. Frontend: Vue 3 + Vite + Pinia + shadcn-vue, module-by-domain, 38 KB gzipped. |
