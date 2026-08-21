# Daily Mission Foundation v1 — Implementation (Phase 2.0B)

> Status: **COMPLETE** (2026-08-20). A deterministic Daily Mission policy foundation: **append-only,
> evidence-backed** mission completion at the **learner-local day**, storing immutable timezone/date/policy
> provenance, idempotent under retries/concurrency, readable and reconcilable. **No RewardGrant/XP/IZL is
> created** — mission completion ≠ reward (the reward bridge is Phase 2.0C). Exactly **two** v1 missions:
> **LEARN_TODAY** and **MASTERY_TEST_90**, both **account-level** (not per-Subject, not DailyPlan items).
> Owner: **TD-136/137/138/139**. No skill/signal/plan/roadmap/review-session/notification/AI writes.

Code: `backend/src/daily-mission/` — `mission/daily-mission.policy.ts` (pure), `daily-mission.repository.ts`,
`daily-mission.service.ts`, `daily-mission.controller.ts`, `daily-mission.module.ts`. Hooks in
`lesson-execution.service.ts` + `review-session.service.ts`. Migration
`20260820170000_harden_daily_mission_completion`.

## 1. Scope
A deterministic policy layer that materializes **mission completions** from already-persisted **objective
ActivityAttempt** evidence, bucketed by the learner's local calendar day. IN scope: mission code registry, two
account-level missions, append-only completion + typed evidence, automatic post-attempt evaluation, read
projection, reconcile repair, idempotency, timezone/date/policy provenance. OUT of scope (deferred): rewards
(XP/IZL/RewardGrant), the missions ATTENTION_CHECK / LIBRARY_15_MIN / COMMUNITY_SHARE, special/streak missions,
per-Subject missions, notifications, weekly/monthly aggregation, AI-driven missions.

## 2. The two v1 missions (TD-139)
| Code | Policy version | Qualifies when | Correctness |
|---|---|---|---|
| **LEARN_TODAY** | `learn-today-mission-v1` | ≥1 SUBMITTED objective ActivityAttempt (lesson OR review) in the local day | irrelevant — a wrong attempt still counts |
| **MASTERY_TEST_90** | `mastery-test-90-mission-v1` | a SUBMITTED `MASTERY_TEST` attempt with `deterministicScore >= 9000` (bp) | correctness threshold at 90% |

Both are **account-level** — one completion row per (user, mission, local day), NOT one per Subject, NOT tied to
a DailyPlanItem. `daily_plan_item_id` stays NULL for v1 (the column/relation is reserved).

## 3. Objective evidence definition (TD-139 §7)
Objective ActivityTypes = **MINI_QUESTION, PRACTICE, MASTERY_TEST** (`OBJECTIVE_TYPES`). These are the only types
that produce a scored `ActivityAttempt`. **Placement/diagnostic** is excluded by construction — it writes
`AssessmentResponse`, never an `ActivityAttempt`. **View-only** content (TEXT/VIDEO/LISTENING/WRITING and any
non-scored activity) is excluded because it never creates an attempt. No text/AI inference is used.

## 4. Pure policy (`daily-mission.policy.ts`)
`qualifiesLearnToday(e)` = `status===SUBMITTED && OBJECTIVE_TYPES.has(type)`. `qualifiesMasteryTest90(e)` =
`status===SUBMITTED && type===MASTERY_TEST && (score ?? -1) >= 9000`. Both operate on a normalized
`MissionEvidence` (attemptId, activityType, status, deterministicScoreBp, submittedAt, reviewSessionId) — **no raw
answer/payload**. `MISSION_CATALOG` is the server-side registry of `{code, version, qualifies}` — there is no rule
DSL and deferred missions are simply absent. Producer versions are immutable; a rule change ships as v2.

