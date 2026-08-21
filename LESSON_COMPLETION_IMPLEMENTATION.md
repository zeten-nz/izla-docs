# Lesson Completion + Lesson Mastery Milestone + Roadmap Reconciliation — Implementation (Phase 1.7C)

> Status: **COMPLETE** (2026-08-20). A learner who has performed every Activity in the exact pinned
> `LessonRevision` can complete the Lesson exactly once; lesson-mastery `SkillMeasurement` milestones are
> derived from MASTERY_TEST evidence only; the Roadmap is reconciled ACTIVE→COMPLETED through the existing
> `LearnerLessonCompletion` authority. **One schema change** — a lesson-backed `SkillMeasurement` idempotency
> partial-unique (migration `20260820120000`, SP-10). Owner decisions: **TD-110 / TD-111 / TD-112**.

Code: `backend/src/lesson-execution/completion/**` (pure `lesson-completion-eligibility.ts`,
`lesson-mastery.engine.ts`; `lesson-completion.repository.ts`; `lesson-completion.service.ts`).

## 1. Scope
Authenticated learner, on their own in-progress execution, can: (a) mark a **view-only** activity step
performed, and (b) **complete** the Lesson when every pinned-revision activity has been performed. Completion
persists `LearnerLessonCompletion` + terminal `LearnerLessonProgress`, derives lesson-mastery
`SkillMeasurement` rows (source `LESSON_MASTERY`), and reconciles any ACTIVE roadmap containing the lesson.
**NOT in scope:** `LearnerSkillState` merge, Signals, XP/IZL, Daily Missions, AI, `RoadmapItem.status` direct
mutation, `DailyPlan` mutation.

## 2. Endpoints
- `POST /api/lesson-executions/:lessonId/activities/:activityId/complete` — mark a view-only step (no body).
- `POST /api/lesson-executions/:lessonId/complete` — complete the lesson (no body; idempotent).

Both are own-user only (global AuthGuard; `userId` from the principal). Foreign / no-progress lesson → 404;
no auth → 401 (tested §44).

## 3. Completion policy (TD-110, lesson-completion-v1)
A Lesson is **eligible** for completion iff `computeEligibility` returns `eligible` — i.e. every activity of the
pinned revision has been **performed** and none is **unsupported**, and the revision has ≥1 activity:
- **Objective** activity (MINI_QUESTION / PRACTICE / MASTERY_TEST) performed = it has a **SUBMITTED**
  `ActivityAttempt` in the pinned revision (existence only — **correctness never gates completion**, §12).
- **View-only** activity (TEXT / EXPLANATION / IMAGE / AUDIO / EXAMPLE) performed = its id is in
  `LearnerLessonProgress.completedActivities` (set-union recorded via the view-only step endpoint).
- **Unsupported** activity (LISTENING / WRITING / SPEAKING / AI_INTERACTION / VIDEO) present as a required step
  ⇒ the lesson **cannot** be completed in v1 → `LESSON_COMPLETION_UNSUPPORTED_ACTIVITY` (409). We do not
  invent optional-vs-required semantics; every pinned activity is required.

Not eligible → `LESSON_NOT_READY_FOR_COMPLETION` (409). Zero-activity pinned revision →
`LESSON_CONFIGURATION_INVALID` (409, §9).

## 4. Pure eligibility engine (`lesson-completion-eligibility.ts`)
`computeEligibility(activities, completedSet, submittedSet) → { eligible, totalActivities, completedActivities,
remainingActivityIds, unsupportedActivityIds }`. No I/O, no clock. `OBJECTIVE_TYPES` / `VIEW_ONLY_TYPES`
classify each activity; anything else (deferred types, VIDEO, unknown) becomes `unsupportedActivityIds`.
`eligible = activities.length > 0 && unsupported.length === 0 && remaining.length === 0`. Unit-tested (§47/50/51).

## 5. View-only step marking (§5)
`markViewOnlyStep` requires an IN_PROGRESS own execution; the activity must belong to the pinned revision and
be a view-only type (objective/deferred/unknown → `ACTIVITY_TYPE_NOT_SUPPORTED`). It calls the reused
`ActivityAttemptRepository.recordActivityStep` (JSONB set-union) — **idempotent**, no `ActivityAttempt` created.
A completed (terminal) lesson rejects further step marking (`LESSON_ALREADY_COMPLETED`).

## 6. Lesson-mastery milestone (TD-111, lesson-mastery-v1)
Milestones are derived **only** from MASTERY_TEST evidence (MINI_QUESTION / PRACTICE never contribute to
mastery). For each MASTERY_TEST activity of the pinned revision:
- **bestScore** = `max(deterministicScore)` across the learner's SUBMITTED attempts for that activity (§20).
- **Attribution**: the activity's `ActivitySkill` rows; if it has **none**, fall back to the Lesson's
  `LessonSkill` rows (§24/25). If neither exists → the evidence is **unattributed** and produces no measurement
  (we never fabricate a skill).
