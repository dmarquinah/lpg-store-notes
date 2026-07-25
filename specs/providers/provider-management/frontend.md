---
project: lpg-store
domain: specs
type: spec-track
spec: provider-management
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-07-09
---

# Provider Management — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]] · UI pattern mirrored from the `customers` module

## Technical Notes

**New `providers` module** (`src/modules/providers/`, per the frontend module
template) — the supply-side mirror of the existing `customers` module. Reuse its
structure (store + service + views + components) and the Piloto design system
(petrol+flame tokens, `formatMoney`, shared `PageHeader`/`Spinner`/`EmptyState`,
`ResponsiveTable` for any table, ≥44px touch targets, 14px type floor).

**Surfaces:**

- **Provider list** — search box (name/phone, debounced ≥2 chars like the orders
  customer search), rows with **debt badges**: money outstanding + empties owed
  (mirror the customer list's debt badges). Show-inactive toggle. "Nuevo
  proveedor" → create dialog.
- **Provider create/edit form** — name/phone/notes/active (mirror
  `StoreFormDialog`/the customers form): dual create/edit dialog, prefilled on
  edit, activate/deactivate switch.
- **Provider detail** — debt summary card (money outstanding via `formatMoney` +
  empties owed per tank type), a **price list** editor (per-product unit price),
  and a **record-payment** action (partial amounts allowed) that calls
  `POST /providers/:id/payments` and refreshes the balance. Clean zero-state when
  no debt.

**Purchase dialog changes** (`inventory` module — the existing
`PurchaseDialog.vue`, opened from the Resumen/StoreDetail supply-purchase
button):
- Add a **required provider selector** (list from `GET /providers`), sent as
  `providerId`.
- **Pre-fill the per-line `unitCost`** from the selected provider's price (falls
  back to the existing last-cost/catalog default when the provider has no price
  for that product). The cost field remains the required, confirmable field from
  `provider-purchase-cost`.
- When `emptyReturned < qty`, show a clear inline note:
  **"Quedaremos debiendo N vacíos a este proveedor"** (distinct from the
  monetary surcharge), so the operator sees the empty-tank debt being created.
- Optional **"pagado ahora" (`amountPaid`)** field if the inline-payment Open
  Question resolves that way; otherwise money is settled from the provider
  detail.

**Accounting registry detail** (`accounting` module): the Egresos card now reads
**cash actually paid to providers**; add **Compras recibidas** (goods received)
and **Deuda a proveedores** (balance / period delta) reconciling lines with the
explicit `pagos + Δdeuda = compras` check. Closed registries render the frozen
snapshot figures. This deliberately updates the done `accounting-registry` UI.

**Untouched:** the orders `AssignDialog` "Comprar y cargar" path sends no
`providerId` (backend nullable) and is unaffected — matches today.

**Routing/nav:** add a `/proveedores` route + nav entry (mirror `/clientes`),
role-scoped like the customers surface.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)
- `src/modules/customers/**` — the entire registry + debt-visibility module to mirror as `src/modules/providers/**` (list, form dialog, detail, store, service)
- `src/modules/inventory/components/PurchaseDialog.vue` — add provider selector + price pre-fill + empty-debt note (+ optional `amountPaid`)
- `src/modules/inventory/views/{StoreStockView,StoreDetailView}.vue` — the supply-purchase button context that opens `PurchaseDialog`
- `src/components/app/ResponsiveTable.vue`, design-system tokens, `formatMoney` — shared UI primitives
- `src/modules/accounting/**` — registry detail: cash-paid egress + **Compras recibidas** / **Deuda a proveedores** reconciling lines (frozen when closed)
- app router + nav — add `/proveedores` (mirror `/clientes`)

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [lpg-frontend-vue] description of what was done -->

[2026-07-09] [lpg-frontend-vue] Frontend track **done**. Built the `providers` UI module and reworked the purchase + accounting-registry surfaces.

**New `providers` module** (`src/modules/providers/{types,service,store,routes,index}.ts` + `views/{ProvidersListView,ProviderDetailView}.vue` + `components/{ProviderFormDialog,ProviderPaymentDialog,ProviderPriceDialog}.vue`), a faithful mirror of `customers`: list with debounced name/phone search + money & empties debt badges + show-inactive toggle; create/edit form (name/phone/notes/active, 409-on-name routed to the name field); detail with debt summary (money outstanding + empties owed per tank type), a **price-list editor** (admin-only; `PUT /providers/:id/prices` by `productKind`+`productId`+`unitPrice`), and a **record-payment** action (partial amounts; store selector — admin=all stores, operator=own stores, sole-store default; `POST /providers/:id/payments`). Clean zero-state hides the debt cards when the provider is at zero. Store consumes an injected service (never the ApiClient); added a `put` verb to `src/lib/apiClient.ts`.

