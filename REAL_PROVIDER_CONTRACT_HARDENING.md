# Real Provider Contract / Persistence Hardening — Implementation (Phase 2.1L-D)

> Status: **COMPLETE (PASS WITH BLOCKER)** (2026-08-21). Prerequisite hardening for real CLICK / Payme integration.
> Adds provider-specific durable protocol persistence + a non-terminal provider-binding primitive + centralized
> terminal event-ids. **NO real adapter, NO provider HTTP route, NO provider call, NO production credentials, NO refund,
> NO PaymentTransaction terminal transition, NO PaymentOrder/Subscription/IZL mutation.** Owner: **TD-233..239**.
> Migration count **20 → 21**; named CHECK **40 → 45**. Standing **CLICK PROTOCOL VERIFICATION BLOCKER** (see §2).

Code (`backend/src/payments/`): `payment-provider-binding.{repository,service}.ts` (non-terminal binding),
`payme-protocol.{repository,service}.ts` (Payme Merchant API protocol persistence), `click-protocol.{repository,service}.ts`
(CLICK Shop API provider-neutral shell), `provider/provider-event-id.ts` (centralized terminal event-ids).
Schema: `payme_merchant_transaction`, `click_shop_transaction` (migration `20260821100000_real_provider_protocol_persistence`).

---

## 0. Mandatory official-protocol verification gate

Every load-bearing constant below was re-checked this phase against the **current official documentation** only. Deep
protocol pages that were SPA shells during the 2.1L-P recon now render through automated fetch, so Payme was verified
directly. The exact CLICK Shop API signature/type/amount/error constants still could not be extracted from an official
current source and are marked BLOCKED. **No protocol constant was implemented from prior knowledge.**

### Payme — VERIFIED (developer.help.paycom.uz, current, fetched 2026-08-21)
| Constant | Verified value | Source page |
|---|---|---|
| Auth | `Authorization: Basic base64(login:password)`; password = key issued after web-cashbox setup | Протокол → Схема взаимодействия |
| amount unit | **tiyin** ("Сумма платежа (в тийинах)"); example `500000` | CreateTransaction |
| time fields | `time`/`create_time`/`perform_time`/`cancel_time` = 13-digit **Unix milliseconds** (example `1399114284039`) | Create/Perform/Cancel/GetStatement |
| params.id | Payme transaction id, string (24-hex example `5305e3bab097f420a62ced0b`) | CreateTransaction |
| merchant `transaction` | merchant's transaction identifier (our PaymentTransaction id) | Create/Perform |
| account | merchant-configured account object (example `{phone}`) | CreateTransaction |
| state | **1** created, **2** performed, **-1** cancelled-before-perform, **-2** cancelled-after-perform | Create/Perform/Cancel |
| CreateTransaction idempotency | existing tx → basic validation returned, no duplicate created | CreateTransaction |
| GetStatement | `from`/`to` = Unix ms over **Payme creation time**; rows: id, time, amount, account, create_time, perform_time, cancel_time, transaction, state, reason, receivers | GetStatement |
| CheckPerformTransaction | `amount`+`account` → `allow` (bool) + optional `detail` (fiscalization receipt items) | CheckPerformTransaction |
| Errors | -31001 wrong amount, -31003 tx not found, -31007 cannot cancel (order completed), -31008 cannot perform (state), -31050‥-31099 account, -32504 auth/privileges, -32300/-32700/-32600/-32601/-32400 RPC | Ошибки (errors) |

### Payme — NOT enumerated on the fetched pages (re-verify at 2.1L-PM)
- Exact `reason` code enumeration (only `reason:1` example shown; the mapping 4=timeout/5=refund is applied as a
  documented owner intent in TD-236 but the numeric list is re-verified when the adapter is built). This is **not
  load-bearing for 2.1L-D** — `reason` is persisted as a nullable integer with no value-enumeration CHECK.

