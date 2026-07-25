---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../day-handoff/index]]"
  - "[[../store-stock-first/index]]"
  - "[[../../stores/store-management/index]]"
last-updated: 2026-07-04
---

# Spec: Quick Assignment (one-tap day-open + purchase auto-load for single-delivery stores)

## Problem Statement

The business is new and, for now, **each store runs with exactly one delivery
driver**. Yet the daily start-of-day flow still forces the operator through the
**general** assignment path built for a scaled-up business: `openDay` requires a
`storeAssignmentId` **and an explicit per-tank-type full/empty quantity for
every type** the store carries (`service.ts` → `openDay`, `loadFromLocation`).
For a single-driver store that just wants *"give this driver everything we have
on the floor today"*, that is a lot of typing and a lot of chances to fat-finger
a count — the exact daily friction the owner wants gone.

Two more rough edges compound it:

1. **Purchases don't reach the driver.** A provider purchase
   (`POST /inventory/stores/:storeId/{tank,item}-purchases` →
   `recordTankPurchase` / `recordItemPurchase`) lands **only on the store
   `location` holder**. When a store has a single driver already out for the day,
   the operator must then *manually* load the just-arrived stock onto the truck
   (a second step, `POST /assignments/:id/loads`). It should just flow to the one
   driver automatically.
2. **The general open-day dialog shows every driver.** `OpenDayDialog` doesn't
   scope its driver picker to the store being opened, so the operator can pick a
   driver who isn't even assigned to that store — noise that gets worse, not
   better, as stores are added.

We explicitly **keep the general workflow** (pick driver + type each quantity)
for the day the business scales past one-driver-per-store. This spec adds a
**"quick" path layered on top of it**, gated on the store having exactly one
active delivery assignment.

## Proposed Solution

A **single-delivery fast path** for daily assignment, plus a driver-list scope
fix on the general dialog.

- **Backend — quick-open endpoint.** New `POST
  /inventory/stores/:storeId/quick-open` that: resolves the store's **sole active
  delivery** `store_assignment` (via `catalog.listStoreAssignmentDetails({
  storeId, role: 'delivery', active: true })`); **409** if there is 0 or >1 (the
  quick path only exists when the assignment is unambiguous — >1 falls back to the
  general dialog); then opens today's day and **auto-loads the store's entire
  current `location` stock** (`tankBalancesByLocation(storeId)`, every type with
  full>0 or empty>0) onto that driver, reusing the existing `openDay` /
  `loadFromLocation` transfer so the ledger, guards, and the driver push
  notification are all unchanged. Same start-of-day rules apply (today's Lima date
  only; unconsolidated prior day → 409).
- **Backend — purchase auto-load.** When a purchase is recorded into a store that
  has **exactly one active delivery with an `open` day today**, the purchased
  full/empty (tanks) — and items — are **also loaded onto that driver's open day**
  in the *same* transaction (tank: reuse `loadFromLocation('load')`; item: the
  item-load equivalent). **When no day is open, nothing auto-loads** — the
  purchase stays on the `location` holder and the next quick-open sweeps it (the
  decided rule; least surprising). >1 delivery, or an ambiguous/closed day → no
  auto-load (unchanged behavior).
- **Frontend — quick buttons on the Resumen cards.** On `/inventario`'s Resumen
  store cards (`InventoryView`), a store with a single delivery and **no open day
  yet** shows a prominent **"Asignar todo el stock a {driver}"** button → one tap
  calls quick-open → toast + refetch. A store that already has an open day shows
  its state instead (no button). Stores with 0 or >1 delivery show a link into the
  general `OpenDayDialog` (unchanged path).
- **Frontend — scope the general dialog's driver list.** `OpenDayDialog` filters
  its driver options to **only delivery users assigned to the selected store**
  (`listStoreAssignments({ storeId, role: 'delivery' })`), so even the manual path
  can't pick an off-store driver.

No schema change — quick-open and purchase auto-load are **compositions of
existing ledger primitives** (`openDay`, `loadFromLocation`, the purchase
services). The order auto-assign expansion the owner floated is **out of scope**
here and captured as its own backlog draft
([[../../orders/single-delivery-auto-assign/index]]).

Backend contract in [[backend]]; buttons + dialog scoping in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Backend (lpg-backend):** ✅ done 2026-07-04

- [x] New endpoint `POST /inventory/stores/:storeId/quick-open` resolves the
      store's **sole active delivery** `store_assignment`; **409** (clear message)
      when the store has **0** or **>1** active delivery assignments. Guarded like
      the other write paths (operator own-store + admin/dev; cross-store → 403).
- [x] Quick-open opens **today's** day for that driver and auto-loads **all**
      current `location` stock onto the truck: every **tank** type with full>0 or
      empty>0 (`tankBalancesByLocation`, **full + empty** — decided) via the
      existing `openDay`/`loadFromLocation` transfer, **and** every **item** with
      qty>0 (`itemBalancesByLocation`) via a **new item-load transfer primitive**
      (`loadItemFromLocation`, mirroring the tank one — paired legs, negative-stock
      guard). Paired ledger legs + driver push notification reused, not
      re-implemented.
