# Verified Payment Finalization Recovery / Reconciliation — Implementation (Phase 2.1H)

> Status: **COMPLETE** (2026-08-21). An internal/admin operational mechanism that retries **already-verified** payments
> stuck in the finalization backlog (`PaymentTransaction.SUCCEEDED` + `PaymentOrder.PENDING`) by reusing the **existing**
> `PaymentFinalizationService`. **No provider call, no re-charge, no new finalization logic, no schema change, no
> scheduler.** Owner: **TD-211..215**. Migration count stays **20**.

Code: `backend/src/payments/` — `payment-finalization-recovery.repository.ts` (read-only backlog),
`payment-finalization-recovery.service.ts` (classification + finalizer reuse), `admin-payments.controller.ts` (admin
routes), `dto/reconcile-finalization.dto.ts`, `finalization-recovery.constants.ts` (permissions + bounds).

## 1. Why recovery is required
2.1G's bridge is best-effort: if finalization defers/fails, the transaction stays SUCCEEDED and the order PENDING (all
partial effects roll back). That leaves a durable, recoverable backlog that operations must be able to drain safely.

## 2. Trusted backlog authority (TD-211 / §4)
Backlog = `PaymentTransaction.status = SUCCEEDED` **AND** `PaymentOrder.status = PENDING`. PV-DB-01 (one SUCCEEDED per
order) means each PENDING order resolves to exactly one trusted transaction. Never inferred from CallbackEvent /
providerTransactionId / IZL records.

## 3. SUCCEEDED/PENDING meaning
`SUCCEEDED` = provider payment already verified (Phase 2.1F) — the reconciler never re-decides this. `PENDING` = internal
economic finalization not yet applied.

## 4. Single finalizer reuse (TD-211 / §3 / §39)
Every backlog item goes through `PaymentFinalizationService.finalizeVerifiedPayment(paymentTransactionId)`. The recovery
module owns **no** business mutation — it has no `PaymentOrder.update` / `Subscription.create` / `SubscriptionCycle` /
REDEEM / CONSUMED / APPLIED writer (grep-verified). One finalizer authority only.

## 5. Backlog query
`paymentTransaction where status=SUCCEEDED AND paymentOrder.is.status=PENDING`, read-only (`findMany`/`count`).

## 6. Deterministic ordering (§5)
`confirmedAt ASC, id ASC` — oldest verified payment first. A corrupted SUCCEEDED row with `confirmedAt = NULL` is not
finalized (the finalizer rejects it → FAILED); createdAt/server-now are never substituted.

## 7. Bounded processing (§6)
`limit` is clamped: default **50**, max **200**; a non-positive/non-integer limit falls back to the default. DTO
`@Max(200)` rejects out-of-range values (400).

## 8. Item isolation (§8)
Serial processing (§7), each item finalized independently — no outer transaction spanning items. Each finalizer owns its
own transaction.

## 9. Outcome classifications (§9)
- **FINALIZED** — PENDING → PAID completed this run (`FinalizationResult.replay = false`).
- **ALREADY_FINALIZED** — the finalizer's replay path validated an already-PAID order (`replay = true`).
- **BLOCKED** — a known deterministic domain condition (`SUBSCRIPTION_PURCHASE_ACTIVE_CONFLICT`).
- **FAILED** — any other/transient/integrity error (`INTERNAL_FINALIZATION_ERROR`).

## 10. BLOCKED semantics (§10 / TD-215)
An ACTIVE-subscription conflict is BLOCKED, not FAILED: no `PaymentTransaction`/`PaymentOrder` FAILED, no redemption
release, no refund, no IZL consumption. State stays SUCCEEDED + PENDING, recoverable; repeated reconcile stays BLOCKED.

## 11. FAILED semantics (§11 / TD-215)
A transient/unexpected finalizer error leaves SUCCEEDED + PENDING intact (2.1G atomic rollback). Recorded FAILED with a
safe code; processing continues; no provider call, no automatic refund.

## 12. Admin access (§12 / §14 / TD-213)
`GET /api/admin/payments/finalization-backlog` (read-only) and `POST /api/admin/payments/finalization-reconcile`
(bounded reconcile). First `/api/admin` routes in the codebase.

## 13. Security (§16 / §18 / §61 / TD-213)
Guarded by dedicated permissions `payments.finalization.read` / `payments.finalization.reconcile` via the global
`AuthGuard` + `PermissionsGuard` (no ADMIN role-name bypass). A learner without the permission → **403**; unauthenticated
→ **401**. There is no learner mark-paid/activate/reconcile authority.

> **RBAC note:** the permission matrix is ops-managed (no bootstrap grant, matching the existing "matrix OPEN" stance).
> Production must grant these codes to an administrative role (a `RolePermission` row) for the routes to be usable; this
> is a data operation, not a migration.

