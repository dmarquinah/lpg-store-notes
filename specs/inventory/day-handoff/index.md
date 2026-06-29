---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../../orders/orders-foundation/index]]"
  - "[[../../orders/orders-multi-location/index]]"
last-updated: 2026-06-15
---

# Spec: Day Handoff (two-signature day close + carry-forward open)

## Problem Statement

The day lifecycle logic from [[../inventory-foundation/index]] is correct, but its **execution is too manual and concentrated on a single role**. Today every step — `openDay`, `closeAssignment`, `carryAssignment` — is gated by the same `canWrite` (operator / delivery / admin) guard, so in practice one operator types the opening counts, types the closing counts, eyeballs the discrepancies, and runs *two* separate end-of-day actions (mark-done, then "Consolidar"). The person entering the numbers is not the person standing at the truck.

We want to **distribute the day-end effort across two roles without losing any traceability**. The append-only ledger and per-day reconstruction already guarantee precision; the gap is purely *who triggers/confirms each step* and *how much re-entry the start of day demands*.

Concretely:

1. **End of day must be a two-signature event.** The **driver** physically counts the truck and marks the day as counted (the first signature / attestation). The **operator** then reviews and **consolidates** — returning the leftover stock to the store for the next day (the second signature / confirmation). Two different roles must act, enforced by the system, not by convention.
2. **Start of day must stop demanding cold re-entry.** When opening a driver's day, the opening counts should **default from what that driver consolidated the previous day** (carry-forward), editable before opening — opening still need not equal yesterday's closing (per [[../index|E10]] / ADR-014). The operator should also be able to see at a glance whether a prior day is still **unconsolidated** (rare, but possible) and resolve it before opening today.
3. **Provider refills at any moment stay recordable.** The operator can receive a provider refill of tanks or items into the store at any point in the day; this is already the `tank-purchases` / `item-purchases` flow and must keep working alongside the new handoff.

## Proposed Solution

The end-of-day model maps almost 1:1 onto the two methods that already exist — the change is **who is allowed to call each**, plus making the driver's count mandatory:

- **`closeAssignment` becomes the driver's attestation.** Gated to the assignment's **own driver** (+ admin override); `finalCounts` becomes **mandatory** (the physical count *is* the attestation). Still writes the `reconciliation` rows and moves `open → closed`. No logic rewrite.
- **`carryAssignment` becomes the operator's confirmation.** Gated to **operator / admin only — `delivery` removed.** Still returns the truck's leftovers to the store and moves `closed → carried`. Because close and carry are gated to *different roles*, the two signatures are structurally enforced; admin is the only escape hatch.
- **`openDay` gains a carry-forward default + a stale-day guard.** A new backend read returns a **suggested opening** for a driver (their last `carried` truck load); the frontend pre-fills the open form with it, fully editable. `openDay` is **guarded** so a driver cannot open today while holding an earlier non-`carried` assignment — those are surfaced for the operator to resolve first (consolidate a `closed` one; force-close + consolidate a stuck `open` one).

No changes to the ledger shape, the `kindToDelta` dispatch, the balance views, or the `open|closed|carried` enum — this is a guard / mandatory-field / read-endpoint / UI change layered on the existing foundation.

Detailed backend design lives in [[backend]]; the driver and operator screens in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `POST /assignments/:id/close` is gated to **the assignment's own driver** (the `userId` behind the assignment's `storeAssignment`) **or admin/developer**; any other caller → 403.
- [ ] `finalCounts` is **mandatory** on close (every tank type with a non-zero balance must be counted); a close with no counts → 400.
- [ ] `POST /assignments/:id/carry` is gated to **operator / admin / developer only**; a `delivery` caller → 403.
- [ ] `carry` still requires `state='closed'` first (409 otherwise) and stays idempotent; the existing paired hand-back `transfer` + reconstruction behaviour is unchanged.
- [ ] A new read returns a **suggested opening** for a driver/store, derived from that driver's most recent `carried` assignment's hand-back amounts (empty/zero when there's no history). It is advisory only — it never auto-applies.
- [ ] `openDay` **rejects** (409) when the target driver has any earlier assignment in `open` or `closed` state, and the error surfaces which assignment(s) block it; a list/query lets the operator find those pending assignments per store/driver.
- [ ] Provider refills remain recordable mid-day and unchanged: `tank-purchases`, `item-purchases` (store), and `recordLoad` (store→truck) all still work while the day is `open`.
- [ ] Role-split + mandatory-count + stale-day-guard each have passing tests; existing inventory tests stay green; typecheck / lint / build green; any migration (if needed) applied to the dev DB.

**Frontend (lpg-frontend-vue):**

- [ ] The **driver** has a clear "Marcar día como contado" flow on their day view: enter the **required** physical count per tank type (expected vs entered shown), submit → the day moves to `closed`. The driver cannot consolidate.
- [ ] The **operator** has a "Consolidar" review surface listing assignments in `closed` awaiting consolidation, showing each one's discrepancies, with a one-click confirm → `carried`. The operator does not enter the closing count.
- [ ] The operator's **open-day** form pre-fills opening counts from the backend suggested-opening (carry-forward), fully editable before opening.
- [ ] The open-day flow surfaces any **pending unconsolidated prior day** for that driver and routes the operator to resolve it before opening today (handles the rare stuck-`open` case via admin/operator force-close + consolidate).
- [ ] Manual smoke test of the full two-party flow: open (carry-forward) → driver counts/closes → operator consolidates → next-day open defaults from it.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Start-of-day driver handshake** — opening stays operator-driven (with carry-forward defaults); no driver "confirmar carga" signature. Explicitly decided out (the two-signature requirement is end-of-day only).
- **Item-load onto the truck** — items live at the store only; no `recordItemLoad`. Tank loads + provider purchases (tanks & items) already cover refills.
- **Auto-consolidation on close** — rejected; the two steps stay separate by design (that *is* the two-signature split).
- **State-machine rename** — the `open|closed|carried` enum is kept; only the role semantics around it change.
- **Notifications** to the operator that a driver has counted — nice-to-have, deferred (the operator polls the "awaiting consolidation" list).

## Open Questions (resolved 2026-06-15)

- **Closing on a driver's behalf is admin-only.** Operators cannot close — close is the driver's attestation; only the owning `delivery` user or an admin/developer (override) may call it. Operators act at the *carry* step.
- **A stuck-`open` prior day, when force-closed, is annotated in the ledger.** The recovery writes an attributable annotation row (`adjustment`, zero-delta, distinctive `cierre tardío` note, `userId` = the admin who closed it) so the deviation is visible on later days for confirmation/evaluation. This is the seed for a future admin signal of *which users aren't following the workflow* so they can be coached over time.
