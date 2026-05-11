# Architecture Decisions — lpg-store

---
project: lpg-store
domain: eng
last-updated: 2026-05-07
---

Append-only log. Newest entries at the top.

---

## ADR-010 — Empty-tank debt is a customer-side ledger, not an inventory column (2026-05-07)

**Decision**: Customer obligations to return empty tanks live in a separate `customer_empty_debts` ledger keyed by `(customerId, tankTypeId)` with signed-delta rows. Each row links back to the inventory transaction that produced it (`refTankTransactionId`). The inventory ledger never represents fictitious empties — a sale where the customer did not return an empty is `{fullDelta: -1, emptyDelta: 0}`, full stop. Per-customer balances are derived via a SQL view in the same shape as `assignment_balance` (see ADR-007).

**Why**: v1 named this case (`order_items.tank_returned: boolean default true`) but never modeled it correctly. `TankSaleStrategy` always emitted `+1 empty` regardless of the flag, and `customer_debts` was monetary-only (`decimal amount`) and could not represent "this customer owes us 1 empty 10kg cylinder of type X". Without a typed ledger, partial returns ("they brought back 2 empties for 1 full to settle a prior debt") cannot be reconciled.

**Consequence**: Sale transactions write at most two ledger rows: one to `tank_transactions` (inventory side) and one to `customer_empty_debts` (customer side, only if the empty count returned is less than the fulls taken). The two ledgers are joined via `refTankTransactionId` for traceability. Walk-in (no customer) sales bypass the customer ledger entirely; the inventory side still records honest deltas. Empty-debt does **not** carry between daily assignments — it lives on customers, not on inventory.

---

## ADR-009 — Audit trail = transaction tables only (2026-05-07)

**Decision**: v2 has no `inventory_status_history`, no `audit_logs`. The `tank_transactions` and `item_transactions` ledgers, plus `order_status_history` (when the orders module lands), are the audit trail. A small `assignment_status_changes` log is added only if a UI surface actually needs it; default is no.

**Why**: v1's per-entity history table duplicated information already present in the transaction ledger and answered no business question that the ledger could not. Generic `audit_logs` had no consumer. Compliance-instinct schemas without consumers rot.

**Consequence**: Reports that previously read `inventory_status_history` will be derived from the transaction ledger. If an audit query needs status changes that are not implied by transactions (e.g., a manual override), it gets its own typed log at the time the override is added.

---

## ADR-008 — Three-state inventory workflow (open / closed / carried) (2026-05-07)

**Decision**: Inventory assignments have three states: `open` (driver has it, transactions accepted), `closed` (driver returned, day-end counts recorded, no further transactions), `carried` (system has created the next day's `open` assignment from this one's closing balances). Transitions: `open → closed → carried`. No `VALIDATED`, no `CONSOLIDATED`, no `OBSERVED`.

**Why**: v1's five states with separate `VALIDATED` (user said done) and `CONSOLIDATED` (system processed) modeled distributed coordination on a single-process, single-store, same-day shop. `OBSERVED` was a catch-all for things gone wrong that no UI surfaced. Stale recovery existed because the workflow had too many states and consolidation could fail silently.

**Consequence**: A late transaction arriving after `closed` is recorded as a reconciliation transaction on the (still-existing) day's assignment, with `kind = "reconciliation"`. The next-day's `open` is created at `carried`-time from a derived view over the previous day's ledger; if it has already been created, we no-op. The assignment row never becomes immutable.

---

## ADR-007 — Ledger + Postgres view; documented upgrade path (2026-05-07)

**Decision**: Inventory current quantities are derived, not stored. The authoritative tables are `tank_transactions` and `item_transactions` (signed-delta rows). A SQL `VIEW assignment_balance` exposes `(inventoryId, tankTypeId, currentFullTanks, currentEmptyTanks)` to callers; equivalent view for items. App code reads from the view as if it were a table. No `current*` / `assigned*` denormalized columns on the assignment-tanks/items tables.

**Why**: v1's denormalization (`currentFullTanks` etc. on `assignment_tanks`) was the root cause of consolidation, auto-routing, and stale-recovery complexity. Every transaction had to dual-write, and corrections required reaching back to patch a column. Ledger-first reverses the polarity: append-only writes, derived reads.

**Why a view**: With the SUM hidden in a view, callers do not know whether the balance is computed live, materialized, or snapshotted — so the storage strategy can be upgraded without touching application code.

**Upgrade path** (additive, non-disruptive):
1. **Today (1 store, dozens of tx/day per assignment)**: plain `VIEW` over the ledger. Index `(inventoryId, tankTypeId)` makes it sub-millisecond.
2. **If reads pressure shows up (~50+ stores, hot cross-store dashboards)**: convert to `MATERIALIZED VIEW` with `REFRESH MATERIALIZED VIEW CONCURRENTLY` on a cron tied to transaction-write rate.
3. **If matview refresh becomes a bottleneck**: replace the matview with a snapshot table maintained by the repository in the same DB transaction as the ledger write. Caller code is unchanged. The ledger remains the source of truth — the snapshot is a cache.

**Consequence**: No assignment row contains a balance. Reads go through `assignment_balance` (or the items equivalent). Writes go to the ledger; the repo never touches a balance column because there is none. Multi-store scaling is structural in the existing keying (`store_assignments → inventory_assignments → tank_transactions`); per-store totals are a `GROUP BY` over the same ledger.

