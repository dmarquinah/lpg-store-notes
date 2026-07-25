---
project: lpg-store
domain: specs
type: spec-track
spec: quick-assignment
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-07-04
---

# Quick Assignment — lpg-frontend-vue track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

Two changes in the **inventory** module: a quick-open button on the Resumen
cards, and a store-scoped driver list in the general dialog.

### Quick-open button (Resumen cards)

- **Where:** `views/InventoryView.vue`, the per-store Resumen card (from
  [[../store-stock-first/index|store-stock-first]]). The card already fetches
  today's driver/day-state for the store, so it knows whether a day is open.
- **When shown:** store has **exactly one** active delivery **and** no open day
  today → render a primary **"Asignar todo el stock a {driver}"** button. If a day
  is already open → show the day state (no button, as today). If **0/>1**
  deliveries → show a secondary link that opens the general `OpenDayDialog` (the
  manual path).
- **Action:** `POST /inventory/stores/:storeId/quick-open` via a new store/service
  method (`inventory/service.ts` + `store.ts`). On success: success toast naming
  the driver ("Se asignó todo el stock a {driver}"), refetch the Resumen so the
  card flips to its open-day state. Surface backend **409** (already open /
  pending prior day / ambiguous delivery) and **403** as clear inline/toast
  messages.
- The single-delivery signal can come from the existing store-assignment data the
  admin/org views already load (`listStoreAssignments({ storeId, role:
  'delivery' })`); if the Resumen payload doesn't carry driver count, add that
  read (or extend the backend Resumen — coordinate with [[backend]]).

### Store-scoped driver list (general dialog)

- `components/OpenDayDialog.vue` currently lists drivers unscoped. Filter the
  picker to **delivery users assigned to the selected store** via
  `listStoreAssignments({ storeId, role: 'delivery' })` (catalog service already
  exposed to the frontend). The `storeAssignmentId` the dialog submits to
  `openDay` should come from that scoped list.

### Purchase dialogs — mostly no change, BUT mind the auto-load side effect

Auto-load is a backend side effect: **the backend shipped** so that a provider
purchase into a **single-delivery store whose driver has an OPEN day today** now
**auto-loads the purchased stock (tanks + items) straight onto that driver**, in
the purchase's own transaction (see [[backend]] Implementation Notes, 2026-07-04).
For the plain **Compra** dialog this needs **no new control** — just refetch
after the purchase and the stock shows on the driver's day (verify in smoke).

### ⚠️ "Comprar y cargar" must NOT double-load on single-delivery stores

The [[../multi-type-fulfillment/index|multi-type-fulfillment]] **"Comprar y
cargar"** resolution is a **two-call** frontend flow: `POST …/tank-purchases`
(purchase the missing type into the store) **then** `POST /assignments/:id/loads`
(load it onto the driver). With backend auto-load now live, for a
**single-delivery store with an open day** the **first** call already moves the
stock onto the truck — so the **second** (explicit load) call finds an **empty
location and returns a stock-out 409**. The inventory end-state is already
correct (the type is on the truck); only the redundant second call errors.

**Required frontend change:** in the "Comprar y cargar" path, when the target
store has a **single delivery driver with an open day**, **skip the explicit
load** (the purchase auto-loads) — or treat the follow-up load's stock-out 409 as
a benign no-op and refetch. Detect single-delivery the same way the quick-open
button does (`listStoreAssignments({ storeId, role: 'delivery' })` → exactly one).
For **multi-delivery** stores (0/>1 drivers) the backend does **not** auto-load,
so the existing two-step flow stays correct and must be preserved.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify:

- `src/modules/inventory/views/InventoryView.vue` — quick-open button on the
  Resumen store cards; conditional on single-delivery + no-open-day.
- `src/modules/inventory/components/OpenDayDialog.vue` — scope the driver picker
  to the selected store.
- `src/modules/inventory/service.ts` — `quickOpen(storeId)` API call.
- `src/modules/inventory/store.ts` — action + refetch wiring; single-delivery /
  open-day state for the card.
