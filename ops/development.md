# Development — lpg-store

---
project: lpg-store
domain: ops
last-updated: 2026-05-07
---

## Backend (lpg-backend)

### Local setup

```bash
cd lpg-backend
cp .env.example .env             # adjust as needed
docker compose up -d db redis    # postgres + redis only
npm install
npm run db:migrate               # apply migrations
npm run dev                      # tsx watch
```

Then `curl localhost:3000/health` returns `{"status":"ok","db":"ok"}`.

### Scripts

| Command | What it does |
|---------|--------------|
| `npm run dev` | tsx watch on `src/server.ts` |
| `npm run typecheck` | `tsc --noEmit` |
| `npm run build` | TypeScript → `dist/` (tsconfig.build.json) |
| `npm start` | `node dist/server.js` |
| `npm test` | Node's built-in test runner against `src/**/*.test.ts` |
| `npm run db:generate` | `drizzle-kit generate` — produce a migration from schema diffs |
| `npm run db:migrate` | apply pending migrations |
| `npm run db:studio` | Drizzle Studio in the browser |

### Environment variables

See `.env.example`. Key ones:

- `DATABASE_URL` — postgres connection string
- `REDIS_URL` — optional; cache disabled if unset
- `JWT_SECRET` — must be at least 32 chars
- `LOG_LEVEL` — fatal / error / warn / info / debug / trace

### Validating the prod image locally

```bash
docker compose up -d --build     # builds from local Dockerfile, brings up
                                 # db + redis + migrate (one-shot) + app
```

`migrate` exits before `app` starts; compose enforces ordering via
`depends_on: condition: service_completed_successfully`.

## Frontend (lpg-frontend-vue)

Setup not yet documented here. Update when the frontend's dev workflow stabilizes.

## Bot (lpg-bot)

Setup not yet documented here. Update when the bot's dev workflow stabilizes.
