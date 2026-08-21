# Review Mastery Evidence + Merge v2 + Signal Recovery — Implementation (Phase 1.9C)

> Status: **COMPLETE** (2026-08-20). A COMPLETED Review Session normalizes into one append-only
> `REVIEW_MASTERY` `SkillMeasurement`; review evidence enters current Skill state through
> **learning-progress-merge-v2** (incremental, never an anchor); newer review evidence recovers REVIEW_DUE /
> WEAK_SKILL / REPEATED_MISTAKE through the **existing** signal policies. Review completion stays independent of
> correctness. **One migration** (REVIEW_MASTERY source + review provenance + idempotency). Owner:
> **TD-129/130/131/132**. No DailyPlan/Roadmap/reward/notification/AI.

Code: `backend/src/review-session/mastery/review-mastery.engine.ts` (pure), `review-mastery.service.ts`;
`learning-progress/merge/{merge-core.ts, learning-progress-merge.engine.ts (v1), learning-progress-merge-v2.engine.ts}`.

## 1. Review completion ≠ review mastery score
Review Session completion means the configured reinforcement set was performed (correctness never gates, 1.9B-2).
Review **mastery** is a separate normalized milestone derived from that same completed evidence. They are distinct.

## 2. review-mastery-v1 (TD-129)
Immutable derivation `review-mastery-v1`. Uses ONLY the session's `LearnerReviewSessionActivity` snapshot + the
SUBMITTED ActivityAttempts with `reviewSessionId = session.id` (§7) — never normal attempts, other sessions,
current mappings, or the current revision. No source re-attribution: the session's target Skill is authority (§10).

## 3. Best attempt per selected Activity (§8/9)
`bestReviewScore(A) = max(deterministicScore)` over this session's SUBMITTED attempts for A. Completion guaranteed
≥1 attempt per selected Activity, so every selected Activity contributes. Raw retries stay append-only.

## 4. Formula (§10/11)
`scoreBp = clamp(round(mean(best per selected Activity)), 0, 10000)` (BigInt round-half-up). One Skill → no extra
attribution. Example: best {10000, 0, 10000} → `round(20000/3) = 6667`.

## 5. Confidence / evidenceCount / observedAt / displayLevel (§12–15)
`confidenceBp = 10000` (complete coverage of the configured snapshot — not statistical certainty).
`evidenceCount = distinct selected Activities` (not retry count). `observedAt = LearnerReviewSession.completedAt`
(logical milestone time). `displayLevel = null`.

## 6. COMPLETED-only + defensive validation (§16/17)
Derivation runs only for `status = COMPLETED` with non-null `completedAt`. It verifies ≥1 selected Activity and
that every selected Activity has a session-linked SUBMITTED attempt; otherwise `REVIEW_MASTERY_CONFIGURATION_INVALID`
(no partial measurement). A non-COMPLETED session yields `{ measured: false }`.

## 7. Measurement source + provenance (TD-129/130)
New `SkillMeasurementSource.REVIEW_MASTERY`. New nullable `SkillMeasurement.reviewSessionId` FK (Restrict,
history-safe). REVIEW_MASTERY rows set `reviewSessionId`; `lessonId` is left NULL (§4) so recurring reviews of the
same Lesson never collide with lesson-backed idempotency. Existing DIAGNOSTIC/LESSON_MASTERY/CHECKPOINT provenance
is unchanged; existing rows keep `reviewSessionId = NULL` (no backfill).

## 8. Append-only + idempotency (§5/18, RM-DB-02)
Append-only (create only; no UPDATE/DELETE). Partial unique `uq_skill_measurement_review_idempotency
(review_session_id, skill_id, source, derivation_version) WHERE review_session_id IS NOT NULL` +
`createMany skipDuplicates` → at most one review-mastery-v1 measurement per session (a future review-mastery-v2 can
coexist). Recurring Review Sessions produce distinct historical measurements (distinct reviewSessionId).

## 9. learning-progress-merge-v2 (TD-131)
v1 (`learning-progress-merge-v1`) is **frozen** and still documented/tested as its historical contract. v2 is the
CURRENT materialization engine. Both delegate to a shared pure core (`merge-core.ts`); they differ ONLY in the
source config. v2 = v1 formulas + anchor semantics, with the incremental source policy extended to include
REVIEW_MASTERY. Refactor is behavior-preserving (v1 tests unchanged).

## 10. v2 sources / anchor / window (§20–24)
Supported: DIAGNOSTIC, CHECKPOINT, LESSON_MASTERY, REVIEW_MASTERY (AI_EVALUATION/ENGINE_RECALC still excluded).
Anchors: **only** DIAGNOSTIC/CHECKPOINT — REVIEW_MASTERY is never an anchor (§20/25). Window = anchor + LESSON_MASTERY
+ REVIEW_MASTERY strictly after it; no anchor → all LESSON_MASTERY + REVIEW_MASTERY. Weights/formulas unchanged
(`effectiveWeight = evidenceCount × confidenceBp`; weighted means; window-sum evidenceCount; max observedAt).

## 11. Incremental, can worsen (§25/36/37)
Review is incremental reinforcement evidence — it moves current state gradually, never resetting the baseline like a
calibration anchor. Poor review **lowers** current mastery (real evidence, not `max`). Example: diagnostic 6000/count4
+ review 9000/count2 → 7000/count6.

