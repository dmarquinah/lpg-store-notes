---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - ui-design/design-system
last-updated: 2026-06-17
---

# Spec: Mobile Layout Audit & Responsive Fix

## Problem Statement

The [[../design-system/index|design-system]] pass restyled **colors and typography**
but never audited **responsive layout**. The home screens (`DeliveryHome`,
`OperatorHome`, `AdminHome`) and the `DeliveryListView` queue were built as card
lists, so they render correctly on a phone — but most other pages were not, and
they overflow or cram on a ~390px viewport.

An audit of all 19 views found **14 affected**. Three recurring anti-patterns and
one chrome bug:

1. **Raw `<table>` with no mobile fallback (17 tables, 0 fallbacks).** The shared
   `components/ui/table/Table.vue` only wraps content in a `overflow-auto` div — an
   *undiscoverable* horizontal scroll, with columns clipped before the user knows to
   scroll. This is the reported "Productos" bug: the `Vacíos` column is clipped and
   an inner scrollbar appears inside the card. Worst offenders: `OrderDetailView`
   (2 tables), `AssignmentDetailView` (4, incl. a 6-col ledger), `InventoryView`
   (4, incl. a 7-col purchases table), `CatalogView` (2× 5-col), `OrdersListView`,
   `AccountingListView`, `CustomersListView`, `OrganizationView`.
2. **Tab strips with no wrap/scroll.** `DeliveryListView`'s 4 tabs (~500px) clip the
   last tab "Todas" (reported). `InventoryView`'s 3 tabs are borderline and clip
   when the "Por consolidar" badge widens.
3. **Fixed-width filter controls on flex bars.** `w-48`/`w-56`/`w-64`/`w-44` controls
   on `flex flex-wrap` bars cascade-wrap into many rows on a phone, pushing content
   far down: `OrdersListView`, `AccountingListView`, `CustomersListView`,
   `InventoryView`.
4. **AppLayout top bar crams.** The right cluster (user name + role `Badge` + theme
   toggle + "Cerrar sesión" with label) exceeds the available width on ~390px,
   squeezing the left brand and wrapping awkwardly (reported).

Why it matters: the audience is **middle-aged operators/drivers with little web-app
experience** (the design-system audience constraint). A clipped column or an
off-screen tab is, for them, missing data — not "scroll to find it." The app is a
PWA used on phones in the field, so phone-first layout is the primary case, not an
afterthought.

## Proposed Solution

A single, prioritized responsive sweep over every affected view — **no behavior,
data, or color changes**, layout only. The dominant fix is one new shared primitive
so the 17 tables are fixed once, not per view.

1. **`ResponsiveTable` primitive** (`src/components/app/`): renders a normal
   `<table>` at `sm+` (≥640px) and **stacks each row as a label/value card below
   640px** (each cell becomes a `Label: value` line). Views declare column headers
   once; the component reuses them as the per-cell labels on mobile. Replaces the bare
   `Table.vue` usage in data views. This becomes the canonical table pattern, codified
   as a new rule in `eng/design-system.md`.
2. **Scrollable/wrapping tab strips**: make `TabsList` not clip — horizontal scroll
   with an edge affordance (or wrap) on narrow screens — so `DeliveryListView` and
   `InventoryView` never hide a tab.
3. **Responsive filter bars**: form controls go full-width on mobile, fixed width at
   `md+` (`w-full md:w-56` idiom) so a filter bar stacks cleanly instead of
   cascade-wrapping.
4. **AppLayout header fix**: collapse the right cluster on narrow screens — keep the
   theme toggle + an icon-only logout (with `aria-label`), move the user name + role
   into the existing menu drawer (or hide the name and keep the role badge), so the
   bar never crams.
5. **Audit pass**: every view eyeballed at 390px in both themes; remaining low-severity
   cramping (tight `justify-between` rows, long unbroken strings) fixed with `min-w-0`
   / `flex-wrap` / `break-words` / `truncate` as needed.

