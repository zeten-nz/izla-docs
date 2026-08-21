# Verified Payment Evidence — Implementation (Phase 2.1F)

> Status: **COMPLETE** (2026-08-20). A provider-verified SUCCESS becomes durable **trusted payment evidence** and
> nothing more: the adapter authenticates a callback, the business layer validates identity/amount/currency, and the
> `PaymentTransaction` transitions **PENDING → SUCCEEDED** (with a `PaymentCallbackEvent` provenance row). The
> `PaymentOrder` stays **PENDING** — no PAID, no IZL debit, no reservation CONSUMED, no redemption APPLIED, no
> Subscription/Cycle, no real Click/Payme. Owner: **TD-189/190/191/192/193/194**. Migration
> `20260820250000_verified_payment_evidence` (migration 19, custom SQL only — PV-DB-01).
>
> **Phase 2.1L-D forward-note (2026-08-21):** a future real adapter feeds this SUCCESS evidence path from a terminal
> provider step (Payme PerformTransaction / CLICK Complete) whose durable protocol state now lives in
> `PaymeMerchantTransaction` / `ClickShopTransaction` (TD-233). Terminal `providerEventId` is centralized
> (`PAYME:{id}:PERFORM`, `CLICK:{clickTransId}:COMPLETE`); the non-terminal id attach (Prepare / CreateTransaction) is
> the separate binding primitive (TD-234), not this terminal writer. See [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md).

Code: `backend/src/payments/` — `payment-callback.service.ts` (`processProviderCallback`), `payments.repository.ts`
(`recordVerifiedCallback` + helpers), `provider/payment-provider.port.ts` (`verifyCallback`, `PaymentCallbackInput`,
`VerifiedPaymentProviderEvent`), `provider/unavailable-payment-provider.adapter.ts`. Test adapter:
`test/test-payment-provider.adapter.ts` (`verifyCallback`).

## 1. Trusted success definition (TD-189 / §2)
The durable trusted-success marker is `PaymentTransaction.status = SUCCEEDED`. It is produced **only** by an
adapter-verified provider callback that passes every business-integrity check. There is no other producer, and no
learner input can create it (§45).

## 2. Verified ≠ finalized (TD-194 / §11 / §31)
This phase answers *"has a trusted provider verified payment success?"* — not *"have all Izlan economic effects been
finalized?"*. A SUCCEEDED transaction is trusted evidence; economic finalization (PaymentOrder PAID + IZL + Subscription)
is the separate Phase 2.1G.

## 3. Provider verification port (TD-190 / §4)
`PaymentProviderPort.verifyCallback(input)` is provider-neutral. The adapter owns signature/authentication, payload
parsing, provider status mapping, and identity extraction. The business layer never parses Click/Payme semantics. Real
adapters are deferred (§8); only a deterministic `TestPaymentProviderAdapter` and a throwing
`UnavailablePaymentProviderAdapter` exist.

## 4. Normalized verified result (§6)
`VerifiedPaymentProviderEvent { provider, providerEventId, merchantPaymentTransactionId, providerTransactionId,
status:'SUCCEEDED', amount, currency, confirmedAt, metadata? }`. The callback envelope (`PaymentCallbackInput {
provider, payload, headers?, query? }`) is **opaque** to the business layer (§5) — raw headers/secrets are never
persisted.

## 5. Merchant / internal transaction identity (TD-191 / §7)
`merchantPaymentTransactionId` is our stable `PaymentTransaction.id` supplied to the adapter at initiation. It is the
primary internal identity used to resolve the exact attempt — callbacks are **never** resolved by amount/user/order
guessing. A non-UUID merchant id is a malformed identity (verification rejection, no writes, §10); a well-formed but
unknown id resolves to no transaction → `IDENTITY_MISMATCH` (§61).

## 6. Provider transaction identity (§16 / §17)
`providerTransactionId` is the external identity. If the transaction has none yet (ambiguous 2.1E init), a valid
callback attaches it in the same processing transaction (§63). An existing different id is never overwritten
(`EXTERNAL_ID_CONFLICT`, §64); an id already owned by another transaction is rejected (`@@unique(provider,
provider_transaction_id)`, §17).

## 7. Callback dedup (TD-192 / §12)
`@@unique(provider, provider_event_id)` (F-19) is the dedup authority. Processing verifies the callback **outside** the
DB transaction, then in a short transaction under the payment lock attempts the callback insert. An exact provider-event
replay (§29/§65) returns the stored outcome without repeating any mutation.

