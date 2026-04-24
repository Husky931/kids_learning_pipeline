# P4 Manual Testing Guide

Covers every piece of P4 — ORM flip, real Postgres data, OSS downloads, `/real/*` removal, health, request logging, unlock bypass — plus smoke tests for adjacent endpoints (`auth/refresh-token`, `auth/logout`, `/sessions/*`, `/courses/:id/progress`) and defensive checks for failure modes (DB down, expired JWT, path traversal, etc.). Each section explains **what the user experiences** and **how to verify it from the API side**.

---

## How to use this guide

Walk top-to-bottom. Every section has an **Expected** block and a **What to check** checklist. Fill in the results table below when you run a pass.

---

## Manual Testing Results

| # | Section                           | Result | Notes |
|---|-----------------------------------|--------|-------|
| 0 | Mockup cleanup verification       | X      | All 4 `git grep` commands empty; all 4 `ls` commands report "No such file or directory". |
| 1 | Health check                      | X      | `status:"ok"`, `db:"ok"`, `environment:"development"`, `uptime` is a positive float. `POST /api/v1/health` → 405 with `allowedMethods:["GET"]`; `GET /health` → 302 → `/api/v1/health`. |
| 2 | Course list                       | X      | Works. Seed drift: DB has **1 course** (id=1, name "Course 1", 49 parts), not the 2 "英语启蒙/进阶课程" in the guide. Edges: `?start=-1` → 400 with `100004` (PARAM_OUT_OF_RANGE — more specific than guide's `100005 PARAM_INVALID`); `?start=5&end=5` → 400; `?start=1000&end=1001` → 200 with `courses:[]`; `?end=-1` → 400; no JWT → 401/600001. |
| 3 | Course detail                     | X      | Happy path returns all documented keys on the course and every card. 49 cards (not 2), all with `locked:false` and days all `status:"unlocked"` (bypass ON). 9999 → 404/101003; `abc` → 400/100002 (PARAM_FORMAT_ERROR — more specific than guide's 100005); 0 → 400/100004; no JWT → 401/600001. |
| 4 | Today's content                   | X      | All documented keys present (`course_id, course_name, day_id, day_title, estimated_duration, activities, playlist`). Real `day_id` is `100100101` (composite), not `1` as guide states. 2 activities, all `unlocked`. `playlist: []` as expected. 9999 → 404/101003; no JWT → 401. |
| 5 | Playlist                          | X      | `audio_resources: []` on seed (no `days.playlist` populated — matches guide's edge case). 9999 → 404/101003. |
| 6 | Part detail                       | X      | Uses real `part_id=1001001` (the guide's `/parts/1` 404s because that ID doesn't exist in the DB). Shape is correct (`part_id, title, name, description, earned_gold, days`); 3 days returned, each with `status:"unlocked"` and activities array with all documented keys. `earned_gold:0` (seed has 0-coin activities). Bogus course/part → 404; `abc` course → 400. |
| 7 | Activity detail                   | X      | Uses real IDs (`/parts/1001001/days/100100101/activities/1`). Returns `activity_id, display_type, interactions`. `display_type: "crosstalk"` (not `"video"` as guide) — seed drift. Bogus activity → 404; `invalid` activity or day → 400. |
| 8 | User profile / stats              | X      | `/profile` and `/stats` are byte-identical. `id:1`, `phone:"18601113318"`, all 5 stats keys present and numeric. Minor guide drift: keys are actually `total_activities_completed` and `total_learning_time_minutes` (guide calls them `activities_completed`, `learning_time_minutes`). No JWT → 401. |
| 9 | Course switching                  | X      | Missing body → 400/100001; `course_id:9999` → 404/101003; `course_id:"abc"` → 400/100002; idempotent switch to 1 returns success. Can't test switching to course 2 — only course 1 exists in DB. |
| 10 | File downloads (OSS)             | X      | Re-tested with real files from `data.json`: `cards/b/baby/baby-child-833-v1.png` (image), `audios/words/g/green/green-v3.mp3` (audio), and `finalversion/flyingcat/part1/flyingcat-p1-crosstalk.mp4` (video) all return **HTTP 206 Partial Content** with real bytes when the signed URL is followed. The original guide sample `shark-p1.mp3` had been removed from the bucket at some point — stale example, not a code bug. Guide (above in §10) now reflects OSS **V4** signing (`x-oss-signature-version=OSS4-HMAC-SHA256`, `x-oss-expires=3600`) and uses the known-working sample paths. |
| 11 | `/real/*` removed                | X      | `/real/courses`, `/real/user/profile`, `/real/courses/1` all 404/101003. `git grep '/real/' yyddp-api/src/api` → empty. No `/real/` strings in the swagger spec (pulled from `/api/v1/docs/swagger-ui-init.js`). |
| 12 | Unlock bypass (ON)               | X      | Verified in §3, §4, §6: every `locked` field is `false`, every `status` field is `"unlocked"`. |
| 13 | Swagger docs                     | X      | `/api/v1/docs` → Swagger UI for humans; **added** `/api/v1/docs-json` → raw OpenAPI 3.0.0 spec as JSON (21 paths documented, no `/real/*`). `src/app.js` now mounts both in non-production environments. The JSON endpoint is what mobile/SDK consumers fetch to generate typed client libraries, import Postman collections, or lint the spec in CI. Guide's `curl /api/v1/docs-json \| jq '.paths \| keys[]'` command now works. |
| 14 | Request logging (basic)          | X      | Logging middleware is wired (`app.use(pinoHttp)` + `requestIdMiddleware` in `src/app.js`). **Fixed**: `logger.js:customLogLevel` now returns `'info'` for every 2xx response (previously GET/PUT/DELETE 2xx returned `'debug'`, which was filtered out by `LOG_LEVEL=info` — so successful reads printed nothing). Re-verified: fresh server logs went from 4 INFO lines (startup only) to 10 INFO after firing a handful of GETs, each carrying the matching `requestId`. You now get full access-log visibility: `INFO: GET /api/v1/courses completed with 200` lines, `WARN: ... 404`, `ERROR: ...` for 5xx. |
| 15 | Full user journey (golden path)  | X      | Completed end-to-end: login → `/courses` → `/courses/1` → `/courses/1/today` → `/courses/1/lessons/1/parts/1001001` → `/activities/1` → `/download/...` (signed URL generated, OSS signed correctly) → `/sessions/start` (session_id 6) → `/sessions/6/interactions` → `/sessions/6/complete` (reward_coins 10, total_coins 23→33) → `/user/profile` reflects 33 → `/courses/switch course_id:1` → `/courses/1/playlist`. Every call 200, coins incremented by exactly 10. |
| 16 | Auth lifecycle (refresh / logout)| X      | Refresh returns a **new** access_token, `expires_in:3600`; the old access_token still works until natural expiry; access-token-used-as-refresh → 401/600004; logout returns 200/"已成功登出"; logout without JWT → 401. **Fixed**: invalid/malformed refresh_token body now returns **401/600004** "Invalid or expired refresh token" instead of 500. See `authService.js refreshToken()` — non-`ApiError` exceptions from `jwt.verify` are now wrapped as `USER_TOKEN_INVALID` so clients can send the user back to login cleanly. |
| 17 | Sessions (start / interact / complete) | X | `/start` returned session_id=6, `activity_id` and `day_id` echoed; `/interactions` → `recorded:true, coins_awarded:0`; first `/complete` → `reward_coins:10, total_coins:33` (profile went 23→33); second `/complete` on same session → `already_completed:true`; bogus activity → 404/101003; missing `activity_id` → 400/100001; bogus session in `/complete` → 404/101003; no JWT on `/start` → 401. |
| 18 | Course progress                  | X      | Top-level shape correct (`course_id, course_name, total_activities, completed_activities, total_coins_earned, parts`): 49 parts, 343 total activities, 1 completed, 10 coins earned. **Fixed**: each part (and each day) now carries its own `total_activities`, `completed_activities`, `coins_earned` counters, so the mobile app can render per-part progress bars without a second call. Re-verified: `sum(parts.total_activities) == 343`, `sum(parts.completed_activities) == 1`, `sum(parts.coins_earned) == 10` — rollups add up to the course-level totals. Swagger schema for `/courses/{id}/progress` updated to describe the new fields. 9999 → 404/101003. |
| 19 | JWT lifecycle edge cases         | X      | Expired (signed with `-1s`) → 401/600004; malformed → 401/600004; no `Bearer` prefix → 401/600001; signature-flipped → 401/600004. **Fixed**: using the refresh token as an access token now returns **401/600004** "Access token required; this token is not an access token." `src/middleware/authMiddleware.js` now explicitly rejects any token whose payload `type` isn't `'access'`. Re-verified a real access token still works on the same route (200). |
| 20 | DB connection loss               | O      | Skipped — Postgres on this box is the shared Homebrew `postgresql@16` process (PID 1677). Stopping it would disrupt unrelated work. Code path is trusted by inspection: `/health` has a `db: "error"` branch, and per §20 the A4 fix throws `DATABASE_CONNECTION_FAILED` in `phone-login`. Needs a dedicated disposable PG or containerized DB to exercise. |
| 21 | Request logging verification     | O      | Deferred per user — app-wide logging pass will cover it. What I did verify: ran a parallel server with `node src/server.js > /tmp/yyddp-8889.log 2>&1 &` and fired 4 deliberate requests. Results — **4xx** logs at **WARN** with `GET /api/v1/courses/9999 completed with 404` + matching `requestId`; **errors** log at **ERROR** with the ApiError stack; `request_id` in each log line matches the `request_id` the client got back. **But** 2xx **GET** responses (e.g. `/courses`, `/health`) don't appear in the log at all — [logger.js:38-43](yyddp-api/src/utils/logger.js#L38-L43) deliberately routes them to `debug`, which `LOG_LEVEL=info` filters. Only 2xx **POST** responses log (at INFO). Config choice, not a bug — flip the last `return 'debug'` to `return 'info'` for access-log style visibility. |
| 22 | Unlock bypass (OFF)              | O      | Spun up a parallel server on port 8889 with `UNLOCK_BYPASS=false PORT=8889 node src/server.js`. Re-ran §3, §4, §6: **nothing changed** — all 49 cards, 147 days, and all activities still returned `locked:false`/`status:"unlocked"`. That matches the guide's own footnote: "If the code currently hardcodes `'unlocked'` in both branches… document that as a known limitation — real unlock logic lands in P5, per Contract §2.4." So the flag is a no-op until P5 ships the unlock engine. |
| 23 | Regression checklist             | X      | **23a** Real auth flow → access token bound to `user_id:1`; `/user/profile` returns user 1's record (not 404); `/courses/switch course_id:1` succeeds; the full §17 session loop ran cleanly against this token. **23b** `GET /auth/otp?phone=18601113318` → 200/`{phone, expires_in:300}`; login with `sms_code:"123456"` succeeds (MOCK_PHONEOTP honored). **23c** `/api/v1/docs/swagger-ui-init.js` contains no `/real/` strings; `git grep '/real/' yyddp-api/src/api` empty. *(Caveat: the guide's `docs-json` endpoint itself doesn't exist — see §13.)* |

**Result codes:** `[x] PASS` · `[O] PARTIAL` · `[!] FAIL` · `[-] BLOCKED`.

<details>
<summary>Previous run — pre-cleanup (2026-04-23, 10/15 pass)</summary>

The earlier 15-section version of this guide ran against a codebase that still had `.env.mockup`, a hardcoded `MOCK_JWTTOKEN`, and the unfixed OTP endpoint. It surfaced three blocking defects:

1. **Mock JWT userId mismatch.** The "Option A — Mockup bypass" token hardcoded `userId: 12345`, which didn't exist in `yyddp_dev` (only `user_id: 1`). Every `/user/profile`, `/user/stats`, `/courses/switch` call 404'd.
2. **OTP crash.** `GET /auth/otp` always called real Aliyun SMS; `.env.test` had `ALIYUN_SMS_DISABLED=true` but no code actually read the flag.
3. **Phantom `/real/user/stats` in Swagger.** One JSDoc annotation still documented the deleted endpoint.

All three are now addressed in code (see §23). The mockup bypass was deleted entirely; the only path to get a JWT is the real auth flow (now usable without real SMS thanks to `ALIYUN_SMS_DISABLED=true`).

</details>

---

## Prerequisites

### Start the server

```bash
cd yyddp-api
pnpm dev
```

This runs with `NODE_ENV=development`, which loads `.env.local` first (your local overrides) and falls back to `.env.development`. Both set:

- `ALIYUN_SMS_DISABLED=true` — `getOTP` skips the Aliyun call; the OTP goes to stdout and is stored in memory.
- `MOCK_CAPTCHA=123456` — the captcha SVG text is forced to `123456` so you don't have to OCR it.
- `MOCK_PHONEOTP=123456` — when SMS is disabled, this is the OTP stored for every `getOTP` call.
- OSS creds pointing at the `ailanguages` bucket.

### Get a JWT (real auth flow — the only supported path)

The legacy mockup-bypass token is gone; going through the real flow is now fast because SMS + captcha are deterministic in dev.

1. **Fetch a captcha.**
   ```
   GET /api/v1/auth/captcha
   ```
   Response includes `captcha_id`. The `captcha_value` field is also returned in non-production for convenience; regardless, `MOCK_CAPTCHA=123456` forces the text to be `123456`.

2. **Verify the captcha.**
   ```
   POST /api/v1/auth/verify-captcha
   Body: { "captcha_id": "<from step 1>", "captcha_value": "123456" }
   ```

3. **Request an OTP.**
   ```
   GET /api/v1/auth/otp?phone=18601113318
   ```
   Because `ALIYUN_SMS_DISABLED=true`, the server logs a `warn`-level line like `ALIYUN_SMS_DISABLED=true — OTP stored but SMS not sent. Use this code to log in.` with `otp: "123456"` in the context. No real SMS is sent.

4. **Log in.**
   ```
   POST /api/v1/auth/phone-login
   Body: { "phone": "18601113318", "sms_code": "123456", "device_info": "ManualTest" }
   ```
   Response gives you `access_token` (use as `Bearer <token>`) and `refresh_token`. The access token is bound to the **real seeded `user_id: 1`** — which is what fixes the pre-cleanup userId=12345 mismatch.

Every authenticated request below requires:

```
Authorization: Bearer <your-access_token>
```

---

## Test data (seed)

| Entity | IDs | Details |
|--------|-----|---------|
| Courses | 1 = "英语启蒙" (Beginner, default), 2 = "进阶课程" (Intermediate) | |
| Parts (cards) | 1 = Part A, 2 = Part B, 3 = Part C | |
| Course 1 parts | Part A (seq 1), Part B (seq 2) | |
| Course 2 parts | Part C (seq 1) | |
| Days | 1, 2, 3, 4 | |
| Part A days | Day 1 (seq 1), Day 2 (seq 2) | |
| Part B days | Day 3 (seq 1) | |
| Part C days | Day 4 (seq 1) | |
| Activities | 1 = video "Watch: Hello!", 2 = matching "Match: Animals", 3 = listeningtest "Listen: Colors", 4 = audiopickcard "Pick Card: Fruits", 5 = video "Watch: Greetings" | Each worth 10 coins |
| Day 1 activities | 1 (video), 2 (matching) | |
| Day 2 activities | 3 (listeningtest) | |
| Day 3 activities | 4 (audiopickcard) | |
| Day 4 activities | 5 (video) | |
| Users | 1 = phone `18601113318`, enrolled in course 1, current_day=1; 2 = enrolled in course 2; 3 = no enrollment | User 1 is what the real auth flow resolves to |

Dev (`yyddp_dev`) may deviate from this seed. The test DB (`yyddp_test`) matches it exactly.

---

## 0. Mockup cleanup verification

The P4 follow-up scrubbed mockup mode from the tree. Before running any other test, confirm the cleanup held.

Run these from `yyddp_copy/`:

```bash
# No mockup references in yyddp-api source, docs, scripts, or env files.
git grep -i -n mockup yyddp-api/src yyddp-api/docs yyddp-api/scripts yyddp-api/prisma yyddp-api/.gitignore

# No stale MOCK_ tokens from the old mockup bypass. MOCK_CAPTCHA + MOCK_PHONEOTP are expected and live.
git grep -n 'MOCK_JWTTOKEN\|MOCK_PHONE[^O]' yyddp-api/

# No dead /real-only controller exports.
git grep -n 'getProfileReal\|getStatsReal\|getPlaylistByCourseIdReal\|getCourseByIdReal\|getPartByIdReal\|getActivityByIdReal\|getAllCourses' yyddp-api/src

# No mockup env file, no JSON stub directory, no orphaned tools/.
ls yyddp-api/.env.mockup 2>&1
ls yyddp-api/src/db/mockups 2>&1
ls yyddp-api/tools 2>&1
ls yyddp-api/debug.js 2>&1
ls yyddp-api/get_to_know_testing.md 2>&1
```

**What to check:**
- All four `git grep` commands print nothing (exit 1). The only acceptable `mockup` hits are under `yyddp-api/memory-bank/` (historical AI notes, not user-facing); you can exclude that with `:(exclude)yyddp-api/memory-bank`.
- All four `ls` commands print `No such file or directory`.
- Running server shows `environment: "development"` in `GET /api/v1/health` (see §1), never `mockup`.

If any command returns hits, flag it in the results table — cleanup regressed somewhere.

---

## 1. Health check — is the server alive and talking to Postgres?

**What the user experiences:** Nothing directly — this is an ops endpoint. If it fails, nothing else works.

```
GET /api/v1/health
```

No auth required.

**Expected:**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "status": "ok",
    "version": "1.0.0-dev",
    "uptime": 42.5,
    "db": "ok",
    "environment": "development"
  }
}
```

**What to check:**
- `status: "ok"` — Express is running.
- `db: "ok"` — Prisma reached Postgres. If `"error"`, every authed endpoint will 500.
- `environment: "development"` — confirms the mockup env is gone.
- `uptime` is a positive float (seconds since start).
- `POST /api/v1/health` → **405 Method Not Allowed**.
- `GET /health` (no prefix) → **302** redirect to `/api/v1/health`.

---

## 2. Course list — "what courses do I have?"

**What the user experiences:** App home screen lists the user's courses with the current one highlighted and a card count.

```
GET /api/v1/courses
```

**Expected (real-user flow → user 1):**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "current_course_id": 1,
    "courses": [
      { "course_id": 1, "name": "英语启蒙", "description": "...", "total_cards": 2, "url": "/courses/1" },
      { "course_id": 2, "name": "进阶课程", "description": "...", "total_cards": 1, "url": "/courses/2" }
    ]
  }
}
```

**What to check (happy path):**
- `current_course_id` is `1` for fresh user 1.
- `total_cards` matches the number of parts linked to each course (2 for course 1, 1 for course 2).
- Every documented field is present on each course row.

**Edge cases:**
- `?start=0&end=1` → only the first course (pagination slice).
- `?start=-1` → **400** (`ret_code: 100005`, PARAM_INVALID).
- `?start=5&end=5` → **400** (end must be greater than start).
- `?start=1000&end=1001` → **200** with `courses: []` (valid but empty slice).
- `?end=-1` → **400**.
- No JWT → **401** (`ret_code: 600001`).
- JWT for user 3 (no enrollment) → `current_course_id: null`, `courses` still lists all available courses (catalog is not filtered by enrollment).

---

## 3. Course detail — "what's inside this course?"

**What the user experiences:** Tapping a course opens a card grid. Each card has a cover, title, and days underneath.

```
GET /api/v1/courses/1
```

**Expected:**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "id": 1,
    "course_id": 1,
    "name": "英语启蒙",
    "total_cards": 2,
    "url": "/courses/1",
    "description": "...",
    "cards": [
      {
        "card_id": 1, "title": "Part A", "url": "/courses/1/cards/1",
        "cover": null, "cover_hd": null,
        "currentTask": true, "todayTask": true, "completed": false, "locked": false,
        "days": [
          { "day_id": 1, "title": "Day 1", "status": "unlocked" },
          { "day_id": 2, "title": "Day 2", "status": "unlocked" }
        ]
      },
      {
        "card_id": 2, "title": "Part B", "url": "/courses/1/cards/2",
        "cover": null, "cover_hd": null,
        "currentTask": true, "todayTask": true, "completed": false, "locked": false,
        "days": [
          { "day_id": 3, "title": "Day 3", "status": "unlocked" }
        ]
      }
    ]
  }
}
```

**What to check (happy path):**
- 2 cards for course 1; Part A nests Day 1 + Day 2, Part B nests Day 3.
- Every card has `card_id`, `title`, `url`, `cover`, `cover_hd`, `currentTask`, `todayTask`, `completed`, `locked`, `days`.
- All `locked: false` and `status: "unlocked"` (UNLOCK_BYPASS on; §22 covers the OFF state).

**Edge cases:**
- `GET /api/v1/courses/9999` → **404** (RESOURCE_NOT_EXISTS).
- `GET /api/v1/courses/abc` → **400** (PARAM_INVALID — not numeric).
- `GET /api/v1/courses/0` or `/courses/-1` → **400** (min=1).
- Course with zero parts (insert a test course 3 with no rows in `courses_parts`) → `cards: []`, `total_cards: 0`, no 500.
- No JWT → **401**.

---

## 4. Today's content — "what do I study right now?"

**What the user experiences:** The main "Start Learning" screen. Current day's activities + the audio playlist for background listening.

```
GET /api/v1/courses/1/today
```

**Expected (user 1, current_day=1):**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "course_id": 1,
    "course_name": "英语启蒙",
    "day_id": 1,
    "day_title": "Day 1",
    "estimated_duration": 15,
    "activities": [
      { "activity_id": 1, "display_type": "video", "title": "Watch: Hello!", "reward_coins": 10, "status": "unlocked" },
      { "activity_id": 2, "display_type": "matching", "title": "Match: Animals", "reward_coins": 10, "status": "unlocked" }
    ],
    "playlist": []
  }
}
```

