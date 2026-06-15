---
project: lpg-store
domain: specs
spec-layout: folder
last-updated: 2026-06-15
---

# Specs — lpg-store

**Spec layout: `folder`.** Each spec is `specs/{category}/{slug}/` with a shared `index.md` plus one track file per repo (`backend.md`, `frontend.md`, …). Track badges below: ✓ done · ⧖ in-progress · ◻ not-started. Overall status is `done` only when every track is done. Use `/focus {category}/{slug}`.

## In Progress
| Spec | Category | Tracks | Summary |
|------|----------|--------|---------|

## Backlog
| Priority | Spec | Category | Summary |
|----------|------|----------|---------|
| low | deployment/first-vps-deploy | deployment | First production deploy of the v2 skeleton to the VPS. Validates the CI/CD pipeline end-to-end. _(folder spec to be created)_ |

## Done (recent)
| Spec | Category | Tracks | Summary | Completed |
|------|----------|--------|---------|-----------|
| [[admin/org-management/index\|org-management]] | admin | backend ✓ · frontend ✓ | Unified admin **Organización** view: location-centric board of stores with their assigned operators/drivers (aggregate `GET /catalog/stores-with-assignments`), with store CRUD, per-store assign/**Quitar**, all-users area (Global / Sin-tienda flags, inline edit), and **Invitar usuario** (`POST /auth/register` + copyable `/invite/:token` link). New `organization` module *composes* catalog/users/auth. **Replaced** the standalone `/usuarios` page (→ redirect) + Catálogo *Tiendas*/*Asignaciones* tabs. typecheck + build green. | 2026-06-15 |
| [[orders/orders-queue-history-search/index\|orders-queue-history-search]] | orders | backend ✓ · frontend ✓ | Orders queue split into **Activos** (live, default) / **Historial** (finished, by date range): segmented Tabs, Desde/Hasta `DatePicker`s (default today, no pager), debounced ≥2-char customer **search** box → `GET /orders?search=` (accent-insensitive end-to-end via the backend `unaccent` fix). Multi-status “Todos” fans out per status in the store (backend `status` is single-enum). Admin store switcher + Tienda column + ownership/transfer affordances preserved. typecheck + build green. | 2026-06-14 |
| [[stores/store-management/index\|store-management]] | stores | backend ✓ · frontend ✓ | Admin write surface for stores + store↔user assignments. Backend (62 tests): `POST`/`PATCH /catalog/stores` (unique active names) + `POST`/`PATCH /catalog/store-assignments` (any role, dup active link → 409, soft-deactivate), all admin-gated. Frontend: catalog **Tiendas** tab → create/edit dialog (+ activate/deactivate, show-inactive); new **Asignaciones** tab (assign operator/delivery ↔ store, list, deactivate). New stores flow straight into the orders switcher/transfer. | 2026-06-14 |
| [[orders/orders-multi-location/index\|orders-multi-location]] | orders | backend ✓ · frontend ✓ | Location + ownership dimension on orders: `storeId` (owning branch) + `ownerId` (owner-gated assign/cancel/transfer; auto-claim on assign); store-scoped operator queues via `store_assignments` (admin = global, `?storeId=` switcher); `POST /:id/transfer` pre-dispatch branch handoff; atomic CAS transitions (409 on lost-update, ADR-016). Frontend: branch switcher + Tienda column, OwnershipBadge + gated actions, TransferDialog, admin owning-branch create selector, 409 re-sync. 59 backend tests. | 2026-06-14 |
| [[ui-design/design-system/index\|design-system]] | ui-design | frontend ✓ | Piloto design system + UI overhaul for low-tech-literacy users: petrol+flame token palette (light+dark, semantic success/warning/info), bundled Public Sans, 14px type floor + ≥44px touch targets, theme toggle (`piloto-theme`), shared `PageHeader`/`Spinner`/`EmptyState` + `formatMoney`, restyle of the app shell + all 7 modules (DeliveryListView → phone-first cards). `eng/design-system.md` is the canonical convention doc. | 2026-06-12 |
| [[orders/orders-foundation/index\|orders-foundation]] | orders | backend ✓ · frontend ✓ | Order lifecycle `pending → delivered` (+cancelled/failed); inventory moves at delivery via `recordSale`/`recordItemSale` in one `db.transaction()`; catalog-default override-able pricing; partial payments + netted customer balance; walk-ins; agreed payment method at registry. Backend module + 54 tests; frontend `orders` module (operator entry wizard + queue + detail + driver delivery view). | 2026-06-10 |
| [[customers/customers-crud/index\|customers-crud]] | customers | backend ✓ · frontend ✓ | Customer registry (CRUD + name/phone search, soft delete) + debt visibility: empty-tank debt + monetary balance (new `customer_debts` table + view); hardens the soft `customer_empty_debts` FK. Frontend: `customers` module — list (search + debt badges), create/edit form, detail with debt summary. | 2026-06-08 |
| [[inventory/inventory-foundation/index\|inventory-foundation]] | inventory | backend ✓ · frontend ✓ | Ledger-first inventory: unified holders (location + driver-day), transfers on open/carry, empty-debt ledger, reconciliation, per-user/date history. Frontend: `inventory` module — assignments list/detail with balances, ledger-entry dialogs (sale/return/load/adjust), close+discrepancies, store availability + purchases. | 2026-06-06 |
| [[stores/stores-and-catalog/index\|stores-and-catalog]] | stores | backend ✓ · frontend ✓ | Reference data (stores, assignments, tank types, items) + seed + read API; tabbed admin catalog UI: list + create tank types/items, read-only stores. | 2026-06-05 |
| [[users/users-crud/index\|users-crud]] | users | backend ✓ · frontend ✓ | Manage existing users (list/filter, view, partial PATCH of name/phone/role/active) with role-assignment + self-lockout guards. Backend module + tests; frontend admin UI (shadcn-vue) with a guard-mirroring edit form. | 2026-06-04 |
| [[auth/auth-foundation/index\|auth-foundation]] | auth | backend ✓ · frontend ✓ | Login + JWT + role-based auth (developer / admin / operator / delivery), invitations, BOOTSTRAP_TOKEN-gated developer creation, Redis logout blocklist. Frontend login slice + token-aware client. | 2026-05-08 |
| [[bootstrap/v2-skeleton/index\|v2-skeleton]] | bootstrap | backend ✓ · frontend ✓ | Reset both repos to clean v2 skeletons. Backend: Express + Drizzle + Docker + GHCR + VPS pipeline. Frontend: Vue 3 + Vite + Pinia + shadcn-vue, module-by-domain, 38 KB gzipped, login flow live. | 2026-05-08 |

