---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../accounting-registry/index]]"
  - "[[../../orders/orders-foundation/index]]"
  - "[[../../inventory/inventory-foundation/index]]"
last-updated: 2026-06-16
---

# Spec: Registry Source Drill-down (verify each block against its records)

## Problem Statement

The accounting registry detail shows only **aggregate totals** — ingress by
payment method, egress as provider tanks / items / surcharge, manual lines, and
a net. There is no way to see *which* orders and *which* purchases produced
those totals. To verify a block — e.g. confirm the `S/ 320.00` cash ingress is
right — the admin must leave the registry, open the orders page, and **sum the
deliveries day by day by hand**. That manual reconciliation is exactly the work
the registry was meant to remove, and it's error-prone.

The underlying records already exist (`order_payments` of delivered orders;
`kind='purchase'` tank/item transactions + `purchase_surcharges`) and the
registry already scopes them to a store + window. They're just never surfaced
at the row level — the read ports `GROUP BY` them away
(`paymentsByMethodForStorePeriod` → `{method, total}`).

The admin needs each block to **expand into its contributing source records,
grouped by day**, so a total can be verified in place. And because a closed
registry serves a **frozen snapshot** (ADR-018), the drill-down for a closed
registry must be frozen too — otherwise the line detail could drift from the
totals it's supposed to explain.

## Proposed Solution

Extend the registry detail with a **per-block, day-grouped drill-down** of the
records behind each total, and freeze that detail into the snapshot at close
(decision 2026-06-16: day-grouped lines, frozen in snapshot).

- **New read ports (ADR-012 composition, no duplicated queries):** thin
  line-level variants of the existing aggregate ports —
  - *Ingress:* delivered-order payments for the store + window, each carrying
    order id, customer name, amount, method, and the payment's Lima business
    date.
  - *Egress:* provider purchases for the store + window, each carrying the
    tank-type/item name, qty, unit cost, line cost, business date, and any
    surcharge.
  These live in the orders / inventory modules (where the joins belong) and are
  composed by accounting, exactly as the existing aggregate ports are.
- **Detail shape:** each block (ingress, egress) gains a drill-down grouped by
  **business day** — a per-day subtotal plus the day's individual records. The
  drill-down subtotals **reconcile exactly** to the block's existing aggregate
  totals (same source, same window, same business-date keying — so the by-method
  / provider totals are unchanged; the drill-down just decomposes them).
- **Frozen for closed registries:** the close snapshot is extended to store the
  line-level drill-down alongside the totals it already freezes, so a closed
  registry's drill-down always matches its frozen books. Open registries compute
  it live.
- **Frontend:** each block in the registry detail becomes expandable, revealing
  the day-grouped records; read-only for closed registries off the frozen
  snapshot.

No new write paths, no change to scoping or to the aggregate totals or net.

Detailed backend design in [[backend]]; the expandable detail UI in
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] New **line-level read ports** (composition, ADR-012 — not duplicated
      queries) return, for a store + window: **ingress** = delivered-order
      payments, each with `{orderId, customerName, amount, method, businessDate}`,
      keyed by the **payment's** business date (same membership rule as the
      aggregate ingress); **egress** = provider purchases, each with
      `{kind: tank|item, productName, qty, unitCost, lineCost, surcharge,
      businessDate}`.
- [ ] The registry detail (`RegistryDetailView`) is extended with a **per-block,
      day-grouped drill-down**: ingress and egress each expose per-day subtotals
      plus the day's records. The drill-down totals **reconcile exactly** to the
      existing aggregate `breakdown` figures (a test asserts the sum of
      drill-down lines equals the block totals).
- [ ] **Closed registries serve a frozen drill-down**: the close snapshot is
      extended to persist the line-level detail, so a closed registry's
      drill-down matches its frozen totals and does not change when later
      payments/purchases land (ADR-018 extended to detail). Open registries
      compute the drill-down live.
- [ ] **Scoping unchanged**: admin/developer global; operators see only their
      `storeIdsForUser` store(s); cross-store → 404/403. No new write routes; the
      drill-down rides on the existing detail endpoint (or a clearly-scoped
      sub-resource) and is role-guarded identically.
- [ ] Tests: open registry — a delivered-order payment and a provider purchase
      appear as drill-down lines under the right day and sum to the block totals;
      close — the drill-down freezes (a later payment doesn't alter a closed
      registry's lines or totals). Existing tests stay green; typecheck / lint /
      build green.

**Frontend (lpg-frontend-vue):**

- [ ] Each block in the registry detail (**ingress by method**, **egress
      provider**) is **expandable** to reveal its day-grouped source records
      (per-day subtotal → the day's orders/purchases), via `formatMoney` and the
      petrol+flame design system.
- [ ] The drill-down visibly reconciles with the block total (the expanded
      subtotals add up to the headline figure the operator already sees).
- [ ] Works **read-only** for a closed registry off the frozen snapshot (same
      expandable view, no live re-query that could drift).
- [ ] typecheck + build green; manual smoke (expand a block → day subtotals add
      up to the block total) left to the operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Cross-period / multi-registry reports & charts** — still the
  accounting-registry's stated "posterior work"; this spec only decomposes a
  single registry's existing blocks.
- **New money sources** — the drill-down explains the *same* ingress (delivered
  payments) and egress (provider purchases + surcharge + manual) the registry
  already aggregates; it adds no new financial inputs.
- **Editing source records from the registry** — the drill-down is read-only;
  orders/purchases are still edited in their own modules.
- **Deep-linking each line back to the order/inventory detail page** — useful but
  optional; can be a follow-up. The line carries enough identity (order id) to
  add it later.

## Open Questions
**All resolved 2026-06-16 (at `/focus` time):**

- **Snapshot size — RESOLVED: freeze full lines.** Persist every payment/purchase
  drill-down line into the closed-registry `snapshot` jsonb. At this scale
  (small-city LPG shop, per-store fortnightly periods ≈ tens–low-hundreds of
  payments + a handful of purchases ≈ 30–50 KB), `jsonb` handles it trivially; no
  pagination/optimization. Keeps a closed registry's drill-down consistent with
  its frozen totals (ADR-018).
- **Ingress drill-down grouping — RESOLVED: per-day subtotal + lines, method as a
  column.** Each ingress day bucket = `{date, subtotal, lines:[{orderId,
  customerName, amount, method}]}`. No redundant per-day `byMethod` map — each
  line carries its method, and the block-level `breakdown.ingress.byMethod`
  already exists for the per-method split.
- **Surface — RESOLVED: new `GET /registries/:id/lines` sub-resource.** Lazy-loaded
  when a block is expanded; keeps the default `GET /registries/:id` detail light.
  Closed registries read the drill-down from the snapshot; open ones compute it
  live. Role-guarded identically to the detail (admin/dev global; operator own
  store; `?storeId` intersected; cross-store → 404/403).
