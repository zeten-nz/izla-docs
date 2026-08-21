# Terminal Payment Reopen Recovery / Reconciliation — Implementation (Phase 2.1K)

> Status: **COMPLETE** (2026-08-21). An internal/admin operational mechanism that recovers the stuck state "terminal
> FAILED/CANCELLED attempt + still-PENDING order" (the post-callback reopen bridge failed and no callback replay arrived)
> by reusing the **existing** `PaymentOrderReopenService`. **No new reopen logic, no provider call, no PT/callback/IZL
> mutation, no automatic new attempt, no scheduler, no schema change.** Owner: **TD-228..232**. Migration count stays
> **20**.

Code: `backend/src/payments/` — `payment-reopen-recovery.repository.ts` (read-only backlog),
`payment-reopen-recovery.service.ts` (classification + reopen-service delegation), `admin-payments.controller.ts`
(reopen routes), `dto/reconcile-reopen.dto.ts`, `reopen-recovery.constants.ts` (permissions).

## 1. Stuck reopen problem
2.1J's post-non-success bridge is best-effort: if it fails and no provider callback replays, the state stays `PT
FAILED/CANCELLED + Order PENDING` — safe but retry-blocked (initiate needs CREATED). Operations must be able to drain
this safely.

## 2. Terminal evidence authority (TD-228 / §2)
Recovery derives from persisted server state: a PENDING order with a terminal (FAILED/CANCELLED) transaction. It never
re-decides provider success/failure (that was 2.1I).

## 3. Candidate backlog (§4 / §5)
Actionable backlog = `PaymentOrder PENDING` **AND** `some transaction FAILED/CANCELLED` **AND** `none PENDING/SUCCEEDED`
— i.e. terminal evidence exists but the order was never reopened. Read-only (`findMany`/`count`).

## 4. Single reopen service reuse (TD-228 / §41 / §60)
Every candidate goes through `PaymentOrderReopenService.reopenAfterTerminalAttempt(terminalPaymentTransactionId)`. The
recovery module owns **no** `PaymentOrder.status` writer (grep-verified) — the reopen service revalidates eligibility
under the `pay(order)` lock (stale/PENDING/SUCCEEDED/PAID protection).

## 5. Finalization-vs-reopen recovery separation (TD-232 / §34)
`SUCCEEDED + PENDING → 2.1H finalization recovery`; `FAILED/CANCELLED + PENDING → 2.1K reopen recovery`; ambiguous
PENDING → neither; REFUNDED → neither. The domains never call each other (this service never invokes the finalizer).

## 6. Deterministic ordering (§7)
Oldest-stuck-first: `PaymentOrder.createdAt ASC, id ASC`. Each candidate carries its most recent terminal attempt
(FAILED/CANCELLED by `createdAt DESC`) as the reopen input — retry safety is order-wide, not "which terminal was last"
(§6, no `latestAttemptId`).

## 7. Bounded batches (§8)
`limit` clamped: default **50**, max **200** (shared with 2.1H); DTO `@Max(200)` rejects out-of-range (400).

## 8. Per-item isolation (§10)
Serial (§9), each item independently, no outer transaction spanning the batch — each reopen owns its own transaction.

## 9. Outcomes (§10 / TD-229)
Reuses the 2.1J outcome model: **REOPENED** / **ALREADY_REOPENED** / **RETRY_ALREADY_IN_PROGRESS** /
**PAYMENT_SUCCESS_PENDING_FINALIZATION** / **ALREADY_PAID** / **NOT_REOPENABLE**; an unexpected error → **FAILED**
(`INTERNAL_REOPEN_ERROR`, no stack/SQL/secrets).

## 10. Stale PENDING protection (§13 / §29 / §52)
The actionable backlog excludes orders with a live PENDING attempt; and if one appears between the read and the reopen
call, the reopen service classifies `RETRY_ALREADY_IN_PROGRESS` — never reopening an order with an active charge path.

## 11. SUCCEEDED protection (§14 / §30 / §53)
Excluded from the backlog; a racing SUCCEEDED yields `PAYMENT_SUCCESS_PENDING_FINALIZATION` (2.1G/2.1H territory). The
finalizer is never called from here.

## 12. PAID protection (§15 / §54)
A PAID order is never PENDING, so never a candidate; a racing PAID yields `ALREADY_PAID`. Subscription/payment untouched.

## 13. Expiry (§31 / §49)
An expired stuck order still recovers to CREATED (reopen ignores `expiresAt`); the existing initiate then rejects a fresh
attempt on the expired quote, and the 2.1D committed-discount release becomes usable. No `PaymentOrder.EXPIRED` producer.

## 14. Discounted recovery (§32 / §48)
For a discounted stuck order, recovery reopens to CREATED with the redemption RESERVED, reservation ACTIVE, and the
ledger/pricing unchanged (no wallet mutation). The learner may then explicitly release via 2.1D (§33).

## 15. No provider verification (§17)
Recovery never calls `verifyCallback`; FAILED/CANCELLED is already trusted 2.1I evidence.

## 16. No provider initiate (§18 / §57)
Recovery never calls `initiate` and creates no new PaymentTransaction — it only restores retryability; the learner
chooses when to retry (no silent re-charge).

## 17. Callback bridge race (§27 / §50)
The 2.1J bridge and admin reconciliation converge through the same idempotent reopen service (`pay(order)` lock): one
`REOPENED`, the other `ALREADY_REOPENED`.

## 18. Dual reconcile (§28 / §51)
Two concurrent reconcile runs may select overlapping candidates — safe; the reopen service is the idempotency authority.
No global recovery mutex.

## 19. Security / RBAC (TD-230 / §25 / §58)
Guarded by dedicated permissions `payments.reopen.read` / `payments.reopen.reconcile`, distinct from the finalization
permissions (a finalization grant does not authorize reopen — regression-tested). Global AuthGuard + PermissionsGuard,
no ADMIN role-name bypass. Learner → 403, unauth → 401. There is no learner reopen/reconcile route.

> **RBAC note:** the permission matrix is ops-managed (no bootstrap grant); production grants these codes to an
> administrative role — a data operation, not a migration.

## 20. No audit infrastructure (§38)
No general `StaffAudit` writer exists; per the 2.1H convention this phase records no audit event and leaves audit
infrastructure unchanged.

## 21. Response shape (§23 / §59)
Backlog: `{ total, limit, items: [{ paymentTransactionId, paymentOrderId, userId, terminalStatus, provider, terminalAt,
payableAmount, currency, discounted }] }`. Reconcile: `{ scanned, reopened, alreadyReopened, retryInProgress,
successPendingFinalization, alreadyPaid, notReopenable, failed, items: [{ paymentTransactionId, paymentOrderId, outcome,
reasonCode? }] }`. No provider metadata / callback payload / auth data / SQL / policy JSON.

## 22. Tests
- **unit** (`payment-reopen-recovery.service.spec.ts`, 4): outcome mapping, safe FAILED code (no leak), limit clamping,
  read-only backlog view.
- **e2e** (`payment-reopen-recovery.e2e-spec.ts`, 9): backlog returns only actionable stuck orders (PENDING/SUCCEEDED/
  non-PENDING-order excluded; learner→403/unauth→401/read-only); FAILED + CANCELLED reconcile → REOPENED (PT unchanged,
  no provider call, no new PT); discounted recovery keeps RESERVED/ACTIVE/ledger; expired recovery → CREATED then
  initiate rejected; bridge/reconcile race → one effect; dual reconcile safe; one-failure isolation; limit validation +
  learner/unauth denied + no leak; reopen permission distinct from finalization permission.
- **Gate:** 390 unit + 414 e2e PASS; `tsc` clean; **no new migration** (count 20); drift clean. Regression: 2.1D / 2.1E
  / 2.1F / 2.1G / 2.1H / 2.1I / 2.1J green.

## 23. Scheduler deferred (§13 of the goal / §63)
No cron/interval/queue/worker is introduced; the operational primitive (bounded, admin-triggered) comes first.
Automation is a future owner decision.

## 24. Real provider readiness
The internal/test-provider payment lifecycle is now complete end-to-end: initiation retry safety, trusted success,
trusted non-success, economic finalization, finalization recovery, order reopen/retry, and reopen recovery. Deferred
(see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): background automation/alerting, provider polling, ambiguous-timeout
handling, purchase cancellation, `PaymentOrder FAILED/CANCELLED`, reservation/redemption TTL, refund/reversal/chargeback,
**real Click/Payme adapters** (next: provider-specific reconnaissance from current official merchant docs), renewal,
notifications, frontend.
