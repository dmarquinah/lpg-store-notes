---
project: lpg-store
domain: specs
type: spec
status: draft
depends-on: [auth/auth-foundation, users/users-crud, stores/stores-and-catalog]
last-updated: 2026-05-07
---

# Spec: Inventory Foundation (ledger-first, three-state lifecycle)

## Problem Statement

The product's core value is **traceability** of LPG tanks: who has which tanks each day, what changed during the day, what was given to or received from customers, and what is left over to carry to the next day. v1 implemented this with denormalized `assigned*`/`current*` columns, a 5-state workflow, a Strategy Pattern over five sign-formulas, and an `inventory_status_history` audit table — ~9k LOC for ~15 endpoints (see [[../../eng/legacy-bloat-analysis]]). It also missed the most common real-world wrinkle: customers who take a full tank but cannot return an empty in exchange. v1 names the case (`order_items.tank_returned`) but never propagates it into inventory or into a typed customer-side debt.

v2 needs an inventory model that:

1. Records every movement as an append-only ledger entry; never patches denormalized counters.
2. Natively expresses the eleven edge cases catalogued in [[index]] (especially E1, E3, E5 — the empty-debt cycle).
3. Preserves traceability without a parallel audit table.
4. Has a clear, additive upgrade path from "single store, dozens of transactions/day" to "many stores, high volume" without app-code churn.

## Proposed Solution

### Data model

Six tables, all append-only on the write side:

- **`inventory_assignments`** — one row per `(storeAssignmentId, date)`. State: `open` | `closed` | `carried`. Created when a driver opens their day; transitions to `closed` when day-end counts are recorded; transitions to `carried` once the next day's `open` row has been created.
- **`assignment_tanks`** — one row per `(inventoryId, tankTypeId)`. Holds catalog-level data (`purchasePrice`, `sellPrice`) only. **No `assigned*` / `current*` columns.**
- **`assignment_items`** — one row per `(inventoryId, inventoryItemId)`. Same pattern, no quantity columns.
- **`tank_transactions`** — append-only ledger. Columns: `id`, `assignmentTankId`, `fullDelta`, `emptyDelta`, `kind` (enum), `userId`, `refOrderId?`, `refCustomerId?`, `refTransactionId?` (for reversals/links), `notes?`, `occurredAt`. Indexes: `(assignmentTankId, occurredAt)` and `(refOrderId)`.
- **`item_transactions`** — append-only ledger. Columns: `id`, `assignmentItemId`, `delta`, `kind`, `userId`, `refOrderId?`, `notes?`, `occurredAt`.
- **`customer_empty_debts`** — append-only ledger keyed conceptually by `(customerId, tankTypeId)`. Columns: `id`, `customerId`, `tankTypeId`, `delta` (positive = customer owes more empties; negative = settling), `refTankTransactionId`, `occurredAt`.

### Derived views

Two SQL views hide the SUM and present a normal-table API to callers:

```sql
CREATE VIEW assignment_tank_balance AS
SELECT at.assignment_tank_id,
       at.inventory_id,
       at.tank_type_id,
       COALESCE(SUM(tt.full_delta),  0) AS current_full_tanks,
       COALESCE(SUM(tt.empty_delta), 0) AS current_empty_tanks
FROM assignment_tanks at
LEFT JOIN tank_transactions tt USING (assignment_tank_id)
GROUP BY at.assignment_tank_id, at.inventory_id, at.tank_type_id;

CREATE VIEW assignment_item_balance AS
SELECT ai.assignment_item_id,
       ai.inventory_id,
       ai.inventory_item_id,
       COALESCE(SUM(it.delta), 0) AS current_quantity
FROM assignment_items ai
LEFT JOIN item_transactions it USING (assignment_item_id)
GROUP BY ai.assignment_item_id, ai.inventory_id, ai.inventory_item_id;

CREATE VIEW customer_empty_debt_balance AS
SELECT customer_id,
       tank_type_id,
       COALESCE(SUM(delta), 0) AS empties_owed
FROM customer_empty_debts
GROUP BY customer_id, tank_type_id;
```

