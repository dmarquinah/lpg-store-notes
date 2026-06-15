---
project: lpg-store
domain: specs
type: spec-track
spec: inventory-foundation
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-05
---

# Inventory Foundation — lpg-backend track

Shared spec: [[index]] · Edge-case catalogue (E1–E11): [[../index]]

## Technical Notes

### Data model

Six tables (count unchanged from the original draft — the assignment catalog tables **generalize into holder tables** so a location can hold stock too; ADR-013). All append-only on the write side.

**Holders** — *what owns inventory*: a location's standing stock, or a driver's day.

- **`inventory_assignments`** — one row per `(storeAssignmentId, date)`. The driver-day lifecycle row. State `open | closed | carried`. Created when a driver opens their day; → `closed` when day-end counts are recorded; → `carried` once the next day's `open` row exists.
- **`tank_holders`** — one row per holder × tank type. `id`, `kind` (`'location' | 'assignment'`), `storeId?` (when `kind='location'`), `assignmentId?` (when `kind='assignment'`, FK → `inventory_assignments`), `tankTypeId`, `purchasePrice`, `sellPrice`. **No `assigned*`/`current*` columns.** CHECK: exactly one of `storeId`/`assignmentId`. Partial unique indexes `(storeId, tankTypeId) WHERE kind='location'` and `(assignmentId, tankTypeId) WHERE kind='assignment'`.
- **`item_holders`** — same shape for non-tank items: `id`, `kind`, `storeId?`, `assignmentId?`, `inventoryItemId`, prices. No quantity columns.

**Ledgers** — append-only, a single clean `holderId` FK.

- **`tank_transactions`** — `id`, `holderId` (FK → `tank_holders`), `fullDelta`, `emptyDelta`, `kind` (enum), `userId`, `refOrderId?`, `refCustomerId?`, `refTransactionId?` (pairs the two legs of a transfer), `notes?`, `occurredAt`. Indexes `(holderId, occurredAt)`, `(refOrderId)`, `(refTransactionId)`.
- **`item_transactions`** — `id`, `holderId` (FK → `item_holders`), `delta`, `kind`, `userId`, `refOrderId?`, `refTransactionId?`, `notes?`, `occurredAt`.

**Customer side** (unchanged, ADR-010).

- **`customer_empty_debts`** — append-only ledger keyed by `(customerId, tankTypeId)`: `id`, `customerId`, `tankTypeId`, `delta` (+ owes more empties; − settling), `refTankTransactionId`, `occurredAt`.

**Extensions** (ADR-013 upgrade path): when a holder kind grows its own attributes or a new kind appears, add a **side table keyed by the row id** (`*_holder_details(holder_id PK)`, or transaction metadata keyed by `tank_transactions.id`) — never new columns on the ledger. The hot ledger's shape stays frozen.

### Derived views

A **single** balance view per entity — the unified holder pays off here: one view answers every level. Per ADR-007 callers always read through the view; it can later become a `MATERIALIZED VIEW` or snapshot table without changing call sites.

```sql
CREATE VIEW tank_balance AS
SELECT h.id AS holder_id, h.kind, h.store_id, h.assignment_id, h.tank_type_id,
       COALESCE(SUM(tt.full_delta),  0) AS current_full_tanks,
       COALESCE(SUM(tt.empty_delta), 0) AS current_empty_tanks
FROM tank_holders h
LEFT JOIN tank_transactions tt ON tt.holder_id = h.id
GROUP BY h.id;
-- item_balance follows the same shape; customer_empty_debt_balance per ADR-010
```

Same view, different filter:

- **Driver's current level / day's delta** → `WHERE assignment_id = :id`
- **Location availability (operator)** → `WHERE kind='location' AND store_id = :store` (shop stock); optionally `+` that store's open assignments for stock currently on trucks.

### Transaction kinds

A single enum: `'opening' | 'load' | 'sale' | 'purchase' | 'return' | 'transfer' | 'reconciliation' | 'adjustment'`. (`opening` = the day's initial assignment; `load` = a mid-day fill-up onto the truck — same delta math, kept distinct in the ledger so a refill isn't mislabeled as an opening.) A pure function maps `(kind, args)` to deltas (no Strategy classes, per ADR-006):

