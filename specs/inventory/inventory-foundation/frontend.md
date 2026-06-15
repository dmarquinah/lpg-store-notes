---
project: lpg-store
domain: specs
type: spec-track
spec: inventory-foundation
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-06
---

# Inventory Foundation — lpg-frontend-vue track

Shared spec: [[index]] · Edge-case catalogue (E1–E11): [[../index]]

## Technical Notes

Not started — shaped once the backend track lands and the API is live. An `inventory` vertical module under `src/modules/inventory/` (per [[../../../eng/patterns/frontend-module-template]]) that drives the backend routes (see [[backend]]):

- **Assignment management**: open a day's assignment, list/filter assignments, view one with embedded balances.
- **Ledger entry**: forms for sale / purchase / return (and adjustment), including the empty-received field that drives empty-debt accrual (E1/E3/E5).
- **Close & carry**: a close form taking optional discrepancy counts, plus a discrepancies view over the `reconciliation` rows.
- Must surface balances (current full/empty per tank type, quantity per item) read from the backend's balance views — the frontend never re-derives sums.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To be created:

- `src/modules/inventory/` — vertical slice (types/service/store/routes/views: assignment list + detail, ledger-entry forms, close/discrepancy views)

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
(empty — frontend track not started)

[2026-06-06] [lpg-frontend-vue] Implemented the `inventory` vertical module (`src/modules/inventory/`), mirroring the `catalog` module. **types.ts**: mirrors backend public shapes (`AssignmentState` const+guard, `TxKind`, `InventoryAssignment` list row, `TankBalanceView`/`ItemBalanceView`/`AssignmentView`, `TankTransaction`, `AvailabilityResponse`) + request payloads + envelope interfaces, with a source-of-truth comment pointing at `lpg-backend/src/modules/inventory/{types,schema}.ts`. **service.ts**: `createInventoryService(apiClient)` — one method per endpoint, unwrapping each envelope key; availability returns the object directly (backend sends `{shop,onTruck}` un-enveloped). **store.ts**: `useInventoryStore` with module-scoped `provideInventoryService`; state for the assignment list (+ date/state filters), current detail (assignment+transactions+discrepancies fetched together), and store availability; a `runWrite` wrapper that toggles `saving`, surfaces `ApiError` messages, and re-pulls the current assignment after every ledger write so balances stay live (never re-derived client-side). **routes.ts/index.ts**: `/inventory` (list+availability) and `/inventory/assignments/:id` (detail), `roles: [admin, operator, developer]`; `createInventoryModule({apiClient})` wired in `main.ts`, routes spread in `router/index.ts`.

Views: **InventoryView** — tabbed (Asignaciones: date+state filters, "Abrir día", clickable rows → detail; Disponibilidad: store select + "incluir camiones" toggle + shop/on-truck tank tables + "Registrar compra"). **AssignmentDetailView** — header (date, state badge, driver/store), tank + item balance tables (from the views), state-aware actions (open: Venta/Devolución/Cargar/Ajuste/Cerrar día; closed: Ajuste/Consolidar; carried: read-only), discrepancies table (reconciliation rows), full ledger table. Seven dialogs (`OpenDay/Sale/Return/Adjustment/Load/Close/Purchase`) mirroring `TankTypeCreateDialog` with client-side validation that mirrors the backend Zod schemas; CloseDialog seeds final-count rows from the current view balances with a toggle to skip reconciliation.

