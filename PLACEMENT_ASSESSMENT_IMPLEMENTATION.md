# Placement (Initial Diagnostic) Assessment — Implementation (Phase 1.5B / 1.5B-2)

> Status: **COMPLETE** (2026-08-19). Phase 1.5B built the objective, versioned, reproducible foundation;
> Phase 1.5B-2 (owner review closed) hardened it into an accepted, skill-balanced, objective-only engine
> with DB-level integrity. Schema change this phase: **one migration**
> (`20260819160000_harden_initial_diagnostic_constraints`, 2 partial-unique indexes) — old migrations untouched.

Code: `backend/src/assessment/**`. Boundary (§40): **assessment evidence only** — no LearnerSkillState /
SkillMeasurement / Roadmap / DailyPlan / XP / IZL / AI-provider. Skill Profile phase (1.5C) derives mastery.

## 0. Terminology mapping (OD-1, ACCEPTED)
"Placement" is the **initial diagnostic**. `AssessmentDefinition.purposeScope = DIAGNOSTIC`;
`AssessmentAttempt.purpose = INITIAL_DIAGNOSTIC`; (future) `SkillMeasurement.source = DIAGNOSTIC`.
No `PLACEMENT` enum exists — this is the accepted reading of the schema enums.

## 1. Scope
Authenticated learner: check availability, start (or resume) a placement attempt, receive server-selected
items, submit answers, have objective items scored by the backend, complete into immutable evidence.
Methodist authoring, AI evaluation, skill mastery, and roadmap are OUT (§40).

## 2. LearningIntent entry (§5/7)
`POST /placement/start {learningIntentId}` uses the learner's OWN intent (loaded by `(id, principal.userId)`).
Requires `trackId` + Subject/Track PUBLISHED + `track.subjectId==subjectId`. Never an arbitrary userId,
never `SubjectAssignment`. `AssessmentAttempt.trackId` records the diagnostic's track context; the
definition is **Subject-scoped** — Track is context, not a definition-selection dimension (OD/§6).

## 3. Definition resolution + subject-level uniqueness (§3/6/8/38/39; TD-94)
Subject-scoped: `(subjectId, purposeScope=DIAGNOSTIC, status=PUBLISHED)`. **At most one such definition per
subject** is now enforced by a partial-unique index (`uq_def_published_diagnostic_per_subject`, PA-07). The
runtime keeps its fail-safe (0 → `ASSESSMENT_NOT_AVAILABLE`; >1 → `ASSESSMENT_CONFIGURATION_INVALID`) as
defense-in-depth. `currentVersionId` must resolve to a PUBLISHED version of the same definition. The client
never sends `definitionVersionId`.

## 4. Version pinning (§9)
Attempt pins `definitionVersionId`, immutable. Resume/submit read the pinned version **by id, no status
filter** — a version ARCHIVED after start stays authoritative. Only NEW attempts apply the PUBLISHED filter.

## 5. Item pool membership (§10/40/51)
Engine selects **only** from `AssessmentVersionItem` for the pinned version — no global lookup, not re-filtered
by current `item.status` (the published version froze the pool). Out-of-pool items are unreachable (tested).

## 6. Adaptive engine — per-skill, skill-balanced (§10/15/16/17/18; TD-96)
`PlacementEngineService` is **pure/deterministic**. Each Skill in the pinned pool carries its OWN state
`{targetDifficulty, answeredCount}` — one Skill's answer never moves another's target (§10, tested). Selection
(§17): exclude presented items → candidate skills = those with an unseen item AND `answeredCount < itemsPerSkill`
→ pick the skill with the **lowest answeredCount** (tie-break skillId ascending) → within it, the unseen item
nearest **that skill's** target (tie-break |distance|, lower difficulty, itemId). No `Math.random`/wall-clock.
Apply result: correct → skill target +`stepUp`, incorrect → `max(1, −stepDown)`; only that skill changes.

## 7. Engine version (§7/20)
`PLACEMENT_ENGINE_VERSION = 'placement-adaptive-v1'`, persisted on `AssessmentAttempt.engineVersion`.
Algorithm change later ⇒ new version string; historical attempts stay reproducible. Never overwritten.

## 8. Config contract + feasibility (§8/12/19/59; TD-96)
`AssessmentDefinitionVersion.config` validated by `parsePlacementConfig` (never blind-cast). Shape:
```jsonc
{ "schemaVersion": "placement-adaptive/v1", "engine": "placement-adaptive-v1",
  "selection": { "startDifficulty": int≥1, "stepUp": int≥0, "stepDown": int≥0 },
  "coverage": { "itemsPerSkill": int≥1 },
  "stopping": { "maxItems": int≥1 } }
```
`stopping.minItems` was **removed** (dead field). Malformed config → `ASSESSMENT_CONFIGURATION_INVALID`
(safe message). **Feasibility (§19/34):** at start, `distinctSkills × itemsPerSkill > maxItems` →
`ASSESSMENT_CONFIGURATION_INVALID`, **no attempt row created** (never runs an impossible coverage plan).
⚠️ Parameter VALUES + calibration are methodist-owned (TD-96) — not product-global defaults.

