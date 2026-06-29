---
project: lpg-store
domain: specs
type: spec-track
spec: multi-type-fulfillment
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-15
---

# Multi-type fulfillment — lpg-backend track

Shared spec: [[index]] · Foundation: [[../inventory-foundation/index]]

## Technical Notes

Tiny, display/UX-only backend slice — **no schema, ledger, view, enum, or migration change**. The whole "Comprar y cargar" resolution is composed on the frontend from endpoints that already exist (`POST /stores/:storeId/tank-purchases` then `POST /assignments/:id/loads`); a provider purchase for a never-stocked type already creates the location holder (`findOrCreateTankHolder`), so no backend gap there.

### The one change: name the tank type in the stock-out error

`InventoryService.loadFromLocation` ([service.ts](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/inventory/service.ts)) rejects an over-load with:

```
throw new ConflictError(`Stock insuficiente en la tienda para el tipo ${tankTypeId} (...)`);
```

Replace the numeric `${tankTypeId}` with the tank-type **name**, and append guidance to register a Compra first. The name is only needed on the (rare) error branch, so resolve it lazily there — add a lightweight `tankTypeLabel(id)` read to the repo (select `name` [+ `sizeLabel`] from `tank_types`) and call it when building the error message. No cost on the success path.

Resulting message shape (example):
`La tienda no tiene stock suficiente de "Balón 45kg" (disponible: 0 llenos / 0 vacíos; solicitado: 1 / 0). Regístralo con una compra antes de cargarlo al repartidor.`

This path is shared by `openDay` and `recordLoad` (both call `loadFromLocation`), so both get the clearer message for free.

### Notes

- `recordTankPurchase` already handles a never-stocked type (creates the holder, `emptyReturned` defaults to `min(qty, 0)=0`). Add a test asserting a purchase for a brand-new tank type succeeds end-to-end (the foundation likely covers it; make it explicit for this scenario).
- Keep the message in Spanish (matches `zod-locale` + the named-error convention).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/service.ts` — `loadFromLocation`: name the tank type + add Compra guidance in the insufficient-stock `ConflictError`.
- `src/modules/inventory/repository.ts` — add `tankTypeLabel(id)` (select name/sizeLabel from `tank_types`); used only on the error branch.
- `src/modules/inventory/__tests__/helpers.ts` — `FakeInventoryRepository`: implement `tankTypeLabel` (seed-backed, default `Tipo #id`); add a `seedTankTypeName` helper.
- `src/modules/inventory/__tests__/` — assert the load stock-out 409 includes the tank-type name + guidance; assert a purchase for a never-stocked type succeeds.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-backend] Backend track **done**. `InventoryService.loadFromLocation`'s insufficient-store-stock `ConflictError` now **names the tank type** instead of the numeric id and **guides the operator to a Compra**: `La tienda no tiene stock suficiente de "Balón 45kg" (disponible: 0 llenos / 0 vacíos; solicitado: 1 / 0). Regístralo con una compra antes de cargarlo al repartidor.` The name is resolved lazily on the error branch via a new `IInventoryRepository.tankTypeLabel(id)` (selects `tank_types.name`; falls back to `tipo {id}` if unknown) — zero cost on the success path. Shared by `openDay` + `recordLoad` (both call `loadFromLocation`). **No schema/ledger/enum/migration change.** Fake: `tankTypeNames` map + `seedTankTypeName` + `tankTypeLabel`. Tests: new `multi-type-fulfillment.test.ts` (2) — (a) loading a never-stocked type → 409 that includes the **name** ("Balón 45kg"), excludes "tipo 2", and mentions "compra"; (b) a provider purchase for the never-stocked type creates its store holder and the subsequent load succeeds. Updated the existing edge-case assertion (`/insuficiente/i` → `/no tiene stock suficiente/i`). **87 tests** (was 85). Gates green: typecheck + lint + 87 tests + build. The smart **"Comprar y cargar"** resolution + per-type **Movimientos** legibility (Tanque column, `load`→"Carga") are the **frontend track** — all backend endpoints they compose (`tank-purchases`, `loads`, `availability`) already exist and `tankTransactionsByAssignment` already returns `tankTypeId`. **Frontend track remains.**