---
project: lpg-store
domain: specs
type: spec-track
spec: store-stock-adjustments
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-28
---

# Store Stock Adjustments — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

A new dialog + an affordance on the store-stock surface (the one re-led by
[[../store-stock-first/index|store-stock-first]]). Pattern: an eighth ledger
dialog alongside the existing `inventory` module dialogs (PurchaseDialog,
AdjustmentDialog, …).

- **"Ajustar stock" affordance** on the store-stock tab header (next to
  "Registrar compra"), visible to admin/developer and own-store operators (reuse
  the role check the page already does for the store options).
- **`StockAdjustmentDialog.vue`** — pre-fill the current per-type full/empty from
  `store.availability.shop` (or the catalog-merged computed from store-stock-first,
  so types at 0 are editable too). Per row: a **mode** of *absolute* ("set to") or
  *±* (adjust by) for full and empty. On submit, compute `delta = target −
  current` (absolute) or use the entered ± directly, drop zero-delta lines, and
  send `{ notes, lines: [{ tankTypeId, fullDelta, emptyDelta }] }` to the new
  endpoint.
- **Required reason.** A `notes` field — proposed as a small preset select
  ("Carga inicial" / "Conteo físico" / "Merma o daño" / "Otro") + free text on
  "Otro", mirroring the accounting `ManualEntryDialog` category pattern. Reads
  clearly later in the store history.
- **Client validation** mirrors the backend: integer counts; block submit if any
  resulting balance would be **negative** (absolute target `< 0`, or `current +
  delta < 0`). Surface backend **403** (cross-store) and **409** (would-go-
  negative) as clear messages.
- **On success:** close, refetch `fetchAvailability(storeId, includeTrucks)` so
  the store-stock table updates immediately; the row also becomes visible in the
  store history ([[../store-history/index]]).
- **Service/store:** add `recordLocationAdjustment(storeId, payload)` to
  `service.ts` + a store action wrapping it (set `saving`, clear `error`), beside
  the existing purchase actions.
- Design system: tokens only, ≥44px targets, `Spinner` on submit; integer inputs
  (no `formatMoney` — these are counts, not money). No raw palette classes
  (`eng/design-system.md`).

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/views/InventoryView.vue` — add the "Ajustar stock"
  button to the store-stock tab header; mount the new dialog.
- `src/modules/inventory/components/StockAdjustmentDialog.vue` — **new** dialog
  (absolute/± per type, required reason, negative-balance guard).
- `src/modules/inventory/service.ts` — `recordLocationAdjustment(storeId,
  payload)` → `POST /inventory/stores/:storeId/adjustments`.
- `src/modules/inventory/store.ts` — store action + `saving`/`error`; refetch
  availability on success.
- `src/modules/inventory/types.ts` — payload type (`notes` + `lines[]` with signed
  integer deltas).
- `src/modules/inventory/components/AdjustmentDialog.vue` — reference for the
  existing (assignment-scoped) adjustment dialog UX to stay consistent.
- `src/modules/accounting/components/ManualEntryDialog.vue` — reference for the
  preset-category + "Otro" reason pattern.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-28] [lpg-frontend-vue] Frontend track **done**. New
`StockAdjustmentDialog.vue` — set/correct a store's standing **tank** stock
directly. Two modes: **Establecer total** (inputs pre-filled to current; `delta =
target − current`) and **Ajustar (±)** (signed change, default 0). Rows are built
from **every active catalog tank type merged with the store's current location
balances**, so a brand-new store can be **seeded type by type from 0** in one
save. Required **Motivo** (preset Carga inicial / Conteo físico / Merma o daño /
Otro + free text on Otro). Client validation mirrors the backend: integers, target
≥ 0, and an adjustment may not drive a balance **negative**; backend **403/409**
surface via `store.error`. On save → `POST /inventory/stores/:id/adjustments`
(service `recordLocationAdjustment` + store action, which refetches availability so
the Balones tab updates at once). Mounted on **`StoreDetailView`** with an **"Ajustar
stock"** header button (outline, beside *Registrar compra*) — the store-stock
surface, which replaced the old InventoryView availability tab. Types
`LocationAdjustmentLine` / `LocationAdjustmentPayload`. Long type lists scroll in a
`scrollbar-thin` panel. Gates green (`npm run typecheck` + `npm run build`, PWA 76
entries). **Backend track** (`POST /stores/:id/adjustments`) pending in
lpg-backend; the dialog is wired to that contract.