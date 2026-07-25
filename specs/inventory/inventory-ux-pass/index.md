---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../day-handoff/index]]"
  - "[[../store-stock-first/index]]"
  - "[[../store-detail-products/index]]"
  - "[[../quick-assignment/index]]"
last-updated: '"2026-07-15"'
---

# Spec: Inventory UX Pass (day-state wording + faster daily actions)

## Problem Statement

The inventory pages accreted across ~8 specs and the daily-work vocabulary + call
-to-action wording no longer read well for the real (low-tech-literacy) operator:

1. **The day-state labels are jargon and are duplicated in code.** The three
   truck-day states show as **`Abierto` / `Cerrado` / `Consolidado`**, hard-coded
   in **four** frontend files (`DriverDayView.vue`, `OpenDayDialog.vue`,
   `InventoryView.vue`, `AssignmentDetailView.vue`) with no shared map. The owner
   finds them unclear: `Abierto` doesn't say "the driver is working", `Cerrado`
   vs `Consolidado` is an opaque distinction, and `Consolidado` is accounting
   jargon. (Backend enums stay `open`/`closed`/`carried` — this is display only.)
2. **"Compra" is the wrong verb and the only action.** On `/inventario` the
   **"Compra"** button is really *"register a supply purchase from the provider"*
   — but reads like a customer sale. And from an inventory page there's **no way
   to start a customer order** for that store; the operator has to leave for
   `/pedidos` and re-pick the store.
3. **The daily loop is slower than it needs to be.** With
   [[../quick-assignment/index|quick-assignment]] adding a one-tap open, the
   surrounding pages should be swept for the same goal: get the operator through
   the day's recurring actions (open, purchase-in, create order, count/consolidate)
   in the fewest taps, on a phone.

## Proposed Solution

A **frontend-only** UX + wording pass over the inventory pages. No backend
change (state enums, endpoints, and payloads are untouched).

- **Rename + centralize the day-state labels.** Adopt **`Activo` → `Contado` →
  `Verificado`** (decided):
  - `open` → **Activo** — the driver is currently working the day.
  - `closed` → **Contado** — the driver did the physical count; awaiting operator
    review.
  - `carried` → **Verificado** — the operator verified & closed the day.

  Introduce a **single shared label map** (`inventory/labels.ts` or the module's
  `types.ts`) and replace the four duplicated inline maps with it. Also align the
  **body-copy verbs** that leak the old words ("consolidar", "abrir",
  "día consolidado", the `InventoryView` help text, the "Por consolidar" tab and
  `Consolidar`/`Forzar cierre` buttons) with the new vocabulary — chosen at
  `/focus` so the action words stay consistent (e.g. tab **"Por verificar"**,
  action **"Verificar"**).
- **Fix the "Compra" call-to-action + add "Crear pedido".** Relabel the supply-
  purchase button to name what it is — **"Registrar compra"** / **"Compra al
  proveedor"** (chosen at `/focus`) — and add a sibling **"Crear pedido"** button
  that navigates to the order-entry form **pre-filled with that store** (the
  location that owns the button) as the assigned store. Reuses the existing order
  wizard via a route query/param; no new order backend.
- **Daily-speed sweep.** Audit the inventory pages (`InventoryView`,
  `StoreDetailView`, `DriverDayView`, `AssignmentDetailView`) for the recurring
  daily actions and cut taps where cheap — surface the primary action per context,
  keep destructive/rare actions secondary, ensure phone-first reach (≥44px, no
  overflow, consistent with the `design-system` + `mobile-layout-audit` rules).
  The concrete cut-list is enumerated at `/focus` from a walk-through.

## Acceptance Criteria

<!-- THE single shared checklist — source of truth (single frontend track). -->

**Frontend (lpg-frontend-vue):**

- [x] Day-state labels read **`Activo` / `Contado` / `Verificado`** everywhere a
      state is shown (badges, dialogs, help text, driver view), driven by a
      **single shared label map** — the four duplicated inline maps
      (`DriverDayView`, `OpenDayDialog`, `InventoryView`, `AssignmentDetailView`)
      are replaced by it. **Backend enums are unchanged** (`open`/`closed`/`carried`
      still on the wire).
- [x] Body-copy verbs and derived UI text that referenced the old words are
      aligned to the new vocabulary (e.g. the "Por consolidar" tab, the
      `Consolidar` action, the `InventoryView` explanatory paragraph, the
      `DriverDayView` "Día consolidado…" line). Final action words fixed at
      `/focus`; whatever is chosen is used **consistently**.
- [x] The `/inventario` supply-purchase button is **relabelled** to clearly mean a
      provider purchase (not a customer sale); wording fixed at `/focus`.
- [x] A **"Crear pedido"** button on the inventory surface(s) opens the order-entry
      form **pre-filled with the originating store** as the assigned store (via
      route param/query into the existing orders wizard) — no manual store re-pick.
- [x] The inventory pages get a **daily-speed sweep**: the primary recurring action
      is surfaced per context; the cut-list from the `/focus` walk-through is
      applied. Phone-first + design-system compliant (tokens, ≥44px, no overflow;
      consistent with [[../../ui-design/design-system/index]] +
      [[../../ui-design/mobile-layout-audit/index]]).
- [x] Display/labels/navigation only — **no backend call, payload, or enum
      change**. `npm run typecheck` + `npm run build` green; operator smoke of the
      renamed states + "Crear pedido" prefill left to the owner.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Backend enum rename.** The wire values stay `open`/`closed`/`carried`; only
  Spanish display labels change. (Optionally align backend Spanish *error* copy
  like "día sin consolidar" later — not in this spec.)
- **The single-delivery automation** (quick-open, purchase auto-load, dialog
  driver scoping) — sibling spec [[../quick-assignment/index]].
- **Order-entry behavior beyond the store prefill** — "Crear pedido" reuses the
  existing wizard as-is; no new order fields or flow.
- **New inventory data/endpoints.** This is a labels + navigation + layout pass.

## Open Questions

- **End-state word: `Verificado` vs `Finalizado`.** Decided `Verificado` (operator
  *verified*). Revisit at `/focus` only if the owner prefers `Finalizado` on
  seeing it in context.
- **Action-verb consistency.** If `carried` displays as **Verificado**, the
  consolidation action/tab should likely read **"Verificar" / "Por verificar"**
  (not the old "Consolidar"). Confirm the full verb set at `/focus` so button, tab,
  and state agree.
- **Supply-purchase button wording:** "Registrar compra" vs "Compra al proveedor"
  vs "Reabastecer". Pick at `/focus` (favor the one the owner recognizes fastest).
- **Where "Crear pedido" lives** — Resumen card, StoreDetailView, or both. Proposed:
  the store contexts that already show per-store actions; confirm at `/focus`.
- **Full cut-list for the daily-speed sweep** — enumerate from a page walk-through
  at `/focus` rather than guessing here.