## 9. Engine state (§9/15/21)
`AssessmentAttempt.engineState` (class E) — `{schemaVersion, engineVersion, presentedItemIds[], answeredCount,
skills:{[skillId]:{targetDifficulty, answeredCount}}}`. Resumable mechanics only; response rows are the truth.
Strictly validated on read (corrupt shape → fail-safe). No JWT/PII/answer keys. Frozen on COMPLETED.

## 10. Resume behavior (§22)
`GET /attempts/:id` is a pure read; the `PRESENTED` response row is the current item. Reload never advances
(tested). DB is authority (no in-memory attempt state).

## 11. Objective scoring — camelCase, duplicate rejection (§3/11/25/26/27/28; OD/§2)
`ObjectiveScorerService` is the sole authority. Formats: `single_choice`, `true_false` (`{selectedOptionId}`),
`multiple_choice` (`{selectedOptionIds:[]}`) — **camelCase** application JSON (OD/§2; DB naming unchanged).
Type-specific normalization (canonical option ids; no blanket lowercasing). Multiple-choice requires an **exact**
set match with **unique** ids — `["A","A","B"]` is malformed → `ASSESSMENT_INVALID_RESPONSE` (no silent
canonicalization, §3). Score = basis points 10000/0 (TD-89); no partial credit.

## 12. Objective-only v1 + AI boundary (§22/23/24/29/30)
`placement-adaptive-v1` supports **objective items only**. Before starting, the **entire pinned pool** is
validated (§23): every payload valid `PLACEMENT_ITEM_V1`, format objective, `skillId` present, difficulty a
positive integer. An `open_ended`/unsupported item, or bad difficulty, → `ASSESSMENT_CONFIGURATION_INVALID`
with **no attempt created** (tested). We do not produce a partially-AI-dependent placement result without a
durable evaluation pipeline. `open_ended` remains parseable for FUTURE engines; AiEvaluation architecture is
untouched but **not called** here. Writing/speaking diagnostics + AI provider are a later engine/provider phase.

## 13. Response sequencing (§24)
`AssessmentResponse.sequenceNo` unique per attempt, assigned monotonically (present → answer → next). The
learner cannot skip/reorder; the answered item must be the current one (else `ASSESSMENT_ITEM_NOT_CURRENT`).

## 14. Transactions & concurrency (§47/48)
Submit = one `$transaction`: score → **atomic guarded transition** `updateMany where {id, status:PRESENTED}`
(`count===1` winner, mirrors auth session-rotation) → advance/complete. Concurrent duplicate: winner advances,
loser resolves via replay/conflict (§15). No duplicate sequence, no double completion (tested).

## 15. Idempotency + immutable-response replay/conflict (§4/5/49)
**No `clientRequestId`** — removed from the DTO/API (§4): the schema has no such column, so accepting it would
imply a guarantee that does not exist. Idempotency is **structural**:
- same already-answered item + **canonically equal** answer → idempotent replay (200, current state);
- same item + **different** answer → **409 `ASSESSMENT_RESPONSE_CONFLICT`** (an immutable response cannot
  silently absorb a second, different answer). Comparison is format-specific canonicalization (order-independent
  for multiple-choice), not raw-JSON.
Duplicate-**start** resolves to a single attempt via the (user,subject) partial-unique (§16).

## 16. Start / resume authority + one-in-progress (§7/8/14/27/28; TD-95)
Resume authority is **(user, subject) IN_PROGRESS INITIAL_DIAGNOSTIC** — NOT the version. If one exists, start
resumes it and keeps its pinned `definitionVersionId`/`engineVersion`/`trackId` (a moved current pointer never
repins to v2, tested). At most one such attempt is enforced by `uq_attempt_inprogress_initial_diagnostic_user_subject`
(PA-10). Concurrent starts: the DB unique is the final authority — the loser catches `P2002` and resumes the
winner (tested: two concurrent starts → one attempt). Completed attempts are unaffected; **retake/REASSESSMENT
policy remains OPEN** (no completed-attempt limit implemented).

