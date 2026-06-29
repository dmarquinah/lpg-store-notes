---
project: lpg-store
domain: specs
type: spec-track
spec: push-notifications
repo: lpg-backend
kind: backend
track-status: done
last-updated: '"2026-06-16"'
---

# Push Notifications — lpg-backend track

Shared spec: [[index]] · Frontend: [[frontend]]

## Technical Notes

A new **`notifications`** vertical module (mirrors the existing module template)
plus two **injected notifier ports** consumed by the inventory and orders
modules. Following **ADR-012** (explicit composition over events), the notifier
is wired in `app.ts` and handed to the triggering services — `notifications` is
never imported by `inventory`/`orders`, so there's no dependency cycle (same
late-bind pattern already used for the accounting↔inventory `AccountedPeriodGuard`).

### Dependency: web-push

Add the **`web-push`** npm package (the standard Web Push protocol + VAPID
signer). No Firebase. One-time key generation: `npx web-push generate-vapid-keys`
→ produces the public/private pair stored in env.

### Env / VAPID

- `VAPID_PUBLIC_KEY` — base64url public key (also shipped to the client as
  `VITE_VAPID_PUBLIC_KEY`; publishable).
- `VAPID_PRIVATE_KEY` — **the secret**; backend env only, never logged, never
  returned by any endpoint.
- `VAPID_SUBJECT` — a `mailto:` or site URL required by the spec.
- Call `webpush.setVapidDetails(VAPID_SUBJECT, VAPID_PUBLIC_KEY, VAPID_PRIVATE_KEY)`
  once at module construction. If the keys are absent (e.g. local dev without
  push), construct the module in a **no-op send** mode rather than crashing
  boot — sends become best-effort no-ops, subscriptions still store.

### Schema + migration

`push_subscriptions`:

- `id` serial PK
- `user_id` → `users.id` (FK, indexed)
- `endpoint` text **unique** (the natural key; re-subscribe upserts on it)
- `p256dh` text, `auth` text (the subscription's encryption keys)
- `user_agent` text null, `platform` text null (optional diagnostics)
- `created_at`, `last_seen_at` timestamptz

Generate the migration **named** (project rule): `npm run db:generate --
push_subscriptions`, then apply with the migrator.

### Endpoints (all authenticated; `/api/v1/notifications`)

- `POST /subscriptions` — body `{ endpoint, keys: { p256dh, auth }, platform? }`.
  **Upsert by `endpoint`** (`onConflictDoUpdate` → refresh `user_id` +
  `last_seen_at`), bound to the caller. Idempotent: re-subscribing the same
  endpoint never duplicates and re-binds it to the current user (handles a shared
  device).
- `DELETE /subscriptions` — body `{ endpoint }`; delete the caller's matching
  subscription (logout / disable). 204 even if already gone.
- *(optional)* `GET /vapid-public-key` — returns `VAPID_PUBLIC_KEY` if the team
  prefers a runtime fetch over the client env var. Not required (frontend bakes
  it in).

A user may only create/delete subscriptions for **themselves** (derive `user_id`
from the auth context, never trust a body field). No endpoint returns the private
key or another user's rows.

### Service: send + prune

`NotificationsService`:

- `notifyUser(userId, payload: { title, body, data?: { url?, orderId? } })` —
  load the user's subscriptions; for each, `webpush.sendNotification(sub,
  JSON.stringify(payload))`. **Best-effort / fire-and-forget**: wrap in
  try/catch per subscription; a send failure must **never** propagate into the
  triggering request (the day still opens / the order still assigns).
- **Prune on Gone:** if `sendNotification` throws with `statusCode` 404 or 410,
  delete that subscription row (dead endpoint) so they don't accumulate.
- Keep the payload small and JSON; the SW reads `title`/`body`/`data.url`.

### Triggers (injected ports, wired in app.ts)

Define a minimal port type the consumers depend on, e.g.
`type DriverNotifier = { notifyUser(userId, payload): Promise<void> }`. In
`app.ts`, after both modules exist, inject the notifications service into:

- **inventory** — wherever a day is **opened/assigned to a driver** (`openDay` /
  the assignment-creation path). Resolve the target **driver user id** for the
  assignment and call `notifyUser(driverId, { title: 'Nueva asignación del día',
  body: '…', data: { url: '/mi-dia' } })`.
- **orders** — `assignOrder` (and/or `dispatch`), where an order gains its
  assigned driver. Call `notifyUser(driverId, { title: 'Nuevo pedido para
  entregar', body: '…', data: { url: `/mis-entregas/${orderId}` } })`. The
  `order-event-timeline` work already resolves the driver on assign — reuse that
  resolution rather than re-querying.

Inject via setters (`setNotifier(...)`) so the modules construct standalone
(tests / no-push dev) with the notifier unset → trigger is a silent no-op.

### Reuse / don't reinvent

- Module shape: follow [[../../eng/patterns/module-template]] (types/schema/
  repository/service/routes/app-wiring).
- Composition/injection: copy the `AccountedPeriodGuard` late-bind pattern (see
  the `provider-purchase-cost` backend track) — setter on the consumer service,
  wired in `app.ts` after notifications is built.
- Errors: `src/lib/errors.ts`. Auth/role guard middleware as elsewhere.

## Related Files

> Confirm exact paths at `/focus` time — these are known by role from the v2
> module conventions, not yet read in this planning pass.

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To create:

- `src/modules/notifications/{schema,types,repository,service,routes}.ts` — the
  new module (`push_subscriptions`, subscribe/unsubscribe, `notifyUser`, VAPID
  send via `web-push`).
- `src/db/migrations/00NN_push_subscriptions.sql` — generated + applied.
- `src/modules/notifications/__tests__/notifications.test.ts` — upsert
  idempotency, `410` prune, empty-subscription no-op, trigger calls the port.

To modify:

- `src/app.ts` — build the notifications module; inject its notifier into
  inventory + orders (late-bind, ADR-012).
- `src/modules/inventory/service.ts` — call the injected notifier when a day is
  opened/assigned to a driver (setter `setNotifier`).
- `src/modules/orders/service.ts` — call the injected notifier on
  assign/dispatch (reuse the driver resolution from `order-event-timeline`).
- env/config loader + `.env.example` — `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY`
  / `VAPID_SUBJECT`.
- `package.json` — add `web-push` (+ `@types/web-push`).

Read-only context:

- The `provider-purchase-cost` backend track — reference implementation of the
  injected-port / `app.ts` late-bind pattern.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-backend] Backend track implemented and all criteria met. Quality gate green (typecheck / biome check / 100 tests / build). Independent validation passed all 7 criteria + 4 risk checks, no gaps.

- **Dependency/env:** added `web-push` + `@types/web-push`. `src/config/env.ts` gained optional `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_SUBJECT`; `.env.example` documents them + `npx web-push generate-vapid-keys`.
- **Module `src/modules/notifications/`:** `schema.ts` (`push_subscriptions`: serial PK, `user_id` FK→users `on delete cascade` + index, **unique `endpoint`**, `p256dh`, `auth`, nullable `user_agent`/`platform`, `created_at`, `last_seen_at`); `repository.ts` (`upsertByEndpoint` via `onConflictDoUpdate` on endpoint → rebind user + bump `last_seen_at`; scoped `deleteByEndpoint(userId,endpoint)`; `listByUser`; `deleteByEndpointAny` for prune); `service.ts`; `routes.ts` (`POST`/`DELETE /subscriptions` both `requireAuth`, userId always from caller; `GET /vapid-public-key` returns only the public key); `index.ts` factory.
- **Migration:** `0013_push_subscriptions.sql` generated (named) + applied to local Postgres; registered in `src/db/schema.ts`.
- **Triggers (ADR-012 late-bind, mirrors `setAccountedPeriodGuard`):** each consumer owns a local `DriverNotifier` port type + `setNotifier()`; neither inventory nor orders imports the notifications module (grep-verified). Wired in `app.ts` after the module is built. `inventory.openDay` fires "Nueva asignación del día" → `/mi-dia`; `orders.assignOrder` fires "Nuevo pedido para entregar" → `/mis-entregas/:id` (reuses the already-resolved `targetUserId`). **Not** fired on dispatch (owner-confirmed). Both fire **after commit** as `void notifier?.(...).catch(()=>{})`, so a push failure can never roll back or break the triggering request.
- **Design decision — injectable `PushSender`:** the side-effecting send is a `PushSender` function injected into `NotificationsService` (default wraps `web-push`). Absent VAPID env → sender is `null` → **no-op send mode** (subscriptions still store; sends are skipped). This single seam doubles as the test double, so the suite + `npm run dev` need no VAPID keys or network.
- **Local testing answer (owner's pre-impl question):** backend `npm run dev` is sufficient — it's an outbound sender, so local backend → vendor push service → local browser works with no tunnel/Docker; only the three VAPID env vars are needed to actually send. `node --test` covers subscribe/prune/no-op/trigger headlessly. iOS Safari is the one path that can't be tested on localhost (needs an installed PWA over real HTTPS).
- **Tests (`__tests__/notifications.test.ts`, lifecycle-style):** upsert idempotent + shared-device rebind; notifyUser no-op when empty + fan-out payload; 410 → prune; no-op send mode; `inventory.openDay` fires the port with `/mi-dia`. Existing suites stay green (100 pass).
- **Note:** `web-push` pulls transitive `qs`/`shell-quote` advisories via its CLI (yargs) tree — not on the runtime `sendNotification` path, so they don't ship in the distroless image.

[2026-06-16] [lpg-backend] All criteria for this repo met. Backend track done; overall spec stays `in-progress` until the frontend (lpg-frontend-vue) track lands.
[2026-06-16] [lpg-backend] Backend track **done** (implemented outside this session; **verified** during the frontend `/focus`). Confirmed against the working tree: new `notifications` module (`schema/types/repository/service/routes/index.ts` + `__tests__/notifications.test.ts`), migration **`0013_push_subscriptions`**, `web-push@^3.6.7` (+ `@types/web-push`), and `VAPID_PUBLIC_KEY`/`VAPID_PRIVATE_KEY`/`VAPID_SUBJECT` as **optional** env in `config/env.ts` + `.env.example`. **Contract the frontend builds to:** mounted at **`/api/v1/notifications`** — `POST /subscriptions` `{ endpoint, keys:{p256dh,auth}, platform? }` → `201 {ok:true}` (upsert on unique `endpoint`, caller-bound); `DELETE /subscriptions` `{ endpoint }` → `204` (idempotent, caller-scoped); `GET /vapid-public-key` → `{ publicKey }`. `notifyUser` is best-effort (never throws), fans out + prunes on `404`/`410`; no-op send mode when VAPID unset. **Triggers (ADR-012 injected ports, wired in `app.ts`):** day-open → `{ title:'Nueva asignación del día', data:{ url:'/mi-dia' } }`; order-assign → `{ title:'Nuevo pedido para entregar', data:{ url:'/mis-entregas/:id', orderId } }`. Notifications tests pass 4/4 (`node:test`). All 7 backend criteria satisfied. Frontend track now in progress.