## 5. Learner-local mission day (TD-137 §17/19)
`local_date` is derived from the **qualifying attempt's `submittedAt`** converted to the learner's
`UserProfile.timezone` (IANA) via `localDateInTimezone` + `toDateOnly` — the **same timezone authority as
DailyPlan** (TD-91). The day is the learner's local calendar date, not UTC. `getToday`/`reconcile` use the
current local date from `Clock.now()` in the same timezone. Timezone conversion is DST/offset-safe (elapsed
duration, never naive calendar math).

## 6. Timezone snapshot immutability (TD-137 §20)
Every completion row freezes `timezone_snapshot` (the IANA tz used to derive `local_date`). If the learner later
changes their profile timezone, **historical completions are not recomputed or moved** — their `local_date` and
`timezone_snapshot` are permanent provenance. New evaluations use the then-current timezone. (e2e §61 asserts a
profile tz change does not move an existing completion.)

## 7. completedAt semantics (TD-137 §29)
`completed_at` = the **qualifying attempt's `submittedAt`**, not wall-clock insert time. For MASTERY_TEST_90 the
qualifying attempt is the first one that reaches ≥9000, so `completedAt` reflects when mastery was actually
achieved. For LEARN_TODAY it is the earliest objective attempt of the day.

## 8. First / earliest evidence wins (TD-137 §41)
When multiple attempts qualify on the same day, the **earliest** one wins (`submittedAt ASC, id ASC`). The
automatic hook writes the first qualifying attempt it sees; the partial-unique index rejects any later duplicate.
Reconcile explicitly scans the day's attempts in `submittedAt ASC, id ASC` order and takes the first match per
mission, so both paths converge on the same earliest evidence.

## 9. Idempotency & concurrency (DM-DB-01, TD-137)
A partial UNIQUE index `uq_daily_mission_completion_day (user_id, mission_code, local_date) WHERE local_date IS
NOT NULL` is the single idempotency/concurrency authority. `createCompletion` inserts the completion + its evidence
in one transaction; on a `P2002` unique violation it returns `false` (the first completion already won). Retries,
concurrent submits, and repeated reconciles therefore all converge to exactly one row per (user, mission, day).

## 10. Append-only (DM-DB-06)
The module only ever `create`s `daily_mission_completion` and `daily_mission_completion_evidence`. There is **no
UPDATE and no DELETE** anywhere in the module (verified by grep). Reconcile never mutates an existing row — it only
inserts a missing one. Completions are immutable historical facts.

## 11. Automatic evaluation hook (TD-138 §22/23)
After an objective `ActivityAttempt` is persisted, `evaluateActivityAttempt(userId, attemptId)` runs as a
**downstream advisory hook**. It is wired into BOTH submit paths:
- `lesson-execution.service.ts` `submitActivityAttempt` — after signal evaluation.
- `review-session.service.ts` `submitAttempt` — wraps `persistReviewAttempt`, then evaluates.

Both call sites wrap the hook in `try/catch` and only `logger.warn` on failure — **a mission failure never rolls
back the authoritative ActivityAttempt** (§22). A deferred/failed evaluation is repaired later by reconcile.

## 12. Evaluate flow (`evaluateActivityAttempt`)
1. `repo.attemptEvidence(userId, attemptId)` — loads the attempt only if it is the caller's own, SUBMITTED, and
   objective; otherwise returns null → no-op.
2. `repo.profileTimezone(userId)` — if missing, `logger.warn` and **defer** (§21), no throw (the attempt already
   persisted).
3. `localDate = toDateOnly(localDateInTimezone(submittedAt, tz))`.
4. For each mission in `MISSION_CATALOG`, if `qualifies(evidence)` → `createCompletion(...)` with
   `completedAt = evidence.submittedAt` and `activityAttemptId = evidence.attemptId`.

One 10000 MASTERY_TEST attempt therefore satisfies BOTH missions and writes two rows in a single evaluation.

