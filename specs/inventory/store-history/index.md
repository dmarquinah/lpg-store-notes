---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
last-updated: 2026-06-29
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../store-stock-first/index]]"
  - "[[../store-stock-adjustments/index]]"
---

# Spec: Store History (a per-store action history book)

## Problem Statement

There is **no way to see everything that happened in a store**. An admin (or a
second admin coming in later) cannot answer *"what's the history of this store —
who restocked it, who adjusted the count, what went out on deliveries, and
when?"*. The traceability the product is built for exists in the data but is
**not viewable per store**: actions are spread across append-only ledgers
(`tank_transactions`, `item_transactions`, purchases, openings/loads to drivers,
sales, returns, adjustments, carries) and the only history surfaces today are
**per-assignment** (a single driver-day's `/transactions`) or **per-customer**
(empty-debts) — never **per store**.

This becomes pressing the moment stores get **editable standing stock**
([[../store-stock-adjustments/index|store-stock-adjustments]]): every
hand-correction of a store's count must be auditable — *who* set/changed it and
*when*. The owner asked for exactly this: a **history book for a single store**,
so any admin can see all actions taking place there.

The good news: it's **all already recorded**. Each `tank_transactions` /
`item_transactions` row carries `kind`, `user_id` (who), `occurred_at` (when),
`notes`, and `ref_order_id` / `ref_customer_id`, and joins to a store via its
`location` holder. A per-store history is a **composition of existing ledgers** —
no new tables, no new writes.

## Proposed Solution

A **read-only, per-store history book**: a time-ordered stream of everything that
touched the store's `location` holder, with *who* and *when*, surfaced as a new
tab/section on the store-stock surface. Order-lifecycle events are **lazily**
fetched on demand so the base history load stays lean (the owner's explicit
ask).

- **Backend — composed history endpoint.** `GET
  /inventory/stores/:storeId/history?from=&to=` returns a unified, **newest-first**
  list merging the store's `location`-holder **tank** + **item** transactions.
  Each entry: `kind` (mappable to a label), `occurredAt`, `businessDate`,
  **actor** (`userId` + resolved **name**, joined from `users`),
  `fullDelta`/`emptyDelta` (tank) or `delta`/`qty` (item), `notes`, and
  `refOrderId` / `refCustomerId` when present. Composed from existing ledgers via
  the location-holder join — **no schema change, no new ledger**. Windowed by Lima
  business date with a sensible default (e.g. last 30 days).
- **Order events stay lazy + separate.** The base history is **inventory
  movements only**. Order-lifecycle events for the store (created / assigned /
  delivered / failed) are fetched by a **separate** call only when the user clicks
  a button — so the default request isn't polluted. Reuse the store-scoped
  `GET /orders?storeId=&from=&to=` (already exists) or a thin dedicated read; the
  frontend merges them in on demand.
- **Frontend — a "Movimientos / Historial" surface.** A new tab/section on the
  store-stock view ([[../store-stock-first/index|store-stock-first]] left the slot)
  renders the timeline: action label, actor, timestamp, the movement (±full /
  ±empty / qty), the note, and a link to the related order/customer when present.
  A **"Ver eventos de pedidos"** button lazily loads + merges the order events.
  Date-window control; operator own-store scoped, admin/dev global.

Backend contract in [[backend]]; the history surface in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Backend (lpg-backend):**

- [x] `GET /inventory/stores/:storeId/history?from=&to=` returns a **newest-first**
      unified stream of the store's **`location`-holder** `tank_transactions` +
      `item_transactions` over a Lima business-date window (sensible default if
      bounds omitted).
- [x] Each entry carries: `kind`, `occurredAt`, `businessDate`, **actor**
      (`userId` **and** resolved `userName` joined from `users`), the movement
      (`fullDelta`/`emptyDelta` for tanks, `delta`/`qty` for items), `notes`, and
      `refOrderId` / `refCustomerId` when set — enough for the UI to render a
      meaningful audit line without extra lookups.
- [x] Composed from **existing** ledgers via the location-holder join — **no
      schema change, no new ledger table, no new write path**.
- [x] **Order-lifecycle events are NOT included** in this response (kept lazy /
      separate per the owner's ask). If a dedicated store-orders-events read is
      added, it is a **separate** endpoint, not folded into `/history`.
- [x] **Scoping:** operator own-store only, admin/developer any store; cross-store
      `:storeId` → **403** (mirror the availability/purchase scoping).
- [x] Tests cover the merged ordering, actor-name resolution, the store/location
      join (an assignment-holder tx for another store does **not** leak in), and
      the date window; existing tests stay green; typecheck / lint / build green.

**Frontend (lpg-frontend-vue):**

- [x] A **"Movimientos / Historial"** tab/section on the store-stock surface
      renders the per-store timeline: a human action **label** (purchase /
      opening / load / sale / return / **adjustment** / carry / …), **actor name**,
      **timestamp**, the **movement** (±full / ±empty / qty), the **note**, and a
      **link** to the related order/customer when present.
- [x] The base history loads **inventory movements only**; a **"Ver eventos de
      pedidos"** button **lazily** fetches the store's order events on demand and
      merges/lists them — the default load issues **no** order request.
- [x] A **date-window** control scopes the history; operator own-store scoping and
      the selected-store context from the page are respected.
- [x] Adjustments made via
      [[../store-stock-adjustments/index|store-stock-adjustments]] appear here with
      their actor + reason, closing the "who changed the count, when" audit loop.
- [x] Design-system compliant (`ResponsiveTable` or a timeline pattern, tokens,
      `Spinner`, ≥44px, status→label mapping); `npm run typecheck` + `npm run
      build` green. Manual smoke (do a purchase + an adjustment + a delivery →
      all three appear with actor/time; order events only after the button) left to
      the operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **A new audit/event table or any new write path.** History is a read-only
  composition of the existing append-only ledgers; nothing new is recorded.
- **Order events in the base response.** They are lazily fetched on demand only —
  keeping the default history request lean is an explicit requirement.
- **Items adjustments / non-tank specifics** beyond surfacing existing
  `item_transactions` rows. The store-stock-adjustments spec is tank-only; item
  history still *displays* if item txns exist.
- **Cross-store / org-wide activity feed.** This is one store at a time, driven by
  the page's store picker. An org-wide feed is a separate, later idea.
- **Exports / printing / retention policy.** A viewable book in-app; CSV/PDF
  export is not in this cut.

## Open Questions

- **Timeline vs. table.** Proposed: a `ResponsiveTable` (date · acción · por ·
  movimiento · nota) for scan-ability and phone stacking, not a vertical timeline
  — but the order-event merge may read better as a timeline. Decide at `/focus`.
- **Default window length.** Proposed last 30 days with Desde/Hasta override
  (consistent with the Compras al proveedor window); confirm.
- **Order-event source.** Reuse `GET /orders?storeId=&from=&to=` (exists) vs. a
  thin dedicated `GET /inventory/stores/:id/order-events`. Proposed: reuse the
  orders read first; add a dedicated endpoint only if the shape is awkward.
- **Pagination.** A busy store could have many rows in 30 days; proposed
  date-window + newest-first is enough for the MVP, add paging only if needed.
