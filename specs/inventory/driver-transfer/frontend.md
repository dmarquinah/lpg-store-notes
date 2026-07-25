---
project: lpg-store
domain: specs
type: spec-track
spec: driver-transfer
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: 2026-07-20
---

# Driver Transfer — lpg-frontend-vue track

Shared spec: [[index]]

> Ports after the backend track exposes the re-point move + reactivate-link +
> order re-home. Mostly a rewiring of the existing **"Agregar repartidor"** carry
> path + making sure the views refresh so no stale duplicate shows.

## Technical Notes

- **`AddDriverDialog` carry path** (`src/modules/inventory/components/
  AddDriverDialog.vue`): route the carry (a driver picked who's active in another
  store) to the new **move-driver** endpoint instead of `seed-driver`. The manual
  onboarding path (new driver, optional starting inventory) keeps `seed-driver`.
  The link create step uses the reactivate-or-create behavior so a round-trip
  doesn't spawn duplicate rows. Toast: "se movió con su inventario y pedidos".
- **View refresh:** after a transfer, refresh `catalog.storeAssignments`,
  `store.todayAssignments`, `store.poolBreakdown`, and the availability so the old
  store drops the driver immediately and the new store shows them once. The
  `/inventario` Resumen "Hoy" must not show days on **inactive** links — if the
  backend cleanup (re-point) is in place this is automatic, but confirm the
  Resumen derives "Hoy" only from days on active links (belt-and-suspenders after
  the round-trip test).
- **No new order UI** — the order carry is entirely backend (re-point + port). The
  frontend just triggers the move and refreshes.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/components/AddDriverDialog.vue` — carry → move-driver
  endpoint; reactivate-or-create link; toast + refresh.
- `src/modules/inventory/service.ts` / `store.ts` — a `moveDriver` (or carry-mode)
  call; catalog `addStoreAssignment` already returns the link.
- `src/modules/inventory/views/InventoryView.vue` — confirm the Resumen "Hoy"
  ignores days on inactive links (post-transfer cleanliness).
- `src/modules/inventory/views/StoreDetailView.vue` — Asignaciones already filters
  to active drivers; ensure it refreshes after the dialog's `added`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-20] [lpg-frontend-vue] Frontend track implemented — carry path rewired to the move-as-re-point endpoint. typecheck ✓ · build ✓. Validation: all 8 shared acceptance criteria MET.

- **New `moveDriver` service + store action.** `service.ts`: `moveDriver(storeId, { fromStoreAssignmentId, toStoreAssignmentId })` → `POST /inventory/stores/:storeId/move-driver`, returning `{ toStoreAssignmentId, movedDays, tanks, items }`. `store.ts`: `moveDriver` action (saving/error wrapper, message "No se pudo mover al repartidor"), exported. `types.ts`: added `MoveDriverPayload`/`MoveDriverResult`; **removed** `fromStoreAssignmentId` from `SeedDriverPayload` (backend made `seed-driver` manual-only) + updated its doc comment.
- **`AddDriverDialog` carry branch → move.** The picked-driver-active-in-another-store path now: (1) `catalog.addStoreAssignment` (reactivate-or-create — a round-trip's inactive link is reactivated, not duplicated, per the backend catalog change), (2) `inv.moveDriver({ from: sourceLink.id, to: link.id })` — pool + active day(s) + those days' orders all follow, (3) `deactivateStoreAssignment(sourceLink.id)` **after** the move (backend requires the source link active to move). Manual/onboarding branch unchanged (`seedDriver` with tanks/items). Driver name + source id captured **before** mutations (post-move the `sourceLink`/`selectedName` computeds recompute to empty). Toast: "…se movió con su inventario y pedidos." Header + carry-box copy updated ("Mover desde {store}", mentions día activo + pedidos).
- **Resumen "Hoy" gated to active links (belt-and-suspenders).** `InventoryView.vue`: new `activeDeliverySaIds` set; `todaysForStore` now requires the day's `storeAssignmentId` to be an active delivery link — a day left on a since-retired link can't surface a ghost driver. Stale/unresolved days still surface via the separate ungated `stalePriorDays` banner, so nothing legit is hidden. `StoreDetailView` already refreshes via the dialog's `added` emit (availability + poolBreakdown + todayAssignments) + the catalog refetch inside add/deactivate — no change needed.
- **No new order UI** (order re-home is entirely backend/port). No dangling `fromStoreAssignmentId` seed references (grep-confirmed).
