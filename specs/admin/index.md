# Admin & Organization — lpg-store

---
project: lpg-store
domain: specs
category: admin
last-updated: 2026-06-15
---

## Context Documents

Read these before working on admin/org specs:

- [[../../eng/design-system]] — Piloto UI conventions (PageHeader, tokens, dialogs, badges); all UI must comply
- [[../../eng/patterns/frontend-module-template]] — vertical module folder convention (the new `organization` module)
- [[../../eng/architecture]] — backend + frontend layout
- [[../../product/overview]] — roles (admin/operator/delivery), store + assignment model
- [[../stores/store-management/index]] — stores + assignment write surface (reused here)
- [[../users/users-crud/index]] — user list/edit (reused here)
- [[../auth/auth-foundation/index]] — invitations (`POST /auth/register` + `/auth/invite/:token`) — the invite/create-user mechanism

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[org-management/index\|org-management]] | done | Unified admin **Organización** view: a location-centric board of stores with their assigned employees, plus store CRUD, assignment management, user edit, and user invite/create — replacing the dispersed `/usuarios` + Catálogo *Tiendas*/*Asignaciones* surfaces. |

## Notes

This category holds cross-cutting **admin management** surfaces that span multiple
domain modules (users + catalog/stores + auth invitations). The first spec
consolidates the scattered admin management screens into one overview so an admin
can see every location and who's assigned where, with few clicks.
