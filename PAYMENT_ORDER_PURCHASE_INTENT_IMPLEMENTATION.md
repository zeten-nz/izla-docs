# PaymentOrder Subscription Purchase Intent — Implementation (Phase 2.1C-PO)

> Status: **COMPLETE** (2026-08-20). A learner can create a concrete, **provider-agnostic** subscription purchase
> intent: the backend selects the deterministic current `PlanPrice` and freezes an immutable pricing snapshot on a
> `PaymentOrder` (status `CREATED`). This order is the future authority for Phase 2.1C-2 subscription-discount
> redemption. **No provider execution. No PaymentTransaction. No Subscription/Cycle. No IZL. No XP.**
> Owner: **TD-167/168/169/170/171**. Migration `20260820210000_payment_order_purchase_intent` (migration 15).

Code: `backend/src/payments/` — `payments.controller.ts`, `payments.service.ts`, `payments.repository.ts`,
`dto/create-subscription-order.dto.ts`, `payments.module.ts`.

## 1. Why purchase context precedes redemption
The Phase 2.1C gap was that no subscription-discount **ceiling** snapshot exists (the cycle's `reward_ceiling_*` is
an *earning* ceiling). The owner decision (TD-171) is that the spending ceiling belongs to a **concrete
PaymentOrder** (`maxDiscountUzs = floor(grossAmount × 2000/10000)`). Therefore a concrete purchase order must exist
before a redemption can bind to it — this phase establishes that order.

## 2. PaymentOrder authority (TD-167)
`PaymentOrder` is the internal concrete purchase authority: it freezes the plan, price, gross/discount/payable, and
purpose. Provider execution is a separate concern (`PaymentTransaction`). v1 supports purpose **SUBSCRIPTION_PURCHASE**
only. The single writer for this phase is `payments.repository`.

## 3. PlanPrice resolution (TD-169 / §6)
At creation time T, `currentPlanPrice` selects the one UZS `PlanPrice` with `effectiveFrom ≤ T`, ordered
`effectiveFrom DESC, id DESC` — the latest eligible immutable price (TD-85). `effectiveFrom == now` is eligible. No
eligible price → `PAYMENT_PLAN_PRICE_NOT_AVAILABLE` (409), no order row. The clock is the injectable `Clock`.

## 4. Immutable gross snapshot (TD-169 / §7)
The order stores `planPriceId` (FK) + `grossAmount` (the price amount) — double protection. Later `PlanPrice`
changes never reprice an existing order (e2e §29/§30).

## 5. Gross / discount / payable semantics (§6)
Initial order: `izlDiscountAmount = 0`, `payableAmount = grossAmount`. The existing CHECK `chk_order_amounts`
enforces `payable = gross − discount` and non-negativity. No discount is applied in this phase.

## 6. Provider separation (TD-168 / §8)
`PaymentOrder.provider` was made **nullable** (migration 15). Purchase-intent creation sets `provider = NULL` — the
order is provider-agnostic. `PaymentTransaction.provider` remains the provider execution authority (CLICK/PAYME). The
`provider` field is neither removed nor redefined; a future provider phase decides its long-term semantics.

## 7. PaymentTransaction boundary (§16)
No `PaymentTransaction` (and no `PaymentCallbackEvent`) is created. Provider execution is entirely out of scope.

## 8. Idempotency (TD-170 / §10)
The create API requires `clientRequestId`. DB authority: partial UNIQUE `uq_payment_order_client_request
(user_id, client_request_id) WHERE client_request_id IS NOT NULL` (PO-DB-01). Different users may reuse the same key
(e2e §37).

## 9. Replay semantics (§11)
A replay of the same `(userId, clientRequestId)` returns the **original** order — never repriced. A reused key with a
different `planId`/purpose is a conflict (`PAYMENT_ORDER_REQUEST_CONFLICT`, 409) — no second order (e2e §12/§38).
Concurrent identical requests converge to one order via P2002 winner-reload (e2e §13/§39).

