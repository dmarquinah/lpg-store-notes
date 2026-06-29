---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../store-stock-first/index]]"
last-updated: 2026-06-28
---

# Spec: Store Stock Adjustments (set a store's standing full/empty stock directly)

## Problem Statement

The business needs to **start from an existing state**: when onboarding a store
(or correcting a miscount) an admin must be able to say *"this store currently
holds N full and M empty tanks of type X"* in **one step**. Today that's
impossible.

The only way to get tanks onto a store's `location` holder is a **provider
purchase** (`POST /inventory/stores/:storeId/tank-purchases`), which models a
supplier delivery (full-for-empty swap, a cost-in, accounting egress) — wrong
semantics for "this is just the opening count we already have on the floor". The
only other route to change a store's standing stock is to **abuse the
driver-day flow**: open a day → record an `adjustment` on the *truck* → close →
consolidate (`carry`) back to the location. That's **4+ steps across two
roles**, it invents a phantom assignment, and — in the owner's words — it
"becomes too confusing for the admin".

There is **no location-level adjustment** capability at all: the existing
`adjustment` transaction kind only has an endpoint scoped to an **assignment**
(`POST /inventory/assignments/:id/adjustments`), never to a store location.

Compounding this, every such correction must be **attributable and timestamped**
— who set/changed the stock, and when — which is exactly why this spec is paired
with the [[../store-history/index|store-history]] book (the ledger already
records `user_id` + `occurred_at`; the history surfaces it).

## Proposed Solution

Add a **direct location-stock adjustment** capability: an admin (or own-store
operator) sets/corrects a store's standing **full and empty** counts per tank
type in a single action, writing an `adjustment` ledger row on the store's
`location` holder — no driver, no day, no purchase semantics.

- **Backend — new location-adjustment endpoint.** A `POST
  /inventory/stores/:storeId/adjustments` that, per tank type, writes an
  `adjustment`-kind `tank_transaction` directly on the store's `location` holder
  (`findOrCreateTankHolder({ kind: 'location', storeId })`). The request carries
  **deltas** (`fullDelta`, `emptyDelta`, signed) plus a **required note/reason**;
  the actor (`userId`) and `occurred_at` are captured by the existing ledger
  insert. Append-only (ADR-009) — a correction is a new `adjustment` row, never an
  in-place mutation of a balance. One request can seed **multiple tank types at
  once**, so a brand-new store goes from nothing to its full opening stock in a
  single call.
- **"Set to" is sugar over deltas.** The endpoint takes deltas to keep the ledger
  honest; the frontend lets the user enter either an **absolute target** ("set to
  12 full / 8 empty") — the client computes `delta = target − current` — or a
  **± adjustment**, and sends the resulting delta. Reaching an absolute count and
  nudging by a few are the same ledger primitive.
- **Frontend — "Ajustar stock" on the store-stock surface.** On the store-stock
  view from [[../store-stock-first/index|store-stock-first]], an **"Ajustar
  stock"** affordance opens a dialog pre-filled with the current per-type
  full/empty; the user edits the numbers (absolute) or applies ±, gives a
  **reason**, and saves → POST → refetch availability so the new counts show
  immediately. Each adjustment lands in the store history.
- **Guardrails.** Integer deltas; an adjustment may not drive a balance
  **negative** (resulting full/empty `< 0` → 409/422); the note is required so the
  history reads meaningfully; scoping mirrors provider purchases (operator
  own-store + admin/dev global; cross-store → 403). **No accounting egress** — an
  adjustment is not a purchase and carries no cost.

No schema change (the `adjustment` kind and `location` holder already exist).
Backend contract in [[backend]]; dialog + surface in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Backend (lpg-backend):**

- [x] A new endpoint `POST /inventory/stores/:storeId/adjustments` writes, per
      tank type in the request, an **`adjustment`-kind** `tank_transaction` on the
      store's **`location`** holder (created via `findOrCreateTankHolder` if
      absent), applying signed `fullDelta` / `emptyDelta`. The request carries a
      **required note/reason**; the inserted rows capture the **actor (`userId`)**
      and **`occurred_at`** via the existing ledger insert.
