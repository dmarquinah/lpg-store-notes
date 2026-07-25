---
project: lpg-store
domain: specs
type: spec-track
spec: quick-assignment
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-04
---

# Quick Assignment — lpg-backend track

Shared spec: [[index]]

## Technical Notes

Additive to the **inventory** module. Two compositions over existing primitives —
**no schema/migration change**.

### 1. Quick-open — `POST /inventory/stores/:storeId/quick-open`

- **Resolve the sole driver.** Use catalog's
  `listStoreAssignmentDetails({ storeId, role: 'delivery', active: true })`
  (already exists — `catalog/service.ts:43`). Exactly **1** row → its
  `storeAssignmentId` is the target. **0** or **>1** → `ConflictError` 409 with a
  message that points to the general dialog (>1) or to assigning a driver (0).
- **Sweep the floor.** Read `tankBalancesByLocation(storeId)`; build the
  `tanks[]` payload as every type with `full>0 || empty>0` (full+empty per the
  Open Question — default both).
- **Reuse `openDay`.** Call the existing `openDay({ storeAssignmentId, tanks },
  userId)` verbatim — do **not** fork the transfer logic. That already enforces
  today's-date-only, the duplicate-day 409, the unconsolidated-prior-day 409, the
  negative-stock guard inside `loadFromLocation`, and fires the driver push
  notification after commit. Quick-open is essentially *"compute the `tanks[]` the
  operator would have typed, then call `openDay`."*
- **Scope guard** mirrors the purchase routes (operator own-store + admin/dev;
  cross-store → `ForbiddenError` 403). NB: `recordTankPurchase` does **not**
  actually enforce own-store scoping today (noted in
  [[../store-stock-adjustments/backend]]) — add the explicit check here rather
  than assume it, using the same `storeIdsForUser` scope helper.

### 2. Purchase auto-load

- In `recordTankPurchase` (`service.ts:487`) and `recordItemPurchase`
  (`service.ts:547`), **after** the purchase legs are written **inside the same
  `repo.transaction`**: look up the store's active delivery assignments; if
  **exactly one** and it has an **`open`** day for today
  (`findAssignmentByDay(storeAssignmentId, today)` with `state === 'open'`), load
  the just-purchased full/empty onto that assignment via the existing
  `loadFromLocation(r, storeId, assignmentId, tankTypeId, full, empty, userId,
  'load')`. Item purchases mirror with the item-load primitive (confirm it
  exists — see Open Questions; scope to tanks-only for v1 if not).
- **Same transaction** → if the load leg fails, the purchase rolls back (no
  half-state). The load is a normal `load`-kind transfer, so the day's
  reconstruction/opening already counts it (see the `recordLoad` comment,
  service.ts:158).
- **Skip conditions** (purchase lands only on `location`, today's behavior): no
  open day, 0 or >1 active delivery, or the day is `closed`/`carried`. **No
  auto-open** — decided.
- Keep this a small private helper (e.g. `autoLoadPurchaseToSoleDriver(r,
  storeId, lines, userId)`) called from both purchase services, so the rule lives
  in one place.

### Reuse / don't reinvent

- `openDay`, `loadFromLocation`, `tankBalancesByLocation`, `findAssignmentByDay`,
  `findUnconsolidated` — all in `inventory/service.ts` + `repository.ts`.
- Sole-driver lookup: `catalog` `listStoreAssignmentDetails` (inject the catalog
  read the inventory module already has access to, or add a thin repo read of
  `store_assignments` filtered by role — prefer reusing catalog to avoid a second
  source of truth).
- Errors: `src/lib/errors.ts` (`ConflictError`, `ForbiddenError`,
  `ValidationError`).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — add `POST /stores/:storeId/quick-open`
  (guard like the tank-purchase route, ~line 250).
- `src/modules/inventory/service.ts` — `quickOpen(storeId, caller)` (resolve sole
  driver → sweep balances → `openDay`); `autoLoadPurchaseToSoleDriver` helper +
  call it from `recordTankPurchase` / `recordItemPurchase`.
- `src/modules/inventory/types.ts` — quick-open has an empty/minimal body (maybe
  optional `date` for tests); no new purchase fields.
- `src/modules/inventory/index.ts` — ensure the module factory has the catalog
  store-assignment read available (deps), if not already.
- `src/modules/inventory/__tests__/*.ts` — new quick-assignment test file.

Read-only context (no change):

- `src/modules/catalog/service.ts` — `listStoreAssignments` /
  `listStoreAssignmentDetails` (sole-driver resolution).
- `src/modules/inventory/repository.ts` — `findOrCreateTankHolder`,
  `tankBalancesByLocation`, `findAssignmentByDay`, `insertTankTransaction`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-04] [lpg-backend] Backend track DONE — all 6 backend criteria met; independent validation GREEN (no bugs); gates green (typecheck + `biome check` + **162** tests + build). **No schema/migration change.**

