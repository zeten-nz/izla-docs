# Payment Execution Foundation — Implementation (Phase 2.1E)

> Status: **COMPLETE** (2026-08-20). A learner initiates a **provider payment execution attempt** for an own
> **CREATED** `PaymentOrder`: a **PENDING** `PaymentTransaction` is persisted + the order moves `CREATED → PENDING`,
> THEN a provider port is called **outside** the DB transaction and the returned `providerTransactionId` is attached.
> This creates **payment execution INTENT, not payment SUCCESS** — no verified callback, no `SUCCEEDED`, no
> `PaymentOrder` PAID, no IZL spend, no `Subscription` activation, no real Click/Payme API. Owner:
> **TD-183/184/185/186/187/188**. Migration `20260820240000_payment_execution_intent` (migration 18).
>
> **Phase 2.1L-D forward-note (2026-08-21):** initiation still returns a `providerTransactionId` from the port; a future
> real adapter will instead learn the provider id at the provider-native NON-terminal step (CLICK Prepare / Payme
> CreateTransaction) and attach it via the non-terminal binding primitive (`provider_transaction_id` only, PENDING PT,
> no status transition — TD-234), persisting protocol state in `ClickShopTransaction` / `PaymeMerchantTransaction`
> (TD-233). See [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md).

Code: `backend/src/payments/` — `payments.repository.ts` (`resolveOrCreateAttempt`, `attachProviderInit`,
`attemptWithOrder`, `assertCommittedDiscountIntegrity`), `payments.service.ts` (`initiate`), `payments.controller.ts`
(`POST orders/:id/initiate`), `dto/initiate-payment.dto.ts`, `provider/payment-provider.port.ts`,
`provider/unavailable-payment-provider.adapter.ts`. Test adapter: `test/test-payment-provider.adapter.ts`.

## 1. What this phase is (TD-183 / §2)
An intent to *start* a provider charge — not a charge that succeeded. The attempt exists so a provider (Click/Payme)
can later drive it to success via a verified callback (Phase 2.1F). Nothing here realizes subscription benefit.

## 2. Attempt lifecycle (TD-183 / §9 / §33)
`POST /api/payments/orders/:id/initiate` (own-user). In one transaction under the per-user IZL lock: load the own
order → create a `PaymentTransaction` `PENDING` (`providerTransactionId = NULL`, `amount = payableAmount`) → move the
order `CREATED → PENDING`. The order transition and the attempt are atomic.

## 3. Eligibility (§6–§11 / §75–§77)
The order must be `purpose = SUBSCRIPTION_PURCHASE`, `status = CREATED`, not expired (`now < expiresAt` when set), and
`payableAmount > 0`. Any failure → `PaymentOrderNotEligibleError` (**409 PAYMENT_ORDER_NOT_ELIGIBLE**); no attempt is
created and the order stays `CREATED`.

## 4. Charge amount authority (TD-186 / §66)
`PaymentTransaction.amount = PaymentOrder.payableAmount` — the committed, post-discount value, **never** `grossAmount`.
A discounted order (`izlDiscountAmount > 0`) charges `gross − discount`. Server-derived only.

## 5. Currency (TD-186 / §36)
`PaymentTransaction` has **no** currency column (reported, not invented — currency lives on `PaymentOrder`). The
attempt inherits `PaymentOrder.currency`; it is passed to the provider and returned in the initiation view.

## 6. Provider authority separation (TD-185 / §32)
`PaymentTransaction.provider` (CLICK/PAYME) is the **sole execution authority**. `PaymentOrder.provider` stays
**NULL** and is never written by initiate (it is the provider-agnostic purchase authority, TD-168). The e2e asserts the
order's `provider` is still NULL after initiation.

## 7. Provider port (TD-184 / §22 / §62)
Payment provider init is behind the injectable token **`PAYMENT_PROVIDER_PORT`** (the SMS_PORT pattern). Production
binds `UnavailablePaymentProviderAdapter`, whose `initiate()` throws `PaymentProviderUnavailableError` (no real
Click/Payme). Tests override the token with a deterministic `TestPaymentProviderAdapter`.

