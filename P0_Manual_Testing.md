# P0 — Test the diff pipeline

Once [P0A](P0A_Manual_Testing.md) is green (Postgres, Redis, API, importer all alive), this doc walks through the four things P0 built, in order:

1. **Fetch latest SQLite from OSS** → `scripts/fetchLatestSqlite.ts`
2. **Convert SQLite → `data.json`** → the existing `pnpm run build`
3. **Export Postgres → `pg-data.json`** → `scripts/exportPg.ts` + `src/exporters/exportFromPg.ts`
4. **Diff the two → `diff.json`** → `scripts/runDiff.ts` + `src/diff/diffData.ts`

Then we open the admin UI and click through the Diff tab.

Everything below runs from `yyddp-importer/`. Set once and forget:

```bash
cd "/Users/enterwizard/My Documents/Work/Beijing/yyddp/yyddp-importer"
```

**Legend:** `[x]` pass, `[ ]` not yet.

---

## 1. Generate the Prisma client

The importer needs a Prisma client that matches the API's schema. `yyddp-importer/prisma/schema.prisma` is a copy of `yyddp-api/prisma/schema.prisma` — we generate the client code from it.

**Do this:**

```bash
pnpm run prisma:generate
```

**What it does:** runs `prisma generate`, reads `prisma/schema.prisma`, writes a typed client into `node_modules/@prisma/client`. Needed before `export:pg` can talk to Postgres.

- [x] 1.1 Output ends with `Generated Prisma Client`.

---

## 2. Fetch the latest SQLite from OSS

`scripts/fetchLatestSqlite.ts` lists `oss://ailanguages/videos/sqlite/`, picks the newest `YYYYMMDD.sqlite` (skipping the stray `input.sqlite` decoy), and downloads it to `./input.sqlite`. The P5 scheduled worker will reuse the same `src/oss/fetchSqlite.ts` helper.

**Do this:**

```bash
pnpm run fetch:latest-sqlite
```

**What it does:** lists OSS objects under the prefix, sorts by date, downloads the winner, prints source + size + timestamp.

- [x] 2.1 Output includes a line like `source: oss://ailanguages/videos/sqlite/YYYYMMDD.sqlite`.
- [x] 2.2 That source does NOT end with `/input.sqlite` (the decoy was correctly skipped).
- [x] 2.3 `ls -lh input.sqlite` shows a file with a recent mtime and non-zero size.

**Sanity check the error path** — clear the creds and re-run:

```bash
key= secret= pnpm run fetch:latest-sqlite 2>&1 | tail -3
```

- [x] 2.4 You see `OSS credentials missing` and exit code is non-zero. (Good — no cryptic ali-oss stack trace.)

---

## 3. Convert SQLite → `data.json`

Pre-existing converter. We're just confirming it still works with whatever you just downloaded.

**Do this:**

```bash
pnpm run build
```

**What it does:** runs `src/converter.ts`, reads `input.sqlite`, walks Course → Lesson → Part → Day → Activity → Interaction, and writes a stable `data.json` at `yyddp-importer/data.json`.

- [x] 3.1 Output ends with `转换完成！已生成 data.json`.
- [x] 3.2 `data.json` mtime is within the last minute:

```bash
stat -f '%Sm %z' data.json
```

- [x] 3.3 Expected activity count (should be 343 for the current SQLite):

```bash
node -e "console.log(require('./data.json').length)"
```

---

## 4. Export Postgres → `pg-data.json`

`src/exporters/exportFromPg.ts` walks the PG schema via Prisma and emits an activity array whose shape matches `data.json`, so the diff is a straight compare. `scripts/exportPg.ts` is the CLI wrapper.