Per ADR-007, callers always read through the views. The view can later be promoted to a `MATERIALIZED VIEW` or replaced by a snapshot table maintained by the repo without changing call sites.

### Transaction kinds

A single enum:

```
'opening' | 'sale' | 'purchase' | 'return' | 'transfer' | 'reconciliation' | 'adjustment'
```

A pure function maps `(kind, args)` to ledger deltas:

```ts
kindToTankDelta('sale',     { qty }):                    { fullDelta: -qty, emptyDelta: 0 }       // empties accrue separately
kindToTankDelta('sale',     { qty, emptyReceived }):     { fullDelta: -qty, emptyDelta: +emptyReceived }
kindToTankDelta('purchase', { qty }):                    { fullDelta: +qty, emptyDelta: -qty }
kindToTankDelta('return',   { fullQty, emptyQty }):      { fullDelta: +fullQty, emptyDelta: +emptyQty }
kindToTankDelta('opening',  { full, empty }):            { fullDelta: +full, emptyDelta: +empty }
// transfer, reconciliation, adjustment: explicit deltas with notes
```

The service exposes one method per business operation: `recordSale`, `recordPurchase`, `recordReturn`, `recordOpening`, `recordTransfer`, `recordReconciliation`, `recordAdjustment`. No Strategy classes (per ADR-006).

### Empty-debt logic for `recordSale`

Inputs: `assignmentTankId`, `qty` (fulls out), `emptyReceived` (defaults to `qty`), `customerId?`, `orderId?`, `userId`, `notes?`.

Pseudocode:

```
emptyDebtDelta = qty - emptyReceived   // how many fewer empties came back than fulls went out
INSERT tank_transactions (fullDelta = -qty, emptyDelta = +emptyReceived, kind = 'sale', refOrderId, ...)
  RETURNING id AS txId
IF customerId IS NOT NULL AND emptyDebtDelta != 0:
  INSERT customer_empty_debts (customerId, tankTypeId, delta = emptyDebtDelta, refTankTransactionId = txId, ...)
```

Walk-in (no `customerId`): the customer-side ledger row is skipped. Inventory still records honest deltas — there is no fictitious empty.

`recordReturn` (E5: customer brings empty back later) inserts a `tank_transactions` row with `{fullDelta: 0, emptyDelta: +qty}` and, if `customerId` is given and they have outstanding debt, inserts a balancing row in `customer_empty_debts` with `delta = -min(qty, currentDebt)`.

### Lifecycle

- **`open`**: created by `recordOpening` (or auto-created from store catalog at day-start). Accepts ledger writes.
- **`closed`**: set by `closeAssignment(inventoryId, userId, finalCounts?, discrepancyNotes?)`. If `finalCounts` provided and they differ from `assignment_*_balance`, a `reconciliation` ledger row is appended with the delta and the notes — the view stays accurate without violating append-only.
- **`carried`**: set when the next day's `open` row has been created. `createNextDayAssignment(inventoryId)` is idempotent — re-running is a no-op if the next-day row exists.

Late transactions arriving after `closed` are appended as `kind = 'reconciliation'` with a note pointing to the late event. The assignment row never becomes immutable.

### Routes

Mounted at `/api/v1/inventory` from `createInventoryModule({ db, customerEmptyDebtsService? })`:

