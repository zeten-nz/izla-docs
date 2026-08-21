# Roadmap Progress Read Model — Implementation (Phase 1.6B)

> Status: **COMPLETE** (2026-08-20). Interprets an existing `LearnerRoadmap` as learner progress (derived
> read model) and adds an idempotent ACTIVE→COMPLETED reconciliation command. **No schema change / no
> migration** — derived states + progress are not persisted; reconciliation uses the existing
> `RoadmapStatus` enum. No new roadmap content, no DailyPlan, no Lesson execution, no AI/XP/IZL/Signal.

Code: `backend/src/roadmap/read/**` + `roadmap.service.reconcileCompletion` + repository batch reads.

## 1. Scope
An authenticated learner reads roadmap items in deterministic order with a derived state each (COMPLETED /
IN_PROGRESS / AVAILABLE / BLOCKED / UNAVAILABLE), an overall progress summary, and the recommended next
executable item. An internal idempotent command transitions ACTIVE→COMPLETED when every item is completed.

## 2. Progress authorities (§2)
- **Completion truth = `LearnerLessonCompletion`** (by logical `lessonId`). `RoadmapItem` is NEVER a second
  completion authority — no `RoadmapItem.status`/percent is written by this phase (the field stays at its
  generation default; nothing here mutates it).
- **Progress = `LearnerLessonProgress`** (row existence = the learner has begun the lesson).
- **Availability = current learner-visible content** (Lesson PUBLISHED + published `LessonRevision` +
  Topic→Module→Level PUBLISHED).
- **Prerequisites = `LessonPrerequisite` + actual completion.**
Nothing derived is persisted.

## 3. Derived item-state precedence (TD-101, §11)
`COMPLETED > UNAVAILABLE > IN_PROGRESS > BLOCKED > AVAILABLE`. Pure `roadmap-progress.ts` (unit-tested).

## 4. COMPLETED (§4)
Item is COMPLETED iff a `LearnerLessonCompletion` exists for `RoadmapItem.lessonId`. Never inferred from
progress %, last activity, roadmap position, or DailyPlan. Completion history wins even if the content is
later archived (§28, tested).

## 5. IN_PROGRESS (§5)
Not-completed + currently available + a `LearnerLessonProgress` row exists. Existence alone means progress
(no invented percentage threshold). COMPLETED and UNAVAILABLE both take precedence.

## 6. AVAILABLE (§10)
Not-completed + not-in-progress + currently visible + every explicit prerequisite completed. The learner may
start it now (no DailyPlan assignment required).

## 7. BLOCKED (§9)
Not-completed + currently visible + at least one explicit prerequisite not completed. A prerequisite that
appears earlier in the roadmap but is not actually completed still BLOCKS (order ≠ completion, §8).

## 8. UNAVAILABLE (§6/7)
Not-completed + the lesson is no longer learner-visible (unpublished/archived, or hierarchy archived after
generation). The roadmap item is **kept** (never deleted); its title is withheld (only the CURRENT published
revision title is ever exposed — no draft/archived body leak, §7, tested §27).

## 9. Prerequisite satisfaction (§8/30)
A prerequisite is satisfied iff its lesson has an authoritative `LearnerLessonCompletion` — regardless of
whether the prerequisite is itself a roadmap item. An external prerequisite completed before generation (so
excluded from the roadmap) still satisfies its dependent; removing that completion re-BLOCKS it (tested §30).

## 10. Current content visibility (§6)
Checked at read time against current lifecycle. Valid-at-generation content that later becomes
archived/unpublished → UNAVAILABLE (never deleted, never auto-completed).

## 11. Logical Lesson / current revision (§25/26)
`RoadmapItem` targets the **logical Lesson** (no revision pin — an accepted 1.6A outcome). The read model
surfaces the CURRENT published revision's title/metadata. A roadmap generated at revision v1 will show v2's
current learner-facing metadata after v2 is published — the roadmap means "learn this logical Lesson", not
"learn exactly revision v1". Revision pinning for attempts/completion is a later Lesson-execution concern.

## 12. progressBp (§13)
Read-model value, NOT persisted: `total>0 ? round(completed/total × 10000) : 0` (0..10000 integer). Plus per
state counts (total/completed/inProgress/available/blocked/unavailable), consistent, no double counting.

