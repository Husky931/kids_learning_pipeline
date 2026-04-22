# P1 — Test the Import-to-PG pipeline

Once [P0_Manual_Testing](P0_Manual_Testing.md) is green, work through this in the order an operator would.

What we're touching:

1. The loader (`src/loader/load.ts`)
2. Sanity checks (`src/loader/sanityChecks.ts`)
3. Media inventory (`src/loader/mediaInventory.ts`)
4. CLI wrapper (`pnpm run import:pg`)
5. Express route (`POST /api/import`)
6. The Import button in the admin UI

Then three production scenarios: first import, re-run, soft-delete.

Run everything from `yyddp-importer/`:

```bash
cd "/Users/enterwizard/My Documents/Work/Beijing/yyddp/yyddp-importer"
export DATABASE_URL="postgresql://yyddp:devpassword@localhost:5432/yyddp_dev"
```

`[x]` pass, `[ ]` not yet.

---

## 1. Baseline — PG empty, diff says "all added"

Just confirm we're starting clean.

```bash
pnpm run export:pg && pnpm run diff
```

This re-reads PG and re-diffs against `data.json`. Read-only, safe to repeat.

- [x] 1.1 `[export:pg] wrote 0 activities` and "PG is empty" message.
- [x] 1.2 `[diff] summary: added=343 removed=0 changed=0 unchanged=0 total=343`.

If `unchanged` is non-zero, you've got leftover rows from a previous run. Wipe and retry:

```bash
psql "$DATABASE_URL" -c "TRUNCATE courses, parts, days, activities, interactions, courses_parts, part_days, day_activities, activity_interactions RESTART IDENTITY CASCADE;"
```

---

## 2. First import via CLI

Run the import from the shell first so any errors surface plainly.

```bash
pnpm run import:pg
```

You'll see a plan line, streaming progress, then a final JSON blob with row counts and sanity results.

The loader uses deterministic primary keys derived from URLs, which is why re-runs become no-ops.

