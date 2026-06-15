---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../orders-foundation/index]]"
  - "[[../../stores/stores-and-catalog/index]]"
  - "[[../../inventory/inventory-foundation/index]]"
  - "[[../../customers/customers-crud/index]]"
last-updated: 2026-06-14
---

# Spec: Orders Multi-Location, Ownership & Concurrency

## Problem Statement

`orders-foundation` shipped the money-recording workflow, but it has three gaps
that surface as soon as more than one operator — or more than one store — is in
play:

1. **An order has no location.** `orders` carries no `storeId`; an order's branch
   is *emergent*, acquired only when `assignmentId` is set at `assign`. While
   `pending` it belongs to no location, so there's no per-branch queue and no way
   to scope operators. (Cross-location *fulfilment* already works mechanically —
   `assign` accepts any open assignment regardless of store — it's just
   unstructured.)
2. **Every operator sees and can act on every order.** Operators aren't scoped to
   a store. With multiple operators this is a **coordination hazard**: operator A
   creates an order intending to route it; operator B grabs it from the shared
   queue and assigns it elsewhere, overriding A's decision.
3. **State transitions aren't atomic.** `assign` (and the other transitions) read
   the current status through the pool, check it, then write in a separate step.
   Two concurrent `assign`s can both read `pending`, both pass the check, and the
   second silently overwrites the first — a classic lost-update **data race** the
   status-machine 409 does *not* prevent.

The business need behind this: when a branch can't fulfil an order, it's often
easier to hand the order to **another branch that has stock** than to scramble
inventory into the first branch (purchases/transfers remain the operator's job —
we don't block order creation on inventory). And each branch should own its own
queue so operators don't step on each other.

## Proposed Solution

Add a **location + ownership** dimension to orders and make transitions atomic.

- **`orders.storeId`** (owning branch) set at creation — from the creating
  operator's active store (admin may pick any). Fulfilment can still cross
  branches via the assignment; `storeId` is the order's *current queue owner*, the
  assignment's store is the *fulfiller* (they may differ).
- **`orders.ownerId`** (the operator currently managing the order), set to
  `createdBy` at creation. Distinct from `createdBy` (which stays the immutable
  audit author). Pre-delivery **management** actions (`assign`, `cancel`,
  `transfer`) require `caller == ownerId` **or** admin/developer — this is the
  fix for the coordination hazard (#2). The creator owns what they create.
- **Store-scoped operators** via the existing `store_assignments` (its comment
  already names "driver/operator"): an operator's store set is its active
  `store_assignments`. `GET /orders` (list + detail) returns only orders in the
  caller's store(s); **admin/developer are global** (the "global operator" role is
  satisfied by admin for now — a dedicated capability is a future improvement).
  Delivery stays scoped to its assigned orders (already started in
  `orders-foundation`).
- **Transfer handoff** — `POST /:id/transfer { storeId }` (pre-dispatch only)
  re-homes the order to another branch: sets `storeId`, clears `assignmentId`,
  reverts status to `pending`, releases ownership (`ownerId = null`), appends
  history. The receiving branch's operators then pick it up (assigning auto-claims
  ownership). This keeps each branch's drivers/inventory decisions local.
