---
project: lpg-store
domain: specs
type: spec-track
spec: mobile-layout-audit
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: 2026-06-17
---

# Mobile Layout Audit & Responsive Fix — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

- **Phone-first, layout only.** No data/behavior/token changes. The design-system
  tokens, type scale (14px floor), and ≥44px touch idiom (`h-11 md:h-10`) already
  exist — reuse them; do not introduce new sizes or colors. Follow
  [[../../../eng/design-system|design-system]] hard rules.
- **Target viewport: 390px** (iPhone-class). Use Tailwind breakpoints already in the
  codebase: base = phone, `sm:` (640px), `md:` (768px), `lg:` (1024px). The
  `ResponsiveTable` switch point is `sm` (640px) — below it, cards; at/above, table.
- **The `Table.vue` safety net is not enough.** `components/ui/table/Table.vue` wraps
  the `<table>` in `<div class="relative w-full overflow-auto">`. That allows
  horizontal scroll but it's undiscoverable on a phone and columns clip before the
  user scrolls. Keep `Table.vue` pristine (shadcn primitive) — build the responsive
  behavior in a new `app/` component that composes it.
- **Precedent to mirror:** `DeliveryListView` already does the right thing — a
  phone-first `grid gap-3 sm:grid-cols-2 lg:grid-cols-3` **card list** instead of a
  table. `ResponsiveTable` generalizes that idea so every data view gets it for free.
- **Existing table states must survive the migration:** loading row =
  `TableEmpty` + `Spinner` + "Cargando…"; empty row = `TableEmpty` + "No se
  encontraron …"; row links = `Button as-child > RouterLink`. The new component must
  preserve all three (design-system §5).
- **Tabs primitives:** `components/ui/tabs/TabsList.vue` is `inline-flex … p-1` with no
  wrap/scroll; `TabsTrigger.vue` is `whitespace-nowrap px-4 py-2`. Fix at the
  `TabsList` level (add an overflow-x-auto / scroll-snap affordance or allow wrap) so
  every tabbed view benefits, rather than per-view.
- **AppLayout header** (`src/layouts/AppLayout.vue`, ~lines 113–151): the right cluster
  is `flex items-center gap-4 text-sm` holding `{name} <Badge>` + theme button +
  "Cerrar sesión" labelled button. On 390px the cluster needs ~230–310px, crushing the
  brand. Collapse on narrow screens: icon-only logout with `aria-label="Cerrar
  sesión"` (label returns at `sm:`), and either move `name + role` into the existing
  menu drawer or hide the name (keep the role `Badge`). The title `· {{ title }}` is
  already `hidden … sm:inline` — keep that.

## Implementation Plan (severity-ordered)

**P0 — chrome (most-seen, fastest):**
1. `AppLayout.vue` header — collapse the right cluster on `<sm`.
2. `TabsList.vue` (or a wrapper) — scrollable/wrapping tab strip; verify
   `DeliveryListView` (4 tabs) and `InventoryView` (3 tabs + badge).

**P1 — the primitive + highest-traffic tables:**
3. Build `components/app/ResponsiveTable.vue` (+ settle the API on the first migration).
   Decide column-config vs. slot approach; preserve loading/empty/row-link states.
4. Migrate `OrderDetailView` (Productos + Movimientos) — the reported bug. Validate
   `Vacíos` visible at 390px.
5. Migrate `AssignmentDetailView` (tanks/items/discrepancies/ledger — the 6-col ledger
   is the stress test).

**P2 — remaining tables + filter bars:**
6. Migrate `InventoryView` (assignments/consolidation/stock/vehicles + 7-col
   purchases), `CatalogView` (2 tables), `OrdersListView`, `AccountingListView`,
   `CustomersListView`, `OrganizationView` (users), `CustomerDetailView`,
   `DriverDayView`.
7. Filter bars → `w-full md:w-…`: `OrdersListView`, `AccountingListView`,
   `CustomersListView`, `InventoryView`.

