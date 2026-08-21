# Learning Progress Merge Engine — Implementation (Phase 1.8A)

> Status: **COMPLETE** (2026-08-20). `LearnerSkillState` is now derived from normalized `SkillMeasurement`
> milestones by a single deterministic merge engine (`learning-progress-merge-v1`). DIAGNOSTIC +
> LESSON_MASTERY merge; CHECKPOINT is supported as a recalibration anchor (enum exists; producer future).
> **One schema change** — `SkillMeasurement.evidenceCount` + `observedAt` (NOT NULL) + a positive-evidence
> CHECK (migration `20260820130000`, LP-01). Owner decisions: **TD-113 / TD-114 / TD-115**.

Code: `backend/src/learning-progress/**` (pure `merge/learning-progress-merge.engine.ts`;
`learning-progress.repository.ts`; `learning-progress.service.ts`; `learning-progress.controller.ts`).

## 1. Scope
Materialize current `LearnerSkillState` from append-only `SkillMeasurement` history. One writer of state
(TD-115). Diagnostic and Lesson-completion flows create measurements and then request a recompute. A repair
endpoint rebuilds any subject's states from history. **NOT** in scope: Signals, Rewards, Roadmap
regeneration, DailyPlan, AI, CEFR/displayLevel, time-decay, IRT, retake scheduling (§69).

## 2. Evidence architecture
`AssessmentResponse → SkillMeasurement(DIAGNOSTIC)`; `ActivityAttempt → LessonCompletion →
SkillMeasurement(LESSON_MASTERY)`; future `Checkpoint → SkillMeasurement(CHECKPOINT)`. `SkillMeasurement`
is the append-only historical layer; `LearnerSkillState` is the mutable materialized projection. There is
**no** current-state history table (§63) — history already lives in `SkillMeasurement`.

## 3. SkillMeasurement normalized contract
Every mergeable milestone provides: `userId, skillId, source, scoreBp (0..10000), confidenceBp (0..10000),
evidenceCount (>0), observedAt, derivationVersion`, plus provenance (`attemptId`/`lessonId`). After 1.8A the
tuple `(scoreBp, confidenceBp, evidenceCount, observedAt)` is the frozen normalized milestone — merge never
reinterprets raw evidence.

## 4. evidenceCount (TD-113, LP-01)
New NOT NULL column `evidence_count` with CHECK `evidence_count > 0`. DIAGNOSTIC = objective response evidence
units for the skill; LESSON_MASTERY = distinct MASTERY_TEST activities attributed to the skill; CHECKPOINT =
its future engine's units. Response-only evidenceCount is gone — it is persisted at creation.

## 5. observedAt (TD-113)
New NOT NULL column `observed_at` = logical evidence-observation time (DIAGNOSTIC = `AssessmentAttempt.completedAt`;
LESSON_MASTERY = `LearnerLessonCompletion.completedAt`). Never materialization/backfill time. `SkillMeasurement`
had only `createdAt` (insertion time) — insufficient for ordering, so a dedicated logical timestamp was added.

## 6. Migration / backfill (§5 safety)
Migration `20260820130000_harden_skill_measurement_merge_metadata`. Both `izlan_dev` and `izlan_test` were
inspected before enforcement and held **0** `skill_measurement` and **0** `learner_skill_state` rows → NOT NULL
columns added directly, no backfill, no fabricated defaults, no MIGRATION DATA GAP. Had rows existed, the plan
was deterministic per-source backfill (DIAGNOSTIC from the attempt's per-skill responses; LESSON_MASTERY from
distinct MASTERY_TEST activities) or a reported gap — never `1`/`0`/`createdAt` fabrication.

## 7. Supported sources (§9/10)
Explicit whitelist: `DIAGNOSTIC, LESSON_MASTERY, CHECKPOINT`. `AI_EVALUATION`, `ENGINE_RECALC`, and any other
enum value are preserved as history but **never** affect v1 current state. CHECKPOINT already exists in
`SkillMeasurementSource`, so the pure engine + fixtures support it even though no CHECKPOINT producer is built.

