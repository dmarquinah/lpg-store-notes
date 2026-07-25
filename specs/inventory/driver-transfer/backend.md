---
project: lpg-store
domain: specs
type: spec-track
spec: driver-transfer
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-20
---

# Driver Transfer — lpg-backend track

Shared spec: [[index]]

## Technical Notes

Everything is in the **inventory** module + a small **orders** re-home path,
wired through an injected port (ADR-012). Admin/dev only.

### 1. Reactivate-or-create the link (catalog)

`catalog.createStoreAssignment` currently 409s only on an **active** dup and
otherwise inserts a new row. Change the "add driver" path so that, when an
**inactive** `(store, user)` link exists, it is **reactivated** (`setActive
true`) and returned instead of inserting a duplicate. (Either in
`createStoreAssignment` itself — reactivate-on-conflict — or a dedicated
`addOrReactivateStoreAssignment`. Keep the plain create's 409-on-active-dup.)

### 2. Move the driver — the carry becomes a re-point (inventory)

Rework the carry so it MOVES the driver's active context instead of copying stock
and orphaning the day. In one transaction (admin/dev only, `isGlobal` guard):

- **Standing stock (pool):** move the source driver's `location` holders to the
  target link — reuse `seed-driver`'s copy+zero (pool only). **Do NOT fold the
  open day's truck into the pool** (the day moves instead).
- **Live day(s):** `findUnconsolidated(fromSA)` → for each, re-point
  `inventory_assignments.store_assignment_id = toStoreAssignmentId`. Truck holders
  (keyed by `assignment_id`) ride along — no stock movement, no re-open. New repo
  method e.g. `reassignAssignmentLink(assignmentId, toStoreAssignmentId)`.
- **Orders:** collect the moved day ids; via an injected **`OrderRehomer`** port,
  set `orders.store_id = targetStore` for every order with `assignment_id ∈` those
  days; for **assigned/in-transit** ones also append a status-history line
  ("trasladado con el repartidor") — status + `assignment_id` unchanged.
- The frontend still retires the source link afterward (catalog).

Decide: keep this in `seedDriverInventory` (carry branch) or a dedicated
`moveDriver`/`transferDriver` endpoint. Prefer a **dedicated endpoint** so
`seed-driver` stays purely the manual/onboarding seeder; the carry path routes to
the new endpoint.

### 3. Orders port (ADR-012)

New `OrderRehomer` interface implemented by `OrdersService`, injected into the
inventory service in `app.ts` (mirrors the `DriverNotifier` / accounting-guard
pattern so inventory never imports orders):

- `rehomeByAssignments(assignmentIds: number[], toStoreId: number, actorUserId):
  Promise<number>` — set `store_id` for all orders on those assignments; append
  history for non-terminal ones; return the count.

Note: the injected write isn't in the inventory transaction (separate module/db
handle) — sequence it after the inventory move commits (like the notifier), or
accept the small non-atomic window (admin-only, testing-grade). Surface a clear
error if it fails (recoverable).

### 4. Guards

