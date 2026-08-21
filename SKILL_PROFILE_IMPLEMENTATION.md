# Skill Profile Derivation — Implementation (Phase 1.5C)

> Status: **COMPLETE** (1.5C + 1.5C-2 contract closure, 2026-08-19). Converts completed INITIAL_DIAGNOSTIC
> evidence into per-Skill `SkillMeasurement` (append-only milestone) + `LearnerSkillState` (current derived
> state). Migrations: `20260819170000_add_skill_measurement_derivation_version` (column + idempotency
> partial-unique) and `20260819180000_harden_skill_measurement_derivation_version` (derivation_version
> NOT NULL + non-empty CHECK). No Roadmap / Signal / XP / IZL / AI.

Code: `backend/src/skill-profile/**` + orchestration `assessment/placement-flow.service.ts`.

## 1. Scope
A completed placement diagnostic deterministically produces, per measured Skill: `masteryScoreBp`,
`confidenceBp`, `evidenceCount`, `displayLevel = null`; persists one append-only diagnostic
`SkillMeasurement` and the current `LearnerSkillState`. Learner can read their current Skill Profile and
the snapshot produced by an exact attempt. No Roadmap.

## 2. Evidence/state model (TD-31/TD-35)
Raw evidence = `AssessmentResponse` (immutable). Milestone evidence = `SkillMeasurement` (append-only).
Current derived state = `LearnerSkillState` (mutable). Changing formulas later must NOT rewrite/delete
historical `SkillMeasurement`; raw evidence stays available for recomputation.

## 3. Diagnostic derivation v1 (TD-97)
`skill-profile-diagnostic-v1` (`SKILL_PROFILE_DIAGNOSTIC_VERSION`) — accepted **implementation** derivation
contract, NOT CEFR/IRT/final truth. Pure component `DiagnosticSkillProfileEngine` (no DB/HTTP/Prisma/AI),
reproducible from immutable inputs `(pinned config, exact pool, ordered submitted responses)`.

## 4. profileScale (TD-97, version-pinned)
`placement-adaptive/v1` config gains `profileScale: { minDifficulty, maxDifficulty }` (`max > min`,
`startDifficulty` within range; every effective item difficulty within range — enforced at start & derivation).
It is a **stable normalization scale**, version-pinned & immutable — so the same difficulty rank maps to the
same mastery regardless of a later version's pool width. NOT CEFR. Values are methodist/content-owned.

## 5. Mastery formula — FROZEN v1 contract (TD-97, owner-ACCEPTED; Phase 1.5C-2)
The accepted placement-adaptive-v1 SELECTION target is a net-correctness walk (difficulty-**insensitive**,
locked by 1.5B §32 tests), so mastery is NOT the engine target and NOT raw percent-correct. Owner reviewed
and ACCEPTED the response-difficulty estimator: it preserves actual item difficulty + correctness, which the
final adaptive target alone cannot. This is now the permanent meaning of `skill-profile-diagnostic-v1`:
```
per response with effective item difficulty d:
  correct   → e = d
  incorrect → e = d − 1        # the PREVIOUS ordinal evidence band (§4)
  e = clamp(e, profileScale.minDifficulty, profileScale.maxDifficulty)
estimatedDifficulty(skill) = arithmetic mean of the skill's per-response estimates e
masteryScoreBp = round((estimatedDifficulty − minDifficulty) / (maxDifficulty − minDifficulty) × 10000)  # clamp 0..10000
```
A learner who succeeds at harder items scores higher than one who only succeeds at easier items, **even at
equal accuracy** (proven by test §45). Single rounding at the end; integers only (TD-89/§42).

**The `−1` is the previous ordinal difficulty rank, NOT `config.selection.stepDown`** (§4): `stepDown`
controls next-item targeting during the assessment; the derivation's `d − 1` is the evidence band for a wrong
answer. Distinct concerns — never substitute one for the other.