**What to check (happy path):**
- `day_id` matches the user's `current_day` from `user_courses`.
- `activities` is sequenced by the `day_activities.sequence` column.
- `playlist` is an array. Empty on seed data because no `days.playlist` JSON is populated.

**Edge cases:**
- `GET /api/v1/courses/9999/today` → **404**.
- User with no enrollment asking for today: the service falls back to the first day of the course via `courses_parts.sequence` + `part_days.sequence`. Verify this against user 3 (no enrollment).
- Course with parts but no days → **404** ("No days found for this course").
- `estimated_duration` is nullable — if the `days.estimated_duration` column is null, the response should show `null`, not crash.

---

## 5. Playlist — "background audio for this course"

**What the user experiences:** A music-player screen. Audio lessons that play while the phone is locked.

```
GET /api/v1/courses/1/playlist
```

**Expected with populated playlist JSON:**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "course_id": 1,
    "audio_resources": [
      { "audio_id": 1, "title": "Good Morning Song", "cover": "/download/images/cover1.jpg", "cover_hd": "/download/images/cover1_hd.jpg", "audio": "/download/audios/song1.mp3", "duration": 120, "status": "not played" }
    ]
  }
}
```

**What to check (happy path):**
- `audio` paths are relative `/download/...` URLs — pass them to the download endpoint (§10) to get a signed OSS URL.
- Every item has `audio_id`, `title`, `audio`, `duration`.

**Edge cases:**
- `GET /api/v1/courses/9999/playlist` → **404**.
- Course where every day has `playlist IS NULL` → **200** with `audio_resources: []` (not a 500).
- Course where `playlist` is `{}` (empty JSON) → `audio_resources: []`.
- Course where `playlist` is a malformed JSON string → should not 500; handler catches and returns empty (spot-check the server log for a parse warning).

---

## 6. Part (card) detail — "what's inside this card?"

**What the user experiences:** Tapping a card shows the days within + the activities under each day — the "lesson plan" view.

```
GET /api/v1/courses/1/lessons/1/parts/1
```

Note: `lessons/1` is a URL segment; the entity hierarchy is course → part → day → activity. `lesson_id` is informational.

**Expected:**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "part_id": 1,
    "title": "Part A",
    "name": "Part A",
    "description": null,
    "earned_gold": 30,
    "days": [
      {
        "day_id": 1, "title": "Day 1", "status": "unlocked",
        "activities": [
          { "activity_id": 1, "display_type": "video", "url": "...", "title": "Watch: Hello!", "status": "unlocked", "currentTask": true, "completed": false, "locked": false },
          { "activity_id": 2, "display_type": "matching", "url": "...", "title": "Match: Animals", "status": "unlocked", "currentTask": true, "completed": false, "locked": false }
        ]
      },
      {
        "day_id": 2, "title": "Day 2", "status": "unlocked",
        "activities": [
          { "activity_id": 3, "display_type": "listeningtest", "url": "...", "title": "Listen: Colors", "status": "unlocked", "currentTask": true, "completed": false, "locked": false }
        ]
      }
    ]
  }
}
```

