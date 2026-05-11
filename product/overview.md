# Product Overview — lpg-store

---
project: lpg-store
domain: product
last-updated: 2026-05-07
---

## What the product does

A management system for a small-city LPG (Liquefied Petroleum Gas) tank delivery business. The owner needs to track the chain of custody: who has which tanks each day, what was sold to whom, when each delivery happened, and how unpaid orders accumulate.

The primary value is **traceability**, not analytics or accounting. The system replaces phone-and-WhatsApp coordination with a structured ledger that survives operator turnover.

## Users

- **Operator** (typically 1) — receives customer calls, creates orders, confirms them, dispatches to a delivery user. Reviews order history and unpaid balances.
- **Delivery user** (a few drivers) — sees orders assigned to them, marks deliveries in-transit / completed / failed, and (eventually) reconciles the day's inventory.
- **Admin** (often the same person as the operator at this stage) — manages user accounts, store catalog, and audits the trail when something goes wrong.

## Daily workflow

1. **Day start**: each delivery user gets an inventory assignment listing the tanks (full and empty) they're starting with.
2. **Order entry**: operator takes a customer call → enters customer name/phone, items, price → confirms the order, which reserves inventory.
3. **Delivery**: driver marks the order in-transit, attempts delivery, and marks completed (commits inventory transactions: full out, empty in) or failed (reservation released).
4. **Day end**: driver reviews inventory; system consolidates and carries quantities to the next day's assignment.

## Tank exchange model

Customers swap empty tanks for full ones. Inventory tracks both `fullTanks` and `emptyTanks` counts per assignment.

## MVP scope (kept)

- Single store now (with `stores` seeded with one row so multi-store is non-breaking later).
- Inventory assignments with start/consolidate cycle and auto-routing for late transactions on consolidated inventories.
- Inventory reservations to prevent overselling.
- Customer registry with debt tracking (paid / unpaid flag, outstanding balance).
- Audit trail via `order_status_history` + `tank_transactions` + `item_transactions`.

## Deferred / dropped

- Vehicles tracking (fuel, maintenance) — out of scope.
- Finance module (supplier orders, expenses, daily cash reconciliation) — out of scope; not what traceability needs.
- Generic `audit_logs` — redundant with the per-entity history tables.
- Push notifications (Firebase FCM) — was built in v1, dropped from v2 unless a clear delivery-confirmation use case re-emerges.
- Multi-store logistics, route optimization, advanced analytics — defer to post-MVP.

## Status

Pre-production. v2 backend skeleton in place; feature porting begins next.