- `POST   /assignments` — open a new assignment (admin/operator only).
- `GET    /assignments` — list with filters (`storeId`, `userId`, `date`, `state`).
- `GET    /assignments/:id` — single assignment with balance view embedded.
- `POST   /assignments/:id/close` — transition to `closed`, optionally pass `finalCounts` for discrepancy detection.
- `POST   /assignments/:id/carry` — idempotently create the next day's `open`.
- `POST   /assignments/:id/sales` — record sale (calls `recordSale`).
- `POST   /assignments/:id/purchases` — record purchase.
- `POST   /assignments/:id/returns` — record return (full or empty).
- `POST   /assignments/:id/transfers` — driver-to-driver, optional in MVP.
- `POST   /assignments/:id/adjustments` — write-offs, damage.
- `GET    /assignments/:id/transactions` — flat ledger listing.
- `GET    /assignments/:id/discrepancies` — list of `kind='reconciliation'` rows.

No `?include=` query parameter (per legacy-bloat-analysis driver 10). Each route returns a canonical shape; relation loading is added when a real consumer asks.

## Acceptance Criteria

### Schema and storage

- [ ] `tank_transactions`, `item_transactions`, `customer_empty_debts` tables exist with append-only access patterns (no UPDATE / DELETE in the repo for these).
- [ ] `assignment_tanks` and `assignment_items` have **no** `assigned*` / `current*` quantity columns.
- [ ] `inventory_assignments.state` is an enum with exactly `open | closed | carried`.
- [ ] Three SQL views (`assignment_tank_balance`, `assignment_item_balance`, `customer_empty_debt_balance`) exist and return correct sums, including for assignments with no transactions (rows present, counts 0).
- [ ] Indexes: `tank_transactions(assignment_tank_id, occurred_at)`, `tank_transactions(ref_order_id)`, `customer_empty_debts(customer_id, tank_type_id)`.
- [ ] Drizzle migration files in `src/db/migrations/`; no `db:push`.

### Module structure

- [ ] `src/modules/inventory/` follows [[../../eng/patterns/module-template]] (routes, service, repository, schema, types, index).
- [ ] `repository.ts` is the only file importing `schema.ts`.
- [ ] No `I*` interface files; no DI container references (per ADR-003).
- [ ] No `dtos/` parallel tree; all types live in `types.ts` as Zod + `z.infer<>` (per ADR-004).

### Transaction logic

- [ ] `recordSale(assignmentTankId, qty, emptyReceived?, customerId?, orderId?, userId, notes?)` writes one `tank_transactions` row and, when `emptyReceived < qty` and `customerId` is given, one balancing `customer_empty_debts` row in the same DB transaction.
- [ ] `recordSale` with no `customerId` and `emptyReceived < qty` writes only the inventory ledger row; no debt row; no error.
- [ ] `recordReturn` for a customer with outstanding debt settles the smaller of `(qty, currentDebt)` against `customer_empty_debts`.
- [ ] Each of the eleven edge cases in [[index]] (E1–E11) has a passing unit test exercising the resulting ledger rows and view balances.
- [ ] No Strategy / StrategyFactory / StrategyProcessor classes (per ADR-006). The kind dispatch is a pure function.

### Lifecycle

- [ ] An `open` assignment accepts all transaction kinds.
- [ ] A `closed` assignment rejects `sale` / `purchase` / `return` / `transfer` with a 4xx error and an explanatory message; accepts `reconciliation` and `adjustment`.
- [ ] `closeAssignment` with mismatched `finalCounts` appends a `reconciliation` row with the delta and the discrepancy note; the balance view reflects it.
- [ ] `carry` is idempotent: calling it twice creates one next-day row, not two; second call returns the existing one.

### API

- [ ] All 11 routes return Zod-validated request and response shapes; routes are thin (validate → service → respond).
- [ ] Errors flow through `src/lib/errors.ts` (`AppError` + subclasses) and the central `errorHandler` middleware.
- [ ] `GET /assignments/:id` includes `currentFullTanks` / `currentEmptyTanks` per tank type and `currentQuantity` per item, sourced from the views.
- [ ] `GET /assignments/:id/discrepancies` returns the `kind='reconciliation'` ledger rows for that assignment.

### Audit

