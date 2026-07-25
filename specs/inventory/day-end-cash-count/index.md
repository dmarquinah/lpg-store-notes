---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../day-handoff/index]]"
  - "[[../inventory-foundation/index]]"
  - "[[../../orders/orders-foundation/index]]"
  - "[[../../accounting/accounting-registry/index]]"
last-updated: '"2026-07-10"'
---

# Spec: Day-End Cash Count (driver money + debt attestation)

## Problem Statement

The end-of-day process for a **delivery driver** is incomplete and poorly surfaced. Two gaps:

1. **The day-end only counts tanks, not money.** [[../day-handoff/index|day-handoff]] made the driver's physical **tank** count a mandatory attestation (`finalCounts`), but the driver never counts or confirms **money**. The real business (see the operator's paper ledger) closes the day by tallying, per driver:
   - **balones entregados** — tanks handed out (the platform has this),
   - **money collected split by payment method** — Efectivo (cash) vs Yape vs Plin/transfer (the platform does **not** surface this per driver-day),
   - **saldo crédito** — money owed that accrued that day, and **cilindros prestados** — empty tanks loaned/owed.

   All the money/debt data exists in the ledger but is **never summarized for the driver**, so the driver has no clear "this is what I'm handing in" figure to attest to, and there is no cash-vs-expected control.

2. **The end-of-day button is always enabled from the start of the day.** The driver can tap "terminar día" at 8am before doing any work; nothing signals *when* it's appropriate. The action needs to read as available-when-you're-done without hard-blocking a legitimate empty day.

**Primary value:** give the driver a clear, summarized end-of-day recap (tanks + money-by-method + debt accrued) that they **attest** to, make cash discrepancies **explained and admin-visible**, and carry that same recap into the operator's second-signature review — extending the two-signature model from tanks to money.

## Proposed Solution

A backend + frontend feature layered on the existing `day-handoff` close/carry flow. **The money and debt figures are computed from existing data** (orders, `order_payments`, `customer_debts`, `customer_empty_debts`) — no change to how money is recorded. What's new is: a **read** that aggregates money + debt per driver-day, **capture** of the driver's counted cash + discrepancy note at close, **admin discoverability** of discrepancies, a new **`vale FISE`** payment tender, and **surfacing** the recap on both the driver and operator surfaces.

Decisions taken with the owner (2026-07-09/10):

- **Driver attests counted cash (not just views it).** At close the driver enters counted money **per payment method**; the backend shows **counted vs expected**. Money analogue of `finalCounts` for tanks.
- **"Expected cash" = what the driver *collected* today (Option B), not just what they delivered.** Expected is derived from **`order_payments` where `recordedBy` = the assignment's driver and `occurredAt` falls in the assignment's business day** — so cash collected **against a prior debt** in the field is included, matching the physical cash-in-hand the driver actually hands in. (Anchored to the assignment: driver = `storeAssignments.userId`, day = `inventory_assignments.date`.)
- **A discrepancy requires an explanation, and the admin can find it.** If counted ≠ expected for any method, the driver must leave a **note** (close rejected otherwise). The close sets a discoverable **`has_cash_discrepancy`** flag on the assignment + persists the per-method counted/expected/note, so an admin/operator can filter to the discrepant days — mirroring the `cierre tardío` audit marker (ADR-017). Any review/sign-off lives in the **operator/admin confirmation** (the `carry`/verify step).
- **New payment tender `vale FISE`.** Add `fise` to `order_payment_method` (enum migration) + "Vale FISE" label. FISE is a Peruvian LPG subsidy voucher used as partial tender at the point of sale, so it belongs alongside `cash`/`yape`/`plin`/`transfer` and must be countable at day-end.
- **Count form: active methods editable, inactive shown muted.** The form shows a counted input for every method with activity today (expected > 0) **plus always Efectivo (cash)**; methods with no movement are still shown but **muted/greyed** (transparency — the driver sees they were 0, not hidden).
- **Debt recap: totals + itemized detail.** Show the headline day totals (money credit created today + empty tanks loaned today) **and** an expandable per-customer/per-order breakdown, so the driver can verify how the totals were reached (confidence + transparency).
- **End-of-day button: always visible, de-emphasized early.** Reachable but **secondary/muted with a hint** ("disponible al terminar tus entregas") until the driver has **≥1 delivered order today**, then the prominent primary action. No hard gate — an empty-day close is still possible.
- **The recap appears on both signatures.** The driver sees tanks + money-by-method + debt-accrued at close; the **operator's Consolidar/verify review shows the same recap** plus counted-vs-expected and any discrepancy notes, so the second signature covers money.

Note on 100%-credit sales (already supported): a registered-customer delivery with no payment writes **no `order_payments` row** and a full-total `customer_debts` charge ([orders/service.ts:346-357](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/orders/service.ts)); the day summary must reflect it as **0 cash + full debt** (Option B handles it naturally — the cash lands on the later payoff day).

