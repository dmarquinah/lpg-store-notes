---
project: lpg-store
domain: specs
type: spec-track
spec: store-management
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-14
---

# Store & Assignment Management — lpg-backend track

Shared spec: [[index]]

## Technical Notes

No schema change — `stores` and `store_assignments` already exist. This is new
write endpoints on the existing catalog module, gated `requireRole('admin')`
(developer bypasses via the guard, as elsewhere), mirroring the existing
`POST /tank-types` / `POST /items`.

### Stores

- `POST /stores` — `createStoreSchema` = `{ name: 1..120, address?: ≤255,
  phone?: ≤32 }`; `active` defaults true. Service `createStore` → repo insert →
  return `PublicStore`.
- `PATCH /stores/:id` — `updateStoreSchema` (all optional: name/address/phone/
  active). Repo `updateStore(id, patch)`; throw `NotFoundError` on 0 rows.

### Store assignments

- `POST /store-assignments` — `{ storeId, userId }`. Validate the store exists and
  the user exists (and is an assignable role — operator/delivery; see Open
  Questions). Insert; the partial-unique `uq_store_assignments_active` means a
  duplicate active link throws — catch the unique violation and surface
  `ConflictError('El usuario ya está asignado a esta tienda')` (mirror the
  customers dup-phone 409 pattern). Return the enriched `StoreAssignmentDetail`
  (reuse the existing list join shape).
- `PATCH /store-assignments/:id` — `{ active: boolean }` (deactivate = false).
  Repo update; `NotFoundError` on 0 rows. Soft, so the partial-unique index frees
  up and the user drops out of `storeIdsForUser` (the orders scope).

Keep everything flowing through `src/lib/errors.ts`. The orders module's
`storeIdsForUser` / `storeExists` already read these tables — no change there.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/catalog/types.ts` — `createStoreSchema`, `updateStoreSchema`, `createStoreAssignmentSchema`, `updateStoreAssignmentSchema`
- `src/modules/catalog/repository.ts` — `createStore`, `updateStore`, `createStoreAssignment` (+ unique-violation mapping), `setStoreAssignmentActive`, `storeExists`/`userExists` helpers if not present
- `src/modules/catalog/service.ts` — `createStore`, `updateStore`, `createStoreAssignment`, `deactivateStoreAssignment` (role check on the target user)
- `src/modules/catalog/routes.ts` — `POST /stores`, `PATCH /stores/:id`, `POST /store-assignments`, `PATCH /store-assignments/:id` (all `requireRole('admin')`)
- `src/modules/catalog/__tests__/*` — create store; create assignment + dup-409; deactivate

Context (read; do not needlessly modify):

- `src/modules/catalog/schema.ts` — `stores`, `store_assignments` (`uq_store_assignments_active`)
- `src/modules/auth/{middleware,schema}.ts` — `requireRole`, the role enum (assignable roles)
- `src/modules/orders/repository.ts` — `storeIdsForUser` / `storeExists` (the consumers; reference only)
- `src/lib/errors.ts`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-14] [lpg-backend] Backend track done. Admin write surface added to the existing `catalog` module — no schema/migration change (tables already existed).

- **types.ts** — `createStoreSchema` (name ≤120, address? ≤255, phone? ≤32), `updateStoreSchema` (all-optional name/address/phone/active; address/phone `.nullable()` so an explicit `null` clears them; `.refine` requires ≥1 field), `createStoreAssignmentSchema` ({storeId,userId} positive ints), `updateStoreAssignmentSchema` ({active}), and a shared `idParamSchema` (`z.coerce.number().int().positive()`) for `:id` params.
- **repository.ts** — extended `ICatalogRepository` + class: `createStore`, `updateStore` (→ Store|null), `getStoreById`, `findActiveStoreByName(name, excludeId?)` (case-insensitive `lower()=lower()`, active-scoped), `findUser`, `findActiveAssignment`, `createStoreAssignment` (maps PG `23505` → `ConflictError` as a race backstop), `setStoreAssignmentActive`, `getStoreAssignmentDetail` (reuses the existing 3-table join, filtered by id, no active filter). Added `CatalogUser` + `StorePatch` exported types.
- **service.ts** — `createStore`/`updateStore` enforce **unique active store names** via a `findActiveStoreByName` pre-check → 409 (owner override: no duplicates). `updateStore` re-checks the name on rename and on reactivation (`active:true`); builds a partial patch (only provided keys). `createStoreAssignment` validates store+user exist (404 each), allows **any role** (owner override), dup active link → 409, returns the enriched `StoreAssignmentDetail` built from the validated store+user. `setStoreAssignmentActive` soft-deactivates; on reactivation re-runs the dup-check.
- **routes.ts** — `POST /stores` (201), `PATCH /stores/:id` (200), `POST /store-assignments` (201), `PATCH /store-assignments/:id` (200), all `requireRole('admin')` (developer bypasses via the guard).
- **__tests__** — extended `FakeCatalogRepository` with the new methods + a `seedUser` helper + a `patchJson` HTTP helper; 3 new lifecycle tests (store create/edit/dup-409/404; assignment create/dup-409/404/deactivate/re-link; non-admin 403). Catalog suite 4 → 7 tests; **project 59 → 62 tests**.

**Owner-decided open questions:** (1) store names **unique** (not duplicates allowed); (2) **any** role assignable (not restricted to operator/delivery). Both leans in `index.md` were overridden accordingly.

Gates green: typecheck ✓ · biome lint ✓ · 62 tests ✓ · build ✓. Independent validation confirmed all 8 backend criteria, no gaps. Note: create-store uniqueness is an app-level pre-check only (no DB unique index, since no schema change is allowed) — a small TOCTOU window acceptable for this low-write admin surface. Frontend track remains.