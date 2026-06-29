---
project: lpg-store
domain: specs
type: spec-track
spec: multi-type-fulfillment
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-15
---

# Multi-type fulfillment — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend track: [[backend]]

## Technical Notes

Two independent surfaces in the `inventory` + `orders` modules. Build against the design system (`formatMoney`, badges, table density per `eng/design-system`).

### 1. Smart "Comprar y cargar" in the assign dialog

`AssignDialog.vue` (`src/modules/orders/components/AssignDialog.vue`) currently resolves oversell warnings with `loadMissingTanks()` → `inventory.recordLoad(...)` per warning, which **fails when the order's store lacks the stock** (the 45kg case). Replace with a store-aware resolution:

- The dialog already has the order's `storeId` and `orders.lastWarnings` (`{ tankTypeId, required, available, shortfall }`).
- Fetch the **order store's** availability (`GET /inventory/stores/:storeId/availability` → `inventory.fetchStoreAvailability` / existing store method) once.
- For each shortfall: if the store has `≥ shortfall` full of that type → `recordLoad(assignmentId, { tankTypeId, full: shortfall, empty: 0 })`; else → `recordTankPurchase(storeId, { items: [{ tankTypeId, qty: shortfall }] })` **then** `recordLoad(...)`. (Purchase only the part the store is missing if you want to be precise; simplest correct rule: purchase the shortfall the store can't cover, then load the full shortfall.)
- Relabel the button **"Comprar y cargar"** (or keep "Cargar faltantes" when the store already has everything). Surface the backend's clearer 409 if anything still fails.

Mirrors the backend 409 message (now naming the tank type) for any residual error.

### 2. Per-type movements legibility

`AssignmentDetailView.vue` (`src/modules/inventory/views/AssignmentDetailView.vue`) + `src/modules/inventory/types.ts`:

- Add `tankTypeId: number` to the `TankTransaction` type (the backend already returns it on `tankTransactionsByAssignment`).
- Add a **Tanque** column to the Movimientos table, resolving `tankTypeId` → tank-type name via the catalog store (reused elsewhere in the module).
- Add `load` to `TX_KINDS` and `KIND_LABELS` (`load: "Carga"`) so all 8 backend kinds are covered and the refill row is labelled (no blank "Tipo").

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify:

- `src/modules/orders/components/AssignDialog.vue` — store-aware "Comprar y cargar" resolution (load-or-purchase-then-load per shortfall).
- `src/modules/inventory/views/AssignmentDetailView.vue` — Tanque column in Movimientos; `KIND_LABELS.load = "Carga"`.
- `src/modules/inventory/types.ts` — add `tankTypeId` to `TankTransaction`; add `"load"` to `TX_KINDS`.
- `src/modules/inventory/{store,service}.ts` — reuse/confirm store-availability fetch + `recordTankPurchase`/`recordLoad` (all exist).

Reference (read for patterns): `src/modules/catalog/` (tank-type name resolution), the existing `PurchaseDialog.vue` / `LoadDialog.vue`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-frontend-vue] Frontend track **done**. Three surfaces shipped, display/flow only — no new endpoints (all composed from existing `getShopBalances`/`recordTankPurchase`/`recordLoad`).

- **Smart "Comprar y cargar"** (`AssignDialog.vue`): on entering the oversell-warnings phase, snapshot the order store's full-tank stock once via `inventory.getShopBalances(storeId)` into a `Map<tankTypeId, fullTanks>`. `resolveShortfalls()` replaces the old `loadMissingTanks()`: per warning, if the store has `≥ shortfall` → `recordLoad` straight; else `recordTankPurchase(storeId, { items:[{ tankTypeId, qty: shortfall − have }] }, false)` **then** `recordLoad(..., full: shortfall, empty: 0)`. Purchases only the part the store lacks (`shortfall − have`, no double-buy); loads the full shortfall. Bails on first failure (store surfaces the backend's now tank-type-named 409). A `needsPurchase` computed flips the button label "Cargar balones faltantes" → **"Comprar y cargar"** and shows a note that a provider Compra will be registered. Warnings are one-row-per-tankTypeId, so the single snapshot is safe.
- **Per-type Movimientos legibility** (`AssignmentDetailView.vue`): added a **Tanque** column resolving `tankTypeId` → name via the existing `tankName()`/catalog resolver (colspan 5 → 6). Added `KIND_LABELS.load = "Carga"` so the refill row is no longer blank.
- **Types** (`types.ts`): added `"load"` to `TX_KINDS` (now all 8 backend kinds; `KIND_LABELS: Record<TxKind,string>` is exhaustive so a gap would fail typecheck) and `tankTypeId: number` to `TankTransaction` (already on the wire — service passes transactions through untouched).

No cross-module type import: the orders module caches store stock as a plain `Map`, avoiding pulling `TankAvailabilityRow` across the boundary. Validation agent: all 3 frontend criteria met, no correctness bugs. Gates green: typecheck + build. Manual multi-type smoke (driver+store both lack 45kg → Comprar y cargar → deliver) left to the operator per the criteria.