### CLICK — BLOCKED (docs.click.uz Shop API detail pages are SPA nav-shells; official click-llc repo is card-token Merchant API)
Verified only (official click-llc/click-integration-php README + docs overview): Prepare params `[click_trans_id,
service_id, click_paydoc_id, merchant_trans_id, amount, action(0), error, error_note, sign_time, sign_string]`; Complete
adds `merchant_prepare_id`, `action(1)`; `sign_time` format `YYYY-MM-DD HH:mm:ss`; integration scheme = Shop API.
**UNVERIFIED (LOAD-BEARING for 2.1L-C, NOT for 2.1L-D schema):** exact Prepare/Complete `sign_string` MD5 concatenation,
hash confirmation, `click_trans_id`/`click_paydoc_id` native types, `merchant_prepare_id`/`merchant_confirm_id` required
type/range, amount format (decimal vs integer) + currency, Shop API error table, `merchant_trans_id` max length/charset
(UUIDv7 compatibility). These MUST be verified from docs.click.uz Shop API before Phase 2.1L-C writes any CLICK adapter.

Per the gate: schema-only provider-neutral work proceeded for CLICK **without depending on any unverified constant**
(§10/§11), and no CLICK real-signature implementation was built from memory.

---

## 1. Provider persistence model (TD-233, §2/§3/§10)
Provider-specific typed tables — **NOT** a generic JSON `ProviderProtocolState`, because CLICK and Payme have materially
different replay/state contracts. Core finance stays provider-neutral; these rows are the adapters' durable protocol
state, not economic authority (§25). `providerMetadata` JSONB remains supplemental.

## 2. Payme typed persistence — `payme_merchant_transaction`
1:1 with PaymentTransaction (unique FK, Restrict). Fields: `payme_transaction_id` (unique, params.id), `amount_tiyin`
(BigInt), `account_snapshot` (JSONB), `provider_created_time_ms` (BigInt — Payme `time`, GetStatement range key),
`create_time_ms`/`perform_time_ms?`/`cancel_time_ms?` (BigInt, merchant-assigned, §5), `state` (int 1/2/-1/-2), `reason?`
(int). CHECKs PMT-DB-03..06 (see [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) §9x). BigInt used so 13-digit ms and
tiyin never pass through JS `Number`. The `PaymeProtocolRepository` is the single writer of the native state machine
(1 create → 2 perform; 1 → -1 cancel), all under the per-order pay advisory lock.

## 3. GetStatement + replay requirement (§8/§9/§23)
Official GetStatement filters on the **Payme transaction creation time**, so `provider_created_time_ms` is preserved
verbatim (never substituted by local `createdAt`) and indexed. Repeated Create/Perform/Cancel/CheckTransaction/GetStatement
reconstruct the SAME persisted result across process restart: each timestamp is written once (replay-stable), and
`getStatement(from,to)` returns in-range rows ordered by creation time then id. Proven directly at the repository level —
no controller needed (§23).

## 4. Payme mapping + refund boundary (TD-236/238, §6/§7/§26)
`state 1` → core PT stays PENDING; `state 2` → future SUCCEEDED evidence; `state -1` → future FAILED (semantic reason,
never auto-CANCELLED); `state -2` and refund → **future Refund/Reversal domain**. Post-success CancelTransaction is
**refused** with `REFUND_DOMAIN_UNSUPPORTED` (no -2 write, no SUCCEEDED→CANCELLED, no economic reversal; Payme -31007
"order completed" semantics). No fake successful refund response is invented. **Payme is not production-ready** until a
refund/reversal model exists or a merchant agreement formally excludes performed-transaction cancellation.

## 5. CLICK typed persistence — `click_shop_transaction` (provider-neutral shell)
1:1 with PaymentTransaction. Fields: `click_trans_id?`, `click_paydoc_id?`, `merchant_prepare_id?`, `merchant_confirm_id?`
(all **String** — lossless superset; native types/format unverified), `prepare_state`/`complete_state`
(`ClickProtocolPhaseState` — Izlan-owned, not a CLICK constant), `prepared_at?`/`completed_at?`. Only invariants that
encode **no** CLICK native constant: partial unique on `click_trans_id` (external-id dedup), and a phase CHECK (ACCEPTED
Complete requires ACCEPTED Prepare — our model). The `ClickProtocolRepository` persists caller-supplied identifiers and
proves **replay stability** (same Prepare → same `merchant_prepare_id`; same Complete → same `merchant_confirm_id`, never
regenerated). It parses no CLICK request, verifies no signature, compares no native amount, maps no CLICK error, and
generates no CLICK-format id. Prepare is strictly non-terminal (§11).

