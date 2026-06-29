---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../../accounting/accounting-registry/index]]"
last-updated: 2026-06-16
---

# Spec: Provider Purchase Cost (editable cost-in, captured per purchase)

## Problem Statement

The accounting registry's **egress** block over-states what the owner paid the
provider. A provider purchase is valued at the **holder's snapshot
`purchase_price`** (`purchaseCostsForStorePeriod` → `SUM(holder.purchase_price ×
qty)`), and that snapshot is taken **once** from `tank_types.purchase_price`
when the location holder is first created and then reused forever. In practice
the catalog's `purchase_price` is set equal to (or near) the **sell price**, so
the egress block charges the business the price it *sells* tanks at — not the
cheaper price the provider charges the owner. The provider deliberately sells to
the owner below the street price so the owner earns a margin; the books must
reflect that lower cost-in.

Worse, there is today **no way to enter the real cost at all**: the purchase
endpoints (`tankPurchaseSchema` / `itemPurchaseSchema`) accept only
`qty`/`emptyReturned`/`surcharge` — no cost field. Each purchase silently
inherits the stale holder snapshot, so even correcting the catalog once wouldn't
capture a provider price that varies purchase-to-purchase.

The owner needs to **enter (and adjust) the provider cost when recording each
purchase**, and the registry's egress must value purchases at that captured
cost.

## Proposed Solution

Capture an **editable per-line unit cost at purchase time** and value egress
against it — leaving the holder snapshot only as the legacy fallback.

- The tank and item purchase requests gain an **optional per-line `unitCost`**
  (`numeric(10,2)`, ≥ 0, ≤ 2 decimals). The frontend **requires** it (pre-filled
  and confirmable); the backend keeps it optional so non-UI callers (the
  "Comprar y cargar" assign path) don't break. When omitted, the service
  defaults it to the **last actual purchase cost** for that store + product (the
  most recent `kind='purchase'` `unit_cost`), falling back to the catalog
  `purchase_price` on the first-ever purchase. **The catalog is not mutated.**
  (Decisions 2026-06-16.)
- A new **nullable `unit_cost`** column on `tank_transactions` and
  `item_transactions` stores the captured cost, set only for `kind='purchase'`
  rows. The ledger stays append-only (ADR-009); this is a per-event price
  capture, the purchase analogue of the existing `purchase_surcharges` side
  data.
- Egress valuation (`purchaseCostsForStorePeriod` and the batched
  `purchaseTotalsForStorePeriods`) values a purchase as
  `COALESCE(tx.unit_cost, holder.purchase_price) × qty` — so historical
  pre-migration purchases keep valuing at the holder snapshot (backward
  compatible) while every new purchase values at the entered cost.
- The frontend tank/item **purchase dialogs** gain a **required** per-line cost
  field, pre-filled with the last purchase cost (catalog fallback).
- **Correcting a cost after the fact:** a `PATCH` endpoint updates the
  `unit_cost` of an existing purchase transaction (operators may mis-enter under
  time pressure) — but **only while the purchase is not yet accounted**. If its
  business date falls inside a **closed** registry the period is frozen for
  everyone → **409** (no edit, no reopen); open/un-accounted periods reflect a
  correction live. The accounted check is an **injected port** (accounting →
  inventory, wired in `app.ts`) so inventory never imports accounting.

No change to how sales value inventory or to the holder `sell_price`. This is a
cost-in correctness fix scoped to provider purchases; the accounting egress
block is the consumer and needs no formula change beyond the `COALESCE`.

Detailed backend design in [[backend]]; the purchase-dialog changes in
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [x] Migration `0012_purchase_unit_cost` (named, applied to the dev DB) adds a **nullable**
      `unit_cost numeric(10,2)` column to both `tank_transactions` and
      `item_transactions`. Existing rows stay `NULL`. No backfill.
- [x] `tankPurchaseSchema` and `itemPurchaseSchema` gain an **optional per-line
      `unitCost`** (number, ≥ 0, ≤ 2 decimals; finite). `recordTankPurchase` /
      `recordItemPurchase` persist it onto the inserted `kind='purchase'`
      transaction; when omitted, the service writes the **last actual purchase
      cost** for that store + product (most recent `kind='purchase'`
      `unit_cost`), falling back to the catalog `purchase_price` on the
      first-ever purchase. **The catalog row is not updated.**
- [x] Egress valuation reads the captured cost: `purchaseCostsForStorePeriod`
      **and** the batched `purchaseTotalsForStorePeriods` value each purchase as
      `COALESCE(tank_transactions.unit_cost, tank_holders.purchase_price) × qty`
      (and the item equivalent). Pre-migration rows (`unit_cost IS NULL`) still
      value at the holder snapshot — verified unchanged.
