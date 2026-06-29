---
project: lpg-store
domain: specs
type: spec-track
spec: registry-source-drilldown
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-16
---

# Registry Source Drill-down — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

Additive change to the `accounting` module's **registry detail** view. The
Ingresos and Egresos cards already render the aggregate blocks (operational by
method + manual; provider tanks/items/recargos + manual). Make each block
**expandable** to reveal the backend's day-grouped drill-down.

- **Expandable blocks:** under each block total, an expander reveals per-day
  rows (date + day subtotal), each expandable (or always-shown) to the day's
  records:
  - *Ingreso:* the day's payments — order ref, customer, method, amount.
  - *Egreso:* the day's purchases — tank-type/item, qty, unit cost, line cost,
    plus any surcharge.
  Keep the existing headline totals exactly as today; the drill-down sits
  beneath them. The day subtotals must visibly add up to the block total.
- **Data source:** consume the extended `RegistryDetailView.drilldown` (or the
  `GET /registries/:id/lines` sub-resource — match whatever the backend track
  ships; lazy-load if it's a sub-resource). For a **closed** registry the same
  view renders the frozen snapshot detail — no separate code path, no live
  re-query.
- **Design system:** `formatMoney` for every figure; petrol+flame tokens only
  (no raw palette classes — `eng/design-system.md` hard rules). Reuse the
  existing table/empty/spinner chrome (`TableEmpty`, `Spinner`) for the
  expanded lists. Phone-first: the operator audience is low-tech-literacy, so
  the expanded view must stay legible on a narrow screen (consider a stacked
  card list per day rather than a wide table on mobile, matching the
  DeliveryListView pattern).
- **Read-only for closed:** no add/remove or edit affordances inside the
  drill-down regardless of status; it's purely verification.

## Related Files

> Confirm exact filenames at `/focus` time — the accounting detail view + its
> Ingresos/Egresos cards are known by role from the `accounting-registry`
> frontend track, not re-read in this planning pass.

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/accounting/` — the **registry detail** view (Ingresos / Egresos
  cards): add the expandable day-grouped drill-down under each block.
- `src/modules/accounting/` service/store + `types.ts` — mirror the backend
  `drilldown` shape (or the `/lines` sub-resource fetch + types).
- `src/lib/format` (`formatMoney`) and shared `components/app/*` chrome
  (`TableEmpty`, `Spinner`, `EmptyState`) — reused for the expanded lists.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-frontend-vue] Frontend track **done**. Read-only per-block, day-grouped drill-down on the registry detail, consuming the backend's `GET /accounting/registries/:id/lines` sub-resource.

**Types** (`types.ts`): mirrored the backend `RegistryDrilldown` = `{ ingress: IngressDay[], egress: EgressDay[] }` (`IngressLine`/`IngressDay`, `EgressLine`/`EgressDay`). No status field — the endpoint serves the frozen snapshot for closed registries, so the UI has **one code path**.

**Service** (`service.ts`): `getRegistryLines(id)` → `apiClient.get<RegistryDrilldown>('/accounting/registries/:id/lines')` (unwrapped).

**Store** (`store.ts`): added `drilldown` / `drilldownFor` / `drilldownLoading` / `drilldownError` + `fetchDrilldown(id, force?)`. **One fetch backs both cards** (single endpoint); no-op once loaded for the current registry (keyed on `drilldownFor`). `fetchRegistry` resets the drill-down state so a different registry's lines can't leak; error path nulls `drilldownFor` so a re-expand retries.

**Component** (`components/RegistryDrilldownDays.vue`): presentational, phone-first. Per business day → a stacked card (`isoToDisplay(date)` header + day subtotal in the block's semantic color) over the day's records. Ingreso line = customer · `Pedido #orderId` · method label · amount; Egreso line = productName · `qty × unitCost` (+ `· recargo` when surcharge>0) · `lineCost + surcharge`. Empty → "Sin registros en este periodo." `formatMoney` throughout; design tokens only.

**View** (`AccountingDetailView.vue`): per the user's refinement at `/focus` time, each card got its **own expand toggle** ("Ver detalle por día"/"Ocultar detalle" + rotating chevron). An expanded card spans the full grid row (`lg:col-span-2`), pushing the other card below — **independent**, so both can be open at once. The existing aggregate summary stays visible; the drill-down renders beneath the block total with a **"Suma del detalle"** reconciliation row echoing `operationalTotal` / `providerTotal` (the fields the day subtotals sum to — not the manual-inclusive `total`). Spinner/Alert for the lazy load.

**Also (user note):** redesigned the store/period data section from a plain `<p>` grid into a metadata `Card` with icon-labelled rows (Store→Tienda, CalendarRange→Periodo, StickyNote→Notas, Lock→Cerrado el).

**Gates:** `npm run typecheck` + `npm run build` green. Independent validation: **all 5 frontend criteria MET, no correctness bugs** (reconciliation uses the right fields; lazy-fetch triggers on expand; drill-down state resets between registries; egress line = lineCost+surcharge; no edit affordances in the drill-down; tokens only). Manual smoke (expand a block → day subtotals add to the block total) left to the operator.

[2026-06-16] [lpg-frontend-vue] All criteria for this repo met.
[2026-06-16] [lpg-frontend-vue] Follow-on (user request, spec stays `done`): the ingress drill-down's **`Pedido #N`** ref is now a `RouterLink` to the `order-detail` route (`/pedidos/:id`) opening in a **new tab** (`target="_blank" rel="noopener"`), styled `text-primary underline-offset-2 hover:underline`. Deep-linking each line back to its order was listed as optional/out-of-scope in [[index]]; added on request for the order side only (egress purchase lines carry no stable order id). Route is the same operator/admin/developer surface as accounting, so no new role exposure. typecheck + build green.