# Izlan — Prisma Schema v1 (Phase 1.3)

> Status: Phase 1.3 COMPLETE (2026-08-27). Birinchi real database implementation.
> Manba: TD-01..TD-92 (ACCEPTED), [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md), [JSONB_GOVERNANCE.md](JSONB_GOVERNANCE.md).
> Kod: `backend/prisma/schema/*.prisma`, `backend/prisma.config.ts`, `backend/prisma/migrations/`.
> Hali NestJS/API/business logic YO'Q — faqat database foundation.

## Environment

| | Qiymat |
|---|---|
| Node | v24.17.0 |
| Package manager | npm 11.13.0 |
| Prisma CLI / Client | **7.9.1** (pinned; devDependency `prisma@7.9.1`, `@prisma/client@^7.9.1`) |
| PostgreSQL | 18.4 (local; `izlan_dev` database) |
| Generator | `prisma-client-js` (Prisma 7'da hali ishlaydi — validate/generate o'tdi) |
| Config | `prisma.config.ts` (Prisma 7 modeli — quyida) |
| TypeScript | 7.0.2 (config yuklashda ishladi) |
| Driver adapter | `@prisma/adapter-pg` (o'rnatilgan; migrate uchun Prisma 7'da avtomatik) |

### Prisma 7 config eslatmasi (arxitektura o'zgarishi)

Prisma 7 `datasource.url`ni schema faylidan olib tashladi. Connection URL endi `prisma.config.ts` ichida:

```ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";
import path from "node:path";
export default defineConfig({
  schema: path.join("prisma", "schema"),      // multi-file folder
  migrations: { path: path.join("prisma", "migrations") },
  datasource: { url: env("DATABASE_URL") },   // .env orqali (dotenv/config kerak)
});
```

