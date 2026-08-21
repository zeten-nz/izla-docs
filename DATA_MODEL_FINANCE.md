# Izlan — Financial Domain Data Model Architecture (Rewards, Subscription, Payment)

> Status: Phase 1.2C-1 — **COMPLETE**; Phase 1.2D refinementlari kiritilgan (TD-84 mission evidence, TD-85 PlanPrice, TD-86 entitlement mode, TD-87 subscription episode, TD-89 numeric width — ACCEPTED). Prisma-ready **architecture** — schema EMAS, migration EMAS, integration EMAS.
> **TD-45..TD-60 hammasi ACCEPTED** ([TECH_DECISIONS.md](TECH_DECISIONS.md)); TD-47/50/52/57 owner clarificationlari shu hujjatga kiritilgan.
> Asos: [REWARDS.md](REWARDS.md), [SUBSCRIPTIONS.md](SUBSCRIPTIONS.md), [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md), [DATA_MODEL_LEARNING.md](DATA_MODEL_LEARNING.md) (1.2A/1.2B COMPLETE).
> Bu domainda **correctness, auditability, idempotency va transactional integrity feature tezligidan ustun.**

## 1. Goals

1. XP (gamification) va IZL (real qiymatli currency) uchun qat'iy ajratilgan, auditable model.
2. IZL — append-only ledger: hech qanday balance overwrite, hech qanday destructive delete.
3. 20% ceiling — subscription cycle'ga snapshot qilingan, keyingi narx o'zgarishlaridan mustaqil.
4. Click/Payme-compatible, lekin provider'ga hard-code qilinmagan payment domain; callback idempotent; frontend hech qachon subscription'ni activate qilmaydi.
5. Har reward grant evidence'ga bog'langan va bir evidence uchun bir marta (DB darajasida himoya).
6. Pricing/policy/rate o'zgarishlari tarixiy iqtisodni o'zgartirmaydi — snapshot/versioning.

## 2. Scope / non-scope

**Scope (NOW):** XP model, IZL wallet/ledger, RewardGrant + policy versioning, 20% ceiling, SubscriptionPlan/PlanPrice/PlanEntitlement, Subscription/SubscriptionCycle/SubscriptionChange, UsageCounter, PaymentOrder/PaymentTransaction/PaymentCallbackEvent, IZLRedemption, IzlRateVersion.

**Non-scope (bu bosqichda modellashtirilmaydi):** Community/Reputation/Moderation/Announcements/Notifications (1.2C-2), o'yin valyutasi vendor integratsiyalari, cash-out implementation, kompaniya buxgalteriyasi/general ledger, soliq/invoice, AI provider implementation. Refund/recurring — faqat extension pointlar (§25, §26).

## 3. Domain principles

1. **Ledger — business truth.** IZL qiymatining yagona haqiqati — append-only ledger; wallet balance — derived cache.
2. **Snapshot over reference.** Narx, ceiling, rate, entitlement — cycle/tranzaksiya paytidagi qiymat snapshot qilinadi; joriy konfiguratsiya tarixni o'zgartirmaydi.
3. **Hech qanday floating-point pul.** Barcha summalar integer (§5).
4. **Evidence-bound rewards.** Har grant learning evidence'ga FK bilan bog'lanadi; engine "random query qilib pul bermaydi" — boundary: RewardGrant (§8).
5. **Trusted-path activation.** Faqat server-verified provider confirmation subscription/cycle yaratadi (§24).
6. **Idempotency by constraint.** Retry/double-click DB unique constraintlar bilan yutiladi, hech qachon ikki marta pul bermaydi/olmaydi.

## 4. XP vs IZL

**Bir ledger/table'da aralashtirilMAYdi (TD-45).** Sabab: har xil qat'iylik darajasi — IZL financial-style (reversal, ceiling, audit, locking); XP esa yengil gamification. Bitta jadvalga qo'shish IZL'ning og'ir invariantlarini XP'ga majburlaydi yoki XP'ning yengilligini IZL'ga yuqtiradi.

| | XP | IZL |
|---|---|---|
| Model | XpBalance (aggregate) + XpGrant (yengil history) | IZLWallet (cache) + IZLLedgerEntry (append-only truth) |
| Qat'iylik | Oddiy transactional yozuv | Financial: locking, reversal, ceiling, audit |
| Korreksiya | Yangi grant (musbat/manfiy) | Faqat ADJUSTMENT/REVERSAL entry |

## 5. Monetary representations

| Variant | Baho |
|---|---|
| A — IZL integer unit | Sodda; earning qiymatlari kichik butun sonlar |
| B — fixed precision numeric | Kasr IZL product'da ko'zda tutilmagan — keraksiz |
| **C — minor-unit integer** | Kasr kerak bo'lsa kelajak yo'li, hozir ortiqcha |

**PROPOSED (TD-46):**

- **IZL:** integer units (bigint) — kasr yo'q. Kelajakda kasr kerak bo'lsa minor-unit migratsiyasi mumkin (hozir qilinmaydi).
- **UZS:** integer so'm — tiyin real muomalada yo'q; **hech qayerda floating-point emas**.
- **DB width (TD-89, Phase 1.2D):** "smallest safe integer width" — bounded per-user/per-order qiymatlar PG INTEGER (CHECK bounds bilan); BIGINT faqat 2^31'ga real yaqinlasha oladigan fieldlar uchun (odat bo'yicha emas). Bu hujjatdagi "bigint" so'zlari conceptual "butun son" ma'nosida — exact width TD-89 bo'yicha.
- Provider amount formati (masalan Payme tiyin ishlatishi) — adapter mapping, business jadvallar UZS integer saqlaydi (§20).
- `1 IZL = X UZS` konversiyasi — data (IzlRateVersion, §28), schema emas; rate o'zgarsa ham arxitektura ishlaydi.

## 6. IZL Wallet

| Variant | Baho |
|---|---|
| A — ledger-only SUM | Har balance ko'rsatishda SUM — sekin; concurrency uchun lock anchor yo'q |
| **B — wallet cached balance + ledger source of truth** | Balance o'qish O(1); wallet qatori tranzaksiyada lock anchor (§30); invariant: balance == SUM(ledger) ✅ |

**PROPOSED (TD-47): B.**

### IZLWallet

| Field | Izoh |
|---|---|
| user_id | PK/FK, 1:1 |
| balance | bigint IZL — **cache**; faqat ledger entry bilan bir tranzaksiyada yangilanadi |
| reserved_amount | bigint IZL — pending checkout reservationlari (TD-57); **spendable = balance − reserved_amount** |
| last_entry_no | bigint — wallet-local ledger ordering hisoblagichi (nom — implementation detail) |
| updated_at | |

