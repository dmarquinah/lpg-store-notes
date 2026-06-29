---
project: lpg-store
domain: specs
type: spec-track
spec: store-history
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-29
---

# Store History — lpg-backend track

Shared spec: [[index]] · Writers it reads back: [[../store-stock-adjustments/backend]]

## Technical Notes

A read-only composition over the existing inventory ledgers. The closest existing
precedent is `GET /inventory/history` (reconstructs **one user's day**) and the
purchases list `GET /inventory/stores/:storeId/purchases` (a store-scoped,
windowed read over the location holder) — this generalises that to **all** kinds,
not just `purchase`.

### Route + scoping

- **Route:** `GET /api/v1/inventory/stores/:storeId/history?from=&to=`.
- **Scoping:** operator own-store, admin/dev global — reuse the same
  `storeIdsForUser` check `getLocationAvailability` / the purchases list use;
  cross-store `:storeId` → `ForbiddenError` 403.
- **Window:** Lima business-date `from`/`to` (reuse `toBusinessDate` / the same
  bounds-defaulting the purchases list does); default e.g. last 30 days.

### Service + repository read

- **`listStoreHistory(storeId, { from, to })`** → unified, newest-first list.
- **Repository read** (`storeHistoryForPeriod`): select `tank_transactions` joined
  to `tank_holders` filtered `kind='location' AND store_id=:storeId`, and the same
  for `item_transactions` / `item_holders`; **join `users`** for the actor name
  (`tank_transactions.user_id` → `users.name`, like the order-timeline enrichment
  did for `changedByName`). Project a common shape:
  `{ source: 'tank'|'item', txId, kind, occurredAt, businessDate, userId,
  userName, fullDelta?, emptyDelta?, delta?, qty?, productId, productName, notes,
  refOrderId, refCustomerId }`. Merge tank + item and sort by `occurred_at DESC,
  id DESC`. Resolve `productName` via the existing `tank_types` /
  `inventory_items` joins the purchases list already uses.
- **Location join is the security boundary:** because every row is reached via a
  `kind='location'` holder for `:storeId`, an **assignment**-holder transaction
  (a driver-day) for another store can't leak in. Add a test asserting this.
- Mapping `kind` → label is a **frontend** concern; the backend returns the raw
  `kind` string.

### Order events — separate, not here

Per the shared spec, `/history` is inventory movements only. Order-lifecycle
events are fetched lazily by the frontend. Prefer **reusing the existing
store-scoped `GET /orders?storeId=&from=&to=`** (added in `orders-multi-location`)
— no new endpoint. Only add a dedicated `GET
/inventory/stores/:id/order-events` if the orders list shape proves awkward to
merge (Open Question in [[index]]).

### Reuse / don't reinvent

- The location-holder + product-name + business-date plumbing already exists in
  `purchaseLinesForStorePeriod` (the purchases list) — generalise it to all kinds
  rather than writing a new query path from scratch.
- Actor-name join: mirror the `users` join pattern from the orders
  status-history enrichment (`order-event-timeline`).
- Errors / scope helpers: `src/lib/errors.ts`, `storeIdsForUser`.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — add `GET /stores/:storeId/history` (scope
  like `/stores/:storeId/availability`).
- `src/modules/inventory/service.ts` — `listStoreHistory(storeId, window, caller)`
  (scope check + window defaulting + merge/sort).
- `src/modules/inventory/repository.ts` — `storeHistoryForPeriod`: tank + item
  location-holder reads joined to `users` (+ `tank_types`/`inventory_items` for
  names). Generalise the existing `purchaseLinesForStorePeriod` shape.
- `src/modules/inventory/types.ts` — the history-entry view type.
- `src/modules/inventory/__tests__/*.ts` — merged ordering, actor-name
  resolution, location-join isolation (no foreign-store/assignment leak), window.

Read-only context (no change):

- `src/modules/inventory/repository.ts` — `purchaseLinesForStorePeriod` (the
  pattern to generalise), `tankBalancesByLocation`.
- `src/modules/auth/schema.ts` / users table — the actor-name source.
- `src/modules/orders/` — `GET /orders?storeId=` (the lazy order-events source the
  frontend will call; no change unless a dedicated read is chosen).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-29] [lpg-backend] Backend track DONE — all 6 backend criteria met; independent validation GREEN (incl. frontend-contract match); gates green (typecheck + `biome check` + **148** tests + build).

- **Route:** `GET /api/v1/inventory/stores/:storeId/history?from=&to=` (routes.ts), guarded by `canPurchase` (operator/admin; developer auto-passes; **delivery → 403** at the role layer). Returns `{ history }`. Reuses `storePurchasesQuerySchema` (`from`/`to`, `from ≤ to`).
- **Service** (`listStoreHistory`, service.ts): own-store scope (`storeIdsForUser` / `isGlobal`) → cross-store **403**; default window = **today + 30 prior Lima days** (`addDays(today,-30)`; 31-day inclusive span); maps rows → the view via `toStoreHistoryView` (ISO `occurredAt`, `businessDate`, and ONLY the leg for the row's `source` — `fullDelta`/`emptyDelta` for tanks, `delta` for items).
- **Repository** (`storeHistoryForPeriod`, repository.ts): generalises `purchaseLinesForStorePeriod` to **all kinds** — two windowed selects (tank_transactions⋈tank_holders⋈tank_types⋈**users**; item_transactions⋈item_holders⋈inventory_items⋈**users**), filtered `kind='location' AND store_id=:storeId`, merged + sorted `occurredAt DESC, id DESC`. New exported `StoreHistoryRow`.
- **Frontend contract MATCHED** (frontend track already shipped): response is `{ history: StoreHistoryEntry[] }`; `userName` is **always a non-null string** (innerJoin + `users.name` NOT NULL + `user_id` NOT NULL FK); `source`-correct deltas; `productName`/`notes`/`refOrderId`/`refCustomerId` present (`refCustomerId` always null for items — item ledger has no customer ref).
- **Security boundary = the location-holder join:** a driver-day (assignment-holder) leg and any foreign store's rows can't appear. Asserted by test (the opening's truck leg `+3` and store-2's `+7` are both absent).
- **Order events NOT here** — the frontend lazy-loads them from the existing `GET /orders?storeId=` (no new endpoint, per the shared spec). **No schema change, no new ledger, no write path.**
- **Tests** (store-history.test.ts, 10): merged newest-first ordering, truck-leg + foreign-store isolation, actor-name resolution, item delta/product/actor, adjustment audit row, date-window exclusion, scoping (own-store 200 / cross-store 403 / delivery 403 / 401), `{ history }` shape + `businessDate`.

### Shipped alongside: own-store scoping fix on the provider-purchase write endpoints (pre-existing gap)

`POST /inventory/stores/:storeId/{tank,item}-purchases` enforced only role, not own-store — an operator could record a purchase (and its **accounting egress**) into ANY store. Fixed: `recordTankPurchase`/`recordItemPurchase` now take `actor: number | {id,role}` and resolve via a new `resolvePurchaseActor` — a **caller object (HTTP path)** is scope-checked (operator own-store; admin/dev global; cross-store → **403**, nothing written), a **bare numeric userId** (trusted internal/composition + tests) stays unscoped (mirrors the `recordSale` tx seam). Routes now pass `caller(req)`. New `purchase-scoping.test.ts` (4 HTTP tests); two legacy HTTP tests (`day-handoff`, `lifecycle`) updated to stock the shop as an own-store/admin actor. This closes the inconsistency where every other store-scoped surface already enforced own-store.