## 17. Completion + result summary (§16/20/21/35/36)
On the stopping rule (every pool skill met `itemsPerSkill`, OR no unseen eligible item, OR `answeredCount ≥
maxItems`): one transaction persists the final response, freezes `engineState`, sets `COMPLETED` + `completedAt`,
writes `resultSummary`. No Roadmap/mastery/XP (asserted absent). `resultSummary` (class B, write-once):
`{schemaVersion, engineVersion, answeredCount, objectiveCorrect, objectiveScored, coverageSkillIds,
coverageComplete, insufficientSkillIds}`. `pendingAiEvaluation` **removed** (objective-only). Insufficient pool
(a skill with too few items) → no repeats, `coverageComplete=false`, skill listed in `insufficientSkillIds`
(§20/33, tested). **No CEFR/level label** — Skill Profile phase owns interpretation.

## 18. Security / answer-key protection (§26; regression)
`projectItemForLearner` exposes only `{id, type, format, prompt, options:[{id,text}]}`; `answerKey`,
`difficulty`, `skillId`, provenance never leave the server (tested: no `answerKey`/`correctOptionIds`/`skillId`/
`difficulty` in any HTTP body). Client never sends correctness; top-level injected fields → 400, nested → 400
`ASSESSMENT_INVALID_RESPONSE`. No answer/config/engineState logged (safe ids only).

## 19. Media (§17)
`AssessmentItemMedia` relational truth; signed-URL/object-storage delivery not implemented (deferred). Objective
v1 works text-only; the projection exposes no media asset (no storageKey/provider leak).

## 20. APIs (§43/44/45)
All authenticated, own-user, no authoring routes.

| Method | Path | Notes |
|---|---|---|
| GET  | `/api/assessments/placement/availability?learningIntentId=` | `{ available }` only |
| POST | `/api/assessments/placement/start` | `{ learningIntentId }`; 200; starts or resumes |
| GET  | `/api/assessments/attempts/:attemptId` | resume/read; pure |
| POST | `/api/assessments/attempts/:attemptId/responses` | `{ itemId, answer }`; 200 |

Error codes: `ONBOARDING_INCOMPLETE` (409), `LEARNING_INTENT_NOT_FOUND` (404), `ASSESSMENT_NOT_AVAILABLE` (404),
`ASSESSMENT_CONFIGURATION_INVALID` (409), `ASSESSMENT_ATTEMPT_NOT_FOUND` (404), `ASSESSMENT_ALREADY_COMPLETED`
(409), `ASSESSMENT_ITEM_NOT_CURRENT` (409), `ASSESSMENT_RESPONSE_CONFLICT` (409), `ASSESSMENT_INVALID_RESPONSE` (400).

## 21. Tests (unit 53, e2e 20; project total unit 130 / e2e 82)
- **Unit:** engine (per-skill init, skill-balance §31, difficulty independence §32, coverage failure §33,
  determinism, no-repeat, stopping, within-skill targeting); scorer (correct/incorrect/malformed/**duplicate
  rejected**/unknown/extra-fields; canonicalize replay §5); config fail-safe (coverage, removed minItems).
- **E2e:** full flow + reproducible evidence + coverage + **side-effects absent**; resume; version pinning +
  **start-resume by (user,subject)** + new learner uses v2 (§9/28/38/50); pool membership (§51); answer-key leak
  (§52); ownership (§53); **skill balance §31**; **per-skill difficulty independence §32** (via engineState);
  **coverage failure §33**; **impossible config §34**; **open-ended pool → config invalid before start §24**;
  **replay same/different §5/30**; **MC order-independent replay + duplicate → 400 §3/30**; **concurrent start →
  1 attempt §27**; **DB diagnostic uniqueness §29**; lifecycle guards (§13/24); injected-score rejection; gates
  (onboarding/availability); auth.
- Fixtures test-only (izlan_test), built directly (no authoring API). Regression: all prior phases green.

## 22. Deferred Skill Profile (§40)
Evidence stops here. `resultSummary` (coverage + counts) + per-response `(itemId, isCorrect, deterministicScore,
item.skillId)` give the next phase everything to derive `SkillMeasurement` / `LearnerSkillState`. Item→skill via
required `AssessmentItem.skillId`.

## Owner decisions closed this phase (ACCEPTED)
OD-1 mapping · PLACEMENT_ITEM_V1 technical contract · camelCase answers · subject-level diagnostic · one
published diagnostic per subject (TD-94) · one in-progress initial diagnostic per user+subject (TD-95) ·
skill-balanced adaptive v1 (TD-96) · objective-only v1.

## Remaining OPEN (do NOT block completion) — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)
Completed-attempt retake/reassessment policy · CEFR/level threshold mapping · confidence-based stopping ·
IRT/advanced psychometrics · writing/speaking diagnostic · AI evaluation provider · media delivery · real
diagnostic question bank · methodist calibration process.
