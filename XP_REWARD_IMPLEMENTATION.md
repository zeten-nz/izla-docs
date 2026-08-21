# Daily Mission XP Reward — Implementation (Phase 2.0C-2)

> Status: **COMPLETE** (2026-08-20). Closes the Phase 2.0C architecture gap. A `DailyMissionCompletion` earns XP
> **exactly once** through the accepted `XpGrant` model (TD-45): deterministic policy, typed mission provenance,
> append-only, one grant per completion, immutable policy snapshot, retry/concurrency-safe, historically
> reconcilable. **No IZL. No RewardGrant writes. No XpBalance writes. No levels/streaks/badges.**
> Owner: **TD-140/141/142/143/144**. Migration `20260820180000_add_xp_grant_mission_provenance`.

Code: `backend/src/xp/` — `policy/daily-mission-xp.policy.ts` (pure), `xp.repository.ts`, `xp.service.ts`,
`xp.controller.ts`, `xp.module.ts`. Bridge in `daily-mission.service.ts`.

## 1. Phase 2.0C gap and closure
Phase 2.0C stopped (PASS WITH ARCHITECTURE GAP) because its premise — "XP is granted via `RewardGrant`" — is
inverted relative to the accepted schema. `RewardGrant` is the **IZL/financial** vehicle (`subscription_cycle_id`
NOT NULL, `amount` = IZL, 1:1 `IZLLedgerEntry`, `RewardPolicyVersion`). Owner review confirmed the correct vehicle
is the already-accepted **`XpGrant`** (TD-45). Phase 2.0C-2 hardens `XpGrant` minimally and implements XP accounting.

## 2. Accepted XP / IZL model separation (TD-45 / TD-142)
Two separate models, deliberately at different rigor levels:
- **XP** → `XpGrant` (light append-only history) + `XpBalance` (deferred cache: `total_xp`, `current_level`).
- **IZL** → `RewardGrant` (cycle-scoped earning) → `IZLLedgerEntry` (signed ledger) → `IZLWallet`.

They never share a table or a `rewardKind`. There is no XP↔IZL conversion or ratio.

## 3. Why RewardGrant was rejected for XP
`RewardGrant.subscription_cycle_id` is NOT NULL (Restrict FK) — a free/no-subscription learner has no
`SubscriptionCycle`, so RewardGrant literally cannot hold their mission XP. Its `amount` is IZL and it 1:1-links an
IZL ledger entry. Using it for XP would require nullable cycle + amount redefinition + a `rewardKind` discriminator
— merging XP into the IZL vehicle, violating TD-45 and the XP≠IZL mandate. Rejected; RewardGrant is unchanged.

## 4. XpGrant authority (TD-143)
`XpGrant` is the current XP source of truth. `totalXp = SUM(XpGrant.amount)` over the full append-only history
(including any correction rows). No second XP authority is introduced.

## 5. XpBalance deferred boundary (TD-143 / §46)
In Phase 2.0C-2 `XpBalance` was a **deferred projection/cache** — never written; `totalXp` derived from `XpGrant`.
**Phase 2.0D activates it** as a rebuildable projection (`totalXp`, `currentLevel`, `progressionVersionCode`),
written only by the XP projection module. `XpGrant` remains the source of truth; the learner-facing read still
derives canonical values from `SUM(XpGrant.amount)`. See [XP_PROGRESSION_IMPLEMENTATION.md](XP_PROGRESSION_IMPLEMENTATION.md).

## 6. DailyMissionCompletion provenance (§18)
`DailyMissionCompletion` is the ONLY mission-XP eligibility authority. XP code never re-reads raw ActivityAttempt
score / activity type / answer / DailyPlan / ReviewSession. Mission semantics were normalized in Phase 2.0B.
Required chain: **ActivityAttempt → DailyMissionCompletion → XpGrant** (never ActivityAttempt → XpGrant).