## 13. Next-item rule (§19/20)
`nextItemId` = earliest-position IN_PROGRESS item, else earliest-position AVAILABLE item, else null. Never
BLOCKED/UNAVAILABLE/COMPLETED. Roadmap `position` (from 1.6A) is canonical — gap ranking is NOT rerun.

## 14. ACTIVE→COMPLETED reconciliation (TD-102, §14/15)
A roadmap is learning-complete iff **EVERY** persisted `RoadmapItem` Lesson has an authoritative completion —
NOT "all available items". An UNAVAILABLE-but-never-completed item keeps the roadmap incomplete (no silently
dropped learning). `reconcileCompletion(userId, roadmapId)` conditionally updates status ACTIVE→COMPLETED
(`updateMany WHERE status=ACTIVE`). `LearnerRoadmap` has no completedAt field, so only status changes. Never
COMPLETED→ACTIVE.

## 15. Idempotency / concurrency (§15/33/34/35)
Reconcile is idempotent (already-COMPLETED → no-op) and concurrency-safe (conditional update; the second of
two concurrent calls matches 0 rows and still returns COMPLETED, tested §35). Partial completion → stays
ACTIVE (§34).

## 16. APIs (§21/22/18)
| Method | Path | Notes |
|---|---|---|
| POST | `/api/roadmaps/diagnostics/:attemptId/initial` | generate (1.6A, unchanged) |
| GET  | `/api/roadmaps/me/subjects/:subjectId/active` | progress read model; excludes COMPLETED/ARCHIVED; 404 else |
| GET  | `/api/roadmaps/:roadmapId` | progress read model (any status; 404 for other users) |
| POST | `/api/roadmaps/:roadmapId/reconcile` | command: ACTIVE→COMPLETED when all items completed (idempotent) |

Both GETs share ONE projector (`RoadmapReadService`, §22/23). **GET endpoints never mutate** (§16, tested):
reconciliation happens only via the command. `reconcileCompletion` is also the clean service hook a future
Lesson-completion flow will call (§17) — no circular module dependency.

## 17. Batch-query design (§24)
Fixed number of queries regardless of item count: `completedLessonIds(userId)`, `inProgressLessonIds(userId)`
(parallel with) `lessonMeta(lessonIds)`, then `prerequisiteEdges(lessonIds)`. O(few) not O(items) — no N+1.

## 18. Security (§47)
Own-user filters everywhere (IDOR-safe → 404 for others; an attacker's reconcile cannot complete the owner's
roadmap, tested §47). No answer keys / assessment internals / draft content / authoring metadata exposed.

## 19. Tests (unit 16, e2e 10; project total unit 183 / e2e 115)
- **Unit:** state precedence (all 5 + null lessonId + external prereq); prerequisite chain transitions §29;
  progress summary §31; next-item §32.
- **E2e:** read-model shape (states + summary + nextItemId + order + title); archived-after-generation →
  UNAVAILABLE + no leak + not auto-completed §27; completed-then-archived → COMPLETED §28; prerequisite
  transitions §29; external prerequisite §30; all-completed reconcile (idempotent) + GET-no-mutation §16/33;
  partial reconcile §34; concurrent reconcile §35; completed-roadmap read + active-excludes-completed §36;
  security §47.
- Regression: all prior phases green.

## 20. DailyPlan boundary (§43)
No `DailyPlan`/`DailyPlanItem`. Phase 1.6B produces the read model DailyPlan will consume.

## 21. Lesson-execution boundary (§44)
No start-lesson / submit-activity / complete-lesson API. Tests seed `LearnerLessonProgress`/`Completion`
directly. Roadmap read/reconcile never mutates progress/completion, LearnerSkillState, or SkillMeasurement
(§39). Mastery updates from lessons are a later phase.

## 22. Regeneration boundary (§40/41/42)
UNAVAILABLE/BLOCKED items are never regenerated, replaced, deleted, or reordered; there is no SKIPPED state
and no completion credit for them. Regeneration / manual-edit policy remains OPEN.

## Remaining OPEN — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)
Roadmap regeneration/replacement · unavailable-content remediation · skip semantics · manual reorder/removal ·
target mastery semantics · next roadmap after completion · DailyPlan extraction · lesson-execution revision
pinning · lesson-completion mastery update · signals · rewards · AI personalization.
