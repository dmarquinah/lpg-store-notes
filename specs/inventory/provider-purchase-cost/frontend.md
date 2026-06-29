---
project: lpg-store
domain: specs
type: spec-track
spec: provider-purchase-cost
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-16
---

# Provider Purchase Cost — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

Additive change to the `inventory` module's **purchase** dialogs (part of the
seven ledger dialogs from `inventory-foundation`). Each purchase line gains an
editable **unit cost** field, pre-filled from the catalog purchase price the
module already loads for tank-types/items.

- **Required + pre-filled.** The cost field is always shown, **required**, and
  pre-filled with the **last purchase cost** for that store + product, falling
  back to the catalog `purchase_price` on the first purchase. The operator
  confirms or edits it every time — so a wrong default can't slip through
  silently. Source the last cost from the backend default (it returns the same
  resolution); if no dedicated field is returned, the catalog `purchase_price`
  the module already loads is the fallback pre-fill.
- **Validation** mirrors the backend Zod: number ≥ 0, ≤ 2 decimals, non-empty.
  Reuse the money-input pattern from the existing dialogs.
- **Correct after the fact.** The inventory **Movimientos / purchases** view
  gets an edit affordance on `purchase` rows → a small dialog that `PATCH`es the
  cost (`PATCH /inventory/purchases/:kind/:id/cost`), same validation. Gated to
  the same roles that record purchases. Lets the operator fix a fat-fingered
  cost. **Accounted purchases are frozen:** a purchase inside a **closed**
  registry returns **409** — surface it as a "periodo cerrado" message (and/or
  hide the edit affordance) rather than a generic error.
- **Wire `unitCost`** into the tank and item purchase request payloads sent to
  `POST /inventory/.../purchases` (the inventory service/store purchase actions).
- **Labelling:** make the field read as the **cost paid to the provider**,
  visually distinct from the sell price, so the operator understands they're
  entering cost-in (the whole point of the spec). Use `formatMoney` for any
  display and the petrol+flame tokens (no raw palette classes — see
  `eng/design-system.md`).
- **"Comprar y cargar" path** (orders `AssignDialog`, from
  `multi-type-fulfillment`): it registers a provider purchase of the missing
  part. Simplest: leave it sending no cost → backend uses the catalog default
  (acceptance criterion allows this). If a cost field is added there too,
  pre-fill it identically.

## Related Files

> Confirm exact filenames at `/focus` time — the inventory purchase dialogs and
> catalog price source are known by role, not yet read in this planning pass.

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/` — the tank + item **purchase** dialogs (two of the
  seven ledger dialogs): add the editable cost field + send `unitCost`.
- `src/modules/inventory/` service/store — purchase action payload types gain
  `unitCost`.
- `src/modules/catalog/` store/service — source of the `purchase_price` pre-fill
  for the selected tank-type/item.
- `src/modules/inventory/` Movimientos / purchases list — add an edit-cost
  affordance on `purchase` rows + a small cost-correction dialog.
- `src/modules/orders/` `AssignDialog` (Comprar y cargar) — unaffected (sends no
  cost → backend default).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-frontend-vue] Frontend track **done**. Captured an editable provider **cost-in** at purchase time and added a post-hoc cost-correction surface. **Purchase dialog** (`PurchaseDialog.vue`): a **required** per-line **"Costo del proveedor (por unidad)"** field for **both** tank and item modes, **pre-filled** from the catalog `purchasePrice` on product select (`onProductChange`) so a blank can't slip through; helper text "Lo que paga al proveedor, no el precio de venta." + a `formatMoney` sell-price reference (`costHint`) make it read as cost-in, distinct from sell price. Validation `parseCost` mirrors the backend Zod (non-empty, `≥ 0`, finite, `Math.round(n*100)===n*100` ≤2dp); `unitCost` wired into both `TankPurchaseLine`/`ItemPurchaseLine` payloads (optional on the type, always sent by the dialog). **Cost correction:** the **Availability** tab gained a **"Compras al proveedor"** table (Desde/Hasta `DatePicker` window, default today) listing the store's purchases from the **new `GET /inventory/stores/:storeId/purchases`** endpoint (enabled by the user in the backend repo, outside the original backend track). Each row shows Fecha/Tipo/Producto/Cantidad/Costo unit./Total (`formatMoney`); rows whose day is in a **closed registry (`accounted: true`)** render a **lock + "Congelado"** with no edit button, others an **"Editar costo"** action → new `CostCorrectionDialog.vue` that `PATCH`es `/inventory/purchases/:kind/:id/cost` (same `parseCost` validation) and refetches the list so an **open** registry's egress reflects the change live. **Service/store:** `listStorePurchases(storeId, {from,to})` + `updatePurchaseCost(kind, txId, unitCost, storeId)`; new `purchases`/`loadingPurchases`/`purchasesFrom`/`purchasesTo` state; `TankTransaction`/`ItemTransaction` types gained `unitCost: string | null` for wire fidelity. **`AssignDialog` "Comprar y cargar" left untouched** — still sends no cost → backend catalog default (criterion allows it). Independent validation: **all 5 frontend criteria met, no blocking bugs, no design-system violations** (tokens only, all money via `formatMoney`). Gates green: `npm run typecheck` clean + `npm run build` (PWA precache 68 entries / 759.29 KiB). Manual smoke (record a purchase with a cost below sell price → registry egress drops to the entered cost) left to the operator.