**What to check (happy path):**
- `earned_gold` is **30** — the sum of `reward_coins` across ALL activities in Part A (activities 1, 2, 3 × 10 coins each). `PartsService.js:55` computes this as a reduce across every day's activities in the part.
- `days` order matches `part_days.sequence`.
- Every activity has all fields (`activity_id`, `display_type`, `url`, `title`, `status`, `currentTask`, `completed`, `locked`).

**Edge cases:**
- `GET /api/v1/courses/9999/lessons/1/parts/1` → **404** (course not found).
- `GET /api/v1/courses/1/lessons/1/parts/9999` → **404** (part not found in this course).
- `GET /api/v1/courses/abc/lessons/1/parts/1` → **400**.
- Part with zero days → `days: []`, `earned_gold: 0`, no 500.
- A day in the part with zero activities → the day appears with `activities: []`.

---

## 7. Activity detail — "what do I actually do here?"

**What the user experiences:** The learning screen. Video player, matching grid, listening quiz, etc.

```
GET /api/v1/courses/1/lessons/1/parts/1/days/1/activities/1
```

**Expected (activity 1, video):**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "activity_id": 1,
    "display_type": "video",
    "interactions": [
      { "interaction_id": 1, "type": "video" }
    ]
  }
}
```

**Expected (activity 2, matching):**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "activity_id": 2,
    "display_type": "matching",
    "interactions": [
      { "interaction_id": 2, "type": "matching" },
      { "interaction_id": 6, "type": "wordcombo" }
    ]
  }
}
```

