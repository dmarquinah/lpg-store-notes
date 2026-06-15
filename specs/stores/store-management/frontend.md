---
project: lpg-store
domain: specs
type: spec-track
spec: store-management
repo: lpg-frontend-vue
kind: frontend
track-status: not-started
last-updated: 2026-06-14
---

# Store & Assignment Management — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

Extends the existing `catalog` vertical module. The catalog page is already a
tabbed admin surface (Tipos de tanque / Artículos / read-only Tiendas), so the
work is: make Tiendas writable + add an Asignaciones tab.

- **Tiendas tab → create/edit.** Add a `StoreFormDialog.vue` mirroring
  `TankTypeCreateDialog`/`ItemCreateDialog` (name required + address/phone
  optional; client-side validation mirroring the backend Zod). Wire create + edit
  + a show-inactive toggle + activate/deactivate, using new catalog store actions
  (`createStore`/`updateStore`) over `POST /catalog/stores` / `PATCH
  /catalog/stores/:id`.
- **Asignaciones tab.** New tab listing active `store_assignments` (who · which
  store), with an "Asignar" dialog: pick a **user** (from the users module — reuse
  its list/search; filter to operator/delivery) + a **store** (catalog stores),
  and a deactivate action per row. New catalog store actions
  `createStoreAssignment`/`deactivateStoreAssignment` over the new endpoints; the
  module already has `fetchStoreAssignments`.
- The catalog `store`/`service`/`types` already model `PublicStore` +
  `StoreAssignmentDetail` and the read calls — add the create/update payloads +
  actions alongside the existing `createTankType`/`createItem` pattern.
- Admin/developer only; "Catálogo" is already in the admin nav (no nav change).
  After creating a store it's immediately available in the orders store switcher /
  transfer dialog (both call `catalog.fetchStores`).

## Related Files

### lpg-frontend-vue

To modify:

- `src/modules/catalog/views/CatalogView.vue` — Tiendas tab create/edit + show-inactive; new Asignaciones tab
- `src/modules/catalog/components/StoreFormDialog.vue` — new (mirror `TankTypeCreateDialog`)
- `src/modules/catalog/components/StoreAssignmentDialog.vue` — new (user + store pickers)
- `src/modules/catalog/{types,service,store}.ts` — create/update store + create/deactivate assignment payloads, service calls, store actions
- (read) `src/modules/users/*` — the user list/search to pick an assignee

Context (read):

- `src/modules/catalog/components/{TankTypeCreateDialog,ItemCreateDialog}.vue` — the create-dialog pattern to mirror
- `src/components/ui/tabs/` — the catalog tab primitive
- `src/modules/orders/components/TransferDialog.vue` + `OrdersListView.vue` — the consumers of `catalog.fetchStores` (reference only)

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
