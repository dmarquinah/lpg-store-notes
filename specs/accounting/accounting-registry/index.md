---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../orders/orders-foundation/index]]"
  - "[[../../inventory/inventory-foundation/index]]"
  - "[[../../stores/stores-and-catalog/index]]"
  - "[[../../orders/orders-multi-location/index]]"
last-updated: 2026-06-15
---

# Spec: Accounting Registry (per-store closing register for ingress/egress)

## Problem Statement

The system records the *operational* chain end-to-end with full traceability — orders → delivery → payments, and inventory → provider purchases — but there is **nowhere the money is reconciled and closed**. An admin cannot answer today: *"for this store over this week, how much came in (by payment method), how much went out (provider purchases + expenses), and what's the net?"*

The money events already exist, but each lives inside its own module's ledger, scoped to that module's concerns:

- **Ingress** is in `order_payments` (`amount`, `method` ∈ cash/yape/plin/transfer, `occurred_at`, `recorded_by`), accrued at/after delivery of an order.
- **Egress** is in the inventory ledger as provider purchases — `tank_transactions`/`item_transactions` of `kind='purchase'`, valued at the holder's snapshot `purchase_price × qty`, plus the `purchase_surcharges` side table for the empty-shortfall.

There is no concept of **grouping a period's money events for a branch into a single reviewable record that can be frozen**. And real businesses also have money movements the operational chain never captures — fuel, salaries, rent, repairs, the occasional refund or other income — with no place to register them.

The product's stated next step is **weekly/monthly financial evaluation**. That evaluation needs a foundation first: an explicit, per-store register that groups a window's ingress and egress (operational + manual) and closes into a snapshot. This spec builds that foundation. The periodic reports/dashboards that read across closed registries are explicitly **later work**.

## Proposed Solution

A new `accounting` vertical module introducing a **per-store accounting registry** — an explicit, openable/closable financial record over a date window. It does **not** re-record money; it **composes** the existing orders + inventory read surfaces (ADR-012, explicit service composition, no duplication) and adds manual entries + a close/snapshot lifecycle on top.

**The register (`accounting_registries`)** — one row per store per period: `storeId`, `periodStart`/`periodEnd` (Lima business dates via `src/lib/date.ts`), `status` (`open` → `closed`), `openedBy`/`closedBy`/`closedAt`, optional `notes`. Registries for one store **cannot overlap in time** — so every money event belongs to at most one register, and "group a list of fulfilled orders into a single registry" falls out of the non-overlapping window (membership keyed by the *money event's* business date — for ingress, the **payment's** date, since an order can fail right before delivery).

