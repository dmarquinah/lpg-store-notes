---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../inventory/inventory-foundation/index]]"
  - "[[../../inventory/provider-purchase-cost/index]]"
  - "[[../../customers/customers-crud/index]]"
  - "[[../../accounting/accounting-registry/index]]"
last-updated: 2026-07-09
---

# Spec: Provider Management (registry + pricing + dual debt tracking)

## Problem Statement

The purchase process is a blind spot. A provider purchase
(`POST /inventory/stores/:storeId/{tank,item}-purchases`) records fulls onto the
store's `location` holder with a per-line `unitCost`, but:

1. **There is no provider identity.** No `providers` table, column, or enum
   exists anywhere (confirmed across all module schemas). "Provider" is purely
   an aggregate *label* in the accounting egress breakdown
   (`providerTanks / providerItems / providerSurcharge`). The owner cannot say
   *who* supplied a purchase, cannot compare suppliers, and cannot manage a
   price per supplier.

2. **The empty-tank exchange is silently lossy.** The business runs a two-way
   exchange with providers exactly as with customers: buying N full tanks means
   handing back N empties (ideally). Today `recordTankPurchase` sets
   `emptyReturned = item.emptyReturned ?? min(qty, availEmpty)` and, when the
   store holds fewer empties than fulls bought, just returns fewer and books a
   monetary `purchase_surcharge`. **It records nothing about the empties we now
   owe the provider.** The current guard only stops returning *more* empties
   than are on hand (409) — it never surfaces the shortfall as a debt. The owner
   explicitly wants to trace "we owe provider X four 10 kg empties."

3. **There is no money debt.** Every purchase is implicitly treated as paid in
   full (the accounting registry books its value as egress). In reality the
   owner sometimes buys on credit and pays the provider partially or later.
   There is nowhere to record that we still owe the provider money, nor to log a
   partial payment.

The owner wants the purchase process to name the provider, carry a per-provider
price, and make both kinds of debt (money — including partial balances — and
empty tanks) visible and traceable, mirroring what already exists for customers.

## Proposed Solution

Build a **provider (supplier) module that mirrors `customers`**, and attribute
purchases to a provider — the supply-side reflection of the customer registry +
dual-debt model.

