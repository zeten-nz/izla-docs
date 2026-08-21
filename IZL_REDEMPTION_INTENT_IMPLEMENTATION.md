# Subscription Discount Redemption Intent — Implementation (Phase 2.1C-2)

> Status: **COMPLETE** (2026-08-20). Closes Phase 2.1C. A learner reserves IZL against a concrete own `PaymentOrder`
> as a **RESERVED** `IZLRedemption` bound 1:1 to an **ACTIVE** `IZLReservation`, under the 20% gross ceiling. This is
> a **reserve-only** phase: no ledger debit, no `APPLIED`, no reservation `CONSUMED`, no PaymentOrder mutation, no
> PaymentTransaction/Subscription. Owner: **TD-172/173/174/175/176/177**. Migration
> `20260820220000_subscription_discount_redemption_intent` (migration 16).

Code: `backend/src/finance/redemption/` — `subscription-discount-redemption.policy.ts` (pure),
`redemption.repository.ts`, `subscription-discount-redemption.service.ts`, `redemption.controller.ts`,
`dto/create-redemption.dto.ts`.

## 1. Phase 2.1C architecture-gap history
Phase 2.1C stopped: no persisted subscription-discount ceiling snapshot exists (the cycle's `reward_ceiling_*` is an
*earning* ceiling). Owner decision (TD-171): the ceiling belongs to a **concrete PaymentOrder** gross price.

## 2. Why PaymentOrder foundation came first
Phase 2.1C-PO established the provider-agnostic purchase order (frozen gross). This phase binds redemption to it.

## 3. Concrete PaymentOrder authority (TD-172)
The reused `IZLRedemption` aggregate binds to a `PaymentOrder` (`paymentOrderId`, now **Restrict**). No competing
`RedemptionRequest` model. Eligibility: own order, purpose `SUBSCRIPTION_PURCHASE`, status `CREATED`, not expired,
and undiscounted (`izlDiscountAmount = 0`, `payableAmount = gross`).

## 4. Policy version (TD-172)
`policyVersionCode = subscription-discount-redemption-v1` on every v1 redemption (nonempty CHECK). Not the earning
`RewardPolicyVersion`.

## 5. SUBSCRIPTION_DISCOUNT-only v1 (§3)
Runtime type is `SUBSCRIPTION_DISCOUNT` only. Other enum values are untouched with no producer.

## 6. Gross price ceiling authority (TD-173 / §23)
`maxDiscountUzs = floor(grossAmount × 2000 / 10000)` — the concrete order gross is the base. `SubscriptionCycle`
reward ceilings are never referenced (§7).

## 7. 20% integer calculation (§20)
Integer-safe: `Math.floor((gross × 2000) / 10000)` (gross ≤ 2^31 → safe in Number).

## 8. Earning/spending ceiling separation (§7 / RD-07)
`SubscriptionCycle.rewardCeilingUzs`/`rewardCeilingIzl` remain earning-only; grep-verified no spending use.

## 9. Rate authority (TD-173 / §19)
The effective rate is the **ACTIVE** `IzlRateVersion` with `effectiveFrom ≤ now` (DB enforces one ACTIVE). None →
`REDEMPTION_RATE_NOT_AVAILABLE`. No client rate, no `RewardPolicyVersion`, no old redemption rate.

## 10. Exact value calculation (TD-173 / §21 / §22)
`valueUzs = amountIzl × rateUzsPerIzl`, computed with **BigInt** (both are integers → exact, **no rounding**). The
BigInt intermediate also guards Int overflow — a value beyond Int range necessarily exceeds the ceiling (≤ gross ≤
2^31) and is rejected before insert.

## 11. Immutable quote snapshots (TD-173 / §9 / §20)
`izlRateSnapshot` + `valueUzs` are frozen on the redemption. Later rate changes never mutate an existing redemption;
a replay never re-resolves or reprices (e2e §41).

## 12. PaymentOrder expiry check (§17 / §69)
If `expiresAt` is non-null and `now ≥ expiresAt`, creation is rejected — with no order mutation and no rows.
`expiresAt = NULL` imposes no expiry rejection.

## 13. No redemption TTL (§18)
No redemption/reservation TTL, no scheduler, no auto-release. The old-rate lock exposure is documented OPEN; manual
trusted release is the only release flow.

## 14. Canonical available IZL (§26 / RD-10)
Availability is `SUM(IZLLedgerEntry.amount) − SUM(ACTIVE IZLReservation.amount)` computed under the finance lock —
never the wallet cache (e2e §40 proves a stale wallet cannot authorize).

## 15. Wallet-not-authority (§40)
The `IZLWallet` projection is a downstream cache; a seeded high wallet does not permit an over-reservation.

## 16. Finance lock (§25 / §28)
Everything runs in **one** `$transaction` under `pg_advisory_xact_lock(hashtext('izl'), hashtext(userId))` — the same
per-user IZL namespace as 2.1A earning and 2.1B reservation, so redemption, earning, and holds serialize without
nested locks/transactions (§29).

## 17. Idempotency (TD-175 / §18)
`clientRequestId` required; partial UNIQUE `uq_izl_redemption_client_request (user_id, client_request_id) WHERE NOT
NULL` (RD-DB-01). Replay of the same `(user, key, order, amount)` returns the original; a reused key with a different
order/amount is `REDEMPTION_REQUEST_CONFLICT` (e2e §36/§37). Concurrent identical → P2002 winner-reload.

