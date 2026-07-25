---
project: lpg-store
domain: specs
category: providers
last-updated: 2026-07-09
---

# Providers — lpg-store

Provider (supplier) registry and debt visibility — the **supply-side mirror** of
[[../customers/index|customers]]. Where customers model the money + empty-tank
debt owed **to us**, providers model what **we owe the supplier**: money (unpaid
purchases, partial payments allowed) and empty tanks (fulls bought without
returning the matching empties). Purchases become attributed to a named provider
with a per-provider price list. This category makes the purchase process
traceable end-to-end.

## Context Documents

Read these vault docs before working on specs in this category:

- [[../../product/overview]] — daily workflow; the two-way tank exchange (full-out / empty-in) applies to providers just as to customers; provider restock flow
- [[../../eng/decisions]] — **ADR-009** (append-only ledger as audit trail), **ADR-010** (empty-tank debt as a signed ledger — mirror for providers), **ADR-012/ADR-013** (cross-module flows via explicit service composition; `purchase` rows land on the store's `location` holder), **ADR-016** (atomic CAS transitions)
- [[../../eng/legacy-bloat-analysis]] — why v1 over-modelled entities; what v2 keeps lean
- [[../../eng/patterns/module-template]] — backend vertical module convention
- [[../../eng/patterns/frontend-module-template]] — frontend vertical module convention
- [[../customers/customers-crud/index]] — the registry + dual-debt pattern to mirror (empty-tank via inventory ledger, monetary via a debt table)
- [[../inventory/provider-purchase-cost/index]] — the existing per-purchase `unitCost` capture + default chain this spec extends (adds the provider price-list tier)
- [[../inventory/inventory-foundation/index]] — the ledger, holders, and edge cases E1–E11 (E8 is the provider-purchase case)
- [[../accounting/accounting-registry/index]] — egress = provider purchases; the credit-purchase interaction is an open question here

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[provider-management/index\|provider-management]] | done · backend ✓ frontend ✓ | Full provider registry (CRUD) + per-provider price list + dual debt tracking (money with partial payments; empty tanks). Attributes purchases to a named provider; empty-tank shortfall on a purchase accrues a tracked debt instead of a silent surcharge. Mirrors the customer registry + debt model on the supply side. |
