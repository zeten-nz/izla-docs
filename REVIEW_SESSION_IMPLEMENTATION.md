# Review Session Execution — Implementation (Phase 1.9B-2)

> Status: **COMPLETE** (2026-08-20). Closes the Phase 1.9B architecture gap. A learner explicitly starts review
> from a currently-valid ReviewCandidate; the server pins the **encountered** LessonRevision, snapshots the
> Skill-specific objective Activities, records deterministic review attempts with durable provenance, and can
> complete the session — **without** touching normal Lesson completion, mastery, skill state, roadmap, plan, or
> signals. **One migration** (dedicated review aggregate + isolation discriminator). Owner: **TD-125/126/127/128**.

Code: `backend/src/review-session/**` (pure `selection/review-activity-selection.ts`; `review-session.repository.ts`;
`review-session.service.ts`; `review-session.controller.ts`) + 1.7C isolation hardening.

## 1. Architecture gap resolution
Phase 1.9B correctly stopped because the generic `LearningSession` cannot represent review provenance. Owner
review accepted a **dedicated** aggregate. This phase implements it: `LearnerReviewSession` + immutable
`LearnerReviewSessionActivity` snapshot + `ActivityAttempt.reviewSessionId`.

## 2. Why generic LearningSession is not reused (TD-128)
`LearningSession` stays a generic time-tracking aggregate (mission/reward/daily-plan). It was NOT expanded with
review-domain FKs/type/context — that would couple review semantics into a shared aggregate. Review gets its own
historical provenance aggregate.

## 3. Dedicated LearnerReviewSession (TD-125)
`{ id, userId, skillId (target), lessonId (logical), lessonRevisionId (pinned encountered), status
(ReviewSessionStatus ACTIVE/COMPLETED), provenance JSONB, startedAt, completedAt }`. Subject is derived via
`Skill.subjectId` (no duplicated `subjectId`). Load-bearing FKs (user/skill/lesson/revision) are `onDelete
Restrict` (provenance is history, never SetNull). Lifecycle ACTIVE→COMPLETED, terminal; a later episode is a new row.

## 4. Selected Activity relational snapshot (TD-126)
`LearnerReviewSessionActivity { id, reviewSessionId, activityId, position }` with `unique(reviewSessionId,
activityId)`, `unique(reviewSessionId, position)`, `CHECK position > 0`. This is the **immutable membership +
order authority** — never `provenance.selectedActivityIds`. Created atomically with the session header (§29).

## 5. ActivityAttempt.reviewSessionId (TD-127)
New nullable FK (`onDelete Restrict`). Normal execution = `NULL`; review attempts = the session id. This is the
**canonical normal/review discriminator** — never `answer` JSON / `clientRequestId` / `learningSessionId`.

## 6. Candidate start authority (§4/15/16)
`POST /api/review-sessions/me/subjects/:subjectId/skills/:skillId/lessons/:lessonId/start` (own-user, no body).
On a NEW start the server re-derives the 1.9A candidate policy from scratch via
`ReviewService.assertCandidateAvailable` (Skill in Subject, ACTIVE supported signal, current candidate,
encountered, visible, explicit relevance). Not a current candidate → `REVIEW_CANDIDATE_NOT_AVAILABLE`. The client
never supplies revision/activities/signalTypes/score/status.

## 7. Active-session resume semantics (§30/31)
Under a per-(user,skill,lesson) advisory lock, an existing ACTIVE session is returned as-is — **no** candidate/
signal revalidation, no re-selection, no snapshot change. Signal revalidation is only for NEW creation.

## 8. Encountered revision pinning (§18)
`LearnerLessonCompletion.lessonRevisionId` (if any) else `LearnerLessonProgress.lessonRevisionId`. Persisted once,
never repinned.

## 9. Completion-over-progress precedence (§8/59)
Completion exposure wins: when both exist, the completion revision is pinned.

## 10. Historical revision behavior (§9/19/20)
The pinned revision may be superseded (current published moved on); review still serves the encountered revision
(revision-pinned history). The revision id always comes from the learner's own Completion/Progress — never an
arbitrary/current substitute. The logical Lesson must still be currently visible (revalidated at start).

## 11. Target Skill (§11/21)
One session targets exactly one Skill + one logical Lesson + one pinned revision. Same Lesson + another Skill is a
distinct intent (allowed).

## 12. Activity selection (TD-126, §22/23)
Supported objective types only (MINI_QUESTION/PRACTICE/MASTERY_TEST, `lesson-activity-objective/v1`). Select A iff
`ActivitySkill(A, target)` OR (A has **zero** ActivitySkill AND `LessonSkill(lesson, target)`).

## 13. Explicit attribution override (§14/24)
The LessonSkill fallback applies **only** to activities with no ActivitySkill at all. An Activity explicitly mapped
to another Skill is never broadened into a target-Skill session (unit + e2e §68).

## 14. Direct-trigger ordering (§16/26)
The ACTIVE REPEATED_MISTAKE signal's `triggerActivityIds` (strict parser) that are selected + in the pinned
revision are ordered first, then Activity.position, then id. Direct-trigger does not bypass attribution (§26).
Order is persisted as `position`; no `directTrigger` boolean is stored.