## 18. One-open-per-order (TD-175 / §9)
Partial UNIQUE `uq_izl_redemption_open_per_order (payment_order_id) WHERE type=SUBSCRIPTION_DISCOUNT AND status IN
(REQUESTED,RESERVED)` (RD-DB-04). A second open redemption for the same order → `REDEMPTION_OPEN_INTENT_CONFLICT`;
after RELEASED, a new one (new key) is allowed (e2e §9/§10).

## 19. Typed reservation provenance (TD-174 / §7)
`IZLReservation.redemptionId` is a typed **unique** FK (Restrict) — the load-bearing 1:1 hold↔redemption relation
(RD-DB-02/03). `purposeCode = SUBSCRIPTION_DISCOUNT_REDEMPTION` and the derived `idempotencyKey` are supplemental,
not the relation authority.

## 20. Atomic create (TD-174 / §22 / §32 / §75)
In one transaction: eligibility → rate → value/ceiling → availability → create `IZLRedemption` (RESERVED) → create
`IZLReservation` (ACTIVE, `redemptionId` set). Forbidden partial states cannot commit (a reservation-insert failure
rolls back the redemption).

## 21. PaymentOrder remains undiscounted (§35 / §11 / RD-16)
During RESERVED creation the order's `grossAmount`/`izlDiscountAmount(0)`/`payableAmount(gross)`/`izlRedemptionId
(NULL)` are unchanged (e2e §35). `izlRedemptionId` stays NULL — reserved for future APPLIED provenance (2.1D).

## 22. Release atomicity (TD-176 / §45 / §76)
`POST /api/izl/redemptions/:id/release`: one transaction under the lock transitions redemption `RESERVED→RELEASED`
**and** reservation `ACTIVE→RELEASED`, atomically. Idempotent (RELEASED → terminal). (e2e §44–47).
**Amended by Phase 2.1D (TD-181):** once a redemption is *committed* to its order (Phase 2.1D), its release also
restores the order pricing in the same transaction and requires the order still `CREATED`; an *uncommitted* release
remains independent of order status as described here. See
[IZL_DISCOUNT_COMMIT_IMPLEMENTATION.md](IZL_DISCOUNT_COMMIT_IMPLEMENTATION.md).

## 23. No ledger on reserve/release (§34 / §54)
Neither reserve nor release writes `IZLLedgerEntry`. Reserved rises/falls; `balanceIzl` is unchanged (e2e ledger
counts).

## 24. PaymentOrder.izlRedemptionId deferred (§11)
`PaymentOrder.izlRedemptionId` is left NULL; it is reserved for the future APPLIED discount provenance so there are
never two active redemption authorities. The intent binding is `IZLRedemption.paymentOrderId`.

## 25. No APPLIED / CONSUMED (§28 / §52 / §53)
No runtime path sets `IZLRedemption.status = APPLIED` or a reservation to CONSUMED; the `IzlReservationStatus` enum
has only ACTIVE/RELEASED (grep-verified). Those belong to Phase 2.1D (atomic spend with a real benefit).

## 26. Security (§63–§65)
Own-user only; a foreign order → 404 on create; a foreign redemption → 404 on GET/release; 401 unauth. The DTO
whitelist accepts only `paymentOrderId`/`amountIzl`/`clientRequestId` — injected `status`/`valueUzs`/`rate`/… → 400
(e2e §65). GET/create/release never leak the rate table, reservation idempotency key, purposeCode, or policy config.

## 27. Tests
- **Unit** (`subscription-discount-redemption.policy.spec.ts`, 6): ceiling floor (100000→20000, 99999→19999, tiny→0);
  within/over ceiling; floor boundary; availability over/exact/negative; non-positive amount.
- **e2e** (`test/izl-redemption.e2e-spec.ts`, 13, §9–§76): create RESERVED+ACTIVE (no ledger, order unchanged,
  reserved rises); idempotent replay; conflict; one-open-per-order + RELEASED→new; ceiling boundary; availability
  over/exact; stale-wallet safety; rate snapshot + no reprice on replay; release (atomic, no ledger, idempotent);
  ineligible orders (expired/status/purpose/discounted); no ACTIVE rate; no APPLIED/CONSUMED/ledger/PaymentTransaction/
  Subscription/XP; cross-user IDOR 404 + 401 + no injection.
- Gate: **361 unit + 310 e2e PASS**; `tsc` clean; migration 16; drift clean.

## 28. Phase 2.1D future application boundary
Deferred: `PaymentOrder CREATED + RESERVED IZLRedemption + ACTIVE IZLReservation` → one atomic application:
`izlDiscountAmount = valueUzs`, `payableAmount = gross − discount`, `izlRedemptionId = redemption.id`, `IZLLedgerEntry
REDEEM = −amountIzl`, reservation → CONSUMED, redemption → APPLIED — **without** PaymentTransaction/Click/Payme/PAID/
Subscription activation. See [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).

> Note on §29 reservation reuse: the redemption transaction inlines the ACTIVE-hold create + canonical availability
> read (rather than calling the 2.1B `IzlReservationService`, which opens its own transaction/lock) so the whole
> operation stays within one transaction and one finance lock. The canonical availability rule and lock namespace are
> identical to 2.1B; no second transaction/lock is opened.
