# Roadmap Foundation — Implementation (Phase 1.6A)

> Status: **COMPLETE** (2026-08-19). Produces the first deterministic ACTIVE `LearnerRoadmap` for a
> learner with a completed diagnostic Skill Profile. **No schema change / no migration** — the accepted
> Phase 1.3 roadmap + content-mapping + prerequisite + provenance + one-active schema was sufficient.
> No AI, no DailyPlan, no Lesson execution, no XP/IZL, no LearnerSignal.

Code: `backend/src/roadmap/**`.

## 1. Scope
A learner with a completed INITIAL_DIAGNOSTIC can receive one deterministic ACTIVE roadmap for that
Subject: ranked Skill gaps → human-approved published content resolved via explicit mappings →
prerequisite-closed → deterministically ordered → persisted. Reproducible; no pedagogy inferred from titles/AI.

## 2. Roadmap source
The exact diagnostic **SkillMeasurement snapshot** (§7) — not mutable current `LearnerSkillState`. So an
old diagnostic never generates a roadmap from unrelated newer learning state (tested §53). Current state is
still available for the profile API; it is not the roadmap priority authority.

## 3. Exact diagnostic snapshot
`POST /api/roadmaps/diagnostics/:attemptId/initial` requires the attempt to be own + COMPLETED +
INITIAL_DIAGNOSTIC, with a derived profile (`SkillMeasurement` rows: `assessmentAttemptId = attemptId`,
`source = DIAGNOSTIC`, `derivationVersion = skill-profile-diagnostic-v1`). Missing profile → 409
`SKILL_PROFILE_NOT_DERIVED`.

## 4. gapPriority formula (TD-99, `roadmap-gap-v1`)
```
weaknessBp    = 10000 − masteryScoreBp
gapPriorityBp = round(weaknessBp × confidenceBp / 10000)   # clamp 0..10000
```
Ranking only — NOT mastery/CEFR/threshold/label/probability (§3). `masteryScoreBp` + `confidenceBp` come
from the snapshot; `evidenceCount` is reproduced from the attempt's SUBMITTED responses (not current state).

## 5. Confidence semantics
Confidence multiplies weakness so a confidently-identified gap outranks a weak-looking-but-low-evidence one
(§4): mastery 2000/confidence 10000 → priority 8000; mastery 2000/confidence 2500 → priority 2000. The
latter is NOT "strong" — it simply gets lower deterministic priority until stronger evidence exists.

## 6. No thresholds (§5/41)
No `mastery < X → weak` cutoff. Every measured Skill with `evidenceCount > 0` (i.e. every SkillMeasurement)
participates and is ranked (mastery 9500 still ranked, tested §41). Sort (§6): gapPriority DESC, mastery
ASC, confidence DESC, evidence DESC, skillId ASC. Pure `GapRankingEngine` (unit-tested).

## 7. Content→Skill mapping authority (§11)
The ONLY mapping used is **`LessonSkill`** (explicit relational truth). No title/hierarchy/keyword/AI
inference; an unmapped lesson is never selected (tested §43). No competing mapping; no Skill ids stuffed
into roadmap JSON.

## 8. Published-content filtering (§9)
Learner-visible lesson = `Lesson.status = PUBLISHED` **and** has a PUBLISHED `LessonRevision`
(`publishedRevisionId != null`) **and** the whole `Topic → Module → Level` chain is PUBLISHED. DRAFT /
ARCHIVED / no-published-revision lessons are excluded (tested §42). Human-approved curriculum only — no
AI-generated / synthetic content (§10).

## 9. Completed-content exclusion (§12)
Lessons the learner has COMPLETED (`LearnerLessonCompletion`) are excluded. Merely started/in-progress
(`LearnerLessonProgress`) lessons remain candidates; progress is never mutated (tested §44).

## 10. Prerequisites (§13/14)
`LessonPrerequisite` (explicit DAG). A candidate's prerequisites are followed transitively: a COMPLETED
prerequisite is satisfied (no item); an uncompleted, learner-visible prerequisite is added and ordered
before its dependent (cross-skill too, tested §46); an uncompleted non-visible prerequisite makes the
dependent unreachable (dropped, its skill may become uncovered).

## 11. Deterministic ordering (§16/28/29)
Priority-aware topological sort over `RoadmapItem.position`: prerequisites precede dependents; among
currently-unblocked lessons the highest **effective** gap priority wins (a prerequisite inherits the highest
priority of anything it unblocks), then `Topic.sortOrder`, `Lesson.sortOrder`, `lessonId`. No randomness, no
DB default order. Pure `prerequisite-ordering` (unit-tested).

## 12. Duplicate handling (§18/47)
A lesson selected by multiple Skills and/or as a prerequisite appears exactly once (tested §47). Its
`RoadmapItem.skillId` is the originating gap Skill that gave it its effective priority (provenance via the
accepted relation, not JSON).