## 8. Callback event semantics (§13)
`PaymentCallbackEvent.result` records the classification (`ACCEPTED`, `DUPLICATE`, or a rejection code such as
`AMOUNT_MISMATCH` / `CURRENCY_MISMATCH` / `IDENTITY_MISMATCH` / `SUCCESS_CONFLICT` / `EXTERNAL_ID_CONFLICT` /
`SUCCESS_DATA_CONFLICT`), with `processedAt` set and `paymentTransactionId` linked to the resolved transaction (null
when identity cannot resolve). No secrets or raw payloads are stored.

## 9. Amount integrity (TD-191 / §22)
Two equalities must hold: `verified.amount === PaymentTransaction.amount` **and** `PaymentTransaction.amount ===
PaymentOrder.payableAmount`. The second detects post-initiation order corruption (§59). Any mismatch → no SUCCEEDED, no
downstream (`AMOUNT_MISMATCH` / `ORDER_AMOUNT_MISMATCH`).

## 10. Currency integrity (TD-191 / §23)
`PaymentTransaction` has no currency column (Phase 2.1E), so `verified.currency === PaymentOrder.currency` is the
authority. Mismatch → `CURRENCY_MISMATCH`, no success. No PT currency column was invented for 2.1F.

## 11. Discounted payable authority (§33 / §55)
For a discounted order the charge equals `PaymentOrder.payableAmount` (the committed post-discount value). Verifying its
payment does not touch the discount: the redemption stays RESERVED, the reservation ACTIVE, the IZL ledger unchanged.

## 12. Provider id attach after ambiguous initiation (§16 / §63)
A 2.1E ambiguous init can leave `providerTransactionId = NULL` while the provider actually created the external
transaction. The merchant id still resolves the exact attempt; the verified callback attaches the external id and
transitions to SUCCEEDED atomically.

## 13. PENDING → SUCCEEDED (§9 / §20)
Only a `PENDING` transaction transitions. `confirmedAt` is set from the trusted `verified.confirmedAt` (never server
`now`, never client input). The accepted callback insert + the transition commit **atomically** (§14) — never a
callback marked accepted with the transaction still PENDING, and never SUCCEEDED without its callback provenance.

## 14. One success per order (TD-193 / §25)
PV-DB-01: partial UNIQUE `uq_payment_transaction_succeeded (payment_order_id) WHERE status='SUCCEEDED'` — a load-bearing
financial invariant. A proactive check rejects a second success (`SUCCESS_CONFLICT`, §27); the DB index is the backstop
(§26 — a direct double-SUCCEEDED update raises P2002). Multiple historical attempts remain possible; exactly one may be
the successful charge authority.

## 15. Repeated provider success (§28 / §66)
A distinct event id for an already-SUCCEEDED transaction that matches (same provider id, amount, currency) is recorded
as a `DUPLICATE` no-op — the transaction and its `confirmedAt` are untouched. A distinct event that claims different
data → `SUCCESS_DATA_CONFLICT` (§30); accepted history is never mutated.

## 16. confirmedAt (§20 / §21 / §68)
The first accepted success sets `confirmedAt` from the trusted normalized result; a later matching event never replaces
it. Client input is impossible (no learner route). e2e §68 asserts immutability.

## 17. Verification outside the DB transaction (§39 / §70)
`processProviderCallback` calls `provider.verifyCallback(...)` and only after it resolves calls
`repo.recordVerifiedCallback(...)`, which opens the transaction and acquires the lock. Verification (crypto / parsing /
future network) never runs while a DB transaction or advisory lock is held. e2e §70 asserts the `verify → record`
ordering.

## 18. Callback processing transaction (§38 / §40)
The authoritative processing runs in one short transaction under `pg_advisory_xact_lock(hashtext('pay'),
hashtext(paymentOrderId))` — a **payment-scoped** lock (not the IZL user lock; no IZL mutation exists here). All events
and attempts of one order serialize, so replays, distinct-success events, and second-attempt successes converge without
races.

## 19. PaymentOrder remains PENDING (§31 / §32)
CRITICAL: after the transition the order is still `PENDING`. Trusted provider success exists, but Izlan economic
activation has not been finalized — so the evidence stays durably recoverable if a later finalization dependency is
missing. The callback path performs **no** `PaymentOrder` write.

## 20. IZL remains reserved (§35 / §72)
The callback path writes no `IZLLedgerEntry` / `IZLReservation` / `IZLRedemption`. `balanceIzl` / `reservedIzl` /
`availableIzl` are unchanged by verified success alone.

## 21. No Subscription (§33 / §73)
No `Subscription` or `SubscriptionCycle` is created. Verified evidence is not activation.

## 22. Failure statuses deferred (TD-189 / §3 / §36 / §74)
Success-only v1. There is no production producer for `FAILED` / `CANCELLED` / `REFUNDED` and no retry/reopen (`PENDING →
CREATED`) path; the enum values remain but are unused. A non-success normalized status is rejected as unsupported
verification (no writes). Failure/retry/late-success/refund semantics need their own owner-reviewed contract.

