---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../../orders/orders-foundation/index]]"
  - "[[../../orders/orders-multi-location/index]]"
last-updated: 2026-06-15
---

# Spec: Multi-type fulfillment — restock-on-assign + per-type ledger legibility

## Problem Statement

Surfaced while flow-testing an order that spans two tank types (1× 10kg + 1× 45kg):

1. **You can't fulfil a tank type the store doesn't stock.** The driver had 10kg but no 45kg, so assign produced the correct oversell warning. But the one-click **"Cargar balones faltantes"** calls `recordLoad` (store → truck), and the *store itself* had no 45kg stock, so it failed with a 409: `Stock insuficiente en la tienda para el tipo 2 (...)`. Two problems: (a) a tank type the store has never carried can only enter inventory via a **provider purchase (Compra)**, not a load — the UI only offered the load, so the operator hits a wall; (b) the error prints the **numeric `tankTypeId`** ("tipo 2"), not "Balón 45kg", and gives no hint that a purchase is the way out. *(It is a clean 409, not a crash — but it reads like a dead end.)*

2. **A multi-type day is unreadable in the assignment ledger.** The **Movimientos** table shows `Fecha · Tipo · Δ Llenos · Δ Vacíos · Notas` but **no tank type** — so `Venta −2` is ambiguous across products (10kg? 45kg? a mix?). The backend already sends `tankTypeId` on every row; the frontend just drops it and renders no column.

3. **The refill row is blank.** The mid-day load renders with an **empty "Tipo"** because the frontend's kind list + label map omit the backend's `load` kind (covers 7 of 8). So a `Carga` looks like an unlabelled mystery row.

Net effect: the operator can't complete a common multi-type order without guessing, and can't audit the resulting day.

## Proposed Solution

- **Backend — clearer stock-out signal.** When a load is rejected for insufficient store stock, the 409 **names the tank type** ("Balón 45kg"), shows available-vs-requested, and tells the operator to register a **Compra** first. (No new endpoint — a provider purchase already creates the store holder for a never-stocked type.)
- **Frontend (assign) — smart "Comprar y cargar".** The oversell-resolution becomes one action: for each shortfall, if the **order's store** has the stock → load it onto the truck; if not → register a provider **purchase** into the store for the missing qty, **then** load. The order becomes fulfillable end-to-end without the operator manually chaining Compra → Cargar. (Branches on store availability so it never double-counts stock the store already holds.)
- **Frontend (movements) — per-type legibility.** Add a **Tanque** column to the Movimientos table (resolve `tankTypeId` → name via catalog), and add `load → "Carga"` so every kind is labelled and all 8 backend kinds are covered.

Detailed backend design in [[backend]]; the assign + movements screens in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] A load rejected for insufficient store stock returns a **409 whose message names the tank type** (e.g. "Balón 45kg"), not the numeric id, includes available-vs-requested, and **guides the operator to register a Compra** before loading.
- [ ] A provider tank purchase for a tank type the store has **never stocked** still succeeds (creates the location holder) — covered by a test.
- [ ] Existing inventory tests stay green; typecheck / lint / build green. No migration (no schema change).

**Frontend (lpg-frontend-vue):**

- [ ] The order-assign oversell-resolution offers a single **"Comprar y cargar"** action: per shortfall, load when the order's store has the stock, else register a provider purchase into the store for the missing qty then load — so a driver-and-store stockout (e.g. 45kg) is fully resolved in one flow.
- [ ] The assignment **Movimientos** table shows a **Tanque** column resolving `tankTypeId` → tank-type name on every row.
- [ ] The **`load`** kind renders as **"Carga"** (no blank "Tipo"); the frontend kind list covers all 8 backend kinds.
- [ ] typecheck + build green; manual smoke (multi-type order: driver and store both lack 45kg → Comprar y cargar → deliver) left to the operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Atomic purchase+load.** The two steps stay separate calls; if the load fails after a purchase, the purchased stock is genuinely in the store (correct) — the operator retries the load. No new combined transactional endpoint.
- **Auto-purchasing without operator action.** "Comprar y cargar" is an explicit operator action (it records money owed to the provider); nothing buys stock silently.
- **Item (accessory) fulfillment onto trucks.** Items live at the store only (no `recordItemLoad`, per the inventory model); this spec is tank-types only.
- **Changing the ledger shape.** No new kind/column/migration — the movements fixes are display-only (the data is already sent).

## Open Questions

- _None._ Approach approved 2026-06-15 (smart load-or-purchase resolution; per-type movements column).