- **Provider registry.** A new `providers` module (backend) + `providers`
  frontend module: CRUD + name/phone search + soft delete, styled like the
  customers module. Int PK (referenced across purchases + debt tables, per the
  project's int-PK-for-shared-tables convention).

- **Per-provider price list.** A `provider_prices` table holds a unit price per
  `provider × product` (tank type or inventory item). This becomes the **first
  tier** of the purchase-cost default chain established by
  [[../../inventory/provider-purchase-cost/index]]: provider price →
  last actual purchase cost → catalog `purchase_price`. The catalog is still
  never mutated.

- **Purchases name a provider.** `tank_transactions` and `item_transactions`
  gain a nullable `provider_id` (set for `kind='purchase'`). The purchase
  requests gain `providerId`, **required in the UI, optional on the wire** — so
  the one-tap "Comprar y cargar" assign path and other non-UI callers don't
  break (same optional-on-wire pattern as `unitCost`). Legacy purchase rows keep
  `provider_id = NULL`.

- **Empty-tank debt to the provider.** A `provider_empty_debts` ledger mirrors
  `customer_empty_debts` (append-only signed `delta` per `tankTypeId`, balance
  view). On a tank purchase of N fulls returning M empties, the shortfall
  `N − M` accrues as empties **we owe the provider** (positive delta); a later
  purchase returning more empties than fulls settles prior debt (negative delta,
  capped at owed — mirroring `recordSale`/`recordReturn`). **This never blocks
  the purchase** (owner's decision): the shortfall becomes a tracked, surfaced
  debt instead of a silent surcharge. The existing physical guard (can't return
  more empties than the store holds → 409) stays.

- **Money debt to the provider, with partial payments.** A `provider_debts`
  (charges) ledger records the amount owed for each purchase (line cost +
  surcharge); a `provider_payments` ledger records payments against a provider,
  **partial amounts allowed** (mirroring `order_payments`). Money owed =
  Σ charges − Σ payments. A purchase may carry an optional `amountPaid` that
  logs an immediate payment; the remainder is outstanding. A standalone
  "record payment" action settles debt later.

- **Visibility.** The provider detail shows both balances (money outstanding +
  empties owed per type); the provider list shows debt badges (mirroring the
  customer list). The purchase dialogs show a clear "quedaremos debiendo N
  vacíos a este proveedor" note when empties fall short.

- **Accounting reflects cash, debt stays explicit.** The accounting registry's
  egress switches to **cash actually paid to providers** (`provider_payments` in
  the period) instead of purchase value. To keep the period balanced and
  legible, the registry also shows two reconciling lines: **Compras recibidas**
  (value of goods received, A) and **Deuda a proveedores** (running balance /
  period delta, C), so `pagos (B) + Δdeuda (C) = compras (A)`. The owner/admin
  sees exactly what the location did each period — bought, paid, still owes —
  with no magic number. This deliberately modifies the done `accounting-registry`.

Detailed backend design in [[backend]]; the provider UI + purchase-dialog
changes in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Providers registry & pricing (backend + frontend):**

- [ ] A `providers` table (int PK; name, phone?, notes?, active, timestamps;
      unique active name) with a `providers` backend module (routes/service/
      repository/schema/types/index factory per the module template) exposing
      CRUD + name/phone search + soft delete. Admin-gated writes; read scoping
      consistent with the customers module. Migration named (continues from
      **0014**).
- [ ] A `provider_prices` table holds a unit price per `provider × product`
      (`productKind` tank|item + `productId`, `unitPrice numeric(10,2) ≥ 0`,
      active, timestamps; one active price per provider+product). CRUD to manage
      it (admin-gated). **On provider creation, seed a price row for every active
      catalog product (tank types + items) defaulting to the catalog
      `purchase_price`**; the admin edits per-provider afterward.
- [ ] Frontend `providers` module (mirrors `customers`): list (search + money &
      empties debt badges), create/edit form, detail (debt summary + price list
      + record-payment). Uses the Piloto design system + `formatMoney`.

**Purchase attribution & pricing default (backend + frontend):**

- [ ] `tank_transactions` and `item_transactions` gain a **nullable**
      `provider_id` FK (set only for `kind='purchase'`; existing rows stay
      NULL, no backfill).
- [ ] `tankPurchaseSchema` / `itemPurchaseSchema` gain `providerId` — **required
      in the UI, optional on the wire**. `recordTankPurchase` /
      `recordItemPurchase` persist it onto the `kind='purchase'` transaction.
- [ ] The purchase `unitCost` default chain gains a top tier: **provider price**
      (`provider_prices` for provider+product) → last actual purchase cost →
      catalog `purchase_price`. The catalog is not mutated. The purchase dialog
      pre-fills from the selected provider's price.
- [ ] The "Comprar y cargar" assign path and any non-UI caller keep working with
      no `providerId` (nullable on the wire; behaviour otherwise unchanged).

**Empty-tank debt (backend + frontend):**

- [ ] A `provider_empty_debts` ledger (mirror of `customer_empty_debts`:
      append-only signed `delta` per `providerId × tankTypeId`,
      `refTankTransactionId`) + a balance view (`empties_owed = SUM(delta)`).
- [ ] On a tank purchase of N fulls returning M empties for a named provider,
      the shortfall `N − M` accrues a positive `provider_empty_debts` delta (we
      owe empties); returning more empties than fulls settles prior debt
      (negative delta, capped at owed). No `providerId` → no empty-debt row
      (legacy/anonymous purchase unchanged).
- [ ] The purchase **never blocks** on an empty shortfall (owner's decision);
      the existing 409 for returning more empties than physically on hand
      stays. The UI shows the resulting "quedaremos debiendo N vacíos" clearly.
- [ ] A read endpoint exposes a provider's empties-owed per tank type (mirrors
      the customer empty-debt read); shown on the provider detail.
- [ ] A shortfall accrues a **tracked empty-tank debt by default (no monetary
      surcharge)**; the existing `purchase_surcharge` path stays **available**
      for providers billed in cash for missing empties (operator opts in).

**Money debt & partial payments (backend + frontend):**

- [ ] A `provider_debts` (charges) ledger + a `provider_payments` ledger
      (partial amounts allowed; `method`, `note`, `recordedBy`, `occurredAt`)
      such that money owed = Σ charges − Σ payments (balance view/read).
- [ ] A tank/item purchase for a named provider records a charge (line cost +
      surcharge); an optional per-purchase `amountPaid` logs an immediate
      payment, leaving the remainder outstanding.
- [ ] A standalone **record-payment** endpoint (`POST /providers/:id/payments`,
      role-guarded) settles provider money debt after the fact; partials
      allowed; the balance reflects it live. Frontend surfaces this on the
      provider detail.
- [ ] The provider detail (and list badges) show money outstanding + empties
      owed; a provider with no debt shows a clean zero-state.

**Accounting integration (backend + frontend) — modifies the done `accounting-registry`:**

- [ ] The registry **egress** becomes **cash actually paid to providers** for
      the period (sum of `provider_payments` by payment business date),
      replacing the purchase-value egress. Legacy purchases with no provider /
      payment keep valuing at purchase value (backward compatible via a
      NULL-provider branch).
- [ ] The registry detail shows two reconciling lines — **Compras recibidas**
      (value of goods received) and **Deuda a proveedores** (running balance
      and/or period delta) — such that `pagos + Δdeuda = compras` holds and is
      shown as an explicit check.
- [ ] Closed registries **freeze** these figures in the snapshot exactly like
      existing egress (no live recompute after close);
      `registry-source-drilldown` line reads stay consistent.
- [ ] Accounting reads provider-payment totals via an **injected
      accounting→providers port** (ADR-012), not a direct import. Existing
      accounting tests are **updated** (not merely kept green) to assert the
      cash-basis egress + the reconciliation identity.

**Cross-cutting:**

- [ ] Lifecycle tests (per the project's lifecycle-test convention): a
      credit purchase → empty shortfall accrues → partial payment → later empty
      return settles the tank debt → later money payment clears the balance,
      each step asserted end-to-end. Existing inventory/accounting tests stay
      green.
- [ ] Quality gate green in **both** repos: backend `npm run typecheck` +
      `npm run check` + `npm test` + `npm run build`; frontend typecheck +
      build.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Backfilling `provider_id` / debt for historical purchases.** Existing rows
  stay `NULL`; no retroactive provider attribution or debt reconstruction.
- **True accounts-payable / aging reports.** Simple current balances (money
  outstanding, empties owed) only — no aging buckets, no due dates, no
  statements.
- **Bot involvement.** No provider surface in lpg-bot.
- **Per-provider store restrictions.** Any provider can supply any store; no
  provider↔store assignment model.

## Resolved Decisions (2026-07-08)

- **Accounting egress = cash paid, debt explicit.** Egress reflects actual
  `provider_payments` in the period; the registry also shows **Compras
  recibidas** (goods received) and **Deuda a proveedores** (balance/delta) as
  reconciling lines so `pagos + Δdeuda = compras`. Not a magic number — the
  owner sees bought/paid/owed per period. Deliberately modifies the done
  `accounting-registry`.
- **Payments: inline + standalone.** The purchase dialog carries an optional
  `amountPaid` (fast path, defaults to full for cash); a standalone
  `POST /providers/:id/payments` settles debt later. Both net against the
  balance; partials allowed.
- **"Comprar y cargar" leaves `provider_id` NULL.** The one-tap assign path is
  unchanged (no provider, no debt); revisit only if the owner later wants those
  traced.
- **Provider price list seeded from catalog, then admin-edited.** On provider
  creation, seed a `provider_prices` row per active catalog product defaulting to
  the catalog `purchase_price`; the admin adjusts per provider. At purchase the
  cost pre-fills from the selected provider's price (fallbacks: last-cost →
  catalog for any product missing a row). Never a precondition to buy.
- **Empty shortfall = tracked tank debt by default; surcharge available.** A
  short return accrues an empty-tank debt (no money charge); the monetary
  `purchase_surcharge` stays available for providers billed in cash for missing
  empties.
# Resolved Decisions (2026-07-08)
- **Store purchase-detail grouping (2026-07-09, added post-`done`).** Purchases
  now carry an explicit **`purchase_id`** (migration 0016) shared by every line +
  the inline payment, so the store `Compras` view (`/inventario/tiendas/:id`)
  groups a multi-product purchase and shows **per-purchase provider, paid vs
  owed, and empties owed** — the store-scoped detail the provider balance page
  doesn't give. Implemented directly (both tracks); see [[backend]] / [[frontend]]
  Implementation Notes. Reassigning a provider to a provider-less purchase after
  the fact remains out of scope.