## 13. Read model — `GET /api/daily-missions/me/today` (TD-138 §38)
Returns `{ localDate, timezone, missions: [{ code, completed, completedAt, policyVersion }] }` for the learner's
**current** local day. It is strictly **read-only** — it never writes. `policyVersion` is the completion's stored
version when completed, else the catalog's current version. Requires a resolved profile timezone (else 409). (e2e
§55 asserts two identical GETs write nothing.)

## 14. Reconcile — `POST /api/daily-missions/me/today/reconcile` (TD-138 §39)
Repairs missing current-day completions from existing evidence: it loads objective attempts within a generous
48h UTC window (`RECONCILE_WINDOW_MS`), filters to the exact current learner-local day (`localDateInTimezone ==
localDateStr`, DST-safe), then for each mission inserts the earliest qualifying attempt (idempotent via DM-DB-01).
Returns the same shape as `getToday`. Reconcile is safe to call repeatedly (§9). It repairs completions that the
automatic hook deferred (e.g., a transient failure, or evidence seeded outside the submit path).

## 15. Why a 48h window + in-memory local-day filter (§40)
UTC-interval math for a learner-local day is fragile across DST transitions and offsets. Instead the repository
bounds the query cheaply by `submittedAt >= now-48h` (covering the current local day for any real-world offset),
and the service filters each attempt by its exact `localDateInTimezone`. This is correct for all offsets/DST and
keeps the query simple and index-friendly.

## 16. Repository boundary (`daily-mission.repository.ts`)
READS: `ActivityAttempt` evidence (own, SUBMITTED, objective; via `attemptEvidence` / `eligibleAttempts`) and
`UserProfile.timezone`. WRITES: only `dailyMissionCompletion` + `dailyMissionCompletionEvidence`, both `create`,
in a transaction. `isUniqueViolation` centralizes the P2002 check. The repository never touches
reward/skill/signal/plan/roadmap/session tables.

## 17. Evidence linkage (DM-DB-05)
`DailyMissionCompletionEvidence` is the polymorphic evidence row (XOR of typed FKs — L-26). Mission completions
set `activity_attempt_id` (FK `onDelete: Restrict`). Restrict is intentional: an attempt that is a mission's
evidence cannot be silently deleted out from under the completion. Consequence for tests: any reset that deletes
`activity_attempt` must first delete the two mission tables (applied to all affected e2e resets — see §22).

## 18. Schema change (migration 11)
`DailyMissionCompletion` was **hardened in place** (rather than adding a new table) — the phase mandate permits the
mission foundation to change the schema. Added `mission_code`, `policy_version`, `local_date` (`@db.Date`),
`timezone_snapshot` (all NOT NULL); changed `daily_plan_item_id` to nullable `@unique`; added
`@@index(user_id, local_date)`. Migration `20260820170000_harden_daily_mission_completion` = Prisma DDL + custom
SQL: partial UNIQUE `uq_daily_mission_completion_day` (DM-DB-01) + 3 nonempty CHECKs (DM-DB-02/03/04). Migration
count → **11**; drift clean on `izlan_dev` + `izlan_test`; re-running `migrate diff` yields an empty migration.

## 19. DB invariants
See [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) §9h. Summary: DM-DB-01 (partial unique, idempotency),
DM-DB-02/03/04 (mission_code / policy_version / timezone_snapshot `length(trim)>0` CHECKs), DM-DB-05 (evidence →
attempt Restrict FK), DM-DB-06 (append-only). App-level DM-01..DM-10 cover the policy and boundary. **Silently
dropped: 0.**

## 20. Error handling
`DailyMissionConfigurationInvalidError` → HTTP **409 `DAILY_MISSION_CONFIGURATION_INVALID`** (mapped in
`auth-exception.filter.ts`). Thrown by `getToday`/`reconcile` when the learner has no resolved profile timezone.
The automatic hook never throws to the caller — it defers (logs) instead, so a missing timezone can never roll
back an attempt.