```ts
kindToTankDelta('sale',     { qty }):                { fullDelta: -qty, emptyDelta: 0 }
kindToTankDelta('sale',     { qty, emptyReceived }): { fullDelta: -qty, emptyDelta: +emptyReceived }
kindToTankDelta('purchase', { qty }):                { fullDelta: +qty, emptyDelta: -qty }
kindToTankDelta('return',   { fullQty, emptyQty }):  { fullDelta: +fullQty, emptyDelta: +emptyQty }
kindToTankDelta('opening',  { full, empty }):        { fullDelta: +full, emptyDelta: +empty }
// transfer, reconciliation, adjustment: explicit deltas with notes
```

`opening` is the day-open transfer **in** (onto an `assignment` holder); the day-close hand-back is a `transfer` **out** (assignment → location). Because `opening` and `transfer` are distinct kinds, the morning load and the evening hand-back are separable by a plain `WHERE kind` — which is what makes per-day reconstruction a filter, not a special case. The service exposes one method per business operation: `recordSale`, `recordPurchase`, `recordReturn`, `recordOpening`, `recordTransfer`, `recordReconciliation`, `recordAdjustment`.

### Empty-debt logic for `recordSale`

Inputs: `assignmentTankHolderId`, `qty` (fulls out), `emptyReceived` (defaults to `qty`), `customerId?`, `orderId?`, `userId`, `notes?`.

```
emptyDebtDelta = qty - emptyReceived
INSERT tank_transactions (holderId, fullDelta = -qty, emptyDelta = +emptyReceived, kind = 'sale', refOrderId, ...) RETURNING id AS txId
IF customerId IS NOT NULL AND emptyDebtDelta != 0:
  INSERT customer_empty_debts (customerId, tankTypeId, delta = emptyDebtDelta, refTankTransactionId = txId, ...)
```

Walk-in (no `customerId`): the customer-side row is skipped; inventory still records honest deltas. `recordReturn` (E5) inserts `{fullDelta: 0, emptyDelta: +qty}` on the assignment holder and, if the customer has outstanding debt, a balancing `customer_empty_debts` row with `delta = -min(qty, currentDebt)`.

### Per-user / per-date reconstruction

Each user-day is its own `assignment` holder and the ledger is immutable, so any past day reads back exactly — no snapshots. For driver X on date D, anchor on the assignment (`store_assignments.user_id = X AND inventory_assignments.date = D`) and read its holder's ledger:

- **Opening level** — `SUM` where `kind = 'opening'`
- **Loaded mid-day** — `SUM` where `kind = 'load'` (fill-ups after the day opened)
- **The day's deltas** — rows where `kind IN ('sale','return','adjustment')`, ordered by `occurredAt`
- **Expected end-of-day** — `SUM` of all rows except the closing hand-back (`kind <> 'transfer'`)
- **Physical end-of-day + discrepancy** — the `reconciliation` row
- **Handed back to shop** — the closing `transfer` magnitude

Surfaced by `GET /assignments/:id` (embeds these) and `GET /assignments?userId=&date=`. Admin analytics is the same shape `GROUP BY` assignment/date/user. Derive via the view; materialize only if profiled (ADR-007). This is what lets the driver self-evaluate any day and admins run cross-driver analytics over the same ledger.

### Lifecycle

