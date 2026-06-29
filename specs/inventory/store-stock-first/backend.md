---
project: lpg-store
domain: specs
type: spec-track
spec: store-stock-first
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-27
---

# Store Stock First — lpg-backend track (aggregate availability endpoint)

Shared spec: [[index]] · Frontend consumer: [[frontend]]

> **Contract track.** This endpoint is the data behind the frontend Resumen
> overview. The frontend track is built against the shape below; implement this in
> `lpg-backend` and the overview goes live. Keep the response shape stable or
> update [[frontend]] in lock-step.

## Technical Notes

A focused read-only addition to the **inventory** module. It generalises the
existing single-store `GET /stores/:storeId/availability` to **all the caller's
stores in one request**, mirroring the caller-scoping already used by
`listAssignmentsForCaller` / `listStorePurchases`.

### Route

- `GET /api/v1/inventory/availability` — role-guarded `requireRole('operator',
  'admin')` (developer passes via the admin alias). Optional `?storeId=` to narrow.
- Add it next to `GET /stores/:storeId/availability` in
  [routes.ts](../../../../lpg-backend/src/modules/inventory/routes.ts). Use the
  existing `caller(req)` helper for `{ id, role }`.

### Response shape (the contract the frontend consumes)

```json
{
  "stores": [
    {
      "storeId": 1,
      "storeName": "Tienda Las Américas",
      "shop":    [{ "tankTypeId": 3, "fullTanks": 12, "emptyTanks": 8 }],
      "onTruck": [{ "tankTypeId": 3, "fullTanks": 5,  "emptyTanks": 0 }]
    },
    { "storeId": 2, "storeName": "Tienda Basilio Auqui", "shop": [], "onTruck": [] }
  ]
}
```

- `shop` / `onTruck` rows reuse the existing `toTankRows` mapping (`{ tankTypeId,
  fullTanks, emptyTanks }`).
- **A scoped store with no holders still appears** with empty arrays (so the UI
  renders it at 0). This means the store set is driven by *which stores the caller
  can see*, not by which stores happen to have holder rows.

### Service — `getAvailabilityForCaller(caller, { storeId? })`

- Resolve the **store set**:
  - admin/developer (`isGlobal`): **all active stores** → needs a catalog read
    (see below).
  - operator: `await this.repo.storeIdsForUser(caller.id)`.
  - if `storeId` is given: intersect with the set (outside scope → empty result),
    same pattern as `listAssignmentsForCaller`.
- Fetch grouped balances + store names, then assemble one entry per store id in the
  set (empty `shop`/`onTruck` when a store has no rows). Order by store name (or
  id) for a stable card order.

### Repository additions

- **Store enumeration + names.** Add a catalog read — e.g. `listActiveStores():
  Promise<{ id: number; name: string }[]>` and/or `storeNamesByIds(ids):
  Promise<Map<number, string>>`. The repo already imports `storeAssignments`,
  `tankTypes`, `inventoryItems` from `../catalog/schema`; import `stores` too.
- **Grouped shop balances** `tankBalancesByLocations(storeIds): Promise<(TankBalanceRow & {
  storeId })[]>` — the existing `tankBalancesByLocation` query with `inArray(tankBalance.storeId,
  storeIds)` instead of `eq`. Short-circuit on empty `storeIds` (mirror
  `listAssignments`).
- **Grouped on-truck balances** `tankBalancesOnTrucks(storeIds)` — the existing
  `tankBalancesOnTruck` join (assignment-holder balances for `open` assignments
  whose store is in the set), selecting `storeAssignments.storeId` so rows can be
  grouped back per store, `inArray(storeAssignments.storeId, storeIds)`.

### Reuse / don't reinvent

- `toTankRows` (service) — the row mapping.
- `storeIdsForUser`, `isGlobal`, the `?storeId=` intersect — copy the
  `listAssignmentsForCaller` scoping verbatim.
- The on-truck SQL already exists in `tankBalancesOnTruck`; the grouped variant is
  the same query with `inArray` + the `storeId` column selected.

### Tests

- Operator sees only `storeIdsForUser` stores; admin/developer see all active
  stores; `?storeId=` intersects (in-scope → that store; out-of-scope → empty).