**Purchase dialog** (`inventory/components/PurchaseDialog.vue`): required **provider selector** → request-level `providerId`; per-line `unitCost` pre-fills from the selected provider's price (falls back to catalog `purchasePrice`); tank lines show **"Quedaremos debiendo N vacíos a este proveedor"** on a shortfall; optional inline **"pagar ahora"** (`amountPaid` + `paymentMethod`). The `emptyReturned > qty` guard is **relaxed when a provider is selected** (over-return settles empty-tank debt) and kept otherwise; the physical empties-on-hand guard always stays. `inventory/types.ts` purchase payloads gained `providerId?`/`amountPaid?`/`paymentMethod?`.

**Accounting registry detail** (`accounting/{types.ts,views/AccountingDetailView.vue}`): the Egresos card now shows **Compras recibidas (A)** (goods received, memo/reference), **Pagado a proveedores (B)**, **Δ Deuda a proveedores (C)** with an explicit **B + C = A** check line, then Manuales and **Total egresos** (= providerPayments + manual). The egress drill-down "Suma del detalle" reconciles to `goodsReceived`. Frozen snapshot renders unchanged when closed.

**Cross-repo naming cleanup (decision this session):** the accounting egress keys were the only Spanish-named ones in an otherwise-English breakdown, so we renamed them in **both repos** — `comprasRecibidas → goodsReceived`, `deudaDelta → debtDelta` (Spanish stays only in the UI labels). Backend note appended to [[backend]]; backend quality gate re-run green (168 tests).

**Wiring:** `/proveedores` route + nav (mirror `/clientes`, roles operator/admin/developer), `createProvidersModule` in `main.ts`, `providersRoutes` in the router.

**Validation:** a read-only validation agent confirmed all 7 frontend acceptance criteria MET; fixed the two items it flagged (price-table composite row-key to avoid tank/item id collisions; hide debt cards on the zero-state). **Quality gate green:** frontend `npm run typecheck` + `npm run build`; backend `typecheck` + `check` + `test` (168 pass). All acceptance criteria for this repo met.

[2026-07-09] [lpg-frontend-vue] Purchase-dialog UX refinements (post-`done`, operator feedback). **Auto-fill "Vacíos devueltos"**: the field now populates to `min(qty, empties on hand)` on product/qty change (and once shop balances load), replacing the implicit "blank = default" — so the returned count is explicit and the empty-debt note shows the true shortfall (`purchased − available`), e.g. buy 3 with 1 on hand → field fills `1`, note "Quedaremos debiendo 2 vacíos". **Error placement**: the submit/validation error `Alert` moved from the modal header to directly above the action buttons (the dialog scrolls; buttons sit at the bottom) so it's visible the moment the operator submits. **Double-render fix**: the dialog no longer falls back to the shared `store.error` (which the page behind the modal also renders); a server-side failure now hoists that message into the dialog-local `formError` and clears `store.error`, so it renders once, next to the buttons. Counting logic unchanged. typecheck + build green.

[2026-07-09] [lpg-frontend-vue] Purchase-dialog round 2 (operator feedback). **Bug — provider list & empties never loaded from the Resumen overview:** `InventoryView` mounts `PurchaseDialog` with `v-if="purchaseStoreId"` and flips `open` true in the same tick, so the component is created with `open` already true and the `watch(open)` never saw a false→true transition — no `fetchProviders`, no `loadShopEmpties` (hence `Vacíos devueltos` stuck at the `0` placeholder and the provider `<Select>` empty unless the Proveedores page had been visited first). Fixed by making the open-watcher `{ immediate: true }`, which covers both the v-if-open-true mount and the always-mounted (`StoreDetailView`) case. **Pagar ahora defaults ON with the amount pre-filled to the purchase total** (`purchaseTotal` computed = Σ unitCost×qty + Σ surcharge); a `paymentTouched` flag stops the auto-total from clobbering a manual edit; the admin lowers it for a partial/credit purchase or turns it off. **Collapsible product lines:** each line has a chevron header (collapsed → one-line `name × qty · lineCost` summary); adding a line auto-collapses the others. **Sticky header + action bar:** the dialog title and the footer (error + buttons) are pinned while the middle scrolls, so the submit error is always visible next to the buttons. typecheck + build green. (Visual QA of the sticky layout left to the operator.)