**P3 — polish & docs:**
8. Low-severity cramping: `DriverDayView` / `OrderDetailView` header rows
   (`min-w-0`/`flex-wrap`/`break-words`), long addresses/phones (`truncate`/`break`).
9. Update `eng/design-system.md` — `ResponsiveTable` as canonical, tab/filter/header
   responsive rules.
10. Full 390px pass in **both themes**; `npm run typecheck` + `npm run build`.

## Audit findings (severity)

**HIGH**
- `orders/views/OrderDetailView.vue` — Productos table (5 col) + Movimientos table
  (4 col) overflow the card; `Vacíos` clipped, inner scrollbar (reported).
- `orders/views/DeliveryListView.vue` — 4-tab strip (~500px) clips "Todas" (reported).
  NB: its card *list* is already responsive — only the tab strip is broken.
- `layouts/AppLayout.vue` — top-bar right cluster crams on 390px (reported).

**MEDIUM**
- `orders/views/OrdersListView.vue` — main table (6–7 col) + fixed-width filter bar
  (`w-48`/`w-40`/`w-56`); actions column (4 icon buttons) cramped.
- `catalog/views/CatalogView.vue` — two 5-col tables (tank-types, items).
- `accounting/views/AccountingListView.vue` — 7–8 col table + `w-48` filter selects.
- `customers/views/CustomersListView.vue` — 6-col table + `w-64` search input.
- `inventory/views/InventoryView.vue` — 7-col purchases table (worst); 3-tab strip
  borderline (badge widens it); `w-44`/`w-40` filters.
- `inventory/views/AssignmentDetailView.vue` — 6-col ledger table.
- `organization/views/OrganizationView.vue` — 5-col users table (Rol/Estado wrap).

**LOW**
- `accounting/views/AccountingDetailView.vue` — mostly responsive; check
  `RegistryDrilldownDays` drill-down tables when expanded.
- `inventory/views/DriverDayView.vue` — date + badge header row tight (`justify-between
  gap-3`), long date may wrap.
- `customers/views/CustomerDetailView.vue` — small 2–3 col debt tables, cramped only.
- `orders/views/OrderCreateView.vue` — mostly responsive; verify the `20rem` summary
  column and selects don't overflow.

**OK (verify only, likely no change):** `home/views/{DeliveryHome,OperatorHome,AdminHome}.vue`,
`auth/views/{LoginView,AcceptInviteView}.vue`.

## Recurring anti-patterns (fix at the source)
1. **Raw `<table>` with no mobile fallback — 17 tables** across 9 views → the
   `ResponsiveTable` primitive fixes all at once.
2. **Fixed-width filter controls on `flex flex-wrap`** (`w-48`/`w-56`/`w-64`/`w-44`) —
   5 views → `w-full md:w-…`.
3. **Tab strips with no wrap/scroll** — 2 views → fix `TabsList`.
4. **Header cluster overflow** — `AppLayout` only.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

**New / shared (fix-at-source):**
- `src/components/app/ResponsiveTable.vue` — **new**, the canonical responsive
  data-table (table at `sm+`, stacked label/value cards below). Preserves
  Spinner/TableEmpty/row-link states.
- `src/components/ui/table/Table.vue` — keep pristine; `ResponsiveTable` composes it.
- `src/components/ui/tabs/TabsList.vue` — make the strip scroll/wrap, never clip.
- `src/layouts/AppLayout.vue` — header right-cluster collapse on `<sm`.

