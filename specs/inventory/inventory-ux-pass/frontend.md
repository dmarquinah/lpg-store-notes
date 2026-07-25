---
project: lpg-store
domain: specs
type: spec-track
spec: inventory-ux-pass
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: '"2026-07-15"'
---

# Inventory UX Pass — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

Frontend-only. Three workstreams: centralize+rename the state labels, fix the
purchase CTA + add "Crear pedido", and a daily-speed sweep.

### 1. Day-state labels — centralize then rename

- Today the map `{ open, closed, carried } → { Abierto, Cerrado, Consolidado }`
  is **inlined in 4 files** (`views/DriverDayView.vue`,
  `components/OpenDayDialog.vue`, `views/InventoryView.vue`,
  `views/AssignmentDetailView.vue`). Create **one** exported map (e.g.
  `src/modules/inventory/labels.ts`, or extend `types.ts`) → `{ open: 'Activo',
  closed: 'Contado', carried: 'Verificado' }` and import it in all four; delete
  the inline copies.
- Grep the module for the old **words as prose** and align them:
  `InventoryView.vue` help paragraph ("…**cerrado** está listo para
  consolidar; …**abierto** debe forzarse el cierre y luego consolidar"), the
  **"Por consolidar"** tab label + **"Consolidar"** / **"Forzar cierre"** buttons,
  `DriverDayView.vue` "Día **consolidado**. El inventario volvió a la tienda",
  `OpenDayDialog.vue` "Sugerido a partir del último día **consolidado**…". Use the
  verb set decided at `/focus` (likely tab **"Por verificar"**, action
  **"Verificar"**). Keep it consistent across badge/tab/button/help.

### 2. Purchase CTA + "Crear pedido"

- The supply-purchase button ("Compra") lives on `views/InventoryView.vue` /
  `views/StoreDetailView.vue` (opens `components/PurchaseDialog.vue`). Relabel to
  the provider-purchase wording chosen at `/focus`.
- Add a **"Crear pedido"** button in the store contexts (Resumen card and/or
  `StoreDetailView`) that routes to the orders wizard **with the store
  pre-selected** — pass the store via route param/query and have the order-entry
  view read it to pre-fill the assigned-store field. Check how the orders module's
  entry view initializes its store selector (it already supports an admin
  owning-branch selector from `orders-multi-location`) and feed the prefill through
  that same field.

### 3. Daily-speed sweep

- Walk `InventoryView`, `StoreDetailView`, `DriverDayView`, `AssignmentDetailView`
  as the operator's daily loop; enumerate the tap-cuts at `/focus` (surface the
  primary recurring action, demote rare/destructive ones, phone reach). Stay within
  `eng/design-system.md` + the `ResponsiveTable`/phone-first rules from
  `mobile-layout-audit` — no token or component-system change.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify:

- `src/modules/inventory/views/InventoryView.vue` — state labels + help copy; "Por
  consolidar" tab; purchase CTA relabel; "Crear pedido" button.
- `src/modules/inventory/views/StoreDetailView.vue` — purchase CTA relabel; "Crear
  pedido" button.
- `src/modules/inventory/views/DriverDayView.vue` — state labels + "Día
  consolidado…" copy.
- `src/modules/inventory/views/AssignmentDetailView.vue` — state labels + any
  consolidate action copy.
- `src/modules/inventory/components/OpenDayDialog.vue` — state label + "último día
  consolidado" copy.
- `src/modules/inventory/labels.ts` (new) or `types.ts` — the shared state-label
  map.

Read-only context (no change):

- `src/modules/orders/` — the order-entry wizard + its store selector (target of
  the "Crear pedido" prefill).
- `src/modules/inventory/store.ts` / `service.ts` — unchanged; labels are display
  only, backend enums stay `open`/`closed`/`carried`.
- `eng/design-system.md`, the `ResponsiveTable` component — sweep must comply.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-15] [lpg-frontend-vue] Full frontend UX + wording pass shipped. Decisions locked at /focus: day-states `Activo`/`Contado`/`Verificado`, operator review action **Verificar** (tab **Por verificar**), admin override **Forzar conteo**, supply-purchase CTA **Compra al proveedor**, **Crear pedido** on both Resumen cards and store detail.

1. **Shared label map** — new `src/modules/inventory/labels.ts` exports `STATE_LABELS` (`open→Activo`, `closed→Contado`, `carried→Verificado`) + `STATE_VARIANT`. Replaced the four duplicated inline maps (`InventoryView`, `DriverDayView`, `AssignmentDetailView`, `OpenDayDialog` — the last previously lowercase prose); dropped now-unused `AssignmentState`/`BadgeVariants` imports. Backend enums untouched (still `open`/`closed`/`carried` on the wire).
2. **Body-copy sweep** — aligned every user-facing string to the new vocab: `InventoryView` stale-day help + hint + "Por verificar" tab + verify-queue copy; `AssignmentDetailView` `Consolidar→Verificar`, `Forzar cierre→Forzar conteo`; `DriverDayView` "Día verificado" + "lo revisará y verificará" + empty-state "día activo"; `OpenDayDialog` "sin verificar"/"último día verificado"; `CountDialog` "lo revisará y verificará"; `CloseDialog` title/button `Cerrar día→Forzar conteo`; `PurchaseDialog` title `→Compra al proveedor` + "día activo"; `StoreDetailView` "asignaciones activas"; `store.ts` three error fallbacks (`verificar el día`, `registrar el conteo del día`, `cola de días por verificar`). Internal code identifiers (`consolidationList`, `fetchConsolidationList`, tab `value="consolidate"`) left as-is (not user-facing).
3. **Purchase CTA** — `Compra`/`Registrar compra` → **Compra al proveedor** (`InventoryView` card, `StoreDetailView` header, `PurchaseDialog` title).
4. **Crear pedido** — new `goToCreateOrder` in `InventoryView` (per Resumen card) + `StoreDetailView` (header) pushing `{ name: 'order-create', query: { storeId } }`; `OrderCreateView` reads `route.query.storeId` (`prefillStoreId`) and pre-selects the owning-branch selector — managers trust it as-is, operators use it only if it's one of their stores (else first). No new order fields/flow.
5. **Daily-speed sweep** — `InventoryView` card footer reordered by recurrence (quick-open ▸ Ver/Abrir día ▸ Crear pedido/Compra al proveedor), `flex-wrap` so no overflow; `StoreDetailView` header leads with Crear pedido (primary) ▸ Compra al proveedor ▸ Ajustar stock. Design-system compliant (existing button sizes/tokens).

No backend/enum/payload/endpoint change. Validated against all six acceptance criteria (a validation agent flagged five leftover strings in `store.ts` + two headings; all fixed). `npm run typecheck` + `npm run build` green.

[2026-07-15] [lpg-frontend-vue] All criteria for this repo met.
