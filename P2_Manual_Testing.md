# P2 — Manual Testing: Submit APIs + Gold Service

**Server:** `http://localhost:8888` (or wherever yyddp-api is running)
**Swagger:** `http://localhost:8888/api/v1/docs/` (auto-generated from JSDoc in `src/api/v1/**/*.js`, configured in `src/config/swagger.js`)
**Auth:** All endpoints require `Authorization: Bearer <JWT>` header

---

## How these endpoints map to the kid using the app

The learning flow for a child using YYDDP goes like this:

1. **Kid opens a lesson and taps an activity** (e.g. a video, a matching game, an audio quiz). The mobile app calls **`/sessions/start`** to tell the server "this kid just started this activity."

2. **Kid interacts with the content** — watches a video, picks a card, matches words. Each individual action (watched the whole video, picked the right card, etc.) is sent to the server via **`/sessions/{id}/interactions`**. The server records the details and instantly awards a coin if the kid got it right.

3. **Kid finishes the activity** (completes all interactions). The mobile app calls **`/sessions/{id}/complete`** to say "done." The server awards the activity completion bonus (default 10 coins), marks the activity as completed in the kid's progress, and returns their new coin total. This call is safe to retry — if the network glitches and the app sends it twice, the kid doesn't get double coins.

4. **Parent or kid checks overall progress.** The app calls **`/courses/{id}/progress`** to show a tree of parts > days > activities with each one marked LOCKED / UNLOCKED / COMPLETED and how many coins they've earned.

---

## Get a JWT token

If you have a user, login via `/auth/phone-login`. Otherwise, generate a dev token:

```bash
cd yyddp-api
node -e "
const jwt = require('jsonwebtoken');
const token = jwt.sign(
  { id: 1, user_id: 1, phone: '18601113318', type: 'access' },
  process.env.JWT_SECRET || 'dev-only-jwt-secret-change-me',
  { expiresIn: '1h' }
);
console.log(token);
"
```

Set it as a variable for the tests below:

```bash
TOKEN="<paste token here>"
```

---

## Test 1: Start a learning session

> **User scenario:** The kid taps on an activity card (say, a "matching game" card inside Day 1 of Part 1). The app opens the activity screen and in the background fires this request to create a server-side session that will track everything the kid does during this activity.

```bash
curl -X POST http://localhost:8888/api/v1/sessions/start \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"activity_id": 2, "day_id": 100100101}'
```

**Expected:** `ret_code: 0`, response has `learning_session_id` (integer). Save this ID for the next tests.

```json
{
  "ret_code": 0,
  "data": {
    "learning_session_id": 1,
    "activity_id": 2,
    "day_id": 100100101
  }
}
```

---

## Test 2: Submit a video interaction

> **User scenario:** The activity contains a short English video. The kid watches 120 out of 150 seconds (85%). When the video ends (or the kid taps "next"), the app sends the watch stats to the server. Since the kid watched more than 80%, the server considers it "correct" and awards 1 gold coin.

Replace `SESSION_ID` with the value from Test 1.

```bash
curl -X POST http://localhost:8888/api/v1/sessions/SESSION_ID/interactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "interaction_id": 1000000003,
    "video_watched_seconds": 120,
    "video_total_seconds": 150,
    "watched_percentage": 85,
    "paused_count": 1,
    "replay_count": 0
  }'
```

**Expected:** `coins_awarded: 1` (because watched >= 80%).

```json
{
  "ret_code": 0,
  "data": {
    "recorded": true,
    "interaction_id": 1000000003,
    "interaction_type": "video",
    "is_correct": true,
    "coins_awarded": 1
  }
}
```

---

## Test 3: Submit an audiopickcard interaction

> **User scenario:** The app plays an English audio clip ("apple") and shows 4 cards with different words. The kid taps a card. The app sends which card was selected, which was correct, and whether the kid got it right. If correct, 1 gold coin is awarded instantly — the kid sees a coin animation on screen.

Find an audiopickcard interaction ID from the database, or use this pattern:

```bash
curl -X POST http://localhost:8888/api/v1/sessions/SESSION_ID/interactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "interaction_id": INTERACTION_ID,
    "selected_card_id": 5,
    "correct_card_id": 5,
    "is_correct": true,
    "response_time": 2500,
    "audio_replay_count": 1
  }'
```

**Expected:** `coins_awarded: 1` when `is_correct: true`, `coins_awarded: 0` when false.

---

## Test 4: Complete the session

> **User scenario:** The kid has finished all the interactions in the activity (watched the video, answered the quiz, did the matching). The app shows a "great job!" screen and in the background tells the server this activity is done. The server awards the activity completion bonus (10 coins), updates the kid's progress to COMPLETED, and returns the new total coin balance that the app displays.

```bash
curl -X POST http://localhost:8888/api/v1/sessions/SESSION_ID/complete \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"time_spent": 120}'
```