## 6. Non-terminal provider binding (TD-234, §17)
`PaymentProviderBindingRepository.bind(paymentTransactionId, provider, providerTransactionId)` attaches the provider id
to an existing **PENDING** PaymentTransaction (CLICK Prepare / Payme CreateTransaction), writing ONLY
`provider_transaction_id` under the pay lock. Outcomes: BOUND / ALREADY_BOUND / CONFLICT (provider mismatch, different id,
or id owned by another attempt via PT-DB-03) / NOT_BINDABLE (missing, non-PENDING, empty id). No status transition, no
order/IZL/Subscription write, no provider call. It deliberately does **not** reuse the 2.1F terminal-evidence writer.

## 7. Timestamp normalization (TD-235, §5/§12)
All merchant-assigned times come from injected `Clock.now()` → integer Unix-ms BigInt, written once and preserved on
replay. Payme `perform_time` is the instant a future adapter will also stamp onto `PaymentTransaction.confirmedAt`
(load-bearing: `SubscriptionCycle.periodStart = confirmedAt`). CLICK `confirmedAt` = accepted-Complete `Clock.now()` (in
2.1L-C); CLICK `sign_time` is never the economic timestamp authority.

## 8. Terminal event-ids (`provider-event-id.ts`, §9/§15)
Centralized + unit-tested FUTURE `PaymentCallbackEvent.providerEventId` values for F-19 dedup:
`PAYME:{paymeTransactionId}:PERFORM`, `PAYME:{paymeTransactionId}:CANCEL`, `CLICK:{clickTransId}:COMPLETE`. CreateTransaction
/ Prepare are non-terminal and get no event-id. Payme's JSON-RPC top-level request `id` is transport correlation only —
never the financial idempotency authority; the financial identity is the provider transaction id.

## 9. Native controller boundary (TD-237, §18/§19)
Future `ClickShopController` (form-urlencoded Prepare/Complete) and `PaymeMerchantApiController` (JSON-RPC) will speak
native protocol; CLICK action numbers / sign_string / merchant_prepare_id and Payme state/reason/error integers never
leak into core finance. `verifyCallback` is not forced to model every non-terminal method — normalized evidence services
are called only on terminal financial evidence. **No provider route is exposed in 2.1L-D.**

## 10. Security / auth (§21/§22)
Payme auth is `Basic base64(login:password)` (login + web-cashbox key), config/secret-store only — never persisted or
logged; source-IP allowlist is config, re-verified at implementation. CLICK signature verification comes from the
re-verified official formula (2.1L-C) with timing-safe compare. New tables store **no** secret, auth header, or raw
callback body. (Verified: greps over the new provider files find no `fetch/axios/http/PAYMENT_PROVIDER_PORT/verifyCallback/
initiate/refund/signature/secret/password/md5/sha1/Basic`.)

## 11. Tests (§30)
- **unit** (`provider-event-id.spec.ts` 4, `payment-provider-binding.service.spec.ts` 1, `payme-protocol.service.spec.ts` 2):
  event-id format/stability/empty-id guard; binding delegation; Clock→integer-ms conversion for create/perform/cancel.
- **e2e** (`provider-protocol-hardening.e2e-spec.ts`, 18): binding (first/replay/conflict/provider-mismatch/non-PENDING/
  cross-attempt integrity/empty/unknown/concurrency); Payme create+state-1/tiyin/creation-time; create replay immutable +
  conflicts; perform once + replay-immutable; cancel -1 + replay-immutable; refund-domain refused (no -2); not-performable
  after cancel; CheckTransaction + GetStatement range ordering; CHECK guards (state/amount/time/reason); CLICK prepare
  non-terminal + replay-stable prepare_id; prepare conflicts + partial unique; complete replay-stable confirm_id +
  requires-prepare; complete-requires-prepare CHECK; boundary (no PT terminal / order / subscription / cycle / ledger /
  reservation write, no provider port call).