**Immutability (§5):** correct/incorrect transforms, mean aggregation, normalization, and confidence are
frozen under `skill-profile-diagnostic-v1`. A future mathematical change ⇒ `skill-profile-diagnostic-v2`;
historical v1 rows stay untouched.

## 6. Confidence formula (coverage, §12)
Confidence = configured evidence coverage, **not** statistical certainty:
`confidenceBp = round(min(1, evidenceCount / coverage.itemsPerSkill) × 10000)` (0..10000). e.g.
itemsPerSkill=4 → 1/2/3/4+ evidence = 2500/5000/7500/10000.

## 7. evidenceCount (§13)
Number of valid SUBMITTED objective responses for that Skill in this diagnostic attempt. Excludes
PRESENTED-only/SKIPPED/other-skill/other-user/other-subject/other-attempt evidence.

## 8. displayLevel = null (§5/43)
CEFR/level thresholds remain OPEN, so new diagnostic measurements/state carry `displayLevel = null`. No
A1/B1/BEGINNER strings are invented. The API returns `"displayLevel": null` — expected. Later methodist
proficiency mapping populates/recomputes labels.

## 9. CEFR boundary
No CEFR hard-code, no `mastery 6000 = B1`, no Level FK. `displayLevel` is a derived presentation value only;
authoritative math is `masteryScoreBp` / `confidenceBp` / `evidenceCount` (TD-35).

## 10. Reproducibility (§11/24)
Derivation is a pure deterministic fold over immutable evidence — `AssessmentResponse` sequence is authority,
NOT `engineState` (resume cache) nor `resultSummary` (display cache). Same attempt → identical measurements.

## 11. SkillMeasurement — append-only (§16/17/18/19)
One milestone per Skill per derivation (not per question). `source = DIAGNOSTIC`,
`assessmentAttemptId = exact attempt`, `scoreBp = masteryScoreBp`, `confidenceBp`, `displayLevel = null`,
`derivationVersion = skill-profile-diagnostic-v1`. Idempotency (DB): partial-unique
`(assessment_attempt_id, skill_id, source, derivation_version) WHERE assessment_attempt_id IS NOT NULL`
(SP-04). `derivationVersion` participates so a future `-v2` / `ENGINE_RECALC` can append a new historical row
without collision (§19). Writes use `createMany({ skipDuplicates })` (ON CONFLICT DO NOTHING — no tx abort).

## 12. LearnerSkillState — current, chronology-guarded (§14/15/34/53)
Current state mirrors the newest applicable measurement: `masteryScoreBp`, `confidenceBp`, `evidenceCount`,
`displayLevel = null`, `lastMeasurementAt`. `@@unique(userId, skillId)` (SP-01). The measurement's logical
time is `AssessmentAttempt.completedAt` (observation time, NOT job time). An older backfill NEVER regresses a
newer state — a guarded `updateMany (lastMeasurementAt ≤ completedAt OR null)` + `createMany skipDuplicates`
keeps it concurrency-safe and monotonic (SP-08; tested §53).

## 13. derivationVersion — mandatory identity (§17/27; TD-98, Phase 1.5C-2)
Dedicated `SkillMeasurement.derivation_version` column (not encoded in displayLevel/feedback/JSON/source).
**NOT NULL + non-empty CHECK** (`length(trim(derivation_version)) > 0`, SP-09) — every measurement must
identify the formula/process that produced it. **No column default** — the caller supplies it explicitly
(future sources conceptually `lesson-mastery-v1`, `checkpoint-profile-v1`, `engine-recalc-v2`; those exact
names are NOT accepted now — only the non-null-identity invariant is). Constant `skill-profile-diagnostic-v1`
for this phase; changing the math ⇒ new constant; v1 behavior never silently changes.

## 14. Idempotency (§18/29/35/52)
Repeated derivation of an attempt never duplicates measurements (DB idempotency index) and yields a
deterministic current state. Concurrent derivations both succeed (tested §52).

## 15. Chronology / backfill (§15/37)
`lastMeasurementAt = completedAt`. Multiple completed diagnostics may exist (retake policy OPEN); the newest
by completion time wins current state. Old backfills append history but don't regress current state.

