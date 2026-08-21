# Lesson Execution Foundation — Implementation (Phase 1.7B + 1.7B-2)

> Status: **COMPLETE** (1.7B foundation + 1.7B-2 activity-contract closure, 2026-08-20). Lesson start (from
> today's DailyPlanItem), exact LessonRevision pinning, progress resume, learner-safe content read, AND
> objective ActivityAttempt submission + deterministic scoring are all implemented. The Lesson Activity v1
> contract (`lesson-activity-objective/v1`, TD-108/109) was owner-approved. **No schema change / no
> migration** — pinning + attempt evidence (incl. `clientRequestId` uniqueness) already existed.

Code: `backend/src/lesson-execution/**` + shared `src/common/clock.module.ts`.

## 1. Scope
Authenticated learner can start an executable Lesson from TODAY'S CURRENT DailyPlanItem, pin its exact
published revision, resume it (staying pinned), and read learner-safe revision content. NOT: complete the
lesson, submit activities, score, update skill/roadmap, or reward.

## 2. DailyPlan-only entry (TD-106, §3/4)
`POST /api/lesson-executions/daily-plan-items/:dailyPlanItemId/start`. The item must belong to the learner's
TODAY CURRENT DailyPlan (join `dailyPlanItem → dailyPlan {userId, localDate=today, status=CURRENT}`). No
arbitrary `lessonId` start route exists — one-topic-per-day cannot be bypassed (tested §42). Everything (user,
today's local date via Clock+profile timezone, item) is server-derived; no request body.

## 3. One-topic enforcement (§42)
Because the only entry is a today-CURRENT DailyPlanItem, and the DailyPlan is one Topic (Phase 1.7A), a
learner cannot start another Topic's lesson via a stale/foreign/fabricated item (all → 404, tested).

## 4. Live executability (§5/6, reuse)
The lesson's current state is derived by reusing the Phase 1.6B pure `deriveItemState` + `RoadmapRepository`
batch loads — prerequisites are not re-derived. Start requires **AVAILABLE or IN_PROGRESS**; COMPLETED →
`LESSON_ALREADY_COMPLETED` (§7); BLOCKED/UNAVAILABLE → `LESSON_NOT_EXECUTABLE` (§6/43/48). Both MUST_DO and
RECOMMENDED are startable only when their live state is executable (RECOMMENDED ≠ always startable, tested §43/44).

## 5. Progress authority (§12)
`LearnerLessonProgress` = current mutable resume state; `LearnerLessonCompletion` = append-only completed
history. Phase 1.7B writes ONLY progress. No new session/execution entity invented (`LearningSession` untouched).

## 6. Completion boundary (§33/53, CRITICAL)
No `LearnerLessonCompletion` is created — even if a learner starts/uses the lesson. Completion policy + mastery
consequences need their own phase. No `executionReadyForCompletion` flag is invented (required-vs-optional
activity semantics are not accepted). Verified: completion count unchanged (tested §53).

## 7. LessonRevision pinning (TD-107, §8)
`LearnerLessonProgress.lessonRevisionId` (TD-37) pins the lesson's CURRENT published revision (resolved via
`Lesson.publishedRevisionId` with the whole Topic→Module→Level chain PUBLISHED) at FIRST start. It is immutable
for that execution.

## 8. Revision-switch behavior (§8/9/45)
After a new revision is published (archive-first per `ux_lesson_published_revision` C7), the pinned execution
stays on its original revision — resume reads the pinned revision **by id, no status filter**, so it works
even after the pinned revision is archived (tested §45). Never repin. Ordinary re-publication does not force
the execution to the new revision.

## 9. Learner-safe content read (§14/15/16/28)
Start/resume return only: `lesson {title, description, estimatedDurationMin}` + `activities [{id, type,
position}]` — from the PINNED revision. **Never the `Activity.payload` body** (its answer-key layout is an
unaccepted contract, so it cannot be safely stripped), nor authoring/lifecycle/source/`aiMetadata` fields, nor
answer keys (tested: no payload / "SECRET" in responses, §49). No ContentBlock model exists (content lives in
payload). Media delivery/signing is not implemented (deferred; text-metadata execution passes).

## 10. Start idempotency + concurrency (§10/11/46)
`unique(userId, lessonId)` is the authority: an existing progress row resumes with its pinned revision (no
duplicate, no repin, tested §10). Concurrent first starts → one row; the loser catches the P2002 and returns
the winner (tested §46).

## 11. Lesson Activity v1 contract — ACCEPTED (`lesson-activity-objective/v1`, TD-108/109)
Owner-approved objective Activity payload contract — a SEPARATE domain from AssessmentItem's `PLACEMENT_ITEM_V1`
(own parser; option/scoring primitives analogous but not coupled, §1/20/36). camelCase (§4/37):
```jsonc
{ "schemaVersion": "lesson-activity-objective/v1",
  "format": "single_choice" | "multiple_choice" | "true_false",
  "prompt": "...", "options": [{ "id": "...", "text": "..." }],
  "answerKey": { "correctOptionIds": ["..."] } }   // answerKey is SERVER-ONLY (§10)
```
Objective **ActivityTypes** (§2/11): `MINI_QUESTION`, `PRACTICE`, `MASTERY_TEST`. Strict validation (§5-8):
≥2 unique non-empty option ids; single_choice/true_false → exactly one correct; true_false → exactly two
options; multiple_choice → set-equality, order-independent, **duplicate learner selections invalid** (§8/27).
Deferred (submission → `ACTIVITY_TYPE_NOT_SUPPORTED`): LISTENING/WRITING/SPEAKING/AI_INTERACTION (§33). View-only
(TEXT/EXPLANATION/IMAGE/AUDIO/EXAMPLE) never create attempts (§3). Publish-time validation of these payloads is
a future authoring-phase requirement (§12).

## 12. ActivityAttempt evidence — implemented, append-only (§22-24)
`ActivityAttempt` (`lessonRevisionId` pinned, `attemptNo` + `unique(userId, activityId, attemptNo)`, `answer`,
`isCorrect`, `deterministicScore`, `clientRequestId`, status **SUBMITTED**). A one-request objective answer is
persisted directly as SUBMITTED (no long-lived IN_PROGRESS row, §23). Once SUBMITTED it is immutable — retry
returns existing evidence, an intentional re-attempt (new `clientRequestId`) creates a new row (§24/49).
`roadmapItemId`/`learningSessionId` left null (progress retains no unambiguous context, §26/27). `EVALUATED`
and `AiEvaluation` remain for a later AI phase — none created here.

## 13. Deterministic scoring (§9/26)
Backend is the scoring authority (`ObjectiveActivityScorerService`): exact-match, 10000 correct / 0 incorrect,
no partial credit, no AI, no fuzzy normalization. The client sends only the answer; injected top-level fields →
400 (ValidationPipe), nested answer fields / unknown/duplicate option ids → `ACTIVITY_INVALID_RESPONSE`.

## 14. Idempotency / attemptNo (§14-24)
`clientRequestId` is REQUIRED (§15) and is the durable dedup identity (`ux_attempt_client_request`, L5). Same
requestId + canonically-equal answer → idempotent replay (200, same attempt); same requestId + different answer
or activity → `ACTIVITY_ATTEMPT_REQUEST_CONFLICT` (409). `attemptNo` is server-assigned via a bounded
retry-on-`unique(userId,activityId,attemptNo)` loop — concurrency-safe: concurrent same-requestId → one row,
concurrent different-requestIds → distinct attemptNos (tested §47/48). Progress cache (`completedActivities`,
`lastActivityId`) is updated once per new attempt (dedup), never on replay (§29).

## 15. Answer-key protection (§10/28/39/49)
Objective activities are now learner-visible via a deliberate projection (`{id, type, position, format, prompt,
options}`) — the raw `Activity.payload` and `answerKey`/`correctOptionIds` are never returned across start /
resume / attempt (tested §39/49). View-only/unsupported types expose metadata only.

## 15. Attempt numbering / idempotency (§23/24, when implemented)
`attemptNo` server-assigned via `unique(userId, activityId, attemptNo)`; `clientRequestId` is the accepted
durable retry-dedup identity (unlike Assessment, which had none). Intentional re-attempts create new evidence.

## 16. Concurrent start (§11/46) — implemented and tested (see §10).

## 17. Progress mutation (§31/32)
Start creates progress (status IN_PROGRESS). No activity submission this phase, so no `lastActivityId`/
`completionPct` advancement is triggered yet (those update once submission exists). No roadmap/dailyplan/skill
mutation.

## 18. APIs (§39/40)
| Method | Path | Notes |
|---|---|---|
| POST | `/api/lesson-executions/daily-plan-items/:dailyPlanItemId/start` | start-or-resume; pins revision; learner-safe content |
| GET  | `/api/lesson-executions/:lessonId` | resume own execution; PINNED revision |
| POST | `/api/lesson-executions/:lessonId/activities/:activityId/attempts` | submit objective answer `{clientRequestId, answer}`; server scores + records evidence |

Errors: `DAILY_PLAN_ITEM_NOT_FOUND` (404), `LESSON_NOT_EXECUTABLE` (409), `LESSON_ALREADY_COMPLETED` (409),
`LESSON_PROGRESS_NOT_FOUND` (404), `ACTIVITY_NOT_FOUND` (404), `ACTIVITY_NOT_IN_PINNED_REVISION` (409),
`ACTIVITY_TYPE_NOT_SUPPORTED` (409), `ACTIVITY_PAYLOAD_INVALID` (409), `ACTIVITY_INVALID_RESPONSE` (400),
`ACTIVITY_ATTEMPT_REQUEST_CONFLICT` (409). **Submit authority (§40/41):** own progress + pinned-revision
membership; the original DailyPlan need NOT remain today's CURRENT for each submit (execution is authority once
started) — only NEW lesson start requires today's plan.

