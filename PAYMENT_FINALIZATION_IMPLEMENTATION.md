# Verified Payment Economic Finalization — Implementation (Phase 2.1G)

> Status: **COMPLETE** (2026-08-21). A persisted trusted `PaymentTransaction.SUCCEEDED` + PENDING `PaymentOrder` (+
> optional committed discount) is converted into a paid subscription in **one replay-safe internal DB transaction** —
> `PaymentOrder` PAID + `Subscription` ACTIVE + `SubscriptionCycle` + `SubscriptionCycleEntitlement`, and (discounted
> only) IZL `REDEEM` + reservation `CONSUMED` + redemption `APPLIED`. **No provider call.** Owner: **TD-205..210**. No
> new migration (post-migration-20 schema is sufficient) — migration count stays **20**.

Code: `backend/src/payments/` — `payment-finalization.repository.ts` (`finalize`, `reconstructFinalized`,
`resolveRewardConfig`), `payment-finalization.service.ts` (`finalizeVerifiedPayment`, `tryFinalizeAfterVerification`),
`payment-callback.service.ts` (bridge). Reuses `common/calendar-month.ts`, `finance/reward/reward-ceiling.ts`.

## 1. Trusted SUCCEEDED authority
The finalizer's only input is an internal `paymentTransactionId`; everything else is loaded from the DB. It requires a
persisted `PaymentTransaction.status = SUCCEEDED` with `confirmedAt` and a matching PENDING order.

## 2. Verified vs finalized
2.1F produces trusted evidence (SUCCEEDED); 2.1G produces the paid subscription. They are separate transactions so a
verified payment is always recoverable even if finalization defers.

## 3. No provider call (§69)
The finalizer never touches `PAYMENT_PROVIDER_PORT` / `initiate` / `verifyCallback` / Click / Payme — it consumes
persisted evidence only (grep-verified).

## 4. Finalization policy (TD-205)
`verified-payment-finalization-v1`. Internal/server-owned. No learner endpoint marks an order paid or activates a
subscription. Client trusted values are never accepted.

## 5. Lock ordering (TD-202 / §9)
One transaction acquires, in the fixed global order, `pg_advisory_xact_lock('sub', userId)` → `('pay', orderId)` →
`('izl', userId)` (izl only when discounted). Never reversed; never nested inside the 2.1F callback transaction.

## 6. Replay authority (TD-209)
`PaymentOrder.status = PAID` + `SubscriptionCycle.paymentOrderId` UNIQUE + FP-DB-04 (one REDEEM/redemption). A PAID
order returns its reconstructed, validated projection without new writes.

## 7. Order expiry boundary (TD-196)
`expiresAt` never blocks finalization of a trusted SUCCEEDED payment (§6) — paid money is not stranded.

## 8. PlanPrice duration authority (§15)
Load the exact frozen `order.planPriceId`; require `planId`/`currency` match and `billingPeriodMonths > 0`. Never select
the current price; never re-price.

## 9. Payment-time authority (§16 / TD-204)
`periodStart = PaymentTransaction.confirmedAt` (not finalizer now / callback receivedAt / order createdAt). No
`PaymentOrder.paidAt`; `PaymentOrder.provider` is not written.

## 10. Subscription episode rules (TD-206)
`Subscription` is one continuous membership episode (F-14 `ux_nonterminal_subscription`). Activation is decided from the
single nonterminal (ACTIVE/EXPIRED) subscription, resolved under the sub-lock.

## 11. EXPIRED reactivation (§19 / §53)
An EXPIRED episode is reused in place: `status → ACTIVE`, `planId → order.planId`, `currentCycleId → new cycle`;
`startedAt` is preserved; historical cycles are untouched; the new cycle's `sequenceNo = MAX(existing) + 1`.

## 12. ACTIVE conflict (§21 / §52 / §89)
An existing ACTIVE subscription → `SubscriptionPurchaseActiveConflictError`. The transaction rolls back: no PAID, no
cycle, no IZL consumption; the redemption stays RESERVED, the reservation ACTIVE, the transaction SUCCEEDED — recoverable
for future operator/renewal handling. Never silently treated as a renewal.

## 13. CANCELLED history (§22 / §90)
CANCELLED subscriptions are terminal and excluded by F-14; only-CANCELLED history → a brand-new Subscription episode.

## 14. sequenceNo (§23)
New subscription → `1`; EXPIRED reactivation → `MAX(existing cycle.sequenceNo) + 1` under the sub-lock (never `count+1`).
Respects `@@unique([subscriptionId, sequenceNo])`.