**What to check (happy path):**
- `display_type` drives the UI template: `video`, `matching`, `listeningtest`, `audiopickcard`.
- `interactions` is ordered by `activity_interactions.sequence`.
- Every interaction has both `interaction_id` and `type`.

**Edge cases:**
- `GET .../activities/999999` → **404**.
- `GET .../activities/invalid` → **400**.
- `GET .../days/invalid/activities/1` → **400**.
- Activity with zero interactions → **200** with `interactions: []` (not 500).

---

## 8. User profile and stats

**What the user experiences:** The "Me" tab. Avatar, phone, coin total, streak, active courses.

```
GET /api/v1/user/profile
GET /api/v1/user/stats
```

Both return the same shape; `stats` is an alias.

**Expected (user 1, fresh):**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": {
    "id": 1,
    "username": null,
    "phone": "18601113318",
    "joined_at": "2026-04-...",
    "total_coins": 0,
    "current_course": 1,
    "active_courses": [...],
    "statistics": {
      "learning_days": 0,
      "streak_days": 0,
      "longest_streak": 0,
      "activities_completed": 0,
      "learning_time_minutes": 0
    }
  }
}
```

**What to check (happy path):**
- `id` matches the JWT's subject (user 1).
- `phone` matches the phone the user logged in with.
- `statistics` object has all five keys, all numeric.
- `total_coins` matches the `users.total_coins` column.
- `/profile` and `/stats` return identical payloads.

**Edge cases:**
- `/profile` without JWT → **401**.
- JWT for a user that was soft-deleted (`users.deleted=true`) → **404** ("User not found").
- After running §17 (sessions), `total_coins` goes up by the completed activity's `reward_coins`.

---

## 9. Course switching — "study a different course"

**What the user experiences:** Tap another course, confirm, app loads the new course.

```
POST /api/v1/courses/switch
Body: { "course_id": 2 }
```

**Expected:**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": { "course_id": 2, "message": "切换成功" }
}
```

