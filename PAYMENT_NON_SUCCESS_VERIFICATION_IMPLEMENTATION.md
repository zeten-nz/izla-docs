# Verified Non-Success Payment Evidence — Implementation (Phase 2.1I)

> Status: **COMPLETE** (2026-08-21). A trusted, adapter-verified **definitive** provider terminal non-success event
> transitions `PaymentTransaction PENDING → FAILED / CANCELLED` and records `PaymentCallbackEvent` evidence. It is
> provider truth only: **the PaymentOrder stays PENDING** — no reopen, no retry, no IZL movement, no refund, no
> finalizer, no real Click/Payme. Owner: **TD-216..221**. **No schema change** (migration count stays **20**).
>
> **Phase 2.1L-D forward-note (2026-08-21):** a future real adapter feeds definitive non-success here from Payme
> CancelTransaction (state -1, semantic reason → FAILED, never auto-CANCELLED, TD-236) / CLICK Complete error, with
> durable protocol state in `PaymeMerchantTransaction` (TD-233). Payme **post-success** cancellation (state -2) is NOT
> routed here — it is `REFUND_DOMAIN_UNSUPPORTED` (future refund domain, TD-238). See [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md).

Code: `backend/src/payments/` — `provider/payment-provider.port.ts` (`NormalizedPaymentStatus`, extended
`VerifiedPaymentProviderEvent`), `payment-callback.service.ts` (`assertVerifiedEvent` + non-success routing),
`payments.repository.ts` (`recordTerminalNonSuccess`). Test adapter: `test/test-payment-provider.adapter.ts`.

## 1. Why non-success evidence is separate from retry (TD-216 / TD-221)
Mirroring the success architecture (evidence in 2.1F, finalization in 2.1G), a verified provider failure is *provider
truth* (`PT FAILED/CANCELLED`), while order reopening / retry is an internal *business decision* (Phase 2.1J). Keeping
them separate isolates the risk and keeps terminal states clean.

## 2. Definitive vs ambiguous (TD-217 / §4 / §5)
FAILED/CANCELLED require the adapter to assert `terminal = true` — this exact provider transaction can no longer normally
become payable. Ambiguous transport/timeout/no-response/processing/browser-abandonment must NOT be normalized as
terminal non-success; the transaction stays PENDING (`assertVerifiedEvent` rejects a non-terminal non-success as
"unsupported").

## 3. Invalid callback distinction (§9 / §10)
A failed signature/authentication (adapter throws) → `PaymentCallbackVerificationError`, zero writes (2.1F regression
preserved). A *successfully authenticated* provider FAILED/CANCELLED event is legitimate trusted evidence and is routed
to the non-success pipeline, **not** through generic invalid-verification handling.

## 4. Normalized provider contract (§3)
`NormalizedPaymentStatus = 'SUCCEEDED' | 'FAILED' | 'CANCELLED'`. `VerifiedPaymentProviderEvent` adds `terminal?`,
`reasonCode?`, and makes `amount?`/`currency?`/`providerTransactionId?` optional (required only for SUCCEEDED). Provider-
specific statuses never leak past the adapter.

## 5. Terminal assertion (§4)
The business layer accepts FAILED/CANCELLED only when `terminal === true`; otherwise it rejects before any DB write.

## 6. FAILED semantics (§6 / §40)
`status = FAILED, terminal = true` → `PaymentTransaction PENDING → FAILED`; PaymentOrder unchanged (PENDING); callback
`result = ACCEPTED_FAILED`.

## 7. CANCELLED semantics (§7 / §41)
`status = CANCELLED, terminal = true` (provider-confirmed) → `PaymentTransaction PENDING → CANCELLED`; PaymentOrder
unchanged; callback `result = ACCEPTED_CANCELLED`. Learner/local UX cancel is never sufficient (TD-218).

## 8. Provider expiry mapping (§8 / §42)
A provider-side definitive expiry normalizes to `FAILED` with `reasonCode = PROVIDER_EXPIRED`; the callback stores
`result = ACCEPTED_FAILED:PROVIDER_EXPIRED`. No PT `EXPIRED` enum is added in v1.

## 9. Merchant identity (§11)
Resolved by the stable internal `PaymentTransaction.id` (`merchantPaymentTransactionId`) — never by amount/order/user
guessing (same as 2.1F success).

## 10. External transaction identity (§13 / §54)
If `providerTransactionId` is NULL and a trusted terminal callback supplies a valid external id, it is attached
atomically with the transition; an existing matching id continues; a different id → `EXTERNAL_ID_CONFLICT`; an id owned
by another PT → conflict. Never overwritten.

## 11. Callback dedup (§15)
`@@unique(provider, provider_event_id)` (F-19) reused. Exact provider-event replay returns the stored outcome; a
matching distinct event for an already-terminal PT is a `DUPLICATE` no-op.

## 12. Atomic transition (§17)
The accepted callback insert + `PENDING → FAILED/CANCELLED` (+ external-id attach) commit in one short transaction under
`pg_advisory_xact_lock('pay', paymentOrderId)`. The order is **not** written.

## 13. Terminal immutability (TD-219 / §19)
`SUCCEEDED / FAILED / CANCELLED` are immutable accepted terminal truths. All cross-terminal transitions are forbidden
(FAILED→SUCCEEDED, CANCELLED→SUCCEEDED, SUCCEEDED→FAILED/CANCELLED, FAILED↔CANCELLED). Arrival order never rewrites
accepted history.