## 21. Security & privacy (TD-138 §68-78)
Own-user only: the controller resolves the user from `CurrentPrincipal`; there is no cross-user path, so no IDOR
(e2e §65). Unauthenticated `GET` → 401 (global AuthGuard). No raw `answer`/`answerKey`/`payload` ever enters
mission evidence or the response — `MissionEvidence` carries only ids/type/status/score/time. The single
`logger.warn` carries an attempt UUID only, never sensitive data.

## 22. Test-DB reset ordering (DM-DB-05 consequence)
Because `daily_mission_completion_evidence.activity_attempt_id` is `Restrict`, every e2e suite that deletes
`activityAttempt` must first delete `dailyMissionCompletionEvidence` then `dailyMissionCompletion`. This was added
to the resets of `lesson-execution`, `lesson-completion`, `review-session`, `review-mastery`, `learner-signals`,
`daily-plan-review`, and the new `daily-mission` spec. (This is the standard FK-ordered-reset discipline used
across phases.)

## 23. Tests
- **Unit** (`mission/daily-mission.policy.spec.ts`, 8): LEARN_TODAY correctness-irrelevant / review / non-objective
  / non-SUBMITTED; MASTERY_TEST_90 8999/9000/10000 boundary, only-MASTERY_TEST, review, null-score/non-SUBMITTED.
- **e2e** (`test/daily-mission.e2e-spec.ts`, 11, §50-73): normal wrong PRACTICE → LEARN_TODAY only + dedup to one;
  normal correct MASTERY_TEST → both; review-linked attempt (reviewSessionId set) → both; no-attempt → both
  incomplete + GET read-only; 8999 vs 9000 boundary; reconcile earliest (0 then 10000 → completedAt at the
  qualifying attempt); network retry (same clientRequestId) → one attempt, unchanged; day boundary (23:59 vs
  00:01 local via Clock override) → two separate completions; timezone snapshot immutable under profile tz change;
  reconcile repair + idempotency; IDOR / 401 / no-leak / side-effect boundary (reward/skill/measure/signal/plan/
  roadmap/session/notification counts unchanged, `aiEvaluation` zero).
- Full gate: **323 unit + 237 e2e PASS**; `tsc` clean; `prisma validate`/drift clean.

## 24. Boundary (grep + test)
The mission module writes **only** `daily_mission_completion` + `daily_mission_completion_evidence` (both create,
append-only). It does **not** write RewardGrant / XP / IZL / LearnerSkillState / SkillMeasurement / LearnerSignal /
DailyPlan / Roadmap / LearnerReviewSession / Notification / AiEvaluation (verified by grep over `src/daily-mission`
and by e2e before/after row counts). The `RewardGrant[] @relation("RewardGrantMission")` on the completion model
is the **IZL** reward relation, reserved for the future Phase 2.1A and untouched here.

**Update (Phase 2.0C-2 / 2.0D / 2.1A):** the mission producer bridges rewards downstream via
`DailyMissionService.bridgeMissionRewards` after completions are created — two **independent** branches, neither
gating the other, a failure in either never rolling back the completion:
- **XP** (`XpService.tryEnsureMissionXpGranted` + projection refresh) → `XpGrant`/`XpBalance` via the XP module —
  see [XP_REWARD_IMPLEMENTATION.md](XP_REWARD_IMPLEMENTATION.md) / [XP_PROGRESSION_IMPLEMENTATION.md](XP_PROGRESSION_IMPLEMENTATION.md).
- **IZL** (`DailyMissionIzlService.tryEnsureMissionReward`) → `RewardGrant` + `IZLLedgerEntry` (atomic, cycle-scoped,
  capped) via the Finance module — see [IZL_REWARD_IMPLEMENTATION.md](IZL_REWARD_IMPLEMENTATION.md). Uses the IZL
  `DailyMissionCompletion.rewardGrants @relation("RewardGrantMission")` relation.
