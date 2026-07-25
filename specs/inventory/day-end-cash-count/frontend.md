---
project: lpg-store
domain: specs
type: spec-track
spec: day-end-cash-count
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: '"2026-07-10"'
---

# Day-End Cash Count — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend track: [[backend]] · Day-end flow: [[../day-handoff/index]]

## Technical Notes

Extends the existing `inventory` module + the driver day surface built in [[../day-handoff/index|day-handoff]] — no new module. Ports right after its backend counterpart is `done` (needs the day-summary read + the extended close payload). Follow `eng/design-system.md` + [[../../ui-design/mobile-layout-audit/index]] (≥44px, no overflow, `formatMoney`, status→badge map). This is the **driver on a phone in the field**, so the count flow must be fast and legible.

### Driver — end-of-day recap + cash attestation

On `DriverDayView` (`/mi-dia`) / the driver's assignment surface:

- **Recap panel** driven by the backend day-summary: **balones entregados** per type, **money collected by payment method** (`formatMoney`, incl. **Vale FISE**), and **debt accrued today** (money + empty tanks) shown as **totals with an expandable per-customer/order detail** so the driver can verify how each total was reached. This is the "what you're handing in" view the driver never had.
- Extend the existing `CountDialog` (or a sibling step) so the close captures **counted cash per method** with the **expected** shown alongside (from the summary). Methods with activity (expected > 0) **plus Efectivo always** are editable; methods with **no movement that day are still shown but muted/greyed** (transparency — the driver sees they were 0, not hidden). Keep the mandatory tank `finalCounts` in the same close.
- When counted ≠ expected for any method, **require a note** before submit (mirror the backend 400 exactly, as the `ConflictError`/`messageFrom` mapping already does for other guards). Show the per-method delta live so the driver sees what they're explaining.
- Submit → the extended `closeAssignment` call (tanks + cash + note). On success the day shows `closed` / "Contado — esperando verificación".

### Driver — end-of-day button gating

- The end-of-day action is **always rendered** but **de-emphasized** (muted/secondary variant + hint text "disponible al terminar tus entregas") until the driver has **≥1 delivered order today**; once there's a delivery it becomes the **prominent primary** action.
- Derive "has delivered today" from the driver's already-fetched order/assignment data (a delivered order bound to today's assignment) — no new endpoint needed if the delivery count is already reachable; otherwise reuse the driver order queue the day view already loads.
- **No hard block** — an empty-day close is still possible (driver took the truck, sold nothing).

### Operator — Consolidar/verify recap

- The operator's **"Por consolidar"** review (`InventoryView` tab + `AssignmentDetailView`) shows the **same money/debt recap** plus the driver's **counted-vs-expected** and any **discrepancy note**, so the `carry`/Consolidar confirmation covers money, not just tanks.
- Surface a **discrepancy badge/filter** so the operator/admin can find the discrepant days (drives the "admin is aware of these scenarios" requirement).

### Role gating

- `delivery` sees the recap + cash-count + close, never carry (unchanged from day-handoff).
- `operator`/`admin` see the recap + counted-vs-expected + notes on the verify surface + the discrepancy filter.
- Mirror backend guards client-side so the UI never offers an action the API will 403/400.

### Coordination

- [[../inventory-ux-pass/index]] also edits `DriverDayView`/`AssignmentDetailView` (day-state wording). Coordinate ordering at `/focus` to avoid churn on the same files; this spec adds the money surface, that spec renames the states.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify (confirm exact paths when the track starts):

- `src/modules/inventory/views/DriverDayView.vue` — recap panel, cash-count step, button gating.
- `src/modules/inventory/components/CountDialog.vue` — extend with counted-cash-per-method + discrepancy note.
- `src/modules/inventory/views/InventoryView.vue` — "Por consolidar" tab: discrepancy badge/filter.
- `src/modules/inventory/views/AssignmentDetailView.vue` — operator verify recap (money + counted-vs-expected + note).
- The inventory store/service layer — new call: day-summary read; extend the close call with `cashCounts`/`cashNote`; new response types.
- `eng/design-system.md` conventions — `formatMoney`, badges, phone-first.

Reference: [[frontend|day-handoff frontend track]] for the driver `/mi-dia` + `CountDialog` + operator "Por consolidar" surfaces this extends, and the client-side guard-mirroring pattern.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-10] [lpg-frontend-vue] Frontend track **done**. Layered the money attestation onto the existing `inventory` module + driver/operator surfaces (no new module). Gates green: `npm run typecheck` + `npm run build` (PWA precache 80 entries / 894.67 KiB). Independent validation confirmed all six frontend acceptance criteria against the diff; owner manual smoke of the attest → discrepancy-note → verify flow left per the project's standing verification method.

