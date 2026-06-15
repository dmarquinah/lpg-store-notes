---
project: lpg-store
domain: specs
type: spec-track
spec: store-management
repo: lpg-backend
kind: backend
track-status: not-started
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
