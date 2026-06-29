---
project: lpg-store
domain: specs
type: spec-track
spec: store-history
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-28
---

# Store History — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

A new **Movimientos / Historial** tab/section on the store-stock surface (the
slot [[../store-stock-first/index|store-stock-first]] leaves), plus a lazy
order-events merge.

- **Tab/section** on `InventoryView.vue`, scoped to the page's selected store
  (`store.selectedStoreId`). Operator own-store / admin-dev all (reuse the page's
  existing store scoping).
- **Base load = inventory movements only.** On entering the tab (or store change),
  call `GET /inventory/stores/:storeId/history?from=&to=` and render newest-first.
  Each row: an **acción** label mapping the raw `kind` (extend the existing
  `KIND_LABELS` / `TX_KINDS` map — `purchase`→"Compra", `opening`→"Apertura",
  `load`→"Carga", `sale`→"Venta", `return`→"Devolución", `adjustment`→"Ajuste",
  `carry`→"Consolidación", …), **por** (actor `userName`), **fecha/hora**
  (`occurredAt`), the **movimiento** (±full / ±empty, or qty for items), the
  **nota**, and a **link** to `/pedidos/:refOrderId` or the customer when present.
  Use `ResponsiveTable` (stacks on phone) — see Open Question on timeline vs table.
- **Lazy order events.** A **"Ver eventos de pedidos"** button issues a *separate*
  request only on click (`GET /orders?storeId=&from=&to=` via the orders service,
  or the dedicated read if the backend adds one), maps order-lifecycle events into
  the same row shape, and merges them into the list by timestamp. The default tab
  load must issue **no** order request (verify in the network panel).
- **Date window** (Desde/Hasta `DatePicker`s, default last 30 days) mirroring the
  Compras al proveedor window already on the page; refetch on change.
- **Service/store:** add `fetchStoreHistory(storeId, {from,to})` +
  `fetchStoreOrderEvents(storeId, {from,to})` (lazy) to the inventory module, with
  their own `loading` flags so the order-events button shows its own spinner
  without blocking the base list.
- Design system: tokens only, `Spinner`, `EmptyState` ("Sin movimientos en el
  rango."), ≥44px targets, status→label via the shared mapping; no raw palette
  classes (`eng/design-system.md`).

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/views/InventoryView.vue` — add the **Movimientos /
  Historial** tab + the "Ver eventos de pedidos" button + date window.
- `src/modules/inventory/service.ts` — `fetchStoreHistory` (`GET
  /inventory/stores/:id/history`) + the lazy order-events read.
- `src/modules/inventory/store.ts` — history state + `loadingHistory` /
  `loadingOrderEvents`; window refs.
- `src/modules/inventory/types.ts` — history-entry type (mirror the backend view);
  extend `TX_KINDS` / `KIND_LABELS` if a kind is missing a label.
- `src/modules/orders/service.ts` — `GET /orders?storeId=` (the lazy order-events
  source) — read for the merge; no change unless a dedicated read is added.
- `src/components/app/ResponsiveTable.vue` — the table/stack component for the
  history rows.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-28] [lpg-frontend-vue] Frontend track **done**. New **Historial** tab on
`StoreDetailView` (Balones · Artículos · Compras · **Historial**). Base load =
**inventory movements only**, **lazy on first open**: `GET
/inventory/stores/:id/history?from=&to=` (service `getStoreHistory` + store
`history`/`loadingHistory` + `historyFrom`/`historyTo` window, default **last 30
days**). Each movement row: Fecha (`isoToDateTimeDisplay`), Acción (`KIND_LABELS`),
Producto, Movimiento (signed ±llenos/±vacíos for tanks, ±qty for items), **Por**
(`userName`), Nota + a link to `/pedidos/:refOrderId` when present. **Order events
stay lazy + separate**: a **"Ver eventos de pedidos"** button (`orderEventsLoaded`
gate) fetches `GET /orders?storeId=&dateFrom=&dateTo=` (reused — **no new
endpoint**; inventory keeps a slim `StoreOrderEvent` shape, not the orders
module's type) into a **second table** (id link · cliente · estado badge · total).
The default tab load issues **no** order request (validated). Window change
refetches the movements (and order events only if already shown); navigating to
another store resets to Balones + clears `orderEventsLoaded`. Adjustments from
[[../store-stock-adjustments/frontend]] surface as **"Ajuste"** rows with actor +
reason. Independent validation: **all 4 criteria PASS, no bugs**. Gates green
(`npm run typecheck` + `npm run build`, PWA 76 entries / 795.79 KiB).
**Deviation** from the draft: order events render as a **separate lazy table**
rather than merged into one timeline (cleaner; resolves the spec's table-vs-timeline
open question). **Backend track** (`GET /stores/:id/history`) pending in lpg-backend.