**What to check (happy path):**
- After switching, `GET /api/v1/courses` reflects `current_course_id: 2`.
- `users.current_course_id` row updated in PG (spot-check with `prisma studio` if available).

**Edge cases:**
- Missing body or missing `course_id` → **400** (`ret_code: 100001` MISSING_REQUIRED_PARAM).
- `course_id: 9999` → **404** (RESOURCE_NOT_EXISTS).
- `course_id: "abc"` → **400** (PARAM_INVALID).
- Switching to the current course twice → idempotent (no error, state unchanged).
- **Remember to switch back to course 1** before running downstream tests.

---

## 10. File downloads (OSS) — "play this video / audio / cover"

**What the user experiences:** The app requests a signed URL, then either follows the 302 redirect or streams the URL.

Use a file that actually exists in the bucket. `OSS_DIRECTORY=/videos` prefixes everything, so the object key inside OSS ends up as `videos/<your-path>`. Good known-working samples pulled from `yyddp-importer/data.json`:

- Image: `cards/b/baby/baby-child-833-v1.png`
- Audio: `audios/words/g/green/green-v3.mp3`
- Video: `finalversion/flyingcat/part1/flyingcat-p1-crosstalk.mp4`

```
GET /api/v1/download/cards/b/baby/baby-child-833-v1.png?redirect=false
```

