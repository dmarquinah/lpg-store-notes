---
project: lpg-store
domain: specs
type: spec-track
spec: store-detail-products
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-28
---

# Store Detail Products — lpg-backend track (per-store item availability)

Shared spec: [[index]] · Frontend consumer: [[frontend]]

> **Contract track.** The frontend Artículos tab is built against the shape below.
> Implement this in `lpg-backend` and the tab goes live.

## Technical Notes

The item analogue of the existing tank `getLocationAvailability`. The
`item_balance` view already exists and is `LEFT JOIN`-based (a holder row appears
even at 0), but it's only ever read **per assignment** today — add the
**per-location** read + an endpoint.

### Route

- `GET /api/v1/inventory/stores/:storeId/item-availability` — auth mirrors the
  existing tank route `GET /stores/:storeId/availability` (authenticated; no extra
  role guard, consistent with that one — revisit if store reads get scoped).
- Add it next to the tank availability route in
  [routes.ts](../../../../lpg-backend/src/modules/inventory/routes.ts).

### Response shape (the contract the frontend consumes)

```json
{ "items": [ { "inventoryItemId": 4, "qty": 12 }, { "inventoryItemId": 7, "qty": 0 } ] }
```

- **Holder-based** (same rule as tanks): an item the store has a `location` holder
  for appears — *including* `qty: 0` after it's been depleted; an item the store
  has **never purchased** is **absent** (no holder → no row).

### Service + repository

- **Repo** `itemBalancesByLocation(storeId): Promise<ItemBalanceRow[]>` — mirror
  `tankBalancesByLocation`: `select().from(itemBalance).where(and(eq(itemBalance.kind,
  'location'), eq(itemBalance.storeId, storeId)))`
  ([repository.ts:492](../../../../lpg-backend/src/modules/inventory/repository.ts)
  is the tank version to copy). `item_balance` exposes `kind`, `storeId`,
  `inventoryItemId`, `currentQty`.
- **Service** `getLocationItemAvailability(storeId): Promise<{ inventoryItemId:
  number; qty: number }[]>` — map `currentQty → qty` (a `toItemRows` helper akin to
  `toTankRows` in [service.ts:952](../../../../lpg-backend/src/modules/inventory/service.ts)).
  Return `{ items }` from the route.

### Reuse / don't reinvent

- `item_balance` view + `ItemBalanceRow` type already exist
  ([schema.ts:215](../../../../lpg-backend/src/modules/inventory/schema.ts)).
- Copy the tank `tankBalancesByLocation` query shape verbatim, swapping the view +
  column (`currentQty`).
- Extend the in-memory `FakeInventoryRepository` with `itemBalancesByLocation`.

### Tests

- A never-purchased item is **absent** from the response.
- A purchased-then-0 item appears at `qty: 0` (LEFT JOIN row).
- Item purchase → the item shows with the bought qty. Gates: typecheck + biome +
  tests + build.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — add `GET /stores/:storeId/item-availability`.
- `src/modules/inventory/service.ts` — `getLocationItemAvailability` + a
  `toItemRows` helper.
- `src/modules/inventory/repository.ts` — `itemBalancesByLocation` (mirror
  `tankBalancesByLocation`).
- `src/modules/inventory/__tests__/*.ts` + `FakeInventoryRepository` — the read +
  holder-based tests.

Read-only context:

- `src/modules/inventory/schema.ts` — `item_balance` view + `ItemBalanceRow`.
- `src/modules/inventory/service.ts` — `getLocationAvailability` / `toTankRows`
  (the tank shape this mirrors).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-28] [lpg-backend] All criteria for this repo met. Added the per-store item availability endpoint `GET /api/v1/inventory/stores/:storeId/item-availability` → `{ items: [{ inventoryItemId, qty }] }`, the item analogue of the tank `getLocationAvailability`:

- **repository** `itemBalancesByLocation(storeId)` — `select().from(itemBalance).where(and(eq(kind,'location'), eq(storeId, …)))`, mirroring `tankBalancesByLocation`. Holder-based over the existing `item_balance` view.
- **service** `getLocationItemAvailability(storeId)` → `toItemRows(...)` (new local helper mapping `currentQty → qty`, akin to `toTankRows`). Route wraps the array as `{ items }`.
- **route** added next to `/stores/:storeId/availability`; identical auth (global `requireAuth` only, no role guard).
- **tests** new `__tests__/store-item-availability.test.ts` (4 service + 2 HTTP): never-purchased absent, purchased-then-0 present at qty 0, no cross-store leak, wire shape `{ items }`, 401 unauth. Refactored the Fake to share a `sumItemHolder` between `itemBalancesByAssignment` (behavior-preserving) and the new `itemBalancesByLocation`.
- **Verified the `.existing()` view SQL** (migration `0002`, the `item_balance` def): `item_holders LEFT JOIN item_transactions … GROUP BY h.id`, `COALESCE(SUM(delta),0)`, **no `HAVING`** — so a depleted location holder really does surface at `qty: 0` in prod (matches the Fake; the "purchased-then-0 present" guarantee holds).
- No schema/migration change. Gates green: typecheck, `biome check`, **119** tests, build.
[2026-06-28] [lpg-backend] Backend track **done** (implemented by the owner).
Contract **verified from the frontend session**: `GET
/inventory/stores/:storeId/item-availability` → `{ items: [{ inventoryItemId, qty
}] }` (`routes.ts` handler `res.json({ items })`), backed by service
`getLocationItemAvailability(storeId)` → repo `itemBalancesByLocation` over the
`item_balance` view, mapped by a `toItemRows` helper (`currentQty → qty`) — the
item analogue of `toTankRows`, holder-based as specified (purchased-incl-0 shown,
never-purchased absent). The frontend Artículos tab consumes it unchanged.