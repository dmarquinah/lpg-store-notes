---
project: lpg-store
domain: specs
type: spec-track
spec: accounting-registry
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-15
---

# Accounting Registry — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend track: [[backend]]

## Technical Notes

A new `accounting` vertical module mirroring the existing module convention (`users`/`catalog`/`orders` as precedent — list view + detail + dialogs, a Pinia store, a typed service). Admin/developer + authorized roles only; new nav entry in `AppLayout`'s `ROLE_NAV`.

Starts only **after** the backend track is `done` (it consumes the registry API). Build against the design system — no bespoke palette, use `formatMoney`, `PageHeader`, `Spinner`, `EmptyState`, the canonical status→badge mapping (see [[../../ui-design/design-system/index]] / `eng/design-system.md`).

### Screens

- **Registry list** (`/contabilidad` or similar) — per-store list of registers with status badges (`open`/`closed`), period (Lima dates), and net. Admin/developer get a **store switcher** across all stores (mirror the orders queue's `?storeId=` switcher); scoped users see only their own store(s). **"Nuevo registro"** dialog opens a period for a store (start/end date pickers reusing the shared `DatePicker`); surface the backend **overlap 409** as a friendly field/dialog error (existing `ConflictError` → message mapping).
- **Registry detail** — the breakdown:
  - **Ingresos** grouped by payment method (cash / yape / plin / transfer) with per-method + total amounts.
  - **Egresos**: provider purchases (read-only, sourced from inventory) and manual egress lines.
  - **Manual entries**: an add form (direction, amount, category, optional method, note) + per-row remove — **shown only while the register is `open`**.
  - **Net** total, prominently.
  - All money via `formatMoney`.
- **Cerrar registro** (admin) — confirm dialog; on close the detail switches to **read-only** against the frozen snapshot (no add/remove; clear "Cerrado" state with closed-by/closed-at).

### Reuse

- `formatMoney` (`lib/format`), `PageHeader`/`Spinner`/`EmptyState`, `Tabs`, `DatePicker`, shadcn-vue dialog/table — all already in the design system.
- Store switcher + `ConflictError` mapping patterns from the orders module.
- Client-side validation mirroring the backend Zod (amount > 0, required category) as elsewhere.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To create:

- `src/modules/accounting/` — `service.ts` (typed client over `/accounting/registries…`), `store.ts` (Pinia), `types.ts` (mirror backend responses), views (`AccountingListView.vue`, `AccountingDetailView.vue`), dialogs (`RegistryCreateDialog.vue`, `ManualEntryDialog.vue`), routes.
- `src/router` — guarded routes (admin/developer + authorized roles).

To modify:

- `src/components/app/AppLayout.vue` — add "Contabilidad" to `ROLE_NAV` for the permitted roles.

Reference (read for patterns, do not modify): `src/modules/orders/` (store switcher, `ConflictError` mapping, queue/detail shape), `src/modules/catalog/` (tabbed admin list + create dialog), `src/lib/format.ts` (`formatMoney`), `src/components/app/{PageHeader,Spinner,EmptyState}`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-frontend-vue] Frontend track **done**. New `accounting` vertical module (`src/modules/accounting/{types,service,store,routes,index}.ts` + `components/` + `views/`) mirroring the orders/catalog conventions, composed in `main.ts` and `router/index.ts`; "Contabilidad" nav (DollarSign) added to `src/layouts/AppLayout.vue` ROLE_NAV for admin/developer/operator. **Service** wraps `/accounting/registries…` — list (`{registries}`), get/add-entry/remove-entry/close (unwrapped `RegistryDetail`), open (`{registry}`). **Store** is a Pinia setup store with module-scoped service injection + a `conflict` flag (overlap 409 → friendly dialog message via `ConflictError`). **List view**: per-store table with status filter (Todos/Abierto/Cerrado), admin/dev store switcher + operator own-store scope banner (reuses `catalog.stores`/`catalog.myStores`), and **net/ingresos/egresos columns** sourced from the backend's `RegistrySummary` totals (no N+1; `null` → "—"). **Detail view**: Ingresos card (per-method operational + manual) and Egresos card (provider tanks/items/recargos + manual) via `formatMoney`, manual add/remove **only while `open`**, prominent **Neto**, admin-only **Cerrar registro** confirm → read-only frozen state with closed-at. **Dialogs**: `RegistryCreateDialog` (role-scoped store select, Lima `DatePicker` dates, overlap 409 surfaced, navigates to the new detail) + `ManualEntryDialog` (direction, amount >0 ≤2dp, direction-aware category presets + "Otro" free text, optional method for ingress, optional note). Design-system: tokens only (`text-success-text`/`text-destructive-text`); canonical status→variant (open→info, closed→secondary) added to `eng/design-system.md` first. Independent validation: **all 5 frontend criteria + design-system rules MET, no gaps**. Gates green: `npm run typecheck` + `npm run build`.

**Path note:** the spec listed `src/components/app/AppLayout.vue`; the real shell is `src/layouts/AppLayout.vue` (modified there).