## 8. External call is outside the transaction (§24 / §26 / §35)
`resolveOrCreateAttempt` (tx #1) persists the PENDING attempt and returns a stable `txId`. The service then calls
`provider.initiate(...)` **outside any DB transaction**, and `attachProviderInit` (tx #2) writes the returned
`providerTransactionId` (+ sanitized metadata). A crash between the two leaves a resumable PENDING attempt.

## 9. Provider-init attach (TD-184 / §27 / §29)
`attachProviderInit` is a no-op unless the attempt is still `PENDING` with `providerTransactionId = NULL`. On a
duplicate external identity (`@@unique([provider, providerTransactionId])`, PT-DB-03) it swallows the P2002 and leaves
the attempt PENDING (defensive, §74) rather than corrupting state.

## 10. Ambiguous / failed provider init (TD-184 / §30 / §31 / §72)
If `provider.initiate` throws (transport error, unavailable adapter), the service logs a warning and returns the
attempt **still PENDING** with **no** `providerTransactionId` and **no** `checkoutUrl`. The order stays PENDING. The
learner retries with the same `clientRequestId`: the stable attempt is returned and the id is attached (§73). No
double-charge, no lost attempt.

## 11. Attempt idempotency (TD-187 / §67)
Initiate requires `clientRequestId`. Partial UNIQUE `uq_payment_transaction_client_request` (payment_order_id,
client_request_id) `WHERE NOT NULL` (PT-DB-01) makes replay durable: same (order, key) → the existing attempt
(`needsProviderInit` iff its id is still NULL). Concurrent identical requests race to P2002 → winner-reload → exactly
one attempt (§50).

## 12. Idempotency conflict (§53 / §68)
Same `clientRequestId` with a **different** provider → `PaymentAttemptRequestConflictError`
(**409 PAYMENT_ATTEMPT_REQUEST_CONFLICT**). An idempotency key binds one provider; it is never silently re-pointed.

## 13. One PENDING attempt per order (TD-187 / §54 / §69)
Partial UNIQUE `uq_payment_transaction_pending` (payment_order_id) `WHERE status = 'PENDING'` (PT-DB-02) allows at most
one in-flight attempt. Because the first initiation moves the order to PENDING, a **new** key against that order fails
eligibility (`status != CREATED` → 409) — a second concurrent provider is never started.

## 14. Committed-discount integrity (§44 / §64)
If the order carries a committed discount (`izlRedemptionId` set), `assertCommittedDiscountIntegrity` revalidates
inside the attempt transaction: the redemption is RESERVED, points back to this order, belongs to the user,
`valueUzs === izlDiscountAmount`, `payableAmount === gross − discount`, and its reservation is ACTIVE with matching
`amountIzl`. A corrupted binding → 409 (no attempt). Initiate itself does **not** mutate the redemption/reservation.

## 15. Reserved economy is untouched (§64 / §56)
Initiating a payment for a discounted order does **not** spend IZL: the redemption stays RESERVED, the reservation
stays ACTIVE, and the `IZLLedgerEntry` history is unchanged. The e2e asserts zero ledger movement + RESERVED/ACTIVE
after initiation. Spend happens only at verified-payment finalization (Phase 2.1F).

## 16. Finance locking (§25)
Both transactions run under `pg_advisory_xact_lock(hashtext('izl'), hashtext(userId))` — the standardized per-user IZL
namespace reused (permitted by §25) so payment initiation **serializes with redemption commit/release**. This closes an
order-state race: committed release requires the order still `CREATED`, and initiate transitions `CREATED → PENDING`;
under the shared lock the two can never interleave into a split state.

## 17. Errors + HTTP mapping (§48)
`PaymentOrderNotFoundError → 404` (foreign/absent order, IDOR-safe); `PaymentOrderNotEligibleError → 409
PAYMENT_ORDER_NOT_ELIGIBLE`; `PaymentAttemptRequestConflictError → 409 PAYMENT_ATTEMPT_REQUEST_CONFLICT`.
`PaymentProviderUnavailableError` is **not** HTTP-mapped — it is caught inside the service (§10) so an unavailable
provider still yields a 201 with a PENDING attempt, not a 5xx.

## 18. Initiation view (§34)
The response is learner-safe: `{ paymentOrder: {id, status, payableAmount, currency}, paymentTransaction: {id,
provider, status}, checkoutUrl? }`. `checkoutUrl` is present only when the provider init succeeded. No lock keys,
reservation ids, policy codes, rate ids, or provider metadata leak.

## 19. Public input (TD-186 / §4 / §79)
`InitiatePaymentDto = { provider: @IsEnum(PaymentProvider), clientRequestId: @IsString @IsNotEmpty @MaxLength(200) }`.
The global `ValidationPipe({ whitelist, forbidNonWhitelisted })` rejects any injected economic field (`amount`,
`status`, `providerTransactionId`, …) → **400**. Everything economic is server-derived.

## 20. Schema changes (migration 18)
`PaymentTransaction`: `providerTransactionId` made **nullable** (attached after the external init, so the internal
PENDING attempt exists first — crash/retry safety); added `clientRequestId` (idempotency). `@@unique([provider,
providerTransactionId])` retained (NULLs distinct, so many pending-no-id rows coexist across orders). No currency
column added.

## 21. Custom SQL (PT-DB-01 / PT-DB-02)
Appended to the migration (Prisma does not model partial uniques):
`CREATE UNIQUE INDEX uq_payment_transaction_client_request ON payment_transaction (payment_order_id, client_request_id)
WHERE client_request_id IS NOT NULL;` (PT-DB-01) and
`CREATE UNIQUE INDEX uq_payment_transaction_pending ON payment_transaction (payment_order_id) WHERE status = 'PENDING';`
(PT-DB-02). Both verified present in `izlan_dev` + `izlan_test`.

## 22. Single writer (§56)
`PaymentTransaction` is written **only** in `payments.repository.ts` (create + provider-init attach). `PaymentOrder` is
written only by the payments module (create + `CREATED → PENDING`) and the pre-existing 2.1D redemption commit/release
(pricing). No other module touches either.

## 23. Boundary — what this phase does NOT do (TD-188 / §86)
No `PaymentCallbackEvent`, no `SUCCEEDED` producer, no `PaymentOrder` PAID, no IZL `REDEEM`/spend ledger, no reservation
`CONSUMED`, no redemption `APPLIED`, no `Subscription`/`SubscriptionCycle` activation, no real provider API. Greps
confirm none of these are written in `src/payments/`. This is the entire finalization surface of Phase 2.1F.

## 24. Tests + gate
- **e2e** (`test/payment-execution.e2e-spec.ts`, 10, §63–§79): basic CLICK initiate (order PENDING, attempt PENDING,
  amount = payable, provider id attached, checkoutUrl; no PAID/callback/SUCCEEDED/subscription/IZL); discounted-order
  initiate (charge = payable, RESERVED redemption + ACTIVE hold + ledger unchanged); idempotent replay (one attempt,
  same provider id); different-provider conflict (409); one-PENDING enforcement (new key on PENDING order → 409);
  concurrent identical + concurrent different-provider (exactly one PENDING); ambiguous provider init (PENDING, no id) +
  retry attaches id (same attempt); ineligible orders (expired / PAID / already-PENDING → 409); zero payable → 409, no
  attempt; foreign order 404; 401 unauth; client field-injection → 400.
- Test adapter overrides `PAYMENT_PROVIDER_PORT` with `TestPaymentProviderAdapter` (deterministic id from the stable
  `paymentTransactionId`; `failMode` toggles the ambiguous-failure path).
- Gate: **361 unit + 330 e2e PASS**; `tsc` clean; migration 18; drift clean (empty `migrate diff`); PT-DB-01/02 present
  in dev + test.

## 25. Verified evidence (Phase 2.1F) + finalization (Phase 2.1G)
**Phase 2.1F (COMPLETE)** adds the trusted-success half: an adapter-verified callback transitions the attempt
`PENDING → SUCCEEDED` and records a `PaymentCallbackEvent`, with a one-SUCCEEDED-per-order invariant (PV-DB-01). See
[PAYMENT_VERIFICATION_IMPLEMENTATION.md](PAYMENT_VERIFICATION_IMPLEMENTATION.md). Crucially the order still stays
`PENDING` and no IZL is spent — SUCCEEDED is trusted evidence, not economic activation.

Still deferred to **Phase 2.1G** (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): real Click/Payme adapters + webhook
routes, and the single atomic finalization — `PaymentOrder` PAID + IZL `REDEEM` ledger `−amountIzl` + reservation
`CONSUMED` + redemption `APPLIED` + `Subscription` + `SubscriptionCycle` (one verified payment = one cycle, F-10;
frontend never activates, F-9) — plus FAILED/cancellation / reversal / refund / expiry semantics.