- The **"Comprar y cargar"** assign-resolution (from
  [[../multi-type-fulfillment/index|multi-type-fulfillment]], in the orders/assign
  flow — grep the frontend for `Comprar y cargar` / the purchase-then-load
  sequence) — make it **single-delivery aware**: skip the explicit load (or ignore
  the follow-up 409) when the store has one delivery driver with an open day; keep
  the two-step flow for multi-delivery stores.

Read-only context (no change):

- `src/modules/catalog/` (or wherever `listStoreAssignments` is consumed) — the
  store→driver read used to scope the dialog and detect single-delivery.
- `src/modules/inventory/views/StoreDetailView.vue` — purchase dialogs live off
  here; confirm the post-purchase refetch reflects the auto-loaded stock.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-04] [lpg-frontend-vue] Frontend track DONE — all 6 shared frontend criteria met; independent validation GREEN (no bugs); gates green (`npm run typecheck` + `npm run build`). End-to-end smoke (login + live backend) left to the user — not runnable in the dev env.

**What shipped**

- **Toast infra (new).** `npx shadcn-vue add sonner` → `vue-sonner@^2.0.9` + `src/components/ui/sonner/{Sonner.vue,index.ts}`. Mounted `<Toaster :theme rich-colors close-button position="top-right" />` once in `App.vue` (theme mapped from `useTheme` mode → light/dark), imported `vue-sonner/style.css` in `main.ts`. First toast host in the app; success/error feedback now available app-wide via `toast()`.
- **Service + store.** `quickOpen(storeId)` in `inventory/service.ts` → `POST /inventory/stores/:storeId/quick-open` (empty body, unwraps `{ assignment }`); `store.ts` `quickOpen` action (toggles `saving`, surfaces error, returns `AssignmentView | null`). No auto-navigation — stays on `/inventario` so the operator can sweep several stores.
- **Quick-open button (Resumen cards, `InventoryView.vue`).** Derived state from the already-loaded `catalog.storeAssignments` (mounted with `all=true`): `soleDriverByStore` (storeId → sole active delivery assignment, ONLY for stores with exactly 1) + `storesWithTodayDay` (any today assignment). Primary **"Asignar todo el stock a {driver}"** shown when `soleDriverByStore.get(id) && !storesWithTodayDay.has(id)` → `onQuickOpen` → `toast.success("Se asignó todo el stock a {driver}")` + refetch overview/today (card flips); backend 409/403 → `toast.error(store.error)` then clears it so the persistent banner doesn't echo. Per-card spinner + all-buttons-disabled during a run. NO new fetch needed (resolved the track's "add that read" open item). Footer restructured: quick button (full-width) → Ver (outline when quick present, else primary) + general "Abrir día" (only for 0/>1 delivery) + Compra (ghost).
- **General dialog store-scoping (`OpenDayDialog.vue`).** New optional `storeId` prop; driver picker now filters `a.active && a.user.role === 'delivery' && (storeId === undefined || a.store.id === storeId)` (was active-only, unscoped, role-agnostic). Card "Abrir día" passes the card's storeId; the Asignaciones-tab "Abrir día" passes none (all active delivery). Empty-picker hint added (store- vs global-worded).
- **"Abastecer y cargar" single-delivery aware (`orders/AssignDialog.vue`).** `isSingleDelivery` computed from `catalog.storeAssignments` for the order's store. `resolveShortfalls` now loads `isSingleDelivery ? min(have, shortfall) : shortfall` and skips the load when that is 0 — because for a single-delivery open-day store the backend auto-loads the just-purchased part onto the driver, so only the store's residual on-hand needs an explicit load (loading the full shortfall would find an empty location and 409). Driver ends with exactly `shortfall` in every case (have<shortfall / have=0 / have≥shortfall); multi/0-delivery keep the full two-step. **Wording:** button `Comprar y cargar` → **`Abastecer y cargar`** (owner's pick — 'Comprar' read as ambiguous; 'Abastecer' = provider restock, distinct from a sale); added explicit behind-the-scenes copy for single-delivery stores describing the auto-load in the warnings phase.

**Decisions (resolved with the owner at `/focus`)**

- **Success feedback = real toasts** (owner chose adding `vue-sonner` over an inline Alert) — first toast infra in the app.
- **Dialog scoping = optional `storeId` prop** (cards pass it; Asignaciones-tab lists all delivery) rather than adding a store selector to the dialog.
- **"Abastecer y cargar" correctness = buy-missing + load-residual `min(have, shortfall)`** (not the spec's looser "skip/ignore 409", which under-delivers by `have` when the store floor holds partial stock). Made explicit in the dialog copy per the owner's transparency ask.
- **Button label = "Abastecer y cargar"** (owner picked over 'Reponer'/'Comprar al proveedor'). NB: the `/inventario` **"Compra"** card button was left untouched — it's owned by the sibling `inventory-ux-pass` draft spec.

**Files:** `App.vue`, `main.ts`, `components/ui/sonner/*` (new), `modules/inventory/{service,store}.ts`, `modules/inventory/views/InventoryView.vue`, `modules/inventory/components/OpenDayDialog.vue`, `modules/orders/components/AssignDialog.vue`, `package.json`.


[2026-07-04] [lpg-frontend-vue] Post-ship UX refinement (owner feedback). typecheck + build green.

- **Purchase auto-load now surfaced in the Compra dialog.** `PurchaseDialog` computes `autoLoadDriver` (store has exactly one active delivery **and** that driver has an `open` day today — derived from `catalog.storeAssignments` + `store.todayAssignments`, both fetched on open so it's accurate from the overview OR a store-detail page). When set, an **`info` Alert** ("Esta tienda tiene un solo repartidor con día abierto: la compra se cargará automáticamente a **{driver}**…") shows before the form. Previously the auto-load was a silent backend side effect — the earlier AC said "no new control", but the owner wanted the operator informed.
- **Confirmation toast after a purchase.** `handleSubmit` now toasts on success — `"Compra registrada y cargada a {driver}."` when auto-load applies, else `"Compra registrada."` (target captured pre-await).
- **New shared `Alert` `info` variant** (`border-info/40 bg-info/10 text-info-text`) mirroring the existing `warning`/`destructive` tinted approach (info tokens already existed).
- **Toast restyle + reposition (owner ask: "closer to our UI", mobile-friendly).** `App.vue` Toaster moved to **`position="bottom-center"`** (thumb reach on phones) and dropped `rich-colors`; `Sonner.vue` per-type classes now mirror the Alert variants (`!bg-{success,error→destructive,info,warning}/10` + `!border-…/40` + `!text-…-text`) on the neutral Card surface, so toasts read as part of the same design language instead of vue-sonner's default palette. Verified the type classes land in the built CSS.

**Files (refinement):** `App.vue`, `components/ui/sonner/Sonner.vue`, `components/ui/alert/index.ts`, `modules/inventory/components/PurchaseDialog.vue`.


[2026-07-04] [lpg-frontend-vue] Toast visual polish (owner feedback round 2). typecheck + build green.

- **On-palette colors.** Dropped the semantic full-surface tint (owner: colors didn't feel part of the UI). Toasts now use the **neutral Card surface** (bg-background/border-border/text-foreground) — identical to dialogs/popovers — with the type signalled by a **colored icon** only (`text-success`/`text-destructive`/`text-info`/`text-warning` in `Sonner.vue`).
- **Close button → top-right** via vue-sonner's `close-button-position="top-right"` prop (built-in; sets the `--toast-close-button-*` vars).
- **Auto-dismiss 2s** — `:duration="2000"` on the Toaster.
- **Countdown bar.** A thin bar at the bottom of each toast drains left→right over the 2s window (`style.css` `[data-sonner-toast]::after` + `@keyframes toast-progress`), colored per type via a `--toast-progress` CSS var set by the Sonner per-type classes (neutral fallback). Inset from the rounded corners; no `overflow:hidden` (would clip the outside close button). Caveat recorded in the CSS comment: sonner pauses the dismiss timer on hover but the CSS animation doesn't — acceptable cosmetic gap at 2s; the 2s duration is duplicated in `App.vue` + `style.css` and must stay in sync.

**Files (round 2):** `App.vue`, `components/ui/sonner/Sonner.vue`, `style.css`.