- **Atomic transitions** — every state move becomes a conditional write inside the
  transaction (`UPDATE orders SET … WHERE id = $id AND status = $expected [AND
  ownership/store guards]`); **0 rows affected → 409** "ya fue actualizado". This
  closes the lost-update race (#3) regardless of scoping.

Single-store today is fine: the ownership guard + atomic transitions deliver value
immediately with multiple operators in one branch; multi-store is non-breaking
(the catalog already notes everything keys on `store.id`).

Detailed data model, endpoints, and file lists live in [[backend]] and
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [x] `orders.storeId` int FK → `stores.id` **NOT NULL**, set at creation from the
  creating operator's active store (admin may pass an explicit `storeId`). Drizzle
  migration; back-fill strategy for any existing rows documented.
- [x] `orders.ownerId` int? FK → `users.id`, set = `createdBy` at creation;
  `createdBy` remains the immutable audit author. Index `(storeId, status)`.
- [x] Operator store-scoping derived from active `store_assignments`. `GET /orders`
  list returns only orders whose `storeId` ∈ caller's stores; `GET /orders/:id`
  for an out-of-scope order is **404** (don't leak existence across branches).
  Admin/developer see all. Delivery scoped to its assigned orders.
- [x] Owner-gated management: `assign`, `cancel`, `transfer` require
  `caller == ownerId` OR admin/developer, else **403**. Assigning an unowned
  (`ownerId IS NULL`) order in the caller's branch **auto-claims** it
  (`ownerId = caller`).
- [x] `POST /api/v1/orders/:id/transfer { storeId }` — pre-dispatch only
  (`pending|assigned|failed`); validates the target store exists; sets `storeId`,
  clears `assignmentId`, reverts status to `pending`, sets `ownerId = null`,
  appends `order_status_history` (reason: transferred). Owner-or-admin gated.
- [x] **Atomic state transitions**: `assign`/`dispatch`/`deliver`/`fail`/`cancel`/
  `transfer` perform the status change as a conditional `UPDATE … WHERE id = ? AND
  status = <expected>` (plus ownership/store guards where they apply); **0 rows →
  409**. Replaces the read-then-write pattern so concurrent callers can't both win.
- [x] `POST /orders` create sets `storeId` + `ownerId`; validates the operator
  belongs to the chosen store (admin exempt).
- [x] Routes/reads keep flowing through `src/lib/errors.ts`; Lima dates via
  `src/lib/date.ts`. The atomic-transition refactor preserves all
  `orders-foundation` behaviour (its tests stay green).
- [x] Lifecycle/concurrency tests: (a) operator sees only own-store orders
  (list + 404 on cross-branch detail); (b) non-owner `assign` → 403, owner
  succeeds, unowned auto-claim sets ownerId; (c) transfer re-homes store, clears
  assignment, reverts to pending, releases owner; (d) two concurrent `assign`s →
  exactly one 200, the other 409 (atomic guard).

**Frontend (lpg-frontend-vue):**

- [x] Order queue is **branch-scoped** (operator sees its store; admin gets a
  store switcher / "all branches" view).
- [x] **Transfer-to-branch** action on the order detail/queue (pre-dispatch),
  picking a target store; surfaces the re-home result.
- [x] **Ownership affordances**: show the owner; disable `assign`/`cancel`/
  `transfer` for non-owners; offer "claim" (or auto-claim on assign) when unowned.
- [x] Store selector on the create form (defaults to the operator's store; admin
  picks any).
- [x] Role/nav wiring; concurrent-edit feedback (surface the 409 "already updated"
  cleanly rather than a generic error).

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

> **Porting order:** backend first (schema + scoping + atomic transitions + the
> transfer endpoint), frontend right after.

## Out of Scope

- **A dedicated "global operator" role** — admin/developer cover cross-branch
  management for now; a first-class capability is a future improvement.
- **Nearest-store auto-routing** from the customer's address/geo — the operator
  picks the store; deriving it is a later enhancement.
- **Automatic inventory transfer/purchase to fulfil an order in-place** — stays
  the operator's manual job via the existing inventory purchase/transfer paths;
  the whole point here is to route the *order* instead.
- **Direct cross-branch driver assignment** (operator A assigning straight to
  store B's driver) — rejected in favour of the transfer handoff so each branch
  owns its drivers/inventory.
- **Per-order soft locks / presence ("operator X is viewing")** — ownership +
  atomic 409s are the MVP guard; live presence is a future nicety.

## Open Questions

- **Payment scoping:** is `POST /:id/payments` owner-gated, or any operator in the
  order's branch (+admin)? (Lean: branch-wide, not owner-gated — payments aren't a
  routing decision.)
- **Claim mechanism:** implicit auto-claim on `assign`, or an explicit
  `POST /:id/claim`? (Lean: implicit auto-claim for MVP; add explicit claim only
  if operators want to reserve before assigning.)
- **Transfer while `assigned`:** always revert to `pending` + clear the
  assignment, and forbid transfer once `in_transit`/`delivered`? (Lean: yes —
  pre-dispatch only; reverting is the clean handoff.)
- **Existing-row back-fill:** any orders created before this spec have no
  `storeId`/`ownerId`; back-fill from the assignment's store (when present) and
  leave `ownerId` null, or assign to the single seeded store? (Lean: single seeded
  store + null owner, since multi-store isn't live yet.)
- **Cross-branch detail:** 404 (chosen above) vs 403 — confirm 404 is the desired
  privacy posture.
