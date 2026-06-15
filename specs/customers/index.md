---
project: lpg-store
domain: specs
category: customers
last-updated: 2026-06-08
---

# Customers — lpg-store

Customer registry and debt visibility. Operators identify customers during a
phone call (search by name/phone), register them, and see what they owe —
both **empty-tank debt** (ADR-010, modelled in `inventory`) and **monetary
unpaid balance** (written by `orders`). This category unblocks `orders`.

## Context Documents

Read these vault docs before working on specs in this category:

- [[../../product/overview]] — daily workflow; "operator takes a customer call → enters customer name/phone"; customer registry with debt tracking is in MVP scope
- [[../../eng/decisions]] — **ADR-010** (empty-tank debt is a customer-side ledger), **ADR-009** (cross-module flows via explicit service composition; customers validates existence), test guidance
- [[../../eng/legacy-bloat-analysis]] — why v1 over-modelled customers; what v2 drops
- [[../../eng/patterns/module-template]] — backend vertical module convention
- [[../../eng/patterns/frontend-module-template]] — frontend vertical module convention

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[customers-crud/index\|customers-crud]] | done | Customer registry (CRUD + name/phone search, soft delete) with debt visibility: empty-tank debt (existing view) + monetary unpaid balance (new `customer_debts` table). Hardens the soft `customer_empty_debts.customer_id` into a real FK. |
