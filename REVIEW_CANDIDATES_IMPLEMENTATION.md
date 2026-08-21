# Review Candidates — Implementation (Phase 1.9A)

> Status: **COMPLETE** (2026-08-20). A read-only derived model: an authenticated learner asks "which previously
> encountered, currently-visible content is relevant for review, and why?" and the server returns deterministic
> candidates grouped by Skill, derived from ACTIVE `REPEATED_MISTAKE` / `WEAK_SKILL` / `REVIEW_DUE` signals.
> **No schema change, no persistence, no execution, no state mutation.** Owner decisions: **TD-122 / TD-123 / TD-124**.

Code: `backend/src/review/**` (pure `candidate/review-candidate.engine.ts` + types; `review.repository.ts`;
`review.service.ts`; `review.controller.ts`).

## 1. Scope
Return deterministic review **candidates** (logical Lessons) grouped by Skill for a Subject. **NOT** in scope:
ReviewSession, ActivityAttempt, LessonProgress, DailyPlan EXTRA, Roadmap insertion, signal resolution, scoring,
review execution, notifications, rewards, AI.

## 2. Review candidate definition (review-candidate-v1, TD-122)
Immutable content-candidate policy `review-candidate-v1`. A candidate is a **logical Lesson** the learner has
already encountered, that is currently learner-visible, and is explicitly relevant to a Skill carrying an ACTIVE
supported signal. It is not a scheduler, DailyPlan/Roadmap policy, execution policy, or AI recommender.

## 3. Derived / read-only architecture (§2/84)
`ReviewCandidate` is reconstructed each request from current ACTIVE signals + learner exposure + current
approved curriculum — **no `ReviewCandidate`/`ReviewQueue`/`SpacedRepetitionCard` table, no persistence.** The
existing `LearnerRecommendation` model is a roadmap-change proposal entity (unused) and does not conflict.

## 4. Active signal inputs (§4)
Only ACTIVE `REPEATED_MISTAKE`, `WEAK_SKILL`, `REVIEW_DUE` (explicit whitelist). RESOLVED/EXPIRED/unknown types
are ignored. `Signal.skillId` is the grouping authority (§5); a supported signal with `skillId = null`
(historical SetNull) is skipped safely — never fabricated.

## 5. Encountered-only policy (TD-123, §6/7)
A logical Lesson qualifies only if the learner has a `LearnerLessonProgress` OR `LearnerLessonCompletion` for
it. A signal never introduces unseen curriculum as "review" — new content belongs to Roadmap/DailyPlan. Unseen
mapped Lessons are excluded (tested §50).

## 6. Exposure authority (§8/52)
Response `exposure` is derived (not persisted): `COMPLETED` if a completion exists, else `IN_PROGRESS`.
Completion wins over progress.

## 7. Current visibility (§9/10/64)
Even if encountered, a Lesson is eligible only if currently learner-visible: `Lesson.status = PUBLISHED` AND a
current `publishedRevisionId` AND `Topic/Module/Level/Track` all `PUBLISHED`, within the requested Subject.
Archived/unpublished content is excluded; historical completion/signals are never deleted. Title comes from the
current published revision.

## 8. Explicit LessonSkill mapping (§11)
`LessonSkill(lesson, S)` makes a visible encountered Lesson relevant to Skill S.

## 9. Explicit ActivitySkill mapping (§11/12/13)
An Activity in the **current published revision** with `ActivitySkill(activity, S)` also makes the Lesson
relevant to S — precise attribution counts even without a broad LessonSkill. General discovery inspects only the
current revision (obsolete old-revision mappings do not count, tested §59).

## 10. REPEATED_MISTAKE direct-trigger provenance (TD-124, §14/16)
`REPEATED_MISTAKE.evidenceRefs.triggerActivityIds` (parsed by the strict `repeated-mistake-signal/v1` parser)
resolve each trigger Activity → its (possibly historical) `LessonRevision` → **logical Lesson**. If that Lesson
is same-Subject, encountered, and currently visible, it is a `directTrigger` candidate — the signal itself
proves the learner struggled with that Skill there. Malformed/missing/cross-subject/absent-Activity evidence is
skipped safely (the Skill can still get candidates via normal mappings, tested §70); no parse error leaks.

## 11. Logical Lesson semantics (§15/57)
A candidate is a logical Lesson id — **not** a pinned revision and **not** an Activity replay contract. A
direct-trigger Lesson that has since republished (v1→v2) remains a candidate; the old revision's Activity payload
is never exposed. Revision/Activity semantics are deferred to a future Review Execution phase.

## 12. No historical Activity replay (§14/15/41)
The API exposes `lessonId` but promises no activity replay / old-revision execution / current-revision execution.
No raw attempt history ("you got Activity X wrong 3×") is returned — only `signalTypes` + `directTrigger`.