**Expected:** `completed: true`, `reward_coins` = the activity's `reward_coins` value (default 10 if 0 in DB), `total_coins` = user's new coin balance.

```json
{
  "ret_code": 0,
  "data": {
    "completed": true,
    "already_completed": false,
    "learning_session_id": 1,
    "reward_coins": 10,
    "total_coins": 11
  }
}
```

---

## Test 5: Idempotency — complete again

> **User scenario:** The kid's phone had bad signal. The "complete" request was sent but the app didn't get a response, so it retried. The server recognizes this session was already completed and returns success without awarding coins again. This prevents the "free coins" exploit where a kid (or a clever parent) could replay the completion request.

Same request as Test 4, same session ID:

```bash
curl -X POST http://localhost:8888/api/v1/sessions/SESSION_ID/complete \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"time_spent": 120}'
```

**Expected:** `already_completed: true`, same `reward_coins`, no double-pay.

```json
{
  "ret_code": 0,
  "data": {
    "completed": true,
    "already_completed": true,
    "learning_session_id": 1,
    "reward_coins": 10
  }
}
```

---

## Test 6: Get course progress

> **User scenario:** The kid (or parent) opens the course overview screen. The app needs to show which activities are done, which are available, and which are still locked — plus how many total coins the kid has earned in this course. This endpoint returns the entire course tree so the app can render it.

```bash
curl -X GET http://localhost:8888/api/v1/courses/1/progress \
  -H "Authorization: Bearer $TOKEN"
```

**Expected:** Full course structure with per-activity status. Completed activities show `status: "COMPLETED"` with `coins_awarded`.

```json
{
  "ret_code": 0,
  "data": {
    "course_id": 1,
    "course_name": "Course 1",
    "total_activities": 343,
    "completed_activities": 1,
    "total_coins_earned": 10,
    "parts": [
      {
        "part_id": 1001001,
        "part_name": "Part 1",
        "days": [
          {
            "day_id": 100100101,
            "name": "Lesson 1 Part 1 Day 1",
            "activities": [
              { "activity_id": 1, "status": "LOCKED", "coins_awarded": 0 },
              { "activity_id": 2, "status": "COMPLETED", "coins_awarded": 10 }
            ]
          }
        ]
      }
    ]
  }
}
```

---

## Error cases

### No auth token

> The app forgot to include the JWT, or the token expired.

```bash
curl -X POST http://localhost:8888/api/v1/sessions/start \
  -H "Content-Type: application/json" \
  -d '{"activity_id": 2, "day_id": 100100101}'
```

**Expected:** `ret_code: 600001` (用户未登录), HTTP 401.

### Activity not found

> The app sent an activity_id that doesn't exist in the database — maybe the content was removed in a re-import, or the app has stale data.

```bash
curl -X POST http://localhost:8888/api/v1/sessions/start \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"activity_id": 999999, "day_id": 100100101}'
```

**Expected:** `ret_code: 101003` (Activity not found), HTTP 404.

### Submit to completed session

> The app tries to submit more interactions after the session has already been marked complete. This shouldn't happen in normal flow but guards against bugs or replays.

```bash
curl -X POST http://localhost:8888/api/v1/sessions/SESSION_ID/interactions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"interaction_id": 1000000003, "video_watched_seconds": 10}'
```

**Expected:** `ret_code: 400002` (Session already completed), HTTP 400.

---

## Interaction types reference

| Type | Detail fields | Gold awarded when |
|---|---|---|
| `video` | video_watched_seconds, video_total_seconds, watched_percentage, paused_count, replay_count | watched_percentage >= 80 |
| `audiopickcard` | selected_card_id, correct_card_id, is_correct, response_time, audio_replay_count | is_correct = true |
| `audiopickimage` | selected_image_id, correct_image_id, is_correct, response_time, audio_replay_count | is_correct = true |
| `wordcombo` | attempt_count, correct_count, incorrect_count, time_to_complete | correct_count > 0 AND incorrect_count = 0 |
| Other types | is_correct (optional) | is_correct = true |

**Gold defaults:** 1 coin per correct interaction, 10 coins per activity completion (or whatever `activities.reward_coins` says). These can be customized via the `gold_rules` table.

---

## Endpoint summary for mobile team

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/sessions/start` | Start learning session. Body: `{ activity_id, day_id }` |
| POST | `/api/v1/sessions/{id}/interactions` | Submit interaction result. Body: `{ interaction_id, ...type-specific fields }` |
| POST | `/api/v1/sessions/{id}/complete` | Complete session (idempotent). Body: `{ time_spent }` |
| GET | `/api/v1/courses/{id}/progress` | Get user progress for a course |

**Swagger UI:** Open `http://localhost:8888/api/v1/docs/` in a browser. The new endpoints are under the **学习会话** tag. You can try them directly from the Swagger interface by clicking "Authorize" and pasting your JWT token.
