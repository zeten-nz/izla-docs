# Daily Plan Foundation — Implementation (Phase 1.7A)

> Status: **COMPLETE** (2026-08-20). A profile-local-day snapshot of the learner's ACTIVE roadmap focused
> on exactly one Topic. **No schema change / no migration** — the accepted `DailyPlan`/`DailyPlanItem`
> schema (localDate, timezoneSnapshot, generationNo, CURRENT status, `roadmapItemId`, section/itemType,
> one-CURRENT + unique-generation invariants) was sufficient. No Lesson execution, rewards, missions, or AI.

Code: `backend/src/daily-plan/**` + `src/common/clock.ts`.

## 1. Scope
Authenticated learner can generate-or-get today's plan and read today's plan. The plan is anchored to the
profile timezone/local date, contains exactly one Topic, classifies items as MUST_DO / RECOMMENDED, returns
the same immutable snapshot all local day, and never advances into a second Topic the same day.

## 2. DailyPlan authority (TD-103)
`daily-plan-roadmap-v1`: a Daily Plan is a **local-day snapshot of the current ACTIVE roadmap, one Topic**.
It is NOT a second/replacement roadmap, mastery engine, reward engine, AI recommendation, or lesson-progress
authority. Item membership/priority is snapshot; executability is derived live from roadmap state.

## 3. localDate (§6)
"Today" = the calendar date in `UserProfile.timezone`, never server UTC. `localDateInTimezone(clock.now(),
tz)` via Intl (full ICU). Stored in `DailyPlan.localDate` (@db.Date, UTC-midnight). Client never sends date.

## 4. timezoneSnapshot (§8, TD-91)
`DailyPlan.timezoneSnapshot = UserProfile.timezone` at generation. After creation, `localDate` +
`timezoneSnapshot` are immutable — a later profile-timezone change does NOT rewrite an existing plan; future
generations use the new timezone (tested §38).

## 5. Clock boundary (§7)
Injectable `Clock.now()` — domain logic never scatters `new Date()`. Overridden with a fixed/advanceable
clock in tests (local-date, midnight, timezone, concurrency reproducibility).

## 6. Version / generation (§30/31)
`DailyPlanStatus` = CURRENT / SUPERSEDED. `generationNo` = `max(existing for user+localDate)+1` (1 for the
first plan of the day). Same-day retry returns the existing CURRENT plan — no increment, no supersede. No
regenerate endpoint (§32); superseding is OPEN.

## 7. One Topic per day (TD-104, §4)
Topic = the one containing the roadmap `nextItem` (Phase 1.6B). The plan is not independently re-planned.
Application invariant: every roadmap-derived item resolves to the same Topic (fail-safe, tested §44).

## 8. Topic selection (§4/5)
Load the ACTIVE roadmap read model (`RoadmapReadService`), take its `nextItemId` (earliest IN_PROGRESS, else
earliest AVAILABLE — 1.6B authority), resolve `nextItem.lesson → topicId`. DailyPlan re-derives no
gap/prerequisite/completion logic. Roadmap source = the learner's ACTIVE roadmap (earliest if multiple —
primary/default Subject is OPEN; single-roadmap is the current reality).

## 9. MUST_DO (§14, TD-105)
Exactly one MUST_DO = the roadmap `nextItem` (IN_PROGRESS or AVAILABLE, i.e. executable). If `nextItemId`
is null → no plan is created (`DAILY_PLAN_NO_EXECUTABLE_CONTENT`), never a BLOCKED/UNAVAILABLE MUST_DO.

## 10. RECOMMENDED (§15/16/17)
All remaining unfinished roadmap items **from the same Topic** after the MUST_DO position, in canonical
roadmap-position order. Included regardless of live state AVAILABLE/BLOCKED/IN_PROGRESS (a BLOCKED item is a
day-scope item, not a broken button — tested §46). Excluded: COMPLETED, UNAVAILABLE, other Topics, other
roadmaps, items before MUST_DO, non-roadmap lessons.

## 11. EXTRA boundary (§18)
Phase 1.7A auto-generates **zero** EXTRA items (tested §49). The `EXTRA` section stays supported for future
review/practice/missions/gamification — no selection policy invented now. Expected shape: 1 MUST_DO + 0..N
RECOMMENDED + 0 EXTRA. No workload/time cap (§19).

## 12. Snapshot immutability (§9)
After generation the DailyPlanItem set is a historical snapshot — never added to/removed from during the day
because a lesson completed, roadmap state changed, an item became available, or content changed.

## 13. Live roadmap states (§26)
On read, each item's state is DERIVED (COMPLETED/UNAVAILABLE/IN_PROGRESS/BLOCKED/AVAILABLE) by reusing the
Phase 1.6B pure `deriveItemState` state machine + `RoadmapRepository` batch loads — not persisted, not a
duplicated state machine (§27). A BLOCKED RECOMMENDED item becomes AVAILABLE live after its prerequisite
completes, with no regeneration (tested §46).