- **Subject scope (§66)**: attributed skills whose `Subject` ≠ the lesson's `Subject` are excluded (no
  cross-subject `SkillMeasurement`); if that leaves nothing, the activity is unattributed.

Per Skill: `scoreBp = clamp(round(mean(best scores of the mastery activities attributed to it)), 0, 10000)`,
`confidenceBp = 10000`, `evidenceCount = # distinct mastery activities contributing`. One measurement row per
Skill (source `LESSON_MASTERY`, `derivationVersion = 'lesson-mastery-v1'`, `displayLevel = null`).

## 7. Pure mastery engine (`lesson-mastery.engine.ts`)
`deriveLessonMastery(inputs: {activityId, bestScoreBp, skillIds}[]) → {skillId, scoreBp, confidenceBp,
evidenceCount}[]`, sorted by skillId. Groups best scores by skill (dedup skillIds within an activity), applies
the mean/round/clamp above, confidence always 10000. No I/O; attribution + subject-scope filtering happen in
the service before calling it. Unit-tested (§53/54/55/26).

## 8. `evidenceCount` is response-only
`SkillMeasurement` has no `evidence_count` column; `evidenceCount` is surfaced in the API response view only.
Persistence writes `{userId, skillId, source, lessonId, scoreBp, confidenceBp, displayLevel, derivationVersion}`.

## 9. Orchestration (`LessonCompletionService.completeLesson`, §38)
Three idempotent phases — **not** one giant transaction (recoverability > atomicity across subsystems):
- **Phase A — completion authority.** Validate eligibility → compute `masteryBestScore` cache (max
  deterministicScore across pinned MASTERY_TEST attempts, else null) → `createCompletion` (completionNo=1) +
  conditional `markProgressCompleted` (`updateMany WHERE status = IN_PROGRESS`). A P2002 on the completion
  unique means a concurrent writer won; we read the winner's `completedAt` and proceed (§15).
- **Phase B — mastery.** `ensureMasteryDerived` (idempotent `createMany skipDuplicates`), safe to re-run.
- **Phase C — roadmap.** For each ACTIVE roadmap containing the lesson, call the existing
  `RoadmapService.reconcileCompletion` (§40). Failures are logged and deferred; the completion still stands.

If a completion already exists, `completeLesson` **replays** Phases B+C and returns current state — recovery
from a partial prior run (§16/39).

## 10. Idempotency (§59/60)
- **Completion**: `@@unique(userId, lessonId, completionNo)` with `completionNo = 1` ⇒ exactly one completion.
- **Progress**: conditional `updateMany` (only IN_PROGRESS→COMPLETED) is safe under replay/concurrency.
- **Measurement**: partial-unique `uq_skill_measurement_lesson_idempotency (user_id, lesson_id, skill_id,
  source, derivation_version) WHERE lesson_id IS NOT NULL` (SP-10) + `createMany skipDuplicates` ⇒ re-running
  completion never duplicates measurements. Concurrent double-complete → one completion row, one measurement set
  (tested §60).

## 11. LessonRevision pinning preserved (§52, L-14)
Completion reads activities/attempts strictly from `LearnerLessonProgress.lessonRevisionId` (the revision pinned
at first start) and stores it on `LearnerLessonCompletion.lessonRevisionId`. Publishing a new revision after
start never repins; the completion records the exact revision the learner performed.

## 12. `masteryBestScore` cache
Both `LearnerLessonCompletion` and the terminal `LearnerLessonProgress` cache `masteryBestScore` = max
deterministicScore across pinned MASTERY_TEST attempts (null when the revision has no MASTERY_TEST). This is a
convenience cache; the authoritative mastery evidence is the `SkillMeasurement` rows.

## 13. `completionPct` untouched (§68 OPEN)
The `completion_pct` scale/formula is an OPEN owner question (DATA_MODEL_LEARNING §269). Completion leaves
`completionPct` at whatever the execution phase set — no fabricated percentage.

## 14. Roadmap reconciliation (TD-112, §40/41)
`RoadmapItem` is **not** a duplicate completion authority — `LearnerLessonCompletion` is. Completion calls the
1.6B `RoadmapService.reconcileCompletion(userId, roadmapId)` hook, which derives item/roadmap state from
completion facts. When every item resolves, the roadmap flips ACTIVE→COMPLETED (tested §62); a partially
completed roadmap stays ACTIVE (§63). Reconciliation is idempotent and re-run on completion replay.

