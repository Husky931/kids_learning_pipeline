# YYDDP 4-Day Launch Roadmap

Two repos: `yyddp-api` (backend) and `yyddp-importer` (SQLite → PG pipeline). Contract at [Contract.md](Contract.md). Full design doc at `~/.claude/plans/read-contract-md-that-is-bubbly-whisper.md`.

## Working branches (locked 2026-04-19)

- **yyddp-api**: `origin/dev` is the canonical trunk (CI auto-deploys from it to `deploy.yyddp.com/deploy`). Do NOT touch `main`. `origin/dev` is ~35-40% of P4+P5 done — audit in the master plan file.
- **yyddp-importer**: `main` / `origin/dev` / `origin/data` / `dev_gligor` are all in sync. Work from `dev_gligor`.

## Upstream system (not ours to build)

[Inspinia](http://116.62.137.90/api/r/yyddp/videoreviewer/dashboard) is the client's content-authoring/review UI. Its only output to us is the daily SQLite dump in `oss://ailanguages/videos/sqlite/YYYYMMDD.sqlite`. We read it; we don't touch Inspinia.

## Phases

| # | File | Title | Owner repo |
|---|---|---|---|
| P0 | [P0.md](P0.md) | Visualize the diff (SQLite JSON vs PG JSON) | yyddp-importer (+ infra) |
| P1 | [P1.md](P1.md) | Import it into PG | yyddp-importer |
| P2 | [P2.md](P2.md) | Submit APIs + Gold Service | yyddp-api |
| P3 | [P3.md](P3.md) | Redo tests (TDD) | yyddp-api |
| P4 | [P4.md](P4.md) | ORM + bind API (GET) + OSS | yyddp-api |
| P5 | [P5.md](P5.md) | Locks + Real SMS auth | yyddp-api |
| P6 | [P6.md](P6.md) | Make importer incremental + deploy | yyddp-importer + ops |

## Day budget (suggestion)

- **Day 1:** P0 §0 (local dev baseline) AM, P0 §1–3 (diff viz) PM ✅
- **Day 2:** P1 (loader + Import button) AM, P2 (submit APIs + gold) PM ✅ / in progress
- **Day 3:** P3 (TDD reset) AM, P4 (GET endpoints) PM — Week-1 milestone locks EOD
- **Day 4:** P5 (SMS, unlock). P6 spills to Day 5 if needed

## Week-1 milestone (Contract §4) = end of P4

SMS login works + course list/parts/days/activities served from real PG.

## Final delivery (Contract §5, §6) = end of P6

Locks + gold + POST writes + incremental importer + deployed to client env.

## Blocking asks to client

See [asks-to-client.md](asks-to-client.md). None of these block P0 §0 (local dev baseline).

## Global infra

- **Dev:** local Postgres 16 + Redis 7 on the operator's Mac. `yyddp-api` runs via `pnpm dev` (nodemon on 8888); `yyddp-importer` runs via `pnpm run preview` (Express 3001 + Vite 5173).
- **Prod:** same services launched via `pm2` (or systemd) on the client's server; nginx as the public reverse proxy.
- `.env.dev.example` at repo root documents the required env vars.