## 13. Uncovered Skills (§19/48)
A measured Skill with no reachable eligible-mapped content is not fabricated — its id is returned in the
generation response `uncoveredSkillIds` (not persisted; no schema field exists for it). Covered Skills still
generate a roadmap (tested §48).

## 14. Roadmap persistence (§55)
`LearnerRoadmap` (status ACTIVE) + all `RoadmapItem`s (itemType LESSON, source INITIAL_GENERATION, lessonId,
skillId, position, status PENDING) created in **one transaction**. Never leaves a half/empty ACTIVE roadmap.

## 15. Provenance (§25)
`LearnerRoadmap.sourceAssessmentAttemptId = attemptId` (exact diagnostic). Item source
`INITIAL_GENERATION` (deterministic system-derived) — no AI source.

## 16. Idempotency (§22/52)
Keyed on the DB one-active invariant + `sourceAssessmentAttemptId`: an existing ACTIVE roadmap from the SAME
attempt → returned unchanged (same id/items, tested §52); from a DIFFERENT attempt → 409
`ROADMAP_ALREADY_ACTIVE` (no replacement, tested §51). No `find→create` reliance.

## 17. Concurrency (§24/50)
The **`ux_active_roadmap`** partial-unique (`user_id, subject_id WHERE status = ACTIVE`, L9 — already in the
schema) is the final authority. Two concurrent generations → one wins the insert, the loser catches P2002
and returns the winner (same source → idempotent, tested §50). No duplicate ACTIVE roadmap/items.

## 18. Active-roadmap semantics (§21/23)
At most one ACTIVE roadmap per (user, subject), DB-enforced. Regeneration/replacement is OPEN — the initial
generator never deletes/replaces an existing ACTIVE roadmap.

## 19. APIs (§36)
All authenticated, own-user, no write/edit endpoint, no client-supplied data.

| Method | Path | Notes |
|---|---|---|
| POST | `/api/roadmaps/diagnostics/:attemptId/initial` | generate/idempotent-return `{ roadmap, uncoveredSkillIds }` |
| GET  | `/api/roadmaps/me/subjects/:subjectId/active` | current ACTIVE roadmap; 404 `ROADMAP_NOT_FOUND` |
| GET  | `/api/roadmaps/:roadmapId` | own roadmap by id (404 for others) |

Errors: `ROADMAP_NOT_FOUND` (404), `ROADMAP_ALREADY_ACTIVE` (409), `ROADMAP_NO_ELIGIBLE_CONTENT` (409),
`ROADMAP_CONFIGURATION_INVALID` (409 — prerequisite cycle / missing track), `ASSESSMENT_ATTEMPT_NOT_FOUND`
(404), `SKILL_PROFILE_NOT_DERIVED` (409).

## 20. Security (§39/54)
Own-user filters everywhere (IDOR-safe → 404 for others). No answer keys / assessment internals / unpublished
content / authoring metadata exposed. Only learner-facing roadmap item fields returned.

## 21. Tests (unit 16, e2e 15; project total unit 167 / e2e 105)
- **Unit:** gap ranking §40/41 (formula, high-mastery-not-dropped, tie-break, guards); prerequisite ordering
  §15/45 (topo, cycle → invalid, priority tie-break), reachability closure (completed satisfied / non-visible
  blocks), priority inheritance §14.
- **E2e:** happy path (gap order + provenance + no side-effects); visibility §42; explicit-mapping §43;
  completion exclusion §44; prerequisite closure §45; cross-skill prerequisite §46; dedup §47; uncovered §48;
  empty §49; concurrent §50; different-source conflict §51; same-source idempotent §52; historical-snapshot
  §53; security §54; GET active 404.
- Regression: all prior phases green.

## 22. Deferred regeneration (§59)
Roadmap regeneration / replacement / manual learner edits / skipping / reordering are OPEN — not implemented.

## 23. DailyPlan boundary (§32)
No `DailyPlan` / `DailyPlanItem`. Daily Plan is a later layer that will consume the roadmap.

## 24. AI boundary (§34)
No AI provider, no LLM, no generated recommendations/explanations. Deterministic selection over approved
content only. `RoadmapItemSource.RECOMMENDATION` / `LearnerRecommendation` (AI acceptance flow) are untouched.

## Reproducibility note
`roadmap-gap-v1` / deterministic selection are CODE identifiers (not persisted — `LearnerRoadmap` has no
engineVersion field, and none was silently added). Reproducibility holds from immutable inputs:
`sourceAssessmentAttemptId` → immutable SkillMeasurements + reproduced evidence counts + published content
state + explicit mappings + prerequisite DAG + the deterministic algorithm. A persisted algorithm-version
field is a small additive option for a future `-v2` (owner decision), not required now.

## Remaining OPEN — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)
Roadmap regeneration/replacement · manual edits/skip/reorder · primary/default Subject · AI roadmap proposals
+ acceptance UX · lesson difficulty personalization · target mastery thresholds · CEFR/Level mapping ·
low-confidence reassessment policy · DailyPlan extraction strategy.
