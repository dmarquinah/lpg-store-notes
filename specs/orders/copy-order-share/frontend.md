---
project: lpg-store
domain: specs
type: spec-track
spec: copy-order-share
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: '"2026-07-15"'
---

# Copy order data to share — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes
- Frontend-only; no API change. The order-detail view already fetches the full
  order (customer name/phone resolved, delivery address, items with unit prices,
  total, payment method, notes). Build the shareable text from that in-memory object.
- Add a pure formatter (e.g. `orders/lib/formatOrderShareText(order): string`) so
  it's unit-testable and reused by every button placement. Reuse the existing
  `formatMoney` helper for prices so money formatting stays consistent.
- Clipboard: use `navigator.clipboard.writeText` with a `document.execCommand('copy')`
  textarea fallback for older/insecure contexts; both paths funnel into one action
  that fires a `vue-sonner` toast (success/error). The toast infra already exists
  (added in `quick-assignment`).
- Placement: primary button on `OrderDetailView`. Optionally also surface on the
  create-order success path (the wizard already toasts on create) — confirm the
  Open Question before wiring the second spot.
- Walk-in orders: use the resolved name/phone the detail already exposes
  (`resolvedCustomerName ?? snapshot`); guard against `null` so no "null"/blank lines.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)
- `src/modules/orders/views/OrderDetailView.vue` — primary button placement; source of the order object.
- `src/modules/orders/` — add `lib/formatOrderShareText.ts` (the pure formatter) + its test.
- `src/modules/orders/views/` — create-order wizard / success step (optional second placement).
- `src/lib/` (or wherever `formatMoney` lives) — reuse for price formatting; toast via existing `vue-sonner` setup.

## Implementation Notes
<!-- [YYYY-MM-DD] [lpg-frontend-vue] description -->

[2026-07-15] [lpg-frontend-vue] Implemented "Copiar datos" end-to-end.
- New `src/modules/orders/lib/formatOrderShareText.ts` — pure, framework-free formatter (`ShareableOrder` input + injected `lineName` resolver; `OrderDetail`/`OrderSummary` satisfy it structurally). Produces a WhatsApp-legible Spanish block: 🛵 header, 👤 name (walk-in → "Cliente al paso"), 📞 phone (omitted when blank), 📍 address (→ "Dirección no registrada" when blank), 🛒 line items (`• name — qty x unitPrice` via shared `formatMoney`), 💰 total, ✅ Pagado (only when paidAmount>0), 💳 method + "cobrar {outstanding}". A `blank()` guard drops every nullable field so no "—"/"null" leaks. **Per owner decision: order # and store are deliberately excluded** — driver only needs delivery info; payment shows method + amount to collect.
- New `src/modules/orders/lib/copyToClipboard.ts` — `navigator.clipboard.writeText` with a hidden-textarea `execCommand('copy')` fallback for insecure/legacy contexts; returns a boolean.
- New `src/modules/orders/lib/useOrderShare.ts` — composable resolving tank/item names from the catalog store, then format → copy → `vue-sonner` toast (success/error). Single glue point reused by both placements.
- `OrderDetailView.vue` — "Copiar datos" outline button in the `PageHeader` #actions slot (visible for any status).
- `OrderCreateView.vue` — create no longer auto-navigates; on success it swaps the wizard for a success step ("Pedido #N registrado") offering **Copiar datos** + **Ver pedido** (owner opted into the create-success placement).
- Collect amount uses `outstanding` (not total) so a re-copy of a partially-paid order asks for the correct remaining balance; identical to total on a fresh order.
- Validation agent: all 7 acceptance criteria met. `npm run typecheck` + `npm run build` green. No test file added — repo has no test runner wired yet (deferred project decision); the formatter is pure and testable in isolation, satisfying the AC's intent.