- [ ] No `inventory_status_history` table, no `audit_logs` table (per ADR-009). The transaction ledgers are the audit trail.
- [ ] Every transaction row has `userId`, `occurredAt`, and (where applicable) `refOrderId` / `refCustomerId` / `refTransactionId`.

## Out of Scope

- **Inventory reservations** — the `orders → inventory` reservation step lives in `orders/orders-foundation`. Until then, sales are recorded directly with no reservation phase.
- **Daily reports / financial close-out** — `daily_reports`-style aggregation belongs to a future spec (likely `reports/`); this spec stops at the per-assignment ledger.
- **Multi-store transfer workflows** — `transfer` kind is in the schema but the orchestration of "store A sends to store B" UX is deferred.
- **Pagination on list endpoints** — added when a consumer needs it; canonical shape returns up to a sensible default cap (likely 100).
- **Push notifications** — dropped per [[../../product/overview]] MVP scope.
- **Stale-recovery / business-day calendar** — not needed; the shop runs every day.

## Technical Notes

- The append-only constraint is enforced by the repo (no `UPDATE` or `DELETE` against the ledger tables). At the DB level we *could* add row-level security or revoke `UPDATE`/`DELETE` from the app role, but that is operations-level hardening and out of scope here.
- `transfer` ledger rows are written as a pair (one per assignment) inside one DB transaction with matching `refTransactionId` cross-pointers. If we ever need to reconcile cross-store totals, this is the join key.
- The `customer_empty_debts` aggregation view is read by both inventory (for "should I record a debt?" decisions) and customers (for "show me what this customer owes" UI). Since the query joins back to `customers` to filter active customers, the view in the customers module wraps this one.
- We do **not** add a `tank_returned` boolean on order items. Whether a return happened is implicit in `tank_transactions.emptyDelta` for the order's `refTankTransactionId` rows — single source of truth.
- View read paths are sub-millisecond at the volume we expect for the next several years. Per ADR-007, if profiling later shows a hot path, we promote `VIEW` → `MATERIALIZED VIEW` → snapshot table without changing callers.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To be created in v2:

- `src/modules/inventory/index.ts`
- `src/modules/inventory/routes.ts`
- `src/modules/inventory/service.ts`
- `src/modules/inventory/repository.ts`
- `src/modules/inventory/schema.ts` — Drizzle tables and the three views (defined as drizzle `pgView` or as raw SQL in a migration)
- `src/modules/inventory/types.ts` — Zod request/response schemas
- `src/modules/inventory/kindToDelta.ts` — pure function dispatching transaction kinds to deltas
- `src/modules/inventory/__tests__/lifecycle.test.ts`
- `src/modules/inventory/__tests__/edge-cases.test.ts` — covers E1–E11
- `src/modules/inventory/__tests__/empty-debt.test.ts`
- `src/db/schema.ts` — re-export inventory schema
- `src/db/migrations/<n>_inventory_foundation.sql` — generated by drizzle-kit; views may need a hand-written follow-up migration
- `src/app.ts` — mount `createInventoryModule({ db })` at `/api/v1/inventory`

v1 reference (read-only):

- `legacy/docs/inventory/PRD - Inventory.md` — requirements
- `legacy/src/db/schemas/inventory/` — entity shapes (do **not** copy the `assigned*`/`current*` columns)
- `legacy/src/services/inventory/inventoryAssignmentService.ts` — flow reference (do **not** copy the dual-write pattern)
- `legacy/src/services/inventory/inventoryDateService.ts` — *anti-reference*: do not port the smart-date logic
- `legacy/src/strategies/transactions/` — *anti-reference*: do not port the Strategy Pattern
- `legacy/src/repositories/inventory/consolidationWorkflow.ts` — *anti-reference*: the 466 LOC of consolidation contortions disappear with ledger-first storage
- `legacy/src/db/schemas/customers/customer-debts.ts` — *anti-reference*: the v1 monetary-only debt model that does not represent typed empty-tank debt

## Implementation Notes

(empty — populated as the spec is implemented)
