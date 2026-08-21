# DailyPlan Review EXTRA Integration — Implementation (Phase 2.0A)

> Status: **COMPLETE** (2026-08-20). When a NEW DailyPlan is generated, the planner may add **at most one**
> optional same-Topic review EXTRA, chosen deterministically from the current ReviewCandidate read model. Core
> one-Topic planning, MUST_DO/RECOMMENDED, and same-day idempotency are unchanged; the EXTRA never affects core
> completion. **No schema change** (`DailyPlanItemType.REVIEW` + nullable `roadmapItemId`/`lessonId`/`skillId`
> already exist). Owner: **TD-133/134/135**. No Roadmap/ReviewSession/signal/reward/notification/AI writes.

Code: `backend/src/daily-plan/daily-review-extra-selection.ts` (pure), `daily-plan.service.ts` (generation hook),
`daily-plan-read.service.ts` (projection), `daily-plan.repository.ts`.

## 1. Scope
A newly-generated DailyPlan gains 0..1 optional review EXTRA from a current ReviewCandidate in the SAME Topic as
the core. NOT: RoadmapItem, mandatory learning, a second core Topic, ReviewSession auto-create, reward evidence,
DailyPlanItem→ReviewSession completion provenance, multiple review items, workload budgets.

## 2. Core one-Topic invariant (§4/19)
TD-104 is authoritative: normal roadmap learning is exactly one Topic per local day (MUST_DO = 1.6B nextItem;
RECOMMENDED = same-Topic later items). Generation still requires an ACTIVE roadmap + nextItem; there is no
review-only plan. The EXTRA is added only after core selection succeeds.

## 3. Same-Topic auto-review (§4/5/11)
An auto review EXTRA MUST belong to `ReviewCandidate.lesson.topicId == core Topic id`. So one generated plan
still contains learning content from only one Topic — automatic daily planning never fans out across Topics.

## 4. Manual cross-topic review independence (§6/62)
The same-Topic limit applies ONLY to automatic EXTRA insertion. A learner may still manually start ANY currently
valid ReviewCandidate (any Topic) via the existing 1.9B ReviewSession start — unchanged (e2e §62).

## 5. One-EXTRA workload cap (TD-133, §7/8)
At most ONE auto review EXTRA per DailyPlan (not per Skill/signal). MVP workload policy; the manual Review flow
is unlimited by this phase.

## 6. Optional EXTRA semantics (§9/26/27/59)
Section = `EXTRA` (never MUST_DO/RECOMMENDED). Review completion is not required for DailyPlan done / Roadmap /
Lesson completion. **Core `done`/`progress` are computed from roadmap-backed core items ONLY** (the read
projector filters out `section=EXTRA`) — the EXTRA never reduces or inflates core completion (mandatory e2e §59).

## 7. ReviewCandidate authority (§10)
During generation, eligibility comes from the Phase 1.9A `ReviewService.getCandidates` (read-only) — no signal
query/rebuild, no WEAK/REVIEW_DUE re-computation, no content inference. The DailyPlan planner only orders and caps.

## 8. Deterministic priority (TD-134, §13)
Among eligible same-Topic, non-core-duplicate candidates, exactly one is chosen by the lexicographic key: (1)
directTrigger, (2) strongest reason, (3) exposure, (4) Skill order, (5) 1.9A Lesson hierarchy, (6) lessonId.
No numeric urgency score. Pure `selectReviewExtra` (unit-tested).

## 9. Direct-trigger priority (§15/52)
`directTrigger = true` wins over a higher-reason non-trigger candidate — it has exact recent-struggle provenance.

## 10. Reason priority (§14/53)
Now an accepted **planning** policy: `REPEATED_MISTAKE > REVIEW_DUE > WEAK_SKILL`. A candidate inherits its Skill
group's active reasons; the strongest present reason ranks it. (In 1.9A this order was serialization only.)

## 11. Exposure priority (§16/54)
After direct-trigger/reason: `COMPLETED` before `IN_PROGRESS` (prefer material whose original lesson flow finished;
IN_PROGRESS stays eligible).