Qoidalar: balance'ga to'g'ridan-to'g'ri UPDATE yo'q — har o'zgarish ledger entry + shu tranzaksiyada balance yangilanishi; davriy reconciliation (SUM tekshiruvi) mumkin; divergence = bug, ledger g'olib.

**Wallet-local ordering (owner clarification, TD-47):** har ledger entry wallet ichida monotonic `entry_no` oladi. Tranzaksiya oqimi: (1) wallet row lock → (2) current version/last_entry_no o'qiladi → (3) next entry number → (4) ledger entry insert → (5) balance (kerak bo'lsa reserved) update → (6) commit. UUIDv7 vaqt bo'yicha yaxshi tartib beradi, lekin financial ordering'ning **yagona authority'si UUID timestamp emas** — entry_no hal qiluvchi.

**Reservation (owner clarification):** reservation — ledgerdagi actual financial movement EMAS; faqat reserved_amount o'zgaradi. Actual REDEEM faqat payment success'da yoziladi (§27).

## 7. IZL Ledger

| Variant | Baho |
|---|---|
| A — simple signed ledger | Yetarli auditability, sodda |
| B — double-entry internal ledger | Ikki tomonlama accounting — **closed-loop, bir valyutali, bitta kontragentli (platforma) reward tizimi uchun overengineering**: accounts/postings apparati foyda bermaydi; kompaniya moliyaviy buxgalteriyasi scope'dan tashqarida (§2) |
| C — hybrid (wallet + immutable entries) | Aslida A + TD-47 wallet — tanlangan yo'l |

**PROPOSED (TD-47): A/C — single-sided, signed, append-only ledger + wallet cache.**

### IZLLedgerEntry

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| user_id | FK (wallet egasi) |
| entry_no | bigint — **wallet-local monotonic tartib**; unique(user_id, entry_no); ledger ordering authority (TD-47 clarification) |
| entry_type | enum: EARN / REDEEM / ADJUSTMENT / REVERSAL (+ EXPIRY — agar keyin qabul qilinsa, enum kengayadi) |
| amount | **signed bigint IZL** (EARN +, REDEEM −, ADJUSTMENT ±, REVERSAL — asl entryga teskari) |
| balance_after | bigint — entry'dan keyingi balans snapshot (audit o'qishini osonlashtiradi) |
| reward_grant_id | nullable FK — EARN uchun |
| redemption_id | nullable FK — REDEEM/REVERSAL uchun |
| subscription_cycle_id | nullable FK — EARN qaysi cycle hisobiga |
| reversal_of_entry_id | nullable FK — REVERSAL qaysi entryni bekor qiladi |
| reason, actor_user_id | ADJUSTMENT uchun majburiy (D-35: amount+reason+actor+timestamp) |
| created_at | |

### Immutability (§9 talabi)

- Ledger row **hech qachon UPDATE qilinmaydi**, **hech qachon DELETE qilinmaydi**.
- Xato grant → **REVERSAL entry** (asl entryga reference, teskari amount); asl entry joyida qoladi.
- Admin korreksiya → **ADJUSTMENT entry** (reason + actor bilan) — hech qachon balance overwrite emas.
- REVERSAL'ning REVERSAL'i o'rniga yangi to'g'ri entry — zanjir soddaligi uchun (bitta darajali reversal).

## 8. RewardGrant

**Boundary (TD-48):** learning domain to'g'ridan-to'g'ri walletga yozmaydi:

```
Learning evidence (attempt/session/plan item)
      ↓
Reward Engine (policy'ni qo'llaydi — formulalar shu yerda)
      ↓
RewardGrant (qaror yozuvi: nima uchun, qaysi evidence, qancha)
      ↓  (bitta DB tranzaksiyada)
IZLLedgerEntry (EARN) + IZLWallet.balance + SubscriptionCycle.earned_izl
```

### RewardGrant

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| user_id | FK |
| category | enum: LEARNING_SESSION / LESSON_ATTENTION / MASTERY / DAILY_MISSION (accepted 4 kategoriya) |
| reward_policy_version_id | FK — qaysi policy asosida (§10) |
| subscription_cycle_id | FK — ceiling hisobiga |
| amount | bigint IZL |
| dedup_key | string; **unique(user_id, dedup_key)** — idempotency (§9) |
| activity_attempt_id / learning_session_id / daily_mission_completion_id | nullable FK'lar — evidence (§9; mission uchun TD-84: completion, plan item emas) |
| status | GRANTED / REVERSED |
| ledger_entry_id | FK (1:1) — yaratilgan EARN entry |
| created_at | |

## 9. Reward evidence / idempotency