## 14. Contradictory late events (§21 / §22 / §47 / §48 / §49)
A late SUCCESS after FAILED/CANCELLED → `TERMINAL_STATUS_CONFLICT` (success path now rejects a non-PENDING PT with that
code); PT unchanged, order PENDING, **no finalizer bridge** (the bridge only runs on a SUCCEEDED outcome). A late
FAILED/CANCELLED after SUCCEEDED → `TERMINAL_STATUS_CONFLICT`; PT stays SUCCEEDED; a paid purchase is never undone.

## 15. Success/failure race (§23 / §50)
Concurrent trusted SUCCESS + FAILED for one PENDING PT serialize on `pay(order)`. Whichever terminal state committed
first is immutable; the other becomes conflict evidence. Exactly one transition; the test asserts one accepted, never
which wins.

## 16. Amount / currency asymmetry (TD-220 / §14)
SUCCESS requires exact `amount` + `currency` equality. A trusted non-success event **may omit** amount/currency (the
provider contract may not supply an authoritative charged amount); when present they are validated (mismatch →
rejected), when absent it is allowed. Merchant/provider/external identity is always required.

## 17. No failure timestamps / reason columns (§25 / §26 / TD-221)
No `failedAt`/`cancelledAt`/`terminalAt`/`failureReasonCode` column is added. Audit authority = the trusted
`PaymentCallbackEvent.receivedAt`/`processedAt`/`result` + the PT status. `confirmedAt` stays success-only (never set on
FAILED/CANCELLED).

## 18. PaymentOrder remains PENDING (§10 / §27)
For an accepted FAILED/CANCELLED the order stays PENDING — never CREATED/FAILED/CANCELLED/EXPIRED. Evidence is separate
from retry/business policy.

## 19. Discount hold unchanged (§12 / §28 / §51)
For a discounted order the `IZLRedemption` stays RESERVED, the `IZLReservation` ACTIVE, the order pricing/pointer and the
IZL ledger unchanged, no wallet recompute.

## 20. No retry (§30 / §53)
No new `PaymentTransaction` is created and `initiate` is not called. After FAILED the order is still PENDING, so a fresh
`initiate` still rejects (`PaymentOrderNotEligibleError` — order not CREATED), proving the 2.1J boundary.

## 21. No finalization (§31)
Only a SUCCEEDED outcome triggers `PaymentFinalizationService`; FAILED/CANCELLED never invoke the finalizer.

## 22. 2.1H interaction (§32 / §52)
The finalization backlog is `SUCCEEDED PT + PENDING order`; a FAILED/CANCELLED PT (status filter) never appears — a
regression test asserts `listBacklog` returns nothing for terminal non-success.

## 23. Security (§55)
There is no learner route to submit FAILED/CANCELLED/`terminal=true`/provider events; the provider-facing service is
internal/test-only until real adapters. No generic learner mark-failed route.

## 24. Tests
- **e2e** (`test/payment-non-success.e2e-spec.ts`, 13, §40–§54): basic FAILED (no confirmedAt, order PENDING); basic
  CANCELLED; provider expiry (PROVIDER_EXPIRED classification); ambiguous/non-terminal → verification reject (zero
  writes); invalid signature → zero writes; duplicate + distinct-matching failure (one transition, terminal no-op); late
  SUCCESS after FAILED/CANCELLED → TERMINAL_STATUS_CONFLICT (no subscription); late FAILURE after SUCCEEDED → PT stays
  SUCCEEDED; success/failure race → exactly one accepted; discounted FAILED leaves RESERVED/ACTIVE/ledger unchanged; no
  finalization backlog; no reopen (fresh initiate rejects); no provider `initiate` call.
- Test adapter emits SUCCEEDED/FAILED/CANCELLED, terminal/non-terminal, and PROVIDER_EXPIRED fixtures deterministically;
  it never writes DB state.
- **Gate:** 384 unit + 391 e2e PASS; `tsc` clean; **no new migration** (count 20); drift clean. Regression: full 2.1F
  success / 2.1G finalization / 2.1H recovery / discounted / XP / learning green.

> **Phase 2.1J update (2026-08-21):** `processProviderCallback` now runs a **post-non-success reopen bridge** (TD-226):
> after an accepted/matching terminal (FAILED/CANCELLED) outcome commits, it best-effort calls
> `tryReopenAfterTerminal` in a **separate** transaction — which reopens the order `PENDING → CREATED` so the learner can
> retry via the existing `initiate` (see [PAYMENT_RETRY_IMPLEMENTATION.md](PAYMENT_RETRY_IMPLEMENTATION.md)). A reopen
> failure never rolls back or rewrites the terminal evidence; `TERMINAL_STATUS_CONFLICT`/mismatch outcomes never trigger
> reopen. This does not change 2.1I: the terminal transition + evidence are exactly as above.

## 25. Phase 2.1J boundary
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): `PaymentOrder PENDING → CREATED` reopen, payment retry, learner
retry UX, purchase cancellation, `PaymentOrder FAILED/CANCELLED` producers, local timeout, reservation/redemption TTL,
refund/reversal/chargeback, real Click/Payme, renewal, notifications, frontend. **Next:** Phase 2.1J — Payment Order
Reopen / Retry Foundation (a definitively terminal PT + PENDING order + no PENDING/SUCCEEDED PT → CREATED, preserving
pricing + committed discount + RESERVED redemption + ACTIVE reservation).