- A store with no holders is **present** with empty arrays.
- Shop vs on-truck split is correct (an open assignment's stock shows under
  `onTruck`, not `shop`).
- Extend the in-memory `FakeInventoryRepository` with the grouped reads + store
  enumeration. Gates: typecheck + `biome check` + tests + build.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — add `GET /availability` (guard
  `operator`/`admin`), call `service.getAvailabilityForCaller(caller(req), query)`.
- `src/modules/inventory/service.ts` — `getAvailabilityForCaller`; reuse
  `toTankRows`, `storeIdsForUser`, `isGlobal`, the `?storeId=` intersect from
  `listAssignmentsForCaller` ([service.ts:739](../../../../lpg-backend/src/modules/inventory/service.ts)).
- `src/modules/inventory/repository.ts` — `tankBalancesByLocations`,
  `tankBalancesOnTrucks`, and an active-store enumeration/name read; import
  `stores` from `../catalog/schema`
  ([repository.ts:492-520](../../../../lpg-backend/src/modules/inventory/repository.ts)).
- `src/modules/inventory/types.ts` — query schema for `?storeId=` (+ the
  `StoreAvailabilityView` response type if typed).
- `src/modules/inventory/__tests__/*.ts` + the `FakeInventoryRepository` — grouped
  reads + scoping tests.

Read-only context:

- `src/modules/catalog/schema.ts` — `stores` (id, name, active).
- `src/modules/inventory/service.ts` `getLocationAvailability` + `toTankRows` (the
  single-store shape this generalises).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-27] [lpg-backend] Shipped the multi-store aggregate availability endpoint `GET /api/v1/inventory/availability` (guard `operator`/`admin`; developer passes via the alias; optional `?storeId=`). Generalises the single-store `getLocationAvailability`:

- **service** `getAvailabilityForCaller(caller, { storeId? })` — resolves the visible store set (global → `listActiveStores()`; operator → `storeIdsForUser`), narrows by `?storeId=` (out-of-scope → empty, never widens — same rule as `listAssignmentsForCaller`), assembles one entry per store with shop + on-truck balances grouped by store id (`groupByStore`). A scoped store with **no holders still appears** with empty arrays (set driven by visibility, not holder rows). Reuses `toTankRows`/`isGlobal`.
- **repository** `listActiveStores()` (ORDER BY name), `storeNamesByIds()`, `tankBalancesByLocations(storeIds)` + `tankBalancesOnTrucks(storeIds)` — the existing location/on-truck reads generalised to `inArray`; on-truck rows tagged with `storeAssignments.storeId` so they group back per store; empty-`storeIds` short-circuit on both. New `StoreTankBalanceRow` type; imported `stores` from catalog schema.
- **types** `availabilityQuerySchema` (`?storeId=` via `intQuery`) + `StoreAvailabilityView`. Response shape `{ stores: [{ storeId, storeName, shop[], onTruck[] }] }` (route wraps the service array as `{ stores }`).
- **tests** new `__tests__/store-availability.test.ts` (8 service + 3 HTTP): operator own-branch scoping, admin/developer see all active (inactive excluded), `?storeId=` intersect (in/out of scope), zero-holder store present with empty arrays, shop-vs-onTruck split, route guard (delivery → 403, unauth → 401). Extended `FakeInventoryRepository` (`seedStore` + the four reads).
- No schema/migration change; no accounting egress. Gates green: typecheck, `biome check`, **114** tests, build.
- Deferred product call (flagged in validation, within the spec's stated contract): an operator with an *active* assignment to a *deactivated* store would still see it (operator scope = `storeIdsForUser`, no `store.active` filter) — an asymmetry vs the admin/global path. Left as-is per the contract; revisit if it bites.
[2026-06-27] [lpg-frontend-vue] Contract **verified from the frontend session**:
the live `GET /inventory/availability` matches this spec — `canPurchase`-guarded,
returns `{ stores: [{ storeId, storeName, shop[], onTruck[] }] }`, caller-scoped
(global → `listActiveStores`; operator → `storeIdsForUser`; `?storeId` intersect),
a no-holder store still appears with empty arrays, and grouped reads
(`tankBalancesByLocations` / `tankBalancesOnTrucks`) back it
(`service.getAvailabilityForCaller`, `routes.ts`). Backend track implemented in
lpg-backend by the owner.