---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: draft
depends-on: []
last-updated: 2026-07-15
---

# Spec: Auto-rehome a stale-day order to today's open inventory on deliver

## Problem Statement
An order is assigned to a specific **driver-day** (`orders.assignmentId` → an inventory
assignment). Delivery records the sale against that day's holder. But an order can
outlive the day it was assigned to: it's created and assigned on day D, not fulfilled,
and day D is later **closed/consolidated**. When the driver then taps **"Confirmar
entrega"** on day D+1, `deliverOrder` correctly refuses — recording a sale against a
closed, already-counted day would corrupt that day's reconciliation — and returns
`El inventario asignado ya está cerrado — reasigne el pedido al inventario abierto del
repartidor` ([service.ts:337-342](../../../../lpg-backend/src/modules/orders/service.ts)).

The problem: **there is no reassign affordance.** `assign` only moves `pending|failed →
assigned`, and this order is already `in_transit`, so there's no path back. The operator/
driver is stuck. The tank is physically on **today's** truck (the previous day carried
forward), so the sale *should* land on today's open day — the system just doesn't do it.

## Proposed Solution
When `deliverOrder` finds the bound assignment is no longer `open`, **automatically
rehome** the order to the **same driver+store's open day today** and record the sale
there — instead of throwing the dead-end error.

- The eligible target is an open assignment sharing the original assignment's
  `storeAssignmentId` (the fixed driver↔store pairing). This guarantees the rehome
  never crosses to a different driver or branch. By the today-only-open invariant there
  is at most one such day.
- The order's `assignmentId` is updated to that open day, persisted **atomically** with
  the `delivered` status flip (via the existing transition patch), so the ledger and
  day-summary attribute the sale to today's day.
- If the driver+store has **no** open day today, deliver is rejected with a clear,
  actionable message ("abra el día del repartidor para poder entregar"), replacing the
  old reassign-yourself dead end.
- The rehome is recorded on the `delivered` status-history event (reason reflects the
  auto-reassignment) so the order timeline shows it happened.

Silent by design (the driver just confirms delivery and it works). No new endpoint, no
migration, no schema change — a branch inside `deliverOrder` plus a lookup on the
inventory read port that already exists (`listAssignments`).

## Acceptance Criteria
<!-- THE single shared checklist — source of truth across all tracks. -->
- [ ] Delivering an order whose bound driver-day is no longer `open` (closed/carried) auto-rebinds it to the **same driver+store's** open day today (matched by the original `storeAssignmentId`) and records the tanks/items/empties/debt against that open day.
- [ ] The order's `assignmentId` is updated to the open day, persisted atomically with the `delivered` transition (a losing concurrent deliver still aborts cleanly — no partial rehome).
- [ ] The rehome never crosses driver or branch: only an open day sharing the original `storeAssignmentId` is eligible; a different driver's/store's open day is never chosen.
- [ ] If the driver+store has **no** open day today, deliver is rejected with a clear, actionable 409 (guide the operator to open the driver's day) — not the old dead-end reassign message.
- [ ] When the bound day is still `open`, delivery is unchanged — no rehome path, byte-for-byte the same behavior and history as today.
- [ ] Traceability: the `delivered` status-history event reflects that an auto-rehome occurred, so the timeline shows the order moved to today's inventory.
- [ ] Backend tests cover: (a) stale/closed day → rehomes to the open day, sale lands there, `assignmentId` updated; (b) no open day today → clear 409; (c) still-open day → unchanged. Full gate green (typecheck, check, test, build).

## Tracks
<!-- Overall status becomes `done` only when EVERY track is done. -->

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | not-started |

## Out of Scope
- **Frontend changes.** The happy path just succeeds after the backend fix; the new
  "no open day today" message is surfaced by the existing error handling. A visible
  "reasignado a hoy" indicator can be a later follow-up if wanted.
- Rehoming at `assign`/`dispatch` — only `deliver` touches the ledger, so only it needs it.
- Any change to how inventory days are opened / closed / consolidated / carried.
- Choosing among multiple open days for one store-assignment (the invariant is ≤1).
- Moving an order to a *different* driver or branch (explicitly disallowed).

## Open Questions
- Should the driver's app show a subtle "se reasignó al inventario de hoy" note on
  success? Deferred — silent by design per the agreed decision; the ledger attribution
  and the `delivered` event's reason already provide the trace.