- **`vale FISE` tender.** Added `fise` + "Vale FISE" to the orders module (`types.ts` `PAYMENT_METHODS`, `components/orderLabels.ts`), so it auto-surfaces in every method selector/label (`PaymentDialog`, `DeliverDialog`, `OrderCreateView` delivery payment; `OrderDetailView`, `OrderTimelineNode`). Also added it to the **accounting** module (`types.ts` + `accountingLabels.ts`) so the registry's by-method grouping renders the label instead of a blank once a `fise` payment lands there (the spec's "FISE shows up there once it's a method"). Inventory keeps its own `CASH_METHODS` + `CASH_METHOD_LABELS`.
- **Contract mirror.** `inventory/types.ts` gained `DaySummary` (+ `MoneyByMethod`/`MoneyDebtItem`/`EmptyDebtItem`/`CashCountView`) and `DaySummaryResponse`, mirroring the backend exactly (money fields are strings → `formatMoney(Number(...))`). `ClosePayload` extended with `cashCounts` + `cashNote`; `AssignmentFilter` gained `hasCashDiscrepancy`. `service.ts`: `getDaySummary(id)` → `GET /assignments/:id/day-summary` (`res.summary`) + `?hasCashDiscrepancy=true` serialization. `store.ts`: `daySummary` + `fetchDaySummary`, `discrepancyDayIds` + `fetchDiscrepancyDays`; `refreshCurrent` also re-pulls the summary **only when a view already has it loaded** so plain sale/return/load writes don't pay for an unused fetch.
- **Driver close (CountDialog).** Added a "Dinero recaudado" section below the mandatory tank count, seeded from `store.daySummary.cashByMethod`. Editable methods = any with `expected > 0` **plus Efectivo always**; no-movement methods shown muted ("Sin movimiento"), not hidden. Each editable row shows counted input + expected + a live per-method delta. `buildPayload` submits `cashCounts` for **every editable row** (covers all `expected>0` methods, avoiding the backend's union-of-methods false-discrepancy) and **requires `cashNote`** when any counted ≠ expected (mirrors the 400). Money parsed as non-negative ≤2-decimals (mirrors backend `countedMoney`).
- **Driver recap + button gating (DriverDayView).** `load()` now fetches `fetchAssignment` + `fetchDaySummary` together. New "Resumen del día" card: balones entregados per type, cobrado por método (+ total, incl. Vale FISE), and deuda generada hoy (crédito otorgado + balones prestados) as totals with an **expandable per-customer/order** breakdown. End-of-day button is **always visible** but rendered `secondary` + "Disponible al terminar tus entregas." until `hasActivityToday` (derived from the summary: any tank out / cash collected / debt accrued — no extra endpoint), then the prominent primary. No hard block.
- **Operator verify (AssignmentDetailView).** For `closed`/`carried` days, loads the summary and renders a **"Cuadre de caja"** card: a counted-vs-expected-vs-diff-vs-note `ResponsiveTable` (from `cashCounts`), a **Descuadre** badge when `hasCashDiscrepancy`, plus the same money/empty debt breakdown and cobrado-por-método context — so the `carry` second signature covers money.
- **Discoverability (InventoryView).** `fetchDiscrepancyDays()` on mount (`?hasCashDiscrepancy=true`) → the "Por consolidar" rows badge a **Descuadre** flag for any id in `discrepancyDayIds`.
- **Unchanged:** the admin/dev **force-close** `CloseDialog` (omits `cashCounts` → backend override bypass) and all tank `finalCounts`/`carry`/discrepancy behaviour. Design-system compliant (tokens, `formatMoney`, `h-11` controls, `ResponsiveTable`, badge variants, Spanish, phone-first).

[2026-07-10] [lpg-frontend-vue] Owner-feedback follow-up (post-completion): **`/mi-dia` reorganized into tabs.** The recap was stacking below the vehicle stock and crowding it. `DriverDayView` now has an **Inventario** tab (current vehicle stock **+ a new "Movimientos del día" ledger** from `store.transactions` — apertura/carga/venta/devolución with time + Δllenos/Δvacíos, so the driver sees *how* the stock changed, not just the ending level) and a **Resumen** tab (the money/debt recap, unchanged content). The "Marcar día como contado" button + state alerts stay below the tabs, always visible. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Owner-feedback follow-up: **show the store/location on `/mi-dia`.** The assignment header card now displays the store name (with a `MapPin` icon) between the date and "Asignación #N". Resolved client-side from the driver's own store-assignment links — `catalog.fetchStoreAssignments(false, auth.user.id)` (added an optional `userId` scope to that action; the `GET /catalog/store-assignments?userId=` route is unguarded so a `delivery` user reads only their own), matched by `storeAssignmentId`. No backend change. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Reworked the `/mi-dia` location source (owner request — avoid leaning on the catalog store-assignments endpoint). The store name now comes straight from the driver's own **day-summary** (`summary.storeName`), so `DriverDayView` no longer fetches `catalog.storeAssignments` (reverted the earlier `userId`-scoped catalog call + its `fetchStoreAssignments(all, userId?)` param). `DaySummary` type gained `storeId` + `storeName`. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Moved the **"Marcar día como contado"** action (+ the closed/carried status alerts) out from below the tabs into the **Resumen tab, beneath the recap** — so closing the day is a conscious action taken after the driver has seen the summary. The header state badge still conveys the day state on any tab. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Reshaped the Resumen tab around a per-order **"Ventas del día"** list (owner request): one card per delivered order tying its money (Total · Pagó · Debe) to the tanks it moved (qty × type) — replacing the separate aggregate "Balones entregados" + "Dinero recaudado" blocks. Kept a compact **"Cobrado por método"** totals block (cash reconciliation) and the **"Deuda generada hoy"** section, now with the credit shown as a total (per-order detail lives in Ventas) + the expandable "Balones prestados". Mirrors the new `DaySummary.orders` (`DayOrder`). typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Added a **"Totales del día"** two-column panel to the Resumen tab (owner picked this layout): left column **Balones vendidos** (per type + total count, from the already-present `tanksHandedOut`), right column **Cobrado** (by method + subtotal) **+ Crédito**, closed by a full-width bold **Total ventas** bar (`cobrado + crédito`). Replaces the loose "Cobrado por método" + "Crédito otorgado" lines with one organized totals block; "Balones prestados hoy" (empties) kept as a separate expandable line below. Frontend-only (no new backend data). typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Redesigned the "Ventas del día" order card for scannability + payment-method visibility (owner feedback): row 1 = customer · #order (left) with the order **total right-anchored** (scan straight down a long list), row 2 = tanks inline, row 3 = **payment-method chips** (`Efectivo S/ 50.00`) from the new `DayOrder.payments`, plus a red **Debe** chip when there's outstanding credit (`bg-destructive/10` + `text-destructive-text`). Mirrors `DayOrderPayment`. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Resumen recap polish (owner feedback): (1) order cards are now **2 rows** — customer·#order + total on row 1, tanks + payment-method chips + Debe chip together on row 2. (2) **Total ventas** moved into the money column as its bold final line, directly under Total Cobrado / Total Crédito (was a detached full-width footer bar that read far apart on desktop). (3) **Prestados** (empties loaned) folded into the Balones column of the Totales panel (warning color when > 0) instead of a separate detached line below — easy to spot alongside Vendidos. Dropped the empties per-customer expander (number integrated); removed the now-unused `ChevronDown`/`showEmptyDebt`. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Reworked the order card to a **receipt-style two-column** layout (owner feedback: the inline 2-row version wasted right-side space on multi-item orders). Left column = customer·#order + one line per tank type (readable for many types); right column = order total (bold) stacked over each payment method (`Efectivo S/ 150.00`) and a red `Debe` line. Both columns grow with content, so neither side is dead space regardless of item count. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Made the order card a true **itemized receipt** (owner request — per-item money): header = customer·#order + order total; one line per tank type showing `qty × tipo` with its **own line cost** right-aligned (`DayOrder.tanks[].lineTotal`); payment-method chips + Debe as the footer. Each item line's cost fills the right edge, so multi-item orders read cleanly with no dead space. typecheck + build green.

[2026-07-10] [lpg-frontend-vue] Reworked "Ventas del día" into a **responsive grid of receipt "tickets"** (owner request): `grid sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4` (1-up on phones, up to 4-up on wide screens). Each order is a bill-styled `<article class="ticket">` — centered "Pedido #N" + customer header, dashed dividers, itemized `qty × tipo · line cost` rows (narrow ticket keeps name+amount together, fixing the wide-card gap), a bold Total, and payment/Debe lines. Added a scoped `.ticket` CSS mask for a **scalloped/torn bottom edge** (degrades to a plain rounded card if masks unsupported). Pulled the Ventas grid + Totales panel out of the single wrapping `Card` so each ticket is its own paper slip on the page surface. typecheck + build green.
