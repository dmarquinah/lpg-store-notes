---
project: lpg-store
domain: eng
last-updated: 2026-05-07
---

# Why v1 Grew to ~35k LOC for a Small Feature Set

A diagnosis of the architectural patterns that inflated the v1 backend, written so future specs can cite specific drivers when justifying v2 design choices. Inventory is the worked example because it is the largest area; the same drivers showed up everywhere.

## Headline numbers

| Area | LOC | Note |
|------|-----|------|
| Total v1 backend | **34,696** | non-test `*.ts` under `legacy/src/` |
| Inventory area | **7,517** | `services/inventory/`, `repositories/inventory/`, `routes/inventory*`, `db/schemas/inventory/` |
| Transaction strategies | **1,541** | `strategies/transactions/` (5-strategy pattern × tank+item) |
| DTOs (parallel tree) | **3,959** | `dtos/request/` + `dtos/response/`, types declared independently of the validators |
| Custom DI plumbing | **467** | `config/modules/*.ts` |
| `I*Repository` / `I*Service` files | **26** | one per concrete impl; no second implementation exists |

The Inventory PRD lists ~15 endpoints. ~9k LOC for 15 endpoints (~600 LOC per endpoint) is not a feature problem — it is an architecture problem.

## Ten drivers of bloat

Each driver lists the specific files in `legacy/` that exemplify it, so the link survives even if a path moves.

### 1. Interface-everywhere ceremony for one implementation

26 `I*Repository` / `I*Service` interface files, each with one concrete sibling. No alternative implementation exists. Every service file *also* declares its own `abstract class IXxxService` at the top. Three abstract surfaces per concrete type.

The custom DI container in `legacy/src/config/modules/*.ts` is pure plumbing: each module declares a `Dependencies` interface, a `createDependencies(coreDeps)` factory, a `getDependencies()` getter, and `null`-init guards. None of it is driving polymorphism.

**Cost**: roughly 3× LOC overhead for every concrete type, plus a learning tax for anyone tracing a call.

### 2. 8-permutation transaction-repo API

`legacy/src/repositories/inventory/IInventoryTransactionRepository.ts` declares the same operation across three orthogonal axes:

- by-`assignmentTankId` vs by-`inventoryId` (lookup style)
- `increment*` vs `decrement*` (sign)
- tank vs item (entity)

That gives 8 methods (plus 2 batch wrappers, plus 2 quantity readers). One signed-delta method per entity (`applyTankDelta(assignmentTankId, dFull, dEmpty, …)`) covers the same surface. The 569-LOC `PgInventoryTransactionRepository.ts` is mostly the cartesian product of these axes.

### 3. Strategy Pattern over five sign-formulas

`legacy/src/strategies/transactions/` is **1,541 LOC** for: 10 strategy classes (Sale, Purchase, Return, Transfer, Assignment) × (Tank, Item), 3 base classes (`TransactionStrategy`, `TransactionRequest`, `TransactionResult`), a `TransactionStrategyFactory` (135 LOC), a `TransactionProcessor` (93 LOC).

Inside `TankSaleStrategy.execute`, the actual logic is `{fullDelta: -qty, emptyDelta: +qty}`. The other four kinds are sibling formulas:

- Sale: `{full: -qty, empty: +qty}`
- Purchase: `{full: +qty, empty: -qty}`
- Return (full): `{full: +qty, empty: 0}`
- Return (empty): `{full: 0, empty: +qty}`
- Transfer: same delta on two assignments with opposite sign
- Assignment: set initial deltas (an opening event)

These are not five behaviors. They are five tiny sign tables. The polymorphism never amortizes.

### 4. Denormalized `assigned*` / `current*` columns that fight you

`legacy/src/db/schemas/inventory/inventory-assignments-tanks.ts` keeps four counters per row: `assignedFullTanks`, `currentFullTanks`, `assignedEmptyTanks`, `currentEmptyTanks`. Every transaction has to insert into `tank_transactions` *and* patch `current*` on the assignment row. Every consolidation has to copy `current*` into the next row's `assigned*`.

This dual-write is the reason `consolidationWorkflow.ts` is **466 LOC** and the bulk of `PgInventoryTransactionRepository.ts` exists. It also makes corrections painful: a "wait, that sale was wrong" cannot be a compensating transaction without also reaching back and patching the snapshot column.

### 5. `store_assignment_current_inventory` pointer + auto-routing

The pointer table and the auto-routing logic at `PgInventoryTransactionRepository.resolveTargetInventoryId` exist *only* because consolidated assignment rows are immutable but late transactions can arrive after consolidation. With ledger-only storage there is no immutable summary to bypass — append the transaction at the right date and read derives the truth.

### 6. Five-state workflow for a single-process, single-day shop

`CREATED → ASSIGNED → VALIDATED → CONSOLIDATED → OBSERVED`. The actual flow needs three: **open** (driver has it), **closed** (driver gave back counts), **carried** (system created next day's row). `VALIDATED` vs `CONSOLIDATED` separates "user said they're done" from "system processed it" — pretend-distributed-system thinking.

### 7. Smart business-day date service

`legacy/src/services/inventory/inventoryDateService.ts` handles weekend skipping, GMT-5 timezone math, stale-inventory detection. The shop runs every day. Stale recovery exists because the workflow has too many states (driver-1) and consolidation can fail silently (driver-4).

### 8. Generic audit on top of per-entity history

`audit/inventory_status_history` records status changes. `tank_transactions` and `item_transactions` already record every change. Two trails, one with business meaning. The audit-tables-for-compliance instinct landed without a consumer.

### 9. Parallel `dtos/` tree

**3,959 LOC** under `legacy/src/dtos/`, request and response trees mirroring entities. Types here are declared independently of the Zod schemas validating them at the route layer, so each shape has two definitions and they can drift. The v2 module template's `types.ts` (Zod schemas + `z.infer<>` co-located) replaces both with one source.

### 10. Premature `?include=` system

`InventoryAssignmentRelationOptions` and the `include` query param add a variance axis to every read path before any consumer asks for it. The four `InventoryAssignment*Relations` types and the conditional drizzle queries flow from it. Add relation loading per-feature when a real request comes in, not speculatively.

## Diagnosis in one sentence

The features were small. The bloat was **architectural patterns chosen for hypothetical futures**: DI for tests that never used it, Strategy Pattern for five sign-formulas, denormalized counters for read performance the volume did not need (paid for with consolidation, auto-routing, and stale recovery), and a five-state workflow for distributed coordination in a single-process app.

## What v2 takes from this

The decisions in [[decisions]] (ADR-003 through ADR-010) bake each driver's lesson into a rule:

- **D1, D2, D3** kill drivers 1–2 (no interfaces / no DI / Zod-first types).
- **ADR-005, ADR-006, ADR-007** kill drivers 3–5 and 7 (signed-delta repo API, function-first transaction kinds, ledger + view storage with documented upgrade path).
- **ADR-008** kills driver 6 (three-state workflow).
- **ADR-009** kills driver 8 (audit trail = transaction tables only).
- **ADR-010** addresses an inventory edge case v1 *named* but never modeled correctly (empty-tank debt as a typed customer-side obligation).

The first feature spec to apply these is [[../specs/inventory/inventory-foundation]].
