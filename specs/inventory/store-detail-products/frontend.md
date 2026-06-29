---
project: lpg-store
domain: specs
type: spec-track
spec: store-detail-products
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-27
---

# Store Detail Products — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

Refactor `StoreDetailView` (from [[../store-stock-first/frontend]]) into tabs +
purchased-only stock + lazy loads.

- **Holder-based tank stock.** Remove the `shopRows` catalog-merge computed; bind
  Stock en tienda directly to `store.availability?.shop ?? []`. Purchased-incl-0
  shows; never-purchased is absent. Empty-text → e.g. "Aún no se ha registrado
  ninguna compra de balones en esta tienda."
- **Tabs:** `<Tabs v-model="storeTab">` (reka-ui forwards `modelValue`, so a
  controlled value works) with **Balones · Artículos · Compras**.
  - **Balones:** Stock en tienda (tanks, holder-based) + En vehículos. Loaded on
    mount (default tab) via the existing `fetchAvailability(id, true)`.
  - **Artículos:** the store's item stock (name + qty) from the new endpoint —
    **lazy** (fetch on first open). Item names from `catalog.items`.
  - **Compras:** the existing purchases list + Desde/Hasta window + cost-
    correction — **lazy** (fetch on first open; today it's eager on mount).
- **Lazy mechanism:** `watch(storeTab)` → on `articulos` call
  `store.fetchStoreItemAvailability(id)`; on `compras` call
  `store.fetchStorePurchases(id)`. `watch(storeId)` (route change) → reset
  `storeTab='balones'` + `fetchAvailability` so a different store starts clean.
- **Service:** `getStoreItemAvailability(storeId)` → `GET
  /inventory/stores/:storeId/item-availability` (`{ items }`).
- **Store state:** `itemAvailability: ItemAvailabilityRow[]` + `loadingItemAvailability`
  + `fetchStoreItemAvailability(storeId)`.
- **Types:** `ItemAvailabilityRow { inventoryItemId: number; qty: number }` +
  `StoreItemAvailabilityResponse { items: ItemAvailabilityRow[] }`.
- "Registrar compra" stays in the page header (PurchaseDialog handles tanks OR
  items). Reuse `ResponsiveTable`, tokens, ≥44px (`eng/design-system.md`).

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/views/StoreDetailView.vue` — tabs, holder-based tank
  stock, lazy Artículos + Compras.
- `src/modules/inventory/store.ts` — `itemAvailability`/`loadingItemAvailability`
  + `fetchStoreItemAvailability`.
- `src/modules/inventory/service.ts` — `getStoreItemAvailability`.
- `src/modules/inventory/types.ts` — `ItemAvailabilityRow` +
  `StoreItemAvailabilityResponse`.
- `src/modules/catalog/store.ts` — `items` (name lookup; `fetchItems`).
- `src/components/ui/tabs/*`, `src/components/app/ResponsiveTable.vue` — reused.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-27] [lpg-frontend-vue] Frontend track **done**. `StoreDetailView`
restructured into **Balones · Artículos · Compras** tabs (controlled `<Tabs
v-model>` for lazy-on-open). **Tank stock is now holder-based** — binds
`store.availability?.shop` directly (dropped the catalog-merge computed): products
the store has purchased show (incl. those depleted to 0), never-purchased tank
types are **absent**; empty-state reads "Aún no se ha comprado ningún balón…".
**Artículos** + **Compras** **lazy-load on first tab open** (the default Balones
tab keeps loading tank availability on mount); a purchase close or Desde/Hasta
change refreshes the open lazy tab; navigating to another store resets to Balones.
New **Artículos** tab lists per-store item stock (name + qty + unit) from
`getStoreItemAvailability` → `GET /inventory/stores/:id/item-availability`
(consumes the [[backend]] contract — shows empty until that endpoint ships). Added
service `getStoreItemAvailability`, store `itemAvailability`/`loadingItemAvailability`/
`fetchStoreItemAvailability`, and types `ItemAvailabilityRow` /
`StoreItemAvailabilityResponse`. Gates green: `npm run typecheck` + `npm run build`
(PWA 76 entries / 782.52 KiB). **Backend track remains** (the item-availability
endpoint, in lpg-backend).