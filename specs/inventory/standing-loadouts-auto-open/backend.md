---
project: lpg-store
domain: specs
type: spec-track
spec: standing-loadouts-auto-open
repo: lpg-backend
kind: backend
track-status: not-started
last-updated: 2026-07-17
---

# Standing Loadouts → Auto-Open — lpg-backend track

Shared spec: [[index]]

> Depends on [[../driver-inventory-pools/backend]] being `done` — this assumes
> `location` holders are already per delivery store-assignment.

## Technical Notes

- **New table** `store_assignment_loadouts` (or `assignment_loadout_lines`):
  per delivery `store_assignment` → per `tank_type` / `inventory_item` target
  quantity. Named migration (`db:generate -- standing-loadouts`).
- **Apply path.** One reusable service `openDayFromLoadout(storeAssignmentId,
  caller)` = resolve the loadout → build the `tanks[]`/`items[]` payload as
  `min(target, available-in-that-driver's-pool)` → delegate to the existing
  `openDay` (today-only, dup-day 409, unconsolidated-prior guard, negative guard,
  driver push all reused). No loadout → fall back to the whole-pool sweep
  (`quickOpen`). Shortfall = a returned per-line advisory, never a throw.
- **Store-level "Abrir día"** = iterate the store's not-yet-open delivery
  assignments, call the apply path for each in one request; aggregate results
  (opened + shortfalls). Skip already-open days.
- **Scheduler.** Opt-in per store (a flag/table). A Lima-business-day job invokes
  the same store-level open; **idempotent** (skip open days); runs as a system
  actor. Mechanism TBD (Open Question — `$0` infra constraint, no paid queue;
  candidates: in-process cron guarded by an advisory lock, or a DB-polled tick).
- Reuse: `openDay`, `loadFromLocation`/`loadItemFromLocation`,
  `activeDeliveryAssignmentsForStore`, `src/lib/date.ts` (America/Lima).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

- `src/modules/inventory/schema.ts` — `store_assignment_loadouts` table (+ per-store
  auto-open flag).
- `src/db/migrations/00XX_standing-loadouts.sql` — new table(s).
- `src/modules/inventory/service.ts` — `openDayFromLoadout`, store-level open-all,
  loadout CRUD.
- `src/modules/inventory/repository.ts` — loadout reads/writes.
- `src/modules/inventory/routes.ts` — loadout CRUD + `POST …/stores/:id/open-all`.
- `src/modules/inventory/types.ts` — loadout + open-all shapes (opened + shortfalls).
- scheduler wiring (module TBD per the Open Question) + `src/modules/inventory/__tests__/*`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
