---
project: lpg-store
domain: specs
type: spec-track
spec: order-event-timeline
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: 2026-06-15
---

# Order Event Timeline — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend track: [[backend]]

## Technical Notes

Consolidates the order detail's two history cards into one horizontal timeline.
Follow `eng/design-system.md` (status→badge mapping, touch targets, `formatMoney`,
Lima-time date helpers). Depends on the backend enrichment (names + assign
target) landing first.

### Merged horizontal timeline

Replace the `Historial` + `Pagos` two-card grid in
[OrderDetailView.vue](../../../../../dev/personal/freelance/lpg-store/lpg-frontend-vue/src/modules/orders/views/OrderDetailView.vue) with a single full-width
component (new `components/OrderTimeline.vue`, replacing the vertical
`StatusTimeline.vue` here — keep or retire that file as appropriate).

- Build one merged, time-sorted list from `order.statusHistory` (status nodes) and
  `order.payments` (payment nodes), keyed by `createdAt` / `occurredAt`.
- Lay out horizontally (scrollable on narrow screens / phone-first for the driver
  view): a connector line with nodes along it, each node a small card with label,
  Lima-time stamp, and actor line(s).
- Node content:
  - Status nodes: the status badge (`ORDER_STATUS_VARIANTS`) + `reason`, plus
    `registrado por {changedByName}`. For the **assignment** node show both actors:
    `asignado por {changedByName} → {targetUserName}`.
  - Payment nodes: `pago · {método} · {monto}` + `registrado por {recordedByName}`.
- Mirror the backend additions in `orders/types.ts` (`changedByName`,
  `targetUserId`, `targetUserName`, `recordedByName`); fall back gracefully when a
  name is missing (old rows: show only the operator).

### Surfaces

Used by both `/pedidos/:id` (operator/admin) and `mis-entregas/:id` (delivery) —
the same `OrderDetailView`. Verify it reads well on a phone for the driver.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify:

- `src/modules/orders/views/OrderDetailView.vue` — replace the two-card grid with
  the single timeline; drop the now-unused Pagos table imports.
- `src/modules/orders/components/OrderTimeline.vue` — new merged horizontal
  timeline (supersedes `StatusTimeline.vue`).
- `src/modules/orders/types.ts` — add the new name/target fields.
- (maybe) `src/modules/orders/components/orderLabels.ts` — any shared label/variant
  helpers the timeline needs.

Reference: the existing `StatusTimeline.vue` (vertical) + the Pagos table in
`OrderDetailView.vue` for the data shapes being merged.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-frontend-vue] Frontend track done → both tracks done, spec done. New `components/OrderTimeline.vue` merges `order.statusHistory` + `order.payments` into one time-sorted **horizontal** timeline (overflow-x-auto, phone-friendly): status nodes show the status badge + reason + `Por {changedByName}`, and when directed (`assigned`) also `→ Para {targetUserName}`; payment nodes show a Pago badge + `{monto} · {método}` + `Por {recordedByName}`. `OrderDetailView.vue` replaced the two-card `lg:grid-cols-2` (Historial + Pagos) with a single full-width Historial card hosting the timeline; dropped the now-unused `StatusTimeline.vue` (deleted, referenced nowhere) + the Pagos table / `PAYMENT_METHOD_LABELS` / `isoToDateTimeDisplay` imports. `types.ts` mirrors the backend additions (`changedByName`, `targetUserId`, `targetUserName`, `recordedByName`). Serves both `/pedidos/:id` and `mis-entregas/:id` (same view, no role gating on the timeline). Independent validation: all 4 frontend criteria PASS, no contract mismatch. Gates green: typecheck + build (PWA precache 64 entries / 713.80 KiB). Manual visual check on a phone left to the operator.

