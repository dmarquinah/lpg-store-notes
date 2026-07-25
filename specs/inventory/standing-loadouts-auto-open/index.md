---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: draft
depends-on:
  - "[[../driver-inventory-pools/index]]"
  - "[[../quick-assignment/index]]"
  - "[[../day-handoff/index]]"
last-updated: 2026-07-17
---

# Spec: Standing Loadouts → One-Click "Abrir día" → Auto-Open (assignment automation)

## Problem Statement

Even after [[../driver-inventory-pools/index|per-driver inventory pools]] make
daily assignment **one click per driver** (each driver sweeps their own pool),
two frictions remain as the business scales:

1. A store with N drivers is still **N taps** to start the day, and the "sweep
   everything currently in the pool" semantics can't express a driver's *usual*
   daily load when a pool holds a **shared pile** to divide.
2. Starting the day is still a **manual act** every morning. The owner's stated
   goal is for this to eventually be **automated** — but automation is only safe
   once the flow is perfected and each driver's intended load is expressed
   **declaratively**, not inferred.

This is the automation phase deliberately deferred out of
[[../driver-inventory-pools/index]].

## Proposed Solution

Give each **delivery store-assignment** an optional **standing daily loadout** —
a saved template of the quantities that driver normally takes (e.g. `30×10kg +
10×45kg`). Then layer two conveniences on top:

- **One-click store open ("Abrir día").** A single action opens **every** driver
  of the store at once, each pulling **their loadout's share** from stock
  (`min(template, available)`; shortfall surfaced, never blocks). Reduces N taps
  → **1 tap for the whole store**, and expresses a shared-pile split declaratively
  (no typing).
- **Scheduled auto-open (0 clicks).** An opt-in per-store schedule (Lima
  business-day aware) that applies the loadouts automatically each morning, so the
  admin does nothing. The one-click button is the manual fallback and the thing
  the schedule simulates.

The **degenerate cases stay simple**: a driver with **no loadout** falls back to
the existing "sweep the whole pool" quick-open; a single-driver store's "Abrir
día" is its existing one-tap.

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

- [ ] A **standing loadout** can be defined/edited/cleared per **delivery
      store-assignment**: per-tank-type (and item) target quantities. New table
      + migration; admin/operator (own-store) scoped.
- [ ] **One-click "Abrir día"** on a store opens **all** its not-yet-open drivers
      in one action, each loading `min(loadout, available-in-pool)`; a driver with
      **no loadout** uses the existing whole-pool sweep. Per-driver **shortfall**
      (loadout > available) is reported, **not** blocked. Start-of-day rules
      (today-only, unconsolidated-prior-day guard) reused unchanged.
- [ ] **Scheduled auto-open** is **opt-in per store**, runs on the Lima business
      day, applies loadouts via the same path as the button, is **idempotent** (a
      day already open is skipped), and records who/what opened it (system actor).
      Failures (e.g. unconsolidated prior day) are surfaced, not silently dropped.
- [ ] Single-driver stores and drivers without a loadout behave **exactly as
      today** (no regression to [[../driver-inventory-pools/index]] or
      [[../quick-assignment/index]]).
- [ ] Full gate green in both repos; new tests for loadout CRUD, one-click
      multi-driver open with shortfall, no-loadout fallback, and schedule
      idempotency.
- [ ] **Frontend:** a loadout editor per driver; the store card's "Abrir día"
      (opens all) + a per-store auto-open toggle; shortfall shown clearly.
      Design-system compliant.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | not-started |
| [[frontend]] | lpg-frontend-vue | frontend | not-started |

## Out of Scope

- The per-driver pool re-key itself — that's [[../driver-inventory-pools/index]]
  (a hard dependency; this spec assumes pools exist).
- Merging duplicate stores (separate deferred cleanup).
- Demand-forecasting / dynamic loadouts — loadouts are **static templates** the
  admin maintains, not computed from history (a possible later evolution).

## Open Questions

- **Scheduler mechanism** — reuse an existing in-process cron, a DB-polled job,
  or the platform's scheduler? (Constraint: `$0` infra — no paid queue.)
- **Loadout unit** — absolute quantities, or "top up to N" vs "take exactly N"
  when the pool already holds stock from carry-forward?
- **Shortfall policy on auto-open** — open partial + flag (lean), or hold the
  driver closed until stock arrives?
- **Carry-forward interaction** — does a loadout mean "on top of carried
  residual" or "target total including residual"?
