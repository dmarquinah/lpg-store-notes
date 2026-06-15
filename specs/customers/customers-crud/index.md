---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../auth/auth-foundation/index]]"
  - "[[../../inventory/inventory-foundation/index]]"
  - "[[../../stores/stores-and-catalog/index]]"
last-updated: 2026-06-08
---

# Spec: Customers CRUD (registry + debt visibility)

## Problem Statement

There is no `customers` table or module in v2. Two consequences:

1. **Inventory already depends on it.** `inventory-foundation` writes
   `customer_empty_debts` rows keyed by a **soft `customerId`** — a plain
   integer with **no FK**, deliberately left dangling until this module lands
   (see `src/modules/inventory/schema.ts`: "soft reference until the customers
   module lands"). Empty-tank debt is therefore recorded against customers that
   the system cannot list, name, or validate.
2. **Orders is blocked.** The next spec (`orders-foundation`) attaches each
   order to a customer and accumulates unpaid balances. It needs a registry to
   look up / create customers and a place to record monetary debt.

The daily workflow (`product/overview`) is phone-driven: *"operator takes a
customer call → enters customer name/phone, items, price"*. So the registry's
primary job is **fast lookup by name or phone**, plus **at-a-glance debt
visibility** so the operator knows on the call whether a customer owes empties
or money.

v1 over-modelled this table (`customerType` enum, `rating`, geolocation
lat/lng/accuracy, `totalOrders`, `lastOrderDate`, `preferredPaymentMethod`) —
see [[../../../eng/legacy-bloat-analysis]]. v2 keeps only what traceability and
order entry need.

## Proposed Solution

A `customers` vertical module on the backend plus a `customers` frontend module,
mirroring the established `users`/`catalog` slices.

- **`customers` table** — `name`, unique `phone`, required `address`, optional
  `locationReference` (delivery hint, common in small cities) and `notes`,
  `active` soft-delete flag, timestamps.
- **`customer_debts` table (NEW, monetary)** — append-style per-debt rows
  (`customerId`, soft `orderId?`, `amount`, `description?`, `isPaid`, `paidAt?`).
  A `customer_debt_balance` view sums unpaid amounts per customer. **This spec
  creates the table + read path only; `orders` is the future writer, so the
  monetary balance reads 0 until orders ships.** (Trade-off accepted to keep the
  schema + read API ready for orders — see Open Questions.)
- **FK hardening** — now that `customers` exists, a migration adds the FK
  `customer_empty_debts.customer_id → customers.id`, closing the soft reference.
- **Empty-tank debt** — reuse the existing `customer_empty_debt_balance` view
  (ADR-010) to surface per-tank-type owed counts + a total per customer.
- **Read API** — list with `?search=` (name OR phone) + `?all=1`, detail with
  full debt breakdown, create (409 on duplicate phone), partial PATCH incl.
  `active`. Roles: **operator + admin** (developer bypass).
- **Frontend** — list (search + show-inactive + debt-flag badges), create/edit
  form (client validation mirroring backend Zod), detail with debt summary.

Detailed schema, routes, and file lists live in [[backend]] and [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `customers` table exists: `id` serial PK, `name` notNull, `phone` notNull **unique**, `address` notNull, `locationReference?`, `notes?`, `active` bool default true, `createdAt`/`updatedAt`. Drizzle migration in `src/db/migrations/` (no `db:push`); re-exported from `src/db/schema.ts`.
- [ ] `customer_debts` table exists: `id` serial PK, `customerId` int FK → `customers.id` notNull, `orderId` int? (soft ref — no FK; orders not built), `amount` numeric(10,2) notNull (`> 0` check), `description?`, `isPaid` bool default false, `paidAt?`, timestamps. Index `(customerId, isPaid)`. Append/update only via the module repo (orders will write later).
- [ ] A `customer_debt_balance` view returns unpaid monetary total per customer (0 / no-row tolerated for customers with none).
- [ ] Migration adds FK constraint `customer_empty_debts.customer_id → customers.id` (hardens the prior soft reference). Existing `idx_customer_empty_debts_cust_type` retained.
- [ ] `src/modules/customers/` follows [[../../../eng/patterns/module-template]]; only `repository.ts` imports `schema.ts`; types are Zod + `z.infer<>` (no `dtos/`, no `I*` interfaces, no DI container). Mounted via `createCustomersModule(...)` in `src/app.ts`.
- [ ] `GET /api/v1/customers` — Zod-validated list; `?search=` matches name OR phone (substring, min 2 chars; <2 → 400); `?all=1` includes inactive (active-only default); basic `?limit`/`?offset` pagination. Each row carries enough to show a debt flag (`outstandingBalance` monetary + `emptyTanksOwed` total).
- [ ] `GET /api/v1/customers/:id` — detail incl. debt breakdown: monetary `outstandingBalance` + the unpaid `customer_debts` rows, and empty-tank debt grouped by tank type (from `customer_empty_debt_balance`). 404 for unknown id.
- [ ] `POST /api/v1/customers` — creates from `{name, phone, address, locationReference?, notes?}`; duplicate phone → 409 (ConflictError). Returns the created customer.
- [ ] `PATCH /api/v1/customers/:id` — partial update of any editable field incl. `active` (soft delete/restore). Phone change to an existing phone → 409.
- [ ] All customer routes guarded to **operator + admin** (developer bypass), reusing `src/modules/auth/middleware.ts`. Errors flow through `src/lib/errors.ts`.
- [ ] At least 2 lifecycle tests (per [[../../../eng/decisions]] test guidance): create → list/search by name and phone; duplicate-phone conflict; detail returns empty-tank debt for a customer with a `customer_empty_debts` balance.

**Frontend (lpg-frontend-vue):**

- [ ] `src/modules/customers/` vertical module (`types`/`service`/`store`/`routes`/`index` + `views/` + `components/`) mirroring the `users` module, wired in `src/main.ts` and the router.
- [ ] List view: search box (name/phone, debounced), "Mostrar inactivos" toggle (`?all=1`), and a per-row debt flag (badge when the customer owes empties and/or money).
- [ ] Create/edit form: `name`, `phone`, `address`, `locationReference?`, `notes?`, `active` — with client-side validation mirroring the backend Zod schema; duplicate-phone 409 surfaced as a field error.
- [ ] Customer detail: shows the debt summary — empty-tank debt by tank type (with type names from `catalog`) + monetary outstanding balance.
- [ ] Routes guarded `roles: [operator, admin, developer]`; a "Clientes" nav entry added to the operator and admin nav in `AppLayout`'s `ROLE_NAV`.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

> **Porting order:** backend track first (it owns the schema + API the frontend
> reads); the frontend track ports right after.

## Out of Scope

- **Inline quick-create during order entry** — the `orders` spec owns the
  inline "new customer on the call" flow; this spec ships the standalone
  registry it reuses.
- **Monetary-debt writers** — `orders` writes `customer_debts` (on delivery /
  payment). This spec only creates the table + read path; the balance reads 0
  until then.
- **Debt settlement / payment-recording UI** — recording a payment against a
  debt belongs to orders/payments, not the registry.
- **v1 fields** — `customerType`, `rating`, geolocation (lat/lng/accuracy),
  `totalOrders`, `lastOrderDate`, `preferredPaymentMethod`, `alternativePhone`.
  Add back only when a concrete consumer needs them.
- **Customer-facing order history** — handled by the orders module.

## Open Questions

- **`customer_debts` with no writer (resolved 2026-06-07):** create the table +
  read path now so orders can wire straight in; the monetary balance reads 0
  until orders writes rows. Accepted over deferring the table entirely.
- **`active` vs `isActive` column naming:** match the prevailing v2 convention
  (`catalog` uses `active`). Confirm in the backend track plan.
- **Pagination depth:** basic `limit`/`offset` is enough for a single small-city
  customer base; revisit if the list grows.