**Evidence reference (TD-48):** polymorphic `source_type + source_id` **rad etiladi** (FK integrity yo'qoladi). Evidence turlari oz va accepted (attempt, session, plan item) — **dedicated nullable FK ustunlar**: kategoriya bo'yicha roppa-rosa bittasi to'ldirilgan (invariant F-4). Yangi kategoriya = migration — bu normal, chunki yangi kategoriya baribir product qarori.

| Category | Evidence FK |
|---|---|
| MASTERY | activity_attempt_id (MASTERY_TEST attempt) |
| LESSON_ATTENTION | activity_attempt_id (mini practice attempt) |
| LEARNING_SESSION | learning_session_id |
| DAILY_MISSION | daily_mission_completion_id (TD-84 — append-only completion; u o'z Evidence child qatorlari bilan post/attempt/session'ni isbotlaydi; completion → plan item → generation snapshot zanjiri TD-42 auditini beradi) |

**Idempotency (TD-48):** deterministic **dedup_key** + `unique(user_id, dedup_key)` DB constraint. Misollar (shakl — konventsiya, exact format implementation'da): `mastery:{activity_id}` (bir activity uchun bir marta — accepted qoida), `attention:{activity_id}`, `session:{local_date}`, `mission:{daily_plan_item_id}`. Retry/double-processing → unique violation → no-op. Bu "same evidence uchun ikki marta IZL yo'q" anti-farming qoidasining DB-darajali himoyasi.

## 10. Reward policies

**PROPOSED (TD-49): RewardPolicyVersion** — versioned config:

| Field | Izoh |
|---|---|
| id, version | |
| status | DRAFT / ACTIVE / ARCHIVED — bir vaqtda bitta ACTIVE |
| config | JSONB — kategoriya taqsimoti, mastery threshold, amal qiymatlari... **exact qiymatlar OPEN — config data, schema emas** |
| effective_from, created_by, created_at | |

- Har RewardGrant o'zi qo'llangan policy versiyasiga FK — "nega shuncha berildi" har doim izohlanadi.
- **SubscriptionCycle cycle boshida ACTIVE policy'ni snapshot qiladi** (reward_policy_version_id) — cycle ichida iqtisod barqaror; Admin konfiguratsiyani o'zgartirsa **keyingi** cycle'larga ta'sir qiladi, eski cycle o'zgarmaydi (§15).
- Mastery threshold misoli: 90% → 85% o'zgarsa — yangi policy version; eski grantlar eski versionga bog'langan, tarix izohlanadi. Full config-diff tarixi ortiqcha — version qatori yetarli.

## 11. 20% ceiling

**ACCEPTED (TD-50, owner clarification bilan):** ceiling **SubscriptionCycle'da snapshot**; basis tanlovi policy'ga bog'lanmagan:

- Cycle snapshotlari: `gross_price_uzs`, `discount_uzs`, `paid_amount_uzs`, **`reward_basis_uzs`** (20% aynan qaysi qiymatga qo'llangani — tarixiy audit), `reward_ceiling_uzs` (= reward_basis'ning 20%i), `izl_rate_snapshot`, `reward_ceiling_izl` (derived, granting shu birlikda sanaydi).
- **Gross yoki net (actual paid) — PRODUCT QUESTION OPEN.** Product keyin tanlaydi; qaror `reward_basis_uzs`ni qaysi qiymatdan to'ldirishni belgilaydi — schema o'zgarmaydi.
- Oldingi cycle keyingi policy change bilan **qayta hisoblanmaydi** (F-7).
- `earned_izl` — cached counter (grant tranzaksiyasida yangilanadi); authoritative = SUM(cycle grantlari); invariant: earned_izl ≤ reward_ceiling_izl (F-6).

## 12. XP model

**PROPOSED (TD-45):**

- **XpBalance:** user_id (1:1), total_xp (bigint), current_level (cached int — formula engine'da, OPEN), updated_at.
- **XpGrant:** id, user_id, amount (int, ± mumkin — korreksiya yangi grant bilan), reason_code (string registry), source_refs (JSONB, nullable — qat'iy FK talab qilinmaydi, pul emas), created_at. Append-only, lekin IZL darajasidagi apparat (reversal zanjiri, locking, ceiling) YO'Q.
- Har micro-earning alohida qator bo'lishi shart emas — engine yig'ma grant yozishi mumkin (masalan session yakunida bitta qator). Achievements/badges — future, XpGrant reason_code bunga tayyor.

## 13. Subscription Plan

**PROPOSED (TD-51): plan identity va pricing ajratiladi** — narx o'zgarishi plan identity'ni ham, tarixiy subscriptionlarni ham buzmaydi.

### SubscriptionPlan

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| code | unique — START / PRO / MAX (conceptual, seedable) |
| name, description? | display |
| status | ACTIVE / ARCHIVED |
| sort_order | |
| timestamps | |

## 14. Pricing / versioning

### PlanPrice (TD-51)

| Field | Izoh |
|---|---|
| id, plan_id | FK |
| currency | default 'UZS' (ustun bor, lekin multi-currency apparati YO'Q — enterprise'lashtirilmaydi) |
| amount | bigint UZS |
| effective_from | timestamp; **unique(plan, currency, effective_from)** |
| created_by, created_at | Admin pricing audit (D-35/TD-81 bilan) |

**TD-85 (ACCEPTED):** `effective_to` YO'Q — row'lar 100% immutable, UPDATE umuman yo'q. Joriy narx = effective_from ≤ now bo'lgan eng yangi row; kelajak narx = kelajak effective_from'li row. Har cycle o'zi ishlatgan PlanPrice'ga FK + amount snapshot (double protection) — current lookup tarixni o'zgartirmaydi. Scheduled-price correction admin policy'si — implementation'da (ishlatilgan tarixiy rowlar hech qachon mutable emas). Sotuvni to'xtatish — plan status orqali. Pricing o'zgarishi StaffAudit yozuvini hosil qiladi.

## 15. Entitlements

| Variant | Baho |
|---|---|
| Plan jadvalida ustunlar (ai_limit, speaking_limit...) | Har yangi feature = migration + jadval kengayadi; feature matrix OPEN bo'lgani uchun ustunlarni hozir taxmin qilish noto'g'ri |
| **PlanEntitlement (feature_code + limit)** | Feature'lar o'sadi; kod registry validatsiyasi (TD-26 permission_code pattern'i bilan bir xil falsafa); enterprise feature-flag platforma EMAS — oddiy jadval ✅ |

**ACCEPTED (TD-52 + TD-86 refinement): PlanEntitlement** — plan_id, feature_code (string, application registry: `ai.tutor_interactions`, `ai.speaking_evaluations`, `subjects.max_active`, `izl.max_earnable`... — misollar, katalog OPEN), **mode enum: DISABLED / ENABLED / LIMITED / UNLIMITED** + limit_value nullable, unique(plan_id, feature_code).

**TD-86 CHECK semantika:** LIMITED → limit_value NOT NULL va ≥ 0; DISABLED/ENABLED/UNLIMITED → limit_value IS NULL. Registry'da value_kind (BOOLEAN/COUNT_LIMIT); registry semantikasi immutable (feature_code ma'nosi qayta ishlatilmaydi); mode↔value_kind mosligi — application validation. SubscriptionCycleEntitlement — **aynan shu kontrakt**.

**Entitlement snapshot (§20-savol):** Admin PRO matrixni o'zgartirsa joriy obunachilarga darhol ta'sir qiladimi — **PRODUCT POLICY OPEN**. Arxitektura ikkalasiga tayyor:

- **Cycle snapshot — normalized, opaque JSONB emas (owner clarification, TD-52):** SubscriptionCycle yaratilishida **SubscriptionCycleEntitlement** qatorlari yoziladi: `(cycle_id, feature_code, limit_value/mode)`, unique(cycle, feature_code), **immutable**. Sabab: usage checks, audit, feature lookup va historical entitlement query'lari oson. Plan entitlement keyin o'zgarsa oldingi cycle qatorlari rewrite qilinmaydi.
- **Entitlement Resolver** (application service) qaysi manbadan o'qishni policy bo'yicha hal qiladi: cycle snapshot (default proposal — cycle ichida barqaror, o'zgarish keyingi cycle'dan) yoki live plan entitlements (agar product darhol qo'llashni tanlasa) — **policy OPEN**.

## 16. Usage metering

**PROPOSED (TD-53):**

### UsageCounter

| Field | Izoh |
|---|---|
| subscription_cycle_id + feature_code | unique pair |
| used | bigint counter |
| updated_at | |

- Quota tekshiruvi: counter vs cycle entitlement_snapshot limiti.
- **Consumption domain entity'ga bog'lanadi:** AI usage o'sishi tegishli domain yozuvi (masalan AiEvaluation) yaratilishi bilan **bir tranzaksiyada** — retry xavfsiz, chunki domain entity o'zi idempotent (TD-44). Alohida UsageRecord jadvali MVP'da YO'Q — usage izi mavjud domain evidence (AiEvaluation va h.k.) orqali; provider cost tracking — alohida kelajak tizimi, user entitlement bilan **aralashtirilmaydi**.
- **Reserve → execute → consume/release apparati MVP'da YO'Q** — check-then-execute + atomic consume; parallel so'rovlarda kichik overshoot nazariy mumkin (limit qattiq moliyaviy emas) — qabul qilinadigan MVP trade-off; abuse ko'rinsa reservation extension qo'shiladi.

## 17. Subscription

### Subscription (TD-54)

| Field | Izoh |
|---|---|
| id, user_id | FK |
| plan_id | FK — joriy plan |
| status | ACTIVE / EXPIRED / CANCELLED (§18-status) |
| started_at | birinchi activation |
| current_cycle_id | nullable FK — qulaylik pointer'i |
| cancelled_at | nullable |
| timestamps | |

**Identity semantikasi (TD-87, ACCEPTED):** Subscription — har payment uchun record EMAS, **userning bitta davomli subscription episode/container'i**. ACTIVE — joriy paid cycle bor; EXPIRED — cycle tugagan, lekin episode **terminal emas**: renewal → **shu Subscription ichida** yangi cycle → ACTIVE; CANCELLED — terminal: keyin qayta obuna → **yangi Subscription record**; tarixiy CANCELLED'lar saqlanadi.

**Uniqueness:** bitta user'da bir vaqtda **ko'pi bilan bitta non-terminal (ACTIVE/EXPIRED) subscription** (partial unique). PENDING holat yo'q — subscription faqat verified payment bilan yaratiladi (§24); to'lovgacha bosqich PaymentOrder'da yashaydi.

### SubscriptionChange (append-only, §19-upgrade)

subscription_id, from_plan_id, to_plan_id, reason/type, changed_at, actor — upgrade/downgrade **policy OPEN** (immediate/prorated/next-cycle final emas), lekin har plan o'zgarishi shu logda audit qilinadi. Prorate hisob-kitoblari hozir modellashtirilmaydi.

## 18. SubscriptionCycle — markaziy entity

Oylik ceiling ham, quota ham aynan cycle'ga bog'liq — alohida entity **shart** (TD-54):

| Field | Izoh |
|---|---|
| id, subscription_id | FK |
| sequence_no | int; **unique(subscription_id, sequence_no)** |
| period_start, period_end | |
| plan_id | snapshot — cycle qaysi planda edi |
| plan_price_id | FK → PlanPrice (ishlatilgan narx versiyasi) |
| gross_price_uzs | bigint — plan narxi snapshot |
| discount_uzs | bigint — IZL discount qiymati |
| paid_amount_uzs | bigint — haqiqiy to'langan summa |
| reward_basis_uzs | bigint — 20% qo'llangan qiymat (gross vs net — policy OPEN, §11) |
| reward_ceiling_uzs | bigint — reward_basis'ning 20%i |
| reward_policy_version_id | FK — cycle iqtisodi (§10) |
| izl_rate_snapshot | cycle boshidagi rate (UZS per IZL) |
| reward_ceiling_izl | bigint — derived snapshot |
| earned_izl | bigint cached (≤ ceiling — F-6) |
| (entitlements) | 1:N SubscriptionCycleEntitlement — normalized immutable snapshot (§15) |
| payment_order_id | FK — cycle'ni yaratgan to'lov |
| status | ACTIVE / COMPLETED |
| created_at | |

```
Subscription
  └── SubscriptionCycle (sequence)
        ├── economics snapshot: price, ceiling, rate, policy, entitlements
        ├── earned_izl (cached; truth = RewardGrant SUM)
        └── UsageCounter'lar (feature_code bo'yicha)
```

**Subscription status minimal modeli (§25-savol):** ACTIVE (joriy cycle amalda) / EXPIRED (cycle tugagan, yangilanmagan — manual renewal bilan yangi to'lov EXPIRED→ACTIVE qiladi) / CANCELLED (user/staff bekor qilgan). PAST_DUE, trialing kabi Stripe-uslub holatlar YO'Q — auto-renew final bo'lsa qayta ko'riladi.

**Expiry behavior (§26):** obuna tugaganda nima ochiq qoladi (block/read-only/grace) — **PRODUCT POLICY OPEN**. Ajratish: Subscription/Cycle — faktlar; **Entitlement Resolver** — policy. Model policy'ga bog'lanmagan: resolver EXPIRED holat uchun qoidani keyin oladi.

## 19. Upgrade / downgrade extension

Policy OPEN (immediate/prorated/next-cycle). Model tayyorligi: SubscriptionChange log (§17) + cycle'lar plan/price snapshot saqlagani uchun har qanday siyosatda tarix buzilmaydi. Prorate/credit hisoblari kerak bo'lsa — PaymentOrder purpose kengayadi (extension), hozir modellashtirilmaydi.

## 20. Payment provider abstraction

```
Payment Domain (PaymentOrder, PaymentTransaction, PaymentCallbackEvent)
        ↓
Provider Port (conceptual: create/init, verify, callback semantics, canonical statuses)
        ↓
ClickAdapter / PaymeAdapter
```

- Provider-specific maydonlar business jadvallarga **tarqalmaydi**: `provider` enum + `provider_transaction_id` + `provider_metadata` JSONB — faqat PaymentTransaction/CallbackEvent ichida.
- Amount formati mapping (masalan tiyin) — adapter ishi (§5).
- Provider qo'shish/almashtirish subscription/reward modellariga tegmaydi (TD-55).

> **Phase 2.1L-D (TD-233, 2026-08-21) — provider protocol persistence:** real provider protocol'lar durable replay/state
> talab qilgani uchun **provider-specific typed** jadvallar qo'shildi (generic JSON EMAS §2): `PaymeMerchantTransaction`
> (Payme Merchant API — amount_tiyin, provider/create/perform/cancel time BigInt Unix ms, state 1/2/-1/-2, official
> developer.help.paycom.uz'dan verified) va `ClickShopTransaction` (CLICK Shop API — provider-neutral shell, **CLICK
> PROTOCOL VERIFICATION BLOCKER** §0, native constant yo'q). Har biri PaymentTransaction bilan 1:1 (unique FK, Restrict).
> Bular **iqtisodiy authority EMAS** (§25) — pul haqiqati core PT/order/callback/subscription/IZL'da qoladi; provider
> protokolini to'g'ri gapirish + idempotent native javob (Create/Perform/Cancel/CheckTransaction/GetStatement, Prepare/
> Complete) reconstruct qilish uchungina. Non-terminal provider-binding (`provider_transaction_id` attach, status
> transition yo'q) TD-234. Real adapter/route/refund YO'Q. Batafsil: [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md).

## 21. PaymentOrder

Business intent — provider tranzaksiyasidan **alohida** (bitta order bir necha provider urinishiga ega bo'lishi mumkin):

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| user_id | FK |
| purpose | enum: SUBSCRIPTION_PURCHASE / SUBSCRIPTION_RENEWAL (kelajakda kengayadi) |
| subscription_id | nullable FK (renewal'da) |
| plan_id, plan_price_id | FK — nima sotib olinyapti (narx versiyasi bilan) |
| currency | 'UZS' |
| gross_amount | bigint — plan narxi snapshot |
| izl_discount_amount | bigint — IZL bilan qoplangan qiymat (UZS) |
| izl_redemption_id | nullable FK → IZLRedemption |
| payable_amount | bigint — gross − discount |
| provider | enum: CLICK / PAYME (tanlangan) |
| status | CREATED / PENDING / PAID / FAILED / CANCELLED / EXPIRED |
| client_request_id | nullable — checkout retry dedup |
| created_at, expires_at | |

Gross/discount/payable **snapshot** — keyin narx yoki rate o'zgarsa tarixiy order o'zgarmaydi (§27–28).

## 22. PaymentTransaction

| Field | Izoh |
|---|---|
| id, payment_order_id | FK |
| provider | enum |
| provider_transaction_id | **unique(provider, provider_transaction_id)** |
| amount | bigint UZS |
| status | canonical: PENDING / SUCCEEDED / FAILED / CANCELLED (+ REFUNDED — reserved, §26) |
| provider_metadata | JSONB — **sanitized minimal** (status kodlari, vaqtlar, provider referencelar) |
| created_at, confirmed_at | |

**Raw payload siyosati:** provider'ning to'liq raw payloadini business jadvalda forever saqlash tavsiya etilmaydi (PII/hajm/keraksizlik) — sanitized metadata yetarli; to'liq raw so'rov/javoblar **cheklangan retention'li operational audit log**da (implementation detail; retention OPEN). Provider state → canonical state mapping — adapter'da.

## 23. Payment idempotency

### PaymentCallbackEvent

| Field | Izoh |
|---|---|
| id, provider | |
| provider_event_id | **unique(provider, provider_event_id)** — dedup kaliti (provider request/event identifikatori; yo'q bo'lsa adapter deterministic hosil qiladi: tx id + action + state) |
| payment_transaction_id | nullable FK |
| received_at, processed_at | |
| result | qisqa natija (ACCEPTED/DUPLICATE/REJECTED sababi) |

**Atomic processing (TD-56):** bitta DB tranzaksiyada:

```
verify (signature/amount — adapter)
→ CallbackEvent insert (unique — duplicate shu yerda to'xtaydi, saqlangan natija qaytariladi)
→ PaymentTransaction state transition (state machine bo'yicha, no-op agar allaqachon terminal)
→ PaymentOrder → PAID
→ Subscription (yaratish/EXPIRED→ACTIVE) + yangi SubscriptionCycle (snapshotlar bilan)
→ IZL discount bo'lsa: redemption APPLIED (§27)
```

Takror callback → unique violation → mavjud natija qaytadi; **bitta verified payment roppa-rosa bitta cycle yaratadi** (F-10).

## 24. Payment lifecycle

- **PaymentOrder:** CREATED → PENDING (provider'ga yo'naltirildi) → PAID / FAILED / CANCELLED / EXPIRED.
- **PaymentTransaction:** PENDING → SUCCEEDED / FAILED / CANCELLED; REFUNDED — reserved canonical state (§26).
- Provider-specific holatlar canonical'ga adapter'da map qilinadi; ortiqcha holat ko'paytirilmaydi.
- **Activation invariant (F-9):** frontend success/redirect sahifasi hech narsani activate qilmaydi — u faqat order statusini poll qiladi. ACTIVE holatga olib keladigan yagona yo'l — §23 dagi server-side verified callback/verification tranzaksiyasi.

> **Implementation split (Phase 2.1E/2.1F/2.1G):** the accepted §23 verified-callback transaction is materialized in
> stages. **2.1E** creates the PENDING attempt (order CREATED→PENDING). **2.1F (COMPLETE)** verifies the provider
> callback and transitions **PaymentTransaction PENDING→SUCCEEDED** + records `PaymentCallbackEvent` (`result` carries
> ACCEPTED / DUPLICATE / a rejection code) — this is **trusted evidence only**: the order intentionally stays PENDING,
> no IZL is spent, no Subscription is created. The single atomic economic finalization (order **PAID** + IZL REDEEM +
> reservation CONSUMED + redemption APPLIED + Subscription + Cycle) is **Phase 2.1G**. Success-only v1:
> FAILED/CANCELLED/REFUNDED have no runtime producer yet. See
> [PAYMENT_VERIFICATION_IMPLEMENTATION.md](PAYMENT_VERIFICATION_IMPLEMENTATION.md).

> **Phase 2.1G COMPLETE (2026-08-21):** the §23 atomic finalization is now implemented (TD-205..210, see
> [PAYMENT_FINALIZATION_IMPLEMENTATION.md](PAYMENT_FINALIZATION_IMPLEMENTATION.md)). A trusted SUCCEEDED transaction +
> PENDING order finalizes in one internal DB transaction (lock order `sub → pay → izl`, no provider call) → `PaymentOrder`
> PAID + `Subscription` ACTIVE (new episode / EXPIRED reactivation; ACTIVE → recoverable conflict) + `SubscriptionCycle`
> (periodStart=confirmedAt, periodEnd=calendar-month, net snapshots, reward-enabled/disabled) + `SubscriptionCycleEntitlement`
> snapshot; discounted also REDEEM + reservation CONSUMED + redemption APPLIED. Replay-safe (order PAID + cycle-per-order
> + one-REDEEM) and recoverable (SUCCEEDED+PENDING backlog). Still deferred: SUBSCRIPTION_RENEWAL, FAILED/CANCELLED/REFUNDED,
> refund/reversal, real Click/Payme, notifications, frontend.

## 25. Renewal / recurring extension

- **MVP:** manual renewal — yangi PaymentOrder (purpose=SUBSCRIPTION_RENEWAL) → verified payment → yangi cycle. Model bunga to'liq yetarli.
- **Future auto-renew (OPEN):** Click/Payme recurring modeli final emas. Bo'lsa: provider-side billing token'ning faqat **provider-safe reference'i** (token id string) alohida kelajak entity'da saqlanadi — **karta ma'lumotlari/credentiallar bizning DB'da hech qachon saqlanmaydi**. Hozir jadval yaratilmaydi — extension point.

## 26. Refund / reversal extension

Refund policy — **OPEN PRODUCT QUESTION** (subscription taqdiri? earned IZL? ishlatilgan IZL?). Model tayyorligi (policy'siz):

- PaymentTransaction'da REFUNDED canonical state reserved;
- kelajak RefundRecord entity — extension point (hozir yaratilmaydi);
- IZL ledger REVERSAL apparati mavjud (§7) — refund'ga bog'liq IZL korreksiyalari uchun;
- jarayon manual staff review orqali boshlanadi (audit bilan); avtomatik refund flow yo'q.

Hech bir refund mavjud history'ni delete qilmaydi (F-11).

## 27. Redemption

**PROPOSED (TD-57): redemption = business record + ledger transaction (ikkalasi).** Ledger entry "qancha" ni aytadi, IZLRedemption "nima uchun/qanday holatda" ni.

### IZLRedemption

| Field | Izoh |
|---|---|
| id, user_id | |
| type | enum: SUBSCRIPTION_DISCOUNT (MVP; kelajak: GAME_CURRENCY, CASH_OUT — **enum kengayishi uchun ochiq, hozir implement qilinmaydi**) |
| amount_izl | bigint |
| izl_rate_snapshot | UZS per IZL — redemption paytidagi rate |
| value_uzs | bigint — amount × rate snapshot |
| payment_order_id | nullable FK (discount turi uchun) |
| status | REQUESTED → RESERVED → APPLIED / RELEASED (cancelled) |
| ledger_entry_id | nullable FK — REDEEM entry (**faqat APPLIED bo'lganda**) |
| created_at, resolved_at | |

**Flow (owner clarification, TD-57 — reservation-first):**

1. **Reservation:** checkout boshlanishida redemption REQUESTED→RESERVED; wallet.reserved_amount += amount (spendable kamayadi: total 500, reserved 100 → spendable 400); **ledger entry hali yozilmaydi**.
2. **Payment success:** trusted confirmation tranzaksiyasi ichida RESERVED→APPLIED: reserved kamayadi + **REDEEM ledger entry** + total balance kamayadi.
3. **Payment failure/expiry/cancellation:** RESERVED→RELEASED: reserved bo'shatiladi; **REVERSAL kerak emas** — actual REDEEM bo'lmagan.
4. **Already-applied reversal:** REDEEM final bo'lgach business reversal/refund kerak bo'lsa — yangi REVERSAL entry; eski entry o'zgarmaydi.

Reservation/order expiry exact policy va cleanup mechanism — implementation/queue bosqichida (OPEN texnik detal).

**Cash-out (§36):** yaratilmaydi; `type` enum kengayishi + support/manual flow — kelajak; **OPEN qoladi**.

## 28. IZL discount / conversion

- PaymentOrder gross/discount/payable **snapshot** qiladi (§21) — tarixiy to'lov keyingi rate/narx o'zgarishidan mustaqil (misol: 300,000 − 40,000 = 260,000 abadiy shu ko'rinishda qoladi).
- **IzlRateVersion (TD-58):** rate — versioned config: id, rate_uzs_per_izl (bigint), effective_from, status, created_by. `1 IZL = X UZS` qiymati ham, **o'zgarishi mumkinligi ham OPEN product qarori** — versioned model ikkala javobga mos (immutable bo'lsa bitta qator abadiy turadi). Rate ishlatilgan har joy (cycle, redemption) snapshot saqlaydi — rate o'zgarsa tarix buzilmaydi.

## 29. Admin adjustments

- Faqat ADJUSTMENT ledger entry: amount (±) + reason + actor + timestamp; staff audit (D-35) bilan bog'lanadi; balance overwrite yo'q (F-2).
- **Negative adjustment balansdan katta bo'lsa:** default recommendation — **rad etiladi (balance flooring: balans 0 dan pastga tushmaydi)**; istisno hollar (fraud clawback va h.k.) uchun negative balance ruxsat etiladimi — **OPEN PRODUCT QUESTION**. Model ikkalasiga tayyor (signed amounts), constraint policy application'da.

## 30. Concurrency / transactions

**PROPOSED (TD-59):** PostgreSQL darajasida conceptual strategiya:

- **Per-wallet serialization:** har grant/redeem/adjustment tranzaksiyasi IZLWallet qatorini lock qiladi (SELECT FOR UPDATE uslubi) — parallel yozuvlar navbatlanadi, balance/ceiling race yo'q.
- **Per-cycle lock:** ceiling tekshiruvi (earned + amount ≤ ceiling) cycle qatorini lock qilib bajariladi — parallel ikkita grant ceiling'dan oshira olmaydi (F-6).
- **Bitta tranzaksiya:** RewardGrant + LedgerEntry + Wallet + Cycle.earned — atomik; xuddi shunday callback processing (§23) va redemption flow (§27).
- **Unique constraintlar backstop:** dedup_key, provider_event_id, provider_transaction_id, (subscription, sequence_no) — lock o'tkazib yuborilgan holatlarda ham ikkilanish imkonsiz.

## 31. Audit boundaries

| Tizim | Nima uchun | Nima EMAS |
|---|---|---|
| **IZL Ledger** | Qiymat harakatining business truth'i | Kim admin panelda nima bosgani emas |
| **StaffAudit** (D-35; TD-81 ACCEPTED entity — [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md)) | "Kim qaysi administrative action qildi" — adjustment, pricing change, plan change, refund initiation; append-only, lightweight polymorphic target | Qiymat hisobi emas |
| **Payment jadvallari** | Provider bilan bo'lgan haqiqat/tarix | Ichki qiymat oqimi emas |
| **SecurityEvent** (1.1) | Auth/security hodisalari | Moliyaviy audit emas |

Universal `logs` jadvali YO'Q. ADJUSTMENT kabi kesishmalarda: ledger entry (qiymat) + staff audit yozuvi (harakat) — ikkalasi, har biri o'z tizimida.

## 32. Retention / deletion

- Financial/reward history (ledger, grants, redemptions, payment jadvallari, cycle'lar) — **hech qachon destructive delete qilinmaydi** (TD-60, TD-30 davomi).
- User erasure/privacy legal policy OPEN — yo'nalish: **anonymization/pseudonymization** (user reference'lar anonimlashtiriladi, summalar/strukturaviy tarix saqlanadi — moliyaviy izchillik buzilmaydi); hard-delete policy invent qilinmaydi.
- Minors payment/reward cheklovlari — legal review OPEN (D-32); schema'ga taxminiy parental-consent jadvallari qo'shilmaydi — extension point.

## 33. Indexing / volume

**Indexes (obvious):** IZLWallet — PK(user); IZLLedgerEntry — (user_id, created_at), (subscription_cycle_id), (reward_grant_id); RewardGrant — unique(user_id, dedup_key), (subscription_cycle_id), evidence FK'lar; Subscription — partial unique(user_id non-terminal), (user_id, status); SubscriptionCycle — unique(subscription_id, sequence_no), (period_start/end); UsageCounter — unique(cycle, feature_code); PaymentOrder — (user_id, status), (created_at); PaymentTransaction — unique(provider, provider_transaction_id); PaymentCallbackEvent — unique(provider, provider_event_id); PlanPrice — (plan_id, effective_from); IZLRedemption — (user_id, status).

**Partitioning:** MVP'da KERAK EMAS — ledger/payment hajmi attempt jadvallaridan ancha kichik bo'ladi; to'g'ri indexlangan PostgreSQL yetarli. Future path: vaqt bo'yicha archive/partition — extension point, hozir qo'shilmaydi.

## 34. Entity Catalog

| Entity | Purpose | Mutable/Append-only | Key relations | Phase |
|---|---|---|---|---|
| XpBalance | XP aggregate + level cache | Mutable (derived) | 1:1 User | NOW |
| XpGrant | Yengil XP history | Append-only (yengil) | N:1 User | NOW |
| IZLWallet | Balans cache + lock anchor | Mutable (faqat ledger bilan) | 1:1 User | NOW |
| IZLLedgerEntry | IZL business truth | **Append-only, qat'iy** | User, Grant?, Redemption?, Cycle?, reversal_of? | NOW |
| RewardGrant | Reward qarori + evidence + idempotency | Append-only (status: GRANTED→REVERSED) | User, Cycle, PolicyVersion, evidence FK, LedgerEntry | NOW |
| RewardPolicyVersion | Versioned reward config | Append-only versiyalar | Grants/Cycles reference | NOW |
| IzlRateVersion | 1 IZL = X UZS versiyalari | Append-only versiyalar | Cycle/Redemption snapshot | NOW |
| IZLRedemption | Redemption business record (reservation-first) | Status lifecycle: REQUESTED→RESERVED→APPLIED/RELEASED | User, Order?, LedgerEntry (faqat APPLIED) | NOW |
| SubscriptionPlan | Tarif identity | Mutable (lifecycle) | 1:N Price, Entitlement | NOW |
| PlanPrice | Narx versiyalari | Append-only (effective_to yopiladi) | N:1 Plan | NOW |
| PlanEntitlement | Feature→limit matritsa | Mutable (joriy konfiguratsiya) | N:1 Plan | NOW |
| Subscription | User obunasi | Mutable (status) | User (bitta non-terminal), Plan; 1:N Cycle | NOW |
| SubscriptionCycle | **Markaziy iqtisod snapshot'i** | Yaratilgach snapshotlar immutable; earned/status mutable | Subscription, PlanPrice, PolicyVersion, PaymentOrder; 1:N UsageCounter | NOW |
| SubscriptionChange | Plan o'zgarish audit'i | Append-only | Subscription | NOW |
| UsageCounter | Cycle feature quota hisobi | Mutable counter | unique(Cycle, feature_code) | NOW |
| SubscriptionCycleEntitlement | Cycle entitlement'larining normalized immutable snapshot'i | Immutable | unique(Cycle, feature_code) | NOW |
| PaymentOrder | Business to'lov intenti | Status lifecycle; summalar snapshot | User, Plan/Price, Redemption?; 1:N Transaction | NOW |
| PaymentTransaction | Provider tranzaksiyasi | Status lifecycle | N:1 Order; unique provider tx id | NOW |
| PaymentCallbackEvent | Webhook dedup/audit | Append-only | Provider event unique; Transaction? | NOW |
| RefundRecord | Refund jarayoni | — | extension | LATER (policy OPEN) |
| BillingTokenRef | Auto-renew provider token reference | — | extension | LATER (recurring OPEN) |
| CashOut/GameCurrency | Redemption kengaytmalari | — | Redemption.type kengayadi | LATER/FUTURE |

## 35. Relationship Map

```
User
 ├──1:1── XpBalance          ├──1:N── XpGrant
 ├──1:1── IZLWallet (balance cache + reserved_amount; spendable = balance − reserved)
 │            └──(balance = SUM; wallet-local entry_no)── IZLLedgerEntry (append-only)
 │                                        ├── reward_grant_id ──▶ RewardGrant
 │                                        ├── redemption_id  ──▶ IZLRedemption
 │                                        └── reversal_of_entry_id (self)
 ├──1:N── RewardGrant ──▶ evidence: ActivityAttempt | LearningSession | DailyMissionCompletion (──1:N── Evidence → Post/Attempt/Session)
 │              ├──N:1── RewardPolicyVersion
 │              └──N:1── SubscriptionCycle (ceiling hisobi)
 ├──0..1── Subscription (non-terminal bittadan) ──N:1── SubscriptionPlan
 │              │                                         ├──1:N── PlanPrice (versioned)
 │              │                                         └──1:N── PlanEntitlement
 │              ├──1:N── SubscriptionChange (append-only)
 │              └──1:N── SubscriptionCycle (unique sequence_no)
 │                          ├── snapshots: gross/discount/paid/reward_basis/ceiling/rate/policy
 │                          ├──1:N── SubscriptionCycleEntitlement (normalized immutable)
 │                          ├──1:N── UsageCounter (feature_code)
 │                          └──N:1── PaymentOrder (cycle'ni yaratgan to'lov)
 └──1:N── PaymentOrder (gross/discount/payable snapshot)
              ├──0..1── IZLRedemption (REQUESTED→RESERVED→APPLIED/RELEASED)
              └──1:N── PaymentTransaction (canonical status; unique provider tx)
                           └──1:N── PaymentCallbackEvent (unique provider event — idempotency)

IzlRateVersion ──snapshot──▶ SubscriptionCycle / IZLRedemption / PaymentOrder
```

## 36. Invariants

1. **F-1:** IZLLedgerEntry append-only — UPDATE/DELETE hech qachon; korreksiya faqat REVERSAL/ADJUSTMENT entry bilan.
2. **F-2:** IZLWallet.balance to'g'ridan-to'g'ri overwrite qilinmaydi — faqat ledger entry bilan bir tranzaksiyada o'zgaradi; balance == SUM(ledger).
3. **F-3:** XP va IZL bir jadval/ledger'da aralashmaydi.
4. **F-4:** Har RewardGrant kategoriyasiga mos roppa-rosa bitta evidence FK'ga ega; evidence'siz grant yo'q (login-only reward strukturaviy imkonsiz).
5. **F-5:** unique(user, dedup_key) — bir evidence uchun IZL bir marta (DB-darajali anti-farming).
6. **F-6:** Cycle earned_izl hech qachon reward_ceiling_izl'dan oshmaydi; tekshiruv cycle lock ostida.
7. **F-7:** SubscriptionCycle iqtisod snapshotlari (price, ceiling, rate, policy, entitlements) yaratilgandan keyin immutable.
8. **F-8:** PlanPrice/RewardPolicyVersion/IzlRateVersion tarixiy qatorlari mutable emas — o'zgarish = yangi versiya; joriy narx/policy oldingi cycle'ni o'zgartirmaydi.
9. **F-9:** Frontend callback/success sahifasi Subscription'ni ACTIVE qila olmaydi — faqat server-verified provider confirmation tranzaksiyasi.
10. **F-10:** Bitta verified payment roppa-rosa bitta SubscriptionCycle yaratadi; provider callback idempotent (unique event id).
11. **F-11:** Refund/reversal mavjud history'ni delete qilmaydi — faqat yangi yozuvlar qo'shadi.
12. **F-12:** Reservation hech qachon spendable'dan oshmaydi (wallet lock ostida); REDEEM ledger entry faqat RESERVED→APPLIED transition'ida yoziladi; RELEASED reservation ledgerga umuman tegmaydi.
13. **F-13:** Bitta user'da bir vaqtda ko'pi bilan bitta non-terminal Subscription.
14. **F-14:** UsageCounter faqat tegishli domain entity yaratilishi bilan bir tranzaksiyada o'sadi.
15. **F-15:** ADJUSTMENT har doim reason + actor bilan; staff audit yozuvisiz admin moliyaviy amal yo'q.
16. **F-16:** Financial jadvallardan hech biri destructive hard-delete qilinmaydi; erasure — anonymization yo'li (policy OPEN).
17. **F-17:** spendable = total_balance − reserved_amount; reservation ledgerdagi financial movement emas — faqat APPLIED haqiqiy debit.
18. **F-18:** Ledger entrylar wallet ichida strictly monotonic entry_no'ga ega (unique(user, entry_no)); ordering authority — entry_no, UUID timestamp emas.

## 37. Open questions

**Product (owner qarori kerak):**

1. Exact `1 IZL = X UZS` qiymati; **rate keyin o'zgarishi mumkinmi** (IzlRateVersion ikkalasiga tayyor).
2. Ceiling asosi: gross plan narximi yoki IZL discount'dan keyingi to'langan summami (§11; default: gross).
3. Reward kategoriya qiymatlari/taqsimoti, mastery threshold (RewardPolicyVersion.config — data).
4. IZL expiry bo'ladimi (ledger enum'i tayyor, policy yo'q).
5. Redemption/cash-out policy (cash-out OPEN, recommend qilinmaydi — legacy holat saqlanadi).
6. Tarif narxlari va feature matrix (PlanPrice/PlanEntitlement — data).
7. Subscription expiry behavior (block/read-only/grace) — Entitlement Resolver policy'si.
8. Entitlement o'zgarishi joriy obunachilarga darhol qo'llanadimi (default: keyingi cycle'dan).
9. Upgrade/downgrade siyosati (immediate/prorated/next-cycle).
10. Recurring/auto-renew Click/Payme bilan bo'ladimi.
11. Refund policy (subscription/IZL taqdiri bilan).
12. Negative balance/clawback ruxsatimi (default: balance flooring 0).
13. Minors payment/reward cheklovlari (legal review).

**Texnik (implementation'da):**

14. Provider callback verification detallari (Click/Payme protokollari) — adapter dizayni.
15. Operational raw-payload log retention.
16. Reconciliation job dizayni (wallet vs ledger SUM).
17. UsageCounter overshoot chegarasi monitoring; kerak bo'lsa reservation extension.

## 38. Recommended model summary

1. **XP ≠ IZL (TD-45):** XpBalance + yengil XpGrant; IZL — financial-style.
2. **Pul (TD-46):** hamma joyda integer bigint (IZL units, UZS so'm); float yo'q; provider formati — adapter.
3. **IZL (TD-47):** append-only signed ledger (double-entry rad — closed-loop uchun overengineering) + wallet cache (lock anchor, reserved_amount bilan; spendable = balance − reserved); wallet-local monotonic entry_no ordering; korreksiya faqat REVERSAL/ADJUSTMENT.
4. **Rewards (TD-48/49):** RewardGrant boundary — evidence FK'lar (polymorphic rad) + dedup_key unique idempotency + RewardPolicyVersion reference.
5. **Ceiling (TD-50):** SubscriptionCycle'da UZS+IZL snapshot; earned cached, lock ostida tekshiriladi.
6. **Plans (TD-51/52):** Plan identity + PlanPrice versiyalari (tarix immutable) + generic PlanEntitlement + **normalized SubscriptionCycleEntitlement snapshot** (opaque JSONB emas) + Entitlement Resolver (qo'llash policy'si OPEN'ga tayyor).
7. **Usage (TD-53):** UsageCounter per (cycle, feature); consumption domain entity bilan atomik; reservation apparati MVP'da yo'q.
8. **Subscription (TD-54):** minimal statuslar (ACTIVE/EXPIRED/CANCELLED); **SubscriptionCycle — markaziy iqtisod entity'si** (barcha snapshotlar shu yerda); SubscriptionChange audit log.
9. **Payments (TD-55/56):** PaymentOrder (business, summalar snapshot) ≠ PaymentTransaction (provider) ≠ PaymentCallbackEvent (idempotency); provider port; frontend hech qachon activate qilmaydi; callback atomic+idempotent.
10. **Redemption (TD-57/58):** reservation-first — REQUESTED→RESERVED (spendable kamayadi, ledger tegmaydi) → payment success'da APPLIED + REDEEM entry / fail'da RELEASED (REVERSAL'siz); applied'dan keyingi reversal — yangi REVERSAL entry. Rate — IzlRateVersion, har ishlatilgan joyda snapshot.
11. **Concurrency (TD-59):** wallet/cycle row lock + bitta tranzaksiya + unique backstop.
12. **Retention (TD-60):** financial history append-only, delete yo'q; erasure — anonymization (OPEN).