**Plain dates** (folded into this ADR): assignments key on `date` (not timestamp). "Next day" is `+1 day`. No business-day service, no weekend skipping. The shop runs every day; if that ever changes, a holidays table is the right answer, not a date service.

---

## ADR-006 — Function-first transaction kinds; no Strategy Pattern (2026-05-07)

**Decision**: Inventory transaction kinds (`sale`, `purchase`, `return`, `transfer`, `opening`, `adjustment`) are expressed as a small enum and a `kindToDelta(kind, qty, opts)` function. There is no `TransactionStrategy` base class, no `*Strategy` per kind, no `TransactionStrategyFactory`, no `TransactionProcessor`. The service has one method per business operation (`recordSale`, `recordPurchase`, …) that calls `kindToDelta` and writes via the repo.

**Why**: v1's 1,541 LOC of strategies dispatched on five sign-formulas. The polymorphism never paid off — these are not five behaviors, they are five tiny sign tables. Switch-case (or a function table) is the right tool when the variation is data, not behavior. See `legacy-bloat-analysis` driver 3.

**Consequence**: Adding a new kind = a new enum value + a new arm in `kindToDelta` + a new service method. Routes and types follow. Total LOC for a new kind: ~30 vs ~300.

---

## ADR-005 — Signed-delta repo API for inventory transactions (2026-05-07)

**Decision**: The transaction repository exposes one signed-delta write per entity:

```
applyTankDelta(assignmentTankId, fullDelta, emptyDelta, kind, userId, refs?, notes?)
applyItemDelta(assignmentItemId,  delta,                   kind, userId, refs?, notes?)
```

Plus `findAssignmentTank(inventoryId, tankTypeId)` / `findAssignmentItem(...)` for callers that have an `inventoryId` not an `assignmentTankId`. No `increment*` / `decrement*` × `byAssignment` / `byInventory` matrix.

**Why**: v1 declared the same operation across three orthogonal axes (driver 2). Sign is data; lookup style is one helper call away.

**Consequence**: One repo method per write path. Batch operations live in the service, not the repo (the repo's job is the single atomic write).

---

## ADR-004 — Zod-first types co-located in module/types.ts (2026-05-07)

**Decision**: Each module's `types.ts` is the single source of types: Zod schemas (`requestBody`, `responseBody`) plus `z.infer<>` aliases. No parallel `dtos/` tree. No TS `interface` declarations duplicating what a Zod schema describes. Routes call `schema.parse(req.body)` and pass the inferred type to the service.

**Why**: v1's 3,959-LOC `dtos/` tree declared types independently of the Zod schemas validating them at the route layer, so each shape had two definitions that drifted (driver 9). The Zod schema is authoritative; the inferred TS type is a free byproduct.

**Consequence**: Changing a request shape is one file. Validators and types cannot drift because there is only one declaration.

---

## ADR-003 — No `I*` interfaces unless polymorphism is real; no DI container (2026-05-07)

**Decision**: Services accept concrete repository instances. Routes accept concrete service instances. We do not declare `IFooService` / `IFooRepository` interfaces unless there is more than one implementation in the codebase. Tests substitute by passing a small in-memory class with the same method shape (TypeScript structural typing). Composition happens in `src/app.ts`; there is no DI container.

**Why**: v1 had 26 `I*` files for 26 single-implementation classes (driver 1). The custom DI container in `legacy/src/config/modules/*.ts` (467 LOC) was pure plumbing — `null`-init guards, `getDependencies()` accessors, factory pairs — none of which drove polymorphism.

**Consequence**: When a real second implementation appears (e.g., an in-memory cache repo for tests, a different transport), introduce the interface at that moment. The cost of adding it later is local; the cost of carrying it now is global.

---

## ADR-002 — Single vault project covers all repos (2026-05-07)

**Decision**: One Obsidian vault project (`lpg-store`) covers `lpg-backend`, `lpg-frontend-vue`, and `lpg-bot`. Each repo's `CLAUDE.md` declares `vault-project: lpg-store`.

**Why**: Most product features (orders, customers, inventory) span backend + frontend + (potentially) bot. A single project with cross-repo `Related Files` lists avoids duplicating spec content per repo. The global vault config explicitly supports this pattern.

**Consequence**: Specs that touch multiple repos list every relevant path. Backend-only or frontend-only specs still live in the same `specs/` board, just with narrower Related Files. Implementation notes get tagged `[YYYY-MM-DD] [repo-name] ...`.

---

## ADR-001 — Reset backend as v2, archive v1 (2026-05-07)

**Decision**: Build a clean v2 of the backend at the repo root with vertical modules; archive v1 under `legacy/` as read-only reference; delete `legacy/` once v2 reaches functional parity.

**Why**: v1 (~22.5k LOC, 7 modules, custom DI, 5-strategy transaction pattern, 998-LOC workflow repo) became hard to maintain — the user reported being unable to track impact of changes across modules. Pre-production state means no users to migrate; lessons live in PRDs + code, not data.

**Consequence**: Features are ported one at a time, each with a spec that points at v1 in `legacy/` for requirements and prior art. Repo grows incrementally; `legacy/` shrinks toward zero.

---