**Picker:** the user shipped `GET /catalog/store-assignments` → the catalog module was extended (`StoreAssignmentDetail` type, `listStoreAssignments` service method, `storeAssignments` store state/`fetchStoreAssignments`); inventory reuses `useCatalogStore` for tank-type/item/store names and the open-day driver picker (cross-module via the catalog module's public exports — no duplicated fetch logic).

**Nav refactor:** centralized navigation in `AppLayout` (`ROLE_NAV` per role, used when a route omits the `nav` prop) — removed the three duplicated local `adminNav` arrays from `home`/`users`/`catalog` routes; operators get a drawer (Inicio + Inventario), admin/developer get the full set incl. Inventario. The shared inventory route renders the correct per-role nav without passing role-specific props.

**Verification:** `npm run typecheck` and `npm run build` both green. Independent validation agent confirmed all four code-verifiable frontend criteria met and every endpoint path/method/query-param/payload-field/envelope-key matches the backend exactly (incl. the un-enveloped availability response and the reka-ui Select empty-value pitfall being avoided via an "all" sentinel). The fifth criterion — a manual open→record→close→carry smoke test against a running backend — is pending live verification.
[2026-06-06] [lpg-frontend-vue] All criteria for this repo met (code + typecheck + build + independent validation). Frontend track marked **done**; the open→record→close→carry smoke test is left to the operator's manual run per the project's standing verification method.
[2026-06-06] [lpg-frontend-vue] Post-ship UI refinements (spec already `done`; no status change): (1) **Spanish route paths** — translated URL `path`s (`/usuarios`, `/catalogo`, `/inventario` + `/inventario/asignaciones/:id`, `/operador`, `/reparto`; `/admin` and `/login` kept) and the `AppLayout` `ROLE_NAV` literals; route **names** left as internal English IDs, so the guard/`ROLE_HOME_ROUTE`/`router.push`/`RouterLink` and the PWA `start_url` were untouched. (2) **Reusable `DatePicker`** — new shadcn-vue-style components `src/components/ui/{popover,calendar,date-picker}/` built on reka-ui `PopoverRoot`/`CalendarRoot` + `@internationalized/date` (added to deps; the project defers `@vuepic/vue-datepicker`), plus a frontend `src/lib/date.ts` helper (`todayISO`/`isoToDisplay`/ISO↔CalendarDate). `v-model` is an ISO `yyyy-MM-dd` string, displays `dd/MM/yyyy`, opens a calendar on click; used in the Asignaciones date filter (defaults to today, clearable) and the open-day dialog (optional). (3) **Disponibilidad** — last-selected store persisted via `useStorage` in the inventory store (`lpg.inventory.storeId`); vehicles always fetched (`includeTrucks` always on) with each panel (Stock en tienda / En vehículos) independently show/hide-able via an eye icon; renamed "camiones" → "vehículos" in the UI (backend `includeTrucks`/`onTruck` identifiers unchanged). typecheck + build green.
[2026-06-06] [lpg-frontend-vue] Two fixes: (1) `DatePicker` trigger made block-level (`flex w-full`; twMerge keeps `flex` over `buttonVariants`' `inline-flex`) so its field label stacks above like the Select fields — previously the inline `<label>` + inline-flex trigger shared a line. (2) **Batched purchases** — the backend replaced the single `/purchases` endpoint with `POST /stores/:id/tank-purchases` and `/item-purchases` (each `{ items: [{tankTypeId|inventoryItemId, qty}], notes? }`, unique products, returns `{ transactions }`). Frontend service/store updated to `recordTankPurchase`/`recordItemPurchase`; `PurchaseDialog` reworked to a mode selector (Balones | Artículos — a purchase is one or the other, never both) with a repeatable multi-line product list, client-side unique-product + positive-qty validation mirroring the Zod schemas. Tank purchases refresh availability; item purchases don't (items aren't in the tank availability view). typecheck + build green.
[2026-06-06] [lpg-frontend-vue] Mirrored the backend's negative-inventory guards client-side (backend added: over-assignment 409 on open/load, bounded purchase empties + surcharge side table, today-only open-day). New store helper `getShopBalances(storeId)` reads a store's shop balances on demand (without touching the Disponibilidad tab's shared state). **OpenDayDialog**: removed the date field entirely (open-day is today-only; server defaults to today) with a static "Se abre para hoy" line; on conductor select it loads that store's shop balances and blocks initial loads exceeding on-hand fulls/empties (aggregated per tank type), with a per-row "Disponible en tienda" hint. **LoadDialog**: resolves the assignment's store, loads shop balances, blocks loads beyond on-hand, shows the same hint. **PurchaseDialog (tanks)**: added optional `emptyReturned` + `surcharge` per line (mirrors the new schema), validates `emptyReturned ≤ qty` and `≤ empties on hand` (loaded for the selected store), with a "Vacíos en tienda" hint; items mode unchanged. typecheck + build green. Note: sales/returns are not bounded client-side because the backend doesn't guard them yet.