## Categories

Spec categories are created on-the-fly under `specs/{category}/`. Each new spec is a folder (`{slug}/` with `index.md` + track files). Current categories:

- `bootstrap/` — v2 reset of **both** repos, merged into one folder spec (done). Replaces the former separate `frontend-bootstrap/` category.
- `auth/` — login + invitations + middleware; backend + frontend tracks (done 2026-05-08)
- `users/` — manage existing users — done 2026-06-04 (backend + frontend tracks)
- `inventory/` — created 2026-05-07 with `inventory-foundation` (approved 2026-06-05)
- `stores/` — created 2026-06-05 with `stores-and-catalog` (foundational reference data; unblocks inventory)
- `customers/` — created 2026-06-07 with `customers-crud` (registry + debt visibility; unblocks orders)
- `orders/` — created 2026-06-08 with `orders-foundation` (done 2026-06-10; the money-recording workflow; depends on inventory + customers)
- `deployment/` — to be created when the first spec in each lands

- `ui-design/` — created 2026-06-12 with `design-system` (design system + full UI overhaul; frontend-only)
- `admin/` — created 2026-06-14 with `org-management` (cross-cutting admin console: users + stores + assignments + invites)

## Status Legend

- **draft** — In backlog, not ready for implementation
- **approved** — In backlog, ready (use `/focus {category}/{slug}`)
- **in-progress** — Being implemented (at least one track started, not all done)
- **done** — Completed across all tracks, permanent reference
- **cancelled** — Abandoned, kept for context

Per-track status (in each folder's Tracks table): `not-started | in-progress | done | cancelled`.

## Porting order (recommended)

Backend modules drive the order; each frontend track ports right after its backend counterpart is `done`.

1. `auth-foundation` — done (backend + frontend login slice, delivered under [[bootstrap/v2-skeleton/frontend|the frontend skeleton]]). Future: invite-completion + change-password screens (separate spec).
2. `users-crud` — done (backend module + frontend admin UI; seeds test users via UI).
3. `stores-and-catalog` — small; unblocks inventory.
4. `inventory-foundation` — the product's core.
5. `customers-crud` — small; unblocks orders.
6. `orders-foundation` — the user-visible business workflow.
7. `first-vps-deploy` — once there's something worth deploying.
