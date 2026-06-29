---
project: lpg-store
domain: specs
type: spec-track
spec: store-stock-first
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-27
---

# Store Stock First — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

A redesign of the `inventory` module's main view into a multi-location overview,
plus a new per-store detail route that absorbs the old "Disponibilidad" content.
Built against the [[backend]] aggregate contract (`GET /inventory/availability`);
the overview is live once that endpoint ships.

### Store + service

- **Service:** add `getStoresAvailability()` → `GET /inventory/availability`
  returning `StoreAvailability[]` (`{ storeId, storeName, shop[], onTruck[] }`,
  envelope key `stores`). Mirror the existing `getAvailability` method.
- **Store state:** `overview: StoreAvailability[]` + `loadingOverview`, with
  `fetchOverview()`. Keep the existing per-store `availability` + `fetchAvailability`
  for the detail route.
- **Today's assignments, decoupled from the filter:** the Asignaciones tab mutates
  `filterDate`/`filterState` (and thus `assignments`), so the overview must not
  read `assignments`. Add `todayAssignments: InventoryAssignment[]` +
  `fetchTodayAssignments()` (`listAssignments({ date: todayISO() })`, caller-scoped)
  for the per-store day status.

### InventoryView (the overview)

- Tabs → **`[ Resumen (default) · Asignaciones · Por consolidar ]`**; drop the
  `availability` tab. `default-value="overview"`. Keep the stale-day banner above
  the tabs and the consolidation badge on "Por consolidar".
- **Resumen** = a responsive grid of **location cards**, one per `overview` entry
  (operator → their stores, admin/dev → all — the backend already scopes this).
  Per card:
  - Header: store name.
  - Totals: **Llenos** = Σ shop.fullTanks, **Vacíos** = Σ shop.emptyTanks.
  - Per-type list: the shop rows with stock (`tankName(tankTypeId)` via catalog);
    if none, "sin stock aún" (never blank).
  - **En vehículos**: Σ onTruck.fullTanks (+ empties if useful).
  - **Hoy**: the `todayAssignments` whose `storeAssignmentId` maps to this store
    (via a `catalog.storeAssignments` → `{storeId, driverName}` map) — each as
    `driver · <state badge>`; if none, "Sin asignación" + an "Abrir día" action.
  - Actions: **Ver** (→ `/inventario/tiendas/:id`), **Registrar compra**
    (opens PurchaseDialog for that store), **Abrir día** (OpenDayDialog).
- onMounted: `fetchOverview()`, `fetchTodayAssignments()`, plus the existing
  `fetchAssignments`/`fetchConsolidationList`/`fetchStaleOpenDays` and catalog
  fetches (tank types + store assignments for the maps). The persisted
  `selectedStoreId` is no longer needed to land somewhere — the overview shows all.

### StoreDetailView (`/inventario/tiendas/:id`)

- New view + child route under `/inventario` (`props: true`, `meta.roles` =
  `["admin","operator","developer"]` like the parent). Receives `id` (storeId).
- Moves the old Disponibilidad content for one store: **Stock en tienda**
  (never-blank — merge `catalog.tankTypes` with `availability.shop`, default 0/0),
  **En vehículos**, **Compras al proveedor** + cost-correction, **Registrar
  compra**. Reuse `PurchaseDialog` / `CostCorrectionDialog` as-is, driven by the
  route's `storeId`. A back link to `/inventario`.
- On mount / id change: `fetchAvailability(id, true)` + `fetchStorePurchases(id)`
  (reuse the store's existing purchase window refs/watchers).
- **Never-blank merge** (the bug-symptom fix): rows = every active
  `catalog.tankTypes` left-joined to `availability.shop` by `tankTypeId`, missing →
  `{ fullTanks: 0, emptyTanks: 0 }`. `empty-text` only shows when there are no
  active tank types at all.

### Design system

Tokens only, `ResponsiveTable` for the detail tables, `PageHeader`, `Spinner`,
`EmptyState`, `formatMoney`, ≥44px touch targets, canonical status→badge mapping
(reuse `STATE_LABELS`/`STATE_VARIANT`). No raw palette classes
(`eng/design-system.md`). Cards use `Card`/design tokens.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/views/InventoryView.vue` — refactor: tabs
  `[Resumen, Asignaciones, Por consolidar]`, Resumen card grid; remove the
  Disponibilidad tab.
- `src/modules/inventory/views/StoreDetailView.vue` — **new** per-store detail
  (moved stock table + on-truck + purchases + cost-correction + Registrar compra).
- `src/modules/inventory/routes.ts` — add child route
  `tiendas/:id` → `StoreDetailView` (`props: true`).
- `src/modules/inventory/store.ts` — `overview`/`loadingOverview` + `fetchOverview`;
  `todayAssignments` + `fetchTodayAssignments`.
- `src/modules/inventory/service.ts` — `getStoresAvailability()` (`GET
  /inventory/availability`).
- `src/modules/inventory/types.ts` — `StoreAvailability` + `StoresAvailabilityResponse`.
- `src/modules/catalog/store.ts` — `storeAssignments` (storeAssignmentId → store /
  driver map for the card "Hoy" line; already loaded).
- `src/components/app/ResponsiveTable.vue`, `PageHeader.vue`, `EmptyState.vue` —
  reused.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-27] [lpg-frontend-vue] Frontend track **done**. Redesigned `/inventario`
into a **multi-location overview**. Service `getStoresAvailability()` → `GET
/inventory/availability`; store `overview`/`fetchOverview` + `todayAssignments`/
`fetchTodayAssignments` (the latter **decoupled** from the Asignaciones date/state
filter so changing it never alters the overview). **InventoryView** tabs reordered
to **[ Resumen (default) · Asignaciones · Por consolidar ]**; the **Disponibilidad
tab is retired**. **Resumen** = a responsive `Card` grid (one per store the caller
oversees, from the aggregate) — Llenos/Vacíos totals, a per-type list of held types
(else "Sin stock aún."), an "En vehículos" total, today's driver(s) + state badge
(else "Sin asignación hoy."), and actions **Ver · Compra · Abrir día**;
**never-blank** (a 0-stock store still shows a card at 0/0). New **`StoreDetailView`**
at `/inventario/tiendas/:id` (child route, `props:true`, parent `meta.roles`)
absorbs the former Disponibilidad content — **never-blank** Stock en tienda (merges
`catalog.tankTypes` so every active type shows at 0/0, keeps deactivated types that
still hold stock, defensively appends holders missing from the catalog), En
vehículos, Compras al proveedor + cost-correction, Registrar compra (reuses
`PurchaseDialog`/`CostCorrectionDialog` driven by the route `storeId`). A card's
**Ver** routes here; after a card purchase the overview re-fetches (watch on the
dialog open state). `catalog.fetchStoreAssignments(true)` so the card "Hoy" status
+ labels resolve even a since-deactivated link (OpenDayDialog filters active
itself). Removed the now-orphaned persisted `selectedStoreId` (+ its `useStorage`
import). Independent validation: **all 6 frontend criteria MET**, no correctness
bugs (one low-likelihood today-status silent-drop fixed via `all:true`). Backend
contract **verified** against the live endpoint (shape/scoping/empty-store match).
Gates green: `npm run typecheck` + `npm run build` (PWA 76 entries / 778.13 KiB).
Manual smoke (cards per store, 0-state, drill-in, purchase refresh) left to the
operator.