**Expected (redirect=false):**
```json
{
  "ret_code": 0,
  "request_id": "...",
  "data": { "url": "https://ailanguages.oss-cn-beijing.aliyuncs.com/videos/cards/b/baby/baby-child-833-v1.png?x-oss-credential=...&x-oss-date=...&x-oss-expires=3600&x-oss-signature-version=OSS4-HMAC-SHA256&x-oss-signature=..." }
}
```

**What to check (happy path):**
- Returned URL is a V4 presigned URL — it carries `x-oss-credential`, `x-oss-date`, `x-oss-expires`, `x-oss-signature-version=OSS4-HMAC-SHA256`, and `x-oss-signature`. (The older V1 `Expires`+`Signature` pair is gone — `ali-oss`'s `signatureUrlV4` has superseded it.)
- Without `?redirect=false`, the endpoint returns **302** and redirects to the signed URL directly.
- The signed URL actually resolves. `curl -r 0-10 -o /dev/null -w "%{http_code}\n" "<url>"` should return **206** (Partial Content) for the samples above.

**Edge cases:**
- No JWT → **401**.
- Empty path (`GET /api/v1/download/`) → **400**.
- Path traversal (`GET /api/v1/download/../../../etc/passwd`) → Express resolves the `..` at the routing layer, so the request falls outside the `/download/*` handler and you get a generic **404**. No filesystem is touched. If the response's `details.path` echoes `/etc/passwd`, that's cosmetic — the handler never ran.
- Bucket key that doesn't exist → endpoint still returns 200 with a signed URL (the API doesn't round-trip OSS to check existence). The URL itself returns `NoSuchKey` 404 when followed. That's expected.
- `?force_download=true` → with V4 signing, `response-content-disposition=attachment` is baked **into the signature**, not appended as a query-string param. So you can't eyeball it in the URL. To verify, follow the URL and check that the `Content-Disposition` response header OSS sends back contains `attachment`.
- Signed URL expiry: the query carries `x-oss-expires=3600` (seconds). Classic V1 `Expires` timestamp is not present.

---

## 11. `/api/v1/real/*` is gone

**What the user experiences:** Nothing — these were dev-only endpoints.

```
GET /api/v1/real/courses
GET /api/v1/real/user/profile
GET /api/v1/real/courses/1
```

**Expected:** All return **404** with:
```json
{
  "ret_code": 101003,
  "request_id": "...",
  "error": {
    "message": "请求的资源不存在",
    "details": { "path": "/api/v1/real/..." }
  }
}
```

Also confirm:
```bash
curl -s http://localhost:8888/api/v1/docs-json | jq '.paths | keys[]' | grep -c '/real/'
```
→ should print `0`. No `/real/*` in the Swagger spec either.

---

## 12. Unlock bypass flag (ON state)

**What the user experiences:** Every activity/day/card appears unlocked regardless of progress. Enables the Week-1 demo to walk the full tree without completing anything.

**Where it appears:**
- `GET /courses/:id` → every card has `locked: false`, every day has `status: "unlocked"`.
- `GET /courses/:cid/lessons/:lid/parts/:pid` → every day and activity has `status: "unlocked"`, `locked: false`.
- `GET /courses/:id/today` → every activity has `status: "unlocked"`.

**How to test ON:**
- Server already starts with `UNLOCK_BYPASS=true` in dev. Re-run §3, §4, §6 and confirm every status/locked field.

§22 covers the OFF state (toggle and retest).

---

## 13. Swagger documentation

```
GET /api/v1/docs
```

Opens the Swagger UI in a browser. Available in non-production.