## 14. Same-day idempotency (§10/11/39)
`POST /today` when a CURRENT plan already exists for the local date returns it unchanged — even if all its
lessons are completed (finished-early). No second Topic, no generationNo increment, no second plan (tested
§39). Core enforcement of one-Topic-per-day.

## 15. Finished-early policy (§11)
Completing today's planned work early does NOT open another Topic the same local day. Extras/review for early
finishers is future work.

## 16. Next-day behavior (§12/40/41)
A new local date may generate a new plan reading current roadmap state. If the roadmap `nextItem` is still in
today's Topic, the new plan legitimately reuses that Topic (tested §40); once this Topic's roadmap items are
completed, the next local day moves to the next Topic (tested §41). One-Topic-per-day ≠ one-day-per-Topic.

## 17. Current-plan lifecycle (§30/33)
One CURRENT plan per (user, localDate) — DB-enforced by `ux_current_daily_plan` (L16). `unique(userId,
localDate, generationNo)` also enforced. These are the concurrency authority.

## 18. Atomic persistence (§58)
`DailyPlan` header + all `DailyPlanItem`s in one transaction — never a partial CURRENT plan.

## 19. Concurrency (§35)
Two concurrent `POST /today` → one CURRENT plan; the loser catches the `ux_current_daily_plan` P2002 and
returns the winner (same id, no duplicate items — tested §35). DB constraint is final authority.

## 20. APIs (§50/51)
| Method | Path | Notes |
|---|---|---|
| POST | `/api/daily-plans/today` | generate-or-return today's CURRENT plan; no body; all server-derived |
| GET  | `/api/daily-plans/today` | read today's CURRENT plan; **never generates/mutates**; 404 `DAILY_PLAN_NOT_FOUND` |

Errors: `DAILY_PLAN_NOT_FOUND` (404), `DAILY_PLAN_NO_EXECUTABLE_CONTENT` (409 — no roadmap nextItem, incl.
all-completed-but-not-reconciled, §24), `DAILY_PLAN_CONFIGURATION_INVALID` (409), `ROADMAP_NOT_FOUND` (404 —
no ACTIVE roadmap). No GET-by-date/by-id endpoints (§52/53). Response shape §54: `{id, localDate, timezone,
generationNo, status, topic:{id,title}, done, progress:{total,completed,progressBp}, items:[{id, kind,
position, state, lesson:{id,title}}]}`. No draft content / assessment answers / gap priority / PII exposed.

## 21. Security (§68)
All own-user (`/today` derives from `principal.userId`; no target id, no body). No public plan, no shared
cache. No sensitive assessment/profile leakage; UNAVAILABLE items expose no title.

## 22. Tests (unit 12, e2e 11; project total unit 195 / e2e 126)
- **Unit:** local-date (timezone-independent §37, midnight §36, not-server-UTC, round-trip); plan-selection
  (MUST_DO=nextItem §42, nextItem-priority §43, same-Topic RECOMMENDED §44, completed-exclusion §45, BLOCKED
  included §46, UNAVAILABLE excluded, null cases).
- **E2e (fixed clock):** one-Topic snapshot + no EXTRA + no side-effects; same-day idempotency/finish-early
  §39; concurrent §35; GET-no-generation §50; multi-day same Topic §40; next Topic next day §41; timezone
  immutability §38; BLOCKED→AVAILABLE live §46; archived→UNAVAILABLE + completed-then-archived §47/48;
  all-completed-not-reconciled → 409 (no hidden reconcile) §24; no-active-roadmap + auth.
- Regression: all prior phases green.

## 23. Lesson boundary (§60/61)
Generation creates NO `LearnerLessonProgress` (MUST_DO ≠ started) and NO `LearnerLessonCompletion`. Existing
rows are read only.

## 24. Reward boundary (§64)
No XP / IZL / RewardGrant / RewardPolicyVersion. `DailyMissionCompletion` untouched.

## 25. Daily Mission boundary (§65)
No special tasks (library/community/login/etc.). DailyPlan v1 handles only roadmap learning work.

## 26. AI boundary (§66)
No AI planning, no LLM, no generated "today focus" prose. Deterministic roadmap snapshot only.

## Side-effect verification
Roadmap / LearnerLessonProgress / LearnerLessonCompletion / LearnerSkillState / SkillMeasurement /
DailyMission / Rewards / XP / IZL / AI — **NONE** written (grep-confirmed: daily-plan writes only `dailyPlan`
+ `dailyPlanItem`; asserted absent in tests).

## Remaining OPEN — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)
Daily workload/time budget · scheduling preferences · manual regeneration · superseding plans · extra-practice
selection · Daily Missions · reward rules · same-day optional practice after completion · lesson execution ·
lesson-revision attempt pinning · automatic roadmap-reconciliation hook · notifications · AI planner ·
primary/default Subject (multi-roadmap selection).
