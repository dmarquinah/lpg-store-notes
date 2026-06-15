---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: approved
depends-on:
  - "[[../orders-foundation/index]]"
  - "[[../orders-multi-location/index]]"
  - "[[../../customers/customers-crud/index]]"
last-updated: 2026-06-14
---

# Spec: Orders Queue — Active/History Split, Date-Range History & Accent-Insensitive Search

## Problem Statement

`/pedidos` is a single flat table of every order, all statuses. It works today
with a handful of orders, but it conflates two jobs with opposite growth
behaviour, so it degrades as the business runs:

1. **The live queue** (`pending` / `assigned` / `in_transit`) — the "what needs
   action right now" view. It's naturally bounded: orders flow *out* of it as
   they're delivered. This is what operators watch all day.
2. **History** (`delivered` / `cancelled` / `failed`) — grows without bound. You
   never want to scroll it; you want to *look something up* by date or customer.

Mixed in one ever-growing table, the active queue gets buried under finished
orders, and finding a past order means scrolling forever. There's also a
**search bug** that bites today: the customer search (used in order creation and
the customers list) matches case-insensitively but **not accent-insensitively** —
typing "Maria" never finds "María", which is the common case for Peruvian names.

## Proposed Solution

Reshape the queue around the two jobs, and fix search end-to-end.

- **Active / History split.** `/pedidos` gets a segmented control: **Activos**
  (default — the open queue, no date filter, naturally short) and **Historial**
  (finished orders, retrieved by date range). The status dropdown still narrows
  within a mode; the admin store switcher + branch column (from
  [[../orders-multi-location/index|orders-multi-location]]) are preserved.
- **Date-range history.** Historial uses the shared `DatePicker` for a Desde/Hasta
  range wired to the backend's existing `dateFrom`/`dateTo`. A narrow range keeps
  the result set small, so **no pager** for now (deferred — see Out of Scope).
- **Accent-insensitive customer search.** Fix the customers-module search to match
  with Postgres `unaccent()` on both sides (so "Maria" ⇄ "María"), fixing the
  order-creation picker and the customers list. Then add a free-text `search`
  param to `GET /orders` (customer name/phone, registered via join + walk-in
  snapshot) so the queue has a type-a-name box powered by the same matching.

Most of the backend already exists — `GET /orders` already accepts `status`,
`dateFrom`, `dateTo`, `storeId`, `limit`, `offset`; the frontend just doesn't
expose date/search yet. The only new backend work is the `search` param + the
`unaccent` fix.

Detailed data model, endpoints, and file lists live in [[backend]] and
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] **Accent-insensitive customer search**: the customers repository matches
  name/phone with `unaccent(col) ILIKE unaccent(pattern)` (still
  case-insensitive, still substring, still min-2-char). "Maria" matches "María"
  and vice-versa. A Drizzle migration enables `CREATE EXTENSION IF NOT EXISTS
  unaccent` (idempotent). Fixes both the order-creation picker and the customers
  list (one code path).
- [ ] **`GET /orders` `search` param** (min 2 chars): matches the order's customer
  by name OR phone, accent-insensitive — covering both registered customers (via
  the existing `customers` join) and walk-in snapshots (`orders.customer_name` /
  `customer_phone`). Composes with `status`, `storeId`, `dateFrom`/`dateTo`, and
  the role/branch scope (no widening). Keep the single-query, no-N+1 shape.
- [ ] Behaviour preserved: existing case-insensitive matching still holds; the
  min-2-char rule still applies; all of `orders-foundation` /
  `orders-multi-location` behaviour and tests stay green. Reads keep flowing
  through `src/lib/errors.ts`; Lima dates via `src/lib/date.ts`.
- [ ] Tests: (a) accent-insensitive match both directions (`María`⇄`Maria`) on the
  customers search; (b) `GET /orders?search=` filters by registered + walk-in
  name/phone and composes with a status/date filter; (c) search intersects the
  branch scope (an operator's `search` never leaks another branch's orders).

**Frontend (lpg-frontend-vue):**

- [ ] `/pedidos` is split into **Activos** (default: `pending`+`assigned`+
  `in_transit`, no date filter) and **Historial** (`delivered`+`cancelled`+
  `failed`) via a segmented control / tabs.
- [ ] Historial exposes a **date-range** filter (Desde / Hasta) using the shared
  `DatePicker`, wired to `dateFrom`/`dateTo`, with a sensible default (today). No
  numbered pager (the range bounds results).
- [ ] A **customer search** box on the queue (debounced, ≥2 chars) → `GET
  /orders?search=`, accent-insensitive end-to-end.
- [ ] The **status** dropdown still narrows within the active mode; the admin
  **store switcher** + **branch column** (all-branches view) are preserved; the
  ownership/transfer affordances keep working.
- [ ] Reuses the shared `DatePicker`; consistent with `eng/design-system.md`. The
  order-creation customer picker now finds accented names (no FE change needed
  beyond verifying the backend fix flows through).

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | not-started |
| [[frontend]] | lpg-frontend-vue | frontend | not-started |

> **Porting order:** backend first (the `search` param + `unaccent` fix), frontend
> right after (it consumes both).

## Out of Scope

- **Numbered pagination / infinite scroll** — a date-bounded Historial keeps
  results small for now; revisit if a single day ever returns too many rows.
- **Analytics / aggregations / reporting** over orders (counts, revenue, etc.) —
  this is about retrieval, not reporting.
- **Saved filters / per-operator default views** — a later nicety.
- **Full-text / fuzzy search** (typo tolerance, ranking) — substring +
  accent-insensitive is the MVP; Postgres `pg_trgm` is a future enhancement.
- **A functional `unaccent` index** for search performance — `unaccent` isn't
  `IMMUTABLE`, so a direct expression index needs an immutable wrapper; skipped
  while the dataset is small (note it as a perf follow-up).

## Open Questions

- **Historial default range:** today (mirrors Inventario's familiarity) vs last 7
  days? (Lean: today, with the operator widening as needed.)
- **Activos date filter:** should Activos also allow an optional date filter, or
  stay deliberately date-less (it's "everything still open")? (Lean: date-less —
  an open order has no single "day".)
- **Range semantics:** filter Historial by `createdAt` (when the order was placed)
  or `updatedAt` (when it finished)? (Lean: `createdAt`, matching the existing
  `dateFrom`/`dateTo` columns; revisit if operators expect "delivered on" dates.)