## 23. Security (§45 / §46 / §75)
No learner-authenticated payment-success route exists; `PaymentCallbackService` has **no controller** — it is
internal/provider-facing only (future Click/Payme controllers call it). Learners keep polling `GET /payments/orders/:id`.
No provider metadata / raw callback / lock keys leak through order/IZL/redemption reads or logs.

## 24. Tests
- **e2e** (`test/payment-verification.e2e-spec.ts`, 15 — invokes `PaymentCallbackService` directly, seeds via Prisma;
  no fake public webhook route, §9/§46): verified success (SUCCEEDED + confirmedAt, order PENDING, no PAID/subscription/
  IZL); discounted success (RESERVED + ACTIVE + ledger unchanged); external-id attach; invalid verification (throws,
  zero writes); business rejections (amount / order-corruption / currency / provider); malformed (non-UUID) identity;
  wrong (unknown) merchant id; external-id conflict (no overwrite); exact replay (one record, one transition); distinct
  success events (DUPLICATE no-op + confirmedAt immutable); success-data conflict after SUCCEEDED; second-attempt
  success conflict; PV-DB-01 DB constraint (P2002 on a second SUCCEEDED); concurrent identical callbacks (one
  transition); verification-before-transaction ordering.
- Test adapter overrides `PAYMENT_PROVIDER_PORT`; `verifyCallback` normalizes a fixture payload and throws on
  `signatureValid:false`.
- Gate: **361 unit + 345 e2e PASS**; `tsc` clean; migration 19; drift clean (empty `migrate diff`); PV-DB-01 present in
  dev + test.
- Atomicity note: literal mid-transaction fault injection (§69) is not deterministically reproducible in the harness;
  no-split-state is guaranteed structurally (one `$transaction`) and exercised by the concurrency + all-or-nothing
  rejection tests.

## 25. Phase 2.1G boundary
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): real Click/Payme verification + webhook routes; FAILED/CANCELLED/
REFUNDED, retry, timeout, late-success, order expiry; and the atomic economic finalization — `PaymentOrder` PAID + IZL
`REDEEM` ledger + reservation `CONSUMED` + redemption `APPLIED` + `Subscription` + `SubscriptionCycle` (reward-policy
selection + earning-ceiling snapshot at cycle creation); plus refund / reversal / chargeback / notification / frontend.

> **Phase 2.1G-D update (2026-08-21):** the finalization *contract + schema inputs* are now frozen (TD-195..204, see
> [PAYMENT_FINALIZATION_CONTRACT.md](PAYMENT_FINALIZATION_CONTRACT.md)) — billing duration, calendar-month period,
> reward basis/ceiling, reward-disabled cycle, CONSUMED, one-REDEEM uniqueness, subscription activation cases, and the
> `sub → pay → izl` lock order. This does **not** change 2.1F: verified success is still trusted evidence only, the
> order stays PENDING, and there is still no PAID/IZL/Subscription producer. The finalizer itself is Phase 2.1G.

> **Phase 2.1G update (2026-08-21):** `processProviderCallback` now runs a **post-verification bridge** (TD-210): once a
> trusted SUCCEEDED transaction is committed, it best-effort calls the finalizer (`tryFinalizeAfterVerification`) in a
> **separate** transaction (non-throwing). So in the wired system a verified callback for an eligible order goes all the
> way to PAID + Subscription + Cycle (see [PAYMENT_FINALIZATION_IMPLEMENTATION.md](PAYMENT_FINALIZATION_IMPLEMENTATION.md)).
> A finalization failure never rolls back or erases the verified-payment evidence — the order simply stays PENDING and
> recoverable. The `payment-verification.e2e` suite stubs the finalization service so it still tests the verification
> layer in isolation (order stays PENDING); the real bridge is covered by `payment-finalization.e2e`.

> **Phase 2.1I update (2026-08-21):** the normalized event now carries a canonical status (SUCCEEDED/FAILED/CANCELLED)
> and a `terminal` flag, and `processProviderCallback` routes a trusted **definitive** FAILED/CANCELLED to a non-success
> pipeline (`PaymentTransaction PENDING → FAILED/CANCELLED`, order stays PENDING, no bridge). Terminal states are
> immutable — a late contradictory event (e.g. SUCCESS after FAILED, or FAILURE after SUCCEEDED) is recorded as
> `TERMINAL_STATUS_CONFLICT` and never rewrites accepted history. See
> [PAYMENT_NON_SUCCESS_VERIFICATION_IMPLEMENTATION.md](PAYMENT_NON_SUCCESS_VERIFICATION_IMPLEMENTATION.md).
