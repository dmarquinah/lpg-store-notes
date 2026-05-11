# Specs — lpg-store

---
project: lpg-store
domain: specs
last-updated: 2026-05-08
---

## In Progress

| Spec | Category | Summary | Started |
|------|----------|---------|---------|
<!-- Specs move here when /focus starts -->

## Backlog

| Priority | Spec | Category | Summary |
|----------|------|----------|---------|
| high | [[users/users-crud]] | users | User CRUD + role assignment. Required to seed test users for inventory + orders. |
| high | [[inventory/inventory-foundation]] | inventory | Ledger-first inventory: tank/item transactions, three-state assignment lifecycle (open/closed/carried), customer empty-debt ledger, reconciliation. The product's core. Shaped by ADR-005 through ADR-010. |
| medium | [[customers/customers-crud]] | customers | Customer registry with search + debt flag. Used by orders. |
| medium | [[orders/orders-foundation]] | orders | Order PENDING → CONFIRMED → IN_TRANSIT → DELIVERED → FULFILLED with reservations + cancellation. |
| medium | [[stores/stores-and-catalog]] | stores | Single-store seed + product catalog (tank types, items). Cheap; needed before inventory. |
| low | [[deployment/first-vps-deploy]] | deployment | First production deploy of the v2 skeleton to the VPS. Validates the CI/CD pipeline end-to-end. |

## Done (recent)

| Spec | Category | Summary | Completed |
|------|----------|---------|-----------|
| [[frontend-bootstrap/v2-skeleton]] | frontend-bootstrap | Archived v1 frontend to legacy/, bootstrapped v2 (Vue 3 + Vite + Pinia + shadcn-vue, module-by-domain, drop i18n + FCM + datepicker). 38 KB gzipped, login flow live. | 2026-05-08 |
| [[auth/auth-foundation]] | auth | Login + JWT + role-based auth (developer / admin / operator / delivery), invitations, BOOTSTRAP_TOKEN-gated developer creation, Redis logout blocklist. | 2026-05-08 |
| [[bootstrap/v2-skeleton]] | bootstrap | Archive v1 to legacy/, bootstrap v2 (Express + Drizzle + Docker + GHCR). | 2026-05-07 |

## Categories

Spec categories are created on-the-fly under `specs/{category}/`. Current categories:

- `bootstrap/` — initial v2 backend setup (done)
- `auth/` — login + invitations + middleware (done 2026-05-08)
- `frontend-bootstrap/` — frontend reset playbook (done 2026-05-08)
- `inventory/` — created 2026-05-07 with `inventory-foundation`
- `users/`, `customers/`, `orders/`, `stores/`, `deployment/` — to be created when the first spec in each lands

## Status Legend

- **draft** — In backlog, not ready for implementation
- **approved** — In backlog, ready (use `/focus {category}/{slug}`)
- **in-progress** — Being implemented
- **done** — Completed, permanent reference
- **cancelled** — Abandoned, kept for context

## Porting order (recommended)

Backend modules drive the order; each frontend module ports right after its backend counterpart is `done`.

1. `auth-foundation` — backend done; frontend login UI live in `frontend-bootstrap/v2-skeleton`. Future: invite-completion + change-password screens (separate spec under `auth-frontend/`).
2. `users-crud` — to seed test users (frontend module ports right after backend).
3. `stores-and-catalog` — small; unblocks inventory.
4. `inventory-foundation` — the product's core.
5. `customers-crud` — small; unblocks orders.
6. `orders-foundation` — the user-visible business workflow.
7. `first-vps-deploy` — once there's something worth deploying.
