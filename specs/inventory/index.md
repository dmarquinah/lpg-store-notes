# Inventory — lpg-store

---
project: lpg-store
domain: specs
category: inventory
last-updated: 2026-05-07
---

## Context Documents

Read these before working on inventory specs:

- [[../../eng/architecture]] — backend layout and conventions
- [[../../eng/legacy-bloat-analysis]] — why v1 grew so much; the ten drivers v2 must avoid
- [[../../eng/decisions]] — ADR-005 through ADR-010 are the inventory-shaping decisions
- [[../../eng/patterns/module-template]] — vertical module folder convention
- [[../../product/overview]] — daily workflow, tank exchange model, MVP scope
- `lpg-backend/legacy/docs/inventory/PRD - Inventory.md` — v1 PRD (reference for requirements; ignore the implementation status sections)
- `lpg-backend/legacy/src/db/schemas/inventory/` — v1 schema (reference for entity shapes; v2 storage model differs per ADR-007)
- `lpg-backend/legacy/src/services/inventory/` — v1 service code (reference; do not port the strategy pattern)

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[inventory-foundation]] | draft | Ledger-first inventory with three-state assignment workflow, six transaction kinds, customer empty-debt ledger, reconciliation/discrepancy reporting. The product's core. |

## Edge cases the architecture must natively express

These were validated against v1 and the user's domain knowledge before drafting `inventory-foundation`. Any spec that touches inventory must remain compatible with all of them:

- **E1** Take 1 full, return 0 empty (debt accrues): inventory `{−1 full, 0 empty}`; customer `+1 empty owed (type X)`.
- **E2** Take 1 full, return 1 empty (normal swap): inventory `{−1 full, +1 empty}`; customer unchanged.
- **E3** Take 1 full, return 2 empties (settles prior debt): inventory `{−1 full, +2 empty}`; customer `−1 empty owed`.
- **E4** Walk-in cash sale, no customer record: same as E1 on inventory; nothing on the customer side.
- **E5** Customer brings empty back later, no purchase: inventory `{0 full, +1 empty}`; customer `−1 empty owed`.
- **E6** Driver receives day-opening assignment: ledger `opening` rows from store snapshot.
- **E7** Failed delivery returns tanks to driver: reversal transaction(s) on the same assignment.
- **E8** Supplier delivers fulls in exchange for empties: store-side `purchase` deltas.
- **E9** Damaged/lost tank: `adjustment` ledger row with note.
- **E10** Day-end consolidation: next day's opening = today's closing fulls/empties; empty-debt does **not** carry (lives on customers, not assignments).
- **E11** Reconciliation discrepancy at day-end: physical count vs ledger mismatch logged as `reconciliation` transaction.
