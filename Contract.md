# APPENDIX 1 — 英语对对碰 (YYDDP) Backend API

**Functional Specification** Document Type: Contractor Scope & Deliverables Reference — Version 1.0

## 1. Product Context

英语对对碰 (YYDDP) is a web/app English-learning product for children aged roughly 3–8, built around a staged "watch → match → immerse" learning loop with daily unlocks, gold-coin rewards, and a spiral review structure. This Appendix covers only the backend API (repository: yyddp-api).

An existing codebase already implements the HTTP server, routing, Swagger, mock services, partial authentication, partial real-DB reads, and OSS signed downloads. The goal of this Project is to take the backend from its current partial state to a launch-ready production service, fully wired to PostgreSQL, SMS, and OSS, with all primary endpoints backed by the real database.

## 2. Scope of Work

### 2.1 Database & ORM Alignment

- Align Prisma schema with the production PostgreSQL DDL and keep the two in sync via db pull / migrate.
- Ensure all tables required by the endpoints in this Appendix are present, indexed, and usable by the ORM.

### 2.2 Authentication (production)

- Captcha endpoint hardened for production use.
- SMS OTP endpoint wired to a real SMS provider with rate limiting.
- Phone + OTP login producing a DB-backed user record and session/JWT pair.
- Token refresh and logout (token invalidation).
- Production JWT signing secrets and basic rotation policy.

### 2.3 Course Content APIs (user-facing, DB-backed)

- GET /courses — user course list from PostgreSQL.
- GET /courses/{course_id} and course switching.
- GET /courses/{course_id}/playlist and today's playlist/bootstrap endpoint.
- Nested part and activity endpoints returning the full activity payload (interactions, media references, reward metadata) from PostgreSQL.
- Remove the transitional /api/v1/real/_ duplicate surface; a single /api/v1/_ is the production contract.

### 2.4 Learning / Progress / Gold Write APIs

- Session start and interaction submit endpoints for activities (matching / 对对碰, audiopickcard, listening test, video activities, etc.).
- Activity completion endpoint, idempotent per learning session (no double-pay).
- Gold-coin accrual at both interaction and activity levels per product rules.
- User progress read endpoint (per course).

### 2.5 Unlock Engine

- Sequential unlock within a Day; next Day opens after previous Day's requirements met; next Part's Day 1 opens after the previous Part is fully cleared.
- End-of-Part / end-of-arc scheduling uses next-calendar-day 10:00 local time; lazy-evaluated on user-driven traffic (no cron).
- Unlock decisions are persisted (unlock rows + audit log) and surfaced as flags on course/part/day/activity payloads.

### 2.6 Profile, Assets, Ops

- GET /user/profile and /user/stats backed by PostgreSQL on the primary path.
- OSS signed-download endpoint verified against production bucket and credentials.
- Basic request logging and health endpoint retained.
- Swagger regenerated and kept current with the final routes.

## 3. Out of Scope (Appendix 1)

- New product features beyond the endpoints and rules listed above.
- Client/front-end changes.
- Advanced observability/APM, multi-region deployment, and load testing beyond a basic sanity pass.
- Content authoring tools (covered by Appendix 2 where applicable).

## 4. Week 1 Milestone (Appendix 1)

- SMS-based phone login works end-to-end against the production SMS provider.
- The app is navigable: a logged-in user can see their course list, open a course, and browse parts/days/activities served from PostgreSQL.
- Unlock enforcement, interaction submits, and gold-coin accrual are NOT required at this milestone. All lessons are unlocked by default.

## 5. Final Delivery (Appendix 1)

- All endpoints in Section 2 implemented, DB-backed, and callable under a single /api/v1/\* surface.
- The server should have no STATE, all states should be in either Redis or production PostgreSQL database, so the server can be scaled to a multiple distributed version.
- Unlock rules and gold-coin rules enforced server-side per Section 2.4–2.5.
- Source code, updated Swagger, environment variable reference, and deployment notes handed over.
- Backend deployed to Party A's production environment and reachable by the client.

---

# APPENDIX 2 — YYDDP Course Importer / Deployer

**Functional Specification** Document Type: Contractor Scope & Deliverables Reference — Version 1.0

## 1. Product Context

The YYDDP course content is authored in a SQLite-based content-management database. Before it can power the backend API (Appendix 1), this content must be transformed, validated, and loaded into the production PostgreSQL database and OSS bucket. This Appendix covers the importer/deployer tool (repository: yyddp-importer), which serves as the one-way pipeline from authoring SQLite to production systems.

An existing codebase already implements significant portions of the converter and importer logic. The goal of this Project is to take the importer from its current partial state to a working deployer that a non-developer operator can run to push course content into production.

All modifications of content in production PostgreSQL database should be "soft-deleted" instead really deleted.

## 2. Scope of Work

### 2.1 SQLite → Intermediate Conversion

- Read the authoring SQLite database and reconstruct the Course → Lesson → Part → Day → Activity → Interaction hierarchy, including the derived Day layer that is not a physical table in SQLite.
- Correctly assemble 对对碰 (matching) content from its constituent tables (segments, combo segments, wordsets, matching games).
- Emit a stable intermediate representation (JSON or equivalent) consumable by the loader.
- A web-based admin UI for the importer that displays the differences from current sqlite to product pg, is required, and there is a button that allows the start of the importation.

### 2.2 Media Processing

- Identify media assets referenced by the course content.
- Perform required media processing steps already present in the repo (e.g. transcoding, image optimization, watermarking) as configured, and upload resulting files to the production OSS bucket with the correct paths.

### 2.3 PostgreSQL Loader

- Load the intermediate representation into PostgreSQL tables matching the backend schema used by Appendix 1.
- Schema conversion/alignment from SQLite types to PostgreSQL types.
- Incremental/idempotent behavior: re-running the importer with unchanged content must not corrupt existing rows or duplicate data.
- Basic sanity checks after load (row counts, referential integrity spot checks, required fields present).

### 2.4 Operator Workflow

- A documented command-line or script-driven workflow that an operator can follow to: (a) point at a SQLite source, (b) run conversion, (c) run media upload, (d) run DB load into the production environment, (e) see a pass/fail report.
- Clear, actionable error messages when a step fails, and a way to retry from the failed step without restarting the whole pipeline.

## 3. Out of Scope (Appendix 2)

- Real-time two-way sync between SQLite and PostgreSQL.
- Content authoring features inside SQLite.

## 4. Week 1 Milestone (Appendix 2)

- No mandatory deliverable for Appendix 2 at the Week 1 checkpoint; priority for Week 1 is the user-visible backend milestone in Appendix 1.
- A recommended (non-binding) goal is that the existing importer can convert a sample SQLite file end-to-end to the intermediate representation without errors.

## 5. Final Delivery (Appendix 2)

- Operator can run the importer against the live authoring SQLite and successfully publish course content to dev PostgreSQL, then to production PostgreSQL and OSS.
- Appendix 1 backend, running against the same production PostgreSQL, serves the imported content correctly to the client.
- Source code, operator runbook, and environment variable reference handed over.

## 6. Joint Acceptance (Appendices 1 & 2)

- Final Delivery is considered complete only when Appendix 1 and Appendix 2 together enable the end-to-end flow: content is imported into production, a real user logs in via SMS on the client, and the user can consume unlocked content with working progress tracking and gold-coin accrual.