## 7. Typed FK (TD-141 / XP-DB-02)
`XpGrant.dailyMissionCompletionId` (nullable, `onDelete: Restrict`) → `DailyMissionCompletion` is the load-bearing
provenance for mission XP. `DailyMissionCompletion.xpGrants @relation("XpGrantMission")` is the reverse relation —
kept **distinct** from `rewardGrants` (the IZL `RewardGrantMission`), so XP and IZL relations are visibly separate.

## 8. policyVersionCode (TD-141 / XP-DB-03)
`XpGrant.policyVersionCode` (nullable String) persists `daily-mission-xp-reward-v1` as an immutable **code string**.
It is **not** related to the IZL `RewardPolicyVersion` model (Int-versioned, cycle-linked, staff config — a
fundamentally different, financial policy authority). A CHECK enforces non-empty when present; non-mission rows keep
it NULL.

## 9. reasonCode (§11)
Mission XP uses `reason_code = DAILY_MISSION` (the existing registry category). Three fields carry distinct meaning:
`reasonCode` = source category, `policyVersionCode` = XP policy contract, `dailyMissionCompletionId` = exact
provenance.

## 10. XP policy v1 (TD-140)
`daily-mission-xp-reward-v1`, an immutable mapping keyed by **both** mission code and mission producer version
(§16). Pure `evaluateDailyMissionXp({missionCode, missionPolicyVersion})` → `{eligible, amount, reasonCode,
policyVersionCode}` or `{eligible:false}`. No Prisma/HTTP/Clock/ActivityAttempt interpretation.

## 11. Amounts (§15)
| Mission (code + producer version) | XP |
|---|---|
| LEARN_TODAY + learn-today-mission-v1 | **10** |
| MASTERY_TEST_90 + mastery-test-90-mission-v1 | **20** |
| anything else | **no grant** (no default) |

## 12. Mission producer-version eligibility (§16)
Reward is keyed by the *pair*, not by code alone. A future `learn-today-mission-v2` would **not** inherit the v1
amount — it requires an explicit reward-policy update. This prevents an unknown future producer from silently
reusing v1 economics.

## 13. Append-only grants (§24 / XP-07)
The mission producer only `create`s XpGrant — never UPDATE/DELETE. A grant's amount is never edited in place. Any
future accounting correction uses the accepted `XpGrant` ±/correction semantics in a separate admin/audit phase —
not here.

## 14. Idempotency (XP-DB-01 / §8)
Partial UNIQUE `uq_xp_grant_mission_completion (daily_mission_completion_id) WHERE NOT NULL` is the DB authority:
one XP grant per completion. `createMissionXpGrant` catches P2002 → returns false. Policy version is **not** part of
uniqueness (a v2 policy must not double-pay one historical completion — completionId is entitlement identity,
policyVersionCode is provenance only, §9).

## 15. Concurrency (§56–59)
Repeated `ensureMissionXpGranted`, concurrent bridge/reconcile calls, and concurrent mission-completion races all
converge to one grant per completion via XP-DB-01 (P2002-skip). Two *different* completions grant independently
(no false collision).

## 16. Automatic bridge (§19 / §25)
After `DailyMissionService` creates or resolves the day's completions (in `evaluateActivityAttempt` and
`reconcileToday`), it calls `XpService.tryEnsureMissionXpGranted(userId, completionId)` for each. The **XP module is
the single XpGrant writer** — `DailyMissionService` never Prisma-writes XpGrant (dependency is one-way: DailyMission
→ Xp; the XP module reads `DailyMissionCompletion` via its own repository, so no cycle).

## 17. Failure boundary (§27 / TD-144)
Authoritative order: ActivityAttempt → DailyMissionCompletion → XpGrant. `tryEnsureMissionXpGranted` swallows
transient failures (logs a warn) so an XP failure never rolls back the attempt or completion; reconcile repairs it.
A cross-user provenance mismatch is not transient — it surfaces as `XpRewardConfigurationInvalidError`.

## 18. Historical reconcile (§35–39 / TD-144)
`POST /api/xp/me/reconcile` (own-user, no body) materializes missing mission XP across **all** history (not
today-only — a Day-A failure may surface on Day C). It queries `DailyMissionCompletion WHERE userId = principal AND
missionCode ∈ supported AND xpGrants none` (batched relation query, **no ActivityAttempt scan**, no N+1), ordered
`completedAt ASC, id ASC` (§38). Each row is re-validated by the pure policy; unsupported producer versions are
skipped safely (§39). Deterministic: amounts derive from the frozen `DailyMissionCompletion.{missionCode,
policyVersion}` — no processing-time semantics.

