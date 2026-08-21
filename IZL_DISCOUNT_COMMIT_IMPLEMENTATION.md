# Subscription Discount Commit — Implementation (Phase 2.1D)

> Status: **COMPLETE** (2026-08-20). Binds a **RESERVED** redemption's frozen discount onto its own **CREATED**
> `PaymentOrder` (pricing), establishing the exact amount a future payment must charge. This is **not** a spend: the
> redemption stays RESERVED, the reservation stays ACTIVE, the IZL ledger is unchanged. A committed discount can be
> safely unwound while the order is still pre-payment. Owner: **TD-178/179/180/181/182**. Migration
> `20260820230000_subscription_discount_commit` (migration 17, custom SQL only).

Code: `backend/src/finance/redemption/` — `redemption.repository.ts` (`commitDiscount`, extended `releaseRedemption`),
`subscription-discount-redemption.service.ts`, `redemption.controller.ts`.

## 1. Why commit is separate from spend (TD-179)
A discounted but **unpaid** order is not yet realized subscription benefit. So commit only writes the discount onto
the order; IZL is consumed later, only at verified-payment finalization (Phase 2.1D+).

## 2. Order pricing authority (TD-178 / §3)
On commit, `PaymentOrder.grossAmount` stays frozen, `izlDiscountAmount = redemption.valueUzs`, `payableAmount =
gross − discount` (CHECK `chk_order_amounts`), `izlRedemptionId = redemption.id`; status stays `CREATED`, provider
unchanged.

## 3. Commit policy (§2 / §8)
`subscription-discount-commit-v1`. `POST /api/izl/redemptions/:id/commit-discount` (own-user, no body). All authority
is already persisted; the client supplies nothing.

## 4. Frozen quote reuse (§12 / §55)
The current IZL rate is **not** re-resolved. Commit uses the frozen `redemption.valueUzs`/`amountIzl`. A later active
rate change does not affect a committed order (e2e §55).

## 5. Ceiling revalidation (§12 / §56)
Only deterministic integrity is revalidated (guards historical/manual corruption): `valueUzs > 0`, `≤ floor(gross ×
2000/10000)`, `≤ gross`. No repricing (e2e §56 rejects a corrupted value).

## 6. Order pointer (TD-180 / §19)
`PaymentOrder.izlRedemptionId` is the committed pointer, with a partial UNIQUE `WHERE NOT NULL` (DC-DB-01) — one
redemption prices one order (no stacking). An order already pointing to a different redemption → `REDEMPTION_COMMIT_
CONFLICT`, never overwritten (e2e §17).

## 7. Redemption stays RESERVED (§4)
Commit does not set `APPLIED`; the redemption remains RESERVED (the quote is committed, the IZL still held).

## 8. Reservation stays ACTIVE (§5)
Commit does not consume the hold; the reservation remains ACTIVE (and the enum still has only ACTIVE/RELEASED).

## 9. Zero ledger movement (§6 / §35)
Commit writes no `IZLLedgerEntry`; the ledger balance is byte-for-byte unchanged.

## 10. Available-balance invariant (§7 / §53)
Because ledger and ACTIVE reservations are unchanged, `balanceIzl`/`reservedIzl`/`availableIzl` are identical before
and after commit (e2e asserts the GET is unchanged).

## 11. Commit idempotency (§16 / §54)
If the order already points to this redemption with consistent discount/payable and RESERVED/ACTIVE state, commit
returns the existing committed view with no further writes. Inconsistent committed pricing → `REDEMPTION_COMMIT_
CONFLICT` (§18).

## 12. Finance locking (§15)
Commit runs in one transaction under `pg_advisory_xact_lock(hashtext('izl'), hashtext(userId))` — the same per-user
IZL namespace as reservation/earning, so commit/release/create serialize without nested locks.

## 13. Release evolution (§23 / §27 — amends TD-176)
Phase 2.1C release never touched the order (orders were never discounted). Now there are two cases: an **uncommitted**
redemption releases exactly as before (independent of order status); a **committed** redemption's release also
restores the order.