## 8. Recalibration anchor (§11/12)
DIAGNOSTIC and CHECKPOINT are calibration anchors; LESSON_MASTERY is incremental. Anchor = the latest
DIAGNOSTIC/CHECKPOINT for the (user, skill), ordered `observedAt DESC → CHECKPOINT over DIAGNOSTIC on a tie →
greatest id (stable)`. A newer accepted calibration recalibrates current state (this is merge behavior only —
it does **not** authorize or schedule retakes).

## 9. Merge window (§13/14/15/16)
With an anchor: window = anchor + every supported LESSON_MASTERY with `observedAt` **strictly after** the
anchor (equal-time lesson loses the tie to the anchor). Older DIAGNOSTIC/CHECKPOINT/LESSON_MASTERY stay in
history but leave the current window. With no anchor (import/migration edge): window = all supported
LESSON_MASTERY (no fabricated diagnostic baseline).

## 10. Effective evidence weight (§17)
`effectiveWeight(m) = evidenceCount(m) × confidenceBp(m)`. Accumulated with BigInt (no division before
summation, no float persistence). The common 10000 confidence scale cancels from the weighted mean.

## 11. Mastery formula (§18)
`masteryScoreBp = clamp(round( Σ(scoreBp·evidenceCount·confidenceBp) / Σ(evidenceCount·confidenceBp) ), 0, 10000)`.
Denominator 0 (all-zero confidence) → `LEARNING_PROGRESS_NO_EFFECTIVE_EVIDENCE` (no fake zero state). A single
uncertain (low-confidence) baseline adapts faster to lessons — accepted v1 behavior (§19).

## 12. Confidence formula (§20)
`confidenceBp = clamp(round( Σ(confidenceBp·evidenceCount) / Σ(evidenceCount) ), 0, 10000)` — an evidence-count
weighted mean of reliability/coverage. NOT Bayesian / IRT / statistical certainty.

## 13. Current evidenceCount (§21)
`LearnerSkillState.evidenceCount = Σ evidenceCount` across the current window — actual evidence UNITS, not the
number of measurement rows (e.g. diagnostic 4 responses + lesson 2 mastery activities → 6).

## 14. displayLevel (§22)
Always `null` in 1.8A. No CEFR / proficiency mapping; old `displayLevel` strings are never consumed as mastery
authority.

## 15. lastMeasurementAt (§23)
`= MAX(observedAt)` in the current window — logical evidence time, never materialization time.

## 16. Pure merge engine (§24/25)
`mergeSkill(NormalizedMeasurement[]) → MergeResult | null` — no Prisma, HTTP, AI, clock, or randomness.
Returns `null` when there is no supported measurement (caller must not write/delete state). Owns source
filtering, anchor + window selection, weights, mastery, confidence, evidenceCount, lastMeasurementAt. Defensive
bounds validation (§38/39) throws `LEARNING_PROGRESS_CONFIGURATION_INVALID`. Repository/service own I/O, auth,
transactions, persistence.

## 17. Single-writer invariant (TD-115, §26/53)
After 1.8A **only** `LearningProgressRepository.upsertState` writes `LearnerSkillState` — verified by grep (the
only `learnerSkillState.upsert/update/create/delete` in `src/`). SkillProfile's `materializeState` was removed;
LessonCompletion never wrote state. Other domains create `SkillMeasurement` then call
`LearningProgressService.recomputeSkills(...)`.

## 18. Concurrency / serialization (§31/32/33)
`recomputeSkill` runs inside a transaction that first acquires a transaction-scoped advisory lock
`pg_advisory_xact_lock(hashtext(userId), hashtext(skillId))` (bound parameters — no raw interpolation; a hash
collision only adds serialization, never corruption), then loads measurements **fresh under the lock**,
computes, and upserts. Lock scope is per (user, skill), so different skills never block each other.
`$executeRaw` (not `$queryRaw`) is used because the lock function returns `void`.

## 19. Diagnostic integration (§27)
Diagnostic derivation now persists `evidenceCount` + `observedAt` on the DIAGNOSTIC measurements, then calls
`recomputeSkills`. A diagnostic-only history reproduces the previous 1.5C state exactly (single DIAGNOSTIC
milestone → mastery = score, confidence, evidenceCount, lastMeasurementAt = attempt.completedAt) — verified by
the skill-profile regression suite. The diagnostic formula is unchanged.