`schema.prisma` datasource bloki faqat `provider = "postgresql"` (url yo'q). `relationMode` default (`foreignKeys`) — referential integrity PostgreSQL FK bilan.

## Schema organization

Multi-file schema (`prisma/schema/` folder), domen bo'yicha:

| Fayl | Modellar | Soni |
|---|---|---|
| `schema.prisma` | datasource, generator, 49 enum | — |
| `core.prisma` | User, UserProfile, Role, UserRole, RolePermission, SubjectAssignment, AuthSession, RefreshToken, OtpChallenge, SecurityEvent, StaffAudit, MediaAsset | 12 |
| `content.prisma` | Subject, Track, Level, Module, Topic, Lesson, LessonRevision, Activity, ActivityMedia, Skill, LessonSkill, ActivitySkill, LessonPrerequisite | 13 |
| `learning.prisma` | Assessment{Definition, DefinitionVersion, Item, VersionItem, ItemMedia, Attempt, Response}, AiEvaluation, LearnerSkillState, SkillMeasurement, LearningSession, ActivityAttempt, LearnerLesson{Progress, Completion}, LearnerRoadmap, RoadmapItem, LearnerRecommendation, RoadmapChange, LearnerSignal, Checkpoint, DailyPlan, DailyPlanItem, DailyMissionCompletion, DailyMissionCompletionEvidence, LearningSchedulePreference, WeeklyProgress, **LearnerLearningIntent** (TD-93, Phase 1.5A-2) | 27 |
| `finance.prisma` | XpBalance, XpGrant, IZLWallet, IZLLedgerEntry, RewardGrant, RewardPolicyVersion, IzlRateVersion, IZLRedemption, SubscriptionPlan, PlanPrice, PlanEntitlement, Subscription, SubscriptionCycle, SubscriptionCycleEntitlement, SubscriptionChange, UsageCounter, PaymentOrder, PaymentTransaction, PaymentCallbackEvent, PaymeMerchantTransaction (2.1L-D), ClickShopTransaction (2.1L-D) | 21 |
| `community.prisma` | CommunityPost, CommunityReply, ReactionType, CommunityReaction, CommunityPostMedia, CommunityReport, ModerationAction, CommunityRestriction, ReputationBalance, ReputationEvent, Announcement, AnnouncementUserState, Notification | 13 |

**Jami: 84 model, 49 enum** (Phase 1.5A-2'da +1: `LearnerLearningIntent`)**.** DB'da: 85 jadval (+`_prisma_migrations`), 173 FK (+3 RESTRICT: user/subject/track), **24 CHECK** (1.5C-2 +1: sm derivation non-empty), **14 partial unique** (1.5B-2 +2 initial-diagnostic uniqueness, 1.5C +1 measurement idempotency), 152 unique index (+1: `unique(user_id, subject_id)`). Phase 1.5C: `SkillMeasurement.derivationVersion` ustuni (1.5C-2'da NOT NULL + non-empty CHECK).

## UUIDv7 implementation (TD-23)

**Qaror: application/Prisma-generated UUIDv7** — har PK `@id @default(uuid(7)) @db.Uuid`. Prisma Client insert paytida UUIDv7 (time-ordered) generatsiya qiladi; PostgreSQL native `uuid` type. DB-side default YO'Q (`gen_random_uuid()` ishlatilmaydi — u v4).

**Sabab:** Prisma `createMany`/`create` bilan to'liq mos; DB extension (masalan `pg_uuidv7`) talab qilmaydi — portable; time-ordered index locality (v7 xususiyati). **Cheklov:** kelajakda kerak bo'lsa raw SQL bulk insert UUIDv7'ni o'zi generatsiya qilishi kerak (yoki DB extension qo'shiladi) — hozir bunday yo'l yo'q.

## Numeric implementation (TD-89)

| Kategoriya | Prisma | PostgreSQL | Bound |
|---|---|---|---|
| UZS summalar (narx, ceiling, to'lov) | `Int` | integer | CHECK ≥ 0 (payment/cycle) |
| IZL (balans, amount, ceiling) | `Int` | integer | CHECK ≥ 0 (wallet), signed (ledger amount) |
| XP, reputation, counters, entry_no, attempt_no, position, sequence_no | `Int` | integer | — / CHECK ≥ 0 (usage) |
| mastery_score_bp, confidence_bp, score_bp | `Int` | integer | **CHECK 0..10000** (basis points) |
| response_time_ms, active_seconds, size_bytes, duration_seconds | `Int` | integer | — |
| rate_uzs_per_izl | `Int` | integer | — (precision izohи quyida) |

Float pul uchun hech qayerda ishlatilmagan. BigInt ishlatilmagan (TD-89: "smallest safe integer width; BigInt by habit emas") — bironta field int4 chegarasiga real yaqinlashmaydi; kerak bo'lsa keyin int8 migratsiyasi.

**Precision flag (TD-58/89):** `IzlRateVersion.rateUzsPerIzl` integer. Agar kelajakda `1 IZL = X.YZ UZS` kabi kasrli kurs kerak bo'lsa, integer UZS-per-IZL yetarli aniqlik bermasligi mumkin (masalan 166.67). Bu **matematik noaniqlik OPEN** — rate qiymati product qarori bilan birga aniqlanганда, kerak bo'lsa minor-unit yoki numeric(precision) ga o'tiladi. Hozir integer (accepted contract; qiymat OPEN).

## Constraint implementation

| Mexanizm | Soni | Misollar |
|---|---|---|
| **Prisma-native** (`@id`, `@unique`, `@@unique`, `@relation onDelete`, `@@index`) | ~ | PK, oddiy UNIQUE (phone, dedup, sequence), FK actions, composite index |
| **PostgreSQL custom SQL** (migration.sql oxirida) | 24 CHECK + 14 partial unique | XOR, basis points, reserved≤balance, earned≤ceiling, entitlement mode↔limit, "one CURRENT/ACTIVE/PUBLISHED/non-terminal", reaction/report dedup, **one published diagnostic per subject + one in-progress initial diagnostic per user+subject** (1.5B-2), **skill-measurement idempotency + derivation non-empty** (1.5C/1.5C-2) |
| **Application-only** (schema+DB enforce qila olmaydi) | — | append-only (PRIV/service), DAG cycle prevention, cross-row parent-consistency, state machines, trust/module boundary |

Partial unique = `partialIndexes` **preview** Prisma 7.9.1'da → production policy (matritsa) preview'ni rad etadi → **custom PostgreSQL migration** (default). CHECK har doim custom SQL. To'liq mapping: [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) Phase 1.3 section.

## Important circular relations

Named `@relation` + nullable FK bilan hal qilingan (Prisma buni qo'llaydi):

| Pointer | Named relation | Parent-consistency invarianti |
|---|---|---|
| `Lesson.publishedRevisionId` → LessonRevision | `LessonPublishedRevision` | Revision shu Lesson'niki — **app-enforce** (C6); + partial unique bitta PUBLISHED revision (C7, DB) |
| `AssessmentDefinition.currentVersionId` → Version | `DefinitionCurrentVersion` | Version shu Definition'niki — **app-enforce** |
| `Subscription.currentCycleId` → Cycle | `SubscriptionCurrentCycle` | Cycle shu Subscription'niki — **app-enforce** |
| `CommunityPost.acceptedReplyId` → Reply | `PostAcceptedReply` | Reply shu Post'niki + type=QUESTION — **app-enforce** (K5) |
| `IZLLedgerEntry.reversalOfEntryId` → self | `LedgerReversal` | Reversal semantikasi — app |
| `RefreshToken.replacedById` → self (implicit) | — | Rotation zanjiri — app |

Cross-row equality'ni Prisma FK o'zi enforce qila olmaydi — application/service qatlamida (kerak bo'lsa PG composite-FK texnikasi keyin qo'shilishi mumkin). Migration order: Prisma nullable circular FK'larni to'g'ri tartibda yaratadi (avval jadvallar, keyin ALTER TABLE ADD FK).

## Delete policy (FK onDelete)

Umumiy: **RESTRICT default + archive-first** (TD-30/60). Migration'da: 133 RESTRICT (+3 LearnerLearningIntent: user/subject/track — content archive intentni delete qilmaydi), 23 SET NULL (optional pointer/actor references), 17 CASCADE.

- **RESTRICT (default):** User → barcha history; content zanjiri; MediaAsset ← barcha junction/response FK; assessment version'lar; ledger/grant/cycle/payment o'zaro; completion/attempt history.
- **CASCADE (faqat xavfsiz):** draft LessonRevision → Activity; Activity → ActivityMedia/ActivitySkill; junction'lar (LessonSkill, CommunityPostMedia, AnnouncementUserState); DailyPlan → DailyPlanItem; RolePermission ← Role. Published/history hech qachon cascade emas.
- **SET NULL:** granted_by/assigned_by/actor/resolved_by/revoked_by kabi optional references; SecurityEvent.user; LearningSession.dailyPlan.
- **User hard-delete:** hech qanday financial/learning history User'dan cascade EMAS (erasure/anonymization policy OPEN — legal). RESTRICT bilan himoyalangan.

**RESTRICT SQLSTATE:** PostgreSQL RESTRICT delete violation = **23001** (`restrict_violation`), `23503` emas — application error handling shuni bilishi kerak.

## JSONB (JSONB_GOVERNANCE.md)

`@db.JsonB` faqat hujjatlashtirilgan joylarda (arbitrary `metadata` har modelga qo'shilmagan): Activity.payload, AssessmentItem.payload, AssessmentDefinitionVersion.config, {ActivityAttempt,AssessmentResponse}.answer, AiEvaluation.{rubric,providerMetadata}, AssessmentAttempt.{engineState,resultSummary}, LearnerLessonProgress.completedActivities, LearnerSignal.evidenceRefs, LearnerRecommendation.proposedChange, RoadmapChange.changePayload, {RoadmapItem,DailyPlanItem}.params, DailyPlan.context, RewardPolicyVersion.config, XpGrant.sourceRefs, PaymentTransaction.providerMetadata, MediaAsset.moderationMetadata, SecurityEvent.metadata, StaffAudit.metadata, Notification.params. Media identity JSONB'da EMAS — junction FK (TD-82).

## Migration

| | |
|---|---|
| Migration 1 | `20260819100830_init` (immutable — tahrirlanmaydi) |
| Migration 2 | `20260819141741_add_learner_learning_intent` (Phase 1.5A-2, TD-93) |
| Migration 3 | `20260819160000_harden_initial_diagnostic_constraints` (Phase 1.5B-2, TD-94/95; 2 partial unique) |
| Migration 4 | `20260819170000_add_skill_measurement_derivation_version` (Phase 1.5C, TD-97/98; +1 column, +1 partial unique) |
| Migration 5 | `20260819180000_harden_skill_measurement_derivation_version` (Phase 1.5C-2; derivation_version NOT NULL + non-empty CHECK) |
| Migration 6 | `20260820120000_add_lesson_skill_measurement_idempotency` (Phase 1.7C, TD-111; +1 partial unique — SP-10, lesson-backed measurement idempotency) |
| Migration 7 | `20260820130000_harden_skill_measurement_merge_metadata` (Phase 1.8A, TD-113; +2 columns evidence_count/observed_at NOT NULL, +1 CHECK LP-01, +1 index) |
| Migration 8 | `20260820140000_add_learner_signal_active_uniqueness` (Phase 1.8B, TD-117; +1 partial unique — SG-01, one ACTIVE signal per user+skill+type) |
| Migration 9 | `20260820150000_add_review_session` (Phase 1.9B-2, TD-125/126/127/128; +1 enum ReviewSessionStatus, +2 tables LearnerReviewSession/LearnerReviewSessionActivity, +ActivityAttempt.review_session_id FK, +1 partial unique RS-DB-01, +1 CHECK RS-DB-04) |
| Migration 10 | `20260820160000_add_review_mastery_measurement` (Phase 1.9C, TD-129/130; +REVIEW_MASTERY enum value, +SkillMeasurement.review_session_id FK+index, +1 partial unique RM-DB-02) |
| Migration 11 | `20260820170000_harden_daily_mission_completion` (Phase 2.0B, TD-136/137; +4 columns mission_code/policy_version/local_date/timezone_snapshot NOT NULL, daily_plan_item_id → nullable, +@@index(user_id, local_date), +1 partial unique DM-DB-01, +3 CHECK DM-DB-02/03/04) |
| Migration 12 | `20260820180000_add_xp_grant_mission_provenance` (Phase 2.0C-2, TD-140/141; XpGrant +2 columns daily_mission_completion_id FK(Restrict)/policy_version_code, +1 partial unique XP-DB-01, +2 CHECK XP-DB-03/04; RewardGrant/IZL/XpBalance TEGILMADI) |
| Migration 13 | `20260820190000_activate_xp_balance_projection` (Phase 2.0D, TD-145/146; XpBalance +1 column progression_version_code, current_level default 0→1, +2 CHECK XPP-DB-01/02; XpGrant/RewardGrant/IZL TEGILMADI) |
| Migration 14 | `20260820200000_activate_izl_wallet_reservation` (Phase 2.1B, TD-156/158; **+new IZLReservation table** [enum IzlReservationStatus, unique(user,idempotency_key), +2 CHECK RES-DB-01/03], IZLWallet +projection_version_code +1 CHECK, **−2 CHECK** chk_wallet_balance_nonneg/chk_wallet_reserved_le_balance DROP [signed projection]; IZLLedgerEntry/RewardGrant/IZLRedemption TEGILMADI) |
| Migration 15 | `20260820210000_payment_order_purchase_intent` (Phase 2.1C-PO, TD-168/170; PaymentOrder.provider NOT NULL→**nullable**, +1 partial unique PO-DB-01 uq_payment_order_client_request; pricing/discount/payable + chk_order_amounts o'zgarmadi; Subscription/Cycle/PaymentTransaction/IZL TEGILMADI) |
| Migration 16 | `20260820220000_subscription_discount_redemption_intent` (Phase 2.1C-2, TD-174/175; IZLRedemption +clientRequestId/policyVersionCode + paymentOrderId FK SetNull→**Restrict**, IZLReservation +redemptionId typed UNIQUE FK Restrict; +2 partial unique RD-DB-01/04 + 4 CHECK RD-DB-05/06/07/08; IZLLedgerEntry TEGILMADI) |
| Migration 17 | `20260820230000_subscription_discount_commit` (Phase 2.1D, TD-178/180; custom SQL only — izl_redemption_id ustuni mavjud edi; +1 partial unique DC-DB-01 uq_payment_order_izl_redemption; IZLRedemption/IZLReservation/ledger TEGILMADI) |
| Migration 18 | `20260820240000_payment_execution_intent` (Phase 2.1E, TD-183/184/187; PaymentTransaction.provider_transaction_id NOT NULL→**nullable** + `client_request_id` ustuni; +2 partial unique PT-DB-01 uq_payment_transaction_client_request / PT-DB-02 uq_payment_transaction_pending; 0 yangi CHECK; PaymentCallbackEvent/PAID/IZL/Subscription TEGILMADI) |
| Migration 19 | `20260820250000_verified_payment_evidence` (Phase 2.1F, TD-189/193; custom SQL only — confirmed_at / provider_transaction_id / payment_callback_event mavjud edi; +1 partial unique PV-DB-01 uq_payment_transaction_succeeded (payment_order_id) WHERE status='SUCCEEDED'; 0 column, 0 yangi CHECK; PaymentOrder PAID / IZL / Subscription TEGILMADI) |
| Migration 20 | `20260821000000_finalization_schema_hardening` (Phase 2.1G-D, TD-195/197/198/201; PlanPrice +billing_period_months [backfill 1, NOT NULL], SubscriptionCycle reward_policy_version_id + izl_rate_snapshot NOT NULL→**nullable** [reward-disabled cycle], IzlReservationStatus +CONSUMED enum value, +2 CHECK FP-DB-01/02-03, +1 partial unique FP-DB-04 uq_izl_ledger_redeem_per_redemption; NO finalizer/PAID/Subscription/Cycle/REDEEM/CONSUMED/APPLIED producer) |
| Migration 20 (note) | Phase **2.1G** (Verified Payment Economic Finalization) added **NO migration** — the finalizer reuses the post-migration-20 schema (PV-DB-01, SubscriptionCycle.paymentOrderId UNIQUE, F-14, FP-DB-04, reservation.redemptionId UNIQUE, ledger entryNo UNIQUE). Migration count stays **20**. Phases **2.1H** (Finalization Recovery), **2.1I** (Verified Non-Success Evidence: PT PENDING→FAILED/CANCELLED via existing enum + free callback.result + F-19 + pay lock) **2.1J** (Order Reopen/Retry: order PENDING→CREATED reuses existing statuses + PT-DB-02/PV-DB-01 + pay lock) and **2.1K** (Terminal Reopen Recovery: read-only backlog + delegates to PaymentOrderReopenService, new RBAC permission data only) also added **NO migration** — count remains **20**. |
| Migration 21 | `20260821100000_real_provider_protocol_persistence` (Phase 2.1L-D, TD-233..239; **+2 tables** `payme_merchant_transaction` [1:1 PaymentTransaction unique FK Restrict; payme_transaction_id UNIQUE; amount_tiyin/provider_created_time_ms/create_time_ms/perform_time_ms?/cancel_time_ms? BigInt; state int; reason int?; +index provider_created_time_ms] + `click_shop_transaction` [1:1 PaymentTransaction unique FK Restrict; click_trans_id?/click_paydoc_id?/merchant_prepare_id?/merchant_confirm_id? TEXT; prepare_state/complete_state enum ClickProtocolPhaseState; +1 partial unique uq_click_shop_click_trans_id]; **+1 enum** ClickProtocolPhaseState; **+5 CHECK** chk_payme_mt_state_valid/amount_positive/time_coherent/reason_coherent + chk_click_shop_complete_requires_prepare; **provider protocol persistence only — NO real adapter/route/provider call/refund/PT terminal transition/PaymentOrder/Subscription/IZL mutation**; Payme verified official docs, CLICK provider-neutral shell [CLICK PROTOCOL VERIFICATION BLOCKER §0]). Count **20→21**, named CHECK **40→45**, partial unique **28→29**. Batafsil: [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md). |
| Generated | `prisma migrate dev --create-only` (init) · `prisma migrate dev` (LI) · custom SQL (harden, derivation, 1.5C-2, 1.7C, 1.8A, 1.8B) · `migrate diff --from-config-datasource --to-schema` + custom SQL (1.9B-2, 1.9C, 2.0B, 2.0C-2, 2.0D, 2.1B, 2.1C-PO, 2.1C-2, 2.1D, 2.1E, 2.1F) · hand-authored backfill + enum ADD VALUE (2.1G-D) |
| Custom reviewed | **40 CHECK** + **28 partial unique** (…+2 payment-transaction client-request/one-pending +1 payment-transaction one-succeeded +1 izl-ledger one-REDEEM-per-redemption; +4 CHECK redemption +2 CHECK 2.1G-D billing-period/cycle-reward-coherence); +2 plain unique (izl_reservation idempotency + redemption_id 1:1); reference: `prisma/migrations/_custom_constraints.reference.sql` |
| Applied | `prisma migrate deploy` → clean `izlan_dev` + `izlan_test` DB'ga muvaffaqiyatli |
| Drift check | `prisma migrate status` → "Database schema is up to date!" (dev + test) |
| Client generation | `prisma generate` → OK (v7.9.1) |

## Critical constraint verification

Transaction ROLLBACK ichida 21 test (18 negative violation + 3 positive) — **21/21 PASS, 0 FAIL**:

phone unique · UserRole unique · basis-points bounds · reserved≤balance · AiEvaluation XOR · Reaction XOR · MissionEvidence XOR · one CURRENT DailyPlan · one ACTIVE roadmap · one non-terminal Subscription · entitlement LIMITED/ENABLED mode↔limit · duplicate callback · reward dedup · MediaAsset RESTRICT · earned≤ceiling · ledger ADJUSTMENT reason+actor · prerequisite self-ref · (positive) multiple lesson completion (relearning) · assessment version chain (def→version→item→attempt) · valid basis points.

## Deferred implementation rules (schema o'zi enforce qila olmaydi)

Bular application/service qatlamida (Phase 1.4+), [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) reference bilan:

- **Append-only** (ledger, measurement, completion, roadmap change, moderation action, reputation event, staff audit, mission completion): repository'da UPDATE/DELETE yo'llari yo'q; ixtiyoriy DB PRIV (app-role'dan UPDATE/DELETE revoke) — **deployment role dizayni** (hozir yo'q, deployment note).
- **Cross-row parent-consistency** (circular pointer'lar) — service validation.
- **DAG cycle prevention** (LessonPrerequisite) — saqlash txn'ida.
- **State machine transitions** (payment, redemption, subscription, response/attempt submit-immutability).
- **Trust boundary** (F-9: frontend subscription activate qila olmaydi) — API dizayni.
- **Module boundary** (C-8/C-14/K-15: community wallet/XP'ni bevosita mutate qilmaydi).
- **Concurrency** (wallet/cycle row lock, ledger entry_no monotonic) — service transaction.
- **Reconciliation** (wallet balance vs ledger SUM; community counters) — job.
- **Content validation** (TD-22 discriminated union payload; media_asset_id↔junction sinxron).
- **Timezone** (TD-91: core learning oldidan resolved IANA tz).

## Known cleanup / non-blocking

- Prisma `migrate dev` keyingi safar custom CHECK/partial indexlarni schema.prisma'da ko'rmaydi (ular custom SQL) — Prisma ularni "drift" deb bermaydi (o'zi boshqarmaydigan obyektlar), lekin schema.prisma introspection to'liq manzarani bermaydi; [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) — yagona to'liq manba.
- `prisma-client-js` generator Prisma 7'da ishlaydi; kelajakda yangi `prisma-client` (ESM, output) generator'ga o'tish ko'rib chiqilishi mumkin — implementation qarori, hozir kerak emas.