Work is ordered by severity (see Implementation Plan in the track file): **P0** chrome
(AppLayout, tab strips) → **P1** the `ResponsiveTable` primitive + the highest-traffic
detail/list tables → **P2** remaining tables + filter bars → **P3** low-severity polish.

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

- [x] **No horizontal page overflow** on any view at 390px width — the document does
      not scroll sideways; nothing is clipped off-screen.
- [x] A shared **`ResponsiveTable`** component exists in `src/components/app/`, renders
      a standard table at `sm+` and a stacked label/value card list below `sm`, and
      preserves the existing table states (loading `Spinner`, empty `TableEmpty`/empty
      message) and row-link affordances.
- [x] **Every data table** is migrated to `ResponsiveTable` (or an equivalent
      phone-first card list): `OrderDetailView` (Productos + Movimientos),
      `AssignmentDetailView` (tanks/items/discrepancies/ledger), `InventoryView`
      (assignments/consolidation/stock/vehicles/purchases), `CatalogView`
      (tank-types/items), `OrdersListView`, `AccountingListView`, `CustomersListView`,
      `OrganizationView` (users), `CustomerDetailView`, `DriverDayView`. No data view
      relies on undiscoverable horizontal table scroll.
- [x] **No table column is clipped** at 390px — specifically the reported
      `OrderDetailView` Productos `Vacíos` column is fully visible.
- [x] **Tab strips never clip a tab** at 390px: `DeliveryListView` (4 tabs incl.
      "Todas") and `InventoryView` (3 tabs incl. the badge) are fully reachable
      (scroll/wrap), and the active tab is always visible.
- [x] **Filter/toolbar controls** in `OrdersListView`, `AccountingListView`,
      `CustomersListView`, and `InventoryView` are full-width on mobile and do not
      cause a multi-row cascade that buries the content; controls stay ≥44px tall.
- [x] **AppLayout top bar** does not cram or wrap on 390px: brand stays on the left,
      the right cluster collapses gracefully (icon-only logout with `aria-label`,
      name/role relocated or hidden), all controls remain ≥44px touch targets.
- [x] **Both themes** (light + dark) verified at 390px for every changed view.
- [x] **No regressions on desktop** (≥1024px): tables render as tables, layouts match
      pre-change appearance.
- [x] **Design-system doc updated**: `eng/design-system.md` documents `ResponsiveTable`
      as the canonical data-table pattern and the responsive tab/filter/header rules,
      with the change made in this spec (per the design-system "break a rule → update
      the doc" rule).
- [x] **No behavior, data, color-token, or copy changes** — layout/markup only; gates
      green (`npm run typecheck` + `npm run build`).

## Tracks

<!-- Frontend-only spec (Piloto). Mirrors design-system / stale-day-banner which also
     ship a single frontend track under the folder layout. -->

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **No new endpoints, data shape, or business-logic changes** — purely presentational.
- **No color/token/typography changes** — that's the [[../design-system/index|design-system]]
  spec's domain; this spec only touches layout/structure.
- **No new UI framework or component library** — shadcn-vue + Tailwind + lucide only
  (design-system hard rule #10).
- **Login / AcceptInvite** views — already single-column and phone-fine; only touched
  if the audit finds a concrete 390px overflow.
- **Charts/maps** (Chart.js / Leaflet) responsive behavior — not part of this audit
  unless a container overflows.
- **Tablet-specific (640–1024px) tuning** beyond the existing `sm:`/`md:`/`lg:` steps
  — the target is phone (≤640px) correctness without desktop regressions.

## Open Questions

- `ResponsiveTable` API shape: column-config prop (`[{key,label,align,cell}]`) vs.
  slot-driven with a parallel mobile-label declaration. The track file proposes a
  pragmatic default; settle during implementation against the first 2–3 migrations.
- For wide list tables (`OrdersListView` actions column, `OrganizationView` users),
  decide per-table whether the mobile card shows all columns or a curated subset +
  a "Ver" link to detail. Lean toward showing all to avoid hiding data from the
  low-tech audience.
