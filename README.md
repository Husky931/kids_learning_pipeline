# yyddp

Course authoring, diff, and import stack for the YYDDP project. Two Node apps: `yyddp-api` (backend) and `yyddp-importer` (SQLite → PG pipeline + admin UI).

## Prerequisites (local dev)

- Node 20+ and pnpm (importer needs Node 24+ per its `engines`).
- PostgreSQL 16 running locally on `localhost:5432` (Postgres.app or `brew install postgresql@16`).
- Redis 7 running locally on `localhost:6379` (`brew install redis`).
- `.env.dev` at repo root — copy `.env.dev.example` and fill in OSS + JWT values.

Create a dev database once:

```bash
createdb yyddp_dev
```

## Running locally

Each service runs in its own terminal.

```bash
# Terminal 1 — yyddp-api (http://localhost:8888)
cd yyddp-api
pnpm install
pnpm dev

# Terminal 2 — yyddp-importer backend + Vite admin UI (http://localhost:3001 + http://localhost:5173)
cd yyddp-importer
pnpm install
pnpm run preview
```

Reachable from host:

- Admin (Vite): http://localhost:5173/admin/
- Importer API: http://localhost:3001
- API direct:   http://localhost:8888
- Postgres:     localhost:5432
- Redis:        localhost:6379

## Manual tests

- Dev smoke: [P0A_Manual_Testing.md](P0A_Manual_Testing.md)
- P0 feature tests: [P0_Manual_Testing.md](P0_Manual_Testing.md)