## 15. Snapshot immutability (§29/63/73)
Membership is fixed at creation; later mapping/content/signal changes never alter an ACTIVE session's activities.

## 16. Scorer reuse (§79)
Reuses `lesson-activity-objective/v1` payload parser, canonicalizer, and deterministic scorer (10000/0). No new
question contract; review changes provenance/selection, not correctness.

## 17. ActivityAttempt idempotency (§35/41/42)
Reuses durable `clientRequestId` semantics: same id + same activity + same session + canonical-equal answer →
replay; conflicting id → `ACTIVITY_ATTEMPT_REQUEST_CONFLICT`; new id → new attempt. `attemptNo` continues the
shared `unique(userId, activityId, attemptNo)` sequence (normal + review interleave; no `reviewAttemptNo`).

## 18. Normal LessonProgress isolation (§31/40)
Review submit never calls `recordActivityStep` and never touches `LearnerLessonProgress` (completedActivities /
lastActivityId / status / masteryBestScore). Verified (e2e §74).

## 19. Normal LessonCompletion isolation (TD-127, §10/43)
1.7C `LessonCompletionRepository.submittedActivityIds` now filters `reviewSessionId IS NULL`. A review attempt can
**never** satisfy normal completion (mandatory e2e §78: normal complete → `LESSON_NOT_READY_FOR_COMPLETION`).

## 20. lesson-mastery-v1 isolation (§11/34/44)
1.7C `LessonCompletionRepository.bestScores` now filters `reviewSessionId IS NULL`. A review MASTERY_TEST score
never alters `LESSON_MASTERY` (mandatory e2e §79: normal 0 wins over review 10000). Both completion-evidence queries
were audited; these are the only two.

## 21. Review completion (§40/46/47/50)
`POST /:sessionId/complete`. Ready iff every `LearnerReviewSessionActivity` has ≥1 SUBMITTED attempt with
`reviewSessionId = sessionId`. Correctness never gates. Not ready → `REVIEW_SESSION_NOT_READY`; already COMPLETED →
idempotent return. A normal attempt does not satisfy review completion (e2e §80). Conditional ACTIVE→COMPLETED
(concurrency-safe, §49).

## 22. Signal boundary (§54/55)
Review start/attempt/complete never mutates `LearnerSignal`. REVIEW_DUE/WEAK_SKILL do not resolve from review.
Review attempts are legitimate objective `ActivityAttempt` history, so a future explicit signal reconcile MAY
consume them — the review module invokes no signal service (grep-verified).

## 23. SkillMeasurement boundary — extended by Phase 1.9C
As shipped in 1.9B-2, review completion created no SkillMeasurement/state update. **Phase 1.9C** now normalizes a
COMPLETED session into one `REVIEW_MASTERY` SkillMeasurement, recomputes current state via merge-v2 (LearningProgress
stays the sole state writer), and recovers signals via LearnerSignals. Review attempts remain excluded from
lesson-mastery-v1; no LearnerLessonCompletion is created. See `REVIEW_MASTERY_IMPLEMENTATION.md`.

## 24. Roadmap / DailyPlan boundary (§56/57)
No RoadmapItem / DailyPlanItem / EXTRA / plan mutation; one-topic-per-day intact. Normal DailyPlan-only Lesson
start is unchanged (§50). No daily review cap (§58). No rewards/notifications/AI.

## 25. Security / concurrency (§88/89, §32/49)
Own-user only (foreign GET/submit/complete → 404; no auth → 401; bad id → 400). Responses expose no
answerKey/payload/raw answers/provenance internals. Concurrency: `uq_review_session_active (user, skill, lesson)
WHERE ACTIVE` partial unique + review-namespaced advisory lock + P2002-winner-resume → one ACTIVE session;
`clientRequestId`/`attemptNo` uniques → one attempt; conditional status update → one COMPLETED transition.

## 26. Tests
- Unit: `selection/review-activity-selection.spec.ts` (6) — target selection, explicit override, no-fallback,
  direct-trigger order, trigger-not-bypassing-attribution, empty.
- E2e: `test/review-session.e2e-spec.ts` (15) — encountered-completion/progress revision pinning; skill-specific
  selection + override + no-key-leak; direct-trigger order; no-reviewable → error; candidate revalidation + active
  resume after signal resolution; snapshot immutability + start idempotency; submit correct/incorrect + provenance
  + progress-untouched + arbitrary/cross-revision reject; **§78 normal-completion isolation**; **§79 lesson-mastery
  isolation**; normal-attempt-not-review-complete; review complete (correctness-irrelevant) + no completion/measurement;
  completed→new-episode; concurrent start/attempt/complete; IDOR + full side-effect boundary + 401.
- Totals: unit **292**, e2e **208**.

## 27. Future Review Mastery (Phase 1.9C)
A future phase normalizes completed-review evidence into a SkillMeasurement source, merges via LearningProgress,
and lets existing signal policies react (REVIEW_DUE/WEAK_SKILL recovery). Deliberately not in 1.9B-2.