## 14. Committed release (TD-181 / §25)
If `order.izlRedemptionId === redemption.id`, release — in the same transaction — requires the order still `CREATED`,
restores it (`izlDiscountAmount = 0`, `payableAmount = gross`, `izlRedemptionId = NULL`), and transitions reservation
`ACTIVE→RELEASED` + redemption `RESERVED→RELEASED` (e2e §59). Once the order is no longer CREATED, committed release
is refused (`REDEMPTION_COMMIT_CONFLICT`, e2e §25) — because a future provider may have been quoted against the
discounted payable. `expiresAt` does not block release (freeing funds / restoring the order is safe).

## 15. No APPLIED (§36)
No runtime path sets `IZLRedemption.status = APPLIED`; commit leaves it RESERVED, release sets RELEASED.

## 16. No CONSUMED (§37)
The `IzlReservationStatus` enum is unchanged (ACTIVE/RELEASED); no CONSUMED transition.

## 17. Payment sequencing (TD-182 / §38)
A future verified-payment finalization owns the atomic `REDEEM ledger −amountIzl + reservation CONSUMED + redemption
APPLIED + PaymentOrder PAID + Subscription activation`. Not implemented here.

## 18. Future payable authority (§39)
A future `PaymentTransaction` must charge `PaymentOrder.payableAmount` (post-commit), never `grossAmount`.

## 19. Concurrency (§40 / §41 / §61)
Concurrent commit calls converge to one committed order (idempotent under the lock). A commit/release race serializes:
either commit-then-release (order restored, both RELEASED) or release-then-commit-rejects — never a split state
(e2e §61 asserts internal consistency).

## 20. Security (§45 / §46 / §47)
Own-resource only (foreign redemption/order → 404); 401 unauth. Commit/release take no body — no client discount/
payable/gross/value/status/ledger authority. No internal leak (lock keys, reservation idempotency, policy, rate ids).

## 21. Tests
- **e2e** (`test/izl-discount-commit.e2e-spec.ts`, 10, §17–§64): basic commit (order priced, redemption RESERVED,
  hold ACTIVE, ledger + available unchanged); idempotent commit; frozen-quote commit after rate change; ceiling-
  corruption rejection; expired/wrong-status order rejection; conflicting-pointer conflict; committed release (order
  restored, hold freed, no ledger, idempotent); committed release blocked once order not CREATED; commit/release race;
  security + no ledger/reward/tx/subscription/xp writes.
- Gate: **361 unit + 320 e2e PASS**; `tsc` clean; migration 17; drift clean. Existing 2.1C-2 (uncommitted) release
  e2e still green (release evolution is backward-compatible).

> **Phase 2.1J update (2026-08-21):** when a payment attempt fails/cancels, the order is reopened `PENDING → CREATED`
> (see [PAYMENT_RETRY_IMPLEMENTATION.md](PAYMENT_RETRY_IMPLEMENTATION.md)) with the committed discount **kept** (redemption
> RESERVED, reservation ACTIVE, ledger unchanged) so a retry uses the same payable. Because the order is CREATED again,
> §14's committed release works unchanged — no new release logic was added; a learner may explicitly release the discount
> after a failed attempt.

## 22. Future verified-payment finalization
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): PaymentTransaction/provider (Click/Payme), PaymentOrder
PENDING/PAID, payment verification, REDEEM/SPEND ledger, reservation CONSUMED, redemption APPLIED, Subscription/Cycle
activation, cancellation/reversal/refund, and the discount-unwind policy after payment execution begins.

> **Phase 2.1E update:** the `CREATED → PENDING` transition and the `PaymentTransaction` PENDING attempt now exist
> (execution *intent*, see [PAYMENT_EXECUTION_IMPLEMENTATION.md](PAYMENT_EXECUTION_IMPLEMENTATION.md)). This reinforces
> §14: committed release requires the order still `CREATED`, so once a payment attempt has moved the order to PENDING,
> the committed discount can no longer be unwound. IZL is still **not** spent (redemption RESERVED, reservation ACTIVE,
> ledger unchanged) — REDEEM/CONSUMED/APPLIED remain deferred to Phase 2.1F.
