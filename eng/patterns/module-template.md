# Module Template — backend vertical modules

---
project: lpg-store
domain: eng
last-updated: 2026-05-07
---

Convention for `lpg-backend/src/modules/<feature>/`. Every feature ported from v1 follows this shape.

## Folder layout

```
src/modules/<feature>/
  routes.ts             # express router; thin — calls service
  service.ts            # business logic; pure where possible
  repository.ts         # drizzle queries; ONLY this file imports schema.ts
  schema.ts             # drizzle table definitions for this feature
  types.ts              # zod schemas + inferred types
  index.ts              # createXxxModule({ db, deps }) — composition root
  __tests__/            # Node test runner tests
```

## Boundaries

- **`schema.ts`** is imported by `repository.ts` only. The rest of the app sees inferred types via `types.ts`.
- **`repository.ts`** exposes a class or factory that takes a `Database` instance. Public methods take/return inferred row types or `types.ts` types — never raw drizzle query builders.
- **`service.ts`** takes a repository instance plus any cross-module deps in its constructor/factory. Exports public methods that the router calls. No direct drizzle imports.
- **`routes.ts`** exports an `express.Router`. Route handlers do: validate body via zod → call service → respond. No business logic, no direct repository access.
- **`types.ts`** holds zod schemas (request/response bodies) and `z.infer<>` types. This is the only file other modules can import to share types.
- **`index.ts`** exports `createXxxModule({ db, deps })` returning `{ router, service }` (or just `{ router }`). The factory wires repository → service → router internally.

## Schema registration

Every module's `schema.ts` is re-exported from `src/db/schema.ts` so drizzle-kit picks it up:

```ts
// src/db/schema.ts
export * from "../modules/users/schema";
export * from "../modules/orders/schema";
// ...
```

After adding a module's schema, run `npm run db:generate` to produce a migration in `src/db/migrations/`.

## Composition

Modules are wired in `src/app.ts`:

```ts
import { createUsersModule } from "./modules/users";

const users = createUsersModule({ db });
app.use("/api/v1/users", users.router);
```

Cross-module deps (e.g. orders needs the inventory service) are passed in via the factory: `createOrdersModule({ db, inventoryService: inventory.service })`.

## Tests

Tests live under `__tests__/` next to the code they test. Use Node's built-in test runner:

```ts
import { describe, it } from "node:test";
import assert from "node:assert/strict";

describe("UserService.create", () => {
  it("rejects duplicate emails", async () => {
    // ...
    assert.equal(...);
  });
});
```

Mock at the repository boundary: services accept a repository instance, so tests pass an in-memory fake. No mock libraries needed for the simple cases.

## Reference for porting from v1

Each feature spec should list:
1. The v1 implementation files in `legacy/src/...` for prior-art reference.
2. The v1 PRD in `legacy/docs/...` for requirements.
3. The expected v2 paths under `src/modules/<feature>/`.