## 15. Cycle commercial snapshots (§26 / TD-207)
Frozen from the order only: `grossPriceUzs ← grossAmount`, `discountUzs ← izlDiscountAmount`, `paidAmountUzs ←
payableAmount`, `rewardBasisUzs ← payableAmount`, `planId`/`planPriceId ← order`; `earnedIzl = 0`. No current-price
lookup, no re-pricing.

## 16. Entitlement snapshot (§31 / §32 / §59)
Each `PlanEntitlement` of `order.planId` is copied into an immutable `SubscriptionCycleEntitlement`
(`featureCode`/`mode`/`limitValue`), deterministically, respecting `@@unique([cycleId, featureCode])`. No `UsageCounter`
is created (deferred, 2.1G-D recon). Later PlanEntitlement changes never mutate an existing cycle snapshot.

## 17. Reward-enabled cycle (§29 / §55)
When a usable ACTIVE `RewardPolicyVersion` (effectiveFrom ≤ periodStart, config parses) and a usable ACTIVE
`IzlRateVersion` (rate > 0) both exist at periodStart: snapshot `rewardPolicyVersionId`, `izlRateSnapshot`,
`rewardCeilingUzs = floor(payable × 20%)`, `rewardCeilingIzl = floor(ceilingUzs / rate)`. Activation grants no IZL (§76).

## 18. Reward-disabled cycle (§30 / §56)
Otherwise: `rewardPolicyVersionId = NULL`, `izlRateSnapshot = NULL`, `rewardCeilingUzs = 0`, `rewardCeilingIzl = 0`
(coherence CHECK from 2.1G-D). `rewardBasisUzs` still records the paid amount.

## 19. Reward config failure boundary (§28 / §57 / §58)
Reward configuration is non-critical: missing / future-effective / malformed policy or invalid rate → reward-disabled
cycle, paid access still activates. The policy parser is pure, so a parse failure is a config failure (→ disabled); a
real DB/system failure is NOT masked as "disabled" (it fails the transaction normally).

## 20. Discounted provenance (§35 / TD-208)
`order.izlRedemptionId = R` requires: `R` RESERVED, type SUBSCRIPTION_DISCOUNT, `paymentOrderId = order.id`, `userId`
match, `valueUzs = izlDiscountAmount`; the typed reservation ACTIVE, `redemptionId = R`, `amount = R.amountIzl`. No loose
matching. `izlDiscountAmount > 0` with a NULL pointer → integrity error (§33); undiscounted requires `discount = 0` and
`payable = gross` (§34).

## 21. Held IZL authorization (§36 / TD-203)
The ACTIVE reservation is the prior authorization; finalization never re-checks `amount ≤ availableIzl`. Even if later
corrections left `balance < reserved`, the hold is consumed and the already-paid purchase is never stranded.

## 22. REDEEM ledger write (§37)
One `IZLLedgerEntry`: `entryType = REDEEM`, `amount = -redemption.amountIzl`, `redemptionId = R`, `subscriptionCycleId =
new cycle`, `entryNo = MAX(entryNo)+1`, `balanceAfter = SUM(amount) − amountIzl` (reusing the 2.1A ledger allocation
pattern). One REDEEM per redemption is DB-enforced (FP-DB-04).

## 23. CONSUMED (§39)
The reservation transitions `ACTIVE → CONSUMED` (never RELEASED) — the hold fulfilled by the actual debit.

## 24. APPLIED (§40)
The redemption transitions `RESERVED → APPLIED`, `resolvedAt = PaymentTransaction.confirmedAt` (not server now).

## 25. Negative balance (§42 / §87)
The REDEEM debit may drive the signed ledger balance negative (e.g. ledger 2, reserved 4 → after −2). Exposed honestly;
never clamped or rejected.

## 26. Available invariant (§41)
Reserve → consume keeps `available` unchanged (before: 10 − 4 = 6; after: 6 − 0 = 6), because the ACTIVE-only reserved
SUM drops by exactly the ledger debit.

## 27. PAID transition (§43 / §44)
`PaymentOrder` moves `PENDING → PAID` (committed with the complete business state, no pricing/provider write). All
writes are in one transaction, so physical ordering does not affect atomicity — PAID never commits without the full
finalization.

## 28. Atomic rollback (§62 / §98)
Any failure (integrity, constraint, active-conflict) rolls back the whole transaction: no partial activation. The
transaction stays SUCCEEDED, the order PENDING, the discount RESERVED/ACTIVE — recoverable. Exercised by an FP-DB-04
constraint failure forced *after* the subscription + cycle writes.

