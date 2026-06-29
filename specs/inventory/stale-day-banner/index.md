---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../day-handoff/index]]"
  - "[[../inventory-foundation/index]]"
last-updated: 2026-06-16
---

# Spec: Stale Day Banner (surface unresolved prior days on /inventario)

## Problem Statement

A driver's day can be left **unresolved across a date boundary** — stuck in
`open` (the day was never counted/closed) or sitting in `closed` (counted but
never consolidated). The [[../day-handoff/index|day-handoff]] work guards
against *compounding* this (the `openDay` 409 blocks a driver from opening today
while holding an earlier non-`carried` day), but it only fires **reactively**,
for the **one driver** the operator happens to be opening, inside the
`OpenDayDialog`. Nothing on `/inventario` tells the office, **proactively and at
a glance**, that there are leftover past days to clear before the day's work can
start cleanly.

Worse, the default surface actively **hides** them: the **Asignaciones** tab's
date filter defaults to **today** ([store.ts:57](../../../../../dev/personal/freelance/lpg-store/lpg-frontend-vue/src/modules/inventory/store.ts)),
so a past-dated `open` assignment is invisible unless someone manually clears the
date and filters by state. The **Por consolidar** tab lists `state=closed` days
but shows **all** of them (including today's), in a separate tab, and never the
stuck-`open` case.

Whoever is working the office that day needs a single, unmissable signal:
**"these prior days are unresolved — clear them so today can start."**

## Proposed Solution

A **persistent banner above the tabs** on `/inventario` that lists every
**non-`carried` assignment dated before today**, within the caller's store
scope. It renders only when there is at least one such day; otherwise nothing
shows. Each row names the day (date, driver/store, state) and links to its
detail so the operator can resolve it — **consolidate** a `closed` day, or
**force-close + consolidate** a stuck `open` one — and the banner clears the row
once resolved.

**No backend change.** The data is already available through the existing,
caller-scoped `GET /inventory/assignments?state=open` and `?state=closed`
(`InventoryAssignment` carries `date` + `state`); the frontend fetches both and
filters to `date < today` client-side — the same "reuse the existing filter"
approach the day-handoff backend track prescribed, and consistent with the
project's anti-bloat stance against new endpoints. Driver/store names resolve
through the catalog map the page already builds (`assignmentLabel`).

Detailed frontend design lives in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

- [ ] A persistent banner/section renders **above the tabs** on `/inventario`
      whenever there is ≥1 assignment in state `open` or `closed` dated **before
      today**, within the caller's store scope. With none, nothing renders (no
      empty/placeholder banner).
- [ ] Each pending prior day is listed with its **date**, **driver / store**
      (resolved via the existing catalog `assignmentLabel` map), and **state**
      using the canonical status→badge mapping (`open`→info "Abierto",
      `closed`→secondary "Cerrado").
- [ ] The banner copy distinguishes the two resolution paths so the office knows
      the action each needs: a `closed` day is **ready to consolidate**; an
      `open` day must be **force-closed then consolidated** first.
- [ ] Each row links to that assignment's detail view, where the existing
      Consolidar / Forzar cierre actions resolve it.
- [ ] **Today's** `open`/`closed` assignments are **not** shown in the banner
      (they aren't stale) — only `date < today`.
- [ ] Data comes from existing endpoints only — `GET /assignments?state=open`
      and `?state=closed`, filtered to `date < today` client-side. **No new
      backend endpoint, no backend change.**
- [ ] After a day is resolved (consolidate/force-close from its detail, or on
      returning to `/inventario`), the banner re-fetches so the cleared day
      drops off.
- [ ] Design-system compliant (Alert/Card tokens, status→badge mapping,
      `Spinner`, ≥44px touch targets); `npm run typecheck` + `npm run build`
      green. Manual smoke (leave a past `open` + a past `closed` day → both
      appear in the banner → resolve each → banner clears) left to the operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Any backend change.** Server-authoritative staleness (Lima business date vs.
  the device-local `todayISO()` the app uses everywhere) is deferred — see Open
  Questions; if midnight-boundary mis-marking ever bites, that becomes a small
  backend track.
- **Auto-resolution / bulk consolidate.** Each day is resolved through its
  existing detail flow; the banner only surfaces + links.
- **Changing the "Por consolidar" tab.** It keeps listing all `closed` days as
  the working consolidation queue; the banner is the proactive, *past-day-only*
  cross-cut (open + closed) sitting above the tabs.
- **Notifications / push** to the office that a prior day is unresolved.

## Open Questions

- **Device-local vs. Lima business date for "today".** Defaulting to the
  device-local `todayISO()` for consistency with the rest of the app (the date
  filter, `getMyDay`, etc. all use it). Revisit only if a near-midnight or
  off-timezone device is observed mis-marking a current day as stale — that
  would be the trigger to add a tiny server-authoritative read (backend track).
- **Dismissible vs. always-on.** Proposed: always shown while pending prior days
  exist (it's a blocker reminder, not a notice to snooze).