## 12. CHECKPOINT still resets review history (§26/61)
A newer CHECKPOINT anchor excludes older DIAGNOSTIC/LESSON_MASTERY/REVIEW_MASTERY from the current window (all
preserved as history). Verified.

## 13. v1 compatibility (§27/62)
For any history with no REVIEW_MASTERY, v2 produces byte-for-byte identical results to v1 (property fixtures compare
`mergeSkill` vs `mergeSkillV2`). No learner state changes merely because the current engine moved to v2. `LearnerSkillState`
carries no mergeVersion column (rebuildable projection; derivation identity lives on SkillMeasurement.derivationVersion).

## 14. Completion workflow (§30/31/40)
`ReviewSessionService.complete`: **A** authoritative ACTIVE→COMPLETED → **B** `ReviewMasteryService.ensureDerived`
(REVIEW_MASTERY) → **C** `LearningProgressService.recomputeSkills([skillId])` (merge-v2 materializes state AND fires
the 1.8C state-signal hook → WEAK_SKILL/REVIEW_DUE) → **D** `LearnerSignalsService.evaluateSkills([skillId])`
(REPEATED_MISTAKE reconcile). B/C/D run outside A's transaction, wrapped so a downstream failure never rolls back the
COMPLETED session. An already-COMPLETED `complete` re-runs B/C/D (idempotent recovery, §31).

## 15. Single writers hold (§33/75/83)
Review creates SkillMeasurement then calls LearningProgress to write `LearnerSkillState` — LearningProgress remains
its sole writer (TD-115). Signals change only via LearnerSignalsService — LearnerSignals remains the sole
`LearnerSignal` writer. Verified by grep (state writer = learning-progress repo only; signal writer = learner-signals repo only).

## 16. REVIEW_DUE recovery (§34/37/69/70)
Review measurement `observedAt = completedAt` advances current `lastMeasurementAt` after merge, so the existing
review-due policy resolves an old REVIEW_DUE (basis strictly before). Resolution requires new evidence, **not**
improvement — a poor review still resolves the old obligation; the next interval schedules from the new state. The
Review module never sets REVIEW_DUE RESOLVED directly.

## 17. WEAK_SKILL recovery (§35/71/72)
If merged state reaches mastery ≥ 6500 & confidence ≥ 7000, the existing weak-skill policy resolves WEAK_SKILL; within
the 5000..6499 hysteresis band it stays ACTIVE. The Review module hardcodes nothing.

## 18. REPEATED_MISTAKE recovery (§38/73/74)
Review ActivityAttempts are legitimate objective evidence (TD-116). Phase D calls `LearnerSignalsService.evaluateSkills`
so REPEATED_MISTAKE reconciles from the latest distinct outcomes (two distinct correct → RESOLVED; one correct →
stays ACTIVE). The Review module implements no detector logic.

## 19. Isolation preserved (§42/43/78)
Review attempts remain excluded from lesson-mastery-v1 and normal LessonCompletion (1.9B-2 `reviewSessionId IS NULL`
filters). REVIEW_MASTERY is a separate source; review completion creates no LearnerLessonCompletion (§43).

## 20. Boundaries (§44–48/82)
Review completion writes only REVIEW_MASTERY SkillMeasurement + LearnerSkillState (via LearningProgress) + LearnerSignal
(via LearnerSignals). It never writes LearnerLessonProgress/Completion, Roadmap/RoadmapItem, DailyPlan/DailyPlanItem,
RewardGrant/DailyMission, Notification, or AiEvaluation (grep + e2e).

## 21. APIs (§49–53)
`POST /:sessionId/complete` response gains a learner-safe `mastery` summary `{ measured, skillId, scoreBp, confidenceBp,
evidenceCount, displayLevel:null }`; `GET /:sessionId` includes it when derived. No client-supplied score/evidence/
confidence/observedAt/skillId (all server-derived, §80). Skill Profile
(`GET /skill-profile/me/subjects/:subjectId`) reflects the merged state; `POST /learning-progress/…/recompute` and
`POST /learner-signals/…/reconcile` are the repair paths (now consuming REVIEW_MASTERY via merge-v2). No new endpoints.

## 22. Security (§80/81)
Own-user only (foreign session → 404). Responses expose no ActivityAttempt.answer / Activity.payload / answer keys /
signal evidenceRefs / thresholds — only the learner-safe mastery summary.

## 23. Tests
- Unit: `review-mastery.engine.spec.ts` (5) — mean/round/evidenceCount, all-wrong, single, midpoint, empty-throws;
  `learning-progress-merge-v2.engine.spec.ts` (9) — review-only, diagnostic+review, lesson+review, review-lowers,
  review-not-anchor, checkpoint-reset, v1-compatibility fixtures, unsupported excluded, null.
- E2e: `test/review-mastery.e2e-spec.ts` (8) — measurement (mean best) + provenance + response summary; retries→count /
  all-wrong; merge-v2 into Skill Profile; idempotent + concurrent complete → one measurement; REVIEW_DUE resolution
  (incl. poor score); WEAK_SKILL hysteresis; REPEATED_MISTAKE recovery; side-effect boundary + review isolation.
- Totals: unit **306**, e2e **216**.

## 24. Future Review Mastery v2 (§87)
Statistical scoring, latest-vs-best future revision, success threshold, retry limits, CEFR, time decay,
source-specific weights — all OPEN. A math change ships as review-mastery-v2 (coexisting via derivationVersion).