**What to check:**
- Page loads, grouped by tag (课程, 用户, 下载, etc.).
- No `/real/*` endpoints appear anywhere (cross-checked via §11).
- `/courses/{id}/today`, `/courses/{id}/progress`, `/sessions/*` all documented.
- "Try it out" works when the access token is pasted into the Authorize dialog.

Also grab the raw spec:
```
GET /api/v1/docs-json
```
Expect valid JSON with every route listed.

---

## 14. Request logging (basic)

**What the user experiences:** Nothing — ops concern. Each request hits stdout.

**How to verify (basic):**
Make any authed request and inspect stdout. You should see a line like:
```
[12:34:56] INFO: GET /api/v1/courses completed with 200
    requestId: "abc-123"
    userId: 1
    responseTime: 15
```

Detailed level-by-level verification is in §21.

---

## 15. The full user journey (end-to-end)

Golden path a real app walks:

1. **Login** → real auth flow (captcha → verify-captcha → OTP → phone-login). Get `access_token`.
2. **See my courses** → `GET /courses` → list with 英语启蒙 highlighted.
3. **Open a course** → `GET /courses/1` → cards + nested days.
4. **Start today's lesson** → `GET /courses/1/today` → Day 1 with 2 activities.
5. **Open Part A** → `GET /courses/1/lessons/1/parts/1` → full day/activity tree.
6. **Start the video activity** → `GET /courses/1/lessons/1/parts/1/days/1/activities/1` → video interaction.
7. **Watch the video** → `GET /download/videos/...?redirect=false` → signed OSS URL → streams.
8. **Start a learning session** → `POST /sessions/start { activity_id: 1, day_id: 1 }`.
9. **Record interactions** → `POST /sessions/:id/interactions`.
10. **Complete the session** → `POST /sessions/:id/complete` → coins awarded.
11. **Check profile** → `GET /user/profile` → `total_coins` went up by `reward_coins`.
12. **Switch course** → `POST /courses/switch { course_id: 2 }`.
13. **Listen to playlist** → `GET /courses/2/playlist`.

Mark PASS only if every step returned 200 and coins incremented correctly.

---

## 16. Auth lifecycle (smoke test) — refresh + logout

These weren't covered in the pre-cleanup guide. Smoke level only; deeper coverage belongs in P5/P6.

### Refresh token

```
POST /api/v1/auth/refresh-token
Body: { "refresh_token": "<from phone-login>" }
```

**Expected:** `200` with a new `access_token` and `expires_in: 3600`.

**What to check:**
- New access token is different from the original.
- Old access token still works until its own expiry (not blacklisted by refresh).
- Invalid refresh token → **401** (`ret_code: 600004`, USER_TOKEN_INVALID).
- Access token passed in place of refresh token → **401** (the service checks `payload.type === 'refresh'`).

### Logout

```
POST /api/v1/auth/logout
Authorization: Bearer <access_token>
```

**Expected:** `200` with a confirmation message.