- **Gate:** 397 unit + 432 e2e PASS (+7 unit, +18 e2e); `tsc` clean; migration 21; named CHECK 45; drift clean (empty
  diff on izlan_dev + izlan_test). Regression: full 2.1E–2.1K / IZL / subscription / XP / learning green.

## 12. Sandbox / onboarding / fiscalization dependencies (§32)
Payme has a documented sandbox (developer.help.paycom.uz/pesochnitsa) that exercises performed-transaction cancellation
— which is why Payme rollout follows the refund architecture (TD-238/239). CLICK sandbox per docs.click.uz Shop API
testing. Fiscalization: Payme CheckPerformTransaction/CreateTransaction support a `detail` receipt object (IKPU/package/VAT)
— whether Izlan must send it is a **business/onboarding dependency**, recorded not invented, and does not block this
schema hardening.

## 13. Rollout order (TD-239, §27) + STOP boundary
1. **2.1L-D (this phase)** — contract/persistence hardening. 2. **2.1L-C — CLICK Shop API** (owner: CLICK first; must
first clear the CLICK PROTOCOL VERIFICATION BLOCKER). 3. **Refund/Reversal architecture** recon + implementation.
4. **2.1L-PM — Payme Merchant API** (re-verify reason enumeration; -2 disabled pending refund). One provider per phase,
each with its own STOP + full regression. **Do not start** any CLICK/Payme endpoint, adapter, credential, refund,
fiscalization, notification, or frontend after 2.1L-D. STOP.

## 14. Open items
See [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md): CLICK protocol constant verification (BLOCKER), CLICK `merchant_trans_id`
UUID compatibility, refund/reversal domain, Payme reason enumeration. (`confirmedAt` authority is NOT open — TD-235
already fixes it to `Clock.now()` at the first accepted successful Complete / PerformTransaction.)

## 15. CLICK Protocol Verification Evidence Appendix (Phase 2.1L-C0, 2026-08-21)

No-code verification pass toward closing the CLICK PROTOCOL VERIFICATION BLOCKER. **Result: PASS WITH CLICK PROTOCOL
VERIFICATION BLOCKER (NOT closed).** The current DOCUMENTED authority (docs.click.uz Shop API detail — Общее / Запросы /
Ошибки) remains a client-rendered SPA that returns only navigation to automated fetch, so no load-bearing constant could
be raised to category-1 "current documentation" verification. However **three official CLICK-owned repositories — two
from 2024 — agree exactly**, materially strengthening the evidence beyond the earlier "historical only" position.

### Official sources consulted (authority hierarchy §0)
| Source | Type | Date | Confirms |
|--|--|--|--|
| docs.click.uz/en/ (overview) | (1) current docs | 2026-08 | Shop API existence — suppliers must implement Prepare + Complete |
| docs.click.uz Shop API detail | (1) current docs | 2026-08 | **NOT extractable** (client-rendered SPA; nav only) |
| click-llc/click-integration-php | (3) official repo | updated 2021-10-26 | Prepare/Complete param list, Shop API scheme (README) |
| click-llc/click-integration-django (`click/utils.py`,`views.py`) | (3) official repo | ~2024-07 | signature formula, MD5, error table, HTTP contract, amount float compare |
| click-llc/woocommerce-clickuz-gateway (`include/*.php`) | (3) official repo | ~2024-08 | signature, `merchant_prepare_id`=integer(insert_id), `merchant_confirm_id`, checkout URL, amount format |