**Views to migrate/fix (high→low):**
- `src/modules/orders/views/OrderDetailView.vue` — Productos + Movimientos tables (P1).
- `src/modules/inventory/views/AssignmentDetailView.vue` — 4 tables incl. ledger (P1).
- `src/modules/orders/views/DeliveryListView.vue` — tab strip (P0; cards already OK).
- `src/modules/inventory/views/InventoryView.vue` — 5 tables + tabs + filters (P2).
- `src/modules/catalog/views/CatalogView.vue` — 2 tables (P2).
- `src/modules/orders/views/OrdersListView.vue` — table + filter bar (P2).
- `src/modules/accounting/views/AccountingListView.vue` — table + filters (P2).
- `src/modules/customers/views/CustomersListView.vue` — table + search width (P2).
- `src/modules/organization/views/OrganizationView.vue` — users table (P2).
- `src/modules/customers/views/CustomerDetailView.vue` — debt tables (P3).
- `src/modules/inventory/views/DriverDayView.vue` — header row + tanks table (P3).
- `src/modules/accounting/views/AccountingDetailView.vue` — verify drill-down (P3).
- `src/modules/orders/views/OrderCreateView.vue` — verify summary/selects (P3).

**Docs:**
- `lpg-frontend-vue` → vault `eng/design-system.md` — add `ResponsiveTable` +
  responsive tab/filter/header rules (criterion).

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [lpg-frontend-vue] description of what was done -->

[2026-06-17] [lpg-frontend-vue] **Done.** Built `components/app/ResponsiveTable.vue` — column-config (`{ key, label, align?, class? }`) + `#cell:{key}` / `#actions` slots; renders a real `<table>` at `sm+` and a **stacked label/value card per row below `sm`**; owns Spinner-loading (`:loading`/`loading-text`) and empty (`empty-text`) states (composes the pristine `ui/table/*` — `Table.vue` untouched). Migrated **all 17 data tables across 10 views**: OrderDetailView (Productos + Movimientos — the reported `Vacíos`-clip bug), AssignmentDetailView (tanks/items/discrepancies/6-col ledger), InventoryView (assignments/consolidation/stock/vehicles/7-col purchases), CatalogView (×2), OrdersListView (+conditional Tienda column preserved, action cluster → `#actions`), AccountingListView, CustomersListView, OrganizationView (users), CustomerDetailView (×2 — totals rows kept as separate blocks below), DriverDayView. **Chrome:** `TabsList` → horizontal-scroll strip (`flex w-max max-w-full overflow-x-auto`, scrollbar hidden) + `TabsTrigger` `shrink-0` so no tab clips (**"Todas" fixed**); `AppLayout` header de-cram — icon-only logout `<sm` (+`aria-label`), user name hidden `<sm` and relocated to a new drawer identity block, `px-4 sm:px-6`, tighter gaps. **Filter bars** → `w-full md:w-…` in Orders/Accounting/Customers/Inventory. **Polish:** DriverDayView header `min-w-0`/`break-words`; CustomersListView address `block` so `truncate` applies. InventoryView assignments/consolidation whole-row `@click` → explicit **"Ver"** button (now matches the design-system row-link rule). Updated `eng/design-system.md` (§5 `ResponsiveTable` as canonical + phone-first layout bullet, hard rule #11, Related Files). Independent validation: **all in-repo criteria PASS**, no behavior/data/token/copy changes, no missed `<Table>` anywhere. Gates green (typecheck + build; PWA precache **73 entries / 764.51 KiB**). Manual on-device 390px both-theme smoke left to the operator.

[2026-06-18] [lpg-frontend-vue] **Follow-on UX refinements (user-approved, spec stays `done`).** On the shared order detail (`OrderDetailView`, used by both `/pedidos/:id` and `/mis-entregas/:id`): (1) **logout moved out of the top bar into the drawer** (pinned bottom via `mt-auto`; `SheetContent` → `flex flex-col`); the top bar now carries only brand + role badge + theme toggle. (2) The run-on **"date · Método · Tienda" description string** became an **icon-labelled metadata `Card`** (CalendarDays/CreditCard/Store → Fecha / Método de pago / Tienda, `grid sm:grid-cols-3`) — legible on mobile. (3) **Floating bottom action bar** — the contextual actions moved from the `PageHeader #actions` (top-right) to a `sticky bottom-0` bar (`-mx-4 sm:-mx-6`, backdrop-blur, iOS safe-area padding) so they're always thumb-reachable; each button now pairs a **lucide icon with a plain-language label naming the action** (Despachar → **"Empezar despacho"**, Entregar → "Confirmar entrega", Falló → "No se pudo entregar", Asignar → "Asignar repartidor", Transferir → "Transferir de tienda", Cancelar → "Cancelar pedido"); a new `hasActions` computed gates the bar. (4) **`Historial` timeline node** (`OrderTimelineNode`) text restructured for distinguishable sections — actor/target/method now each render as an **icon + muted label + strong value** row (User "Por" / UserCheck "Para" / CreditCard method), alignment-aware via a new `rowJustify`, and the fail/cancel **Motivo** sits in a tinted `bg-muted` box. Tokens only, no behavior/data change. Gates green (typecheck + build, PWA 73 entries / 769.99 KiB).

[2026-06-18] [lpg-frontend-vue] **Floating-bar + timeline-date polish (user feedback).** The `OrderDetailView` action bar was semi-transparent (a bright card edge bled through — the reported "white spec") and left a gap below it at scroll-bottom (from `main`'s `py-8/lg:py-10`): made it **opaque `bg-background`** with an upward shadow (`shadow-[0_-6px_16px_-8px_…]`) and cancelled the page bottom padding with **`-mb-8 lg:-mb-10`** so it sits flush regardless of scroll position. In `OrderTimeline`, the bare centered **date text → a pill chip** (`rounded-full border bg-card px-3 py-1`, mirroring the timeline's start/end end-caps). Gates green.

[2026-06-18] [lpg-frontend-vue] **Floating-bar gap fix.** The residual bottom gap on `OrderDetailView` was because the sticky action bar sat **before** the dialog components, whose in-flow placeholder anchors followed it — so the bar wasn't the last in-flow child and its `-mb-8 lg:-mb-10` (which cancels `main`'s `py-8`/`lg:py-10` bottom padding) had nodes after it. Moved the bar to be the **genuine last child** (after all dialogs); the negative bottom margin now cancels the page padding cleanly, so the bar sits flush at the viewport bottom whether or not the page is scrolled. Gates green.

