# Payment Order Reopen / Retry Foundation — Implementation (Phase 2.1J)

> Status: **COMPLETE** (2026-08-21). A definitively terminal (FAILED/CANCELLED) `PaymentTransaction` can make its own
> **PENDING** `PaymentOrder` retryable again by transitioning **PENDING → CREATED** — the only field written. The
> learner then retries via the existing `initiate` flow with a fresh `clientRequestId`. **No new attempt is auto-created,
> no provider call, no IZL release, no PaymentOrder FAILED/CANCELLED, no schema change.** Owner: **TD-222..227**.
> Migration count stays **20**.
>
> **Phase 2.1L-D forward-note (2026-08-21):** retry may switch provider (PT.provider per-attempt). Each real attempt will
> own a distinct provider protocol row (`PaymeMerchantTransaction` / `ClickShopTransaction`, 1:1 with PaymentTransaction,
> TD-233), so a reopened order's fresh attempt binds a fresh provider transaction id via the non-terminal binding
> primitive (TD-234). See [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md).

Code: `backend/src/payments/` — `payment-order-reopen.repository.ts` (the reopen transaction),
`payment-order-reopen.service.ts` (wrapper + post-terminal bridge helper), `payment-callback.service.ts` (2.1I → 2.1J
bridge).

## 1. Provider truth vs retry policy (TD-222 / §2)
Mirroring the success split (evidence in 2.1F, finalization in 2.1G), 2.1I commits provider truth (`PT FAILED/CANCELLED`)
and 2.1J reopens the order in a **separate** transaction. Evidence and business retry policy are decoupled.

## 2. Reopen authority (§3 / §5)
Input is the internal `PaymentTransaction.id` only; everything else is loaded from the DB. The PT must be `FAILED` or
`CANCELLED` (PENDING/SUCCEEDED/REFUNDED are ineligible). No client supplies status/order/provider/failure/"reopen".

## 3. FAILED / CANCELLED eligibility (§5 / §68 / §69)
Both definitive terminal non-success states authorize reopen identically (a provider-confirmed CANCELLED needs no special
release, §58).

## 4. Ambiguous PENDING exclusion (§59 / §82)
A PENDING transaction is never reopen-eligible (`NOT_REOPENABLE`) — ambiguous evidence stays PENDING (2.1I / 2.1E).

## 5. PENDING → CREATED (§9)
Under `pg_advisory_xact_lock('pay', paymentOrderId)`, after reloading, the order transitions `PENDING → CREATED`. Only
`PaymentOrder.status` is written.

## 6. No PENDING PT rule (TD-227 / §10 / §48 / §73)
Reopen is refused (`RETRY_ALREADY_IN_PROGRESS`) whenever a live `PENDING` transaction exists for the order — a stale
terminal callback must never reopen an order that has an active charge path, or a duplicate-charge window would open.

## 7. No SUCCEEDED PT rule (TD-227 / §11 / §49 / §74)
Reopen is refused (`PAYMENT_SUCCESS_PENDING_FINALIZATION`) whenever any `SUCCEEDED` transaction exists — that order
belongs to finalization/recovery (2.1G/2.1H), never to retry.

## 8. PAID protection (§12 / §50 / §75)
A PAID order never reopens (`ALREADY_PAID`); a historical failed/cancelled attempt cannot undo a paid purchase /
subscription.

## 9. Expiry semantics (§18 / §53 / §81)
Reopen does NOT read `expiresAt`; a definitively terminal attempt reopens `PENDING → CREATED` even for a long-expired
order. The existing 2.1E `initiate` then rejects a fresh attempt on the expired quote, and the 2.1D committed-discount
release becomes usable again — no `PaymentOrder.EXPIRED` producer is invented.

## 10. Frozen commercial state (§19 / §21)
Reopen never mutates plan/planPrice/gross/discount/payable/currency/`izlRedemptionId`/provider/expiresAt/clientRequestId/
purpose — only `status`. Retry is a payment-execution retry against the exact frozen purchase, not a new quote.

## 11. Discount hold (TD-225 / §20 / §79)
For a discounted order the `IZLRedemption` stays RESERVED, the `IZLReservation` ACTIVE, the ledger and pricing unchanged.
A failed attempt ended one provider try, not the purchase intent, so the hold is preserved for an immediate retry at the
same payable amount.

## 12. Explicit release reuse (TD-225 / §22 / §80)
Because the order is CREATED again, the existing 2.1D `SubscriptionDiscountRedemptionService.release` unwinds the
discount unchanged (order → discount 0 / payable gross / pointer NULL, redemption+reservation RELEASED). No 2.1J-specific
release logic.