- [x] Start-of-day rules are **unchanged**: today's Lima date only; an existing
      day for today → 409 ("ya existe un inventario"); an unconsolidated prior day
      → 409 (the existing pending-day guard). A store with **empty** location stock
      quick-opens an **empty** day (0 loads), it does not error.
- [x] **Purchase auto-load:** when `recordTankPurchase` **or**
      `recordItemPurchase` targets a store that has **exactly one active delivery
      with an `open` day today**, the purchased quantities (tanks **and** items)
      are **also loaded onto that day** in the **same transaction** (`load`-kind
      legs, via `loadFromLocation` / `loadItemFromLocation`). If the transfer
      fails, the whole purchase rolls back.
- [x] Purchase auto-load is **skipped** (purchase lands only on `location`, as
      today) when: no day is open, the store has 0 or >1 active delivery, or the
      day isn't `open`. Decided rule: **no auto-open on purchase.**
- [x] The **general** `openDay` path and its explicit-quantity contract are
      **untouched** (the quick path is additive). New tests: quick-open all-stock,
      quick-open empty store, 0-delivery 409, >1-delivery 409, cross-store 403,
      purchase-auto-load-when-open, purchase-no-autoload-when-no-open-day,
      auto-load-skipped->1-delivery. Existing tests green; typecheck / lint / build
      green.

**Frontend (lpg-frontend-vue):** ✅ done 2026-07-04

- [x] On `/inventario` Resumen cards (`InventoryView`), a **single-delivery store
      with no open day today** shows a primary **"Asignar todo el stock a
      {driver}"** button → calls `POST …/quick-open` → success toast naming the
      driver + refetch (the card flips to its open-day state). Backend 409s
      (already-open / pending prior day / ambiguous delivery) surface as clear
      messages.
- [x] A store that already has an open day, or has **0/>1** deliveries, shows
      **no** quick button — the latter links into the general `OpenDayDialog`
      instead (the manual path stays reachable).
- [x] `OpenDayDialog`'s driver picker lists **only delivery users assigned to the
      selected store** (`listStoreAssignments({ storeId, role: 'delivery' })`), not
      all drivers.
- [x] The plain **Compra** dialog needs **no** new control — auto-load is a
      backend side effect; after a purchase into a single-delivery open-day store, a
      refetch shows the stock on the driver's day (verify in smoke).
- [x] **"Comprar y cargar" is single-delivery aware** (backend now auto-loads on
      purchase — [[../multi-type-fulfillment/index|multi-type-fulfillment]]): for a
      store with **one** delivery driver and an open day, the flow **skips the
      explicit `POST /assignments/:id/loads`** (the purchase already loaded it), or
      treats its stock-out **409 as a benign no-op**; the two-step flow is kept for
      **multi-delivery** stores (backend does not auto-load those). No double-load,
      no spurious error toast.
- [x] Design-system compliant (tokens, ≥44px targets); `npm run typecheck` +
      `npm run build` green.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Order auto-assign** (create an order for a single-delivery store → auto-assign
  the driver). The owner floated it as a "maybe expand"; deferred to its own
  backlog draft [[../../orders/single-delivery-auto-assign/index]].
- **Auto-opening a day on purchase.** Decided against — a purchase must not open a
  driver-day as a side effect; the quick-open button is the explicit act. Auto-load
  only targets an **already-open** day.
- **Multi-driver quick path.** By definition the quick path only exists when the
  store has exactly one delivery. >1 driver stays on the general dialog.
- **End-of-day wording + inventory-page UX sweep** — sibling spec
  [[../inventory-ux-pass/index]].
- **Empty-tank customer debt / accounting egress** — unchanged; quick-open and
  auto-load are ordinary loads/purchases, no new money or debt semantics.

## Open Questions (resolved at `/focus` 2026-07-04)

- **Load empties too, or fulls only, on quick-open?** → **Both** full and empty
  (owner: "all the available current inventory"), reusing the general transfer.
- **Purchase auto-load when the day is `closed`/`carried` (already counted)?** →
  **Skip** (treat like no open day; can't load onto a counted day). No auto-open.
- **Scope — operators own-store only?** → **Yes**, mirror purchase scoping
  (`resolvePurchaseActor`: operator own-store + admin/dev; cross-store → 403).
- **Item auto-load parity.** → **In scope.** The model had no item→driver-day
  transfer (`openDay` was tanks-only; items only got an assignment holder at sale
  time). Owner wants items to work from day 1, and the seeded catalog already has
  items. Resolution: build a **`loadItemFromLocation`** primitive mirroring the
  tank one and thread items through both quick-open and purchase auto-load. The
  shared `inventory_tx_kind` enum **already includes `load`** (tank+item share it)
  → **no migration**; only `kindToItemDelta` gains a `load` case.