## 19. Total XP derivation (§30 / TD-143)
`totalXp = SUM(XpGrant.amount) WHERE userId = principal`, integer aggregation, null → 0. It sums the **whole** XP
history (not only mission rows), so any accepted correction row is reflected (§31).

## 20. Correction rows (§63)
Because total reads all `XpGrant`, a `+30` mission history plus a `-5` generic correction yields `totalXp = 25`.
This phase exposes no correction-creation API; corrections are a future admin concern.

## 21. GET XP (§33 / §34)
`GET /api/xp/me` → `{ totalXp }`. Read-only (never writes XpBalance). Zero grants → `200 {totalXp: 0}` (not 404).
No `currentLevel`/rank/streak/badge fields.

## 22. Security (§68–71)
Own-user only via `CurrentPrincipal`; no userId path/param. Unauthenticated → 401 (global AuthGuard). No endpoint
accepts `xpAmount`/`missionCompletionId`/`policyVersionCode`/`reasonCode` — all server-derived (§42/§70). Minimal
projection; no `answer`/`payload`/`answerKey`/evidence/IZL internals ever leak (§71). Cross-user provenance is
rejected in the service (XP-02).

## 23. No manual claim (§41 / §42)
No `POST /claim`, no `/missions/:id/reward`, no `POST /xp/grants`. XP is automatically materialized from the
authoritative mission completion (or reconciled). The learner never chooses when/how much XP.

## 24. Zero IZL boundary (§43 / TD-142)
RewardGrant / IZLLedgerEntry / IZLWallet / IZLRedemption / SubscriptionCycle / Payment writes: **ZERO** (grep +
e2e counts). `RewardGrant` and `RewardPolicyVersion` schemas are unchanged. No XP→IZL numeric relationship.

## 25. Schema & DB invariants
Migration 12 (`20260820180000_add_xp_grant_mission_provenance`) = Prisma DDL (+2 nullable columns + FK Restrict) +
custom SQL: partial UNIQUE XP-DB-01, CHECK XP-DB-03 (policy nonempty when present), CHECK XP-DB-04 (mission-backed
amount > 0 — non-mission rows keep ±/correction semantics; no broad amount CHECK). Migration count → **12**;
31 CHECK / 20 partial unique; drift clean (dev+test); empty-diff schema match. See
[DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) §9i (XP-DB-01..04, XP-01..13). Test resets that delete
`daily_mission_completion` now delete `xp_grant` first (Restrict FK) — applied to all 7 mission-touching e2e resets.

## 26. Tests & future XP progression
- **Unit** (`policy/daily-mission-xp.policy.spec.ts`, 5): 10/20 mapping, unknown code, unknown producer version,
  cross-paired code+version.
- **e2e** (`test/xp.e2e-spec.ts`, 14, §34/43/47–69): LEARN_TODAY→10, MASTERY→both (30), attempt-farming→one grant,
  mastery retry→one 20, next-day 30+30=60, unsupported code/version→0, reconcile repair+idempotency, historical
  reconcile, concurrent reconcile, correction-inclusive total, XpBalance/RewardGrant/IZL non-mutation, cross-user
  reconcile, zero-state/401/no-leak, network replay.
- Gate: **328 unit + 251 e2e PASS**; `tsc` clean; drift clean.
- **Earning time** (§62): `XpGrant.createdAt` is physical insertion (may be delayed on reconcile); the historical
  earning time is `dailyMissionCompletionId → DailyMissionCompletion.completedAt` — no duplicated timestamp column
  was added. A future XP history API can expose `earnedAt` via that relation.
- **Future (deferred):** XpBalance activation/rebuild, `currentLevel` formula, level curve, badges, streaks, XP
  history UX, admin corrections, additional XP sources, IZL rewards — see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