## 15. DailyPlan is snapshot-only (§64)
Completion never mutates `DailyPlan` / `DailyPlanItem`. `GET /daily-plans/today` continues to derive item state
(COMPLETED) from the same completion authority; re-`POST /today` returns the same plan (no new plan / next topic
manufactured by completion). Verified §64.

## 16. `LearnerSkillState` boundary — superseded by Phase 1.8A (§36/65)
As shipped in 1.7C, completion wrote `SkillMeasurement` but **not** `LearnerSkillState` (the multi-source merge
policy was OPEN). **Phase 1.8A intentionally lifts this boundary:** completion now, after ensuring the
LESSON_MASTERY measurements, calls `LearningProgressService.recomputeSkills(affected)` so current state is
merged (DIAGNOSTIC + LESSON_MASTERY) through the single writer. LessonCompletion still has **no direct**
`LearnerSkillState` write — state changes only via the merge engine (verified by the single-writer grep). The
old "state byte-for-byte unchanged" e2e was replaced with an assertion that completion recomputes the affected
skill through merge while untouched skills stay put (§70). See `LEARNING_PROGRESS_MERGE_IMPLEMENTATION.md`.

## 17. No other side effects
No Signals, XP/IZL, reward grants, Daily Missions, or AI evaluations are created (tested: counts stay 0, §51).
The completion module's only writes are `learnerLessonCompletion.create`, `learnerLessonProgress.updateMany`,
`skillMeasurement.createMany` (verified by side-effect grep).

## 18. Errors (AuthExceptionFilter → HTTP, leak-safe)
| Domain error | HTTP | code |
|---|---|---|
| `LessonConfigurationInvalidError` | 409 | `LESSON_CONFIGURATION_INVALID` |
| `LessonNotReadyForCompletionError` | 409 | `LESSON_NOT_READY_FOR_COMPLETION` |
| `LessonCompletionUnsupportedActivityError` | 409 | `LESSON_COMPLETION_UNSUPPORTED_ACTIVITY` |
| `LessonAlreadyCompletedError` | 409 | `LESSON_ALREADY_COMPLETED` (reused) |
| `ActivityNotFoundError` / `ActivityNotInPinnedRevisionError` / `ActivityTypeNotSupportedError` | 404/409/409 | reused (1.7B/1.7B-2) |
| `LessonProgressNotFoundError` | 404 | `LESSON_PROGRESS_NOT_FOUND` (reused) |

## 19. Schema change (the one migration)
`20260820120000_add_lesson_skill_measurement_idempotency` — the SP-10 partial-unique above. No table/column
additions; `evidence_count` intentionally **not** added (response-only). Drift clean on dev + test (6 migrations).

## 20. Security
Own-user IDOR enforced on both endpoints (foreign lesson → 404, tested §44). No answer keys / `Activity.payload`
/ scores are logged (the reconcile logger emits only roadmap ids). Answer-key evaluation stays server-side
(reused 1.7B-2 scorer path). Test DB `izlan_test` isolated via `assertTestDatabase`.

## 21. Reused building blocks
`ActivityAttemptRepository.recordActivityStep` + `findActivity` (1.7B-2), `RoadmapService.reconcileCompletion`
(1.6B), the objective scorer's `deterministicScore` evidence (1.7B-2), `LearnerLessonProgress` pinning (1.7B).
`RoadmapModule` now additionally exports `RoadmapService`; `LessonExecutionModule` imports it (one-way, no cycle).

## 22. Tests
- Unit: `lesson-completion-eligibility.spec.ts` (6), `lesson-mastery.engine.spec.ts` (5).
- E2e: `test/lesson-completion.e2e-spec.ts` (9) — mixed-lesson readiness + mastery + `LearnerSkillState`
  unchanged; incorrect-still-completes; best-attempt + multi-mastery mean; LessonSkill fallback; no-mapping /
  no-mastery-test → no measurement; unsupported-type + zero-activity rejection; idempotent + concurrent
  completion; roadmap reconcile (full → COMPLETED, partial → ACTIVE) + DailyPlan snapshot; IDOR + 401.

## 23. Acceptance mapping
TD-110 §3/4/5; TD-111 §6/7/8; TD-112 §14/15; boundaries §16/17; idempotency §10; pinning §11; security §20.

## 24. Follow-ups / OPEN
- `completion_pct` scale (OPEN, §13) — future.
- Multi-source `LearnerSkillState` merge (DIAGNOSTIC + LESSON_MASTERY + CHECKPOINT) — **Phase 1.8A** (STOP here).
- Deferred activity types (LISTENING/WRITING/SPEAKING/AI/VIDEO) completion support — future contract phases.