**Auto-aggregated content** — a registry's detail derives, for its store + window:
- **Ingress** — `order_payments` of delivered orders for that store, grouped by `method` (cash / yape / plin / transfer).
- **Egress** — provider purchases (`kind='purchase'` tank + item transactions on the store's `location` holder, valued at `purchase_price × qty`) plus `purchase_surcharges`.

**Manual entries (`accounting_entries`)** — free-form typed lines attached to a registry, covering "register *any* ingress/egress": `direction` (`ingress`/`egress`), `amount`, `category` (fuel / salary / rent / repair / other-income / …), optional `method`, `note`, `occurredAt`, `recordedBy`. Addable/removable only while the register is `open`.

**Close = freeze.** Closing snapshots the auto-aggregated totals into the record so later edits to underlying payments/purchases never change a closed period's books. A closed register is read-only (no new/edited entries, no re-aggregation).

**Scope & permissions** — per-store via `store_assignments` / `storeIdsForUser`: admin/developer global; an authorized scoped user sees only their own store(s); cross-store access denied. (Exact non-admin role gating → Open Questions.)

Detailed backend design lives in [[backend]]; the admin registry screens in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] A new `accounting` vertical module (`src/modules/accounting/{routes,service,repository,schema,types,index}.ts`) follows the module template and is composed in `src/app.ts`; cross-module reads (payments, purchases) go through **explicit service composition** (ADR-012), not duplicated queries.
- [ ] `accounting_registries` table: `storeId` (FK), `periodStart`/`periodEnd` (Lima business dates), `status` (`open|closed`), `openedBy`/`closedBy` (FK users), `closedAt`, `notes?`. Opening a register whose window **overlaps** an existing register for the same store → 409.
- [ ] `accounting_entries` table: manual lines on a registry — `registryId` (FK), `direction` (`ingress|egress`), `amount` (`numeric(10,2)`, > 0), `category`, `method?`, `note?`, `occurredAt`, `recordedBy` (FK). Insert/delete allowed **only while the registry is `open`** (4xx otherwise).
- [ ] A registry detail returns, for its store + window: **ingress** = sum of `order_payments` of `delivered` orders **grouped by `method`**; **egress** = provider purchases (`kind='purchase'` valued at snapshot `purchase_price × qty`) + `purchase_surcharges`; **plus** the manual entries; and a computed **net**.
- [ ] **Closing freezes a snapshot**: on `close`, the auto-aggregated totals are persisted onto the register so subsequent changes to underlying payments/purchases do **not** alter the closed period's reported figures; the detail of a `closed` register serves the frozen snapshot.
- [ ] **Per-store scoping & role split**: admin/developer manage every store's registries; **operators** open registers + add/remove manual entries + view detail for their own `storeIdsForUser` store(s) only; **closing is admin/developer only**. A registry/entry for a store outside the caller's scope → 404/403 (never a widening grant; a tampered `storeId` intersects to the caller's scope).
- [ ] API covers the lifecycle: open a register (store + window), list (by store + status), get detail (with the ingress/egress/manual/net breakdown), add + remove a manual entry (while open), and close. Every route is role-guarded.
- [ ] Migration `0011_*` (named, applied to the dev DB). Lifecycle tests: open → a delivered-order payment + a provider purchase land in the breakdown → add manual ingress + egress → close → snapshot frozen (a later payment doesn't move the closed totals) + further entry writes rejected; overlap → 409; scoping → 403/404. Existing tests stay green; typecheck / lint / build green.

**Frontend (lpg-frontend-vue):**

- [ ] An `accounting` module (admin + authorized roles, new nav entry) with a **per-store registry list** (open / closed) and a **"Nuevo registro"** action to open a period for a store (start/end Lima dates, mirroring the backend overlap rule's error).
- [ ] **Registry detail** shows the **ingress breakdown by payment method**, the **egress breakdown** (provider purchases + manual), the **manual-entry add/remove** UI (only while `open`), and the **net** — all via `formatMoney` and the petrol+flame design system.
- [ ] A **"Cerrar registro"** action (admin) freezes the period; a `closed` register renders **read-only** against its frozen snapshot (no add/remove, clear closed state).
- [ ] Store scoping mirrors the backend: admin/developer get a store switcher across all stores; a scoped user sees only their own store(s).
- [ ] typecheck + build green; manual smoke (open → entries land → add manual → close → read-only) left to the operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Weekly/monthly cross-period reports / dashboards / charts** — the stated "posterior work". This spec delivers only the per-period register those will read; no aggregation across registries, no trend views.
- **Org-wide consolidated registry** — per-store only now; rolling up all stores into one set of books is a later concern (the per-store snapshots are the inputs).
- **Profit / margin / true COGS accounting** — egress is *cash out* (what was paid to the provider + manual expenses), not cost-of-goods-sold matched against revenue. No inventory valuation, depreciation, or accruals.
- **Tax, invoicing (boletas/facturas), or accounting-software export.**
- **Editing or reopening a `closed` register** — closed is final for this spec (admin reopen could be a later override).
- **Customer debt / receivables as a balance-sheet item** — `customer_debts` stays in the customers module; this register is cash ingress/egress, not A/R.

## Open Questions (resolved 2026-06-15)

- **Membership = non-overlapping window.** A money event belongs to the register whose store + `[periodStart, periodEnd]` contains its business date; **registers for a store may never overlap in time** (→ 409). No link table, no per-order selection. *Manually excluding individual orders from a register was considered and deferred* — it would require allowing overlapping/partial ranges, which has no current justification. Keep it simple: the window defines the group.
- **Ingress membership is keyed by the payment's business date**, not the order's delivery date — an order can fail right before delivery, and cash-in timing is what the books should reflect. (`order_payments.occurred_at` → Lima business date.)
- **Permission split.** Admin/developer manage every store's registers globally. **Operators** may open a register and add/remove **manual entries** for **their own store(s)** (`storeIdsForUser`), and view that store's detail. **Closing a register is admin/developer only** (the operator records, the admin finalizes — same record-then-confirm split as the day-handoff). Cross-store access → 404/403.
- **Snapshot storage** — denormalized totals columns vs a JSON snapshot column on `accounting_registries`, or a side table. Functionally equivalent; decide at implementation.
- **Egress value = total cash paid to the provider** = `purchase_price × qty` + `purchase_surcharges.amount`.