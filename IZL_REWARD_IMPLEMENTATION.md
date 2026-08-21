# Daily Mission IZL Economic Reward — Implementation (Phase 2.1A)

> Status: **COMPLETE** (2026-08-20). The first **real-value** earning producer: a supported
> `DailyMissionCompletion` may become eligible for **IZL**, materialized as an atomic `RewardGrant` +
> `IZLLedgerEntry` within the completion's **historical SubscriptionCycle** and that cycle's **snapshotted**
> `RewardPolicyVersion`, under daily/cycle caps + per-user concurrency locking. The ledger is the canonical IZL
> accounting truth. **No redemption. No reservations. No payments. No wallet activation. No XP changes.**
> Owner: **TD-150/151/152/153/154/155**. **No schema change** (existing finance invariants sufficed).

Code: `backend/src/finance/` — `reward/daily-mission-izl.policy.ts` (pure), `reward/reward.repository.ts`,
`reward/daily-mission-izl.service.ts`, `finance.controller.ts`, `finance.module.ts`. Bridge in `daily-mission.service.ts`.

## 1. IZL vs XP
Two independent accounting domains: **XP** (`XpGrant`/`XpBalance`, gamification, non-monetary) and **IZL**
(`RewardGrant` → `IZLLedgerEntry`, real-value). One `MASTERY_TEST_90` completion yields both (10+20 XP **and** 1 IZL)
via independent bridges. No XP↔IZL conversion or ratio.

## 2. Real-value boundary
IZL requires financial-grade accounting: cycle-scoped policy provenance, immutable snapshots, atomic ledger
posting, caps, and concurrency serialization — far stricter than XP's lightweight append-only history.

## 3. DailyMissionCompletion eligibility
`DailyMissionCompletion` (Phase 2.0B) is the normalized learning evidence and the only IZL eligibility source (never
raw ActivityAttempt). It is **not** an economic entitlement — entitlement exists only after `RewardGrant.status =
GRANTED` with a matching ledger credit.

## 4. RewardGrant semantics
Reused as-is (the accepted IZL vehicle): `category = DAILY_MISSION`, `amount` = IZL, `missionCompletionId` (Restrict
provenance), `subscriptionCycleId`, `rewardPolicyVersionId`, `dedupKey`, `status = GRANTED`. Schema unchanged.

## 5. Ledger authority (TD-150)
`IZLLedgerEntry` is the canonical IZL accounting truth: append-only, signed, `entryNo` monotonic per user,
`balanceAfter` running balance. `balanceIzl = SUM(amount)`. RewardGrant is reward provenance, not the balance.

## 6. Economic atomicity (TD-150 / §31 / §39)
`RewardGrant` and its `EARN` `IZLLedgerEntry` are created in **one** transaction — both or neither. `IZLLedgerEntry.
rewardGrantId` is `@unique` (1:1). Forbidden states (grant without ledger, ledger without grant) cannot occur; a
ledger-insert failure rolls back the grant (distinct from rolling back upstream mission evidence).

## 7. Subscription cycle authority (TD-152 / §10)
The covering cycle is `where subscription.userId = user AND periodStart <= completedAt < periodEnd`. Time authority
is the frozen `DailyMissionCompletion.completedAt` — never processing time or the current cycle. `SubscriptionCycle
Status` ∈ {ACTIVE, COMPLETED}; both are valid historical context (a completed/ended cycle is still eligible, §11).
Zero covering cycles → no grant (mission/XP unaffected). Multiple covering cycles (corruption) → integrity error,
no grant (§13).

## 8. Policy snapshot (TD-152 / §15)
The cycle's snapshotted `rewardPolicyVersionId` is the policy authority. Reward materialization reads
`cycle.policyVersion.config` — **never** the current ACTIVE policy. A later, richer ACTIVE policy does not rewrite a
historical cycle's rewards (e2e §63: old cycle keeps 1 IZL even when a newer policy says 2).