## 29. Replay / concurrency (§48 / §49 / §50 / §51)
Duplicate finalization creates no second cycle / subscription / REDEEM / APPLIED / CONSUMED. Concurrent same-order
finalizations serialize (sub → pay → izl) to exactly one effect. Two paid orders for one user: one activates, the second
hits the ACTIVE conflict and stays PENDING.

## 30. Callback bridge (§63 / TD-210)
After the 2.1F callback transaction commits a trusted SUCCEEDED, `PaymentCallbackService` calls
`tryFinalizeAfterVerification` (best-effort, non-throwing) in a **separate** transaction — never nested. Only when the
outcome carries a SUCCEEDED transaction.

## 31. Bridge failure recovery (§64 / §65 / §66)
A finalization failure never rolls back or erases the verified-payment evidence (PT SUCCEEDED, PaymentCallbackEvent) —
the order stays PENDING, recoverable. A matching callback replay (DUPLICATE) may retry a stuck finalization idempotently.
Provider-facing callback success is not downgraded to failure because internal finalization deferred.

## 32. Wallet downstream boundary (§60 / §61)
For a discounted finalization, `IzlWalletService.tryRecompute(userId)` runs **after** the transaction commits, non-
throwing — a projection failure never rolls back PAID / Subscription / Cycle / REDEEM / CONSUMED / APPLIED. Undiscounted
finalization does not recompute. `GET /api/izl/me` stays canonical from ledger + ACTIVE reservations.

## 33. Audit chain
`PaymentTransaction(SUCCEEDED) → PaymentOrder(PAID) → SubscriptionCycle(paymentOrderId @unique) → Subscription`;
discounted adds `Cycle ↔ IZLRedemption(APPLIED) ↔ IZLReservation(CONSUMED) ↔ IZLLedgerEntry(REDEEM, redemptionId)`.
Cross-user integrity is validated in the finalizer (userId equality on redemption/reservation/cycle).

## 34. Tests
- **e2e** (`test/payment-finalization.e2e-spec.ts`, 18, §85–§101 — invokes `PaymentFinalizationService` /
  `PaymentCallbackService` directly, seeds via Prisma; no learner finalization route): undiscounted (PAID + ACTIVE +
  calendar-month period + net snapshots + reward-enabled ceilings); discounted (REDEEM −4 / CONSUMED / APPLIED /
  resolvedAt=confirmedAt / available invariant); negative balance; EXPIRED reactivation (same episode, plan updated,
  startedAt preserved, seq+1, old cycle untouched); ACTIVE conflict (recoverable, discounted hold intact); CANCELLED →
  new episode; reward-disabled / malformed / future-effective config; entitlement snapshot (+ no UsageCounter); replay
  (undiscounted + discounted, one effect); concurrent same-order; two paid orders same user; post-callback bridge auto-
  finalize (no provider re-charge); bridge-does-not-break-verification on conflict; discount-pointer-null integrity;
  mid-transaction rollback (atomic).
- **Verification isolation:** the 2.1F `payment-verification.e2e` suite stubs `PaymentFinalizationService` so it keeps
  testing the verification layer alone (order stays PENDING); the real bridge is covered by this suite.
- **Gate:** 378 unit + 369 e2e PASS; `tsc` clean; **no new migration** (count 20); drift clean (empty diff). Regression:
  2.0B–2.1F + XP + daily-mission + learner-signals + verification green.

> **Phase 2.1H update (2026-08-21):** the SUCCEEDED+PENDING backlog is now operationally drainable — an internal/admin
> reconciler retries stuck items through **this same finalizer** (no second implementation, no provider call). See
> [PAYMENT_FINALIZATION_RECOVERY_IMPLEMENTATION.md](PAYMENT_FINALIZATION_RECOVERY_IMPLEMENTATION.md). The finalizer's
> replay/idempotency and lock/unique guarantees are what make the bridge and the reconciler converge safely.

## 35. Remaining payment-lifecycle work
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): real Click/Payme adapters + webhook routes; FAILED / CANCELLED /
REFUNDED / retry / timeout / late-success; PaymentOrder FAILED/CANCELLED/EXPIRED runtime; SUBSCRIPTION_RENEWAL / upgrade
/ downgrade / proration / auto-renew / recurring; refund / reversal / chargeback; zero-payable activation; reservation /
redemption TTL; `earnedIzl` projection; finalization reconciliation scheduler / admin tooling; notifications; frontend.