[2026-07-09] [lpg-frontend-vue] Purchase-dialog layout fix (round 3). The round-2 sticky header/footer (position:sticky + negative margins) rendered wrong — the whole `DialogContent` still scrolled and the header/footer bled over content. Replaced with a proper flex layout for this dialog: `DialogContent` overridden to `flex flex-col overflow-y-hidden p-0` (so it no longer scrolls itself), a single child `<form class="flex min-h-0 flex-1 flex-col">`, a `shrink-0` `DialogHeader`, a `flex-1 min-h-0 overflow-y-auto` body (the only scroll region), and a `shrink-0` footer bar. Title + action buttons now stay fixed while only the body scrolls. Self-contained (no change to the shared dialog primitives). typecheck + build green. NOTE: could be promoted to a reusable `DialogBody`/scrollable-dialog primitive if other large forms need the same — deferred pending a request.

[2026-07-09] [lpg-frontend-vue] Generalized the scrollable-dialog layout into reusable primitives (operator feedback) + purchase-dialog polish. **Reusable modal layout:** added a `scrollable` prop to `components/ui/dialog/DialogContent.vue` (opt-in — switches the panel to `overflow-y-hidden p-0 gap-0` fixed-header/scroll-body/fixed-footer mode; default unchanged, so no other dialog is affected) and a new `DialogBody.vue` — the `flex-1 min-h-0 overflow-y-auto` scroll region using the themed `.scrollbar-thin` utility so the scrollbar matches the rest of the app in both themes. Usage: `<DialogContent scrollable><form class="flex min-h-0 flex-1 flex-col"><DialogHeader class="shrink-0 …"/><DialogBody/><footer class="shrink-0 …"/></form></DialogContent>`. `PurchaseDialog` now consumes these (replacing its bespoke flex override). Other large dialogs (order wizard, open-day) can adopt the same primitives. **Polish:** input helper captions dropped from `text-xs` (14 px floor) to `text-[13px]` so they read as secondary; the payment row switched `items-end → items-start` so `Método` aligns with the `Monto pagado` input instead of being pushed down by that field's helper line. typecheck + build green.

[2026-07-09] [lpg-frontend-vue] Purchase dialog: clearer required-provider cue. The provider `<Select>` (required in the UI per spec) now shows a `*` on the label, an amber `border-warning ring-warning` outline while unselected, and an amber prompt ("Elija el proveedor de esta compra…") that swaps to the neutral helper once picked — so the operator sees it must be filled. (There is still no backend endpoint to attribute a provider to an already-created provider-less purchase, so "pick later" isn't supported yet — flagged for a possible follow-up.)

[2026-07-09] [lpg-frontend-vue] **Store `Compras` tab reworked into per-purchase cards** (pairs with the backend `purchase_id` enrichment above). `/inventario/tiendas/:id` → Compras now groups lines by `purchaseId` (legacy/provider-less lines each form a single-line group): one card per purchase showing the **provider name**, date, and **money badges** (Pagada / "Debe S/X") + an **empties-owed** badge, the product lines (qty × unit = total, surcharge, and per-line "debemos N vacíos" / "saldó N vacíos"), and a footer with **Total · Pagado · Debe**. Per-line "Editar costo" (unit cost) and the frozen/`Congelado` state are preserved. Types: `PurchaseLineView` gained `purchaseId/providerId/providerName/emptyDebt`; the service returns `{ purchases, purchasePayments }` and the store exposes `purchasePayments` (keyed by `purchaseId`). The provider-level totals on `/proveedores/:id` are unchanged (that view stays a general balance). typecheck + build green.

[2026-07-09] [lpg-frontend-vue] Compras/date follow-ups. **(1) Date format:** `isoToDisplay` (+ the instant date formatters) now render **`dd-Mon-yyyy`** (e.g. `09-Jul-2026`, capitalized Spanish month abbrev) instead of `dd/MM/yyyy`; wrapped the remaining raw-ISO renders (`StoreDetailView` Compras, `InventoryView` stale-day rows, `OpenDayDialog` pending days) in `isoToDisplay` — the app never shows raw `yyyy-MM-dd` now. **(2) Legacy purchases** (no `purchaseId`, pre-enrichment): per-purchase paid/owed can't be attributed, so the card no longer shows a (wrong) "Debe S/total" — it shows the total plus a **"Ver saldo del proveedor →"** link to `/proveedores/:id` (where the money is correctly netted). New purchases keep exact paid/owed. **(3) Edit total + credit:** the cost-correction dialog now edits **either the per-unit cost or the line total** (two-way; a typed total ÷ qty rounds to a 2-decimal unit, with the effective total previewed). If lowering the cost drops the purchase below what was paid for it, a **confirmation** warns it leaves `S/X a favor` with the provider before saving (the provider balance = Σ value − Σ payments naturally reflects the credit). typecheck + build green.