## 10. Price-change behavior (§29–§31)
Order A created at price 100000 stays 100000 after the price rises to 120000; a replay of A's key still returns
100000; a new request (new key) gets 120000. Historical purchase economics are frozen.

## 11. CREATED-only lifecycle (§14)
New orders are `status = CREATED`. This phase performs no transition to PENDING/PAID/FAILED/CANCELLED/EXPIRED — there
is no provider execution.

> **Phase 2.1E update:** the `CREATED → PENDING` transition is now owned by payment execution initiation (a
> `PaymentTransaction` PENDING attempt; see [PAYMENT_EXECUTION_IMPLEMENTATION.md](PAYMENT_EXECUTION_IMPLEMENTATION.md)).
> `PaymentOrder.provider` still stays NULL (execution authority is `PaymentTransaction.provider`, TD-185), and PAID
> remains out of scope until Phase 2.1F verified-payment finalization.

## 12. Subscription-after-payment boundary (§15 / TD-167)
Creating an order creates **no** `Subscription`, `SubscriptionCycle`, `Entitlement`, or `UsageCounter`. The accepted
lifecycle (DATA_MODEL_FINANCE) is PaymentOrder → verified payment → Subscription → SubscriptionCycle. No premature
subscription (e2e §41).

## 13. No IZL side effects (§17)
Creating an order writes no `IZLRedemption`, `IZLReservation`, `IZLWallet`, `IZLLedgerEntry`, or `RewardGrant`. The
initial `izlDiscountAmount` is 0; redemption reservation is Phase 2.1C-2 (e2e §42).

## 14. Future discount ceiling (TD-171 / §19)
Recorded for 2.1C-2: `maxDiscountUzs = floor(grossAmount × 2000/10000)` (20% of the concrete purchase gross,
integer-safe). Not used to mutate the order here; it becomes redemption validation authority in 2.1C-2. No discount
ceiling field is added to `SubscriptionCycle`.

## 15. Earning-ceiling separation (§21)
`SubscriptionCycle.rewardCeilingUzs`/`rewardCeilingIzl` remain **earning-only** (TD-50) — no spending/discount code
uses them.

## 16. GET ownership / security (§23–§26)
`GET /api/payments/orders/:id` is own-user only (404 for another user's order), read-only (no reselection/transition/
expiration). The create API accepts only `planId` + `clientRequestId`; the ValidationPipe (whitelist +
forbidNonWhitelisted) rejects any injected economic field (grossAmount/provider/status/…, e2e §26). Unauthenticated →
401.

## 17. Tests
- **e2e** (`test/payment-order.e2e-spec.ts`, 9): CREATED order (frozen gross, discount 0, payable=gross, provider
  NULL) + no side effects; idempotent replay (not repriced); idempotency conflict; price-change new gross; price
  eligibility (no price / future-only → 409, `effectiveFrom==now` → 201); archived plan → 404; cross-user key reuse →
  two orders; concurrent identical → one; GET own read-only + IDOR 404 + 401 + no client economic authority.
- Gate: **355 unit + 297 e2e PASS**; `tsc` clean; migration 15; drift clean.

## 18. Deferred (renewal / provider / payment)
Out of scope: SUBSCRIPTION_RENEWAL/upgrade/downgrade/proration, provider selection + PaymentTransaction + Click/Payme
+ callbacks, PAID transition, order expiry duration, Subscription/Cycle activation orchestration, the
one-nonterminal-subscription enforcement at PAID. See [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).

## 19. Future 2.1C-2 binding
Next: `PaymentOrder (CREATED) + effective IzlRateVersion + canonical available IZL → IZLRedemption RESERVED + typed
ACTIVE IZLReservation`, enforcing `valueUzs ≤ maxDiscountUzs(grossAmount)` and `≤ grossAmount`, one-open-per-order,
idempotent, with no ledger debit and no APPLIED. That phase also hardens `IZLRedemption` (subscriptionCycle vs
paymentOrder binding, clientRequestId, typed reservation FK).
