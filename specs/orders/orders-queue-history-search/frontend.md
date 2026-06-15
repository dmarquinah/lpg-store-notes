---
project: lpg-store
domain: specs
type: spec-track
spec: orders-queue-history-search
repo: lpg-frontend-vue
kind: frontend
track-status: not-started
last-updated: 2026-06-14
---

# Orders Queue / History / Search — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

Builds on the existing `orders` module. The store/service already carry
`dateFrom`/`dateTo` in `ListOrdersFilters` and the list query builder — only
`search` is new. The change is concentrated in `OrdersListView.vue`.

- **Active / History segmented control.** A segmented control (or tabs) at the top
  toggles two modes:
  - **Activos** (default): force status ∈ `pending|assigned|in_transit`, no date
    filter. The status dropdown narrows *within* this set.
  - **Historial**: status ∈ `delivered|cancelled|failed`, plus the date range.
  Model the mode as local state that maps to the appropriate `status`/date
  filters before calling `fetchOrders`. Keep the admin **store switcher** + the
  **branch column** (all-branches view) and the ownership/transfer affordances
  from [[../orders-multi-location/frontend|orders-multi-location]].
- **Date-range filter (Historial).** Two shared `DatePicker`s (Desde / Hasta) bound
  to `filters.dateFrom`/`dateTo`. Default to today; widening is the operator's
  call. No pager. Mirror the Inventario page's `DatePicker` usage.
- **Customer search box.** A debounced (≥2 chars, ~300ms) text input → set
  `filters.search` → `fetchOrders`. Mirror `CustomerSelect.vue`'s debounce; show a
  "Sin coincidencias" empty state. Available in both modes.
- **Store/types/service**: add `search` to `ListOrdersFilters` + the `listQuery`
  builder (one line each); `dateFrom`/`dateTo` are already supported.
- The **order-creation customer picker** (`CustomerSelect.vue`) needs no FE change
  — it gets accent-insensitive results for free once the backend fix lands; just
  verify "Maria" finds "María".

## Related Files

### lpg-frontend-vue

To modify:

- `src/modules/orders/views/OrdersListView.vue` — Activos/Historial segmented control, date-range pickers, customer search box (the bulk of the work)
- `src/modules/orders/types.ts` — `search?: string` on `ListOrdersFilters`
- `src/modules/orders/service.ts` — set `search` in `listQuery`
- `src/modules/orders/store.ts` — (only if a helper for mode→filter mapping is cleaner here than in the view)

Context (read):

- `src/components/ui/date-picker/DatePicker.vue` + `src/modules/inventory/views/InventoryView.vue` — the shared date-picker usage pattern
- `src/modules/orders/components/CustomerSelect.vue` — the debounced-search pattern to mirror (and the picker that benefits from the accent fix)
- the segmented/tabs primitive (`src/components/ui/tabs/`) if used for the mode toggle

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