## 9. Policy config v1 (TD-151 / §7)
`RewardPolicyVersion.config` schema `izl-reward-policy/v1` (distinct from the row's Int `version`):
```
{ "schemaVersion": "izl-reward-policy/v1",
  "dailyMissionRewards": { "MASTERY_TEST_90": { "missionPolicyVersion": "mastery-test-90-mission-v1", "amountIzl": 1 } },
  "caps": { "dailyMissionIzlPerLocalDate": 1, "dailyMissionIzlPerCycle": 30 } }
```
Strict TD-92 parser (`parseIzlRewardPolicyConfig`): validates schemaVersion, mission entries (`missionPolicyVersion`
non-empty, `amountIzl` positive int ≤ daily cap), caps (non-negative ints). Malformed → `IzlRewardConfigError`. No
generic reward DSL (§9).

## 10. MASTERY_TEST_90 = 1 IZL (TD-151 / §4)
Only `MASTERY_TEST_90 + mastery-test-90-mission-v1` is rewardable in v1 → 1 IZL. Unknown code / producer version /
schema → no grant, no default.

## 11. LEARN_TODAY = 0 IZL (§4 / §58)
LEARN_TODAY requires only one objective attempt (a wrong one counts) — valid behavioral XP evidence, but too weak
for real-value issuance. It earns zero IZL (absent from the policy → not eligible).

## 12. Daily cap (TD-153 / §21)
Max 1 IZL of `DAILY_MISSION` per (user, cycle, `DailyMissionCompletion.localDate`). The frozen mission localDate is
used (never recomputed from the current timezone). With the current catalog (one MASTERY per local day) the daily
cap is structurally bounded; the parser also rejects any `amountIzl` above the daily cap.

## 13. Cycle cap (TD-153 / §23)
Max 30 IZL of `DAILY_MISSION` per `SubscriptionCycle`. Other reward categories do not consume this DAILY_MISSION cap.
No global lifetime cap. The cap resets by cycle identity (e2e §66). No global earned/ceiling cache is touched
(cycle `earnedIzl`/`rewardCeilingIzl` is the separate subscription-level ceiling — out of scope; the cycle is not
mutated, §54).

## 14. Anti-farming v1 (§27)
Layers: (1) mission qualification; (2) one completion per mission/localDate (DM-DB-01); (3) MASTERY_TEST_90-only;
(4) exact producer version; (5) one grant per completion (dedup); (6) daily cap 1; (7) cycle cap 30; (8) valid
cycle required. No device/IP/AI/first-attempt/manual-review layers in v1.

## 15. Posting-time cap semantics (TD-153 / §25 / §43)
Cap consumption is measured from already-**GRANTED** `RewardGrant` rows (category DAILY_MISSION), not from attempts/
completions/XP/wallet. A completion is only a candidate; entitlement is decided when the grant is atomically posted.
All-or-nothing (cap exhausted → no grant, no partial, no pending entitlement). Reconcile processes candidates in
historical order and never revokes/reorders existing grants (no retroactive rebalance).

## 16. Dedup (§18 / IZL-DB-01)
`dedupKey = daily-mission-izl:<completionId>` — entitlement identity is the completion + IZL producer (not the
policy version, so v1/v2 cannot double-pay one historical completion). Enforced by the existing F-5
`unique(userId, dedupKey)`. Client never supplies it.

## 17. Concurrency (TD-153 / §29 / §69)
A per-user transaction-scoped advisory lock (`pg_advisory_xact_lock(hashtext('izl'), hashtext(userId))`) serializes
cap check + posting and the ledger `entryNo`. Inside the lock: reload cycle/policy, check existing grant, compute
fresh cap usage, then atomically post. Two completions racing for the final cap slot → the second sees updated usage
→ no grant (e2e §68/§69).

## 18. Failure boundary (TD-154 / §38)
Upstream ActivityAttempt → DailyMissionCompletion; downstream **independent** branches XP and IZL. An IZL failure
never rolls back the attempt / completion / XP (`tryEnsureMissionReward` is non-throwing). The economic posting
itself is atomic (§6). XP and IZL do not gate each other (e2e §71: no cycle → IZL no-op, XP still 30).

## 19. Automatic bridge (§37 / §85)
After `DailyMissionService` materializes the day's completions, `bridgeMissionRewards` calls the XP bridge **and**
`DailyMissionIzlService.tryEnsureMissionReward` per completion — independently. The **Finance module is the only
RewardGrant + IZLLedgerEntry writer**; DailyMission never Prisma-writes finance rows (one-way DailyMission → Finance;
Finance reads DailyMissionCompletion/SubscriptionCycle via its own repository — no cycle).

## 20. Historical reconcile (§40–44)
`POST /api/izl/me/reconcile` (own-user, no body): scans the learner's rewardable `DailyMissionCompletion` (code ∈
supported, no reward grant), ordered `completedAt ASC, id ASC` (no ActivityAttempt scan), and posts missing IZL —
each within its historical cycle + snapshot policy under the lock. Per-completion config/integrity errors are
skipped so other valid completions still post (§44). Returns `{balanceIzl, grantsCreated}`. Idempotent.