- [x] One request can adjust **multiple tank types atomically** (single
      `db.transaction`), so a store with **no holders** can be seeded to a full
      opening stock (several types, full + empty) in **one call** — no
      driver-day, no purchase.
- [x] **No negative balances:** if applying a delta would make a holder's
      resulting full or empty balance `< 0`, the whole request is rejected
      (`ConflictError`/`ValidationError`, 409/422) and nothing is written.
- [x] **Validation:** `fullDelta`/`emptyDelta` are integers (at least one
      non-zero per line), `tankTypeId` is a valid active catalog type, and the
      note is non-empty. Zero-effect lines are rejected or ignored
      (decide at `/focus`).
- [x] **Scoping** mirrors provider purchases: operator may adjust **own-store**
      only, admin/developer any store; a cross-store `:storeId` → **403**. No
      assignment / driver-day holder is touched.
- [x] **No accounting impact:** an `adjustment` carries no `unit_cost` and does
      **not** appear in the registry egress (only `kind='purchase'` does). Verified
      by test.
- [x] Append-only: corrections are new `adjustment` rows (no balance mutated in
      place). New tests (seed-from-zero, multi-type, negative-balance reject,
      cross-store 403, no-egress); existing tests stay green; typecheck / lint /
      build green.

**Frontend (lpg-frontend-vue):**

- [x] The store-stock surface (from
      [[../store-stock-first/index|store-stock-first]]) gains an **"Ajustar
      stock"** affordance, visible to admin/developer and own-store operators.
- [x] The dialog pre-fills the **current** per-type full/empty and lets the user
      either set an **absolute target** or apply a **± delta** per type, with a
      **required reason**; on save it sends the computed **deltas** to `POST
      /inventory/stores/:storeId/adjustments` and **refetches availability** so the
      new counts show immediately.
- [x] Client validation mirrors the backend (integers; a target/delta that would
      go **negative** is blocked before submit); backend **403 / 409** are surfaced
      as clear messages (cross-store / would-go-negative).
- [x] A brand-new store (empty "Stock en tienda") can be brought to its opening
      stock entirely from this dialog (no purchase, no driver-day).
- [x] Design-system compliant (tokens, `formatMoney` not needed for counts, ≥44px
      targets); `npm run typecheck` + `npm run build` green. Manual smoke
      (seed a store from 0 → counts appear → adjustment shows in history) left to
      the operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Items (non-tank inventory).** This spec is tank full/empty standing stock.
  Item-holder adjustments can follow the same shape later if needed.
- **Empty-tank customer debt.** Location empties are stock, not customer debt;
  the `customer_empty_debts` ledger is untouched.
- **Accounting / cost capture.** An adjustment is explicitly *not* a purchase —
  no `unit_cost`, no egress. Restocks that *should* cost money keep using the
  provider-purchase flow ([[../provider-purchase-cost/index]]).
- **Editing/voiding past adjustments.** Append-only — a wrong adjustment is fixed
  by a compensating adjustment, not an edit. (Mirrors the ledger's stance.)
- **The history surface itself** — see [[../store-history/index|store-history]];
  this spec only *writes* the attributable rows it will display.

## Open Questions

- **Who may adjust — admin-only, or own-store operators too?** Proposed: mirror
  provider-purchase scoping (operator own-store + admin/dev) since operators
  already record purchases that change the same holder. Tighten to admin/dev-only
  if the owner wants seeding to be an admin-only act. (Confirm at `/focus`.)
- **Reason as free text vs. preset categories** (e.g. "Carga inicial",
  "Conteo físico", "Merma/daño"). Proposed: a small preset list + "Otro" free
  text, mirroring the accounting `ManualEntryDialog` pattern — reads better in the
  history than free text alone.
- **Dedicated `kind` vs. reuse `adjustment`.** Proposed: reuse the existing
  `adjustment` kind (no schema/enum change). A distinct `opening`/`seed` kind
  would read more clearly in the history but adds an enum value; revisit only if
  the history can't disambiguate seed-from-correction from the note.