## 13. Deduplication (§23/61)
A Lesson qualifying via several routes (LessonSkill, current ActivitySkill, direct trigger, multiple signal
types) appears **once** per Skill group, with `directTrigger` true if any trigger route applied.

## 14. Grouping by Skill (§20/62)
One group per Skill (`user + subject + skill`). Multiple ACTIVE signal types for a Skill produce one group whose
`signalTypes` lists them; the same Lesson may appear under two Skill groups only if eligible for both.

## 15. Deterministic ordering (§21/24/38/71)
Groups: `Skill.sortOrder`, then name, then id. Within a group: `directTrigger` first, then curriculum hierarchy
(`Level.sortOrder → Module.sortOrder → Topic.sortOrder → Lesson.sortOrder → Lesson.id`). `signalTypes` serialize
in the canonical order `REPEATED_MISTAKE → REVIEW_DUE → WEAK_SKILL` (stable serialization, **not** a score).
Repeated calls are identical. No composite urgency score, no candidate cap (§22/25/26/72).

## 16. Uncovered Skills (§27/28/29/69)
A Skill with an ACTIVE supported signal but zero eligible candidates is listed in `uncoveredSkillIds` (no fake
content). A Subject with signals but no candidates → `200 { groups: [], uncoveredSkillIds: [...] }`. A Subject
with no ACTIVE supported signals → `200` empty (not 404). A missing Subject → 404 (existing safe-resource
semantics).

## 17. Roadmap independence (§32/65/80)
A candidate may be an encountered Lesson no longer on the ACTIVE Roadmap — allowed (review history ≠ current
roadmap snapshot). No RoadmapItem is created/reordered; no Roadmap regeneration.

## 18. DailyPlan independence (§33/66/79)
A candidate may be outside today's Topic — allowed (read-only). No DailyPlanItem / no EXTRA generation; one-topic-per-day intact.

## 19. Execution boundary (§34/35/67/78)
No LearnerLessonProgress write, no resume/repin, no ActivityAttempt, no review start endpoint. Candidates are
not executable; normal Lesson start still requires today's DailyPlanItem.

## 20. Signal lifecycle boundary (§36/68)
Reading candidates never resolves REVIEW_DUE / WEAK_SKILL / REPEATED_MISTAKE — seeing a recommendation is not
learning evidence. No LearnerSignal write. Time-based REVIEW_DUE eligibility is **not** re-evaluated in this GET
(§77): callers needing fresh timing run `POST learner-signals reconcile` first — no hidden mutation.

## 21. Security (§73/74)
Own-user only (`principal.userId`); a foreign request sees only the caller's exposure/signals. `subjectId` is
UUID-validated (400); no auth → 401; missing Subject → 404. Responses expose no `evidenceRefs` /
`triggerAttemptIds` / `triggerActivityIds` / `Activity.payload` / answer keys / `dueAt` / `basisLastMeasurementAt`
(asserted). No sensitive logging.

## 22. Query batching (§45)
One request = O(few) batched queries: ACTIVE signals; in-subject Skill metadata; encountered progress +
completion; visible in-subject Lessons (+ hierarchy sort + current revision id + title); LessonSkill;
current-revision ActivitySkill; trigger-Activity → Lesson. No N+1, no per-Skill/per-Lesson loops, no cache.

## 23. Tests
- Unit: `candidate/review-candidate.engine.spec.ts` (10) — empty; LessonSkill / current-ActivitySkill maps;
  no-inference → uncovered; direct trigger (+ dropped when not visible); multi-signal/multi-mapping dedup +
  canonical order; multi-skill group order; direct-first hierarchy order; no cap.
- E2e: `test/review-candidates.e2e-spec.ts` (10) — no-signals/RESOLVED-ignored; unseen-excluded + exposure;
  LessonSkill/ActivitySkill/no-inference; direct trigger (historical revision) + archived-trigger excluded;
  current-mapping general discovery; multi-signal dedup; multi-skill + cross-subject + hidden-content;
  read-doesn't-resolve + uncovered + malformed-evidence fallback; deterministic order + no cap; IDOR/401/400 +
  no-leak + side-effect boundary.
- Totals: unit **286**, e2e **193**.

## 24. Future Review Session consumer
A later phase may define learner-triggered review **execution** over these candidates (revision pinning, Activity
retry selection, review mastery evidence, signal resolution from review) — out of scope here.

## 25. Future DailyPlan EXTRA consumer
A later phase may let a planner consume candidates into DailyPlan (EXTRA) or Roadmap review insertion under an
accepted workload/prioritization policy — explicitly deferred; 1.9A returns candidate groups only.