## 13. Old / new clientRequestId (TD-224 / §30 / §31 / §76 / §77)
A fresh retry uses a **new** clientRequestId → a new PENDING PT (via the existing `initiate`, whose replay check runs
before eligibility so it never mistakes a fresh key for a replay). Reusing the **old** key returns the old terminal
attempt (PT-DB-01) — no new PT.

## 14. Same / different provider (TD-224 / §32 / §33 / §78)
Retry may keep CLICK or switch to PAYME (`PaymentTransaction.provider` is the per-attempt authority; `PaymentOrder.provider`
stays NULL). Reusing the old key with a *different* provider is a conflict (the key is bound to the old attempt's
provider).

## 15. Stale terminal callback protection (TD-227 / §48 / §49 / §50)
Eligibility is order-wide absence of a live/success attempt, evaluated under the `pay(order)` lock and re-read after the
lock. A replayed old terminal reopen can never overwrite a newer PENDING retry, a SUCCEEDED evidence, or a PAID order.

## 16. No latest pointer (TD-227 / §51 / §52)
No `latestAttemptId`/`reopenedAt`/`retryCount` field is added. Retry safety depends on the DB relational authorities
(PT-DB-02 one PENDING, PV-DB-01 one SUCCEEDED), not a mutable latest-attempt pointer.

## 17. Post-terminal bridge (TD-226 / §24 / §70)
After the 2.1I terminal callback transaction **commits**, `PaymentCallbackService` calls
`reopen.tryReopenAfterTerminal(paymentTransactionId)` in a **separate**, non-throwing transaction — only for
accepted/matching terminal outcomes (ACCEPTED/DUPLICATE with FAILED/CANCELLED), never for conflicts/mismatches (§28).

## 18. Bridge failure (TD-226 / §25 / §71)
A reopen failure never rolls back or rewrites the provider evidence (`PaymentCallbackEvent` / `PT FAILED/CANCELLED`) — the
order stays PENDING and recoverable.

## 19. Callback replay recovery (TD-226 / §27 / §72)
A matching terminal callback replay (PT already terminal, order still PENDING) retries reopen idempotently — no second PT
transition, no duplicate effect.

## 20. Concurrency (§46 / §83)
Two concurrent reopen calls serialize on `pay(order)`: one `REOPENED`, one `ALREADY_REOPENED`; no duplicate writes. A
reopen/initiate race is safe — initiate before reopen sees PENDING and rejects; after reopen it creates one PENDING PT
(§47/§84).

## 21. No auto charge (§42)
Reopen only transitions `PENDING → CREATED`; it never creates PT-B. The learner decides when to retry via `initiate`,
preventing silent re-charge.

## 22. Security (§61 / §85)
No learner reopen route; the reopen service is internal/server-owned. The only learner-visible retry is the existing
`initiate` once the server has made the order CREATED. Reopen makes no provider call (spy-tested).

## 23. Tests
- **unit** (`payment-order-reopen.service.spec.ts`, 2): delegation; `tryReopenAfterTerminal` non-throwing.
- **e2e** (`payment-order-reopen.e2e-spec.ts`, 14, §59–§85): FAILED/CANCELLED reopen (terminal PT untouched); callback
  bridge auto-reopen (no provider call); bridge failure preserves evidence + matching-replay recovery; stale reopen vs
  newer PENDING / SUCCEEDED / PAID; PENDING/SUCCEEDED not eligible; idempotent + concurrent reopen; retry new key → new
  PT / old key → old attempt; provider switch (+ old-key-different-provider conflict); discounted reopen keeps
  RESERVED/ACTIVE/ledger; 2.1D release after reopen; expired order reopens then initiate rejected; writes-only-order-status.
- **Gate:** 386 unit + 405 e2e PASS; `tsc` clean; **no new migration** (count 20); drift clean. Regression: 2.1D release,
  2.1E initiate, 2.1F/2.1G/2.1H/2.1I, XP/learning green.

## 24. Operational stuck-reopen gap → closed by Phase 2.1K
If terminal evidence commits, the reopen bridge fails, and no provider callback ever replays, the state remains `PT
FAILED/CANCELLED + Order PENDING` — safe but retry-blocked.

> **Phase 2.1K update (2026-08-21):** this stuck state is now operationally drainable — an internal/admin reopen
> reconciler retries stuck orders through **this same `PaymentOrderReopenService`** (no second writer, no provider call).
> See [PAYMENT_REOPEN_RECOVERY_IMPLEMENTATION.md](PAYMENT_REOPEN_RECOVERY_IMPLEMENTATION.md). A background scheduler is
> still deferred.

## 25. Future purchase cancellation / failure lifecycle
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): terminal-reopen operational reconciliation/admin tooling,
background retry/scheduler, local timeout, provider polling, `PaymentOrder FAILED/CANCELLED` producers, purchase-intent
cancellation, reservation/redemption TTL, refund/reversal/chargeback, real Click/Payme, renewal, notifications, frontend.