## 19. Security (§47)
Own-user everywhere: a foreign/stale DailyPlanItem → 404; another user's progress → 404; no-auth → 401
(tested §47). No answer keys / draft content / authoring metadata exposed.

## 20. Tests (unit 25 new; e2e 17; project total unit 220 / e2e 143)
- **Unit:** payload contract §46 (formats, malformed rejection, learner projection strips key, objective-type
  set); scorer §45 (single/true_false/multiple correct/incorrect/malformed/unknown/**duplicate-invalid**/extra;
  canonicalize). Executability reuses already-tested `deriveItemState`.
- **E2e (foundation, 10):** start + pin + content + no leak + no side-effects; idempotent start §10;
  revision-pin-after-republish §45; no-arbitrary/stale §42; BLOCKED→startable §43/44; already-completed §7;
  archived §48; concurrent start §46; IDOR §47; resume-before-start 404.
- **E2e (submission, 7):** correct/incorrect scoring + append-only + progress step + no side-effects §50/51;
  idempotency (replay / conflict / new-attempt) §47; concurrency (same/different requestId, attemptNo) §48;
  revision authority (v1 OK / v2 reject) §42; client-score injection + unsupported type §44/33; learner-safe
  objective content §39; cross-lesson + submit IDOR §43/47.
- Regression: all prior phases green.