Admin/developer only: route `requireRole('admin')` (developer passes the
middleware; operator/delivery rejected) + a defensive `isGlobal(caller.role)`
check in the service (403 otherwise). Validate target link is an active delivery
link of the target store; source link is an active delivery link of its store.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/service.ts` — the carry → re-point move (pool move + day
  re-point + orders port call); split from / alongside `seedDriverInventory`.
- `src/modules/inventory/repository.ts` — `reassignAssignmentLink`
  (`inventory_assignments.store_assignment_id`), + the fake.
- `src/modules/inventory/routes.ts` — the move-driver endpoint (admin-only), or
  the carry mode on `seed-driver`.
- `src/modules/inventory/types.ts` — request/response shapes.
- `src/modules/orders/service.ts` — `rehomeByAssignments`; `src/modules/orders/
  repository.ts` — list-by-assignments + `store_id` update + history; + fake.
- `src/modules/catalog/service.ts` — reactivate-or-create the link.
- `src/app.ts` — wire the `OrderRehomer` port into the inventory service.
- `__tests__` — day re-point moves orders' store; status preserved for
  assigned/in-transit; link reactivation; admin-only 403.

Read-only context:

- `src/modules/inventory/service.ts` `seedDriverInventory` (the current carry to
  supersede), `findUnconsolidated`, `activeDeliveryAssignmentsForStore`.
- `src/modules/orders/service.ts` `transferOrder` (existing re-home that reverts
  to pending — the contrast: this new path preserves status), `transition`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-20] [lpg-backend] Backend track implemented — move-as-re-point + reactivate-or-create link + order-rehome port. All gates green (typecheck ✓ · check ✓ · **test ✓ 212** (was 207) · build ✓).

- **Dedicated `move-driver` endpoint (not a seed-driver mode).** New `POST /inventory/stores/:storeId/move-driver` → `InventoryService.transferDriver(targetStoreId, { fromStoreAssignmentId, toStoreAssignmentId }, caller)`, admin/dev only (route `canManageDrivers` = `requireRole('admin')` + defensive `isGlobal` → 403). In one inventory tx it (1) validates target link is active-delivery of `:storeId` and source link is active-delivery of its store (409 each), (2) **moves the standing pool** via new private `movePool` (add source levels onto target, then zero source — a true move, pool only, no trucks), (3) **re-points** each `findUnconsolidated(fromSA)` day with new repo `reassignAssignmentLink(assignmentId, toStoreAssignmentId)` (truck holders keyed by `assignment_id` ride along; `findAssignmentByDay(target, date)` pre-check → clean 409 instead of a raw `uq_inventory_assignment_day` collision), collecting moved day ids. After the tx commits, calls the injected `OrderRehomer` with the moved day ids + target store.
- **`seedDriverInventory` is now MANUAL-only** — the `fromStoreAssignmentId` carry branch (+ its schema field and the two related refines) removed; onboarding a fresh driver at absolute levels is unchanged. `seedDriverSchema` trimmed; new `moveDriverSchema` (`{ fromStoreAssignmentId, toStoreAssignmentId }`, distinct).
- **Order re-home port (ADR-012).** New `OrderRehomer` type + `setOrderRehomer` on `InventoryService` (mirrors `setDriverDayMoneyResolver`). Implemented by `OrdersService.rehomeByAssignments(assignmentIds, toStoreId, actorUserId)`: new orders repo `moveOrdersToStoreByAssignments` (single UPDATE … WHERE `assignment_id` IN ids AND `store_id != target` RETURNING id+status) sets `store_id`; the service appends a `"trasladado con el repartidor"` status-history line (from=to, unchanged) for **non-terminal** orders only (`TERMINAL = delivered|failed|cancelled` move silently); returns the count. Wired in `app.ts` next to `setDriverDayMoneyResolver`. **Confirmed** `grep 'modules/orders' src/modules/inventory/` is empty — inventory never imports orders.
- **Reactivate-or-create link (catalog).** `catalog.createStoreAssignment` still 409s on an active dup, but reactivates an existing **inactive** `(store,user)` link (new repo `findInactiveAssignment`, most-recent by id) instead of inserting a duplicate — so round-trips don't accumulate `inactive+active` pairs.
- **Tests (+5 net).** `driver-pools.test.ts`: replaced the 2 carry-via-seed-driver tests with 4 move-driver tests (re-points the live day cross-store — truck rides along, source store total → 0; injected `OrderRehomer` called with `[dayId]`+target; pool-only move when no live day, no port call; operator → 403). `orders.test.ts`: `rehomeByAssignments` moves store_id for all orders on the moved days, status + assignment_id preserved, history line for non-terminal only, delivered/failed/cancelled silent; empty-ids no-op. `catalog.test.ts`: re-adding a driver reactivates the same link id (single row, no duplicate); active-dup still 409.
- **ADR:** added **ADR-023** to `eng/decisions.md` (move-as-re-point + OrderRehomer port + reactivate-or-create; supersedes the seed-driver carry).

[2026-07-20] [lpg-backend] **Follow-up fix (found in end-to-end testing on the real DB).** The first cut re-homed orders only on **re-pointed unconsolidated days** and required an **active** source link — but real duplicate-store leftovers (e.g. order #24, `in_transit`) sit on an **inactive** link whose days are all **carried**, so nothing moved. Reworked per owner decisions (scope = undelivered only; re-pair rather than leave assignment-less):
- **`transferDriver` now accepts an INACTIVE source link** (source only has to exist; schema keeps source ≠ target; target still an active delivery link).
- **New re-pair path** (`OrderRepairer` port → `orders.repairUndeliveredOrders`): the driver's **undelivered** orders on the source's **finished (carried)** days move to the target store **and rebind** to a live target day (target link's open day today, found or created) → status `assigned`, so they're deliverable again. Only undelivered move; delivered/cancelled stay. Ride-along path (open/closed days) unchanged. `transferDriver` resolves/creates the target day (inventory owns days), passes its id to the port (orders owns the rebind) — ADR-012 respected. New orders repo `undeliveredOrdersByAssignments`.
- **ADR-023 updated** (two order paths + inactive source). +3 tests (215 total): inventory inactive-source re-pair (fake `OrderRepairer` called with the carried day + created target day); orders `repairUndeliveredOrders` (rebind → assigned + history, terminal untouched; null-target store-only fallback). Gates: typecheck ✓ · check ✓ · **test ✓ 215** · build ✓.
- **Frontend follow-up needed** (frontend track): the "Agregar repartidor" picker currently only offers a driver's **active** link as the source; to consolidate a defunct store's leftovers it must let the admin pick the **old/inactive** source link (or a dedicated "consolidate leftovers" entry point). Until then the endpoint is reachable only by direct API call.

[2026-07-21] [lpg-backend] **Fix: re-pair now names the target driver on the reassignment event.** The order detail derives "Repartidor" from the LAST `assigned` status-history event's `targetUserName` (frontend `OrderDetailView` `driverName`). The first re-pair cut left `target_user_id` null on that event → the order showed `Asignado` but "Repartidor: Sin asignar" (deliverability was fine — that's via `assignment_id`; only the label/timeline were blank). Threaded the target link's `userId` through: `OrderRepairer` port gains a `targetUserId` param, `transferDriver` passes `target.userId`, and `repairUndeliveredOrders` sets it on the history line (mirrors `assignOrder`). +assert in both tests; gates green (215). Patched the 16 already-applied rows (`target_user_id = 4` / Juan) so #24 + the batch now name the driver.
