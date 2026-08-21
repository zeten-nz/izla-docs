# Learner Signals Foundation — Implementation (Phase 1.8B + 1.8C)

> Status: **COMPLETE** (1.8B repeated-mistake + 1.8C weak-skill/review-due, 2026-08-20). Three advisory
> `LearnerSignal` producers: `REPEATED_MISTAKE` (objective attempt evidence, 1.8B), `WEAK_SKILL` and
> `REVIEW_DUE` (current `LearnerSkillState` + `Clock`, 1.8C). Each has separate deterministic, versioned
> semantics; all three may coexist for a Skill. Signals are **advisory** — no mastery/roadmap/plan/reward/
> notification authority. **1.8C adds NO schema change** (reuses `evidenceRefs` JSONB + statuses + the one-active
> partial unique). Owner decisions: **TD-116 / TD-117 / TD-118** (1.8B), **TD-119 / TD-120 / TD-121** (1.8C).

Code: `backend/src/learner-signals/**` (pure `repeated-mistake.detector.ts`; `learner-signals.repository.ts`;
`learner-signals.service.ts`; `learner-signals.controller.ts`).

## 1. Scope
Detect repeated objective mistakes and manage the `REPEATED_MISTAKE` signal lifecycle (ACTIVE → RESOLVED,
recurrence → new episode). Read current ACTIVE signals; reconcile/repair from existing evidence. **NOT** in
scope: WEAK_SKILL, REVIEW_DUE, spaced repetition, EXPIRED auto-policy, rewards, notifications, AI, Roadmap/DailyPlan mutation.

## 2. Signal authority
`LearnerSignal` (existing model): `type` is a **String registry** (REPEATED_MISTAKE), `status` is the
`SignalStatus` enum (ACTIVE/RESOLVED/EXPIRED), `skillId` is a real column (no hidden JSON), `subjectId` NOT
NULL, timestamps `firstDetectedAt`/`lastSeenAt`/`resolvedAt`, `evidenceRefs` JSONB. No architecture gap.

## 3. Advisory boundary (TD-118)
A signal is a recommendation/fact. It does **not** become mastery, Roadmap, DailyPlan, completion, or reward
authority. Producers of state (LearningProgress) and planners (Roadmap/DailyPlan) are untouched.

## 4. repeated-mistake-signal-v1 (TD-116)
Immutable detector contract `repeated-mistake-signal-v1`. A rule change (trigger count, distinct-Activity
semantics, recovery rule, evidence source) ships as v2 — never a silent v1 change. The version is documented in
code + carried in `evidenceRefs.schemaVersion = "repeated-mistake-signal/v1"` (no new column added).

## 5. Eligible Activity evidence (§6)
Only lesson objective evidence: `ActivityAttempt.status = SUBMITTED` with `isCorrect` set, for objective types
MINI_QUESTION / PRACTICE / MASTERY_TEST. Excluded: AssessmentResponse (diagnostic is calibration), view-only,
LISTENING / WRITING / SPEAKING / AI_INTERACTION, AI evaluations.

## 6. Skill attribution (§7)
ActivitySkill authority; if an Activity has **zero** ActivitySkill rows, fall back to the logical Lesson's
LessonSkill. Never inferred from prompt/title/keywords/AI/type. A multi-skill Activity contributes to each
mapped Skill independently (no weights).

## 7. Latest-outcome-per-Activity collapse (§10)
For each (user, skill, Activity), all eligible attempts collapse to the **latest** SUBMITTED outcome (order:
`submittedAt DESC, attemptNo DESC, id DESC`). Retry spam therefore counts as one distinct Activity, not several
mistakes. Raw retries remain in `ActivityAttempt` history for future analytics.

## 8. Trigger = 3 distinct wrong (§12)
With no ACTIVE REPEATED_MISTAKE signal for (user, skill): activate iff the **3 most-recent distinct** Activity
outcomes are all incorrect (≥3 distinct required). No time window, no mastery threshold.

## 9. Recovery = 2 distinct correct (§13)
With an ACTIVE signal: resolve iff the **2 most-recent distinct** Activity outcomes are both correct. Not based
on `LearnerSkillState` (no weak-skill threshold is accepted) nor on lesson completion.

