---
project: lpg-store
domain: specs
type: spec-track
spec: driver-inventory-pools
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-18
---

# Per-Driver Inventory Pools — lpg-backend track

Shared spec: [[index]]

## Technical Notes

The whole change is in the **inventory** module + one migration. The `assignment`
holder (the driver-day truck) is **already** per-driver; only the `location`
holder (the store floor) is shared and needs re-keying.

**Resolved model (see [[index]] → Resolved Decisions):** the location holder is
**two-tier** — a nullable `store_assignment_id` (NULL = store **parking pool**,
set = **driver pool**) beside the **retained, denormalized `store_id`**. Store
total = sum of both tiers. Migration is **non-destructive** (0-driver → parking,
1-driver → backfill onto driver, >1 → abort).

### 1. Schema — re-key the `location` holder

In `src/modules/inventory/schema.ts`, `tank_holders` and `item_holders`:

- Add `store_assignment_id integer references store_assignments(id)`.
- `location` holders: **`store_id` retained + denormalized** (resolved: yes),
  plus a **nullable** `store_assignment_id`.
  - **Owner check** becomes:
    `location` → `store_id NOT NULL AND assignment_id NULL` (`store_assignment_id`
    free: NULL = parking pool / set = driver pool);
    `assignment` → `assignment_id NOT NULL AND store_id NULL AND
    store_assignment_id NULL` (unchanged otherwise).
  - **Two partial unique indexes** replace `uq_tank_holder_location`:
    `(store_id, tank_type_id)` where `kind='location' AND store_assignment_id IS
    NULL` (parking), and `(store_assignment_id, tank_type_id)` where
    `kind='location' AND store_assignment_id IS NOT NULL` (driver). Item mirror on
    `inventory_item_id`.
  - Balance views `tank_balance` / `item_balance` add a `store_assignment_id`
    column (recreated in the migration).

`assignment` holders: **no change** (still `assignment_id`, keyed by the
driver-day).

### 2. Migration `0019` — backfill

`npm run db:generate -- driver-scoped-location-holders`, then hand-verify the
generated SQL and add the backfill (drizzle won't infer it):

1. Add `store_assignment_id` **nullable**.
2. **Pre-check query** (ship as a runbook snippet, run on prod first): every store
   that owns a `location` holder with `full>0 || empty>0` (or item qty>0) whose
   count of **active delivery** `store_assignments` ≠ 1 → must be resolved before
   step 3. (Active delivery = `store_assignments.active AND users.role='delivery'`,
   the same predicate as `activeDeliveryAssignmentsForStore`.)
3. **Backfill**: `UPDATE …_holders SET store_assignment_id = (the store's sole
   active delivery store_assignment)` for `kind='location'`. Expected 1:1 for
   every real store (the whole premise — today each store runs one driver).
4. Set `store_assignment_id NOT NULL` for location rows; swap the unique index +
   owner check constraint; (drop `store_id` from the check only if we choose not
   to denormalize — see Open Question).

Resolution rules (resolved at `/focus`): **0 delivery** → **keep as parking
pool** (`store_assignment_id` NULL — non-destructive, no block); **1 delivery**
→ backfill onto that driver's store-assignment; **>1 delivery on a single real
store** → **abort, fail loud** with a report (prod has none). Duplicate stores
stay separate (deferred merge).

Store-total balance reads must now **aggregate** across a store's location
holders (`SUM ... GROUP BY store_id, {type}`) since a store can own >1 (parking +
drivers); per-driver reads add a `...ByStoreAssignment` method. Egress + history
queries are **unchanged** — they already filter location holders by `store_id`.

### 3. Repository — reads become store-assignment-scoped, store totals aggregate

`src/modules/inventory/repository.ts`:

- `tankBalancesByLocation(storeId)` / `itemBalancesByLocation(storeId)` — split
  into a **per-driver** read (`…ByStoreAssignment(storeAssignmentId)`) + a
  **store aggregate** that sums over the store's active delivery assignments. The
  Resumen availability + store detail use the aggregate; quick-open uses the
  per-driver read.
- `findOrCreateTankHolder` / `findOrCreateItemHolder` — the `location` branch now
  keys on `store_assignment_id` (callers pass the resolved driver).
- `activeDeliveryAssignmentsForStore(storeId)` — **already exists** (added in
  [[../quick-assignment/backend]]); reuse it for sole-driver resolution + the
  aggregate's assignment set.

### 4. Service — attribute to a driver

`src/modules/inventory/service.ts`:

- `quickOpen(storeId, caller, { storeAssignmentId? })` — resolve the target
  driver (sole when omitted; the named one for multi-driver; 0 → 409), sweep
  **that assignment's** location pool → `openDay`. The existing `openDay` /
  `loadFromLocation` / `loadItemFromLocation` transfer is reused verbatim.
- `recordTankPurchase` / `recordItemPurchase` / adjustments — accept an optional
  target `storeAssignmentId` (sole-resolve when omitted), write the `location`
  legs onto **that driver's** holder.
- `autoLoadPurchaseToSoleDriver` → **`autoLoadPurchaseToDriver`** — auto-load
  onto the **attributed** driver's open day (now works for multi-driver, since the
  purchase names the driver). Same-tx, same skip-when-not-open rule.
