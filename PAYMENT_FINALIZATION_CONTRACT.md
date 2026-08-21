# Verified Payment Finalization — Contract + Schema Hardening (Phase 2.1G-D)

> Status: **COMPLETE** (2026-08-21). Prepares every input + economic invariant the future Phase 2.1G finalizer needs,
> **without** finalizing anything: NO PaymentOrder PAID, NO Subscription/Cycle creation, NO IZL REDEEM / reservation
> CONSUMED / redemption APPLIED producer, NO provider call. Owner: **TD-195..204**. Migration
> `20260821000000_finalization_schema_hardening` (migration 20). Behavioral change (only): 2.1A earning now enforces the
> cycle economic ceiling + no-ops a reward-disabled cycle.

Code: `backend/prisma/schema` (PlanPrice.billingPeriodMonths, SubscriptionCycle reward nullable, IzlReservationStatus
CONSUMED), `src/common/calendar-month.ts`, `src/finance/reward/reward-ceiling.ts`, `src/finance/reward/reward.repository.ts`
(disabled no-op + ceiling). Incidental fix: `src/lesson-execution/lesson-execution.service.ts` now stamps `submittedAt`
from the injected `Clock` (see §24).

## 1. Trusted evidence input
The future finalizer consumes an existing `PaymentTransaction.status = SUCCEEDED` (with its `paymentOrderId`, `amount`,
`provider`, `providerTransactionId`, `confirmedAt`). It never accepts provider/amount/status/paidAt/userId from a
client, and never re-calls Click/Payme.

## 2. Why finalization stays separate
Provider verification (2.1F) and internal economic finalization (2.1G) are decoupled so a verified payment is durable
evidence even if a finalization dependency (config, policy, rate) is momentarily missing — the payment is recoverable,
never stranded.

## 3. Billing duration authority (TD-195)
Commercial duration is the immutable `PlanPrice.billingPeriodMonths` (amount + duration belong to one historical offer;
`PaymentOrder.planPriceId` freezes it). CHECK `> 0` (FP-DB-01). Existing v1 rows backfilled to **1 calendar month**.
A future duration change is a new PlanPrice version, never an in-place edit.

## 4. Calendar-month semantics (§48)
`addCalendarMonths(start, months)` (pure, UTC, end-of-month clamping, time-of-day preserved): `2026-01-31 +1 →
2026-02-28`, `2028-01-31 +1 → 2028-02-29`, `2026-03-31 +1 → 2026-04-30`. Unit-tested; no cycle created here.

## 5. periodStart (future, §35)
`SubscriptionCycle.periodStart = PaymentTransaction.confirmedAt`. Internal finalization latency never shortens access.

## 6. periodEnd (future, §36)
`periodEnd = addCalendarMonths(periodStart, PaymentOrder.planPrice.billingPeriodMonths)` using the frozen
`PaymentOrder.planPriceId` (never the current price). DB CHECK `period_end > period_start` already exists.

## 7. Expiry after success (TD-196)
`PaymentOrder.expiresAt` gates only *new initiation* (2.1E). Once a `SUCCEEDED` transaction exists, an elapsed expiry
MUST NOT block internal finalization — paid money stays finalizable. No `PaymentOrder.paidAt`; the payment timestamp is
the unique SUCCEEDED `confirmedAt`.

## 8. Subscription episode semantics (TD-200 / F-14)
`Subscription` is one continuous membership episode (TD-87), enforced by `ux_nonterminal_subscription (user_id) WHERE
status IN ('ACTIVE','EXPIRED')`. Statuses: ACTIVE / EXPIRED / CANCELLED.

## 9. ACTIVE conflict (TD-200 / §26 / §27)
A finalized `SUBSCRIPTION_PURCHASE` when an **ACTIVE** subscription already exists → a **recoverable
subscription-conflict**: PT stays SUCCEEDED, order stays PENDING, no partial effects. Silently converting a purchase to
a renewal would invent commercial semantics; SUBSCRIPTION_RENEWAL is separate future scope.

## 10. EXPIRED reactivation (TD-200)
An **EXPIRED** episode is reactivated in place (EXPIRED → ACTIVE) with a new cycle in the *same* subscription; only
CANCELLED history starts a new subscription record.

## 11. Cycle commercial snapshots (§37)
Frozen from the order, never re-priced: `planId`/`planPriceId` ← order; `grossPriceUzs` ← `grossAmount`; `discountUzs`
← `izlDiscountAmount`; `paidAmountUzs` ← `payableAmount`; `rewardBasisUzs` ← `payableAmount`.

## 12. Reward basis (TD-197)
`rewardBasisUzs = PaymentOrder.payableAmount` (net actually-paid), **not gross** — IZL-discounted value not paid in
fiat never inflates earning capacity.

## 13. Reward monetary ceiling (TD-197 / §49)
`rewardCeilingUzs = floor(rewardBasisUzs × 2000 / 10000)` (20%, integer-safe). Pure helper `rewardCeilingUzs`.

## 14. Reward IZL conversion (TD-197 / §50)
`rewardCeilingIzl = floor(rewardCeilingUzs / izlRateSnapshot)`; below one IZL's worth ⇒ 0; no rounding-up. Pure helper
`rewardCeilingIzl`. `izlRateSnapshot` = the ACTIVE `IzlRateVersion.rateUzsPerIzl` at periodStart.

