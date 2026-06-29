---
project: lpg-store
domain: specs
type: spec-track
spec: day-handoff
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: 2026-06-15
---

# Day Handoff — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend track: [[backend]]

## Technical Notes

Reshapes the existing `inventory` + delivery surfaces around the two-party day-end. No new module — extends the `inventory` module and the driver's delivery view. Follow the canonical conventions in `eng/design-system.md` (touch targets, status→badge mapping, `PageHeader`/`Spinner`/`EmptyState`, `formatMoney`).

### Driver — "Marcar día como contado"

On the driver's day view (the assignment they own — same surface as `/mis-entregas` / inventory assignment detail):

- A prominent end-of-day action that opens a **count form**: one row per tank type with a non-zero balance, showing **expected (from balances) vs entered**, full + empty inputs **required**. Phone-first (this is the driver on a device in the field).
- Submit → `POST /assignments/:id/close`. On success the day shows as `closed` / "Contado — esperando consolidación". The driver has **no** consolidate action (the button/route is gated out for `delivery`).
- Surface a clear discrepancy preview before submit so the driver knows what they're attesting to.

### Operator — "Consolidar" review

- A list/section of assignments in `closed` **awaiting consolidation** (per the operator's store scope), e.g. via `GET /assignments?state=closed`.
- Each row → review showing the driver's counted discrepancies (`GET /assignments/:id/discrepancies`) and the closing balances, with a single **Consolidar** confirm → `POST /assignments/:id/carry`. The operator does **not** type the closing count.

### Operator — open day with carry-forward

- The open-day form pre-fills opening counts from the backend **suggested-opening** read, fully **editable** before submit (opening may differ from yesterday's closing).
- Before opening, surface any **pending unconsolidated prior day** for the chosen driver and route the operator to resolve it first (consolidate a `closed` one; the rare stuck-`open` one via the admin/operator force-close path). Map the backend 409 (with blocking-assignment payload) to a clear, actionable message rather than a generic error.

### Role gating

- `delivery` sees the count/close action but never carry.
- `operator`/`admin` see carry + open + the awaiting-consolidation list; the closing-count entry is the driver's, not theirs.
- Mirror the backend guards client-side (as `orders-multi-location` did for ownership) so the UI never offers an action the API will 403.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify (confirm exact paths when the track starts):

- `src/modules/inventory/` — open-day form (carry-forward pre-fill + pending-prior-day handling), the operator "awaiting consolidation" list + Consolidar review, role gating of the carry action.
- The driver delivery view (`/mis-entregas` and/or the assignment detail) — the required "Marcar día como contado" count flow.
- The inventory store/service layer — new calls: suggested-opening, list `state=closed`, the role-aware close/carry.
- Nav/routing — ensure the driver reaches the count flow and the operator reaches the consolidation list.

Reference: [[frontend|inventory-foundation frontend track]] for the existing assignment list/detail + ledger dialogs, and the `orders-multi-location` frontend track for the client-side guard-mirroring pattern.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-frontend-vue] Frontend track implemented over the existing `inventory` module + the driver surface (no new module). **Two-signature day-end:**

- **Driver close (1st signature):** new delivery-scoped route `/mi-dia` → `views/DriverDayView.vue` (phone-first, card list like DeliveryListView). Loads the driver's today assignment via new store `getMyDay(today)` (caller-scoped `listAssignments({date})`) then `fetchAssignment(id)` for balances. When `open`: prominent **"Marcar día como contado"** → new `components/CountDialog.vue` — **mandatory** full+empty per tank type, expected-vs-entered labels + live per-row discrepancy preview, completeness validation mirroring the backend close gate, `closeAssignment`. When `closed`: "Ya contaste tu día…" (no carry action). Added "Mi día" to the `delivery` `ROLE_NAV` (AppLayout) + a card on `DeliveryHome`.
- **Operator consolidate (2nd signature):** `views/InventoryView.vue` gains a **"Por consolidar"** tab (count badge) listing `state=closed` assignments via new store `consolidationList`/`fetchConsolidationList()` (caller-scoped) → row click routes to `AssignmentDetailView` (existing discrepancies table + **Consolidar**/carry button, unchanged).
- **Role gating (mirrors backend):** `AssignmentDetailView` — the old "Cerrar día" button is now **admin/developer-only** and relabelled **"Forzar cierre"** (override/recovery; operators never close). "Consolidar" stays for operator/admin/developer. Delivery never reaches this view (route excludes it).
- **Open-day carry-forward + stale guard:** `OpenDayDialog` — on driver select, fetches the backend **suggested-opening** (new service `getSuggestedOpening` → `GET /suggested-opening?storeAssignmentId=`, envelope key `tanks`) and pre-fills the tank rows (fully editable; muted hint explains the source / "sin historial"). Also fetches the driver's **pending** non-`carried` days (new store `getPendingAssignments`, filtered open/closed); if any exist it shows a destructive Alert linking each to its detail to resolve and **blocks submit** (mirrors the backend openDay 409, which still surfaces as a fallback via `store.error`).

New types: `SuggestedOpeningResponse`. Gates green: `npm run typecheck` + `npm run build` (PWA precache 64 entries / 712.72 KiB). Manual two-party smoke test (open→count→consolidate→next-day carry-forward) left to the operator per the project's standing verification method.

[2026-06-15] [lpg-frontend-vue] All criteria for this repo met. Independent validation: criteria 1, 2, 3, guards A & B PASS; criterion 4 PASS — the validator's "partial" flag assumed a bare-string 409 body, but the backend `errorHandler` sends the structured `{error:{code,message}}` envelope, so `apiClient` maps 409 → `ConflictError` with the backend's Spanish message and the store's `messageFrom` surfaces it (the client-side `hasPending` pre-check is the primary path; the 409 message is the correct fallback). Gates green (typecheck + build). Frontend track **done**; both tracks done → spec done. Manual two-party smoke test left to the operator per the project's standing verification method.