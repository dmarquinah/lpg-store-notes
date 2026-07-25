---
project: lpg-store
domain: specs
type: spec-track
spec: deliver-rehome-open-day
repo: lpg-backend
kind: backend
track-status: not-started
last-updated: 2026-07-15
---

# Auto-rehome a stale-day order on deliver — lpg-backend track

Shared spec: [[index]]

## Technical Notes
- The whole change lives in `OrdersService.deliverOrder` (`src/modules/orders/service.ts:328`).
  Today it fetches `assignment = inventory.getAssignment(order.assignmentId)` and, if
  `assignment.state !== 'open'`, throws the dead-end error (`service.ts:337-342`). Replace
  that throw with a rehome:
  - Look up the same driver+store's open day:
    `const [openDay] = await this.inventory.listAssignments({ storeAssignmentId: assignment.storeAssignmentId, state: 'open' })`
    (`listAssignments` is the trusted internal read port, `service.ts:1243`; `state:'open'` is
    inherently today-only by the open-day invariant, so no `today()` dependency is needed).
  - If `openDay` exists → `deliveryAssignmentId = openDay.id`, `rehomed = true`.
  - If not → `throw new ConflictError('El repartidor no tiene un inventario abierto hoy. Abra el día del repartidor para poder entregar este pedido.')`.
- Thread the resolved `deliveryAssignmentId` (rehomed or the original) through the rest of the
  method: it's the id passed to `inventory.recordSale` / `inventory.recordItemSale`
  (`service.ts:384,401`) — currently the local `assignmentId` (`service.ts:343`). Rename/point
  that at the resolved id.
- **Persist the rebind atomically.** The `delivered` transition already accepts a patch
  (`h.orders.transition(id, order.status, 'delivered', patch?)`, `service.ts:381`). When
  `rehomed`, pass `{ assignmentId: deliveryAssignmentId }` so the order row moves to today's
  day in the same compare-and-swap that flips status — a losing concurrent deliver gets 0 rows
  and aborts before any write, so there's never a partial rehome. When not rehomed, keep the
  call patch-less (unchanged behavior).
- **Traceability without schema churn.** Set the `delivered` status-history event's `reason`
  to reflect the rehome only when it happened (e.g. `'entregado (reasignado al inventario
  abierto de hoy)'`), leaving the normal path's `'entregado'` untouched. `notes` stays the
  user's delivery note — don't mix system text into it.
- **Ledger correctness is free.** `recordSale`/`recordItemSale` already enforce non-negative
  stock (E-invariants) against whatever holder they're given, so an oversell on today's day
  surfaces exactly as it would for a normal same-day delivery. No special handling.
- No change to `assign` (it still requires an open day, `service.ts:257-261`) — assign is a
  routing decision made at bind time; the rehome only repairs the ledger target at deliver.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `src/modules/orders/service.ts` — `deliverOrder` (328-444); the stale-day throw to replace (337-342), the sale-target id (343, 384, 401), the delivered transition patch (381).
- `src/modules/inventory/service.ts` — `listAssignments` (1243) read port; `getAssignment` (1237); `AssignmentFilter` shape (supports `storeAssignmentId` + `state`).
- `src/modules/orders/__tests__/orders.test.ts` — add the three lifecycle cases (stale→rehome, no-open-day→409, still-open→unchanged).
- `src/modules/orders/__tests__/helpers.ts` + `src/modules/inventory/__tests__/helpers.ts` — the fakes; `FakeInventoryRepository.assignments`/`tankHolders` are public, so a test can stage a closed original day + a second open day for the same `storeAssignmentId` (mirroring close→carry→open-next-day) without driving `openDay` across a date boundary.

## Implementation Notes
<!-- [YYYY-MM-DD] [lpg-backend] description -->