- [x] A **purchase-cost correction** endpoint (`PATCH`, role-guarded like
      recording a purchase: operator own-store + admin/dev) updates an existing
      purchase transaction's `unit_cost`; non-purchase rows are rejected
      (`ValidationError`). **Frozen once accounted**: if the purchase's business
      date is covered by a **closed** registry the edit is rejected (`409`) for
      every role; open/un-accounted periods recompute live. Tests cover both a
      correction moving an open registry's egress and the closed-period 409.
- [x] A **purchases list** endpoint `GET /inventory/stores/:storeId/purchases?from=&to=`
      returns the store's provider purchase lines (tank + item) over a Lima
      business-date window — each with `productName`, `qty`, `unitCost`,
      `lineCost`, `surcharge`, `occurredAt`, and an **`accounted`** flag (its day
      is covered by a closed registry → cost frozen). Operator own-store scoped;
      admin/dev global; bounds default to today. The shared repo read
      (`purchaseLinesForStorePeriod`) is the one the `registry-source-drilldown`
      spec will reuse.
- [x] `purchase_surcharges` valuation is unchanged; sale/return/other tx kinds
      are unaffected (their `unit_cost` stays `NULL`). Scoping/permissions on
      purchases are unchanged.
- [x] The multi-type-fulfillment **"Comprar y cargar"** purchase path keeps
      working: omitting `unitCost` falls back to the catalog default (no caller
      is forced to send a cost).
- [x] Tests: a purchase with an explicit `unitCost` **below** the sell price
      values egress at the entered cost (not the sell/holder price); an omitted
      `unitCost` values at the **last purchase cost** (catalog on the first
      purchase); a pre-existing
      `unit_cost IS NULL` row still values via the holder snapshot. Existing
      tests stay green; typecheck / lint / build green.

**Frontend (lpg-frontend-vue):**

- [x] The **tank** and **item** purchase dialogs in the `inventory` module gain
      a **required** per-line unit cost field, pre-filled with the last purchase
      cost (catalog fallback), validated (≥ 0, ≤ 2 decimals), sent as
      `unitCost`. The operator confirms or edits it every purchase.
- [x] The cost field uses `formatMoney`/the petrol+flame design system and
      reads clearly as "cost paid to the provider" (distinct from the sell
      price).
- [x] The orders `AssignDialog` "Comprar y cargar" smart-purchase path is
      unaffected (continues to send no cost → catalog default) or, if a cost
      surface is added there, pre-fills it the same way.
- [x] The inventory **Movimientos / purchases** view lets an authorized user
      **correct a purchase's cost** after the fact (calls the PATCH endpoint,
      same validation); the change reflects in an open registry's egress. A
      purchase whose period is **accounted (closed registry)** surfaces the
      backend **409** as a clear "periodo cerrado" frozen state (no edit).
- [x] typecheck + build green; manual smoke (record a purchase with a cost below
      sell price → registry egress drops to the entered cost) left to the
      operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Re-pricing past purchases / backfilling `unit_cost`** — existing rows keep
  valuing at the holder snapshot via `COALESCE`. No retroactive correction.
- **Changing how sales or on-hand inventory are valued** — `sell_price` and the
  holder snapshot for non-purchase concerns are untouched. This is cost-in only.
- **Catalog price-management UX** — editing `tank_types`/`inventory_items`
  default `purchase_price` stays where it is; this spec only *reads* it as a
  pre-fill default and never writes it back.
- **True COGS / margin accounting** — egress remains cash-out to the provider,
  not cost-of-goods matched to revenue (per the accounting-registry scope).

## Open Questions (resolved 2026-06-16)

- **Default pre-fill = last actual purchase cost.** Pre-fill the cost the owner
  last paid the provider for that store + product (most recent `kind='purchase'`
  `unit_cost`), falling back to the catalog `purchase_price` only on the
  first-ever purchase. Tracks the real, varying provider price without trusting a
  possibly-stale catalog value (the catalog has no edit route today).
- **`unitCost` is required in the UI, optional on the wire.** The purchase
  dialog always shows it pre-filled and blocks submit on an invalid amount, so
  the operator confirms what was paid every time; the backend keeps it optional
  (defaulting as above) so the one-tap "Comprar y cargar" assign path and any
  other non-UI caller never break.
- **"Comprar y cargar" uses the default silently** (no extra prompt in the
  time-sensitive assign-and-deliver flow); the cost can be corrected afterward.
- **Corrections after the fact are supported** via a `PATCH` on the purchase
  transaction's `unit_cost` (operators mis-enter under pressure), **but frozen
  once accounted** — a closed registry covering the purchase's day blocks the
  edit (`409`) for everyone (user decision 2026-06-16: an accounted period is
  *final*, not merely snapshot-shielded). The accounted check is an injected port
  (accounting `isStorePeriodAccounted` → inventory, wired in `app.ts`), avoiding
  an inventory→accounting import. Reopening a closed period stays out of scope.
