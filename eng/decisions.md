---
last-updated: '"2026-06-05"'
---
# Architecture Decisions — lpg-store

---
project: lpg-store
domain: eng
last-updated: 2026-05-07
---

Append-only log. Newest entries at the top.

---

## ADR-019 — Production runs behind a Traefik edge proxy; deploy config centralized under docker/ + infra/, prod-only compose, no Redis (2026-06-24)

**Status:** Accepted

**Context:** The product goes live on one 2 GB self-hosted VPS. The prior setup assumed an already-usable host and published the backend on `:3000` directly — no TLS, no reverse proxy, the frontend (Piloto) not deployed at all, and (despite `docs/DEPLOYMENT.md` + the vault claiming a full CI→GHCR→VPS pipeline) **no actual build/push/deploy job existed** in `main.yml`. The owner wanted both apps reachable over HTTPS on real subdomains, deploy config centralized so a future VPS migration is trivial, the frontend served as its own nginx image (not Node), no Redis on the small box, and the env-file handling fixed (compose kept ignoring a `.env.prod` and defaulting to `.env`).

**Decision:**
- **Traefik edge.** A single Traefik v3 service terminates TLS (**automatic Let's Encrypt**, HTTP-01 challenge), redirects HTTP→HTTPS, applies a shared `secure-headers` middleware (HSTS/nosniff/frameDeny/referrer), and routes by host via docker-provider labels — `api.<domain>` → `api:3000`, `app.<domain>` → `frontend:80`. Chosen over a hand-written nginx edge for auto-TLS + label routing. The backend **stops publishing host ports** (reachable only via Traefik on the internal network); `db` is internal-only.
- **Centralized, prod-only compose.** All Docker/infra config moved off the repo root into `docker/` (`Dockerfile` + the single `docker-compose.yml`) and `infra/` (Traefik `dynamic.yml` + `.env.prod.example`). **There is no dev compose** — local dev runs the app on the host against a host Postgres, so the compose file is purely a production artifact. `.dockerignore` (build-context-relative) stays at the root. This **deviates from the earlier "one compose file for dev+prod, env-switched" working preference**: dev no longer uses compose, and prod is its own centralized file — migrating boxes is "copy `docker/` + `infra/` + `.env`".
- **No Redis.** The JWT logout blocklist store is omitted on the 2 GB box; `logout` degrades to a **stateless client-side token drop** (204; the token expires on its own in 24 h) instead of a 503. Re-adding Redis later restores instant server-side revocation **with no code change** — the graceful no-op only triggers when no cache is configured.
- **Env handling.** The prod env is a file named exactly `.env` in `/srv/lpg-backend`, auto-loaded by compose for both `${VAR}` interpolation **and** container `env_file:` injection — eliminating the `--env-file`-ignored confusion (a file named `.env.prod` is never auto-loaded). Hosts/ACME-email/secrets are all env-driven; nothing is committed. `NODE_ENV=production` and `DATABASE_URL` are forced in `environment:` so a missing/lax `.env` can't silently boot dev mode or mispoint the DB.
- **CI is now real.** A `deploy` job builds `docker/Dockerfile` → GHCR (`:latest` + `:sha-<short>`) → `scp`s the compose file + `infra/` to the VPS → `pull && up -d --remove-orphans` → health-checks `https://api.<domain>/health` **through Traefik**. The `migrate` one-shot **reuses the same distroless image** (only the command differs) — no second, fatter build.

**Consequence:** Migrating to a new VPS = copy `docker/` + `infra/` + the `.env`, point DNS, bring up. The frontend is wired as a **profile-gated** service (`COMPOSE_PROFILES=frontend`), off until `lpg-frontend-vue` publishes its image — the open **frontend track** of `first-vps-deploy`; the cross-cutting "whole stack over HTTPS" criterion closes when that ships. Mounting the docker socket **read-only** into Traefik is an accepted single-box risk surface; a socket-proxy is a later hardening step. The first deploy is manual (fresh-box provisioning + first-developer bootstrap via a one-time `BOOTSTRAP_TOKEN` + `POST /api/v1/auth/bootstrap`); CI takes over for subsequent pushes. `app.set('trust proxy', 1)` was added so client IP/protocol are correct behind the edge.

**Amended (2026-06-24, same session) — more decoupled layout per owner review:**
- **Traefik split out of the app compose** into its own `infra/traefik/docker-compose.yml` (edge infra ≠ application). `docker/` now holds only the backend app: `Dockerfile` + `docker-compose.yml` (`db` + one-shot `migrate` + profile-gated `seed` + `api`).
- **Explicit networks** replace the default bridge: `lpg-edge` is a **shared external** network (`docker network create lpg-edge` once) carrying only what Traefik must reach (`api`, and the frontend from its own repo); `lpg-internal` is private to backend↔db. **`db` is never on the edge.** Traefik routes across separate compose projects via the docker socket + a per-service `traefik.docker.network=lpg-edge` label + `--providers.docker.network=lpg-edge`.
- **Frontend is NOT defined in lpg-backend** — it ships its own nginx image + compose from lpg-frontend-vue and just attaches to `lpg-edge` with its own labels. The earlier profile-gated `frontend` service was removed from the backend compose.
- **CI builds + pushes only; deploys are manual.** The SSH `deploy` job was replaced by a `publish` job (build `docker/Dockerfile` → GHCR `:latest`+`:sha-<short>`). The owner deploys by hand over SSH (`docker compose pull && up -d` from `/srv/lpg-backend`), so CI needs **no VPS secrets** — only the automatic `GITHUB_TOKEN`.
- **Prod catalog seed.** `seed.ts` was split so under `NODE_ENV=production` it seeds **only** the real product catalog (tank types + items) — never the demo users/store/customers — run via a profile-gated one-shot `seed` service mirroring `migrate` (`docker compose run --rm seed`).
- **Two runbooks:** `docs/VPS-SETUP.md` (one-time provisioning) + `docs/DEPLOYMENT.md` (per-release). Env split into a backend `.env` (`docker/.env.prod.example`) and a Traefik `.env` (`infra/traefik/.env.example`, just `ACME_EMAIL`), each auto-loaded from its own stack's directory.
- **VPS holds a git clone** of the repo; config changes apply with `git pull` (the gitignored `.env` files are untouched), each stack running from its own directory in the clone (`docker/`, `infra/traefik/`). Compose **project names are pinned** (`name: lpg-backend` / `lpg-traefik`) so volumes/networks stay stable regardless of run directory; a **read-only deploy key** gives the box git access when the repo is private. The backend compose service was also renamed `app` → **`api`** (avoids clashing with the `app.<domain>` frontend host; the frontend's single-origin nginx proxies `/api` → `api:3000` over `lpg-edge`).

## ADR-018 — Closing an accounting registry freezes a snapshot (derived-balance rule does not apply to financial periods) (2026-06-15)

**Status:** Accepted

**Context:** Everywhere else in this codebase, totals are **derived live** from the append-only ledger and never stored (ADR-007: "balance = SUM view"; ADR-009: the ledger *is* the audit trail). The `accounting-registry` spec adds a per-store **closing register** whose detail aggregates a period's ingress (delivered-order `order_payments`) and egress (provider purchases). Membership is keyed by the money event's **business date** — for ingress, the **payment's** `occurredAt` (ADR-decision in the spec: a payment can be recorded *after* delivery, even back-dated into a period). That creates a hazard the live-derive rule cannot handle: a payment recorded *after* a period is closed, but dated *inside* its window, would retroactively change last month's reported figures every time new data lands. "Closing the books" would mean nothing.

**Decision:** Closing a registry **freezes a snapshot**. `closeRegistry` (admin/developer only) computes the full breakdown once and persists it into an `accounting_registries.snapshot` (jsonb) column alongside `status='closed'` + `closedBy`/`closedAt`. A registry's detail is **live-computed while `open`** and **served verbatim from the frozen `snapshot` once `closed`** — never recomputed. Manual `accounting_entries` are blocked from insert/delete once closed (4xx), so the entries that feed the snapshot can't drift either. This **deliberately overrides ADR-007's derive-don't-store rule, scoped to closed financial periods only** — open registries still derive live, and the underlying ledgers (`order_payments`, `tank_transactions`) remain the untouched source of truth; the snapshot is a *frozen report*, not a second source.

**Consequence:** A future reader who sees the `snapshot` column must NOT "fix" it back to live-recompute — that silently reintroduces retroactive drift in closed periods. The freeze is the feature. Reopening a closed registry is intentionally unsupported (a closed period is immutable); if ever needed it gets an explicit admin path that re-snapshots. Cross-module reads stay composition-based (ADR-012): the breakdown reads `OrdersService.paymentsByMethodForStorePeriod` + `InventoryService.purchaseCostsForStorePeriod` rather than querying their tables from accounting. Egress valuation reads each holder's snapshot `purchase_price` — which required wiring item-holder price snapshotting in inventory (previously hardcoded `0.00`), so item egress is no longer silently zero. 84 backend tests (was 81); frontend track separate.

## ADR-017 — Two-signature day handoff: close (driver) and carry (operator) are role-split (2026-06-15)

**Status:** Accepted

**Context:** `inventory-foundation` made the whole day lifecycle (`open → closed → carried`) callable by the same `canWrite` (operator/delivery/admin) guard, so one operator typed the opening counts, typed the closing counts, eyeballed discrepancies, and ran both end-of-day actions. The execution was too manual and concentrated on one role; the person entering the numbers wasn't the person at the truck. The owner wanted the day-end split across the two people physically present, without losing ledger precision.

**Decision:** End-of-day is a **two-signature** event enforced by distinct role gates on the two existing methods — no new lifecycle states, ledger columns, or migration.
- **Close = the driver's physical-count attestation.** `POST /assignments/:id/close` is gated to `requireRole('delivery','admin')` (developer auto-passes), and the *service* additionally requires the caller to be the assignment's **own** driver (`storeAssignment.userId === caller.id`) or an admin/developer override; any other caller → 403. For the owning driver, `finalCounts` is **mandatory and complete** (every tank type with a non-zero balance must be counted) → 400 otherwise. Admin override may close without counts (recovering an abandoned day). The Zod `closeSchema` keeps `finalCounts` optional; completeness is enforced in the service so the override path can omit it.
- **Carry = the operator's confirmation.** `POST /assignments/:id/carry` is gated to `requireRole('operator','admin')` — **`delivery` removed**. Since close and carry require *different* roles, the two signatures are structural; admin is the only actor who can do both (the escape hatch).
- **Start-of-day stays simple but carry-forward-aware.** A new advisory read `GET /suggested-opening?storeAssignmentId=` returns the driver's last `carried` truck load (sum of that day's non-`transfer` rows) as the default opening — the operator edits before opening, so opening may still differ from yesterday's closing (ADR-014). `openDay` is **guarded**: it rejects (409) when the driver still holds any earlier `open`/`closed` (non-`carried`) day, naming the blocker(s), so a driver never holds two live truck-holders at once.
- **Workflow deviations are annotated in the ledger.** A day closed after its own calendar date (`assignment.date !== today`) writes an attributable zero-delta `adjustment` row noted `cierre tardío: …` (with `userId` = the closer, and a marker when an admin closed it instead of the driver). Per ADR-009 the ledger *is* the audit trail; this is the seed for a future admin signal of which drivers aren't following the daily flow, so they can be coached.

**Consequences:** Role enforcement lives at the middleware (role) + service (row-level ownership) seam, mirroring `orders-multi-location`'s 403-vs-409 split (ADR-016). `carryAssignment` logic (idempotency, paired hand-back `transfer`, reconstruction) is untouched; only its caller-role changed. The late-close annotation overloads the existing `adjustment` kind (distinguished by note prefix) rather than adding a kind/column. Frontend work (driver "marcar como contado" + operator "Consolidar" review + carry-forward open form) is a separate track. 80 backend tests (was 70).

## ADR-016 — Optimistic concurrency for order state via conditional UPDATE (2026-06-14)

**Status:** Accepted

**Context:** `orders-foundation` moved an order's state with a read-status → check-in-service → write pattern across the connection pool. Two concurrent callers (e.g. two operators both assigning a `pending` order) could each read `pending`, each pass the in-service state-machine check, and both write — a classic lost-update race the status-machine 409 does *not* prevent. With multi-operator branches (`orders-multi-location`) this became a real hazard, not a theoretical one.

**Decision:** Every order state move is a **compare-and-swap**: a single conditional `UPDATE orders SET status = <to>, …patch WHERE id = ? AND status = <expected> RETURNING *`, where `<expected>` is the status read at the start of the operation. 0 rows affected ⇒ someone already moved it ⇒ raise `ConflictError` (409 "el pedido ya fue actualizado"). Implemented as `OrdersRepository.transition(id, fromStatus, toStatus, patch?)`; the service wraps it in `conflictIfLost`. Ownership/store *authorization* stays a **pre-check** (403/404) and is deliberately kept OUT of the `WHERE` clause, so a 0-row result unambiguously means "lost the race" (409) and never "not allowed" (403) — the two error semantics can't collapse into one indistinguishable failure. For `deliver`, the conditional flip to `delivered` is the **first** write inside the cross-module `db.transaction()` (ADR-012), so a losing concurrent deliver aborts the whole unit of work before any inventory/payment/debt write lands.

**Consequences:** No row locks / `SELECT … FOR UPDATE` needed; the guard is the `WHERE status =` predicate. The compare value is the exact status read (tightest CAS) so the audit `from_status` is always correct. Callers see a clean 409 on conflict and can retry. Tests exercise the guard via the conditional primitive (the in-memory fake mirrors it); a true DB-level race is asserted at the SQL `WHERE` level, not via OS threads. Replaced the old `setStatus`/`setAssignment` repo methods entirely.

## ADR-015 — Inventory never goes negative; open-day is today-only (2026-06-06)

**Decision**:
- **No over-assignment.** `openDay` / `recordLoad` validate against the location's current balance — you cannot move more fulls/empties onto a truck than the shop holds (→ 409). Assignment never drives location stock negative.
- **Purchase swap is bounded.** A tank purchase honors the empty-for-full swap only up to empties on hand: each line's `emptyReturned` defaults to `min(qty, emptiesOnHand)` and is capped at `qty` and at availability (→ 400/409 if exceeded), so a purchase never drives empties negative. The shortfall (fulls bought beyond empties returned) is paid via a `surcharge`, recorded in a `purchase_surcharges` **side table** keyed by the purchase transaction (ADR-013 pattern), not on the ledger.
- **Open-day is today-only.** The assignment date must equal today's Lima business date (central date service); past and future are rejected (→ 400). "Today" is injected into the service so multi-day flows stay testable.

**Why**: Physically impossible states — negative shop stock, returning empties you don't have, opening a day that hasn't started — surfaced in the UI as nonsensical negatives and arbitrary dates. The shop manages stock at the location and runs every day; the next day is opened explicitly when it arrives (ADR-014).

**Consequence**: `openDay`/`recordLoad`/`recordTankPurchase` do a balance read before writing inside the same DB transaction. The surcharge captures the cost of buying fulls without enough empties to swap. Backfilling a past day is intentionally unsupported here; if ever needed it gets its own explicit admin path.

---

## ADR-014 — Carry consolidates to the location; next-day opening is an explicit, restockable decision (2026-06-05)

**Decision**: `closeAssignment` reconciles physical counts (appends `reconciliation` rows) and marks the day `closed`. `carryAssignment` then **consolidates**: it hands the truck's day-end leftovers back to the **location** holder (paired `transfer`) and marks the assignment `carried` — and nothing more. It does **not** create or pre-fill the next day's assignment. The next day is opened **explicitly** via `openDay` with operator-chosen quantities, *after* the operator restocks the location from the provider (a `purchase` on the location holder). Next-day opening is therefore **not** constrained to equal today's closing.

**Why**: End-of-day reconciliation is a joint operator + delivery verification of the level. Afterward the location responsible (operator) requests more full tanks from the provider so the shop can function the next day; the next day's load is a fresh decision based on the **restocked** shop level, not a copy of yesterday's leftovers. The first implementation auto-reopened the next day with the previous closing quantities, which contradicted this workflow. This also corrects the E10 catalogue note ("next day opening = today's closing"), which was an oversimplification.

**Consequence**: The location holder is the single source of overnight stock — `openDay` debits it, the day-end hand-back credits it, restock `purchase`s credit it, and the next `openDay` debits it by the chosen load. `carry` stays a pure consolidation step. The three-state lifecycle (ADR-008) is unchanged (`open → closed → carried`), but `carried` now means "leftovers returned to the location," not "next day created" — amending ADR-008's note about next-day creation at carry-time. (A future stock-level alert when the location runs low is a natural add-on, not part of this spec.)

---

## ADR-013 — Location is a first-class inventory holder; the driver day-assignment is a stock bucket (2026-06-05)

**Decision**: Inventory is owned by a **holder**, of which there are two kinds: (1) **location stock** — a store's standing inventory per tank/item type (the shop's own stock between days), and (2) **driver day-assignment** — one driver's daily bucket (the existing `open | closed | carried` lifecycle, ADR-008). The append-only ledger (`tank_transactions` / `item_transactions`) scopes every row to a holder. Supplier `purchase` and shop-level `adjustment` rows hit the **location** holder. **Opening** a driver's day is a `transfer` (−qty from location, +qty onto the assignment); `sale` / `return` hit the **assignment**; **closing** reconciles the assignment's physical counts and `transfer`s the leftovers (unsold fulls + collected empties) back to the location. Both "current levels" are the *same* `SUM`-over-holder view (ADR-007): the driver's level / daily delta = his assignment ledger; the location availability the operator reads = the location-stock holder (optionally + that store's open assignments for stock currently out on trucks). Realized by generalizing the assignment catalog tables into **holder tables keyed by holder kind (`location | assignment`)** — see the `inventory-foundation` spec for the table layout.

**Why**: Restocking purchases land at the location *as a whole*, not on any one driver, and the operator works a queue (bank-teller model) reading **location-level availability** to assign each incoming order to a driver (manual now, automatable later). The original assignment-only six-table model had no home for stock that isn't on a driver, and ADR-007's "per-store total = `GROUP BY` over assignments" cannot represent shop stock or driver-agnostic purchases. Making the location a holder fixes this while keeping one uniform ledger. The truck-is-a-bucket choice (vs. opening-as-mere-allocation) keeps a single mental model — *every* level is a holder + a `SUM` — so daily driver accountability falls out of the assignment ledger instead of bespoke allocation math.

**Consequence**: `transfer` (already in the kind enum, ADR-006) and the open/close lifecycle (ADR-008) are reused — opening and closing write **paired** transfer rows (−location / +assignment) in one DB transaction, cross-linked by `refTransactionId`. The driver is **not** a location; the assignment remains a daily delivery record that *draws from* the location. Scales to N drivers per location (N assignment holders transferring from one location holder); one driver today is N=1. Table count is unchanged — `assignment_tanks`/`assignment_items` generalize into `tank_holders`/`item_holders` rather than adding tables. **Amends ADR-007** (the location is its own first-class holder, not a `GROUP BY` over assignments).

**Upgrade path** (additive, mirrors ADR-007's view path). The unified holder table stays narrow. When a holder *kind* needs its own attributes, or a new kind appears (truck/vehicle bucket, customer consignment, quarantine), **do not widen `tank_holders`/`item_holders`, and never widen the ledger** — extend via a **side table keyed by the row id**: kind-specific holder attributes → `*_holder_details(holder_id PK → holders.id, …)` (promotes the single holder table into a supertable + class-table-inheritance details; the ledger's `holder_id` FK is unchanged, so **zero ledger migration**); kind-specific transaction metadata → a side table keyed by `tank_transactions.id`. Core tables carry only universal hot-path columns; everything optional or kind-specific is a sidecar. This keeps the append-only ledger's shape frozen forever while the model extends, and the promotion is reversible-grade cheap because it never touches ledger rows.

---

## ADR-012 — Cross-module work uses explicit service composition; in-process events deferred (2026-06-05)

**Decision**: Cross-module transactional flows (`orders ↔ inventory ↔ customers`) are wired by **explicit service composition**: the dependent module's service is injected as a `dep` via the `createXxxModule({ db, deps })` factory and called **inside a single `db.transaction()`**. No in-process `EventEmitter` / event bus for these flows. An event mechanism is reserved for genuinely **non-transactional side-effects** (notifications, webhooks, cache invalidation) and is introduced only when a concrete consumer actually exists.

**Why**: The core flows must be atomic — `recordSale` writes a `tank_transactions` row *and* a `customer_empty_debts` row in one DB transaction (ADR-010); order delivery must commit inventory transactions atomically with the order state change. A Node `EventEmitter` is fire-and-forget and cannot join a DB transaction, so routing these across an event boundary forces outbox/saga machinery to *re-earn* the guarantee `db.transaction()` gives for free. Events also hide the call graph — reintroducing the "can't track the impact of a change across modules" pain that motivated the v2 reset (ADR-001). Same "no speculative infrastructure without a consumer" discipline as ADR-003 and ADR-009.

**Consequence**: Modules stay decoupled at the boundary (factory composition in `src/app.ts`) but invoke each other through explicit, greppable service calls. When a real fire-and-forget consumer appears (e.g. a notification on delivery), an emitter is introduced at that moment for that side-effect only — never for ledger-affecting writes.

---

## ADR-011 — Adopt shadcn-vue components; upgrade reka-ui to 2.x (2026-06-04)

**Decision**: The frontend's first real feature screen (`users-crud`) is built from **shadcn-vue** components scaffolded into `src/components/ui/` (button/input/label/select/table/badge/card/switch/alert) rather than hand-rolled Tailwind. To make the current shadcn-vue CLI work, `reka-ui` was upgraded `1.0.0-alpha.11 → 2.9.9` (it had no usages yet), `components.json` was migrated to the current schema (dropped the now-invalid `tsConfigPath`/`framework` keys), and `baseUrl`+`paths` for the `@/` alias were added to the **root** `tsconfig.json` so the CLI's resolver finds the alias (it reads `tsconfig.json`, while the app paths live in `tsconfig.app.json`). Existing `LoginView`/`AppLayout` were refactored onto the new components.

**Why**: The project `CLAUDE.md` commits to shadcn-vue, but the v2 skeleton shipped with no components scaffolded and an alpha `reka-ui` the current CLI rejects. Building the users UI by hand would have entrenched a parallel, non-shadcn styling path; aligning the toolchain once unblocks every later screen and keeps the component layer consistent.

**Consequence**: Main bundle grew ~38 → ~64 KB gzip (reka-ui primitives pulled in via the shared Button/Select), but feature views are code-split and Select's primitives sit in a shared lazy chunk. New screens compose from `@/components/ui/*`. v-model caveat: reka-ui's `AcceptableValue` (`string|number|bigint|object|null`) widens Select/Input v-model types — bind `:model-value` + `@update:model-value` with a coercion handler when the local ref is a narrow `string` (Switch's boolean v-model is fine).

---

## ADR-010 — Empty-tank debt is a customer-side ledger, not an inventory column (2026-05-07)

**Decision**: Customer obligations to return empty tanks live in a separate `customer_empty_debts` ledger keyed by `(customerId, tankTypeId)` with signed-delta rows. Each row links back to the inventory transaction that produced it (`refTankTransactionId`). The inventory ledger never represents fictitious empties — a sale where the customer did not return an empty is `{fullDelta: -1, emptyDelta: 0}`, full stop. Per-customer balances are derived via a SQL view in the same shape as `assignment_balance` (see ADR-007).

**Why**: v1 named this case (`order_items.tank_returned: boolean default true`) but never modeled it correctly. `TankSaleStrategy` always emitted `+1 empty` regardless of the flag, and `customer_debts` was monetary-only (`decimal amount`) and could not represent "this customer owes us 1 empty 10kg cylinder of type X". Without a typed ledger, partial returns ("they brought back 2 empties for 1 full to settle a prior debt") cannot be reconciled.

**Consequence**: Sale transactions write at most two ledger rows: one to `tank_transactions` (inventory side) and one to `customer_empty_debts` (customer side, only if the empty count returned is less than the fulls taken). The two ledgers are joined via `refTankTransactionId` for traceability. Walk-in (no customer) sales bypass the customer ledger entirely; the inventory side still records honest deltas. Empty-debt does **not** carry between daily assignments — it lives on customers, not on inventory.

---

## ADR-009 — Audit trail = transaction tables only (2026-05-07)

**Decision**: v2 has no `inventory_status_history`, no `audit_logs`. The `tank_transactions` and `item_transactions` ledgers, plus `order_status_history` (when the orders module lands), are the audit trail. A small `assignment_status_changes` log is added only if a UI surface actually needs it; default is no.

**Why**: v1's per-entity history table duplicated information already present in the transaction ledger and answered no business question that the ledger could not. Generic `audit_logs` had no consumer. Compliance-instinct schemas without consumers rot.

**Consequence**: Reports that previously read `inventory_status_history` will be derived from the transaction ledger. If an audit query needs status changes that are not implied by transactions (e.g., a manual override), it gets its own typed log at the time the override is added.

---

## ADR-008 — Three-state inventory workflow (open / closed / carried) (2026-05-07)

**Decision**: Inventory assignments have three states: `open` (driver has it, transactions accepted), `closed` (driver returned, day-end counts recorded, no further transactions), `carried` (system has created the next day's `open` assignment from this one's closing balances). Transitions: `open → closed → carried`. No `VALIDATED`, no `CONSOLIDATED`, no `OBSERVED`.

**Why**: v1's five states with separate `VALIDATED` (user said done) and `CONSOLIDATED` (system processed) modeled distributed coordination on a single-process, single-store, same-day shop. `OBSERVED` was a catch-all for things gone wrong that no UI surfaced. Stale recovery existed because the workflow had too many states and consolidation could fail silently.

**Consequence**: A late transaction arriving after `closed` is recorded as a reconciliation transaction on the (still-existing) day's assignment, with `kind = "reconciliation"`. The next-day's `open` is created at `carried`-time from a derived view over the previous day's ledger; if it has already been created, we no-op. The assignment row never becomes immutable.

---

## ADR-007 — Ledger + Postgres view; documented upgrade path (2026-05-07)

**Decision**: Inventory current quantities are derived, not stored. The authoritative tables are `tank_transactions` and `item_transactions` (signed-delta rows). A SQL `VIEW assignment_balance` exposes `(inventoryId, tankTypeId, currentFullTanks, currentEmptyTanks)` to callers; equivalent view for items. App code reads from the view as if it were a table. No `current*` / `assigned*` denormalized columns on the assignment-tanks/items tables.

**Why**: v1's denormalization (`currentFullTanks` etc. on `assignment_tanks`) was the root cause of consolidation, auto-routing, and stale-recovery complexity. Every transaction had to dual-write, and corrections required reaching back to patch a column. Ledger-first reverses the polarity: append-only writes, derived reads.

**Why a view**: With the SUM hidden in a view, callers do not know whether the balance is computed live, materialized, or snapshotted — so the storage strategy can be upgraded without touching application code.

**Upgrade path** (additive, non-disruptive):
1. **Today (1 store, dozens of tx/day per assignment)**: plain `VIEW` over the ledger. Index `(inventoryId, tankTypeId)` makes it sub-millisecond.
2. **If reads pressure shows up (~50+ stores, hot cross-store dashboards)**: convert to `MATERIALIZED VIEW` with `REFRESH MATERIALIZED VIEW CONCURRENTLY` on a cron tied to transaction-write rate.
3. **If matview refresh becomes a bottleneck**: replace the matview with a snapshot table maintained by the repository in the same DB transaction as the ledger write. Caller code is unchanged. The ledger remains the source of truth — the snapshot is a cache.

**Consequence**: No assignment row contains a balance. Reads go through `assignment_balance` (or the items equivalent). Writes go to the ledger; the repo never touches a balance column because there is none. Multi-store scaling is structural in the existing keying (`store_assignments → inventory_assignments → tank_transactions`); per-store totals are a `GROUP BY` over the same ledger. **[Amended by ADR-013 (2026-06-05): the location is now its own first-class inventory holder with its own balance view — supplier purchases and shop stock live on the location holder, not a `GROUP BY` over driver assignments. Driver assignments `transfer` stock from the location holder at day-open.]**

**Plain dates** (folded into this ADR): assignments key on `date` (not timestamp). "Next day" is `+1 day`. No business-day service, no weekend skipping. The shop runs every day; if that ever changes, a holidays table is the right answer, not a date service.

---

## ADR-006 — Function-first transaction kinds; no Strategy Pattern (2026-05-07)

**Decision**: Inventory transaction kinds (`sale`, `purchase`, `return`, `transfer`, `opening`, `adjustment`) are expressed as a small enum and a `kindToDelta(kind, qty, opts)` function. There is no `TransactionStrategy` base class, no `*Strategy` per kind, no `TransactionStrategyFactory`, no `TransactionProcessor`. The service has one method per business operation (`recordSale`, `recordPurchase`, …) that calls `kindToDelta` and writes via the repo.

**Why**: v1's 1,541 LOC of strategies dispatched on five sign-formulas. The polymorphism never paid off — these are not five behaviors, they are five tiny sign tables. Switch-case (or a function table) is the right tool when the variation is data, not behavior. See `legacy-bloat-analysis` driver 3.

**Consequence**: Adding a new kind = a new enum value + a new arm in `kindToDelta` + a new service method. Routes and types follow. Total LOC for a new kind: ~30 vs ~300.

---

## ADR-005 — Signed-delta repo API for inventory transactions (2026-05-07)

**Decision**: The transaction repository exposes one signed-delta write per entity:

```
applyTankDelta(assignmentTankId, fullDelta, emptyDelta, kind, userId, refs?, notes?)
applyItemDelta(assignmentItemId,  delta,                   kind, userId, refs?, notes?)
```

Plus `findAssignmentTank(inventoryId, tankTypeId)` / `findAssignmentItem(...)` for callers that have an `inventoryId` not an `assignmentTankId`. No `increment*` / `decrement*` × `byAssignment` / `byInventory` matrix.

**Why**: v1 declared the same operation across three orthogonal axes (driver 2). Sign is data; lookup style is one helper call away.

**Consequence**: One repo method per write path. Batch operations live in the service, not the repo (the repo's job is the single atomic write).

---

## ADR-004 — Zod-first types co-located in module/types.ts (2026-05-07)

**Decision**: Each module's `types.ts` is the single source of types: Zod schemas (`requestBody`, `responseBody`) plus `z.infer<>` aliases. No parallel `dtos/` tree. No TS `interface` declarations duplicating what a Zod schema describes. Routes call `schema.parse(req.body)` and pass the inferred type to the service.

**Why**: v1's 3,959-LOC `dtos/` tree declared types independently of the Zod schemas validating them at the route layer, so each shape had two definitions that drifted (driver 9). The Zod schema is authoritative; the inferred TS type is a free byproduct.

**Consequence**: Changing a request shape is one file. Validators and types cannot drift because there is only one declaration.

---

## ADR-003 — No `I*` interfaces unless polymorphism is real; no DI container (2026-05-07)

**Decision**: Services accept concrete repository instances. Routes accept concrete service instances. We do not declare `IFooService` / `IFooRepository` interfaces unless there is more than one implementation in the codebase. Tests substitute by passing a small in-memory class with the same method shape (TypeScript structural typing). Composition happens in `src/app.ts`; there is no DI container.

**Why**: v1 had 26 `I*` files for 26 single-implementation classes (driver 1). The custom DI container in `legacy/src/config/modules/*.ts` (467 LOC) was pure plumbing — `null`-init guards, `getDependencies()` accessors, factory pairs — none of which drove polymorphism.

**Consequence**: When a real second implementation appears (e.g., an in-memory cache repo for tests, a different transport), introduce the interface at that moment. The cost of adding it later is local; the cost of carrying it now is global.

---

## ADR-002 — Single vault project covers all repos (2026-05-07)

**Decision**: One Obsidian vault project (`lpg-store`) covers `lpg-backend`, `lpg-frontend-vue`, and `lpg-bot`. Each repo's `CLAUDE.md` declares `vault-project: lpg-store`.

**Why**: Most product features (orders, customers, inventory) span backend + frontend + (potentially) bot. A single project with cross-repo `Related Files` lists avoids duplicating spec content per repo. The global vault config explicitly supports this pattern.

**Consequence**: Specs that touch multiple repos list every relevant path. Backend-only or frontend-only specs still live in the same `specs/` board, just with narrower Related Files. Implementation notes get tagged `[YYYY-MM-DD] [repo-name] ...`.

---

## ADR-001 — Reset backend as v2, archive v1 (2026-05-07)

**Decision**: Build a clean v2 of the backend at the repo root with vertical modules; archive v1 under `legacy/` as read-only reference; delete `legacy/` once v2 reaches functional parity.

**Why**: v1 (~22.5k LOC, 7 modules, custom DI, 5-strategy transaction pattern, 998-LOC workflow repo) became hard to maintain — the user reported being unable to track impact of changes across modules. Pre-production state means no users to migrate; lessons live in PRDs + code, not data.

**Consequence**: Features are ported one at a time, each with a spec that points at v1 in `legacy/` for requirements and prior art. Repo grows incrementally; `legacy/` shrinks toward zero.

---
