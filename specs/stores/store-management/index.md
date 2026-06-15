---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../stores-and-catalog/index]]"
  - "[[../../users/users-crud/index]]"
  - "[[../../orders/orders-multi-location/index]]"
last-updated: 2026-06-14
---

# Spec: Store & Assignment Management (admin write surface)

## Problem Statement

`stores` and `store_assignments` are **seed-only** today — there's no API or UI to
create a store or to assign a user to one. The catalog "Tiendas" tab is read-only,
and store↔user assignments exist only in `db:seed`.

This blocks exercising the multi-location work shipped in
[[../../orders/orders-multi-location/index|orders-multi-location]]:

- **Transfer-to-branch** and the **admin store switcher** need a *second store* to
  exist — impossible to create from the UI.
- **Operator branch-scoping** (an operator sees only its store's queue) is driven
  by active `store_assignments`; with no way to assign operators to a new store,
  the scoping can't be tested with a real second operator.
- New drivers/operators can't be wired to a branch at all without a DB edit.

Good news: **no schema change is needed.** Both tables already exist (`stores`:
name/address/phone/active; `store_assignments`: storeId/userId/active with a
partial-unique `uq_store_assignments_active` on `(storeId, userId) WHERE active`,
so a user may belong to multiple stores but not double-bind to one). This is
purely an **admin write surface**.

## Proposed Solution

Add admin-only write endpoints + UI on top of the existing catalog module.

- **Stores CRUD** — `POST /catalog/stores` (create) and `PATCH /catalog/stores/:id`
  (edit name/address/phone, activate/deactivate). The "Tiendas" tab gains a
  create/edit dialog mirroring the existing tank-types/items dialogs + a
  show-inactive toggle. **This alone unblocks transfer + the admin switcher.**
- **Assignment management** — `POST /catalog/store-assignments` (link a user to a
  store; honour the partial-unique active constraint → 409) and
  `PATCH /catalog/store-assignments/:id` (deactivate). A new "Asignaciones"
  surface (catalog tab) lists who's assigned where and lets admin add/deactivate.
  **This unblocks operator branch-scoping testing.**

Both are admin/developer-gated, consistent with `POST /tank-types` / `POST /items`.

Detailed endpoints and file lists live in [[backend]] and [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `POST /catalog/stores` (admin/developer): create a store — `name` required
  (≤120), `address?` (≤255), `phone?` (≤32); `active` defaults true. Returns the
  `PublicStore`. Zod-validated, mirroring the tank-types/items create shape.
- [ ] `PATCH /catalog/stores/:id` (admin/developer): partial update of
  `name`/`address`/`phone`/`active`; 404 when the store doesn't exist.
- [ ] `POST /catalog/store-assignments` (admin/developer): link `{ storeId, userId }`
  — both must exist; the user's role should be assignable (operator/delivery);
  respect the partial-unique active constraint → **409** when an active link
  already exists. Returns the enriched `StoreAssignmentDetail`.
- [ ] `PATCH /catalog/store-assignments/:id` (admin/developer): deactivate
  (`active: false`) so the user leaves that branch's scope; 404 when missing.
  (Soft deactivate, not hard delete — preserves history.)
- [ ] No schema/migration change (tables exist). Reads/writes flow through
  `src/lib/errors.ts`. Existing `stores-and-catalog` read behaviour + tests stay
  green.
- [ ] Tests: (a) create store → appears in `GET /stores`; (b) create assignment →
  appears in `GET /store-assignments`, and a duplicate active link → 409;
  (c) deactivate assignment → drops from the active list (and from the user's
  branch scope).

**Frontend (lpg-frontend-vue):**

- [ ] Catalog **"Tiendas"** tab becomes admin **create + edit** (dialog mirroring
  `TankTypeCreateDialog`/`ItemCreateDialog`, client-side validation mirroring the
  backend Zod), with a show-inactive toggle and activate/deactivate.
- [ ] A **store-assignments** management surface (new "Asignaciones" catalog tab):
  pick a user + a store to assign, list active assignments (who · where), and
  deactivate one. Admin/developer only.
- [ ] Role/nav consistent (Catálogo already in the admin nav); consistent with
  `eng/design-system.md`. After creating a store it's immediately usable in the
  orders store switcher / transfer dialog (no extra wiring).

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

> **Porting order:** backend first (stores create/edit + assignment create/deactivate),
> frontend right after. The **stores CRUD** slice alone already unblocks transfer +
> the admin switcher, so it can ship before assignment management if needed.

## Out of Scope

- **Schema changes** — none; the tables already exist.
- **Store geo/coordinates, hours, per-store config** — just name/address/phone for
  now (nearest-store routing was already out of scope in orders-multi-location).
- **Bulk import** of stores or assignments.
- **A dedicated "global operator" capability** — still covered by admin; deferred
  as in orders-multi-location.
- **Reassigning a driver's open inventory day across stores** — assignment changes
  are forward-looking; they don't rewrite existing inventory/orders.

## Open Questions

- **Store name uniqueness:** enforce unique (active) store names (409 on dup), or
  allow duplicates? (Lean: allow — branches can legitimately share a name; revisit
  if it confuses the switcher.)
- **Assignable roles:** restrict `store_assignments.userId` to operator/delivery,
  or allow any role? (Lean: operator + delivery — they're the scoped roles; admin
  is global and needs no assignment.)
- **Assignments UI placement:** a catalog tab ("Asignaciones") vs a section inside
  a store's detail view? (Lean: a catalog tab, matching the tabbed catalog page.)
- **Deactivate vs delete** for assignments. (Lean: soft deactivate — preserves the
  audit trail and matches the `active` column already on the table.)