## 16. Incomplete coverage (§23/48)
Coverage may be incomplete (a skill with too few pooled items). Derivation still runs: mastery from available
evidence, confidence reduced by evidence count. The whole profile is not failed for partial coverage.

## 17. Subject scope (L-3, §21/54)
Before writing, every answered item's Skill must belong to `AssessmentAttempt.subjectId`. Cross-subject
evidence fails safely (`ASSESSMENT_CONFIGURATION_INVALID`, no writes; tested §54).

## 18. APIs (§30/31/32/39/40)
All authenticated, own-user only (IDOR-safe), no client-supplied scores, no PATCH.

| Method | Path | Notes |
|---|---|---|
| GET  | `/api/skill-profile/me/subjects/:subjectId` | current `LearnerSkillState` for principal + subject |
| GET  | `/api/skill-profile/diagnostics/:attemptId` | milestone snapshot; 409 `SKILL_PROFILE_NOT_DERIVED` if absent |
| POST | `/api/skill-profile/diagnostics/:attemptId/derive` | idempotent repair/backfill (200) |

Deterministic Skill order: `sortOrder`, then `name`, then `id` (§33). Errors: `SKILL_PROFILE_NOT_DERIVED`
(409), `ASSESSMENT_ATTEMPT_NOT_FOUND` (404), `ASSESSMENT_CONFIGURATION_INVALID` (409), `RESOURCE_NOT_FOUND` (404).

## 19. Assessment completion integration (§28/29)
`AssessmentService` stays evidence-only (no skill-state writes — grep-verified). `PlacementFlowService`
(assessment module → one-way import of `SkillProfileModule`) wraps submit: on `COMPLETED`, calls
`SkillProfileService.ensureDiagnosticDerived` in a **separate transaction**. If derivation fails, the
completed attempt is NOT rolled back — it is logged (safe id) and recovered via idempotent retry / the repair
endpoint. No circular Nest dependency (skill-profile reuses assessment's pure config parser + types by file
import only, and reads assessment tables via its own repository).

## 20. Tests (unit 18, e2e 8; project total unit 151 / e2e 90)
- **Unit:** FROZEN v1 transform §3/§13 (correct→d, incorrect→d−1 ordinal, min-clamp, mean); normalization §44
  (min→0/max→10000/clamp/integer); difficulty-matters §45 (same accuracy → different mastery); per-skill
  independence §46; confidence §47; incomplete coverage §48; zero evidence §49; determinism + out-of-pool ignore.
- **DB integrity §15:** derivation_version NULL rejected (NOT NULL), '' / whitespace rejected (CHECK), real value accepted.
- **E2e:** auto-derivation §56 (+ exact attempt link §50 + difficulty-sensitive mastery + side-effects absent);
  profile/snapshot APIs + IDOR + no-leak §55; append-only §51; concurrent derive §52; old-backfill no-regress
  §53; subject-scope fail-safe §54; snapshot-before-derive → 409.
- Regression: all prior phases green (assessment version pinning, pool membership, answer-key protection,
  concurrency, diagnostic uniqueness, skill-balanced adaptive).

## 21. Deferred universal progress engine (§36/63)
This phase materializes ONLY diagnostic evidence. No weighted merge of diagnostic + lesson-mastery +
checkpoint + AI + review. Current state reflects this milestone if it is the newest applicable measurement.

## 22. Roadmap boundary (§64/65/66)
Output is the INPUT for a later Roadmap phase. No `LearnerRoadmap`/`RoadmapItem`/gap detection/recommendation,
no `LearnerSignal` (low mastery is normal state, not a signal — TD-40), no XP/IZL. Verified absent in tests.

## Remaining OPEN — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)
CEFR/proficiency mapping · mastery thresholds · universal progress-update formula · checkpoint/current-state
merge · reassessment merge policy · advanced/statistical confidence · IRT · writing/speaking · AI provider.