## 12. Core dedup (§12/50)
If a candidate's logical Lesson is already in today's core (MUST_DO/RECOMMENDED), it is skipped — the next eligible
candidate is chosen instead (e2e §50). A Lesson qualifying under multiple Skills is a distinct (skill, lesson)
intent (§45); the persisted EXTRA keeps its `skillId` (ReviewSession needs it).

## 13. Snapshot immutability (§20/24/29/56/57)
The selected EXTRA is part of the immutable DailyPlan snapshot: `section=EXTRA, itemType=REVIEW, roadmapItemId=NULL,
lessonId, skillId`. No ReviewCandidate object / signal ids / evidenceRefs / dueAt / mastery / ReviewSession id are
persisted (`itemType=REVIEW` is the unambiguous discriminator — no new enum, no source JSON needed). A same-day
re-`POST /today` returns the same plan with the same EXTRA even if signals changed (e2e §56/57).

## 14. Stale EXTRA (§24/61)
If the snapshotted EXTRA is no longer a current ReviewCandidate at start time, the existing 1.9B ReviewSession
start revalidation rejects it. The DailyPlan snapshot is not mutated, the DailyPlanItem does not authorize start,
and signals are not reopened — historical plan truth and live execution authority stay separate.

## 15. ReviewSession revalidation (§23/61)
The client uses the EXTRA's `subjectId/skillId/lessonId` with the existing ReviewSession start, which re-derives
the current candidate (1.9B security preserved). The DailyPlanItem is not an execution authority.

## 16. No ReviewSession auto-create (§22/60)
Generating a DailyPlan creates no `LearnerReviewSession` — daily planning is not execution (e2e §60).

## 17. Same-day behavior (§29/30)
Review EXTRA selection happens only at NEW plan generation. A later REVIEW_DUE/repeated-mistake/weak signal, or a
stale EXTRA, never appends/changes an EXTRA that day. Finishing core early adds no Topic and no EXTRA.

## 18. Next-day reevaluation (§31/32/33)
A fresh plan on the next local date re-evaluates current candidates: the EXTRA may disappear, repeat, or change
Skill/Lesson per current signal/review state and v1 ordering. No carry-forward row, no hidden cooldown.

## 19. DailyPlan done/progress boundary (§27/59)
Core completion denominator = roadmap-backed core items only. Verified: core `done=true` with an un-reviewed EXTRA
still present.

## 20. Security (§66/67/68)
Own-user only (unchanged). Visibility is enforced upstream by ReviewCandidate (no archived/draft review lesson).
The response exposes no signal evidenceRefs / triggerAttemptIds / answers / dueAt / thresholds — only lesson +
skill identity (e2e §68).

## 21. Query batching (§41)
Generation = existing roadmap read/selection + ONE batched `getCandidates` (1.9A O(few)) + pure selection. Read
projection adds one batched skill-name lookup for EXTRA items. No N+1.

## 22. Tests
- Unit: `daily-review-extra-selection.spec.ts` (9) — empty, same-Topic, cross-Topic excluded, core dedup, one cap,
  direct-trigger priority, reason priority, exposure priority, stable tie / order-independence.
- E2e: `test/daily-plan-review.e2e-spec.ts` (10) — same-Topic EXTRA (+ no session, no leak); cross-Topic excluded;
  core dedup; one cap; no-signal → 0 EXTRA; core done ignores EXTRA; same-day immutable + no-late-EXTRA; GET
  read-only + side-effect boundary; concurrent generation → one plan/≤1 EXTRA; manual cross-topic still startable.
- Totals: unit **315**, e2e **226**.

## 23. Future workload policy (§74)
OPEN: >1 automatic review/day, time/minute budget, personalized workload, cross-topic automatic review, opt-out,
manual EXTRA choice, DailyPlanItem→ReviewSession completion provenance, days-overdue prioritization, rewards,
notifications, review streaks, AI planning. A future policy ships as daily-review-extra-v2.
