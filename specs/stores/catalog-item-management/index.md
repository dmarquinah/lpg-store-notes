---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../stores-and-catalog/index]]"
  - "[[../store-management/index]]"
last-updated: 2026-06-18
---

# Spec: Catalog Item Management (edit + deactivate/restore tank types & items)

## Problem Statement

The v2 catalog module is **create-only** for the two product tables. `POST
/catalog/tank-types` and `POST /catalog/items` exist, but there is **no `PATCH`**
for either — only `stores` and `store-assignments` got edit/activate endpoints (see
`src/modules/catalog/routes.ts`). So once a tank type or item is created, an admin
**cannot fix a mistyped price, rename it, or retire a discontinued product** without
a direct database edit.

v1 had the full lifecycle — `PUT /products/tanks/:id`, `DELETE` (soft), and
`PATCH …/restore`, likewise for items (`legacy/src/routes/productRoutes.ts`). This
is the one genuine business-relevant parity gap surfaced by the 2026-06-18
legacy-vs-v2 audit (everything else v1 had that v2 lacks was a deliberate v2
simplification — workflow-over-CRUD, FCM/analytics removal, etc.).

**Scope discipline.** This is the *editing* gap only — not a reintroduction of v1's
full product surface (no cross-product `GET /products/search`, no per-store catalog
re-initialize). Keep it to what the business needs: correct and retire catalog
entries. Pricing-history semantics are already handled — inventory holders snapshot
the price in effect when a day opens, so editing a catalog price only affects
**future** day-opens, never closed/consolidated days. Confirm and preserve that.

## Proposed Solution

Add the missing write surface to the existing `catalog` module, mirroring the shape
already used for `PATCH /catalog/stores/:id` (partial update + an `active` flag for
soft deactivate/restore) — no new module, no schema redesign.

- **Backend:** `PATCH /catalog/tank-types/:id` and `PATCH /catalog/items/:id` —
  admin-only, partial update of the editable fields (name, prices, and `active` for
  deactivate/restore), reusing the validation shapes already defined for create. The
  list endpoints already support `?all=1` to include inactive rows, so deactivated
  entries remain visible to admins. Verify the tables already carry an `active`/soft
  flag from `stores-and-catalog`; if a tank type or item is referenced by inventory
  holders, deactivation hides it from new pickers but must not break historical rows.
- **Frontend:** the catalog admin tabs (Tipos de tanque / Artículos) gain **edit**
  affordances reusing the existing create dialogs in edit mode (prefilled), plus an
  activate/deactivate toggle and a show-inactive switch — exactly mirroring how the
  Tiendas tab already does store edit/activate (the `store-management` pattern).

Backend detail in [[backend]]; the admin UI in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

- [x] `PATCH /catalog/tank-types/:id` — admin-only, partial update of the editable
      fields (name + prices), validated with the same rules as create; returns the
      updated row.
- [x] `PATCH /catalog/items/:id` — same, for inventory items.
- [x] Both endpoints support **deactivate/restore** via an `active` flag (soft, not a
      hard delete), consistent with `PATCH /catalog/stores/:id`. Inactive rows are
      excluded from default lists and included under `?all=1`.
- [x] Deactivating a tank type/item that is referenced by existing inventory holders
      or historical transactions does **not** break those rows or any
      closed/consolidated day; it only removes the entry from new pickers. Editing a
      price affects only future day-opens (holders snapshot price at open) — verified.
- [x] Backend lifecycle test(s) cover: edit a field, deactivate (drops from default
      list, present with `?all=1`), restore; admin-gating enforced. Existing tests
      stay green; typecheck + `biome check` + tests + build green.
- [x] Frontend: the Tipos de tanque and Artículos tabs offer **edit** (prefilled
      dialog) + **activate/deactivate** + show-inactive, mirroring the existing
      Tiendas tab; validation mirrors the backend; design-system tokens only,
      ≥44px touch targets. typecheck + build green.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Cross-product search** (`GET /products/search` in v1) — orders/customers already
  have `?search=`; a unified catalog search isn't a business need.
- **Per-store catalog initialize/re-init** (v1 `POST /stores/:id/catalog/initialize`)
  — v2 doesn't scope the catalog per store this way.
- **Hard delete** — soft deactivate only, to protect referential integrity with
  inventory/orders history.
- **Restoring v1's `is_popular` / `scale` / other trimmed fields** — keep the lean v2
  shape from `stores-and-catalog`.

## Open Questions

- Do the `tank_types` / `inventory_items` tables already have an `active` (or
  soft-delete) column from `stores-and-catalog`, or does deactivate require a small
  migration to add one? (Check `src/modules/catalog/schema.ts` — if absent, a tiny
  additive migration is in scope.)
- Are there fields that must **not** be editable after creation (e.g. a unit/weight
  that inventory rows depend on)? Confirm which fields are safe to PATCH.