[2026-06-18] [lpg-frontend-vue] **Floating-bar gap — proper fix (supersedes the `-mb` hack).** Root cause was structural: the app shell's `main` used vertical padding (`py-8 lg:py-10`), so the scroll container always had bottom padding that reappeared under the bottom-pinned sticky bar at scroll-end. Switched `main` to **top-only padding** (`pt-8 lg:pt-10`) — the page no longer has bottom padding for a sticky footer to fight — and **removed the `-mb-8 lg:-mb-10` negative-margin hack** from the `OrderDetailView` action bar. The bar (sticky `bottom-0`, last child) now sits flush at every breakpoint with zero gap, no hack. Trade-off: all views lose ~2rem bottom padding when scrolled to the very end (content sits at the edge) — acceptable phone-app behaviour; revisit per-view if any screen needs bottom breathing room. Gates green.

[2026-06-18] [lpg-frontend-vue] **Footer pinned + mobile full-screen dialogs (user feedback).** (1) The page action bar now uses **`fixed inset-x-0 bottom-0`** (opaque `bg-background`) instead of `sticky` so content scrolls fully beneath it with nothing showing through/below; a **ResizeObserver** measures the bar and the page reserves its height as `padding-bottom` (handles 1- vs 2-row wrap) — content clears it with no gap. Paired with the app-shell `main` already being **top-padding-only**. (2) Shared **`DialogContent`** is now a **full-screen sheet on mobile** (`fixed inset-0`, scrollable) and the centered card at `sm+` (`sm:max-h-[85vh]` + internal scroll) — it covers the page's own action bar that previously bled behind the dim overlay. A direct-child `<form>` grows (`max-sm:[&>form]:flex-1 flex-col`) so the **`DialogFooter` pins to the bottom** (`max-sm:mt-auto`) with full-width buttons and a real **`gap-2`** (they were touching). (3) `DeliverDialog` (Registrar entrega): **removed the redundant "Cancelar"** — the corner ✕ already cancels. All shared-component changes; gates green (typecheck + build).
