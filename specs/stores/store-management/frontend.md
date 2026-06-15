---
project: lpg-store
domain: specs
type: spec-track
spec: store-management
repo: lpg-frontend-vue
kind: frontend
track-status: done
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

[2026-06-14] [lpg-frontend-vue] Frontend track implemented. Extended the existing `catalog` vertical module — no new routes/nav (Catálogo already admin-gated).

- **types.ts** — added `CreateStorePayload` (name + optional address/phone), `UpdateStorePayload` (all-optional; `address`/`phone` `string | null` so an explicit `null` clears the column, mirroring the backend `.nullable()` update schema), `CreateStoreAssignmentPayload` ({storeId,userId}), and `StoreResponse`/`StoreAssignmentResponse` envelopes.
- **service.ts** — `createStore` (`POST /catalog/stores`), `updateStore` (`PATCH /catalog/stores/:id`), `createStoreAssignment` (`POST /catalog/store-assignments`), `setStoreAssignmentActive` (`PATCH /catalog/store-assignments/:id`). Mirror the existing `createTankType`/`createItem` shape.
- **store.ts** — actions `createStore`/`updateStore` (refetch `fetchStores`), `createStoreAssignment`/`deactivateStoreAssignment` (refetch `fetchStoreAssignments(showInactive)`), each toggling `loading`/`error`. The apiClient already maps **409 → ConflictError** carrying the backend Spanish message, and `messageFrom` surfaces it directly — dup store name / dup active assignment show the friendly text in the dialog Alert with no extra wiring.
- **StoreFormDialog.vue** (new) — mirrors `TankTypeCreateDialog`; `store?` prop → create (null) / edit. Fields name (≤120) + optional address (≤255) / phone (≤32), client-side validation mirroring backend Zod; an **Activa** Switch in edit mode (activate/deactivate). Create omits blank optionals; edit sends blanks as `null` to clear.
- **StoreAssignmentDialog.vue** (new) — two `Select`s (mirroring `TransferDialog`): **user** (reuses `useUsersStore`, fetched `{active:true}`, client-filtered to active **operator/delivery**) + **store** (active catalog stores). Submit → `createStoreAssignment`; 409 renders in the Alert.
- **CatalogView.vue** — Tiendas tab now has **Nueva tienda** + per-row **Editar** (Acciones column; removed the read-only note); new **Asignaciones** tab (Usuario · Rol · Tienda · Estado · Desactivar) with an **Asignar** button. `fetchAll()` adds `fetchStoreAssignments(showInactive)`; the show-inactive toggle re-fetches all four lists. Design-system compliant (PageHeader, TableEmpty/Spinner, success/outline badges, tokens only, canonical dialog chrome).

After creating a store it's immediately usable in the orders store switcher / TransferDialog (both call `catalog.fetchStores`) — no extra wiring.

Gates green: typecheck (`vue-tsc -b --noEmit`) ✓ · build (`vue-tsc -b && vite build`, PWA precache 53 entries / 673.80 KiB) ✓. No test runner wired in this repo — manual both-theme smoke left to the operator.

[2026-06-14] [lpg-frontend-vue] All criteria for this repo met. Independent validation confirmed all 3 frontend acceptance criteria; the assignee picker restricting to operator/delivery is intentional per this track's Technical Notes (backend allows any role, frontend offers the scoped roles). Gates green (typecheck + build). Frontend track **done** → both tracks done → spec **done**.