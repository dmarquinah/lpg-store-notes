# Auth — lpg-store

---
project: lpg-store
domain: specs
category: auth
last-updated: 2026-05-08
---

## Context Documents

Read before working on auth specs:

- [[../../eng/architecture]] — backend v2 architecture (modules, repository pattern, error handling)
- [[../../eng/patterns/module-template]] — vertical module folder convention
- [[../../product/overview]] — user roles (operator, delivery, admin) and daily workflow

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[auth-foundation]] | done | Login + JWT + role-based authorization. Single users table, delivery_profile sub-table, admin-issued invite tokens, Redis logout blocklist, BOOTSTRAP_TOKEN-gated developer creation. |