### Load-bearing constant classification
| Item | Finding (official-evidenced) | Status |
|--|--|--|
| A signature formula | Prepare `md5(click_trans_id+service_id+secret+merchant_trans_id+amount+action+sign_time)`; Complete inserts `merchant_prepare_id` after `merchant_trans_id`; amount used raw | OFFICIAL-CORROBORATED (2021+2024×2, exact) — not current-doc |
| B hash | MD5 | OFFICIAL-CORROBORATED (2024) — not current-doc |
| C merchant_trans_id | checkout `transaction_param` is a string; callbacks treat it as a numeric order id (`int(payment_id)`); no documented max length/charset | **UNVERIFIED — HARD BLOCKER** (numeric-leaning risk) |
| D merchant_prepare_id | merchant-generated **integer** (DB insert_id), echoed unchanged at Complete | OFFICIAL type=int (2024); exact range not current-doc |
| E merchant_confirm_id | merchant-generated **integer** returned at Complete | OFFICIAL type=int (2024); range not current-doc |
| F amount | checkout = **integer so'm** (`number_format(total,0,'.','')`); callback compared as float ±0.01; signature uses raw string; currency UZS | OFFICIAL-evidenced (2024) — economic parser definable |
| G inbound non-success | CLICK inbound `error` on Complete (≠0 → problem); merchant-RETURN codes ≠ inbound provider states | PARTIAL — needs current-doc for definitive-cancel taxonomy |
| H merchant response error table | 0 success / -1 sign / -2 amount / -3 action / -4 already paid / -5 user / -6 tx not found / -8 bad request / -9 cancelled | OFFICIAL-CORROBORATED (2021+2024, exact) — not current-doc |
| I replay / event-id | click_trans_id = numeric provider financial id, stable Prepare↔Complete + in signature; `CLICK:{click_trans_id}:COMPLETE` viable | OFFICIAL id-stability; retry semantics not current-doc |
| J HTTP contract | request `x-www-form-urlencoded` POST; response JSON {click_trans_id, merchant_trans_id, merchant_prepare_id\|merchant_confirm_id, error, error_note} | OFFICIAL-CORROBORATED (2024) — not current-doc |
| K checkout URL | `https://my.click.uz/services/pay` (or `//my.click.uz/pay/checkout.js`): merchant_id, merchant_user_id, service_id, transaction_param, amount, return_url; amount integer so'm | OFFICIAL-evidenced (2024) — matches 2.1L-D `initiate()` design |

### Schema fit — NO CHANGE
`ClickShopTransaction` remains sufficient: `click_trans_id`/`merchant_prepare_id`/`merchant_confirm_id` stored losslessly
as String accommodate the native integer forms; `providerTransactionId = click_trans_id` is safe; economic amount lives
in core PaymentTransaction (integer UZS). No migration.

### Closure verdict (§29) — NOT CLOSED
Hard remaining items: **C** (merchant_trans_id UUID compatibility — no positive evidence, numeric-leaning) and the
**current-documentation authority gap** (docs.click.uz Shop API detail unreadable), plus **G** (inbound non-success
taxonomy) and **D/E exact integer range**. Per §3, items A/B/F/H/I/J/K are OFFICIAL-CORROBORATED (recent 2024, cross-source)
and the owner MAY explicitly accept them as sufficient implementation authority — but that is an owner decision, not taken
here (no TD added, nothing raised to current-documentation authority).

### Recommended resolution before 2.1L-C
1. Human-read docs.click.uz Shop API (Общее / Запросы / Ошибки) or obtain CLICK's downloadable integration manual /
   merchant-cabinet doc to raise A/B/F/G/H to category-1 current-documentation authority.
2. Obtain CLICK's documented `merchant_trans_id` max length/charset. If UUIDv7 (36 chars) is accepted → freeze
   `merchant_trans_id = PaymentTransaction.id`; else adopt a durable short numeric provider-specific correlation reference
   (NOT `PaymentOrder.id`, no truncation, no hash-without-collision-analysis, no phone/user encoding).
3. Owner decision: accept the 2024-official-corroborated evidence (A/B/F/H/I/J/K) as sufficient, or require category-1
   current-doc confirmation before 2.1L-C.