## 15. Reward configuration fallback (TD-198)
Reward config is **enabled** only when a usable RewardPolicyVersion (ACTIVE, effectiveFrom ≤ periodStart, parses) AND a
usable IzlRateVersion (ACTIVE, effectiveFrom ≤ periodStart, rate > 0) both exist at periodStart. Otherwise the cycle is
**reward-disabled** — paid access is never blocked by gamification config.

## 16. Reward-disabled cycle (TD-198 / FP-DB-02/03)
`rewardPolicyVersionId = NULL`, `izlRateSnapshot = NULL`, `rewardCeilingUzs = 0`, `rewardCeilingIzl = 0`,
`earnedIzl = 0`; `rewardBasisUzs` may still record the paid amount for audit. No fake policy, no fake/zero rate. The
coherence CHECK `chk_cycle_reward_config_coherent` enforces enabled (policy+rate, rate>0) XOR disabled (both NULL + zero
ceilings).

## 17. 2.1A economic ceiling enforcement (TD-199)
The cycle economic ceiling is now load-bearing: the effective per-cycle cap = `min(policyDefinedCycleCap,
SubscriptionCycle.rewardCeilingIzl)`. A reward-disabled cycle (policy NULL) yields no grant / no ledger / no error (the
mission + XP branch is never rolled back). Authoritative consumption remains `SUM(GRANTED RewardGrant per cycle)` under
the per-user IZL lock; `earnedIzl` stays non-authoritative (no incremental `+=`; a future projection may activate it).

## 18. CONSUMED semantics (TD-201 / §34)
`IzlReservationStatus + CONSUMED` = an ACTIVE hold fulfilled by a REDEEM ledger debit — distinct from RELEASED (freed,
no spend). No runtime producer yet (the 2.1G finalizer is the first).

## 19. REDEEM uniqueness (TD-201 / FP-DB-04)
`uq_izl_ledger_redeem_per_redemption (redemption_id) WHERE redemption_id IS NOT NULL AND entry_type = 'REDEEM'` — at
most one REDEEM debit per redemption. Not a global `redemption_id` UNIQUE (future REVERSAL/ADJUSTMENT audit entries may
share the provenance).

## 20. Negative-balance held-funds semantics (TD-203)
Discounted finalization does **not** re-check `amount ≤ current availableIzl` — the ACTIVE reservation is the prior
authorization. If later ledger corrections made `balance < reserved`, the held amount is still consumed and the REDEEM
debit may drive the signed ledger balance negative (exposed honestly). An already-paid purchase is never stranded / auto-
released / resized / rejected because current balance moved.

## 21. Payment timestamp (TD-204)
The unique SUCCEEDED `PaymentTransaction.confirmedAt` is the payment-time authority; a future redemption `resolvedAt =
confirmedAt`. No `PaymentOrder.paidAt`.

## 22. Provider provenance (TD-204)
`PaymentTransaction.provider` is the provider authority; `PaymentOrder.provider` stays NULL/non-authoritative (not
snapshotted at finalization). Audit chain = `PaymentOrder → SUCCEEDED transactions` (PV-DB-01) +
`SubscriptionCycle.paymentOrderId` (`@unique`); no redundant pointer.

## 23. Lock order (TD-202)
Global multi-lock order for any operation needing more than one: **`sub(userId) → pay(paymentOrderId) → izl(userId)`**
(izl only when discounted). The finalizer runs in a separate transaction *after* the verification commit — never nested
inside it, never with a provider call inside the DB transaction. Existing single-lock flows (reward/reservation/
redemption = izl; callback = pay) are unchanged; none takes these in reverse.

## 24. Future atomic finalizer + the incidental clock fix
Phase 2.1G will run one replay-safe DB transaction under the lock order producing: PaymentOrder PAID + Subscription
activate/reactivate + SubscriptionCycle + entitlement snapshot + (if discounted) REDEEM + reservation CONSUMED +
redemption APPLIED — idempotent (order PAID + `SubscriptionCycle.paymentOrderId` unique + one-REDEEM) and recoverable
(SUCCEEDED + PENDING backlog). **Incidental fix in this phase:** `lesson-execution.service` stamped attempt
`submittedAt` with a real `new Date()`; it now uses the injected `Clock` (the codebase-wide discipline). This was a
latent clock leak exposed when the run date (2026-08-21) diverged from the frozen test clock (2026-08-20), breaking the
learner-local mission day; the `learner-signals` e2e was updated to advance its frozen clock per submit (as real time
does).

## 25. Recovery model + tests
- **Tests:** `calendar-month.spec.ts` (9), `reward-ceiling.spec.ts` (8) — pure unit; `izl-reward.e2e` (+3:
  reward-disabled no-op, ceiling governs, policy-cap governs); `finalization-schema-hardening.e2e` (3: billing CHECK,
  CONSUMED reserved-SUM exclusion, one-REDEEM-per-redemption + REVERSAL still allowed).
- **Gate:** 378 unit + 351 e2e PASS; `tsc` clean; migration 20; drift clean (empty diff); constraints verified in dev +
  test. Regression: 2.0B–2.1F + XP + daily-mission + learner-signals green.
- **Recovery (future):** the finalizer must be idempotent (PAID / cycle-per-order / one-REDEEM), and a `SUCCEEDED PT +
  PENDING PaymentOrder` pair is a recoverable backlog for an internal reconcile (no learner "mark paid" route).

See [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) for the deferred Phase 2.1G finalizer + everything in §59.
