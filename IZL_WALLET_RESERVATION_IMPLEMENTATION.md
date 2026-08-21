# IZL Wallet Projection + Reservation Safety — Implementation (Phase 2.1B)

> Status: **COMPLETE** (2026-08-20). Activates a rebuildable `IZLWallet` projection and an internal, concurrency-safe
> `IZLReservation` hold primitive. `IZLLedgerEntry` remains the accounting authority; the wallet is only a cache and
> reservations reserve spendability **without** debiting IZL. **No redemption. No spend ledger entry. No reservation
> consumption. No cashout. No transfer. No payment. No catalog.** Owner: **TD-156/157/158/159/160**.
> Migration `20260820200000_activate_izl_wallet_reservation` (migration 14).

> **Reconnaissance divergence (reported):** the phase assumed an `IZLReservation` model *exists* — it did **not**.
> The actual schema had `IZLWallet.reservedAmount` (a cache field) and `IZLRedemption` (a redemption/payment entity:
> `RedemptionType`, `RedemptionStatus`, `izlRateSnapshot`, `valueUzs`, `paymentOrderId`). Reusing `IZLRedemption` as a
> pure hold would conflate reservation with redemption/payment (out of scope, §18/§56/§60), so a **new dedicated
> `IZLReservation` table** was created (per the owner contract TD-158 and §2's "schema hardening IS allowed");
> `IZLRedemption` is untouched.

Code: `backend/src/finance/wallet/` (`izl-balance.engine.ts` pure, `izl-wallet.repository.ts`, `izl-wallet.service.ts`)
and `backend/src/finance/reservation/` (`izl-reservation.policy.ts` pure, `izl-reservation.repository.ts`,
`izl-reservation.service.ts`). Controller/reconcile extended in `finance.controller.ts` / `daily-mission-izl.service.ts`.

## 1. Ledger authority
`IZLLedgerEntry` is the canonical IZL accounting truth: `balanceIzl = SUM(amount)`. The wallet and reservations never
become the source of economic truth (§4/§9/§45).

## 2. Wallet projection (TD-156)
`IZLWallet{balance, reservedAmount, lastEntryNo, projectionVersionCode}` is a rebuildable mutable cache. `balance`
mirrors the signed ledger SUM; `reservedAmount` mirrors the ACTIVE reservation SUM. Repair direction is always
ledger + reservations → wallet (§9). Single writer: `izl-wallet.repository`.

## 3. Wallet version (§3)
`projectionVersionCode = izl-wallet-projection-v1` tags which contract produced the cache; `NULL` = stale/unversioned,
upgraded on recompute (§47).

## 4. Full-history recompute (§11 / §12)
`recomputeUserIzlWallet`: under the per-user IZL advisory lock, `SUM(ledger)` + `SUM(ACTIVE reservations)` → UPSERT
wallet. **Full recompute**, never `balance += amount` — so corrections/releases/imports/stale caches all reconcile.

## 5. Source-derived GET (§37 / §38)
`GET /api/izl/me` → `{balanceIzl, reservedIzl, availableIzl}` computed from the ledger + ACTIVE reservations via the
pure engine — **not** from the wallet cache. It stays correct even when the wallet is stale or absent.

## 6. Wallet failure boundary (TD-160 / §13)
Ledger and reservations are authoritative; the wallet projection is refreshed downstream (`tryRecompute`,
non-throwing). A cache failure never rolls back a ledger posting or a reservation; reconcile repairs the wallet.

## 7. Balance semantics (TD-157)
`balanceIzl = SUM(IZLLedgerEntry.amount)` (signed).

## 8. Reserved semantics
`reservedIzl = SUM(amount) WHERE status = ACTIVE`. Only ACTIVE holds reduce spendability; RELEASED is terminal and
excluded.

## 9. Available semantics (§6)
`availableIzl = balanceIzl − reservedIzl`. **Never clamped** — a later accounting correction can make it negative
(e.g. balance 2, reserved 3 → available −1), which stays visible to accounting logic.

## 10. Negative-correction behavior (§42 / §80)
If a correction lowers the ledger below active reservations, GET reports the signed negative available; the existing
reservation stays ACTIVE (no auto-release, no ledger rewrite, no clamp), and any new reservation is rejected.

## 11. Reservation definition (TD-158)
`IZLReservation{userId, amountIzl, status, idempotencyKey, purposeCode, createdAt, releasedAt}` — a temporary hold
against available IZL for a future trusted spend/redemption workflow. It is not a RewardGrant, redemption, or payment.

## 12. No debit on reserve (§35)
Creating an ACTIVE reservation writes no ledger entry and no RewardGrant; `balanceIzl` is unchanged, only
reserved/available shift.

## 13. Internal-only reservation (§16 / §88)
There is **no** learner-facing reservation create/release endpoint. `IzlReservationService` is an exported internal
trusted primitive for the next spending/redemption phase. The public API stays balance-read + reconcile only.
(e2e asserts `POST /api/izl/reservations` and `.../me/reservations` → 404.)

## 14. Idempotency (§19 / §20)
Reservations are keyed by `unique(userId, idempotencyKey)`. Replay of the same logical request (same amount +
purpose) returns the existing reservation; a reused key with a different amount/purpose is a conflict
(`IzlReservationConflictError`). The key is server/trusted, never client-supplied.

## 15. Concurrency lock (§29 / §50)
Reserve, release, wallet recompute, and 2.1A reward posting all use the same per-user lock
`pg_advisory_xact_lock(hashtext('izl'), hashtext(userId))` — one key per user, always acquired first in each
transaction, so no races and no deadlock. Availability is authorized from the committed canonical state under the lock.

## 16. Over-reservation prevention (§7 / §22 / §23)
Inside the lock: `available = SUM(ledger) − SUM(ACTIVE reservations)`; a new hold requires
`amount > 0 && amount <= max(available, 0)` (pure `canReserve`). Never authorized from the wallet cache (which may be
stale). Two concurrent holds against balance 1 → exactly one succeeds (e2e §75).

## 17. Immutable reservation provenance (§34)
After creation, `amount`/`user`/`purpose`/`idempotencyKey` are immutable; the only permitted update is
`status ACTIVE → RELEASED` (+ `releasedAt`).

## 18. Release semantics (§26 / §36)
`ACTIVE → RELEASED` removes the hold from reserved; it creates no ledger entry and refunds nothing (the reservation
never debited). Idempotent — releasing an already-RELEASED reservation returns the terminal row with no side effect.

## 19. No consume (§29 / TD-159)
There is no runtime `ACTIVE → CONSUMED` path — the `IzlReservationStatus` enum has only `ACTIVE`/`RELEASED`. A
consumed hold must be tied atomically to a SPEND ledger debit (future phase), otherwise the hold would vanish while
the balance stayed unchanged, falsely restoring available funds.

## 20. Why consume requires a ledger debit (§29)
Consumption is a real spend; it must post a negative ledger entry atomically with the reservation transition so the
canonical balance and the reserved amount both move. That belongs to the redemption/spend phase.

## 21. No expiry scheduler (§31)
No cron/scheduler/TTL. `EXPIRED` is not modeled in v1; expiry policy is deferred.

## 22. Reconcile (§44 / §45 / §46)
`POST /api/izl/me/reconcile` repairs missing mission earnings (2.1A) **and** rebuilds the wallet projection, returning
`{balanceIzl, reservedIzl, availableIzl, grantsCreated}`. It never changes reservation statuses / consumes / releases.
A corrupt or unversioned wallet is recomputed to the canonical v1 projection (e2e §82).

## 23. Security (§77 / §87 / §89 / §90)
Own-user only; 401 unauthenticated; another user's ledger/holds never appear. No public route accepts amount /
idempotencyKey / purpose / balance / reserved for mutation. GET exposes only the safe balance triple — no ledger ids,
RewardGrant ids, idempotency keys, purpose internals, or policy config.

## 24. Side-effect boundaries (§91)
The reservation primitive writes **only** `izl_reservation` (create/update — no DELETE, §33); the wallet projector
writes **only** `izl_wallet` (upsert). Reserve/release write no ledger and no RewardGrant. 2.1A earning still posts
RewardGrant + IZLLedgerEntry atomically (the wallet recompute is a separate downstream step, never inside that
transaction). No XpGrant/XpBalance/Subscription/SubscriptionCycle/Payment/IZLRedemption/Notification/AI writes.

## 25. Tests
- **Unit**: `izl-balance.engine.spec.ts` (4 — available, zero, negative reserved>balance, negative ledger);
  `izl-reservation.policy.spec.ts` (6 — fits/exact/over/zero-available/negative-available/non-positive+non-integer).
- **e2e** (`izl-wallet-reservation.e2e-spec.ts`, 14, §68–88): zero-state + reject; reserve (no ledger) + projection;
  full/over reserve; idempotent reserve; idempotency conflict; concurrent (one wins); multiple holds; release + replay
  (no ledger); negative correction (signed available, hold stays, new reject); GET canonical vs stale cache
  (read-only); reconcile repairs corrupt wallet; wallet absent → GET correct + reconcile creates; GET IDOR/401;
  no public reservation endpoint (404).
- Gate: **355 unit + 288 e2e PASS**; `tsc` clean; migration 14; drift clean.

## 26. Future redemption consumer
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): RedemptionRequest model, redemption catalog, reservation source
entity, reservation expiration/TTL, reservation consumption + SPEND ledger debit, partial redemption, fulfilment
states, wallet performance / incremental projector + global rebuild job, ledger corrections/reversals, negative-
available operational handling, fraud/risk holds, staff adjustments, transfer, cashout, IZL fiat value, tax/accounting.
**Next: owner review — Phase 2.1C (Redemption Intent Foundation).**

> **Phase 2.1G-D update (2026-08-21):** `IzlReservationStatus` now has a third value **CONSUMED** (TD-201) = an ACTIVE
> hold fulfilled by a REDEEM ledger debit, distinct from RELEASED (freed, no spend). The reserved SUM query is
> **unchanged** (`status = ACTIVE`), so both RELEASED and CONSUMED are excluded from availability. There is **no runtime
> CONSUMED producer** yet — the Phase 2.1G finalizer will be the first. A one-REDEEM-per-redemption partial unique
> (`uq_izl_ledger_redeem_per_redemption`) was also added. See
> [PAYMENT_FINALIZATION_CONTRACT.md](PAYMENT_FINALIZATION_CONTRACT.md).