**What shipped**

- **Quick-open endpoint** `POST /api/v1/inventory/stores/:storeId/quick-open` (routes.ts), guarded by `canPurchase` (operator/admin; developer auto-passes; **delivery → 403**). Returns `201 { assignment }` (the `AssignmentView`, mirroring `POST /assignments`). Empty body.
- **`quickOpen(storeId, caller)`** (service.ts): own-store scope check FIRST (mirrors `resolvePurchaseActor` — operator `storeIdsForUser` must include the store, else `ForbiddenError` **403**) → resolve sole driver → sweep `tankBalancesByLocation` (full>0 || empty>0, **both** legs) + `itemBalancesByLocation` (qty>0) → delegate to the **existing `openDay`** (verbatim: today-only, dup-day 409, unconsolidated-prior 409, negative guard, driver push all reused). 0 drivers → 409; >1 → 409 (points to the general dialog). Resolved via `const [driver, ...rest] = …` (clean 0/1/>1 branch, no off-by-one).
- **Sole-driver resolution** — new repo read `activeDeliveryAssignmentsForStore(storeId)` (repository.ts): `store_assignments ⋈ users` where `active=true AND users.role='delivery'`, ordered by id. Kept in the inventory repo (it already imports `storeAssignments`/`users` schema — no new cross-module coupling; avoids a second source of truth vs. catalog). New `DeliveryAssignmentRow` type.
- **Item-load primitive** — new `loadItemFromLocation` (service.ts) mirroring `loadFromLocation`: paired ref-linked legs (location −qty, assignment +qty), negative-stock guard naming the item (new `inventoryItemLabel` repo read). `openDay` gained an **optional** `items` loop (`openDaySchema.items` is `.optional()`, iterated as `input.items ?? []`) — behaviorally identical when omitted, so the general dialog path is untouched. The shared `inventory_tx_kind` enum already had `load` (tank+item share it) → only `kindToItemDelta` gained a `load` case. **No migration.**
- **Purchase auto-load** — `autoLoadPurchaseToSoleDriver(r, storeId, {tanks?,items?}, userId)` (service.ts), called INSIDE both `recordTankPurchase` and `recordItemPurchase` transactions (tx-bound repo `r` → a failed transfer rolls the whole purchase back). Fires only when exactly **1** active delivery AND its **today** day is **`open`** (`findAssignmentByDay` + `state==='open'`); every other shape (0/>1 drivers, no day, closed/carried) is a **no-op** — purchase stays on location. **No auto-open.** Tanks auto-load the newly-purchased fulls (empty=0; returned empties left to the provider); items auto-load the purchased qty.
- **Tests** — new `quick-assignment.test.ts` (14 cases: sweep-all tanks+items, empty-store, 0-delivery 409, >1-delivery 409, cross-store 403, already-open 409, tank+item auto-load-when-open, no-autoload-no-day, no-autoload-closed-day, **multi-delivery skip + manual `recordLoad` still works**, + HTTP 201/403-delivery/401/409). Fake repo (`helpers.ts`): `seedStoreAssignment` gained optional `{role,active}` (default active `delivery`), + `activeDeliveryAssignmentsForStore`/`inventoryItemLabel` fakes.

**⚠️ Cross-track interaction — for the frontend track (multi-type-fulfillment "Comprar y cargar")**

Auto-load fires on **every** purchase into a single-delivery open-day store. This **supersedes the manual second step** of the multi-type "Comprar y cargar" flow: for a single-delivery store, purchasing a type the store lacks now **auto-loads it onto the driver** — the frontend's follow-up explicit `POST /assignments/:id/loads` would find an **empty location and 409**. The net inventory state (type on the truck) is already correct; the explicit load is redundant. **Frontend must**, for single-delivery stores, skip the explicit load (rely on auto-load) or ignore the resulting stock-out 409. Two backend tests (`edge-cases` "flexible loading", `multi-type-fulfillment` "provider purchase → load") were updated to assert the new auto-load behavior; `recordLoad`'s manual-success path is now covered by the new multi-delivery test.

**Decision (Open Questions, resolved with the owner at `/focus`):** quick-open sweeps **full + empty**; scope = own-store operator + admin/dev; auto-load skips closed/carried days, no auto-open; **items are in scope** (owner wants it from day 1; seeded item catalog exists) → built the item-load primitive rather than deferring.
