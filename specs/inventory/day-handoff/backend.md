---
project: lpg-store
domain: specs
type: spec-track
spec: day-handoff
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-15
---

# Day Handoff — lpg-backend track

Shared spec: [[index]] · Foundation: [[../inventory-foundation/index]] · Edge-case catalogue (E1–E11): [[../index]]

## Technical Notes

This is a thin layer over the existing inventory module — **no schema/ledger/view/enum changes** unless the stale-day query needs an index. Four changes:

### 1. Role split on close / carry

Both routes currently share `canWrite = requireRole('operator','delivery','admin')` ([routes.ts:110](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/inventory/routes.ts), `:121`). Split them:

- **`POST /assignments/:id/close`** — the caller must be **the assignment's own driver** or admin/developer. The assignment → `storeAssignment` → `userId` is the owning driver; compare to `req.user.id`. A non-owning `delivery`/`operator` → 403. Admin/developer bypass (override). This likely needs an ownership check *inside the service* (it already loads the assignment + storeAssignment), not just a role middleware, since "own driver" is row-scoped.
- **`POST /assignments/:id/carry`** — `requireRole('operator','admin','developer')` only; **drop `delivery`**. This is the structural second signature: the driver who closed cannot also consolidate.

Mirror the existing scoping helpers already used by `listAssignmentsForCaller` (the orders-multi-location work added `storeIdsForUser` / caller-scoping) so the ownership check is consistent with how the rest of the module reasons about callers.

### 2. Mandatory physical count on close

`closeSchema` currently has `finalCounts?` optional ([types.ts](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/inventory/types.ts)). Make it **required and complete**: every tank type with a non-zero live balance on the assignment must appear in `finalCounts` (full+empty). Reject (400) a close with missing/empty counts. The reconciliation rows are still written exactly as today — the only change is that the count is now compulsory, so the driver's attestation always carries numbers and a clean (zero-discrepancy) close still writes the explicit physical count.

### 3. Carry-forward suggested-opening read

New read endpoint (advisory only, no writes), e.g. `GET /assignments/suggested-opening?storeAssignmentId=` (or fold into the existing `GET /stores/:storeId/availability` response). Returns, per tank type, the **hand-back amounts from that driver's most recent `carried` assignment** — i.e. what came off the truck last time — as the default opening. No history → empty/zeros. The frontend pre-fills the open form; the operator edits freely. Opening remains an explicit `openDay` decision and may differ from the suggestion (E10 / ADR-014 unchanged).

> Source of truth is the backend (it owns the ledger). The frontend only presents + lets the operator adjust.

### 4. Stale prior-day guard on openDay

`openDay` already enforces today-only dating and per-`(storeAssignment, date)` uniqueness, but **does not** stop a driver from opening today while an earlier day is still `open` or `closed` (not yet `carried`). Without a guard, stock silently splits across two assignment holders.

- In `openDay`, before creating today's row, check for any assignment for the same `storeAssignment` (i.e. same driver) in state `open` or `closed`. If found → **409** with a payload naming the blocking assignment(s) (id, date, state) so the UI can route the operator to resolve them.
- Expose a query to find these: `GET /assignments?state=open|closed&storeAssignmentId=` (filters already exist) — the operator's open-day screen lists pending ones.
- **Recovery paths:** a `closed` prior day → operator runs `carry` (consolidates, stock returns to store). A stuck `open` prior day → admin/operator force-closes (the close path, admin override) then consolidates. (See Open Questions in [[index]] re: admin-only + recovery note.)

### Notes

- No new transaction kind, no new ledger columns, no enum change. If the stale-day lookup is hot, add an index on `inventory_assignments(store_assignment_id, state)`; otherwise skip (low volume).
- Keep `carry` idempotent and the paired hand-back `transfer` + `reconstructDay` behaviour exactly as in the foundation.
- Provider refills (`tank-purchases` / `item-purchases` / `recordLoad`) are untouched and must keep working while `open`.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — split guards on `close` (own-driver/admin) and `carry` (operator/admin, drop delivery); add the suggested-opening route.
- `src/modules/inventory/service.ts` — ownership check on `closeAssignment`; mandatory-count validation; `suggestedOpening(...)`; stale-day guard in `openDay`.
- `src/modules/inventory/types.ts` — make `closeSchema.finalCounts` required+complete; suggested-opening response type.
- `src/modules/inventory/repository.ts` — query for prior non-`carried` assignments by storeAssignment; last-`carried` hand-back amounts for the suggestion.
- `src/modules/inventory/__tests__/` — role-split (driver-only close, operator-only carry, 403s), mandatory-count (400), stale-day guard (409 + payload), suggested-opening (last carried → defaults).
- (maybe) `src/db/migrations/<n>_*.sql` — only if an index is added.

Reference: [[backend|inventory-foundation backend track]] for the existing lifecycle, scoping helpers (`listAssignmentsForCaller`, `storeIdsForUser`), and the close/carry implementation.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-backend] Backend track done. **Role split (two signatures):** `POST /assignments/:id/close` now gated `canCount = requireRole('delivery','admin')` (developer auto-passes) + a service-level ownership check (`storeAssignment.userId === caller.id`, else admin/dev override, else 403); `POST /assignments/:id/carry` gated `canConsolidate = requireRole('operator','admin')` — **`delivery` removed**, so the driver who closes can't also consolidate. `closeAssignment` signature changed `userId` → `caller: {id, role}`. **Mandatory count:** the owning-driver close requires `finalCounts` covering every non-zero tank type (400 otherwise); admin override may close without counts (`closeSchema.finalCounts` stays optional, completeness enforced in the service). **Carry-forward:** new advisory `GET /suggested-opening?storeAssignmentId=` → `service.suggestedOpening` sums the last `carried` day's non-`transfer` rows (repo `findLastCarried`). **Stale guard:** `openDay` rejects 409 (naming blockers) when the driver holds an earlier non-`carried` day (repo `findUnconsolidated`). **Late-close annotation:** a close where `assignment.date !== today` writes a zero-delta `adjustment` row noted `cierre tardío: …` with `userId` = closer (override marker when not the driver) — ledger-as-audit-trail (ADR-009), seed for a future "who isn't following the flow" admin signal. No schema/view/enum change, **no migration**. Recorded **ADR-017**. Updated existing tests (lifecycle carry → operator + driver-carry-403/operator-close-403 asserts; 3 service-test close call sites → `{id, role}`); new `day-handoff.test.ts` (role split, mandatory/complete count, admin override, suggested-opening, stale-guard, late-close annotation). Gates green: typecheck + lint + **80 tests** (was 70) + build. Independent validation: all 8 backend criteria MET (the one flagged "developer not in middleware" is a false positive — `createRequireRole` auto-passes `developer`). Frontend track remains.