[2026-06-15] [lpg-frontend-vue] Post-ship timeline redesign (spec already `done`; no status change) after operator UI feedback. The first cut had broken connectors (per-column half-lines + `gap` left visible breaks) and fixed `w-52` columns that forced a horizontal scroll. shadcn-vue has no Timeline primitive, so rebuilt custom: extracted the merge/sort into a pure `components/timeline.ts` (`buildTimelineNodes`) and the node content into a shared `OrderTimelineNode.vue`. `OrderTimeline.vue` is now **responsive** — **desktop**: horizontal, nodes as equal `flex-1` columns (no fixed width → no scroll) with a **single continuous rail** inset by `50/n %` so it connects the first/last dot centers exactly (dots `ring-4 ring-background` mask the rail into each node); **mobile** (the driver's surface): vertical with a continuous left line. Added a coarse **hour-range caption** — floors the first event to its hour and ceils the last to the next ("Entre las 11:00 a. m. y las 12:00 p. m.") via new `@/lib/date` helpers `isoToTimeDisplay` / `isoToLimaHour` / `limaHourLabel`; nodes now show time-only (date lives in the caption). Gates green (typecheck + build).

[2026-06-15] [lpg-frontend-vue] Timeline UI v3 (professional pass, spec stays `done`). Reworked to a **centered alternating vertical spine** on desktop: a single continuous center line, cards alternating left/right hugging the spine, the **time on the opposite side** of each card, and the **hour range as a pill capping the top of the spine** (point 1). Mobile keeps a single left-aligned spine with the range pill on top. Rebuilt the node into a real card with **clear hierarchy** (point 4): color-coded Badge = what happened; a strong line = headline (payment **amount** bold / the **actor** name bold) with muted "Por"/"Para" labels; muted detail lines for target, notes, and a **Motivo:** line shown only for `failed`/`cancelled` (canonical status reasons were dropped as redundant with the badge). `OrderTimelineNode` now renders the card box (no inline time — time lives beside the spine). Gates green (typecheck + build).

[2026-06-15] [lpg-frontend-vue] Timeline UI v4 — back to **horizontal** with icons + split range caps. Desktop is now a CSS-grid horizontal spine: a narrow cap column at each end + one flexible `minmax(11rem,1fr)` column per event (3 rows: card / spine / time), cards **alternating** above/below, time opposite each card, continuous rail via per-cell left/right half-segments (no gaps), `overflow-x-auto` only when many events. **Range split to the two ends** (per operator clarification): floored start hour caps the **left** terminus, ceiled end hour caps the **right** terminus (a time axis), with the date as a centered caption above. **Per-event icons + color** on each spine node (size-9 circle): pending=FileText/warning, assigned=UserCheck/secondary, in_transit=Truck/info, delivered=PackageCheck/success, failed=XCircle/destructive, cancelled=Ban/muted, payment=Banknote/success — instantly distinguishable. Mobile keeps a vertical spine with Inicio/Fin caps top & bottom. Gates green (typecheck + build).

[2026-06-15] [lpg-frontend-vue] Killed the desktop horizontal scroll. Event columns changed from `minmax(11rem,1fr)` (which overflowed past ~5 events) to `minmax(0,1fr)` with `min-w-0` cells, so N columns always share the card width — no scroll for normal orders. Past 8 events the horizontal grid would cramp, so a `dense` guard falls back to the vertical spine (grows downward, never sideways) on desktop too. Removed `overflow-x-auto`. (Considered a serpentine/wrapping line but fit-to-width + vertical fallback is simpler and robust at realistic event counts.) Gates green.

[2026-06-15] [lpg-frontend-vue] Added an elapsed-time summary above the timeline: **"Entregado en {creation→delivered}"** and **"Pago total / Pago parcial {delivery→last payment}"** — label keyed on the order's `paymentStatus` prop (`paid` → total, else parcial), measured from delivery (post-fulfilment credit lag; pay-at/before-delivery → `al instante`; hidden until delivered). Reads e.g. "Pago total al instante" / "Pago parcial 2 d". New `formatDurationEs(ms)` helper in `@/lib/date` (coarse Spanish: `2 h 6 min` / `3 d 4 h` / `al instante`). Computed from the timeline nodes — delivery from the `delivered` status event, payment from the latest payment's `occurredAt`; each chip hidden until its milestone exists. Gates green.

[2026-06-15] [lpg-frontend-vue] Two follow-ups (spec stays `done`). (1) **People on the sale** — `OrderDetailView` gained a **Responsables** card showing the **Operador** (registering operator = first/creation status event's `changedByName`) and **Repartidor** (assigned driver = latest `assigned` event's `targetUserName`, "Sin asignar" until assigned / for pre-0010 orders). Derived from the already-enriched `statusHistory`, no new endpoint. (2) **`/mis-entregas` (DeliveryListView)** — replaced the bucket **Select** with a **Tabs** selector (one-click Por entregar / Con saldo / Entregadas / Todas) and switched the card list to a responsive **grid** (`sm:grid-cols-2 lg:grid-cols-3`, smaller cards) from the single-column stack. Removed the now-unused Select/Label imports. Gates green (typecheck + build).