- `availability` aggregate endpoint — sum per-driver pools per store.

### 5. Routes / types

`routes.ts`: quick-open / purchase / adjust bodies gain an optional
`storeAssignmentId` (or `driverId`). `types.ts`: request shapes + a per-driver
breakdown in the store-detail response. Scope guards (own-store operator +
admin/dev) unchanged.

### Reuse / don't reinvent

- `openDay`, `loadFromLocation`, `loadItemFromLocation`,
  `activeDeliveryAssignmentsForStore`, `findAssignmentByDay` — all present.
- Errors: `src/lib/errors.ts`. Business dates via `src/lib/date.ts` (America/Lima).

### ADR

Append an ADR to the vault `eng/decisions.md` superseding **ADR-013**'s
store-level location holder: location holders are per **delivery store-assignment**;
restate **E6 / E8 / E10** (per the category index [[../index]] edge-case list) in
per-driver terms. E1–E5, E7, E9, E11 unaffected.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/schema.ts` — re-key `location` holders (owner check,
  unique indexes, `store_assignment_id`).
- `src/db/migrations/0019_driver-scoped-location-holders.sql` — generated +
  hand-added backfill; pre-check query in the runbook.
- `src/modules/inventory/repository.ts` — location reads/creates → store-assignment;
  per-driver + store-aggregate balance reads.
- `src/modules/inventory/service.ts` — `quickOpen` driver target, purchase/adjust
  attribution, `autoLoadPurchaseToDriver`, availability aggregate.
- `src/modules/inventory/routes.ts` — `storeAssignmentId` on quick-open / purchase
  / adjust.
- `src/modules/inventory/types.ts` — request shapes + per-driver store-detail
  breakdown.
- `src/modules/inventory/__tests__/*` — update keying; add multi-driver sweep,
  purchase attribution, migration-resolution (0/1/>1) cases.

Read-only context:

- `src/modules/catalog/schema.ts` — `store_assignments` (`storeId`, `userId`,
  `active`; role via `users`).
- `src/modules/inventory/schema.ts` — `holderKindEnum`, `tankHolders` /
  `itemHolders` owner check + unique indexes (the constraints being changed).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-17] [lpg-backend] Backend track implemented (two-tier model per resolved decisions).

- **Schema** (`schema.ts`): added nullable `store_assignment_id` (FK → `store_assignments`) to `tank_holders` + `item_holders`; retained denormalized `store_id`. Owner check for `location` → `store_id NOT NULL AND assignment_id NULL` (store_assignment_id free); `assignment` → `assignment_id NOT NULL AND store_id NULL AND store_assignment_id NULL`. Replaced the single location unique with two partial uniques (`…_location_parking` on `(store_id, {type})` where store_assignment_id IS NULL; `…_location_driver` on `(store_assignment_id, {type})` where NOT NULL). Added `store_assignment_id` to the `tank_balance`/`item_balance` view column decls.
- **Migration `0019_driver-scoped-location-holders.sql`**: non-destructive. Prepended a `DO $$ … RAISE EXCEPTION $$` pre-check that aborts if any store holds nonzero location stock (tank or item, via the balance views) with >1 active delivery driver. Generated DDL (add col, FKs, two partial uniques, swapped owner checks) + hand-added the single-driver backfill (`UPDATE …_holders SET store_assignment_id = <sole active delivery SA>` for 1-driver stores; 0-driver stays NULL/parking) + DROP/CREATE both balance views with the new column. **Applied to dev DB**: 4 tank location holders all backfilled to their sole driver (0 parking), store totals intact (store 1 t1 = 14/7), views carry the column, `store_id` matches each backfilled SA's store, all pools are active-delivery — verified by query.
- **Repository (`repository.ts`)**: `HolderRef` location variant → `{ storeId; storeAssignmentId?: number|null }`; `findOrCreate{Tank,Item}Holder` match/write null-aware on `store_assignment_id`. `tankBalancesByLocation`/`itemBalancesByLocation`/`tankBalancesByLocations` now AGGREGATE (`SUM … GROUP BY`) → store totals (new narrow return types `LocationTankTotalRow`/`LocationItemTotalRow`). New per-pool reads `tank/itemBalancesForLocationPool(storeId, storeAssignmentId|null)` and per-driver breakdown `locationTankPoolsByStore` (+ `LocationTankPoolRow`). Egress + store-history reads UNCHANGED (still filter location holders by `store_id`, which every pool carries).
- **Service (`service.ts`)**: `loadFromLocation`/`loadItemFromLocation` source from the driver's own pool (`sourceStoreAssignmentId`); `openDay` passes `input.storeAssignmentId`, `recordLoad` passes `assignment.storeAssignmentId`. `quickOpen(storeId, caller, { storeAssignmentId? })` resolves the target driver (sole when omitted; named for multi; 0 → 409, unknown → 409, multi-without-target → 409) and sweeps only that driver's pool. New `resolveLocationPool` (explicit driver validated / sole active delivery / else parking-null) used by `recordTankPurchase`, `recordItemPurchase`, `recordLocationAdjustment` — target-pool availability + holder writes. `autoLoadPurchaseToSoleDriver` → `autoLoadPurchaseToDriver` (auto-loads onto the attributed driver's open day; parking → no-op). `carryAssignment` hands leftovers back to the driver's own pool (E10 per-driver). New `getLocationPoolBreakdown` (own-store/admin scope).
- **Routes/types**: optional `storeAssignmentId` on `quickOpenSchema` (new), `tankPurchaseSchema`, `itemPurchaseSchema`, `locationAdjustmentSchema`. Quick-open route parses the optional body. New `GET /stores/:storeId/pool-breakdown`.
- **Tests**: updated the in-memory fake repo (holder keying + aggregated/per-pool reads + breakdown). Fixed single-driver suites that had seeded operator SAs with the default `delivery` role (lifecycle, store-availability, store-history, provider-purchase) so those stores are genuinely single-driver; rewrote the quick-assignment multi-delivery auto-load test to the new attributed-auto-load behavior. Added `driver-pools.test.ts` (9 tests): driver-attributed vs parking purchase, store total = Σ pools, independent multi-driver sweep ("never driver B gets 0"), quick-open 409s, carry → own pool, adjustment attribution, per-driver breakdown + cross-store 403.
- **Gates**: `npm run typecheck` ✓, `npm run check` ✓, `npm test` ✓ (194 pass, was 185), `npm run build` ✓. Migration applied + verified on dev DB.

[2026-07-17] [lpg-backend] Validation passed all 8 backend acceptance criteria (independent audit), no bugs. **Follow-up noted:** parking-pool stock (`store_assignment_id NULL`) has no *direct* path onto a truck — `openDay`/`recordLoad`/quick-open all draw from a driver's own pool (intended: the quick-open determinism decision). Moving parking → a driver today takes two `adjustment` lines (parking down, driver up) or a driver-attributed purchase; a dedicated **parking→driver transfer** op is a small future add if the manual two-step proves fiddly. Backend track **done**; overall spec stays `in-progress` pending the frontend track.

[2026-07-18] [lpg-backend] Added the **late-binding pool transfer** (owner request — parked stock must be assignable to a driver quickly, as an internal movement, so both "assign at purchase" and "assign later" work).

- **`POST /stores/:storeId/pool-transfers`** → `recordPoolTransfer(storeId, input, caller)`. Body (`poolTransferSchema`): `{ toStoreAssignmentId, fromStoreAssignmentId?: number|null (null/omitted = parking), tanks?, items?, notes? }`. Moves stock as **paired `transfer` legs** (−from / +to, cross-linked by `refTransactionId`) from the parking pool (default) or another driver **into the target driver's pool**. Validates: destination is an active delivery SA of the store (else 409), source SA if given belongs to the store (else 409), source ≠ destination (schema), each product once (schema), at least one line moves something (schema), and **never drives the source pool negative** — validated per line before any write (409, nothing moved). Own-store operator / admin scope (cross-store → 403). Lands in the driver's **pool**, not their truck (composes with quick-open / mid-day load; no surprise auto-load onto an already-open day). No accounting egress. Returns the destination driver's resulting pool (`tanks`/`items`) for a UI refresh.
- This resolves the earlier follow-up (parking stock had no path to a driver): the two-step adjustment workaround is replaced by a first-class internal movement. **ADR-022 updated** with the transfer op.
- Tests: +5 in `driver-pools.test.ts` (parking→driver then quick-open sweeps it; driver→driver rebalance incl. items; over-transfer 409 nothing-moved; unknown destination 409; cross-store 403). Gates: typecheck ✓ · check ✓ · **test ✓ 199 pass** · build ✓.

[2026-07-18] [lpg-backend] Follow-on (user-directed, post-`done`): **crew open-all endpoint.** New `POST /inventory/stores/:storeId/quick-open-all` → `quickOpenAll(storeId, caller)` opens today's day for EVERY active delivery driver at once, each swept from their OWN pool. A driver already open today (pre-checked via `findAssignmentByDay`) or blocked by an unconsolidated prior day (`openDay` ConflictError caught) is **SKIPPED with a reason** rather than aborting the batch — idempotent on re-tap. 0 drivers → 409; cross-store operator → 403; the parking pool is never swept. Returns `{ opened: AssignmentView[], skipped: {storeAssignmentId, reason}[] }`. +3 tests in `driver-pools.test.ts` (open-all sweeps each own pool + parking untouched; skip-already-open + opens-the-rest + idempotent re-tap; 0-driver 409). Gates: typecheck ✓ · check ✓ · **test ✓ 202** · build ✓.

[2026-07-19] [lpg-backend] Follow-on: **add-driver / move-driver inventory** — the driver+inventory slice of the deferred "merge duplicate stores" scope. New **admin/dev-only** `POST /inventory/stores/:storeId/seed-driver` → `seedDriverInventory`: sets a target driver's location stock to an absolute level, either by **MOVING** a source link's holdings (its standing stock + **every unconsolidated day's truck** — the duplicate-store consolidation) or from **MANUAL** tank/item levels (onboarding a fresh driver).
- **Carry is a true MOVE:** after seeding the target it **ZEROES the source** (pool + each live day's truck), so total stock is conserved — no cross-store double-count once the caller retires the now-empty source link. (Independent validation caught the copy-vs-move bug; fixed here.)
- Absolute set (delta vs the target's current) → a fresh pool is seeded directly and never goes negative (levels ≥ 0).
- **Guards:** admin/dev only (route `requireRole('admin')` — developer passes, operator/delivery rejected — **plus** a defensive `isGlobal` check → 403); target must be an active **delivery** link of `:storeId` (409); source must be an active **delivery** link of its store (409); `from` XOR manual (schema).
- Owns ONLY the stock — creating the target link + retiring the source link are the caller's catalog steps.
- +5 tests in `driver-pools.test.ts` (manual seed tanks+items; carry moves pool+truck **and zeroes the source**; cross-store carry **conserves total**; operator 403; unknown target 409). Gates: typecheck ✓ · check ✓ · **test ✓ 207** · build ✓.

[2026-07-20] [lpg-backend] Reworded the day-end carry hand-back note: `cierre: devolución a tienda` → `cierre: regresa al inventario del repartidor`. With per-driver pools the truck's leftovers return to the **driver's OWN pool** (a self-transfer, E10 per-driver), not a shared store floor — the old wording was a centralized-model vestige. No behavior/ledger change: the truck→pool round-trip stays (it keeps the driver's standing stock accurate and `store total = Σ pools` clean; the owner confirmed the pool model is right). Note isn't matched anywhere in code/tests. Gates: check ✓ · test ✓ 207 · build ✓.
