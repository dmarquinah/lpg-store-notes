---
project: lpg-store
domain: specs
type: spec-track
spec: orders-multi-location
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-14
---

# Orders Multi-Location — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

Builds on the existing `orders` frontend module (queue / create / detail /
delivery views shipped under `orders-foundation`). Changes are additive.

- **Branch-scoped queue.** The list already returns only the caller's branch
  orders (backend scopes by `store_assignments`); for **admin**, add a store
  switcher / "all branches" option (and show the branch column when viewing all).
- **Transfer-to-branch.** Add a "Transferir a otra tienda" action on the order
  detail/queue (visible pre-dispatch, owner/admin only) → store picker → calls
  `POST /api/v1/orders/:id/transfer { storeId }`. After transfer the order leaves
  the current branch's queue (reverts to `pending`, unowned) — reflect that.
- **Ownership affordances.** Show the order's owner; disable `assign`/`cancel`/
  `transfer` for non-owners; when the order is unowned, offer "Tomar" (claim) — or
  rely on assign auto-claiming. Make the owned/unowned state legible in the queue.
- **Store selector on create.** Default to the operator's store; for admin, a
  selector to pick the owning branch.
- **Concurrent-edit feedback.** Surface the backend **409 "el pedido ya fue
  actualizado"** as a friendly "another operator just changed this order — refresh"
  toast, not a generic error. Re-fetch the order on 409.
- Role/nav wiring consistent with the existing `ROLE_NAV` pattern.

## Related Files

### lpg-frontend-vue

To modify (mirror the established `orders` module slices):

- `src/modules/orders/{service,store,types}.ts` — `transfer` call; `storeId`/`ownerId`/owner fields; store-switcher state for admin
- `src/modules/orders/views/` — queue (branch scope + admin switcher), detail (transfer + ownership actions), create (store selector)
- `src/modules/orders/components/` — transfer dialog; ownership badge; 409 handling
- router / `AppLayout` `ROLE_NAV` — any nav changes

Context (read):

- the `orders-foundation` frontend track — the module structure these changes extend
- the `customers` / `catalog` modules — store list for the selectors

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

- [2026-06-14] [lpg-frontend-vue] **Frontend track complete — all 5 frontend acceptance criteria met (independently validated).** Additive changes to the existing `orders` module.
  - **Types/service/store** (`types.ts`/`service.ts`/`store.ts`): `storeId` + `ownerId` on `OrderSummary`; `TransferPayload`; optional `storeId` on `CreateOrderPayload`; `storeId` on `ListOrdersFilters` (threaded into the list query string). New `transferOrder(id, {storeId, notes?})` service call + store action (→ `POST /orders/:id/transfer`). **409 concurrent-edit handling**: `resyncOnConflict()` — on a `ConflictError` (apiClient already maps 409), keep the backend's friendly Spanish message ("El pedido ya fue actualizado por otra persona") and re-fetch the order/list so the stale action bar re-syncs; wired into `assignOrder` + the shared `runMutation` (covers dispatch/deliver/fail/cancel/transfer/payment).
  - **Ownership affordances**: `orderLabels.ownerState()` + `OwnershipBadge.vue` render the order's owner relative to the caller (Tú / Otro operador / Sin asignar). Gating mirrors the backend exactly: admin/dev are global managers; owner can manage; **assign** is offered when owned-by-you/manager/**or unowned** (auto-claims); **cancel/transfer** require owner-or-admin only (disabled with a "Gestionado por otro operador" tooltip for non-owners). **Field actions (dispatch/deliver/fail) stay un-gated by ownership — matching the backend** (they're the driver's; `requireOwner` is only on assign/cancel/transfer).
  - **TransferDialog.vue**: store picker (other active branches only) + optional notes → `transferOrder`. On success the detail view routes back to the queue (the order reverts to pending + unowned in the *other* branch, so a non-admin's detail refetch would 404 — navigating away is the clean handoff).
  - **OrdersListView**: admin/dev get a **branch switcher** ("Todas las tiendas" + per-store, client-selected → server `storeId` param) and a **Tienda column** (manager-only "all branches" view); ownership badge per pre-dispatch row; assign/cancel gated. Operators rely on the existing server-side branch scope (no switcher/column).
  - **OrderDetailView**: owner badge + owning-branch name in the header; Transfer button (pre-dispatch, owner/admin); assign/cancel gated by ownership.
  - **OrderCreateView**: admin/dev get a **required owning-branch selector** (gates step 1); operators omit it (server defaults to their active store).
  - **Backend enhancement (cross-repo, user-approved this session):** added a `storeId` query param to `GET /orders` so the admin switcher filters server-side instead of client-side — see [[backend]] note. Gates: frontend typecheck + build green (precache 663 KiB); backend 59 tests + typecheck + lint + build green.