Postgres is empty right now (P1 hasn't built the loader yet), so we expect `[]`.

**Do this:**

```bash
pnpm run export:pg
```

**What it does:** opens a Prisma connection, walks `courses → courses_parts → parts → part_days → days → day_activities → activities → activity_interactions → interactions`, builds activity URLs using the hierarchy mapping (see the header comment in `exportFromPg.ts`), writes `pg-data.json`.

- [x] 4.1 Output says `wrote 0 activities`.
- [x] 4.2 Output also hints `PG is empty — this is expected before P1 Import lands`.
- [x] 4.3 `cat pg-data.json` prints literally `[]`.

> If this step errors with `relation "courses" does not exist`, Prisma migrations haven't been applied to `yyddp_dev` yet. That's fine for P0 — the script is supposed to handle an empty/unmigrated DB. If you hit a different error, check `DATABASE_URL` in `.env.dev`.

---

## 5. Run the diff

`src/diff/diffData.ts` does the deep-diff: it keys both sides by activity URL, classifies each URL as `added` / `removed` / `changed` / `unchanged`, and emits one canonical `diff.json`. Pure function, deterministic — same input twice = byte-identical output.

Right now `data.json` has 343 activities and `pg-data.json` is `[]`, so we expect **all 343 as added**.

**Do this:**

```bash
pnpm run diff
```

**What it does:** loads both JSON files, runs `diffActivities()`, writes `diff.json` + prints the summary.

- [x] 5.1 Summary reads `added=343 removed=0 changed=0 unchanged=0 total=343`.
- [x] 5.2 Verify from the file itself:

```bash
node -e "console.log(JSON.stringify(require('./diff.json').summary))"
```

- [x] 5.3 Every entry has status `added`:

```bash
node -e "const d=require('./diff.json');console.log([...new Set(d.activities.map(a=>a.status))])"
```

Output: `[ 'added' ]`.

**Determinism check** — run it twice, output must be identical:

```bash
cp diff.json /tmp/diff1.json && pnpm run diff >/dev/null && diff /tmp/diff1.json diff.json && echo IDENTICAL
```

- [x] 5.4 Prints `IDENTICAL`. If not, something in `diffActivities` is non-deterministic — bug.

## 6. The importer Express routes

The admin UI fetches `data.json`, `pg-data.json`, and `diff.json` via three HTTP endpoints on `yyddp-importer/server/index.ts`. Confirm they work before opening the browser.

**`/api/data` — the SQLite-side activities:**

```bash
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:3001/api/data
```

- [x] 6.1 Returns `200`.

**`/api/pg-data` — the Postgres-side activities:**

```bash
curl -sS http://localhost:3001/api/pg-data
```

- [x] 6.2 Returns `[]` (the exact contents of `pg-data.json`).

**`/api/diff` — the diff report:**

```bash
curl -sS http://localhost:3001/api/diff | node -e "let s='';process.stdin.on('data',c=>s+=c);process.stdin.on('end',()=>console.log(JSON.parse(s).summary))"
```

- [x] 6.3 Prints the same summary as step 5.1.

**404 behavior** — hide `diff.json` and make sure the route fails loud, not silent:

```bash
mv diff.json /tmp/diff.bak
curl -sS -i http://localhost:3001/api/diff | head -6
mv /tmp/diff.bak diff.json
```

- [x] 6.4 Status is `404` and the body includes a `"hint"` telling you which command to run.

---

## 7. Admin UI — the Diff tab

This is the feature. `preview/src/App.vue` has two tabs (Preview + Diff). The Diff tab renders `preview/src/components/DiffView.vue`, which pulls from the three endpoints above and shows a tree on the left, details on the right.

**Open it:**

```
http://localhost:5173/
```

(Dev Vite runs at the root, not `/admin/`. Production will use `/admin/` via `VITE_BASE`.)

**Click through:**

- [x] 7.1 Top bar shows two tabs: **Preview** (default) and **Diff (SQLite vs PG)**.
- [x] 7.2 Click **Diff**. Body switches. URL hash becomes `#diff`.
- [x] 7.3 Summary bar shows four colored badges: `added 343`, `removed 0`, `changed 0`, `unchanged 0`.
- [x] 7.4 Left pane lists Lesson 1–N. Click a lesson → Parts expand. Click a Part → Days expand with their activities. Every activity has a green `added` badge (expected — PG is empty).
- [x] 7.5 Click any activity → right pane shows SQLite JSON on the left column and a "Postgres 中不存在" placeholder on the right.
- [x] 7.6 Refresh the page with `#diff` still in the URL. The Diff tab stays active (doesn't drop you back to Preview).
- [x] 7.7 Click **Preview**. The old DayView flow works — no regression.

---

## Green?

If §1–§7 are `[x]` (and §8 if you ran it), P0 is done. The diff viewer works end-to-end on your local stack.

Mark the corresponding boxes in [P0.md](P0.md):

- §0 last checkbox (dev smoke) → `[x]` if P0A was green
- §0.5 all → `[x]`
- §1, §2, §3 all → `[x]`
- Definition of Done → `[x]`

Next: **P1** (build the Import button that writes the diff into Postgres).

## Known caveats (carry into P1)

1. **Hierarchy mapping.** `exportFromPg.ts` parses the part number out of `parts.part_name` (regex `/Part\s+(\d+)/i`, fallback `1`). If seed data uses a different naming convention, you'll see a warning in the log and the URL will default to Part 1 — fine for P0, something to tighten in P1.
2. **Prisma schema is a copy.** `yyddp-importer/prisma/schema.prisma` is a hand-copy of `yyddp-api/prisma/schema.prisma`. When the API schema changes, re-copy and re-run `pnpm run prisma:generate`.
3. **No migrations applied yet.** Dev Postgres starts empty — migrations are P1's job. §8 above is the only test that needs a migrated schema.