## 10. Lifecycle (§15/16)
Activation creates one ACTIVE episode. If the recovery condition is unmet, the episode stays ACTIVE unchanged
(no duplicate row, no createdAt rewrite, no evidence rewrite). RESOLVED/EXPIRED are terminal — never reactivated.

## 11. One-active invariant (TD-117, SG-01)
Partial unique `uq_learner_signal_active (user_id, skill_id, type) WHERE status = 'ACTIVE'` (custom SQL) — the
DB is the final concurrency authority. RESOLVED/EXPIRED history is unconstrained. Table was empty in dev+test,
so the constraint was added safely.

## 12. Recurrence episodes (§16)
After a RESOLVED episode, a fresh 3-distinct-wrong run creates a **new** ACTIVE row (the partial unique only
constrains ACTIVE, so a new episode never collides with the resolved one). The resolved row is unchanged —
signal history without a separate history table.

## 13. Concurrency (§25/26/31)
`evaluateSkill` runs inside a transaction that acquires a **signal-namespaced** advisory lock
`pg_advisory_xact_lock(hashtext('sig:'+userId), hashtext(skillId))` (distinct keyspace from the merge lock, so
signal eval and skill-state merge never block each other), then loads evidence **fresh under the lock**, runs
the pure detector, and applies. A concurrent create hits the partial unique (P2002 caught → already active).

## 14. ActivityAttempt integration (§27)
After a persisted `ActivityAttempt`, `LessonExecutionService` resolves the Activity's attributed skills and
calls `LearnerSignalsService.evaluateForActivity` — wrapped so a signal failure never rolls back the attempt.

## 15. Failure recovery (§28/29)
On an idempotent `clientRequestId` replay, signal evaluation runs again and stays idempotent (partial unique +
conditional resolve). A missed evaluation is repaired by the reconcile endpoint or a natural retry — the
`ActivityAttempt` is never mutated.

## 16. Reconcile endpoint (§30/31)
`POST /api/learner-signals/me/subjects/:subjectId/reconcile` (own-user, no body). Evaluates every relevant skill
in the subject — skills with eligible evidence ∪ skills with an ACTIVE signal — each under its own lock. It can
create ACTIVE or resolve ACTIVE; it never creates ActivityAttempt / SkillMeasurement / LearnerSkillState / Roadmap / DailyPlan.

## 17. Learner read API (§32)
`GET /api/learner-signals/me/subjects/:subjectId` returns only CURRENT ACTIVE signals `{ id, type, status,
skill: { id, name }, createdAt }`, ordered `firstDetectedAt DESC, skillId ASC, id ASC`. No history/filter API in v1.