## 14. Callback bridge coexistence (§21 / §24 / TD-214)
The 2.1G post-verification bridge and operational reconciliation are two recovery paths that converge through the same
idempotent finalizer. The existing finalizer's locks (`sub → pay → izl`) + DB uniques (cycle-per-order, one-REDEEM) are
the serialization authority. Callback replay retry (§65 of 2.1G) is unchanged.

## 15. Overlapping reconcile (§22 / TD-214)
Two concurrent reconcile runs may select overlapping rows — acceptable. No global reconciliation mutex; the finalizer is
the idempotency authority. Both complete with safe classifications, one economic effect per item.

## 16. Idempotency (§23)
A row that becomes PAID between the backlog SELECT and its finalizer call converges to ALREADY_FINALIZED (finalizer
replay path), never a false FAILED.

## 17. No provider call (§20 / §60)
The reconciler never touches `PAYMENT_PROVIDER_PORT` / `initiate` / `verifyCallback` / Click / Payme (grep + spy test).
It consumes persisted SUCCEEDED evidence only.

## 18. No refund/release (§44)
BLOCKED/FAILED items trigger no refund, no redemption release, no order cancellation. Operator/refund policy is future.

## 19. Recovery without re-charge (§2)
The reconciler retries *internal business finalization* only. Provider success was already decided by Phase 2.1F; it is
never re-verified or re-charged.

## 20. Operational response (§36)
Reconcile returns `{ scanned, finalized, alreadyFinalized, blocked, failed, items: [{ paymentTransactionId,
paymentOrderId, outcome, reasonCode? }] }`. Backlog returns `{ total, limit, items: [{ paymentTransactionId,
paymentOrderId, userId, confirmedAt, payableAmount, currency, discounted }] }`. No stack traces, SQL, provider metadata,
callback payload, policy JSON, or auth data (leak test).

## 21. Audit behavior (§38)
No general `StaffAudit` writer is currently in use for admin-maintenance routes (only `SecurityEvent` for auth), so — per
the §38 fallback — this phase records **no** audit event and leaves audit infrastructure unchanged. A future audit
action (`PAYMENT_FINALIZATION_RECONCILE`, safe aggregate counts) can be added when that infrastructure is adopted.

## 22. Tests
- **unit** (`payment-finalization-recovery.service.spec.ts`, 6): FINALIZED/ALREADY_FINALIZED/BLOCKED/FAILED
  classification, safe reason codes (no leak), item isolation, limit clamping (default/max/invalid), read-only backlog
  view.
- **e2e** (`payment-finalization-recovery.e2e-spec.ts`, 9): admin backlog (only SUCCEEDED+PENDING, read-only,
  learner→403/unauth→401); basic reconcile → FINALIZED + PAID; discounted → REDEEM/CONSUMED/APPLIED (one effect);
  blocked-does-not-starve-valid; repeated-blocked stays BLOCKED (no mutation); already-finalized → ALREADY_FINALIZED;
  concurrent reconcile + finalizer race → one cycle; no provider call (spy); limit validation + learner/unauth denied.
- **Gate:** 384 unit + 378 e2e PASS; `tsc` clean; **no new migration** (count 20); drift clean. Regression: full 2.1F /
  2.1G / IZL / subscription / XP / learning green.

> **Phase 2.1I update (2026-08-21):** trusted terminal non-success now exists — a `PaymentTransaction` can reach
> `FAILED`/`CANCELLED`. The backlog filter (`status = SUCCEEDED`) excludes these, so a failed/cancelled attempt never
> enters reconciliation (regression-tested in `payment-non-success.e2e`). The order stays PENDING; reopening/retry is
> Phase 2.1J, not recovery.

> **Phase 2.1K update (2026-08-21, TD-232 domain separation):** there are now two distinct payment recovery domains —
> **this one** (`SUCCEEDED + PENDING → PaymentFinalizationService`) and **reopen recovery**
> (`FAILED/CANCELLED + PENDING → PaymentOrderReopenService`, see
> [PAYMENT_REOPEN_RECOVERY_IMPLEMENTATION.md](PAYMENT_REOPEN_RECOVERY_IMPLEMENTATION.md)). They use **separate** admin
> permissions and never call each other; ambiguous PENDING and REFUNDED belong to neither.

## 23. Scheduler deferred (§25 / §67)
No cron / interval / queue / worker / scheduler is introduced. The operational primitive (bounded, admin-triggered)
comes first; automation is a future owner decision.

## 24. Future failure lifecycle
Deferred (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): background scheduler/automation, alerting/notifications, operator
refund/ACTIVE-conflict resolution, payment FAILED/CANCELLED/REFUNDED lifecycle, retry/timeout/late-success, real
Click/Payme, renewal/upgrade/downgrade, refunds/reversals/chargebacks, frontend. **Next recon:** Phase 2.1I-P (Payment
Failure / Cancellation / Retry Lifecycle).