- **`open`**: `openDay(storeAssignmentId, date?, tanks?)` creates the `inventory_assignments` row (`date` defaults to the **Lima business date** via the centralized date service) and writes an **opening transfer** per tank loaded — paired `{−}` on the `location` holder / `{+}` on the `assignment` holder (`kind='opening'`), same `refTransactionId`, one DB transaction. `tanks` may be **empty** (start with nothing); `recordLoad(assignmentId, {tankTypeId, full, empty})` assigns stock from the location **mid-day** as a distinct `load` transfer (recorded separately from the day's `opening`, but still counted as received in reconstruction), so a purchase can be handed to a driver as it arrives. Accepts ledger writes while `open`.
- **`closed`**: `closeAssignment(assignmentId, userId, finalCounts?, notes?)` appends a `reconciliation` row for any mismatch between `finalCounts` and the view (delta + notes), then marks the assignment `closed`. *(Implementation: the hand-back to the location happens at **carry**, not at close — keeps close a pure reconcile + state change.)*
- **`carried`**: `carryAssignment(assignmentId)` **consolidates** the day — it hands the truck's closing leftovers back to the `location` holder (paired `transfer` rows) and marks the assignment `carried`. It does **not** open the next day: that's an explicit `openDay` decision made after the operator **restocks** the location from the provider, so next-day opening is free to differ from today's closing (**ADR-014**). Idempotent. A late `adjustment` after `closed`/`carried` is still accepted; the row never becomes immutable.

### Routes (mounted at `/api/v1/inventory`)

**Assignment (driver-day):** `POST /assignments` (`tanks` optional → open empty; `date` defaults to Lima business date), `GET /assignments` (filters: `userId`, `date`, `storeId`, `storeAssignmentId`, `state`), `GET /assignments/:id` (embeds balances), `POST /assignments/:id/loads` (assign stock mid-day), `POST /assignments/:id/{sales,returns,adjustments}`, `POST /assignments/:id/close`, `POST /assignments/:id/carry`, `GET /assignments/:id/transactions`, `GET /assignments/:id/discrepancies`, `GET /history?userId=&date=` (per-user/date reconstruction). *(No per-assignment purchase/transfer endpoints — purchases are location-scoped; cross-store transfer deferred.)*

**Location stock (operator/admin):** `GET /stores/:storeId/availability` — current `location`-holder balance (shop), `?includeTrucks=1` adds open assignments. `POST /stores/:storeId/tank-purchases` (`{ items: [{ tankTypeId, qty, emptyReturned?, surcharge? }], notes? }` — `emptyReturned` defaults to the largest swap the shop can honor and is capped at `qty`/availability; `surcharge` is the extra paid for the shortfall, stored in `purchase_surcharges`) and `POST /stores/:storeId/item-purchases` (`{ items: [{ inventoryItemId, qty }], notes? }`) — supplier restock onto the `location` holder. **Separate endpoints** because tanks are a full-for-empty exchange (`tank_transactions`) while articles are a plain stock add (`item_transactions`), and they're distinct supplier events. Both batched in one DB transaction; each line a **unique product** (duplicate → 400).

No `?include=` parameter (legacy-bloat driver 10).

### Design constraints

- A holder is `location` **or** `assignment`, never both (CHECK). The ledger references a single `holderId`; future holder kinds or kind-specific attributes are **side tables keyed by `holderId`/`transactionId`**, never new ledger columns (ADR-013) — the hot ledger's shape is frozen.
- Append-only enforced by the repo (no `UPDATE`/`DELETE` on ledger tables). `opening`/`transfer` write as a **pair** inside one DB transaction with matching `refTransactionId`.
- Cross-module writes (e.g. an order's delivery committing inventory) are explicit service calls inside one `db.transaction()`, not events (ADR-012).
- No `tank_returned` boolean on order items — whether a return happened is implicit in `tank_transactions.emptyDelta`.
- View reads are sub-millisecond at expected volume; promote to materialized/snapshot only if profiling shows a hot path (ADR-007).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To be created:

- `src/modules/inventory/{index,routes,service,repository,schema,types}.ts`
- `src/modules/inventory/kindToDelta.ts` — pure kind→delta dispatch
- `src/modules/inventory/__tests__/{lifecycle,edge-cases,empty-debt,history}.test.ts` (edge-cases covers E1–E11; history covers per-user/per-date reconstruction)
- `src/db/schema.ts` — re-export inventory schema
- `src/db/migrations/<n>_inventory_foundation.sql` (holder/ledger tables + a hand-written follow-up migration for the balance views)
- `src/app.ts` — mount `createInventoryModule({ db })` at `/api/v1/inventory`

v1 reference (read-only / anti-reference): `legacy/docs/inventory/PRD - Inventory.md` (requirements); `legacy/src/db/schemas/inventory/` (shapes — do **not** copy `assigned*`/`current*`); `legacy/src/services/inventory/inventoryAssignmentService.ts` (flow, not the dual-write); `legacy/src/services/inventory/inventoryDateService.ts`, `legacy/src/strategies/transactions/`, `legacy/src/repositories/inventory/consolidationWorkflow.ts` (anti-references); `legacy/src/db/schemas/customers/customer-debts.ts` (v1 monetary-only debt — does not represent typed empty-tank debt).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
[2026-06-05] [lpg-backend] Implemented `src/modules/inventory/`: unified `tank_holders`/`item_holders` (`location|assignment`, CHECK + partial-unique per kind), append-only `tank_transactions`/`item_transactions`/`customer_empty_debts`, three SQL balance views (`tank_balance`/`item_balance`/`customer_empty_debt_balance` — hand-written `CREATE VIEW` in migration `0002`, declared `.existing()` in schema), pure `kindToDelta` dispatch (ADR-006), three-state lifecycle, atomic multi-row ops via `repo.transaction()` (ADR-005/010/012). Routes at `/api/v1/inventory` (open/sale/return/adjustment/close/carry, GET balances/transactions/discrepancies, `/history` reconstruction, `/stores/:id/availability`, `/stores/:id/purchases`, `/customers/:id/empty-debts`).

**Implementation choices vs. the original notes:** purchases are **location-scoped only** (no assignment purchase endpoint), per ADR-013; the cross-store transfer **endpoint** is deferred (the `transfer` kind is used internally for the hand-back); the day-close **hand-back happens at carry** (close stays a pure reconcile+state-change); **carry consolidates only** — it returns leftovers to the location and does **not** auto-open the next day, so next-day opening is an explicit, restockable `openDay` decision that may differ from today's closing (**ADR-014**); `refTransactionId` pairing is **unidirectional** (second leg → first leg) to keep the ledger strictly append-only; `customerId`/`refOrderId`/`refCustomerId` are **soft int refs** until customers/orders land.

**Date handling:** all business-day logic goes through `src/lib/date.ts` (`businessToday`/`toBusinessDate`/`addDays`/`isBusinessDate`, `BUSINESS_TZ='America/Lima'`, GMT-5). `openDay` defaults the assignment `date` to the Lima day, so a late-evening delivery never lands on the wrong calendar day. Unit-tested at the UTC/Lima boundary (`src/lib/__tests__/date.test.ts`).

**Flexible loading:** opening with no tanks is allowed; `recordLoad` assigns stock from the location to a driver at any point while `open` (supports "no upfront assignment; load as purchases arrive"). The truck is **not** loaded between days — `carry` returns everything to the location.

**Tests (22):** `kindToDelta`, E1–E11 + E3b (over-return debt cap), lifecycle (HTTP open→sell→close→carry + auth/state guards), per-user/date history. Independent validation: all 15 backend criteria met; fixed one ledger-correctness bug (capped negative customer debt on a sale over-return). typecheck / lint / test (31 total) / build all green; tables + views applied to the dev DB and the three views smoke-verified (correct sums incl. zero-rows) inside a rolled-back transaction.

[2026-06-06] [lpg-backend] `POST /stores/:storeId/purchases` now takes a **batch**: `{ items: [{ tankTypeId, qty }], notes? }` — one supplier delivery can restock several tank types, all in one DB transaction; a Zod refine rejects a duplicate `tankTypeId` within a purchase (400). Response is `{ transactions: [...] }`. Also added `GET /api/v1/catalog/store-assignments` (in the catalog module) so the open-day form can offer a real driver picker. 38 tests; typecheck/lint/build green.

[2026-06-06] [lpg-backend] Split purchases by product class (distinct supplier events / mechanics): `POST /stores/:storeId/tank-purchases` (full-for-empty → `tank_transactions`) and `POST /stores/:storeId/item-purchases` (plain +qty → `item_transactions`). Both batch multiple lines in one DB transaction, each line a unique product (duplicate → 400). `recordPurchase` → `recordTankPurchase` + new `recordItemPurchase`. 39 tests. (Note: a location **item**-availability read isn't exposed yet — only tank availability; add when the UI needs it.)

[2026-06-06] [lpg-backend] Added endpoint validations (ADR-015): **(1) no over-assignment** — `openDay`/`recordLoad` reject moving more fulls/empties to a truck than the shop holds (409), so location stock never goes negative; **(2) purchase swap bounded** — `emptyReturned` per line defaults to `min(qty, emptiesOnHand)`, capped at qty/availability (no negative empties), with an optional `surcharge` for the shortfall persisted in a new `purchase_surcharges` side table (migration `0003`, ADR-013 pattern); **(3) open-day today-only** — assignment date must equal today's Lima date (past/future → 400). `InventoryService` now takes an injectable `today()` (Lima date provider) so multi-day tests stay deterministic. Edge-case tests reworked to stock the shop before opening + use an injected clock. 44 tests; typecheck/lint/build green; migration `0003` applied to dev DB.

[2026-06-07] [lpg-backend] Added a distinct `load` transaction kind so a mid-day fill-up onto the truck is no longer mislabeled as the day's `opening` ("Apertura") in the movements/ledger. `openDay` writes `opening`; `recordLoad` writes `load` (same delta math, separate kind). Day-reconstruction now reports `loadedFull/Empty` separately from `openingFull/Empty` (both still count toward expected end-of-day). Migration `0004` (`ALTER TYPE inventory_tx_kind ADD VALUE 'load'`). 44 tests green. Note: the assignment `state` field (`open|closed|carried`) is unchanged and correct — a load doesn't change lifecycle state.