## 21. LessonCompletion deferred (§33) — see §6.
## 22. Skill update deferred (§36) — no `LearnerSkillState`/`SkillMeasurement` writes.
## 23. Roadmap reconcile deferred (§34) — `reconcileRoadmapCompletion` not called; roadmap unchanged.
## 24. Rewards / signals / AI boundary (§37/38/30)
No XP/IZL/RewardGrant/DailyMissionCompletion, no LearnerSignal, no AI/AiEvaluation. Verified absent.

## Side-effect verification
LearnerLessonCompletion / ActivityAttempt / Roadmap / DailyPlan / LearnerSkillState / SkillMeasurement /
LearnerSignal / Rewards / XP / IZL / AI — **NONE** (grep-confirmed: writes only `learnerLessonProgress`).

## Architecture gap — CLOSED
The Phase 1.7B gap (**Owner Review — Lesson Activity v1 Contract**) is RESOLVED: the owner accepted
`lesson-activity-objective/v1` (TD-108/109). ActivityAttempt submission + objective scoring are now
implemented. No schema change was needed. No remaining gap.

## Remaining OPEN — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)
Lesson Activity v1 contract · free-study/library entry · relearning completed lessons · lesson completion rule ·
required vs optional activity policy · mastery-from-lesson evidence · lesson SkillMeasurement · retry limits ·
hints · partial credit · AI/writing/speaking activities · time-on-task · roadmap reconcile hook · rewards/signals.
