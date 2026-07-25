---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../quick-assignment/index]]"
  - "[[../inventory-foundation/index]]"
  - "[[../store-stock-first/index]]"
  - "[[../store-stock-adjustments/index]]"
last-updated: '"2026-07-18"'
---

# Spec: Per-Driver Inventory Pools (independent stock per driver within one store)

## Problem Statement

The daily one-tap quick-assign only works when a store has **exactly one**
delivery driver, because a store's inventory is a **single shared `location`
holder** — the "floor," keyed `(storeId, tankTypeId)` in `tank_holders` /
`item_holders`. To run **two drivers out of one physical shop**, the admin
**duplicates the store** ("Tienda Las Américas" + "Americas 2") so each fake
store gets its own floor and therefore its own one-tap `quick-open`.

That workaround corrupts the **store as the unit of everything else**: one
physical shop becomes two identities, splitting its orders, customers,
accounting registry, and history into two books for one place.

The root cause: a store's inventory is **pooled**, then must be **split** across
drivers at assignment time — an inherently ambiguous operation. "Sweep
everything to a driver" hands the first driver the whole floor and leaves the
second at **zero**. The duplicate store sidesteps the split by giving each
driver an **independent pool**. We want that independence **without** the fake
identity.

## Proposed Solution

Make inventory **attributed per driver within a store** instead of pooled per
store. Give the `location` holder a nullable **`store_assignment_id`** (the
delivery driver↔store link, `store_assignments`) alongside its existing
denormalized `store_id`, turning it into a **two-tier** concept:

- **Driver pool** — a `location` holder with `store_assignment_id` set: a
  driver's independent standing stock.
- **Store parking pool** — a `location` holder with `store_assignment_id`
  **NULL**: the store's unattributed floor, where stock lands when we don't yet
  know which driver takes it (a provider purchase into a multi-driver store, a
  future "parking spot" that transfers to other locations). A store's total
  stock = the **sum of all its location holders** (parking + every driver pool).

Then:

- A store's total stock = the **sum** of its parking pool + its drivers'
  independent pools. A **single-driver store is the degenerate case** (its floor
  backfills onto the one driver's pool = the store total) → behaves
  **identically to today**.
- Stock is attributed to a driver **when it arrives** (purchase / adjustment
  names the driver; single-driver auto-resolves) — the same choice the duplicate
  store forced via "Americas" vs "Americas 2," now without a second store.
- Quick-assign sweeps **that driver's own pool → their truck** (never the shared
  parking pool): one click per driver, and **never "driver B gets 0."** Parking
  stock is swept only after it's explicitly attributed/transferred to a driver.
- The **manual split** the admin sometimes needs becomes a natural op: attribute
  / adjust / transfer stock into a driver's pool.
- Carry-forward stays **within a driver's own pool** day to day.

Store **identity** (orders, customers, accounting, history) stays unified on the
single store record. Existing duplicate stores (e.g. "Americas 2") are **left
as-is**; merging them back is a deferred cleanup (owner: "decide later").

This reworks **ADR-013** (which modelled the `location` holder as store-level,
"not tied to any driver") — a new ADR records the per-driver keying and restates
edge cases E6 / E8 / E10 in per-driver terms.

Backend contract + migration in [[backend]]; card/dialog UX in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Data model & migration**

- [x] `location`-kind holders (`tank_holders`, `item_holders`) gain a nullable
      `store_assignment_id` (FK → `store_assignments`) beside the retained,
      **denormalized `store_id`**. Owner check: `location` → `store_id NOT NULL
      AND assignment_id NULL` (`store_assignment_id` free: NULL = **parking
      pool** / set = **driver pool**). **Two** partial unique indexes replace the
      old one: `(store_id, {type})` where `store_assignment_id IS NULL`, and
      `(store_assignment_id, {type})` where set. `assignment`-kind holders
      unchanged.
- [x] Migration (named, `drizzle-kit generate`; next number **0019**) is
      **non-destructive**: it backfills each `location` holder of a store with
      **exactly one** active delivery driver onto that driver's store-assignment;
      a store with **0** active delivery drivers keeps its floor as the **parking
      pool** (`store_assignment_id` left NULL — no block, no placeholder). A
      **pre-migration verification query** reports any store holding location
      stock with **>1** active delivery drivers and **aborts, fail loud** — the
      migration **does not silently guess**. Duplicate stores are **not** merged
      (deferred).
- [x] The single-driver path produces **identical** stock numbers before/after
      migration (regression check on the dev DB).

**Reads aggregate by store**

- [x] Store-total reads **sum across the store's location pools** (parking +
      every driver pool): Resumen
      availability (`GET /inventory/availability`), store detail tank + item
      availability, store history, and the store-stock-adjustments target. A
      single-driver store shows the same figures as today.
- [x] Store detail can break the store total down **per driver** (each driver's
      own pool).

**Assignment (the one-click goal)**

- [x] `POST …/quick-open` targets a **specific driver** (store-assignment) and
      sweeps **that driver's own location pool → their truck**. Single-driver
      store: **one tap, unchanged**. Multi-driver: **one button per
      not-yet-opened driver** (identity = which button), each a **single click**;
      a driver with an open day shows **Activo**. 0 drivers → 409. Never "driver B
      gets 0."