**What to check:**
- Call succeeds once. (Session invalidation / token blacklist behavior is P5 — for now smoke-test that the endpoint exists and doesn't 500.)
- Without JWT → **401**.

---

## 17. Sessions (smoke test) — start / interactions / complete

### Start a session

```
POST /api/v1/sessions/start
Body: { "activity_id": 1, "day_id": 1 }
```

**Expected:**
```json
{
  "ret_code": 0,
  "data": { "learning_session_id": 123, "activity_id": 1, "day_id": 1 }
}
```

### Record an interaction

```
POST /api/v1/sessions/:learning_session_id/interactions
Body: { "interaction_id": 1, "interaction_type": "video", "payload": { ... } }
```

**Expected:** `{ recorded: true, interaction_id, interaction_type, is_correct, coins_awarded }`.

### Complete the session

```
POST /api/v1/sessions/:learning_session_id/complete
Body: { "time_spent": 30 }
```

**Expected:** `{ completed: true, already_completed: false, learning_session_id, reward_coins: 10, total_coins: 10 }`.

### Idempotency

Call `/complete` again with the same session ID. Expect `already_completed: true` and `total_coins` unchanged.

### What to check overall
- `GET /user/profile` after complete → `total_coins` went up by exactly the activity's `reward_coins` (10 for activity 1), not 20.
- Non-existent activity in `/start` → **404**.
- Missing `activity_id` or `day_id` in `/start` → **400**.
- Non-existent session in `/interactions` or `/complete` → **404**.
- No JWT → **401** on all three.

---

## 18. Course progress

```
GET /api/v1/courses/1/progress
```

**Expected (before §17 runs):**
```json
{
  "ret_code": 0,
  "data": {
    "course_id": 1,
    "course_name": "英语启蒙",
    "total_activities": 3,
    "completed_activities": 0,
    "total_coins_earned": 0,
    "parts": [
      { "part_id": 1, "title": "Part A", "total_activities": 3, "completed_activities": 0 },
      { "part_id": 2, "title": "Part B", "total_activities": 1, "completed_activities": 0 }
    ]
  }
}
```

After completing one activity in §17, expect `completed_activities: 1` and `total_coins_earned: 10`.

**What to check:**
- `total_activities` matches the seed count across all parts.
- Per-part rollup sums to the course-level total.
- Non-existent course → **404**.

---

## 19. JWT lifecycle edge cases

Use a JWT minting helper (or the auth flow) to generate each variant.

| Variant | How to produce | Expected |
|---------|----------------|----------|
| Expired access token | Mint with `expiresIn: '-1s'` (edit a local test helper) | **401** `600004` USER_TOKEN_INVALID |
| Malformed token | `Bearer not.a.valid.token` | **401** `600004` |
| Missing `Bearer ` prefix | `Authorization: <raw-token>` | **401** `600001` USER_NOT_LOGGED_IN |
| Token for a soft-deleted user | Log in, soft-delete user 1 (`update users set deleted=true`), reuse old token | **404** (or 401, depending on middleware — document what you see) |
| Wrong signature | Manually flip a byte in the signature segment | **401** `600004` |
| Swapped refresh-for-access | Use the refresh token where an access token is expected | **401** (middleware rejects — payload `type` doesn't match) |

---

## 20. DB connection loss

Stop Postgres (`pg_ctl -D <datadir> stop`, or kill your docker container). Then:

- `GET /api/v1/health` → **200** with `db: "error"` (graceful — Prisma ping failed but the endpoint itself still returns).
- `GET /api/v1/courses` (authed) → **500** `{ ret_code: 200001 }`, stdout has an `error`-level log line referencing the Prisma failure.
- `POST /api/v1/auth/phone-login` → **500** after OTP verification (the `DATABASE_CONNECTION_FAILED` throw from the A4 fix — no more silent `Date.now()` userId).

Restart Postgres and confirm `/health` flips back to `db: "ok"` on the next request.

---

## 21. Request logging verification (detailed)

Make three deliberate requests while tailing the server stdout:

1. **200 authed** — `GET /api/v1/courses` with a valid token.
2. **4xx** — `GET /api/v1/courses/9999` with a valid token (→ 404).
3. **5xx** — stop Postgres and hit `GET /api/v1/courses` (→ 500). Restart PG after.

**What to check:**
- The 200 line is logged at `info` level, includes `method`, `url`, `statusCode`, `responseTime`, `userId: 1`, `requestId`.
- The 4xx line is logged at `warn` level, includes `statusCode: 404` and `userId: 1`.
- The 5xx line is logged at `error` level, includes `statusCode: 500` and the error message in the context.
- `/api/v1/health` (unauthenticated) logs at `info` with **no** `userId` field.
- Every log line has a unique `requestId` that matches the `request_id` in the response body.

---

## 22. Unlock bypass (OFF state)

Restart the server with `UNLOCK_BYPASS=false`:

```bash
UNLOCK_BYPASS=false pnpm dev
```

Re-run §3, §4, §6 and document what changes. Expected behavior (best guess — P4 didn't ship the real unlock engine, so this exposes the fallback):

- Activities past `user_courses.current_day` should show `locked: true` and `status: "locked"`.
- Cards beyond the current part should show `locked: true`.
- Today's content (§4) for the user's current day should still be `unlocked`.

If the code currently hardcodes `'unlocked'` in both branches (grep `PartsService.js` for the ternary on line 65–78), document that as a known limitation — real unlock logic lands in **P5**, per Contract §2.4.

---

## 23. Regression checklist — the three defects from 2026-04-23

These should all PASS on the fresh codebase. Flag any failure immediately.

### 23a. Mock JWT userId mismatch (was BLOCKING)

The legacy `MOCK_JWTTOKEN` (userId=12345 in an unknown-secret JWT) was deleted.

**Verify:**
- Going through the real auth flow (captcha → verify-captcha → OTP → phone-login) with phone `18601113318` produces an access token bound to `user_id: 1`.
- `GET /user/profile` returns user 1's data (not 404).
- `POST /courses/switch { course_id: 2 }` succeeds (not 404).
- Full §17 sessions loop runs cleanly with this token.

### 23b. OTP endpoint crash (was FAIL)

`GET /auth/otp` used to always call real Aliyun SMS and crash.

**Verify:**
- `GET /api/v1/auth/otp?phone=18601113318` returns **200** with `{ phone, expires_in: 300 }`.
- Server stdout shows: `WARN: ALIYUN_SMS_DISABLED=true — OTP stored but SMS not sent. Use this code to log in.` with `otp: "123456"` in the context.
- Subsequent `POST /auth/phone-login` with `sms_code: "123456"` succeeds.
- In production (`NODE_ENV=production`, `ALIYUN_SMS_DISABLED` unset), the real SMS path runs — this is not verifiable locally without prod creds, so just grep the code: `authService.js` branches on `process.env.ALIYUN_SMS_DISABLED === 'true'`.

### 23c. Phantom `/real/user/stats` Swagger (was PARTIAL)

**Verify:**
```bash
curl -s http://localhost:8888/api/v1/docs-json | jq '.paths | keys[]' | grep -c '/real/'
```
→ prints `0`.

No `/real/*` in `src/api/**` either:
```bash
git grep -n '/real/' yyddp-api/src/api
```
→ empty.

---

## Error response format

All errors follow:

```json
{
  "ret_code": 600001,
  "request_id": "...",
  "error": {
    "message": "用户未登录",
    "details": { "message": "Authentication required. Please provide a valid JWT token." }
  }
}
```

Common codes:

| Code | Meaning | HTTP |
|------|---------|------|
| 0 | Success | 200 |
| 100001 | Missing required parameter | 400 |
| 100005 | Invalid parameter value | 400 |
| 101003 | Resource not found | 404 |
| 200001 | Internal server error | 500 |
| 300007 | External service unavailable (OSS etc.) | 500 |
| 600001 | Not logged in (no JWT) | 401 |
| 600004 | Invalid JWT token | 401 |

---

## Automated test suite

All of the above (happy paths + most error paths) is also covered by automated tests:

```bash
cd yyddp-api
pnpm test
```

Current state: **72 passing, 0 failing** against the seeded `yyddp_test` database.

The tests mirror the sections of this guide. When a section here changes, update the corresponding `*.test.js` file to match.