- [x] 2.1 Progress reaches `343/343 — course 1` with no errors.
- [x] 2.2 Final JSON has `"ok": true`.
- [x] 2.3 Row counts: activities 343, interactions 29426, courses 1, parts 49, days 147. (Numbers drift with content — the point is they're all > 0.)
- [x] 2.4 All nine sanity checks pass. The `ref integrity:` ones prove no orphan junction rows.

Spot-check the DB directly:

```bash
psql "$DATABASE_URL" -c "SELECT
  (SELECT count(*) FROM courses WHERE deleted=false) AS courses,
  (SELECT count(*) FROM parts WHERE deleted=false) AS parts,
  (SELECT count(*) FROM days WHERE deleted=false) AS days,
  (SELECT count(*) FROM activities WHERE deleted=false) AS activities,
  (SELECT count(*) FROM interactions WHERE deleted=false) AS interactions;"
```

- [x] 2.5 Numbers match §2.3 exactly.

---

## 3. Idempotency — the big one

A non-idempotent loader corrupts production the second time you click Import. So we test this hard.

The loader self-checks every import: right after writing, it re-exports from PG and diffs against the `data.json` it just imported. Any non-zero delta = bug. Failure flips `ok: false` and exits non-zero.

**Test 1 — first import's self-check passed.** Look at the JSON from §2:

- [x] 3.1 `sanity` array has `"name": "self-check: post-import diff is zero"` with `"pass": true`. Detail says `unchanged=343`.
- [x] 3.2 `"ok": true`.

**Test 2 — second import is a full no-op.**

```bash
pnpm run import:pg
```

- [s] 3.3 Every counter in `tables.*` is 0 (inserted, updated, soft_deleted, skipped). The diff said "nothing to do" so the loader didn't even touch the rows.
- [x] 3.4 Self-check still green, `ok: true`.

**Test 3 — prove the guard actually catches bugs.** Break the exporter the way a careless PR would:

```bash
cp yyddp-importer/src/exporters/exportFromPg.ts /tmp/exportFromPg.ts.bak
sed -i '' 's/comment_type: activity.title,//' yyddp-importer/src/exporters/exportFromPg.ts
pnpm --prefix yyddp-importer run import:pg; echo "exit=$?"
```

- [x] 3.5 Self-check now `"pass": false` with `changed=343`.
- [x] 3.6 `"ok": false`.
- [x] 3.7 `exit=1`.

Restore:

```bash
cp /tmp/exportFromPg.ts.bak yyddp-importer/src/exporters/exportFromPg.ts
pnpm --prefix yyddp-importer run import:pg | grep -A1 "self-check"
```

- [x] 3.8 Green again, `ok: true`.

> Why `comment_type`? `data.json` has it, PG doesn't — `exportFromPg` re-emits it from `activity.title` to keep the round-trip stable. If real content ever has `comment_type ≠ title`, this check correctly fires.

**Optional double-check via external commands:**

```bash
pnpm run export:pg && pnpm run diff
```

- [x] 3.9 `unchanged=343` — same numbers as the self-check.

---

## 4. Admin UI — Import button

The operator-facing surface. The button shows the change count, disables when there's nothing to do, and streams progress while running.

Open it (server must be running — `pnpm run preview`):

```
http://localhost:5173/#diff
```

PG is already loaded from §2, so:

- [x] 4.1 Diff tab renders. Summary shows all unchanged.
- [x] 4.2 Button reads **`Import to PG (0 changes)`**, disabled, tooltip explains why.
- [x] 4.3 Click Refresh — still disabled.

To actually click it, give it something to import:

```bash
psql "$DATABASE_URL" -c "TRUNCATE courses, parts, days, activities, interactions, courses_parts, part_days, day_activities, activity_interactions RESTART IDENTITY CASCADE;"
pnpm run export:pg && pnpm run diff
```

- [x] 4.4 Reload UI. Summary shows `added 343`. Button reads **`Import to PG (343 changes)`**, enabled.
- [x] 4.5 Click. Progress panel appears, status cycles through courses, result panel shows counts + sanity (all green) + media inventory.
- [x] 4.6 Page auto-refreshes. Button back to disabled `(0 changes)`. That's the operator loop closed.

---

## 6. Soft-delete

The contract says deletions are always soft. Loader sets `deleted = true` on activities + interactions whose URLs vanish from `data.json`.

Swap in a temporary `data.json` missing one activity:

```bash
cp data.json /tmp/data.json.bak
node -e "const d=require('./data.json'); const gone=d.shift(); console.log('dropped', gone.url); require('fs').writeFileSync('./data.json', JSON.stringify(d, null, 2));"
pnpm run diff
```

- [x] 6.1 `dropped /courses/1/lessons/1/...` printed.
- [x] 6.2 `[diff] summary: removed=1 unchanged=342`.

Run the import:

```bash
pnpm run import:pg
```

- [x] 6.3 Progress shows `phase=soft-delete (1 activities)`.
- [x] 6.4 Result has `tables.activities.soft_deleted >= 1` and same for interactions.

Confirm in SQL — this is the audit evidence:

```bash
psql "$DATABASE_URL" -c "SELECT activity_id, title, deleted, updated_at FROM activities WHERE activity_id = 1;"
psql "$DATABASE_URL" -c "SELECT count(*) FROM activities WHERE activity_id = 1;"
```

- [x] 6.5 Row shows `deleted | t`, recent `updated_at`.
- [x] 6.6 `count(*) == 1` — the row physically still exists. That's the whole point.

Restore:

```bash
cp /tmp/data.json.bak data.json
pnpm run import:pg
pnpm run export:pg && pnpm run diff
```

- [ ] 6.7 `unchanged=343`. Back to the §3 idempotent state.

The restore worked because `upsertActivity` flips `deleted=true` back to `false` when the URL reappears. Look for `tables.activities.updated == 1` in the result — that's the resurrect.

- [ ] 6.8 Re-run once more, all zeroes.

---

## Green?

If §1–§7 are `[x]`, P1 is done. Loader is correct, idempotent, soft-deletes properly, has rails, drives end-to-end through CLI or UI.

Mark in [P1.md](P1.md):

- §1 Loader core — proved in §2, §3, §6
- §2 Media upload — partial; inventory ships, uploads are P5
- §3 Admin UI Import button — proved in §4
- §4 Safety rails — proved in §7
- DoD — all `[x]`

Next: **P2** (redo tests TDD-style in `yyddp-api`).

## Known caveats

1. **Media uploads don't actually upload yet.** Inventory only reports. Real uploads are P5. Current OSS paths happen to exist for current content.
2. **Synthetic course/part/day names.** Loader writes `"Course 1"`, `"Part 3"`, etc. Good enough for the backend. When authoring ships real titles, revisit.
3. **`IMPORT_TARGET=prod` still points at the dev DB.** We have the gate, not the switch. Wiring a separate `DATABASE_URL` is P5.
4. **Transaction timeout at 5 minutes.** Local first import is ~20s. If a course balloons past ~500 activities, may need per-Day transactions.
5. **Prisma schema is a hand-copy** from `yyddp-api`. Re-copy + `prisma:generate` when the API schema changes.