## 18. Security (§64/65)
Both endpoints are `principal.userId`-scoped (a foreign call sees only the caller's own signals and never
touches another user's state); `subjectId` is UUID-validated; no auth → 401. Responses expose no
`ActivityAttempt.answer` / `Activity.payload` / answer keys / `evidenceRefs` (trigger ids stay internal). No
sensitive logging (the module logs nothing beyond a generic deferral warning in the caller).

## 19. WEAK_SKILL — implemented in Phase 1.8C
See Part 1.8C below (§25–33). Owner-accepted thresholds (TD-119); the merge→signal integration makes it
automatic after a state recompute.

## 20. REVIEW_DUE — implemented in Phase 1.8C
See Part 1.8C below (§25–33). Deterministic elapsed-time intervals (TD-120); time-passage eligibility is
evaluated via the reconcile command (no scheduler).

## 21. EXPIRED policy deferred (§5/21)
`SignalStatus.EXPIRED` support is preserved, but v1 defines **no** automatic expiration duration and performs
no ACTIVE → EXPIRED transition. `resolvedAt` is set on resolve; there is no `expiresAt` column.

## 22. Roadmap / DailyPlan boundary (§38/39)
Signal activation inserts no review Lesson, regenerates no Roadmap, reorders no RoadmapItem, and creates no
DailyPlan item. One-topic-per-day is intact. A future accepted planner may consume signals.

## 23. Rewards / notification boundary (§40/41/42/66)
No XP/IZL/RewardGrant/DailyMissionCompletion, no Notification rows, no AI. Verified: after activation/resolution
only `LearnerSignal` changes (e2e asserts LearnerSkillState/SkillMeasurement/Roadmap/DailyPlan/completion/reward/
notification/AI unchanged; grep confirms the module writes only `learnerSignal`).

## 24. Tests (1.8B)
- Unit: `repeated-mistake.detector.spec.ts` (10) — 2-wrong, 3-distinct-wrong, mixed, most-recent-3-only, active
  + one/two correct, active-vs-recent-wrong, ACTIVATE/RESOLVE gating, collapse (retries + latest-wins).
- E2e: `test/learner-signals.e2e-spec.ts` (11) — 3-distinct-wrong activates + read API + side-effect boundary +
  no raw-evidence leak; retries/latest/mixed no-trigger; recovery resolve + correct-retry stays active +
  recurrence new episode; different-skill independence; LessonSkill fallback; no-mapping no-signal; cross-subject
  blocked; unsupported-type excluded; network retry idempotent + concurrent trigger one signal; reconcile
  create + resolve; IDOR/401/400.

---

# Part 1.8C — WEAK_SKILL + REVIEW_DUE

## 25. Signal-type coexistence (§2/56/57)
`REPEATED_MISTAKE`, `WEAK_SKILL`, and `REVIEW_DUE` are independent producers; the one-active partial unique is
**per type**, so all three may be ACTIVE for the same Skill at once. No dedup across types. `REPEATED_MISTAKE`
behavior is unchanged.

## 26. weak-skill-signal-v1 (TD-119)
Immutable contract `weak-skill-signal-v1` (version in `evidenceRefs.schemaVersion`; a threshold change → v2).
**Input authority = current `LearnerSkillState` ONLY** (masteryScoreBp / confidenceBp / evidenceCount) — never
raw attempts/measurements (no duplicate pedagogy; those already feed the merge engine). Pure
`weak-skill-signal.policy.ts`.

## 27. Weak activation / resolution + hysteresis (§5/8/9/10)
- **Activate** (no active signal): `mastery < 5000` AND `confidence >= 7000` AND `evidenceCount >= 3` (all
  required; a null `confidenceBp` coerces to 0 and fails the gate). No CEFR meaning — operational thresholds.
- **Resolve** (active): `mastery >= 6500` AND `confidence >= 7000`.
- **Hysteresis band** `5000 ≤ mastery < 6499`: neither creates nor resolves. A confidence drop while active
  never resolves (only the confident-recovery condition does).
- **Recurrence**: after RESOLVED, a state again satisfying activation creates a NEW episode; the resolved row
  is untouched.
Activation snapshot `evidenceRefs = { schemaVersion:"weak-skill-signal/v1", masteryScoreBp, confidenceBp,
evidenceCount, lastMeasurementAt }` — never rewritten while active; no raw answers.

## 28. review-due-signal-v1 (TD-120)
Immutable contract `review-due-signal-v1`. **Authority = current `LearnerSkillState` + `Clock.now()`** — not
login / completion / roadmap / plan / REPEATED_MISTAKE. Pure `review-due-signal.policy.ts`.

## 29. Review intervals + elapsed-time semantics (§15/16/17)
Interval (confidence-first, then mastery bands): `confidence < 5000 → 1d`; else `mastery < 5000 → 1d`,
`5000..6999 → 3d`, `7000..8499 → 7d`, `>= 8500 → 14d`. `dueAt = lastMeasurementAt + intervalDays × 24h` — exact
elapsed duration from the logical evidence timestamp, never a server/local calendar date (DST/timezone-safe).
Daily presentation/scheduling stays a separate future concern.

## 30. Review activation / resolution / recurrence (§18/20/21/22/50/52/53)
- **Activate** (no active signal): requires `state` with `evidenceCount > 0` and non-null `lastMeasurementAt`,
  and `now >= dueAt` (exact due activates). Snapshot `evidenceRefs = { schemaVersion:"review-due-signal/v1",
  basisLastMeasurementAt, masteryScoreBp, confidenceBp, evidenceCount, intervalDays, dueAt }`.
- **Resolve** (active): current `lastMeasurementAt` is **strictly after** `basisLastMeasurementAt` (new accepted
  evidence arrived) — mastery need not improve; same logical timestamp does not resolve.
- After resolving, a new episode activates only if the fresh state is already due again now (normally future).
- **Recurrence**: a later reached `dueAt` opens a new episode; the resolved row remains.

## 31. Clock + time-based reconcile (§26/32/33)
Uses the shared injectable `Clock` (DailyPlan boundary), so review timing is testable with a fixed clock — no
scattered `new Date()` in policy. REVIEW_DUE can become eligible purely from time passing, so the **reconcile
endpoint** (which evaluates via `Clock.now()`) is the deterministic time-based evaluation command. **No cron /
queue / scheduler** in this phase. `GET` stays strictly read-only — it never activates a due review.

## 32. Learning Progress integration + failure boundary (§29/30/31/62/63)
After `LearningProgressService` recomputes a Skill state (single writer, TD-115), it calls
`LearnerSignalsService.evaluateStateSignals(userId, skillId)` **outside** the merge transaction. The pure merge
engine stays signal-unaware; module dependency is one-way (`LearningProgress → LearnerSignals`, no cycle). A
signal-evaluation failure is caught and logged — it **never** rolls back the authoritative state (§31); the
reconcile command repairs it. Diagnostics and lesson completions therefore produce/resolve WEAK_SKILL and
REVIEW_DUE automatically.

## 33. Concurrency + reconcile skill set (§28/58/59/60)
State-signal evaluation runs under the same signal-namespaced per-(user,skill) advisory lock, reading state
fresh; the one-active partial unique (P2002-catch) makes concurrent activation converge to one ACTIVE per type.
Subject reconcile evaluates REPEATED_MISTAKE candidates (eligible evidence ∪ active) and state-signal candidates
(skills with a `LearnerSkillState` ∪ skills with an active WEAK_SKILL/REVIEW_DUE), each deterministically ordered.

## 34. EXPIRED still deferred (§5/24)
`SignalStatus.EXPIRED` remains available; 1.8C defines no auto-expiration and performs no ACTIVE→EXPIRED
transition. `resolvedAt` is set on resolve.

## 35. Boundaries (§64–68) & evidence governance (§69)
WEAK_SKILL/REVIEW_DUE activation/resolution never touches Roadmap/RoadmapItem, DailyPlan/DailyPlanItem, rewards
(XP/IZL/RewardGrant), Notification, or AI, and never mutates SkillMeasurement/LearnerSkillState (verified by grep
+ e2e). Strict `evidenceRefs` parsers enforce `schemaVersion` and bounded numeric/timestamp/interval fields; the
read API never returns `evidenceRefs`.

## 36. Future v2 policy
Threshold/interval changes ship as `weak-skill-signal-v2` / `review-due-signal-v2` (immutable v1). Methodist
calibration, statistical mastery, forgetting curve, exponential spacing (Leitner/SM-2/FSRS), review content
selection, DailyPlan/Roadmap consumption, dismiss/snooze, expiration TTL, notifications, rewards, and AI remain OPEN.

## 37. Tests (1.8C)
- Unit: `weak-skill-signal.policy.spec.ts` (9) — activation, confidence/evidence gates, exact threshold,
  hysteresis hold band, resolution, resolve-confidence, confidence-drop-holds, null-state. `review-due-signal.policy.spec.ts`
  (13) — interval bands (1/3/7/14), activation not-due/exact-due, no-state/null-time/zero-evidence, resolution
  strictly-after / same-timestamp-holds, strict basis parser.
- E2e: `test/learner-signals-policy.e2e-spec.ts` (11) — weak activate→resolve(hysteresis)→recurrence; weak gates;
  review exact-due; interval bands; review no-state/newer-resolves/same-timestamp/recurrence; weak+review
  coexist; concurrent one-per-type; reconcile-creates + GET read-only; automatic weak on merge + checkpoint-reset
  resolve; automatic review resolution on newer evidence; read API + IDOR + no-leak + side-effect boundary.
- Totals: unit **276**, e2e **183**.
