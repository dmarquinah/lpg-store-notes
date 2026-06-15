---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../auth/auth-foundation/index]]"
  - "[[../../users/users-crud/index]]"
  - stores/stores-and-catalog
last-updated: 2026-06-06
---

# Spec: Inventory Foundation (ledger-first, three-state lifecycle)

## Problem Statement

The product's core value is **traceability** of LPG tanks: who has which tanks each day, what changed during the day, what was given to or received from customers, and what is left over to carry to the next day. v1 implemented this with denormalized `assigned*`/`current*` columns, a 5-state workflow, a Strategy Pattern over five sign-formulas, and an `inventory_status_history` audit table — ~9k LOC for ~15 endpoints (see [[../../../eng/legacy-bloat-analysis]]). It also missed the most common real-world wrinkle: customers who take a full tank but cannot return an empty in exchange.

v2 needs an inventory model that:

1. Records every movement as an append-only ledger entry; never patches denormalized counters.
2. Natively expresses the eleven edge cases catalogued in the category index [[../index]] (especially E1, E3, E5 — the empty-debt cycle).
3. Preserves traceability without a parallel audit table.
4. Has a clear, additive upgrade path from "single store, dozens of transactions/day" to "many stores, high volume" without app-code churn.

The frontend needs screens to open/close assignments, record sales/returns, and read per-assignment balances + discrepancies.

## Proposed Solution

- **Backend:** append-only **holder + ledger** model (ADR-013): `inventory_assignments` (driver-day lifecycle), `tank_holders`/`item_holders` (a holder is a **location's standing stock** *or* a **driver's day**, keyed by `kind`), `tank_transactions`/`item_transactions` (one clean `holderId` FK), and `customer_empty_debts`. **One** balance **view** per entity hides the SUM and serves every level (driver-day, location availability). A single transaction-`kind` enum dispatched by a **pure function** (no Strategy classes); a three-state assignment lifecycle (`open | closed | carried`); day-open and day-close are `transfer`s between the location and assignment holders. Routes at `/api/v1/inventory`, plus `GET /stores/:id/availability` for the operator.
- **Frontend:** assignment management + ledger entry UI (open day, record sale/purchase/return, close with discrepancy counts) and read views over the balance/discrepancy endpoints.

Detailed data model, views, transaction logic, lifecycle, and routes live in [[backend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `tank_transactions`, `item_transactions`, `customer_empty_debts` tables exist with append-only access (no UPDATE/DELETE in the repo for these).
- [ ] `tank_holders`/`item_holders` carry catalog data only — **no** `assigned*`/`current*` quantity columns; a holder is `location` *or* `assignment` (CHECK), and supplier `purchase` rows land on a `location` holder, not a driver (ADR-013).
- [ ] `inventory_assignments.state` is an enum with exactly `open | closed | carried`.
- [ ] One balance view per entity (`tank_balance`, `item_balance`, `customer_empty_debt_balance`) returns correct sums per holder, including zero-rows for holders with no transactions; the operator reads location availability (`kind='location'`, by store) from the same `tank_balance` view.
- [ ] Indexes: `tank_transactions(holder_id, occurred_at)`, `tank_transactions(ref_order_id)`, `tank_transactions(ref_transaction_id)`, `customer_empty_debts(customer_id, tank_type_id)`. Drizzle migrations in `src/db/migrations/`; no `db:push`.
- [ ] `src/modules/inventory/` follows [[../../../eng/patterns/module-template]]; only `repository.ts` imports `schema.ts`; no `I*` interface files, no DI container, no `dtos/` tree (types are Zod + `z.infer<>`).
- [ ] `recordSale(...)` writes one `tank_transactions` row and, when `emptyReceived < qty` and `customerId` is given, one balancing `customer_empty_debts` row in the same DB transaction; with no `customerId` it writes only the inventory row (no debt, no error).
- [ ] `recordReturn` for a customer with outstanding debt settles `min(qty, currentDebt)` against `customer_empty_debts`.
- [ ] Each of E1–E11 from [[../index]] has a passing test over the resulting ledger rows and view balances.
- [ ] No Strategy/StrategyFactory/StrategyProcessor classes (ADR-006); kind dispatch is a pure function.
- [ ] An `open` assignment accepts all kinds; a `closed` one rejects `sale`/`purchase`/`return`/`transfer` (4xx) but accepts `reconciliation`/`adjustment`; `closeAssignment` with mismatched `finalCounts` appends a `reconciliation` row; `carry` is idempotent.
- [ ] All routes return Zod-validated shapes (thin: validate → service → respond); errors flow through `src/lib/errors.ts`. `GET /assignments/:id` embeds balances from the views; `GET /assignments/:id/discrepancies` returns `kind='reconciliation'` rows.
- [ ] No `inventory_status_history` / `audit_logs` table (ADR-009); the ledgers are the audit trail (every row has `userId`, `occurredAt`, and applicable `ref*`).

- [ ] Opening a day writes a paired `opening` transfer (`−` location / `+` assignment) and `closeAssignment` writes a paired hand-back `transfer` (`+` location / `−` assignment), each pair in one DB transaction with matching `ref_transaction_id`.
- [ ] For any `(userId, date)` the API reconstructs that driver-day from the immutable ledger — opening level (`kind='opening'`), the day's deltas (`sale|return|adjustment`), expected end-of-day (excluding the hand-back `transfer`), and physical end-of-day + discrepancy (`reconciliation`); a past date reads back unchanged.
- [ ] Future holder kinds / extra attributes are side tables keyed by `holder_id` / `transaction_id`, never new columns on the ledger (ADR-013).

**Frontend (lpg-frontend-vue):**

- [ ] An `inventory` module lets an operator open an assignment, record sale/purchase/return entries, and close it with optional discrepancy counts.
- [ ] Per-assignment view shows current full/empty per tank type and quantity per item, sourced from the balance endpoints.
- [ ] Discrepancies (reconciliation rows) are listed for a closed assignment.
- [ ] Operator view shows location availability (shop stock, optionally + on-truck), sourced from the availability endpoint.
- [ ] Manual smoke test of the open → record → close → carry flow against the backend.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Inventory reservations** — the `orders → inventory` reservation step lives in `orders/orders-foundation`.
- **Daily reports / financial close-out** — belongs to a future `reports/` spec.
- **Multi-store transfer workflows** — `transfer` kind is in the schema but the cross-store UX is deferred.
- **Pagination on list endpoints** — added when a consumer needs it.
- **Push notifications** — dropped per [[../../../product/overview]] MVP scope.
- **Stale-recovery / business-day calendar** — not needed; the shop runs every day.

## Open Questions

None remaining for the backend design (validated against v1 and the edge-case catalogue). Frontend screens to be shaped when the frontend track starts.