- [x] The general/manual open-day + explicit-quantity path still works per
      driver; "manual split" = attributing/adjusting stock into a driver's pool.
- [x] **Late-binding pool transfer.** An internal **pool→pool movement**
      (`transfer`-kind paired legs) re-attributes stock from the **parking pool**
      (default) or another driver **into a target driver's pool**, so stock parked
      at purchase time (driver unknown) can be assigned to a driver later without a
      re-purchase. Guarded (own-store operator / admin), never drives the source
      pool negative, and lands in the driver's **pool** (composes with quick-open /
      mid-day load; no surprise auto-load onto an already-open day). Realizes the
      "transfer stock into a driver's pool" natural op from the Proposed Solution.

**Purchases & adjustments attribute to a driver**

- [x] `recordTankPurchase` / `recordItemPurchase` and `POST …/adjustments` target
      a **driver's pool** (store-assignment). Single-driver **auto-resolves** (no
      UI change); multi-driver: the purchase/adjust dialog **picks the driver**.      Provider debt / cost / accounting-egress semantics are **unchanged**.
      When **no** driver is named on a multi-driver store (driver unknown), the
      purchase/adjust lands on the store's **parking pool**.
- [x] Purchase **auto-load** becomes **unambiguous for multi-driver too**: a
      purchase attributed to a driver whose day is **open** auto-loads onto **that
      driver's** truck (same tx); no open day → stays on that driver's pool for the      next sweep. A purchase attributed to the **parking pool** stays there (no
      auto-load). Supersedes the old "exactly one active delivery" gate.

**Edge cases / ADRs**

- [x] A new ADR supersedes the store-level assumption in **ADR-013**; edge cases
      **E6** (opening transfer), **E8** (supplier purchase → a driver's pool), and
      **E10** (carry returns leftovers to the **driver's own** pool) are restated
      in per-driver terms. **E1–E5, E7, E9, E11** are unaffected.

**Gates (both repos)**

- [x] `npm run typecheck` + `npm run check` + `npm test` + `npm run build` green
      in lpg-backend; `typecheck` + `build` green in lpg-frontend-vue. Existing
      inventory tests updated to the per-driver keying; new tests for multi-driver
      sweep, purchase attribution, and migration backfill resolution (0 / 1 / >1
      driver).
- [x] **Frontend:** Resumen cards show the store total (Σ pools) + per-driver
      assign buttons on multi-driver stores (single-driver card unchanged);
      purchase/adjust dialogs gain a driver selector **only** when the store has
      >1 driver; store detail shows a per-driver breakdown. Design-system
      compliant (tokens, ≥44px targets).

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Merging existing duplicate stores** (Americas / Americas 2) — deferred
  cleanup, its own future spec.
- **Standing per-driver loadouts + store-level one-click "Abrir día" + scheduled
  auto-open** — the automation phase; captured as its own backlog draft once this
  model ships. (This spec is the foundation that makes it unambiguous.)
- **Money / debt / accounting semantics** beyond re-pointing the holder key —
  provider debt, egress valuation, and customer empty-debt are all unchanged.
- **Non-delivery store-assignments as pools** — only **delivery** assignments own
  inventory pools (operators do not).

## Resolved Decisions

_(Resolved with the owner at `/focus`, 2026-07-17.)_

- **Denormalize `storeId` on the location holder — YES.** A store-assignment's
  store never changes, so `store_id` stays on the location holder. This keeps
  the hot store-scoped reads (Resumen aggregate, accounting egress, store
  history) filtering by `store_id` **join-free and unchanged**; only the balance
  aggregation sums across a store's holders.
- **Two-tier location model (parking pool + driver pools).** Rather than a full
  re-key, `store_assignment_id` is **nullable** on the location holder: NULL =
  the store's **parking pool** (unattributed floor), set = a **driver pool**.
  Store total = sum of both tiers. This makes the migration non-destructive and
  gives unattributed stock (driver-unknown purchases, future transfer spot) an
  honest home instead of a placeholder hack.
- **Migration, store with 0 active delivery drivers — keep as parking pool.** No
  block, no placeholder: the floor simply stays `store_assignment_id` NULL. (In
  theory such stores shouldn't exist unattended; the parking pool is the
  constrained, honest representation when they do.)
- **Migration, store already with >1 active delivery driver — abort, fail loud.**
  Prod has **no** such store (confirmed by the owner). If the assumption is ever
  wrong the pre-check reports it and the migration refuses, rather than
  mis-attributing. Duplicate stores stay separate (deferred merge).
- **Quick-open sweeps only the driver's own pool**, never the shared parking
  pool — keeps the one-click deterministic and preserves "never driver B gets
  0." Parking stock is swept only after explicit attribution/transfer to a
  driver.
- **Carry-forward returns leftovers to the driver's own pool** (E10 per-driver)
  — forced by the model (no store-level shared floor remains) and matches the
  physical reality (driver drops unsold tanks back at the shop, still "theirs"
  for tomorrow).