Detailed backend design (new read + capture + migration + FISE enum + discrepancy record) lives in [[backend]]; the driver end-of-day screen, button gating, and operator review UI in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `order_payment_method` gains **`fise`** (vale FISE) via an enum migration; existing payment recording accepts it wherever `cash`/`yape`/`plin`/`transfer` are accepted (no other behavior change).
- [ ] A new read returns a **driver-day money/debt summary** for a single assignment: **money collected by payment method** = `order_payments` where `recordedBy` = the assignment's driver **and** `occurredAt` in the assignment's business day (**Option B — collected-today**, includes cash servicing a prior debt); **tanks handed out** per type; **money debt accrued that day** (`customer_debts` rows for orders on this assignment, `createdAt` in the day — the delta, not the cumulative balance); **empty-tank debt accrued that day** (`customer_empty_debts` via `refTankTransactionId` → this assignment, `occurredAt` in the day — delta). Debt is returned both as **totals** and **itemized** (per customer/order) so the figures are verifiable. Business-day boundaries via `src/lib/date.ts` / `-05:00`, anchored to `inventory_assignments.date`.
- [ ] The close flow captures the driver's **counted cash per payment method** plus an optional **discrepancy note**, persisted with a snapshot of the **expected** amount per method (new `assignment_cash_counts` table + migration; money analogue of the `reconciliation` tank rows).
- [ ] If counted ≠ expected for any method, a **discrepancy note is required** → a close missing the note is rejected (400); a clean close needs no note; admin/developer override may bypass, consistent with the existing tank-close override.
- [ ] A close with any per-method discrepancy sets a discoverable **`has_cash_discrepancy`** marker on the assignment, and the assignments list supports a **`?hasCashDiscrepancy=true`** filter, so admin/operator can find discrepant days without scanning.
- [ ] The operator's consolidation/verify read (the `closed`-awaiting-carry surface) returns the **same money/debt recap** plus counted-vs-expected and the driver's discrepancy note(s), so the second signature covers money.
- [ ] The existing tank `finalCounts` attestation, `carry` hand-back, tank discrepancy reads, and all `day-handoff` guards are **unchanged**; new tests cover the FISE method, the summary read (Option B money + debt delta, itemized), counted-vs-expected capture, discrepancy-note-required (400), and the `hasCashDiscrepancy` filter; existing inventory/orders tests stay green; typecheck / lint / test / build green; migrations applied to the dev DB.

**Frontend (lpg-frontend-vue):**

- [ ] The driver's end-of-day surface (`/mi-dia` / `DriverDayView`) shows a **summarized recap**: balones entregados (per type), **money collected by payment method** (incl. Vale FISE), and **debt accrued today** — shown as **totals with an expandable per-customer/order detail** so the driver can verify the figures. Phone-first, `formatMoney`.
- [ ] The end-of-day flow captures **counted cash per method** with **expected shown alongside**; methods with activity (expected > 0) plus **Efectivo always** are editable, methods with no movement are **shown muted/greyed** (not hidden). When counted ≠ expected for any method the UI **requires a note** before submit (mirrors the backend 400). Tank `finalCounts` capture stays part of the same close.
- [ ] The end-of-day button is **always visible but de-emphasized** (muted + "disponible al terminar tus entregas" hint) until the driver has **≥1 delivered order today**, then renders as the prominent primary action. No hard block on an empty-day close.
- [ ] The operator's **Consolidar/verify** review shows the **same money/debt recap** with the driver's counted-vs-expected and discrepancy note(s); the operator confirms (`carry`) with money visibility, not just tanks.
- [ ] An admin/operator can **find the discrepant days** in the UI (badge on the consolidation/assignment surface, backed by the `hasCashDiscrepancy` filter).
- [ ] "Vale FISE" appears as a payment method wherever methods are chosen/shown (delivery payment + this recap). Follows `design-system` + `mobile-layout-audit`; `npm run typecheck` + `npm run build` green; owner smoke of the full attest → discrepancy-note → verify flow.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Changing how money is recorded.** Payment capture (`order_payments`, partials, the credit/walk-in rules) is untouched apart from **adding the `fise` method**; this feature aggregates, attests, and surfaces money — it does not rewrite delivery/payment logic.
- **Store/period accounting registry.** The per-store closing register ([[../../accounting/accounting-registry/index]]) stays as-is; this is the per-driver-day view, reusing the by-method aggregation pattern keyed on the driver+day instead of the store. (FISE will also show up there once it's a method — no extra work, same grouping.)
- **The day-state wording pass.** [[../inventory-ux-pass/index]] renames `Abierto/Cerrado/Consolidado` and touches `DriverDayView`; coordinate file ordering, but the vocabulary rename is that sibling spec's job.
- **Backend enum/state-machine changes to the day lifecycle.** `open|closed|carried` and the two-signature role split are unchanged (only the `order_payment_method` enum grows).

## Open Questions

_All resolved with the owner (2026-07-09/10) — folded into Proposed Solution + Acceptance Criteria:_

- **Which payments count as the day's cash** → **Option B (collected-today):** `order_payments` by `recordedBy` = driver + `occurredAt` in the day, including cash servicing prior debts.
- **Methods shown in the count form** → active methods (expected > 0) + always Efectivo editable; no-movement methods shown **muted**, not hidden. **Vale FISE added** as a new method.
- **Discrepancy record shape** → dedicated `assignment_cash_counts` table (per-method expected/counted/note).
- **Debt recap granularity** → **both** day totals **and** an itemized per-customer/order breakdown (verifiability/transparency).
- **Admin discoverability** → `has_cash_discrepancy` marker on the assignment + a `?hasCashDiscrepancy=true` list filter.

Confirmed in code (not a question, but recorded): **100%-credit sales are already supported** — a registered-customer delivery with no payment writes no `order_payments` row + a full-total `customer_debts` charge; walk-ins must pay in full ([orders/service.ts:346-357](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/orders/service.ts)). The summary treats such a delivery as 0 cash + full debt.

Remaining minor decision for `/focus` (implementation detail, not blocking): whether a driver working **two stores in one day** (two assignments) should see combined or per-assignment cash — Option B's `recordedBy`+day scope naturally combines the physical cash; per-assignment splitting only matters if that case actually occurs.