## 21. GET IZL (§46 / §47)
`GET /api/izl/me` → `{balanceIzl}` = signed SUM over the ledger. Zero entries → `200 {balanceIzl: 0}` (not 404). No
fiat value / exchange rate. Read-only. **(Phase 2.1B extends this to `{balanceIzl, reservedIzl, availableIzl}` and
activates the wallet projection + reservation hold primitive — see
[IZL_WALLET_RESERVATION_IMPLEMENTATION.md](IZL_WALLET_RESERVATION_IMPLEMENTATION.md).)**

## 22. Wallet boundary (§48 / §49)
`IZLWallet` is **not** written. The ledger is canonical; `entryNo`/`balanceAfter` are derived from the ledger
(`MAX(entryNo)+1`, `SUM(amount)+amount`) inside the lock. No FK forces a wallet row. Wallet activation (available/
reserved projection) is deferred to a later phase.

## 23. No redemption (§50 / §51 / §53)
No reservations, redemption, withdraw, cashout, spend, transfer, or gift. IZL can be earned and read only. Its
cash/redemption value remains OPEN.

## 24. No payments (§52)
No PaymentOrder/Click/Payme/callbacks. Reward issuance is independent of buying/subscribing. No Subscription/cycle/
entitlement mutation (§54) — the cycle is read as historical context only.

## 25. Security (§77–79)
Own-user only via `CurrentPrincipal`; 401 unauthenticated. No endpoint accepts amount/cycle/policy/completion/dedup/
category/ledger fields — all server-derived. Cross-user provenance is rejected (`RewardConfigurationInvalidError`);
another learner's reconcile never posts a victim's reward (e2e §77). No `answer`/`payload`/`config`/`dedupKey`/
evidence leak (§79).

## 26. Tests
- **Unit** (`reward/daily-mission-izl.policy.spec.ts`, 6): strict config parse (valid; unknown schema/missing caps/
  bad cap/bad amount/amount>cap/null); eligibility (MASTERY→1, LEARN_TODAY→not eligible, unknown version→not eligible).
- **e2e** (`test/izl-reward.e2e-spec.ts`, 14, §58–79): automatic MASTERY→1 IZL + EARN ledger; LEARN_TODAY→0;
  unknown version→0; no cycle→0; historical cycle (grant on Cycle A not current); policy snapshot (old cycle keeps
  1 vs newer ACTIVE 2); cycle cap (cap 3 → 3 grants, 4th refused); next-cycle reset; dedup + reconcile idempotency;
  concurrent reconcile → one grant/ledger; audit chain 1:1 grant↔ledger + running balance; IZL failure doesn't break
  XP; signed ledger read (earn + correction); GET read-only/zero-state/IDOR/401/no-leak.
- Gate: **345 unit + 274 e2e PASS**; `tsc` clean; **no schema change** (13 migrations), drift clean.

## 27. Future reservation / redemption phase
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): IZL cash/redemption value + catalog, withdrawal/cashout,
reservation flow, wallet activation (available vs reserved), cycle cap v2, plan-specific rates/caps, additional IZL
mission/lesson/review rewards, first-attempt-only real-value rules, fraud/risk engine, holds, reversals, staff
corrections, dispute handling, ledger audit tooling, reward notifications, subscription multipliers, tax/accounting.
**Next: owner review — Phase 2.1B (IZL Wallet / Reservation Foundation).**

> **Phase 2.1G-D update (2026-08-21):** two earning changes (TD-198/199, see
> [PAYMENT_FINALIZATION_CONTRACT.md](PAYMENT_FINALIZATION_CONTRACT.md)). (1) A **reward-disabled** SubscriptionCycle
> (`rewardPolicyVersionId = NULL`) now makes `materializeMissionReward` a no-op — no grant, no ledger, no error (the
> mission + XP branch is never rolled back). (2) The effective per-cycle cap is now `min(policyDefinedCycleCap,
> SubscriptionCycle.rewardCeilingIzl)` — the cycle economic ceiling is load-bearing. Authoritative consumption is still
> `SUM(GRANTED RewardGrant per cycle)`; `earnedIzl` remains non-authoritative (no incremental update).