## 20. Lesson integration (§28)
`LessonCompletion.finalize`: **A** authoritative completion → **B** ensure LESSON_MASTERY measurements (now with
evidenceCount + observedAt = completion time) → **B2** `recomputeSkills(affected)` → **C** roadmap reconcile.
Each step is idempotent; a later-step failure never rolls back A/B (recover via the repair endpoint / retry).
A lesson with no MASTERY_TEST measurement produces no affected skills → no merge → state unchanged (§56).

## 21. Checkpoint boundary (§29)
No Checkpoint assessment feature was built. The enum value exists, so a normalized CHECKPOINT measurement
(created directly in tests, or later by a real producer) is applied with full reset-anchor semantics by the
merge engine. No new checkpoint API.

## 22. Repair / recompute (§35)
`POST /api/learning-progress/me/subjects/:subjectId/recompute` (own-user; no body). Rebuilds every affected
current state (skills in the subject with supported evidence or an existing state) from existing measurements —
each skill in its own lock/transaction (no giant subject transaction). It **never** creates measurements.

## 23. Idempotency (§58)
Recompute is a pure projection of history: repeated calls yield identical state and create no measurement rows.
`recomputeSkill` upserts under the advisory lock, so replays and concurrent calls converge (§50).

## 24. Security (§71)
Recompute uses `principal.userId` only — a foreign call recomputes the caller's own (possibly empty) scope and
never touches another user's state. No body / no arbitrary userId. `subjectId` is `ParseUUIDPipe`-validated. No
raw answers / payloads / answer keys / scores are logged (the module logs nothing). Advisory-lock SQL is
parameterized.

## 25. Tests
- Unit: `merge/learning-progress-merge.engine.spec.ts` (13) — diagnostic-only parity, diagnostic+lesson,
  low-confidence baseline, multi-lesson, CHECKPOINT reset, latest-calibration, no-anchor, equal-timestamp
  CHECKPOINT precedence, unsupported-source exclusion, null-on-no-support, midpoint rounding, config-invalid /
  no-effective-evidence, equal-time lesson vs anchor.
- E2e: `test/learning-progress.e2e-spec.ts` (9) — diagnostic-only recompute, merge + CHECKPOINT reset through
  the DB, old-backfill no-regress, materialization recovery + idempotency, subject scope, no-supported-measurement
  preservation, concurrent same-skill convergence, different-skill independence, IDOR/401/400. Plus updated
  regressions: skill-profile diagnostic-through-merge (§53 rewritten to the projection-rebuild guarantee) and
  lesson-completion §47/48 (state now merges; skB untouched).
- Totals: unit **244**, e2e **161**.

## 26. merge-v2 is now the current engine (Phase 1.9C, TD-131)
`learning-progress-merge-v1` stays **frozen** (this document is its accepted historical contract). Phase 1.9C added
`learning-progress-merge-v2` as the CURRENT materialization engine and switched `LearningProgressService` +
repository to it. Both delegate to a shared pure core (`merge/merge-core.ts`); they differ ONLY by source config —
v2's incremental sources add **REVIEW_MASTERY** (anchors unchanged: DIAGNOSTIC/CHECKPOINT; REVIEW_MASTERY is never an
anchor). For any history with no REVIEW_MASTERY, v2 == v1 byte-for-byte. `LearnerSkillState` still carries **no**
`mergeVersion` column — it is a rebuildable projection; formula identity lives on `SkillMeasurement.derivationVersion`.
See `REVIEW_MASTERY_IMPLEMENTATION.md`.

## 27. Signals / rewards boundary (§72, STOP)
1.8A writes only `LearnerSkillState` (+ the schema metadata migration). It writes no Roadmap/RoadmapItem/
DailyPlan/LearnerSignal/RewardGrant/DailyMission/XP/IZL/AiEvaluation (grep + tests). `SkillMeasurement` is never
UPDATE/DELETE'd at runtime (append-only §61). Next phase (Signals) is **not** started.
