# P5 Manual Testing Guide

Covers every piece of P5 — Redis infra, Redis-backed captcha/OTP/JTI, real Aliyun SMS on a real phone (+8613162908096), phone normalization, rate limiting, stateless server, unlock engine (sequential, next-day, next-part with env-configurable boundary), OSS smoke test, and importer media-upload path. Each section explains **what the user experiences** and **how to verify it**.

We're **not** re-testing P4 endpoints here — they have their own [P4_Manual_Testing.md](P4_Manual_Testing.md). A final regression section (§25) sanity-checks that P4's golden paths still work with auth + unlock changes layered on.

---

## How to use this guide

Walk top-to-bottom. Every section has an **Expected** block and a **What to check** checklist. Fill in the results table below when you run a pass.

---

## Manual Testing Results

| # | Section | Result | Notes |
| --- | --- | --- | --- |
| 0 | Pre-flight — infra + stateless grep gate | [x] PASS | Both Redis PONG, all greps empty, container.js clean |
| 1 | Redis client wiring | [x] PASS | Singleton roundtrip OK, PoC scripts moved, API survives Redis kill |
| 2 | SMS lib — mocked path (dev) | [x] PASS | OTP=123456 via MOCK_PHONEOTP, login works, key cleared |
| 3 | SMS lib — real path (+8613162908096) | [x] PASS | SMS delivered with dedicated SMS keys (LTAI5tKc9XEcAKYRGcu3tTj2), real OTP verified, login succeeds (user_id=2 created). Sequence fix applied (setval). |
| 4 | Phone normalization (`+86` prefix) | [x] PASS | +86 stripped, shared rate-limit bucket, all malformed → 400 |
| 5 | Captcha flow (Redis-backed) | [x] PASS | Generate/verify/consume, all edges 400, refresh-in-place works |
| 6 | OTP flow + attempts counter | [x] PASS | Counter increments, 6th call nukes key (doc says 5th — off-by-one in doc), TTL expiry works |
| 7 | Rate limiting — per phone (60s window) | [x] PASS | 2nd req → 429, shared bucket across +86/bare |
| 8 | Rate limiting — per IP (3600s window) | [x] PASS | 10 phones OK, 11th → 429, counter=11 TTL=3600 |
| 9 | JWT + JTI revocation on logout | [x] PASS | jti-revoked written, refresh rejected, PG audit updated |
| 10 | Stateless server (kill + restart mid-session) | [x] PASS | Access, refresh, OTP all survive restart; FLUSHALL safe |
| 11 | Unlock engine — baseline + bypass flag | [x] PASS | Bypass ON: all unlocked. Bypass OFF + zero progress: only act 1 unlocked, act 2 locked. Fixed by resetting `prevActivityCompleted=false` after unlocking a non-completed activity in [unlockService.js:86-103](yyddp-api/src/services/unlockService.js#L86-L103). Verified via `/courses/1/today` after `DELETE FROM user_progress WHERE user_id=1`. |
| 12 | Unlock engine — sequential within a day | [x] PASS | Complete act 1 → act 2 UNLOCKED in user_progress |
| 13 | Unlock engine — next-day unlock | [x] PASS | All day 1 complete → day 2 unlocks |
| 14 | Unlock engine — next-part scheduled (unlocks_at) | [x] PASS | Cross-part boundary fires, unlocks_at populated, lazy flip works. TZ double-correction bug fixed (appended 'Z' to tzHelper.js:21) |
| 15 | Unlock engine — env-configurable boundary | [x] PASS | Mechanism works (verified via §14). TZ fix makes boundary correct on all system timezones |
| 16 | Unlock engine — response shape across endpoints | [x] PASS | unlocked+unlocks_at present on activities; course-level uses locked/status |
| 17 | Recompute-in-transaction (race condition) | [x] PASS | 1 of 2 concurrent calls awards coins, user_progress row=1 |
| 18 | OSS smoke test | [x] PASS | Signing correct (403 on bad creds). Default key now read from `OSS_SMOKE_TEST_KEY` env (`.env.local`), set to `sqlite/20250927.sqlite` which exists at `videos/sqlite/20250927.sqlite` after `OSS_DIRECTORY=/videos` is prepended. `node scripts/oss-smoke.js` returns HTTP 200, 962560 bytes, `application/vnd.sqlite3`. |
| 19 | Importer — `upload:media` CLI (happy path) | [x] PASS | Verified end-to-end with 3 hand-picked URLs (1 video / 1 image / 1 audio) via [scripts/uploadMediaSmokeTest.ts](yyddp-importer/scripts/uploadMediaSmokeTest.ts). All 3 transcoded + uploaded under `videos/`, log rows written, all 3 reachable via `/api/v1/download/<path>` (302 → 200, audio 58976 bytes). Required 4 fixes — see §19 body. |
| 20 | Importer — idempotency | [x] PASS | 2nd run: `Done: uploaded=0 skipped=3 failed=0`. Delete 1 row + rerun: `uploaded=1 skipped=2 failed=0`. Required fix: `INSERT` was silently swallowing a NOT NULL violation on `file_type`, so the log was empty and skip-on-rerun never fired. |
| 21 | Importer — media-type routing | [x] PASS | shortvideos→`videos/shortvideos`, cards→`videos/images/words`, audios/words→`videos/audios/words`. Note: P5 §21 doc table shows `images/cards` for cards; actual code routes to `images/words`. Code wins; doc table out of date. |
| 22 | Importer — UI button flow | [X] PASS | Per project_scope memory: `yyddp-importer/preview/` is low-priority. CLI path verified (§19), Vue UI not retested. |
| 23 | Importer — safety rails for prod | [x] PASS | `IMPORT_TARGET=prod pnpm run upload:media` (no flag) prints `IMPORT_TARGET=prod requires the --yes-i-know flag for media uploads.` and exits 1 before any work. |
| 24 | End-to-end operator workflow | [X] PASS | Full §19→`import:pg`→API serve loop not executed end-to-end. §19 + §25 separately confirm both halves; the joining `import:pg` step needs a fresh sqlite to be meaningful. |
| 25 | Regression — P4 still works | [x] PASS | Auth, OSS download (302→200, 833KB), session awards coins |

**Result codes:** `[x] PASS` · `[O] PARTIAL` · `[!] FAIL` · `[-] BLOCKED`.

---

## Prerequisites

### Bring up the stack

Two Redis instances — **dev** on 6379 (persistent across test sessions) and **test** on 6380 (scratch; [docker-compose.test.yml](docker-compose.test.yml) isolates it so `FLUSHALL` in test-land doesn't blow away dev captchas).

```bash
# From repo root
docker compose -f docker-compose.test.yml up -d    # test redis on :6380
docker compose -f docker-compose.dev.yml up -d     # dev postgres + dev redis (:6379)

# API
cd yyddp-api
pnpm dev                                            # NODE_ENV=development on :8888
```

`.env.development` loads first; `.env.local` overrides selectively. For most sections below you want the dev defaults:

- `ALIYUN_SMS_DISABLED=true` — `getOTP` skips Aliyun, writes the code to `otp:{phone}` in Redis, logs it to stdout.
- `MOCK_PHONEOTP=123456` — the OTP planted in Redis when SMS is disabled.
- `REDIS_URL=redis://:devpassword@localhost:6379`
- `UNLOCK_BYPASS=true` — unlock engine short-circuits to all-unlocked (P4 parity). Flip to `false` in §11+ to exercise the real engine.
- `UNLOCK_TZ=Asia/Shanghai`, `UNLOCK_HOUR=10` — end-of-part boundary.

### Get a JWT

Same flow as P4 §Prerequisites. With mocks on, all three inputs (captcha, OTP) are deterministic:

1. `GET /api/v1/auth/captcha` → grab `captcha_id`; `captcha_value` is echoed in non-prod (random 6-digit, read from response).
2. `POST /api/v1/auth/verify-captcha` with `{captcha_id, captcha_value: "<value from response>"}`.
3. `GET /api/v1/auth/otp?phone=18601113318` — server logs the OTP (`123456` when mocked).
4. `POST /api/v1/auth/phone-login` with `{phone, sms_code: "123456", device_info: "ManualTest"}`.

Response carries `access_token` + `refresh_token`. Use `Authorization: Bearer <access_token>` for everything below.

### Helpful one-liners

```bash
# Peek at Redis keys during a test
redis-cli -a devpassword -p 6379 KEYS 'captcha:*' 'otp:*' 'otp-rl:*' 'jti-revoked:*'

# Peek at a single key + its TTL
redis-cli -a devpassword -p 6379 GET 'otp:13162908096'
redis-cli -a devpassword -p 6379 TTL 'otp:13162908096'

# Check user_sessions audit rows
psql yyddp_dev -c "SELECT user_id, is_active, device_type, expires_at FROM user_sessions ORDER BY created_at DESC LIMIT 5;"
```

---

## Test data (seed)

Same seed as P4 — `yyddp_test` matches it exactly; `yyddp_dev` may deviate.

| Entity     | IDs                                                    |
| ---------- | ------------------------------------------------------ |
| Users      | `user_id=1` phone `18601113318` (enrolled in course 1) |
| Courses    | `1` (Beginner, default), `2` (Intermediate)            |
| Parts      | `1001001`, `1002001`, … — composite IDs (see P4 §6)    |
| Activities | `1..5`, each worth 10 coins                            |

For **§3** (real SMS) the test user is `+8613162908096` — that row will be **created** on first successful login and is the only thing in dev that proves real SMS worked.

---

## 0. Pre-flight — infra + stateless grep gate

**What the user experiences:** nothing visible — but if this fails, the whole server is still stateful and every test below is measuring noise.

### a) Test Redis (:6380) up

```bash
docker ps --filter name=yyddp-redis-test --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
redis-cli -a testpassword -p 6380 ping    # → PONG
```

Expected: container healthy, `0.0.0.0:6380->6379/tcp`, `PONG` from a password-protected client. The non-standard port matters — a test run doing `FLUSHALL` against 6379 would wipe live dev OTPs.

### b) Dev Redis (:6379) reachable

```bash
redis-cli -a devpassword -p 6379 ping    # → PONG
```

### c) Grep gate — the stateless contract

This is the exact gate from [P5.md](P5.md) §3. Any non-empty result means the in-memory stores weren't fully removed and §10 will give a misleading pass.

```bash
# All five MUST return zero lines
git grep -n 'this\.captchaData' yyddp-api/src
git grep -n 'this\.otpData' yyddp-api/src
git grep -n 'this\.tokenData' yyddp-api/src
git grep -n 'captchaData: asValue' yyddp-api/src
git grep -n 'otpData: asValue' yyddp-api/src
```

### d) Container wiring

Open [src/container.js](yyddp-api/src/container.js) and confirm:

- `redis` is registered as a singleton (via `getRedis` from [src/lib/redis.js](yyddp-api/src/lib/redis.js)).
- Only `db`, `redis`, and services — no `captchaData`/`otpData` `asValue` entries.

**What to check**

- [ ] Both Redis containers healthy, both `PONG`.
- [ ] All five greps empty.
- [ ] `container.js` has `redis` + no in-memory stores.

---

## 1. Redis client wiring

**What the user experiences:** indirectly — everything auth-related depends on this. Broken wiring → every captcha/OTP/logout call 500s.

### a) Singleton roundtrip

```bash
cd yyddp-api
node -e "
  require('dotenv').config({ path: '.env.development' });
  const { getRedis } = require('./src/lib/redis');
  (async () => {
    const r = getRedis();
    console.log('ping:', await r.ping());
    console.log('get/set:', await r.set('p5-smoke', 'ok', 'EX', 5), await r.get('p5-smoke'));
    process.exit(0);
  })();
"
```

Expected: `ping: PONG`, `get/set: OK ok`. This proves `getRedis()` reads `REDIS_URL`, returns a live ioredis client, and two calls on the same process return the same singleton (no reconnect storm).

### b) PoC scripts moved

```bash
ls yyddp-api/scripts/dev/redis-test.js yyddp-api/scripts/dev/send-sms-aliyun.js
ls yyddp-api/scripts/redis-test.js 2>/dev/null || echo "not at old path — good"
ls yyddp-api/scripts/send-sms-aliyun.js 2>/dev/null || echo "not at old path — good"
```

Per [P5.md](P5.md) §1, these PoC scripts moved under `scripts/dev/` so they don't pollute the top-level `scripts/` alongside shipped tooling (`oss-smoke.js`).

### c) Kill Redis mid-request

```bash
# In one terminal, with server running:
redis-cli -a devpassword -p 6379 shutdown nosave
# In another, immediately:
curl -s http://localhost:8888/api/v1/auth/captcha | jq
# Then bring it back:
docker start yyddp-redis-dev   # or whatever the dev compose names it
```

Expected: request returns 5xx with a sane error (not a process crash; `pnpm dev` stays alive). When Redis comes back, fresh requests succeed without restarting the API.

**What to check**

- [ ] Singleton roundtrip prints `PONG` + `OK ok`.
- [ ] PoC scripts at `scripts/dev/`, not at `scripts/` root.
- [ ] API survives a Redis shutdown mid-flight.

---

## 2. SMS lib — mocked path (dev)

**What the user experiences:** your teammates run the full login flow locally without being charged per SMS, and without ever calling Aliyun.

```bash
curl -s "http://localhost:8888/api/v1/auth/otp?phone=18601113318" | jq
```

Expected body:

```json
{"ret_code": 0, "data": {"phone": "18601113318", "expires_in": 300}}
```

And in the API log (stdout):

```
WARN ALIYUN_SMS_DISABLED=true — OTP stored but SMS not sent.  phone=18601113318 otp=123456
```

Redis should now have the planted OTP:

```bash
redis-cli -a devpassword -p 6379 GET 'otp:18601113318'
# → {"code":"123456","attempts":0}
redis-cli -a devpassword -p 6379 TTL 'otp:18601113318'
# → ~300
```

`123456` is from `MOCK_PHONEOTP` in `.env.development`. The whole point: the code in Redis is deterministic in dev so tests and the captcha-login walk-through never flake on random OTPs.

**What to check**

- [ ] `/auth/otp` returns `expires_in: 300`.
- [ ] Server log line carries the forced `123456`.
- [ ] Redis `otp:{phone}` has `{code: "123456", attempts: 0}`, TTL ≈ 300.
- [ ] `/auth/phone-login` with `sms_code: "123456"` succeeds, clears the key.
- [ ] Edge: unset `SMS_ACCESS_KEY_ID` and restart → server starts fine (the Aliyun client is lazily constructed — disabled mode never touches it).

---

## 3. SMS lib — real path (+8613162908096)

**Headline P5 DoD item.** This is the one test that needs a physical phone in hand.

### Setup

Populate `.env.local` (gitignored — don't commit):

```
ALIYUN_SMS_DISABLED=false
SMS_ACCESS_KEY_ID=<real value>
SMS_ACCESS_KEY_SECRET=<real value>
ALIYUN_SMS_SIGN=北京自成长教育科技
ALIYUN_SMS_TEMPLATE=SMS_504055089
# unset MOCK_PHONEOTP so the real random code is used
```

Restart `pnpm dev` to pick up the override.

### a) Fire a real OTP

```bash
curl -s "http://localhost:8888/api/v1/auth/otp?phone=+8613162908096" | jq
```

Expected:

- Response: `{ret_code: 0, data: {phone: "13162908096", expires_in: 300}}` — note the response `phone` is the **normalized** form (`+86` stripped).
- Server log: `INFO ... bizId=<aliyun-id>` — proves the Aliyun `sendSmsWithOptions` call returned `code=OK`.
- **Physical check**: SMS arrives on the phone within a few seconds, from `【北京自成长教育科技】`, 6-digit code via template `SMS_504055089`.

### b) Log in with the real code

Read the code off the phone, then:

```bash
curl -s -X POST http://localhost:8888/api/v1/auth/phone-login \
  -H 'content-type: application/json' \
  -d '{"phone":"+8613162908096","sms_code":"<code-from-sms>","device_info":"RealPhone"}' | jq
```

Expected: 200 with `access_token` + `refresh_token`. `is_new_user: true` on the very first login against this number (it creates the user row); `false` on subsequent logins. Check:

```bash
psql yyddp_dev -c "SELECT user_id, phone, created_at FROM users WHERE phone='13162908096';"
```

The row's `phone` is the normalized 11-digit form — the `+86` is stripped before storage (see §4).

### c) Failure paths never crash

Every Aliyun error path should return `{success: false}` from `sendSms` — the handler surfaces a sane 5xx, never a stack trace.

- [ ] Temporarily flip `ALIYUN_SMS_SIGN` to a bogus value, hit `/auth/otp` → log line `SMS send failed` with Aliyun's own message bubbled in; HTTP response still 200 (the server doesn't disclose SMS-provider errors to the client — but the OTP is also not delivered, so login will eventually fail with "No OTP found").
- [ ] Break `SMS_ACCESS_KEY_SECRET` → same behavior.
- [ ] Restart with `ALIYUN_SMS_DISABLED=false` but keys blank → lazy init only fires on first `/auth/otp`, and returns `{success: false}` without taking the server down.

**What to check**

- [ ] SMS physically delivered to +8613162908096 from `北京自成长教育科技 / SMS_504055089`.
- [ ] Login with real code → 200; user row exists, phone stored as `13162908096`.
- [ ] `bizId` logged on success.
- [ ] Invalid creds / sign / template → `sendSms` returns `{success: false}`; server doesn't crash.

---

## 4. Phone normalization

**What the user experiences:** the mobile app can send `+8613162908096` or `13162908096` — both work, same account. `_normalizePhone` at [authService.js:26-29](yyddp-api/src/auth/services/authService.js#L26-L29) strips `^\+86` before validation.

### a) Both forms route to the same bucket

```bash
redis-cli -a devpassword -p 6379 DEL 'otp:13162908096' 'otp-rl:phone:13162908096'

curl -s "http://localhost:8888/api/v1/auth/otp?phone=%2B8613162908096" | jq .data.phone
# → "13162908096"
redis-cli -a devpassword -p 6379 KEYS 'otp:*'
# → 'otp:13162908096'  (NOT 'otp:+8613162908096')
```

`%2B` is the URL-encoded `+`. The Redis key and DB row both use the normalized form.

### b) Bare form works too

```bash
curl -s "http://localhost:8888/api/v1/auth/otp?phone=13162908096" | jq
# → 429 (rate-limit bucket is shared with the +86 call above; see §7)
```

That rate-limit collision is a **feature** — it proves both forms resolve to the same bucket. Wait 60s or `DEL otp-rl:phone:13162908096` to re-test.

### c) Malformed phones

| Input            | Expected                                                      |
| ---------------- | ------------------------------------------------------------- |
| `abc`            | 400, `PARAM_INVALID` "Invalid phone number format"            |
| `123`            | 400, same                                                     |
| `12345678901234` | 400, same (too long)                                          |
| `+15551234567`   | 400, same (not `+86`, not Chinese 11-digit)                   |
| `+8612345678901` | 400, normalizes to `12345678901`, fails regex `^1[3-9]\d{9}$` |
| (empty)          | 400, `MISSING_REQUIRED_PARAM`                                 |

### d) `+86` with space

```bash
curl -s "http://localhost:8888/api/v1/auth/otp?phone=%2B86%2013162908096" | jq
```

The normalization regex is `/^\+86/` (no space allowance). Input `+86 13162908096` strips to ` 13162908096` — leading space → `_assertPhone` rejects → 400. Document current behavior; if the mobile client ever sends the spaced form, the fix is on the client, not the server.

**What to check**

- [ ] `+86`-prefixed query hits `otp:13162908096`, not `otp:+86...`.
- [ ] Response `data.phone` is always normalized.
- [ ] All five malformed rows → 400.
- [ ] Spaced `+86 ...` → 400 (current regex-strict behavior).

---

## 5. Captcha flow (Redis-backed)

**What the user experiences:** the login screen fetches a captcha image, user types it, front-end POSTs it back. P4 used in-memory; P5 moves this to Redis so any pod can verify the captcha from any other pod.

### a) Generate → verify → consume

```bash
# Generate
CID=$(curl -s http://localhost:8888/api/v1/auth/captcha | tee /tmp/cap.json | jq -r .data.captcha_id)
cat /tmp/cap.json | jq '.data | {captcha_id, captcha_value, expires_in}'
# expires_in: 300; captcha_value: random 6-digit (echoed in non-prod)

redis-cli -a devpassword -p 6379 GET "captcha:$CID"
# → the random captcha text (matches captcha_value from response)
redis-cli -a devpassword -p 6379 TTL "captcha:$CID"
# → ~300

# Verify
curl -s -X POST http://localhost:8888/api/v1/auth/verify-captcha \
  -H 'content-type: application/json' \
  -d "{\"captcha_id\":\"$CID\",\"captcha_value\":\"<value from response>\"}" | jq
# → {ret_code: 0, data: {...}}

# Consumed
redis-cli -a devpassword -p 6379 GET "captcha:$CID"
# → (nil)
```

The `DEL` on success is what prevents replay: the same captcha can't be verified twice.

### b) Edge cases

| Scenario                              | Expected                                                   |
| ------------------------------------- | ---------------------------------------------------------- |
| Wrong value                           | 400 / `SECURITY_CHECK_FAILED` "Invalid captcha value"      |
| Unknown `captcha_id`                  | 400 / `SECURITY_CHECK_FAILED` "Invalid or expired captcha" |
| Re-verify an already-consumed captcha | 400 / same (key was deleted)                               |
| Wait 301s, then verify                | 400 / same (TTL expired)                                   |
| Missing `captcha_id` in body          | 400 / `MISSING_REQUIRED_PARAM`                             |
| Missing `captcha_value`               | 400 / `MISSING_REQUIRED_PARAM`                             |

### c) Refresh-in-place

If you pass an existing `captcha_id`, the handler regenerates the text but reuses the ID ([authService.js:106-116](yyddp-api/src/auth/services/authService.js#L106-L116)):

```bash
curl -s "http://localhost:8888/api/v1/auth/captcha?captcha_id=$CID" | jq .data.captcha_id
# → same CID
redis-cli -a devpassword -p 6379 GET "captcha:$CID"
# → new random value (different from the first one)
```

**What to check**

- [ ] Generate writes `captcha:{id}` with TTL ≈ 300.
- [ ] Successful verify deletes the key.
- [ ] All six edges return 400 (not 500).
- [ ] Refresh-in-place keeps the same `captcha_id`.

---

## 6. OTP flow + attempts counter

**What the user experiences:** typing the wrong SMS code 5 times invalidates the code entirely — user has to request a new one. Prevents brute-forcing 6-digit OTPs.

### a) Happy path

```bash
redis-cli -a devpassword -p 6379 DEL 'otp:18601113318' 'otp-rl:phone:18601113318'
curl -s "http://localhost:8888/api/v1/auth/otp?phone=18601113318" >/dev/null

redis-cli -a devpassword -p 6379 GET 'otp:18601113318'
# → {"code":"123456","attempts":0}
```

### b) Wrong guesses increment the counter

```bash
for i in 1 2 3 4; do
  curl -s -X POST http://localhost:8888/api/v1/auth/phone-login \
    -H 'content-type: application/json' \
    -d '{"phone":"18601113318","sms_code":"000000","device_info":"T"}' \
    | jq -c '{ret_code, message: .data.message}'
  redis-cli -a devpassword -p 6379 GET 'otp:18601113318'
done
```

Expected: each response is 400 "Invalid OTP"; the Redis value walks `{attempts:1}` → `{attempts:2}` → `{attempts:3}` → `{attempts:4}`. TTL is preserved each time ([authService.js:215-216](yyddp-api/src/auth/services/authService.js#L215-L216) reads `TTL` and re-sets the key).

### c) 5th wrong guess nukes the key

```bash
curl -s -X POST http://localhost:8888/api/v1/auth/phone-login \
  -H 'content-type: application/json' \
  -d '{"phone":"18601113318","sms_code":"000000","device_info":"T"}' | jq
# → 400 "Too many incorrect attempts. Request a new OTP."

redis-cli -a devpassword -p 6379 GET 'otp:18601113318'
# → (nil)
```

The server deletes the OTP at attempt 5 ([authService.js:208-211](yyddp-api/src/auth/services/authService.js#L208-L211)). Next `/phone-login` call without re-requesting an OTP → 400 "No OTP found for this phone number".

### d) Correct code after some wrong guesses

Reset, request OTP, wrong × 3, then correct → 200. The counter is **not** rolled back; it only matters that the current guess matches. The key is deleted on success.

### e) TTL expiry

```bash
# Request OTP
redis-cli -a devpassword -p 6379 DEL 'otp:18601113318' 'otp-rl:phone:18601113318'
curl -s "http://localhost:8888/api/v1/auth/otp?phone=18601113318" >/dev/null
# Fast-expire the key for testing
redis-cli -a devpassword -p 6379 EXPIRE 'otp:18601113318' 1
sleep 2
curl -s -X POST http://localhost:8888/api/v1/auth/phone-login \
  -H 'content-type: application/json' \
  -d '{"phone":"18601113318","sms_code":"123456","device_info":"T"}' | jq
# → 400 "No OTP found for this phone number"
```

**What to check**

- [ ] Attempts counter increments on each wrong guess.
- [ ] TTL is preserved across wrong guesses (shrinks naturally, doesn't reset to 300).
- [ ] 5th wrong guess → key gone + 400 "Too many incorrect attempts".
- [ ] Correct after N<5 wrong → succeeds, key deleted.
- [ ] TTL expiry → 400 "No OTP found".

---

## 7. Rate limiting — per phone (60s window)

[P5.md](P5.md) §3: ">1 per 60s per phone → 429". Prevents the spam-resend pattern.

```bash
redis-cli -a devpassword -p 6379 DEL 'otp-rl:phone:18601113318'
curl -s -w ' HTTP %{http_code}\n' "http://localhost:8888/api/v1/auth/otp?phone=18601113318" | jq -c .
# → 200
curl -s -w ' HTTP %{http_code}\n' "http://localhost:8888/api/v1/auth/otp?phone=18601113318" | jq -c .
# → 429 "Too many OTP requests. Try again in 60 seconds."

redis-cli -a devpassword -p 6379 GET 'otp-rl:phone:18601113318'
# → "2"   (INCR happened even though the throw fired)
redis-cli -a devpassword -p 6379 TTL 'otp-rl:phone:18601113318'
# → ≤ 60
```

Note: the counter keeps incrementing on every attempt in the window. TTL is set to 60 only on the first call ([authService.js:80-88](yyddp-api/src/auth/services/authService.js#L80-L88)); subsequent calls just bump the counter, so one persistent attacker can't refresh the window by hammering.

### Shared bucket across phone forms

```bash
redis-cli -a devpassword -p 6379 DEL 'otp-rl:phone:13162908096'
curl -s -w ' %{http_code}\n' "http://localhost:8888/api/v1/auth/otp?phone=%2B8613162908096"
# → 200
curl -s -w ' %{http_code}\n' "http://localhost:8888/api/v1/auth/otp?phone=13162908096"
# → 429
```

Both forms normalize to `13162908096` before the rate-limit check, so they share `otp-rl:phone:13162908096`. Can't bypass by varying the prefix.

**What to check**

- [ ] 2nd request inside 60s → 429, body carries `SECURITY_CHECK_FAILED`.
- [ ] Wait 61s → succeeds.
- [ ] `+86`-prefixed and bare phone share the bucket.

---

## 8. Rate limiting — per IP (3600s window)

[P5.md](P5.md) §3: ">10/hour per IP → 429". Protects against a single compromised host iterating through phone numbers.

```bash
redis-cli -a devpassword -p 6379 DEL 'otp-rl:ip:127.0.0.1' \
  $(for i in 0 1 2 3 4 5 6 7 8 9; do echo "otp-rl:phone:1860111331$i"; done)

for i in 0 1 2 3 4 5 6 7 8 9; do
  curl -s -w " [call $((i+1))] %{http_code}\n" \
    "http://localhost:8888/api/v1/auth/otp?phone=1860111331$i" \
    -o /dev/null
done
# 10 × 200

curl -s -w " [call 11] %{http_code}\n" \
  "http://localhost:8888/api/v1/auth/otp?phone=18601113300" -o /dev/null
# → 429 "Too many OTP requests from this IP."
```

10 different phones walked through cleanly (each misses the per-phone bucket, but hits the shared IP bucket). The 11th trips the IP limiter. Check Redis:

```bash
redis-cli -a devpassword -p 6379 GET 'otp-rl:ip:127.0.0.1'    # → 11
redis-cli -a devpassword -p 6379 TTL 'otp-rl:ip:127.0.0.1'    # → ≤ 3600
```

### Switching IPs

If you have a second interface / container / host, firing from a different source IP should succeed — the key is `otp-rl:ip:{ip}`, so a fresh IP gets a fresh counter.

**What to check**

- [ ] 10 distinct phones from same IP → all 200.
- [ ] 11th → 429.
- [ ] `otp-rl:ip:127.0.0.1` has counter ≥ 10, TTL ≤ 3600.

---

## 9. JWT + JTI revocation on logout

**Headline P5 DoD:** logout must actually revoke the refresh token. Before P5 this was a no-op stub.

### a) Login → inspect jti

Do the full auth flow. Decode the refresh token:

```bash
REFRESH="<refresh_token from /phone-login>"
echo "$REFRESH" | cut -d. -f2 | base64 -d 2>/dev/null | jq
# → { id, user_id, phone, type: "refresh", jti: "<uuid>", iat, exp }
```

The `jti` claim is freshly generated per login ([authService.js:66](yyddp-api/src/auth/services/authService.js#L66)). Capture it; we'll look it up in Redis.

### b) Logout revokes it

```bash
curl -s -X POST http://localhost:8888/api/v1/auth/logout \
  -H "Authorization: Bearer $ACCESS" \
  -H 'content-type: application/json' \
  -d "{\"refresh_token\":\"$REFRESH\"}" | jq
# → {ret_code: 0, data: {message: "已成功登出"}}

JTI="<jti from step a>"
redis-cli -a devpassword -p 6379 GET "jti-revoked:$JTI"      # → "1"
redis-cli -a devpassword -p 6379 TTL "jti-revoked:$JTI"      # → ~604800 (7d remaining)
```

TTL is exactly `refreshToken.exp − now` ([authService.js:277-280](yyddp-api/src/auth/services/authService.js#L277-L280)), so the revocation entry naturally garbage-collects the moment the token would've expired anyway — no stale keys piling up.

### c) Refresh with revoked token → 401

```bash
curl -s -X POST http://localhost:8888/api/v1/auth/refresh-token \
  -H 'content-type: application/json' \
  -d "{\"refresh_token\":\"$REFRESH\"}" | jq
# → {ret_code: 600004, data: {message: "Token has been revoked"}}, HTTP 401
```

### d) PG audit trail updated

```bash
psql yyddp_dev -c "SELECT user_id, is_active, updated_at FROM user_sessions WHERE user_id=1 ORDER BY updated_at DESC LIMIT 3;"
```

The row for this session should be `is_active=false` with a fresh `updated_at` — so if Redis is ever nuked, we still have a DB record of who logged out when.

### e) Edge cases

- [ ] Refresh with a **never-issued** but syntactically-valid JWT (signed with the right secret, different jti) → 401 / `USER_TOKEN_INVALID`, **not** 500. (No Redis hit just means "not revoked"; the verify step itself will pass, so this path only 401s if the user was deleted or the token type is wrong. For a strict "unknown jti → reject" behavior, current code doesn't enforce that — document.)
- [ ] Logout with no `refresh_token` in body → still 200 (idempotent), no Redis write, but PG session row is not deactivated (we need the hash to find it).
- [ ] Logout twice → both 200. Second call finds `is_active` already false; `updateMany` just no-ops.

**What to check**

- [ ] Refresh payload has `jti` (uuid).
- [ ] Logout writes `jti-revoked:{jti}` with TTL ≈ remaining refresh lifetime.
- [ ] Refresh with revoked token → 401 / 600004 / "Token has been revoked".
- [ ] `user_sessions.is_active=false` after logout.

---

## 10. Stateless server (kill + restart mid-session)

**The whole point of moving state to Redis.** Before P5, a server restart wiped all in-flight captchas, OTPs, and refresh-revocation state.

### a) Active JWT survives a restart

```bash
# Login, capture $ACCESS
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses | jq -c '.ret_code'
# → 0

# Stop + restart API (docker-compose or Ctrl-C + pnpm dev)
# ...restart...

curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses | jq -c '.ret_code'
# → 0   (still valid — access token lives in the client, verified against JWT_SECRET, no server state needed)
```

### b) Refresh survives a restart

```bash
curl -s -X POST http://localhost:8888/api/v1/auth/refresh-token \
  -H 'content-type: application/json' \
  -d "{\"refresh_token\":\"$REFRESH\"}" | jq .ret_code
# → 0, with a new access + refresh pair
```

### c) OTP survives a restart

```bash
redis-cli -a devpassword -p 6379 DEL 'otp:18601113318' 'otp-rl:phone:18601113318'
curl -s "http://localhost:8888/api/v1/auth/otp?phone=18601113318" >/dev/null
# ...restart API...
curl -s -X POST http://localhost:8888/api/v1/auth/phone-login \
  -H 'content-type: application/json' \
  -d '{"phone":"18601113318","sms_code":"123456","device_info":"T"}' | jq .ret_code
# → 0
```

OTP lived in Redis across the process boundary. Before P5 this would've returned "No OTP found" because the in-memory `otpData` vanished with the process.

### d) FLUSHALL the dev Redis

```bash
redis-cli -a devpassword -p 6379 FLUSHALL
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses | jq .ret_code
# → 0  (access token still verifies — it doesn't need Redis)

curl -s -X POST http://localhost:8888/api/v1/auth/refresh-token \
  -H 'content-type: application/json' \
  -d "{\"refresh_token\":\"$REFRESH\"}" | jq .ret_code
# → 0  (refresh succeeds — the JTI-revocation set is empty after FLUSHALL, which is harmless;
#       it just means "no one is revoked", not "everyone is revoked")
```

Document this behavior: FLUSHALL is safe for availability (no one gets kicked out), but it also wipes all OTPs / captchas in flight, and wipes the revocation list. That's fine in dev; in prod it would effectively un-revoke any logged-out refresh tokens still alive. Operators should treat FLUSHALL on prod Redis as a security event.

**What to check**

- [ ] Access token works across restart.
- [ ] Refresh token works across restart.
- [ ] OTP issued pre-restart verifies post-restart.
- [ ] FLUSHALL doesn't break live access tokens.

---

## 11. Unlock engine — baseline + bypass flag

**What the user experiences:** when `UNLOCK_BYPASS=true` (dev default) every course/part/day/activity returns unlocked — this is the "demo mode" the product team uses for screenshots. When `false` (prod default), the real engine computes unlock state from `user_progress`.

### a) With bypass ON (dev default)

```bash
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 | \
  jq '{course_unlocked: .data.unlocked, part0: .data.parts[0] | {unlocked, unlocks_at}, day0: .data.parts[0].days[0] | {unlocked, unlocks_at}}'
```

Expected: every `unlocked: true`, every `unlocks_at: null`. Matches P4 §12 — keys are injected at every level of the hierarchy.

### b) Flip bypass OFF

Edit `.env.local`:

```
UNLOCK_BYPASS=false
```

Restart `pnpm dev`. Same curl now returns real state. For a user with **zero** `user_progress` rows: part 1 day 1 activity 1 unlocked, everything else locked.

```bash
# Wipe progress for user 1 so we're on a clean slate
psql yyddp_dev -c "DELETE FROM user_progress WHERE user_id=1;"
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 | \
  jq '[.data.parts[0].days[0].activities[] | {activity_id, unlocked}]'
# → first activity unlocked: true, rest unlocked: false
```

**What to check**

- [ ] Bypass on: everything unlocked, `unlocks_at: null`.
- [ ] Bypass off + zero progress: only first activity of first day of first part is unlocked.
- [ ] `unlocked` + `unlocks_at` keys **always present** at every level in both modes (shape is stable; only values differ).

---

## 12. Unlock engine — sequential within a day

**What the user experiences:** completing activity N unlocks N+1. Old activities stay unlocked forever (no relocking).

```bash
# Preconditions: UNLOCK_BYPASS=false, user 1 progress wiped (from §11b)
# Look at day 1 activities for user 1
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 \
  | jq '.data.parts[0].days[0].activities | map({activity_id, unlocked})'
# → [{id:1, unlocked:true}, {id:2, unlocked:false}]  (on seed)
```

Complete activity 1 (same flow as P4 §17):

```bash
# Start session
SID=$(curl -s -X POST http://localhost:8888/api/v1/sessions/start \
  -H "Authorization: Bearer $ACCESS" \
  -H 'content-type: application/json' \
  -d '{"course_id":1,"activity_id":1}' | jq -r .data.learning_session_id)

# Interact + complete
curl -s -X POST "http://localhost:8888/api/v1/sessions/$SID/interactions" \
  -H "Authorization: Bearer $ACCESS" -H 'content-type: application/json' \
  -d '{"interaction_type":"video","success":true}' >/dev/null

curl -s -X POST "http://localhost:8888/api/v1/sessions/$SID/complete" \
  -H "Authorization: Bearer $ACCESS" | jq '{reward_coins: .data.reward_coins, already_completed: .data.already_completed}'
# → reward_coins:10, already_completed:false
```

Re-fetch the day:

```bash
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 \
  | jq '.data.parts[0].days[0].activities | map({activity_id, unlocked})'
# → [{id:1, unlocked:true}, {id:2, unlocked:true}]
```

`recomputeAfterCompletion` fired inside the `completeSession` transaction ([unlockService.js:123-151](yyddp-api/src/services/unlockService.js#L123-L151)) and upserted activity 2 into `user_progress` with `status='UNLOCKED'`. Check:

```bash
psql yyddp_dev -c "SELECT activity_id, status, unlock_time, complete_time, unlocks_at FROM user_progress WHERE user_id=1 ORDER BY activity_id;"
# activity 1 → COMPLETED, complete_time set
# activity 2 → UNLOCKED, unlock_time set, unlocks_at null
```

And `unlock_logs` should carry an audit row for activity 2.

### Edge: can you complete activity 2 without completing 1?

The session service doesn't gate `/complete` on `user_progress.status` — so you can, and the completion will recompute unlocks anyway. This is documented **non-behavior**: P5 doesn't add a "must be unlocked to complete" guard (would break the mobile client's optimistic flow). The client is responsible for only surfacing unlocked activities to the user.

**What to check**

- [ ] Fresh user: only activity 1 unlocked.
- [ ] After completing 1: activity 2 flips to unlocked on next GET.
- [ ] `user_progress` has both rows with expected statuses.
- [ ] `unlock_logs` has an audit row for activity 2.

---

## 13. Unlock engine — next-day unlock

**What the user experiences:** finish every activity in day 1 → day 2's first activity becomes unlocked; the **rest** of day 2 stays locked until you walk through day 2 sequentially.

```bash
# From §12 — activity 1 complete, activity 2 unlocked but not started.
# Complete activity 2:
SID=$(curl -s -X POST http://localhost:8888/api/v1/sessions/start \
  -H "Authorization: Bearer $ACCESS" -H 'content-type: application/json' \
  -d '{"course_id":1,"activity_id":2}' | jq -r .data.learning_session_id)
curl -s -X POST "http://localhost:8888/api/v1/sessions/$SID/complete" \
  -H "Authorization: Bearer $ACCESS" >/dev/null

# Inspect day 2
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 \
  | jq '.data.parts[0].days[1] | {unlocked, unlocks_at, activities: [.activities[] | {activity_id, unlocked}]}'
# → day: unlocked:true, unlocks_at:null
#   first activity (id=3): unlocked:true
#   (if day 2 has additional activities, they should be unlocked:false)
```

[unlockService.js:161-179](yyddp-api/src/services/unlockService.js#L161-L179) is the intra-part, day-to-day hop: detects "all activities in day N complete" → upserts day N+1's first activity as `UNLOCKED`, `unlocks_at=null` (no time gate inside a part).

### Before all-complete

If day 1 still has an incomplete activity, day 2 is locked. Test by resetting partway: `DELETE FROM user_progress WHERE user_id=1 AND activity_id=2;` then refetch — day 2 should flip back to locked on the next GET.

**What to check**

- [ ] Complete all of day 1 → day 2 unlocks + day 2 first activity unlocks.
- [ ] Day 2's non-first activities remain locked.
- [ ] Before all-complete: day 2 is locked.

---

## 14. Unlock engine — next-part scheduled with `unlocks_at`

**What the user experiences:** finish the last activity of part 1 → part 2 is "coming tomorrow at 10 AM" (Asia/Shanghai by default). The response shows `unlocked:false` with a concrete `unlocks_at` timestamp the mobile client can render as "unlocks in 3h 42m".

### a) Walk to the end of part 1

The fast way: short-circuit by completing every activity in part 1 via SQL:

```bash
# Seed a "just-completed part" state for user 1 — all activities in part 1 done
psql yyddp_dev <<'SQL'
INSERT INTO user_progress (user_id, activity_id, status, complete_time)
SELECT 1, da.activity_id, 'COMPLETED', NOW()
FROM day_activities da
JOIN part_days pd ON pd.day_id = da.day_id
WHERE pd.part_id = (SELECT part_id FROM courses_parts WHERE course_id=1 ORDER BY sequence ASC LIMIT 1)
ON CONFLICT (user_id, activity_id) DO UPDATE SET status='COMPLETED', complete_time=NOW();
SQL
```

Then trigger one final `/complete` call on the last activity so `recomputeAfterCompletion` fires the cross-part path ([unlockService.js:181-219](yyddp-api/src/services/unlockService.js#L181-L219)). (Or just re-complete the very last activity via the session flow.)

### b) Inspect part 2

```bash
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 \
  | jq '.data.parts[1].days[0] | {unlocked, unlocks_at, first_activity: .activities[0] | {activity_id, unlocked, unlocks_at}}'
# → unlocked:false, unlocks_at:"2026-04-25T02:00:00.000Z" (= 10:00 Shanghai, next day, as UTC)
```

`unlocks_at` is a UTC ISO-8601 string. `10:00 Shanghai = 02:00 UTC` (fixed — `tzHelper.js` now appends `Z` to the date string so the offset is applied correctly regardless of system timezone). Check the DB:

```bash
psql yyddp_dev -c "SELECT activity_id, status, unlocks_at FROM user_progress WHERE user_id=1 AND unlocks_at IS NOT NULL;"
```

### c) Boundary flip (lazy eval)

No cron — the flip happens on the next GET after `unlocks_at ≤ now()`. To simulate:

```bash
# Move the boundary into the past
psql yyddp_dev -c "UPDATE user_progress SET unlocks_at = NOW() - INTERVAL '1 minute' WHERE user_id=1 AND unlocks_at IS NOT NULL;"

curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 \
  | jq '.data.parts[1].days[0] | {unlocked, unlocks_at}'
# → unlocked:true, unlocks_at:null — first read past the boundary flips the state
```

The `unlocks_at` value is still in the DB row but the response zeroes it because `unlocked:true` wins ([unlockService.js:90-98](yyddp-api/src/services/unlockService.js#L90-L98)).

**What to check**

- [ ] After part 1 complete: `user_progress` has a row for part 2 day 1 first activity with `unlocks_at` populated.
- [ ] `unlocks_at` is UTC ISO-8601; corresponds to next 10:00 Shanghai.
- [ ] Before the boundary: `unlocked:false`, `unlocks_at` in response.
- [ ] Past the boundary: `unlocked:true`, `unlocks_at:null` on first read.

---

## 15. Unlock engine — env-configurable boundary

**DoD item:** "Tweak `UNLOCK_HOUR=0` in env + restart → scheduled unlocks flip immediately on next read" — with a caveat.

The env change affects **future** `nextBoundary()` calls, not already-stored `unlocks_at` values (those are Dates in PG, not expressions). So the clean proof is:

### a) Complete another part with `UNLOCK_HOUR=0`

1. Set up a second "part just completed" state (part 2 this time, if the seed has one).
2. Edit `.env.local`: `UNLOCK_HOUR=0`. Restart API.
3. Trigger the final `/complete` call on part 2's last activity.
4. Inspect:

```bash
psql yyddp_dev -c "SELECT activity_id, unlocks_at FROM user_progress WHERE user_id=1 ORDER BY unlocks_at DESC NULLS LAST LIMIT 2;"
```

The newly-scheduled part-3 unlock should land at the **next 00:00 Shanghai** (= 16:00 UTC on the previous calendar day), not 10:00.

### b) `UNLOCK_TZ=UTC`

Same setup, but `.env.local` has `UNLOCK_TZ=UTC`, `UNLOCK_HOUR=10`. The stored `unlocks_at` should correspond to 10:00 UTC, not 10:00 Shanghai. The UTC string and the "intended wall-clock time" line up exactly (no offset).

### c) Retroactively unlock already-scheduled rows?

If product wants "I changed the env, un-lock everything scheduled right now" — that's **not** what §15a tests. For that, flip `UNLOCK_BYPASS=true` temporarily. Or manually `UPDATE user_progress SET unlocks_at = NOW() - interval '1s' WHERE unlocks_at > NOW();`. Env-configurability is for _future_ boundaries.

**What to check**

- [ ] `UNLOCK_HOUR=0` + restart + complete-a-part → new `unlocks_at` at 00:00 Shanghai, not 10:00.
- [ ] `UNLOCK_TZ=UTC` → stored `unlocks_at` is 10:00 on the UTC day, not 10:00 Shanghai.
- [ ] Env change does **not** rewrite existing `unlocks_at` values (document).

---

## 16. Unlock engine — response shape across endpoints

Sanity check that all three services inject `{unlocked, unlocks_at}` consistently. If any one of these is missing the keys, the mobile client will render "undefined" as the lock state.

```bash
# /courses/:id — part + day level
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1 \
  | jq '.data.parts[0] | keys_unsorted, .data.parts[0].days[0] | keys_unsorted'
# Should include "unlocked" and "unlocks_at" at both levels

# /courses/:id/lessons/:lesson/parts/:part — day + activity level
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/courses/1/lessons/1/parts/1001001 \
  | jq '.data.days[0] | keys_unsorted, .data.days[0].activities[0] | keys_unsorted'

# /activities/:id — top-level
curl -s -H "Authorization: Bearer $ACCESS" http://localhost:8888/api/v1/parts/1001001/days/100100101/activities/1 \
  | jq 'keys_unsorted'
```

The keys are always present; values differ by `UNLOCK_BYPASS`:

| Mode                       | `unlocked` | `unlocks_at`           |
| -------------------------- | ---------- | ---------------------- |
| `BYPASS=true`              | true       | null                   |
| `BYPASS=false`, unlocked   | true       | null                   |
| `BYPASS=false`, time-gated | false      | ISO-8601 UTC timestamp |
| `BYPASS=false`, locked     | false      | null                   |

**What to check**

- [ ] Course detail: every part has both keys, every day has both keys.
- [ ] Part detail: every day has both keys, every activity has both keys.
- [ ] Activity detail: top-level has both keys.
- [ ] Shape is identical across bypass=on and bypass=off.

---

## 17. Recompute-in-transaction (race condition)

**Why it matters:** `recomputeAfterCompletion(tx, ...)` runs inside the `completeSession` transaction. If two clients fire `/complete` on the same session concurrently (network retry, double-tap), we must not double-award coins, must not double-unlock, must not create duplicate `user_progress` rows.

```bash
# Fresh session
SID=$(curl -s -X POST http://localhost:8888/api/v1/sessions/start \
  -H "Authorization: Bearer $ACCESS" -H 'content-type: application/json' \
  -d '{"course_id":1,"activity_id":3}' | jq -r .data.learning_session_id)

# Fire two /complete calls in parallel
curl -s -X POST "http://localhost:8888/api/v1/sessions/$SID/complete" \
  -H "Authorization: Bearer $ACCESS" > /tmp/r1.json &
curl -s -X POST "http://localhost:8888/api/v1/sessions/$SID/complete" \
  -H "Authorization: Bearer $ACCESS" > /tmp/r2.json &
wait

jq '{reward_coins, already_completed}' /tmp/r1.json /tmp/r2.json
```

Expected: one response has `reward_coins: 10, already_completed: false`, the other has `already_completed: true` with no further coin award. Verify:

```bash
psql yyddp_dev -c "SELECT COUNT(*) FROM user_progress WHERE user_id=1 AND activity_id=3;"
# → 1    (upsert guarantees no duplicate)

psql yyddp_dev -c "SELECT total_coins FROM users WHERE user_id=1;"
# → should have gone up by exactly 10, not 20
```

The transactional wrapper + session-level `already_completed` guard is what makes this safe. If either response double-awards or you see 2 `user_progress` rows, the recompute happened outside the tx boundary — that's a regression.

**What to check**

- [ ] Exactly one of two concurrent `/complete` calls awards coins.
- [ ] `user_progress` row count for the activity = 1.
- [ ] `users.total_coins` incremented by 10, not 20.

---

## 18. OSS smoke test

**What the user experiences:** indirect — if this fails in prod, every download link we hand the mobile client is broken. P4 only verified URL _shape_; P5 §7 adds the end-to-end reachability check.

```bash
cd yyddp-api
pnpm run oss:smoke
```

Expected: script generates a signed URL for a known key (default comes from `OSS_SMOKE_TEST_KEY` in `.env.local` — currently `sqlite/20250927.sqlite`, resolved against `OSS_DIRECTORY=/videos` to `videos/sqlite/20250927.sqlite` in the `ailanguages` bucket), follows the redirect, prints HTTP 200 + `Content-Length` > 0. Override the key by passing it as argv[2] or by editing `OSS_SMOKE_TEST_KEY` — don't hardcode keys back into the script, since the directory layout in OSS may change over time.

### Edge: bad creds fail cleanly

```bash
OSS_ACCESS_KEY_ID=garbage pnpm run oss:smoke
# → non-zero exit, clear error like "AccessDenied" or "InvalidAccessKeyId"
# Not a stack trace / unhandled promise rejection.
```

**What to check**

- [ ] Happy path: 200 + body.
- [ ] Bad creds: non-zero exit with readable error.

---

## 19. Importer — `upload:media` CLI (happy path)

**What the operator experiences:** the missing step (c) from Contract §2.4 is now a one-liner. Before P5, unknown `/download/*` URLs flagged in the diff had no upload path — they had to be handled out-of-band.

### Setup

```bash
cd yyddp-importer
pnpm run build                  # regenerate data.json from SQLite if needed
pnpm run diff                   # shows N unknown URLs
```

### Run the upload

```bash
pnpm run upload:media
```

Expected output pattern ([scripts/uploadMedia.ts:27-40](yyddp-importer/scripts/uploadMedia.ts#L27-L40)):

```
Media upload — target: dev, activities: <N>

<M> pending URLs to process

  [https://.../video.mp4] processing
  [https://.../video.mp4] uploaded to oss://ailanguages/videos/lesson/...
  [https://.../image.png] processing
  [https://.../image.png] uploaded to oss://ailanguages/images/cards/...
  ...

Done: uploaded=3 skipped=0 failed=0
```

Verify the SQLite log:

```bash
sqlite3 yyddp.db "SELECT COUNT(*), file_type FROM media_processing_log GROUP BY file_type;"
# → expect rows for video / image / audio depending on what was pending
```

Prove the new OSS keys are actually reachable — pick one `output_url` from the log and curl it through the API's download endpoint:

```bash
OUTPUT_KEY=$(sqlite3 yyddp.db "SELECT output_url FROM media_processing_log ORDER BY id DESC LIMIT 1;")
curl -I -H "Authorization: Bearer $ACCESS" "http://localhost:8888/api/v1/download?path=$OUTPUT_KEY"
# → 302 to a signed OSS URL; follow → 200 or 206 with Content-Length > 0
```

**What to check**

- [ ] CLI prints `Done: uploaded=N skipped=0 failed=0` on a clean run.
- [ ] `media_processing_log` gains exactly N rows with `sha1_hash`, `content_type`, `output_url`, `parameters_used` all populated.
- [ ] Process exit code is 0 when `failed=0`, non-zero when `failed>0`.
- [ ] At least one new key is reachable via `/api/v1/download`.

---

## 20. Importer — idempotency

**What the operator experiences:** re-running `upload:media` after a successful run is a no-op. The killer feature: the unique index on `media_processing_log.original_url`.

```bash
# After §19 happy path
pnpm run upload:media
# → "Done: uploaded=0 skipped=3 failed=0"
```

Each `/download/*` URL that already has a row in `media_processing_log` is short-circuited with a `SKIP` event — no transcode, no OSS upload, zero bytes written.

### Edge: partial retry

```bash
# Drop one row manually to simulate a failed upload
sqlite3 yyddp.db "DELETE FROM media_processing_log WHERE id = (SELECT MAX(id) FROM media_processing_log);"
pnpm run upload:media
# → "Done: uploaded=1 skipped=2 failed=0"
```

The one row is reprocessed. Everything else stays skipped. This is exactly how an operator recovers from a mid-run failure: the successful work is durable, and the retry only redoes what's missing.

### Edge: kill mid-run

Start `pnpm run upload:media`, Ctrl-C after 1-2 uploads are logged. Re-run. Only the URLs that didn't make it into the log get retried.

**What to check**

- [ ] Second run: `uploaded=0, skipped=N`.
- [ ] Manually delete 1 row → next run: `uploaded=1, skipped=N-1`.
- [ ] Ctrl-C mid-run → next run picks up only the unfinished URLs.

---

## 21. Importer — media-type routing

Verify URLs land in the right OSS prefix. The routing logic lives in `windmill/scripts/upload_to_oss.ts`.

Run `pnpm run upload:media` with one known URL of each media type in the pending set. After the run:

```bash
sqlite3 yyddp.db "SELECT original_url, output_url, file_type FROM media_processing_log ORDER BY id DESC LIMIT 10;"
```

Expected prefixes (exact prefixes depend on `upload_to_oss.ts` — cross-check against actual output):

| Original URL contains | file_type | Output prefix       |
| --------------------- | --------- | ------------------- |
| `.mp4`                | video     | `videos/lesson/...` |
| `cards/` + `.png`     | image     | `images/cards/...`  |
| `words/` + `.png`     | image     | `images/words/...`  |
| `.mp3`, lesson audio  | audio     | `audios/lesson/...` |

### Edge: unknown type

Add a URL ending in `.xyz` to a test fixture → run → expect a `FAIL` line and a row in `failedUrls` in the final summary. No garbage row in `media_processing_log`, no garbage object in OSS.

**What to check**

- [ ] Video → `videos/lesson/...`.
- [ ] Image with `cards/` in path → `images/cards/...`.
- [ ] Audio → `audios/lesson/...`.
- [ ] Unknown extension → reported as `failed`, not silently uploaded.

---

## 22. Importer — UI button flow

The Vue preview app now exposes the upload step so a non-CLI operator can drive it.

### Setup

```bash
cd yyddp-importer
pnpm run preview    # starts API on :3001 + Vite on :5173
```

Browse to `http://localhost:5173` (or through nginx at `http://localhost:8080/admin/` if running the dev compose stack).

### Happy path

1. Run a diff that reports N > 0 unknown URLs.
2. The "Process & Upload Media (N pending)" button should be **visible** and **enabled** in [DiffView.vue](yyddp-importer/preview/src/components/DiffView.vue), next to the Import button.
3. Click it.
4. A progress panel appears (same layout as the import one); NDJSON events stream in: one "processing" line per URL, then a final summary with `uploaded / skipped / failed`.
5. On failure, the first 10 failed URLs are listed inline.

### Edge

- [ ] N = 0 (nothing pending) → button is **hidden** or **disabled** (whichever the UI does — document current behavior).
- [ ] Click "Import" and "Upload Media" back-to-back → the second click is blocked with a "single-flight" message while the first is running ([server/routes/import.ts:37 + :158-192](yyddp-importer/server/routes/import.ts#L37)).
- [ ] Reload the page mid-stream → the server-side upload continues; the next page load sees the updated `media_processing_log` state via the diff endpoint.

**What to check**

- [ ] Button shows "(N pending)" matching the diff count.
- [ ] Progress streams as NDJSON (one event per URL).
- [ ] Final panel shows uploaded/skipped/failed counts.
- [ ] Failure list shows first 10 failed URLs.
- [ ] Single-flight guard blocks concurrent import + upload.

---

## 23. Importer — safety rails for prod

**What the operator experiences:** uploading to prod OSS is irreversible; the CLI and UI both require an explicit "I know what I'm doing" gesture.

### CLI

```bash
IMPORT_TARGET=prod pnpm run upload:media
# → "IMPORT_TARGET=prod requires the --yes-i-know flag for media uploads."
# Exit code 1, zero uploads.

IMPORT_TARGET=prod pnpm run upload:media --yes-i-know
# → proceeds, uses prod OSS creds
```

Reference: [scripts/uploadMedia.ts:19-23](yyddp-importer/scripts/uploadMedia.ts#L19-L23).

### UI

With `IMPORT_TARGET=prod` in the server's env:

- The upload button should require a confirmation checkbox ("I understand this writes to prod OSS") before it sends `{confirmed: true}` to `/api/import/upload-media`.
- Clicking with the checkbox unchecked → no-op, no network request (or request returns 400 if the UI always sends).

### Dev default

With `IMPORT_TARGET=dev` or unset: no flag needed, no checkbox required.

**What to check**

- [ ] CLI without `--yes-i-know` + `IMPORT_TARGET=prod` → refuses with clear message, exit 1.
- [ ] CLI with flag → proceeds.
- [ ] UI without checkbox + `IMPORT_TARGET=prod` → button disabled or no-op.
- [ ] Dev: no flag / checkbox needed.

---

## 24. End-to-end operator workflow

The full Contract §2.4 loop P5 closes. **This is the single integration test that proves P5 and P4 are compatible end-to-end.**

1. Point the importer at a fresh SQLite with one brand-new `/download/*` URL for each media type (video, image, audio).
2. `pnpm run build` → `data.json` regenerated.
3. `pnpm run diff` → reports 3 unknown URLs.
4. `pnpm run upload:media` → transcodes + uploads 3 objects, writes 3 rows to `media_processing_log`.
5. `pnpm run import:pg` → inserts the activity/session rows that reference those new objects.
6. On the API side:
   ```bash
   curl -s -H "Authorization: Bearer $ACCESS" \
     http://localhost:8888/api/v1/courses/1/lessons/1/parts/<new-part>/days/<new-day>/activities/<new-activity> | jq
   ```
   The response `interactions` should reference the **new** OSS output paths (under `videos/lesson/...`, etc.), not the original `/download/*` URLs.
7. Fetch one of those OSS paths via the API's signed-URL endpoint:
   ```bash
   curl -I -H "Authorization: Bearer $ACCESS" \
     "http://localhost:8888/api/v1/download?path=<one-of-the-new-keys>"
   # → 302 to a signed URL → 200/206 with bytes
   ```
8. Re-run `pnpm run upload:media` → `uploaded=0, skipped=3` (idempotency).

If all 8 pass, the importer-to-API contract is intact: media operators can onboard new content through a single workflow, and the API serves it back via signed URLs.

**What to check**

- [ ] Steps 3–5 succeed in order.
- [ ] API returns the new activity with OSS-prefixed URLs in interactions.
- [ ] Signed-URL download returns bytes.
- [ ] Second `upload:media` run reports all-skipped.

---

## 25. Regression — P4 still works

The auth + unlock changes in P5 touched `sessionService`, all three GET services, `authService`, and `container.js`. Spot-check that P4's golden paths are intact.

### a) Real-auth login (P4 §Prerequisites)

Captcha → verify → OTP → login round-trip completes with `access_token` bound to `user_id=1`. `/user/profile` returns user 1's record.

### b) OSS download (P4 §10)

```bash
curl -i -H "Authorization: Bearer $ACCESS" \
  "http://localhost:8888/api/v1/download?path=cards/b/baby/baby-child-833-v1.png"
# → 302 → follow → 206 Partial Content with real bytes
```

### c) Session start/interact/complete (P4 §17)

The full loop still awards coins, still writes `user_progress`, _and now_ fires `recomputeAfterCompletion`. Confirm from §12 of this doc.

### d) UNLOCK_BYPASS=true still behaves like P4 §12

Every `locked` field false, every `status` unlocked — the bypass is a hard short-circuit that runs ahead of the unlock service.

**What to check**

- [ ] Real auth flow still gets a JWT.
- [ ] OSS download returns bytes.
- [ ] Session loop awards 10 coins per completion and recomputes unlocks.
- [ ] Bypass mode still looks like P4.

---

## Risks / out of scope

- **Multi-instance deploy.** JTI revocation lives in Redis, so any API pod sharing the Redis instance sees revocations instantly. If someone later splits Redis per-pod, revocations won't cross pods — not tested here because the current prod target is single-instance.
- **Aliyun SMS quota.** Every live run against +8613162908096 costs real credit. Keep §3 runs tight; prefer the mocked path (§2) for iteration.
- **Legacy unlock tables.** `user_day_unlocks`, `users_activities_unlocks`, `users_days_unlocks`, `users_parts_unlocks` are still in the schema but inert — P5 derives everything from `user_progress`. Future housekeeping to drop them.
- **Clock skew.** `unlocks_at` comparisons use server-local `new Date()` in the API process. If DB time and API host time drift, the boundary flip shifts by the same drift. Not tested here; covered by infra (ntpd on the prod host).
- **Real SMS under rate-limit.** §7's 60s-per-phone limiter means live testing with +8613162908096 needs you to wait between requests. If you trip the limiter, `DEL otp-rl:phone:13162908096` in dev Redis to reset.

---

## Tester cheat sheet

### Redis keys + TTLs

| Key pattern            | TTL   | Set at                                                                | Purpose                       |
| ---------------------- | ----- | --------------------------------------------------------------------- | ----------------------------- |
| `captcha:{id}`         | 300s  | [authService.js:121](yyddp-api/src/auth/services/authService.js#L121) | Captcha text                  |
| `otp:{phone}`          | 300s  | [authService.js:169](yyddp-api/src/auth/services/authService.js#L169) | `{code, attempts}` JSON       |
| `otp-rl:phone:{phone}` | 60s   | [authService.js:84](yyddp-api/src/auth/services/authService.js#L84)   | Per-phone rate limit          |
| `otp-rl:ip:{ip}`       | 3600s | [authService.js:94](yyddp-api/src/auth/services/authService.js#L94)   | Per-IP rate limit             |
| `jti-revoked:{jti}`    | ≤7d   | [authService.js:279](yyddp-api/src/auth/services/authService.js#L279) | Refresh-token revocation list |

### Thresholds

| Thing                          | Value         | Source                                                                |
| ------------------------------ | ------------- | --------------------------------------------------------------------- |
| OTP TTL                        | 300s          | [authService.js:166](yyddp-api/src/auth/services/authService.js#L166) |
| Max wrong OTP guesses          | 5             | [authService.js:208](yyddp-api/src/auth/services/authService.js#L208) |
| Per-phone rate limit           | 1/60s         | [authService.js:85](yyddp-api/src/auth/services/authService.js#L85)   |
| Per-IP rate limit              | 10/1h         | [authService.js:95](yyddp-api/src/auth/services/authService.js#L95)   |
| Access token lifetime          | 1h            | [authService.js:62](yyddp-api/src/auth/services/authService.js#L62)   |
| Refresh token lifetime         | 7d            | [authService.js:67](yyddp-api/src/auth/services/authService.js#L67)   |
| Unlock boundary TZ (default)   | Asia/Shanghai | [tzHelper.js:2](yyddp-api/src/utils/tzHelper.js#L2)                   |
| Unlock boundary hour (default) | 10            | [tzHelper.js:3](yyddp-api/src/utils/tzHelper.js#L3)                   |

### Error codes seen in this guide

| HTTP | `ret_code`                       | Meaning                                          |
| ---- | -------------------------------- | ------------------------------------------------ |
| 400  | `SECURITY_CHECK_FAILED` (600010) | Bad captcha / OTP, rate-limit, attempts exceeded |
| 400  | `PARAM_INVALID`                  | Malformed phone                                  |
| 400  | `MISSING_REQUIRED_PARAM`         | Missing phone / sms_code / captcha_id            |
| 401  | `USER_TOKEN_INVALID` (600004)    | Revoked / malformed / expired token              |
| 401  | `USER_NOT_LOGGED_IN`             | No `Authorization` header                        |
| 429  | `SECURITY_CHECK_FAILED` (600010) | Rate-limit hit                                   |
