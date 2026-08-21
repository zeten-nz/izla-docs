# Izlan — Technical Decision Log

> Texnik qarorlar log'i. Product qarorlari: [PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md).
> **Data architecture holati:** Phase 1.2A..1.2D — COMPLETE (TD-21..TD-92 ACCEPTED). **Phase 1.3 (Prisma Schema v1) — COMPLETE** (2026-08-27): `backend/` da schema+migration+client, verification 21/21 PASS ([PRISMA_SCHEMA_V1.md](PRISMA_SCHEMA_V1.md)).

## Phase 1.3 — Prisma Schema v1 (implementation notes)

Yangi arxitektura qarori emas — TD-01..92 ijrosi. Implementation tanlovlari ([PRISMA_SCHEMA_V1.md](PRISMA_SCHEMA_V1.md)):

- **IN-01 — Prisma 7.9.1 (pinned):** current stable. Prisma 7 `datasource.url`ni schema'dan olib tashladi → `prisma.config.ts` (`defineConfig` + `dotenv/config` + `env()`). Generator `prisma-client-js` (ishlaydi).
- **IN-02 — UUIDv7 generation joyi (TD-23 OPEN edi):** **application/Prisma-generated** — `@default(uuid(7)) @db.Uuid`; DB-side default yo'q (portable, extension'siz, createMany-mos). Raw bulk insert kelajakda UUIDv7'ni o'zi bersin.
- **IN-03 — Partial unique (TD-owner policy ijrosi):** `partialIndexes` Prisma 7.9.1'da preview → production policy preview'ni rad etadi → 11 partial unique **custom PostgreSQL migration**da. CHECK (23) ham custom SQL.
- **IN-04 — Numeric (TD-89 ijrosi):** hamma joyda `Int` (integer); BigInt ishlatilmadi; basis points CHECK 0..10000. **Flag:** `IzlRateVersion.rateUzsPerIzl` integer — kasrli kurs kerak bo'lsa precision noaniqligi (rate qiymati OPEN bilan birga hal qilinadi).
- **IN-05 — RESTRICT SQLSTATE:** PostgreSQL RESTRICT delete = `23001` (`restrict_violation`), `23503` emas — app error handling shuni bilishi kerak.
- **IN-06 — Append-only PRIV deferred:** append-only jadvallar hozir repository-discipline bilan; DB-level UPDATE/DELETE revoke — deployment role dizayni (kelajak).
- **Naming:** modellar PascalCase, DB snake_case (`@@map`/`@map`); barcha column snake_case (userId/roleId ham `@map`landi).

## Phase 1.4A — Backend foundation (implementation notes)

Yangi arxitektura qarori emas — accepted stack ijrosi ([BACKEND_FOUNDATION.md](BACKEND_FOUNDATION.md)):

- **IN-07 — NestJS 11.2.1 + Fastify 5 (platform-fastify):** `NestFastifyApplication` + `FastifyAdapter` (Express emas). Global prefix `/api`.
- **IN-08 — Prisma 7 driver adapter:** `new PrismaPg({ connectionString }) → new PrismaClient({ adapter })` PrismaService ichida; PrismaPg pool egasi, `$disconnect()` yopadi. Accelerate/Data Proxy yo'q. Bitta managed Prisma boundary per process (§33); `new PrismaClient()` modullarda tarqalmaydi.
- **IN-09 — Module system: CommonJS:** Nest decorators + Prisma client (`default.js`) + jest CommonJS'da barqaror; ESM'ga o'tilmadi.
- **IN-10 — TypeScript 5.9.3 (1.3'dagi TS 7 native preview'dan tushirildi):** Nest ekotizimi bilan barqaror; Prisma config bilan ham mos. Schema/migration o'zgarmadi (zero churn).
- **IN-11 — Config: @nestjs/config global + explicit `validateEnv` fail-fast:** class-validator/zod TASODIFAN kiritilmadi (business validation library qarori keyinga). Secret log qilinmaydi.
- **IN-12 — DatabaseModule @Global:** cross-cutting infrastructure; takroriy import noise'i yo'q.
- **IN-13 — Health: minimal custom (Terminus yo'q); liveness ≠ readiness;** DB ping `SELECT 1` read-only; readiness fail → 503.
- Phase 1.3 schema/migration **o'zgarmadi** — faqat `prisma generate`; drift yo'q.

## Phase 1.4B — Auth & Users core foundation (implementation notes)

TD-08..TD-20 ijrosi ([AUTH_FOUNDATION_IMPLEMENTATION.md](AUTH_FOUNDATION_IMPLEMENTATION.md)). HTTP flow yo'q (1.4C).

- **IN-14 — Phone normalization:** bitta backend authority `normalizeUzPhone`; canonical E.164 `+998`+9 raqam; frontend'ga tayanmaydi (TD-16).
- **IN-15 — OTP hashing: HMAC-SHA-256 + server pepper** (TD-11 accepted option). 6-raqam `crypto.randomInt` (leading zero saqlanadi); constant-time verify; raw OTP saqlanmaydi/log qilinmaydi; DB codeHash. Tuning (TTL 180s/cooldown 60s/max 5/hourly 5/daily 10) env orqali — product qarori emas.
- **IN-16 — Refresh token: 256-bit opaque (base64url) + SHA-256 hash** (TD-09). Rotation atomik conditional `updateMany` (count) — bir parent'dan ≤1 aktiv child; reuse → family revoke + SecurityEvent. Grace window yo'q (strict; parallel false-positive 1.4C).
- **IN-17 — Repository/transaction pattern:** service `PrismaService.$transaction` boundary ochadi, query'lar repository'da (optional `tx?: Prisma.TransactionClient`); UnitOfWork abstraction yo'q.
- **IN-18 — System role bootstrap:** idempotent `upsert` (LEARNER/METHODIST/MODERATOR/ADMIN); `npm run db:seed:system`; faqat role identity (demo yo'q).
- **IN-19 — Test DB isolation:** izlan_test (TEST_DATABASE_URL) + NODE_ENV/db-name guard; cleanup faqat auth jadvallar; izlan_dev casual ishlatilmaydi.
- **IN-20 — Auth crypto: Node built-ins** (crypto randomInt/randomBytes, HMAC, SHA-256, timingSafeEqual) — bcrypt/argon2/JWT/Passport o'rnatilmadi (§56).
- Schema/migration **o'zgarmadi** (0 churn); drift yo'q.

## Phase 1.4C — Auth HTTP flow + access JWT + web security (implementation notes)

Owner-accepted implementation qarorlari ([AUTH_HTTP_IMPLEMENTATION.md](AUTH_HTTP_IMPLEMENTATION.md)):

- **IN-21 — Access JWT: RS256** (asymmetric; `jsonwebtoken` to'g'ridan, @nestjs/jwt/Passport yo'q). Minimal claims (sub/sid/typ/iss/aud/iat/exp); TTL 900s. Verify: algorithms ['RS256'] (confusion block), issuer/audience/kid/typ tekshiruvi.
- **IN-22 — Access transport (web): response body → memory Bearer** (localStorage/cookie YO'Q — frontend contract).
- **IN-23 — Refresh transport (web): HttpOnly cookie** `izlan_refresh`, SameSite=Lax, Path=/api/auth/refresh, Domain omitted, Secure (prod majburiy — env validation). Manual serialize/parse (external cookie dep yo'q).
- **IN-24 — CSRF: header + Origin + Sec-Fetch-Site** (custom `X-Izlan-CSRF: 1` + exact Origin allowlist + cross-site rad + credentialed CORS; sinxron CSRF token store yo'q).
- **IN-25 — JWT kid + key rotation readiness:** `AUTH_JWT_PUBLIC_KEYS_JSON` map (kid→PEM); additive rotation (DB key jadvali yo'q). Keys base64 env; startup fail-fast parse; `npm run generate:jwt-keys`.
- **IN-26 — Auth HTTP adapter:** global AuthGuard + @Public + PermissionsGuard (APP_GUARD); AuthExceptionFilter (domain→HTTP, leak/enumeration-safe `{statusCode,code,message}`); ValidationPipe (whitelist+forbidNonWhitelisted); bounded access-token revocation (sensitive routelar live-check; global blacklist/Redis yo'q).
- **Production SMS: NOT CONFIGURED** — UnavailableSmsAdapter default; TestSmsAdapter test-only override. Schema/migration o'zgarmadi.

## Phase 1.5A — Learner onboarding + profile (implementation notes)

Profile/onboarding foundation ([LEARNER_ONBOARDING_IMPLEMENTATION.md](LEARNER_ONBOARDING_IMPLEMENTATION.md)). Schema/migration **o'zgarmadi**.

- **IN-27 — DOB date-only:** YYYY-MM-DD → UTC midnight `@db.Date`, timezone shift yo'q; age saqlanmaydi (D-32); post-onboarding DOB PATCH rad etiladi (`PROFILE_DOB_LOCKED`) — xavfsizlik cheklovi, policy OPEN.
- **IN-28 — IANA timezone validator:** Node `Intl.supportedValuesOf` + `DateTimeFormat` fallback (dependency yo'q); canonical string; TD-91 (profil tz o'zgarishi tarixiy snapshotlarni qayta yozmaydi).
- **IN-29 — Onboarding completion:** authority `onboardingCompletedAt`; conditional first-write (idempotent); required = displayName/dateOfBirth/timezone; qo'shimcha boolean yo'q; resumable (DB-derived).
- **IN-30 — Profile DTO allowlist + explicit field map:** ValidationPipe whitelist/forbidNonWhitelisted + repository explicit map (mass assignment yo'q); IDOR — principal.userId (target request'dan olinmaydi).
- **IN-31 — Content discovery:** narrow read-only repo, faqat PUBLISHED, deterministik tartib; SubjectAssignment (staff) va LearnerRoadmap (assessment-derived) learner selection uchun ISHLATILMAYDI.

### TD-93 — LearnerLearningIntent (ACCEPTED by Product Owner, Phase 1.5A-2)
- **Decision:** Learner subject/track selection persistence gap owner review'dan o'tdi → yangi model **LearnerLearningIntent** (id, userId, subjectId, trackId nullable, timestamps; unique(userId, subjectId)). Real schema o'zgarishi — migration `20260819141741_add_learner_learning_intent` (dev+test apply, drift yo'q).
- **Semantika:** learnerning muayyan Subject bo'yicha **hozirgi learning direction'i**. **NOT:** enrollment / roadmap / assessment / staff assignment / subscription.
  - **Multi-subject:** bir user bir nechta Subject bo'yicha parallel intent (unique per user+subject) — shuning uchun UserProfile.selectedSubjectId EMAS (REJECTED).
  - **track nullable:** onboarding resumable (subject-only saqlanadi).
  - **No status/goal field:** unique(user,subject) qatorining o'zi = current intent; goal/motivation/targetScore YO'Q (Track direction'ni ifodalaydi); intent-history/versioning yo'q.
  - **Current preference, historical emas:** assessment reproducibility joriy intentga tayanmaydi (immutable version refs assessment modellarida — §28).
  - **Assessment/roadmap coupling YO'Q:** assessmentAttemptId/roadmapId qo'shilmadi (reverse coupling oldini olindi).
  - **SubjectAssignment (staff) va LearnerRoadmap (assessment-derived) — ISHLATILMADI** (REJECTED options).
- **Onboarding readiness:** profile fields (displayName/dob/timezone) + kamida bitta COMPLETE (trackId to'ldirilgan, hozir PUBLISHED) LearningIntent. Constraint: [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) LI-01..06.
- **API:** GET /api/onboarding/learning-intents, PUT /api/onboarding/learning-intent (single resumable endpoint, trackId optional).
- **Status:** ACCEPTED. Learner selection persistence gap **RESOLVED**.

## Phase 1.5B / 1.5B-2 — Placement (initial diagnostic) assessment (implementation notes)
> Status: **COMPLETE** (1.5B foundation + 1.5B-2 hardening, 2026-08-19; owner review closed the gap). To'liq: [PLACEMENT_ASSESSMENT_IMPLEMENTATION.md](PLACEMENT_ASSESSMENT_IMPLEMENTATION.md).
- **IN-32 — Enum mapping (implementation reading, not a new decision):** "placement" = initial diagnostic → `purposeScope=DIAGNOSTIC`, `attemptPurpose=INITIAL_DIAGNOSTIC`. Schema'da `PLACEMENT` enum YO'Q.
- **IN-33 — Version pinning + pool scope (TD-83 ijrosi):** attempt `definitionVersionId`ga pin; engine faqat `AssessmentVersionItem` pooldan; resume/submit pinned versiyani id bo'yicha (status filtri YO'Q → ARCHIVED ham resumable). Definition **Subject-scoped** (trackId ustuni YO'Q); track attemptda saqlanadi.
- **IN-34 — Objective scorer (backend authority, TD-89):** `ObjectiveScorerService` — single/multiple/true_false; type-specific normalization (canonical option-id, universal toLowerCase YO'Q); exact-match, partial credit YO'Q; 10000/0 basis points. Answer key `payload.answerKey` (server-only) — `projectItemForLearner` strip qiladi (§52 leak test).
- **IN-35 — Concurrency/idempotency:** submit — `$transaction` + guarded `updateMany where status=PRESENTED` (count===1 winner, session-rotation patterni); loser idempotent replay. `clientRequestId` schema'da YO'Q — qo'shilmadi (DTO'da qabul, persist YO'Q); idempotency structural (presented-row + already-submitted replay). Double-start → mavjud IN_PROGRESS attempt resume.
- **IN-36 — JSONB contracts (TD-92 ijrosi):** runtime validators — `PLACEMENT_ADAPTIVE_V1` (config), `PLACEMENT_ITEM_V1` (payload), engineState (class E), resultSummary (class B, `"B2"` label YO'Q). Malformed config → `ASSESSMENT_CONFIGURATION_INVALID` fail-safe (maxItems majburiy — infinite attempt YO'Q). Hech qanday `as PlacementConfig` blind cast YO'Q.
- **IN-37 — Evidence-only boundary (§40):** assessment LearnerSkillState/SkillMeasurement/Roadmap/DailyPlan/XP/IZL YOZMAYDI, AI provider CHAQIRMAYDI.

### Phase 1.5B-2 hardening (owner review closed) — TD-94/95/96 ACCEPTED
- **IN-38 — camelCase answer JSON (OD/§2):** learner answerlar `{selectedOptionId}` / `{selectedOptionIds:[...]}` (application/HTTP JSON camelCase). Stale conceptual `selected_option_id` → camelCase (DB snake_case naming O'ZGARMADI).
- **IN-39 — Multiple-choice duplicate rad etiladi:** `["A","A","B"]` → 400 ASSESSMENT_INVALID_RESPONSE (silent canonicalize EMAS). Exact set match, partial credit YO'Q.
- **IN-40 — clientRequestId OLIB TASHLANDI:** DTO/API'dan (schema ustuni yo'q → mavjud bo'lmagan idempotency kafolatini bildirmasin). Idempotency structural: presented-row + guarded transition + canonical replay/conflict.
- **IN-41 — Immutable response replay/conflict:** bir item ikkinchi marta — canonical answer bir xil → idempotent 200; **boshqa answer → 409 ASSESSMENT_RESPONSE_CONFLICT** (immutable response boshqa javobni jimgina qabul qilmaydi). Format-specific canonicalization.
- **IN-42 — Start/resume authority = (user, subject) IN_PROGRESS INITIAL_DIAGNOSTIC:** version-bo'yicha EMAS. Current pointer v2 ga o'zgarsa ham mavjud attempt v2 ga REPIN qilinmaydi; concurrency → DB partial-unique + P2002 catch/resume.
- **IN-43 — Per-skill adaptive state:** global `targetDifficulty` o'rniga har skill uchun alohida `{targetDifficulty, answeredCount}`. Bir skill javobi boshqa skill target'ini o'zgartirmaydi. Skill-balanced selection (lowest answeredCount, tie-break skillId asc; skill ichida nearest-target). Skill ID'lar EXACT pinned pooldan.
- **IN-44 — Coverage + feasibility:** config `coverage.itemsPerSkill` (yangi), `stopping.minItems` OLIB TASHLANDI (dead). Stopping: har skill quota, yoki unseen item yo'q, yoki maxItems. Start'da `distinctSkills × itemsPerSkill > maxItems` → ASSESSMENT_CONFIGURATION_INVALID (attempt yaratilmaydi). Insufficient pool → repeat YO'Q, result `coverageComplete=false` + `insufficientSkillIds`.
- **IN-45 — Objective-only v1 + pool validation:** `placement-adaptive-v1` faqat single/multiple/true_false. Start'dan oldin BUTUN pinned pool validated (payload PLACEMENT_ITEM_V1, format objective, skillId, difficulty positive int); open_ended/unsupported → ASSESSMENT_CONFIGURATION_INVALID (attempt yaratilmaydi). AiEvaluation provider — hali NONE. resultSummary `pendingAiEvaluation` OLIB TASHLANDI.

### TD-94 — Initial diagnostic subject-level uniqueness (ACCEPTED)
- **Decision:** bir Subject uchun ko'pi bilan bitta PUBLISHED DIAGNOSTIC `AssessmentDefinition`. Track = assessment context, definition-selection dimensioni EMAS (MVP subject-level). Partial-unique index (migration `20260819160000_harden_initial_diagnostic_constraints`; PA-07). Runtime fail-safe (>1 → CONFIGURATION_INVALID) defense-in-depth sifatida qoladi.

### TD-95 — Initial diagnostic in-progress uniqueness (ACCEPTED)
- **Decision:** bir (user, subject) uchun ko'pi bilan bitta IN_PROGRESS INITIAL_DIAGNOSTIC attempt. Partial-unique index (PA-10). Completed attemptlar ta'sirlanmaydi; concurrent start DB constraint bilan yakuniy hal (P2002 → resume winner). REASSESSMENT policy OCHIQ.

### TD-96 — Placement adaptive engine v1 contract (ACCEPTED as implementation engine contract)
- **Decision:** `placement-adaptive-v1` — ACCEPTED **implementation engine contract**: deterministik, skill-balanced, objective-only mechanika + per-version config (`PLACEMENT_ADAPTIVE_V1`) + immutable-version reproducibility.
- **Clarification (owner):** TD-96 yakuniy psychometric validity yoki CEFR thresholdlarni ANIQLAMAYDI. Methodist quyidagilarni egallaydi: item calibration, config parametr QIYMATLARI, kelajakdagi confidence algoritmlari, CEFR/Level mapping. Difficulty = subject-neutral ordinal (global CEFR mapping YO'Q, fixed upper bound YO'Q).

## Phase 1.5C — Skill Profile derivation (implementation notes)
> Status: **COMPLETE** (2026-08-19). To'liq: [SKILL_PROFILE_IMPLEMENTATION.md](SKILL_PROFILE_IMPLEMENTATION.md). Migration: `20260819170000_add_skill_measurement_derivation_version` (derivation_version column + assessment-backed idempotency partial-unique).
- **IN-46 — profileScale (config kengaytmasi):** `placement-adaptive/v1` config'ga `profileScale{minDifficulty,maxDifficulty}` (max>min, startDifficulty ichida, har effective item difficulty ichida — start & derivation'da enforce). Version-pinned immutable normalization shkalasi (CEFR EMAS; qiymatlar methodist-owned).
- **IN-47 — Difficulty-aware mastery:** placement-adaptive-v1 target net-correctness walk (difficulty-INSENSITIVE, 1.5B §32 test'lar bilan qulflangan) → mastery uning target'idan EMAS, javob berilgan itemlar DIFFICULTY'sidan (correctness bilan): `estimate = correct ? diff : max(min, diff-1)`; skill bo'yicha o'rtacha; profileScale bo'yicha 0..10000 normalize. **Same accuracy, higher boundary → higher mastery** (§45 test). Raw percent-correct EMAS (§10).
- **IN-48 — Coverage confidence:** `confidenceBp = min(1, evidenceCount/itemsPerSkill)*10000` — statistik certainty EMAS (§12). evidenceCount = skill bo'yicha SUBMITTED objective javoblar soni.
- **IN-49 — Auto-derivation orchestration:** `AssessmentService` evidence-only qoladi; `PlacementFlowService` (assessment→skill-profile bir tomonlama import) completion'da alohida transaction'da derive qiladi. Derivation fail bo'lsa completed attempt ROLLBACK QILINMAYDI (log + idempotent retry/repair endpoint). Circular Nest dependency YO'Q (skill-profile assessment jadvallarini o'z repository'si bilan o'qiydi + pure config parser'ni file-import qiladi).
- **IN-50 — Chronology guard:** `lastMeasurementAt = completedAt` (observation time, job time EMAS). Eski backfill yangi state'ni REGRESS QILMAYDI (guarded updateMany + createMany skipDuplicates; concurrency-safe, P2002 tx-abort YO'Q). displayLevel = null (CEFR OPEN, §5/43).

### TD-97 — Diagnostic Skill Profile v1 derivation (ACCEPTED — FINAL immutable formula; refined Phase 1.5C-2)
- **Decision (exact immutable v1 contract):** `skill-profile-diagnostic-v1` — raw authority `AssessmentResponse`; effective item difficulty pinned version pooldan; **correct evidence = d, incorrect evidence = d − 1 (previous ordinal rank)**, `clamp(e, profileScale.min, profileScale.max)`; Skill bo'yicha **arithmetic mean** evidence difficulty; profileScale normalize → basis points; **coverage-based confidence** `min(1, evidenceCount/itemsPerSkill)*10000`; evidenceCount = skill bo'yicha SUBMITTED objective javoblar; `displayLevel = null`; `derivationVersion = skill-profile-diagnostic-v1`.
- **Clarification (owner):** bu "final adaptive target" derivation'ni ALMASHTIRADI — final target ~ startDifficulty + correct/incorrect transitions, bir xil accuracy'li lekin turli difficulty'li evidence setlarini farqlay olmasligi mumkin; response-difficulty estimator actual item difficulty + correctness'ni saqlaydi (MVP uchun mos). **`-1` = previous ordinal band, `config.selection.stepDown` EMAS** (stepDown = next-item targeting; alohida semantika). Formula IMMUTABLE bu version ostida — matematik o'zgarish ⇒ `skill-profile-diagnostic-v2`. **CEFR mapping ACCEPTED EMAS** (OPEN).
### TD-98 — SkillMeasurement derivation identity + idempotency (ACCEPTED; hardened Phase 1.5C-2)
- **Decision:** `SkillMeasurement.derivationVersion` — **NOT NULL + non-empty CHECK** (`chk_sm_derivation_version_nonempty`, SP-09); DB default YO'Q (caller explicit beradi; `'unknown'`/`'v1'` default TAQIQLANGAN). Har measurement uni ishlab chiqargan formula/jarayonni aniqlaydi. Assessment-backed idempotency partial-unique `(assessment_attempt_id, skill_id, source, derivation_version) WHERE assessment_attempt_id IS NOT NULL` (SP-04) — derivationVersion NOT NULL bo'lgach semantikasi kuchliroq. derivationVersion idempotency'da qatnashadi → kelajakdagi `-v2`/`ENGINE_RECALC` eski measurement'ni o'chirmasdan yangi tarixiy qator qo'sha oladi (§19). SkillMeasurement append-only. Migration `20260819180000_harden_skill_measurement_derivation_version` (dev+test empty tasdiqlandi → backfill YO'Q).

## Phase 1.6A — Roadmap Foundation (implementation notes)
> Status: **COMPLETE** (2026-08-19). **SCHEMA O'ZGARMADI** — Phase 1.3 roadmap/content-mapping/prerequisite/provenance/one-active schema yetarli edi. To'liq: [ROADMAP_FOUNDATION_IMPLEMENTATION.md](ROADMAP_FOUNDATION_IMPLEMENTATION.md).
- **IN-51 — Roadmap source = exact diagnostic SkillMeasurement snapshot** (mutable current state EMAS): `(attemptId, source=DIAGNOSTIC, derivationVersion=skill-profile-diagnostic-v1)`. evidenceCount attempt'ning SUBMITTED javoblaridan qayta hisoblanadi (reproducible; current state EMAS). §53 test.
- **IN-52 — Content→Skill mapping = LessonSkill (yagona authority)**; RoadmapItem target = logical Lesson (`itemType=LESSON`, `lessonId`; LessonRevision pin EMAS — roadmap = learning path, published revision execution vaqtida). Prerequisites = LessonPrerequisite (explicit DAG). Title/hierarchy/keyword/AI inference YO'Q.
- **IN-53 — Learner-visible filter:** Lesson PUBLISHED + published LessonRevision (`publishedRevisionId != null`) + Topic→Module→Level chain PUBLISHED; completed lesson (LearnerLessonCompletion) exclude; in-progress qoladi (progress mutate QILINMAYDI).
- **IN-54 — Deterministik tartib:** priority-aware topological sort (prereq before dependent; unblocked'lar orasida effective gap priority DESC — prereq dependent priority'sini meros oladi — keyin Topic.sortOrder, Lesson.sortOrder, lessonId). RoadmapItem.position. Cycle → ROADMAP_CONFIGURATION_INVALID (infinite-loop YO'Q).
- **IN-55 — Idempotency/concurrency:** DB `ux_active_roadmap` (user,subject WHERE ACTIVE, L9 — mavjud) yakuniy authority + sourceAssessmentAttemptId. Bir xil attempt → idempotent replay; boshqa attempt → 409 ROADMAP_ALREADY_ACTIVE (replacement YO'Q); concurrent → P2002 catch → winner resume. Header+items bitta transaction; empty ACTIVE roadmap yaratilmaydi (ROADMAP_NO_ELIGIBLE_CONTENT). uncoveredSkillIds generation response'da (persist YO'Q — schema field yo'q).
- **IN-56 — Read-only boundary:** roadmap LearnerSkillState/SkillMeasurement/LessonProgress/LessonCompletion/DailyPlan/LearnerSignal/XP/IZL/AiEvaluation YOZMAYDI (grep+test). `roadmap-gap-v1`/deterministic selection — code identifierlar (persist YO'Q; LearnerRoadmap engineVersion field yo'q — silently qo'shilmadi; reproducibility immutable inputlardan).

### TD-99 — Initial Roadmap Gap Ranking v1 (ACCEPTED)
- **Decision:** `roadmap-gap-v1` — `gapPriorityBp = round((10000 − masteryScoreBp) × confidenceBp / 10000)` (clamp 0..10000). Ranking-only: threshold YO'Q, label YO'Q, CEFR YO'Q, statistical probability EMAS. Deterministik sort (§6). Har measured skill (evidenceCount>0) qatnashadi.
### TD-100 — Deterministic Initial Roadmap v1 (ACCEPTED)
- **Decision:** exact diagnostic SkillMeasurement snapshot; human-approved PUBLISHED content only (LessonSkill explicit mapping); prerequisite closure (LessonPrerequisite DAG); deterministik sequence (RoadmapItem.position); one ACTIVE roadmap per user+subject (ux_active_roadmap, mavjud); source=INITIAL_GENERATION; AI YO'Q, DailyPlan YO'Q, XP/IZL YO'Q, LearnerSignal YO'Q. Schema o'zgarmadi.

## Phase 1.6B — Roadmap Progress Read Model (implementation notes)
> Status: **COMPLETE** (2026-08-20). **SCHEMA O'ZGARMADI** — derived states + progress persist QILINMAYDI; reconciliation mavjud RoadmapStatus enum bilan. To'liq: [ROADMAP_PROGRESS_IMPLEMENTATION.md](ROADMAP_PROGRESS_IMPLEMENTATION.md).
- **IN-57 — Derived read model (persist YO'Q):** roadmap item state COMPLETED/UNAVAILABLE/IN_PROGRESS/BLOCKED/AVAILABLE application-level derived; progressBp/counts derived; RoadmapItem.status (PENDING) authority sifatida ISHLATILMAYDI/YOZILMAYDI (ikkinchi completion truth EMAS).
- **IN-58 — Progress authorities:** completion = LearnerLessonCompletion (by logical lessonId); progress = LearnerLessonProgress (row existence); availability = current visible content (Lesson PUBLISHED + published revision + Topic→Module→Level PUBLISHED); prerequisite = LessonPrerequisite + actual completion (roadmap membership/order EMAS).
- **IN-59 — Precedence:** COMPLETED > UNAVAILABLE > IN_PROGRESS > BLOCKED > AVAILABLE. Completed content keyin archived bo'lsa ham COMPLETED; valid-at-generation content keyin unpublish → UNAVAILABLE (item o'chirilmaydi, auto-complete QILINMAYDI). Logical Lesson target — current published revision metadata surface (revision pin YO'Q).
- **IN-60 — Next item:** earliest IN_PROGRESS → earliest AVAILABLE → null. RoadmapItem.position canonical (gap re-rank YO'Q). progressBp = round(completed/total*10000), persist YO'Q.
- **IN-61 — Reconciliation:** ACTIVE→COMPLETED iff HAR persisted RoadmapItem lesson completed (available-only EMAS); conditional updateMany (status=ACTIVE) — idempotent, concurrency-safe, COMPLETED→ACTIVE YO'Q, completedAt field yo'q (status only). GET endpointlar hech qachon mutate qilmaydi (§16); reconcile = command + kelajakdagi lesson-completion flow hook. Read-only against progress/completion/SkillState/SkillMeasurement. Batch queries (O(few), N+1 YO'Q).

### TD-101 — Roadmap Progress Read Model v1 (ACCEPTED)
- **Decision:** derived item-state precedence COMPLETED > UNAVAILABLE > IN_PROGRESS > BLOCKED > AVAILABLE. Authorities: completion=LearnerLessonCompletion, progress=LearnerLessonProgress, availability=current visible content, prerequisites=LessonPrerequisite+actual completion. RoadmapItem completion authority sifatida DUBLIKAT QILINMAYDI. Read-only, persist YO'Q.
### TD-102 — Roadmap Completion v1 (ACCEPTED)
- **Decision:** LearnerRoadmap ACTIVE→COMPLETED iff har persisted RoadmapItem lesson authoritative completion'ga ega. Conditional/idempotent transition. Unavailable content uchun avtomatik item removal YO'Q (silently dropped learning YO'Q). Regeneration/replacement policy OCHIQ.

## Phase 1.7A — Daily Plan Foundation (implementation notes)
> Status: **COMPLETE** (2026-08-20). **SCHEMA O'ZGARMADI** — DailyPlan/DailyPlanItem schema (localDate/timezoneSnapshot/generationNo/CURRENT status/roadmapItemId/section/itemType + ux_current_daily_plan L16 + unique(user,localDate,generationNo)) yetarli. To'liq: [DAILY_PLAN_IMPLEMENTATION.md](DAILY_PLAN_IMPLEMENTATION.md).
- **IN-62 — localDate = profile timezone** (server UTC EMAS): `localDateInTimezone(clock.now(), UserProfile.timezone)` (Intl, full ICU). `@db.Date` UTC-midnight. Client date/timezone yubormaydi.
- **IN-63 — Clock boundary:** injectable `Clock.now()` — domain'da `new Date()` tarqatilmaydi; testda fixed/advanceable clock (local-date/midnight/timezone/concurrency reproducibility).
- **IN-64 — Snapshot immutability + TD-91:** timezoneSnapshot/localDate generation'da olinadi, keyin O'ZGARMAYDI (profil tz o'zgarsa mavjud plan qayta yozilmaydi; kelajakdagi generation yangi tz). Item membership/priority snapshot; executability read'da derived (1.6B `deriveItemState` reuse — duplicate state machine YO'Q; §26/27).
- **IN-65 — Topic selection = 1.6B nextItem** (gap/prereq/completion RE-DERIVE QILINMAYDI): nextItem.lesson.topicId = kunning topic'i. Roadmap source = ACTIVE roadmap (bir nechta bo'lsa earliest — primary-subject OPEN). Concurrency/idempotency: ux_current_daily_plan (L16) + P2002 catch/resume; same-day retry = same CURRENT plan (generationNo increment YO'Q, ikkinchi topic YO'Q). Read-only: roadmap/LessonProgress/LessonCompletion/SkillState/SkillMeasurement/reward/AI YOZILMAYDI.

### TD-103 — Daily Plan Roadmap Snapshot v1 (ACCEPTED)
- **Decision:** `daily-plan-roadmap-v1` — profile-local date; immutable timezoneSnapshot/localDate; ACTIVE-roadmap source; nextItem Topic tanlaydi; relational snapshot (roadmapItemId/lessonId/skillId); lesson/reward mutation YO'Q.
### TD-104 — One Topic Per Local Day (ACCEPTED)
- **Decision:** DailyPlan roppa-rosa bitta roadmap Topic; same-day takroriy generation = same CURRENT plan; erta tugatish shu kun ichida keyingi Topic'ni OCHMAYDI; keyingi local date shu Topic'ni davom ettirishi yoki keyingi Topic'ga o'tishi mumkin. (one-topic-per-day ≠ one-day-per-topic.)
### TD-105 — Daily Plan Item Priority v1 (ACCEPTED)
- **Decision:** roppa-rosa bitta MUST_DO = roadmap nextItem; RECOMMENDED = keyingi tugallanmagan SAME-Topic roadmap itemlar (position order); EXTRA auto-generation = NONE (v1). Duration/workload limit YO'Q.

## Phase 1.7B — Lesson Execution Foundation (implementation notes)
> Status: **PASS WITH ARCHITECTURE GAP** (2026-08-20). **SCHEMA O'ZGARMADI** — revision pinning (LearnerLessonProgress.lessonRevisionId TD-37), Activity→revision, ActivityAttempt evidence (clientRequestId/attemptNo/unique) allaqachon mavjud. To'liq: [LESSON_EXECUTION_IMPLEMENTATION.md](LESSON_EXECUTION_IMPLEMENTATION.md).
- **IN-66 — Entry = today's CURRENT DailyPlanItem only** (arbitrary lessonId route YO'Q → one-topic-per-day bypass qilib bo'lmaydi). Live executability 1.6B `deriveItemState` reuse (AVAILABLE/IN_PROGRESS kerak; COMPLETED→LESSON_ALREADY_COMPLETED; BLOCKED/UNAVAILABLE→LESSON_NOT_EXECUTABLE).
- **IN-67 — Revision pinning:** first start Lesson.publishedRevisionId (published chain) ni LearnerLessonProgress.lessonRevisionId ga pin qiladi; resume pinned revision'ni id bo'yicha o'qiydi (status filtri YO'Q → archive-first republish'dan keyin ham pinned qoladi; §9/45). unique(userId,lessonId) idempotency authority; concurrent → P2002 catch/resume.
- **IN-68 — Learner-safe content:** faqat pinned revision title/description/estimatedDurationMin + activities [{id,type,position}]. **Activity.payload body QAYTARILMAYDI** (answer-key layout accepted contract EMAS → xavfsiz strip qilib bo'lmaydi); authoring/lifecycle/source/aiMetadata YO'Q; answer-key leak YO'Q.
- **IN-69 — Boundary:** faqat LearnerLessonProgress yoziladi. LearnerLessonCompletion/ActivityAttempt/Roadmap/DailyPlan/SkillState/SkillMeasurement/Signal/reward/AI YOZILMAYDI (grep+test). `executionReadyForCompletion` boolean invent QILINMADI (required/optional activity semantikasi accepted EMAS).

### TD-106 — Lesson Execution Entry v1 (ACCEPTED)
- **Decision:** faqat today's CURRENT DailyPlanItem'dan; live state AVAILABLE/IN_PROGRESS bo'lishi shart; arbitrary lesson start YO'Q; one-topic-per-day bypass qilib bo'lmaydi.
### TD-107 — LessonRevision Execution Pinning (ACCEPTED)
- **Decision:** first start exact PUBLISHED LessonRevision'ni pin qiladi; resume pinned qoladi; keyingi current-revision o'zgarishlari aktiv progress'ni REPIN QILMAYDI.
## Phase 1.7B-2 — Lesson Activity v1 contract closure (implementation notes)
> Status: **COMPLETE** (2026-08-20, owner review yopdi). **SCHEMA O'ZGARMADI** — ActivityAttempt evidence (clientRequestId, attemptNo, isCorrect, deterministicScore, status) + ux_attempt_client_request (L5) + unique(user,activity,attemptNo) allaqachon mavjud.
- **IN-70 — Objective activity contract (SEPARATE domain):** `lesson-activity-objective/v1` — o'z parser'i (`objective-activity-payload.ts`), PLACEMENT_ITEM_V1 ko'r-ko'rona reuse EMAS (§1/20). camelCase; format single/multiple/true_false; MINI_QUESTION/PRACTICE/MASTERY_TEST; strict validation; multiple-choice duplicate learner selection INVALID; answerKey server-only (projection strip).
- **IN-71 — Deterministic scoring + idempotency:** `ObjectiveActivityScorerService` — exact-match 10000/0, partial credit YO'Q, AI YO'Q. clientRequestId REQUIRED (durable dedup ux_attempt_client_request); same reqId+same answer → replay; same reqId+different → 409 ACTIVITY_ATTEMPT_REQUEST_CONFLICT; attemptNo server-assigned (retry-on-unique loop, concurrency-safe). ActivityAttempt SUBMITTED + append-only. roadmapItemId/learningSessionId null.
- **IN-72 — Progress + boundary:** SUBMITTED activity resume cache'ga qo'shiladi (completedActivities dedup + lastActivityId; incorrect ham "performed step", §30) — completion authority EMAS. LearnerLessonCompletion/Roadmap/DailyPlan/SkillState/SkillMeasurement/Signal/reward/AI YOZILMAYDI. View-only/deferred types submission → ACTIVITY_TYPE_NOT_SUPPORTED. Submit authority = own progress + pinned-revision membership (original DailyPlan today's CURRENT bo'lishi shart EMAS; faqat NEW lesson start uchun).

### TD-108 — ActivityAttempt Evidence v1 (ACCEPTED)
- **Decision:** supported ActivityTypes MINI_QUESTION/PRACTICE/MASTERY_TEST; objective formats single_choice/multiple_choice/true_false; strict camelCase answers; backend deterministic 0/10000 (partial credit YO'Q); ActivityAttempt SUBMITTED + append-only; server attemptNo (unique(user,activity,attemptNo)); durable clientRequestId retry-dedup (ux_attempt_client_request); pinned LessonRevision membership. Writing/Speaking/Listening/AI_interaction ACCEPTED EMAS (deferred).
### TD-109 — Lesson Activity Objective Payload v1 (ACCEPTED)
- **Decision:** `lesson-activity-objective/v1` content contract — Assessment PLACEMENT_ITEM_V1'dan ALOHIDA domain (option/scoring primitivlari o'xshash, lekin coupled EMAS). Payload: schemaVersion/format/prompt/options/answerKey(correctOptionIds, server-only). Publish-vaqti validation kelajakdagi authoring fazasi talabi.
### TD-110 — Lesson Completion v1 (ACCEPTED)
- **Decision:** Lesson complete bo'ladi (bir marta, `completionNo=1`) faqat pinned LessonRevision'ning HAMMA activity'si performed bo'lsa. Objective (MINI_QUESTION/PRACTICE/MASTERY_TEST) performed = SUBMITTED ActivityAttempt mavjud (correctness gate EMAS). View-only (TEXT/EXPLANATION/IMAGE/AUDIO/EXAMPLE) performed = `LearnerLessonProgress.completedActivities` set-union. Unsupported required activity (LISTENING/WRITING/SPEAKING/AI_INTERACTION/VIDEO) → complete BLOCK (`LESSON_COMPLETION_UNSUPPORTED_ACTIVITY`). Zero-activity revision → `LESSON_CONFIGURATION_INVALID`. Pinned revision abadiy saqlanadi (repin YO'Q). `completionPct` OPEN (tegilmaydi). Idempotent; concurrent → bitta completion.
### TD-111 — Lesson Mastery Milestone v1 (ACCEPTED)
- **Decision:** `source=LESSON_MASTERY`, `derivationVersion='lesson-mastery-v1'`, faqat MASTERY_TEST evidence. bestScore = max(deterministicScore) SUBMITTED attemptlar bo'yicha. Attribution: ActivitySkill authority → yo'q bo'lsa LessonSkill fallback → ikkalasi ham yo'q bo'lsa measurement YO'Q (fabricate qilinmaydi). Subject-scope (§66): cross-subject skill exclude. Per skill: `scoreBp=clamp(round(mean(best per attributed mastery activity)),0,10000)`, `confidenceBp=10000`, `evidenceCount`=distinct mastery activity soni (response-only; DB'da column YO'Q), `displayLevel=null`. **LearnerSkillState YOZILMAYDI** (multi-source merge OPEN → Phase 1.8A).
### TD-112 — Lesson Completion → Roadmap Reconciliation (ACCEPTED)
- **Decision:** LessonCompletion = item completion authority; RoadmapItem duplicate authority EMAS. Complete 1.6B `RoadmapService.reconcileCompletion(userId, roadmapId)` hook'ini chaqiradi (faqat lesson'ni o'z ichiga olgan ACTIVE roadmaplar). Idempotent; roadmap ACTIVE→COMPLETED hamma item bitgach. DailyPlan snapshot-only (mutatsiya YO'Q). Reconcile fail → completion baribir turadi (recoverable).
### TD-113 — SkillMeasurement Normalized Evidence Metadata (ACCEPTED)
- **Decision:** Har mergeable milestone abadiy saqlaydi: scoreBp + confidenceBp + evidenceCount + observedAt + source + derivationVersion + provenance. **evidenceCount** NOT NULL, CHECK `evidence_count > 0` (LP-01) — DIAGNOSTIC=skill bo'yicha objective response soni, LESSON_MASTERY=skill'ga attributed distinct MASTERY_TEST activity soni. **observedAt** NOT NULL = logical evidence time (DIAGNOSTIC=attempt.completedAt, LESSON_MASTERY=completion.completedAt; materialization/backfill vaqti EMAS). Merge engine eski pedagogy'ni qayta talqin qilmaydi. Response-only evidenceCount tugadi — creation'da persist qilinadi. Migration `20260820130000` (dev+test bo'sh → backfill YO'Q).
### TD-114 — Learning Progress Merge v1 (ACCEPTED)
- **Decision:** `learning-progress-merge-v1` (IMMUTABLE; formula o'zgarsa → v2). Sources whitelist: DIAGNOSTIC/LESSON_MASTERY/CHECKPOINT (AI_EVALUATION/ENGINE_RECALC merge EMAS, history saqlanadi). Anchor = eng oxirgi DIAGNOSTIC/CHECKPOINT (observedAt DESC → CHECKPOINT>DIAGNOSTIC tie → id stable); window = anchor + observedAt anchor'dan STRICTLY keyin bo'lgan LESSON_MASTERY (anchor'siz → hamma lesson). `effectiveWeight = evidenceCount × confidenceBp` (BigInt, float persist YO'Q). mastery = `clamp(round(Σ(score·evidence·conf)/Σ(evidence·conf)),0,10000)` (denominator 0 → LEARNING_PROGRESS_NO_EFFECTIVE_EVIDENCE); confidence = `round(Σ(conf·evidence)/Σ(evidence))`; evidenceCount = window sum (row soni EMAS); displayLevel=null. Har recompute FULL history'dan qayta quriladi (§30). Anchor reset = recalibration semantics ONLY (retake schedule EMAS).
### TD-115 — LearnerSkillState Single Writer (ACCEPTED)
- **Decision:** Faqat Learning Progress Merge service LearnerSkillState yozadi. Boshqa domenlar SkillMeasurement yaratadi → keyin `recomputeSkills(...)` chaqiradi. SkillProfile `materializeState` OLIB TASHLANDI; LessonCompletion state yozmaydi. Per (user,skill) advisory-lock serialization; SkillMeasurement runtime UPDATE/DELETE YO'Q (append-only §61). LearnerSkillState = rebuildable projection (mergeVersion column YO'Q — formula identity SkillMeasurement.derivationVersion'da).
### TD-116 — Repeated Mistake Signal v1 (ACCEPTED)
- **Decision:** `repeated-mistake-signal-v1` (IMMUTABLE; rule o'zgarsa → v2). Faqat objective lesson ActivityAttempt evidence (SUBMITTED, MINI_QUESTION/PRACTICE/MASTERY_TEST, isCorrect set); AssessmentResponse/diagnostic/view-only/LISTENING/WRITING/SPEAKING/AI_INTERACTION EMAS. Attribution ActivitySkill → yo'q bo'lsa LessonSkill fallback (title/AI inference YO'Q); subject-scope (skill subject == lesson subject). Har (user,skill,Activity) LATEST SUBMITTED outcome'ga collapse (submittedAt DESC, attemptNo DESC, id DESC) — retry spam = 1 distinct. **Trigger:** active signal yo'q bo'lsa, eng oxirgi **3 distinct** Activity outcome hammasi incorrect → ACTIVATE. **Recovery:** active signal bor bo'lsa, eng oxirgi **2 distinct** outcome hammasi correct → RESOLVE. Time-window YO'Q, mastery threshold YO'Q. Multi-skill activity har skill'ga mustaqil hissa (weight YO'Q). Faqat REPEATED_MISTAKE (WEAK_SKILL/REVIEW_DUE deferred).
### TD-117 — Learner Signal Episode Lifecycle (ACCEPTED)
- **Decision:** Har (user, skill, type) uchun ko'pi bilan bitta ACTIVE — partial unique `uq_learner_signal_active` WHERE status='ACTIVE' (SG-01; DB = concurrency authority). ACTIVE → RESOLVED (resolvedAt set) / kelajakda EXPIRED. Terminal rowlar HECH QACHON reactivate bo'lmaydi (RESOLVED/EXPIRED → ACTIVE YO'Q). Recurrence yangi ACTIVE episode yaratadi (eski RESOLVED o'zgarmaydi) — alohida history table YO'Q. v1'da avtomatik EXPIRED policy YO'Q (TTL invent qilinmaydi). evidenceRefs.schemaVersion='repeated-mistake-signal/v1' (yangi column YO'Q; strict minimal snapshot: triggerActivityIds/triggerAttemptIds; answer/key YO'Q).
### TD-118 — Signal Advisory Boundary (ACCEPTED)
- **Decision:** Signal generation MUTATE QILMAYDI: LearnerSkillState, SkillMeasurement, Roadmap/RoadmapItem, DailyPlan/DailyPlanItem, LearnerLessonCompletion, RewardGrant/XP/IZL, DailyMission, Notification, AiEvaluation. Signal = advisory fact/recommendation; kelajakdagi explicit consumer policy qabul qilinmaguncha hech qanday downstream authority YO'Q. LessonExecution submit → evaluateForActivity (downstream advisory; fail → attempt rollback YO'Q, reconcile repair).
### TD-119 — Weak Skill Signal v1 (ACCEPTED)
- **Decision:** `weak-skill-signal-v1` (IMMUTABLE; threshold o'zgarsa → v2). Input authority = **faqat current LearnerSkillState** (masteryScoreBp/confidenceBp/evidenceCount) — raw ActivityAttempt/SkillMeasurement O'QILMAYDI (duplicate pedagogy YO'Q). **Activate:** `mastery < 5000` AND `confidence >= 7000` AND `evidenceCount >= 3` (hammasi; null confidence → 0, gate fail). **Resolve:** `mastery >= 6500` AND `confidence >= 7000`. **Hysteresis** band 5000..6499: create ham, resolve ham QILMAYDI; active vaqtida confidence tushishi resolve QILMAYDI. Recurrence = yangi episode. evidenceRefs snapshot {schemaVersion:'weak-skill-signal/v1', mastery/confidence/evidenceCount/lastMeasurementAt} — active vaqtida qayta yozilmaydi. CEFR semantics YO'Q (operational threshold).
### TD-120 — Review Due Signal v1 (ACCEPTED)
- **Decision:** `review-due-signal-v1` (IMMUTABLE; interval o'zgarsa → v2). Authority = current LearnerSkillState + **Clock.now()** (login/completion/roadmap/plan/REPEATED_MISTAKE EMAS). **Interval** (confidence-first): `confidence < 5000 → 1d`; else `mastery < 5000 → 1d`, `5000..6999 → 3d`, `7000..8499 → 7d`, `>= 8500 → 14d`. `dueAt = lastMeasurementAt + intervalDays × 24h` (exact elapsed duration; server/local timezone/calendar EMAS — DST-safe). **Activate:** active yo'q AND state mavjud AND evidenceCount>0 AND lastMeasurementAt!=null AND now>=dueAt (exact due activate). **Resolve:** current lastMeasurementAt basisLastMeasurementAt'dan STRICTLY keyin (mastery yaxshilanishi shart EMAS; same timestamp resolve QILMAYDI). Recurrence = yangi dueAt'ga yetganda yangi episode. evidenceRefs {schemaVersion:'review-due-signal/v1', basisLastMeasurementAt, mastery/confidence/evidenceCount, intervalDays, dueAt}.
### TD-121 — Time-Based Signal Evaluation Boundary (ACCEPTED)
- **Decision:** State o'zgargandan keyingi evaluation immediate activation/resolution'ni hal qiladi (LearningProgress recompute → evaluateStateSignals, tx tashqarisida, fail → state rollback YO'Q §31). Vaqt o'tishi (REVIEW_DUE eligibility) explicit **reconcile** command yoki kelajakdagi scheduler talab qiladi. **GET read-only** (vaqt o'tgani uchun signal YARATMAYDI). Bu fazada cron/queue/scheduler YO'Q. Pure merge engine signal-unaware; module dep bir tomonlama (LearningProgress → LearnerSignals, cycle YO'Q).
### TD-122 — Review Candidate v1 (ACCEPTED)
- **Decision:** `review-candidate-v1` — DERIVED READ MODEL (persistence YO'Q; ReviewCandidate/ReviewQueue/SpacedRepetitionCard table YARATILMAYDI). Input: faqat ACTIVE REPEATED_MISTAKE/WEAK_SKILL/REVIEW_DUE (whitelist; RESOLVED/EXPIRED/unknown ignore). Signal.skillId grouping authority (null skill → skip). Skill-grouped; composite urgency score YO'Q (reviewPriorityBp/urgencyScore YO'Q); candidate cap YO'Q; scheduler/execution/AI EMAS. Signal type serialization order canonical REPEATED_MISTAKE→REVIEW_DUE→WEAK_SKILL (score EMAS, stable serialization). Group order Skill.sortOrder/name/id; candidate order directTrigger→hierarchy(Level/Module/Topic/Lesson sortOrder)→id. `review-candidate-v2` kelajakda.
### TD-123 — Encountered Content Review Boundary (ACCEPTED)
- **Decision:** Review candidate Lesson: (1) LearnerLessonProgress OR LearnerLessonCompletion mavjud (encountered); (2) hozir learner-visible (Lesson PUBLISHED + published revision + Topic/Module/Level/Track PUBLISHED, subject scope); (3) explicit Skill-relevant. Review ko'rilmagan curriculum'ni "review" sifatida OCHMAYDI (yangi content = Roadmap/DailyPlan). exposure derived (COMPLETED completion bo'lsa, else IN_PROGRESS; persist YO'Q). Archived/unpublished exclude (completion/signal/measurement O'CHIRILMAYDI).
### TD-124 — Review Content Attribution v1 (ACCEPTED)
- **Decision:** General discovery (WEAK_SKILL/REVIEW_DUE, va REPEATED_MISTAKE general): LessonSkill OR **current published revision** ActivitySkill (obsolete old-revision mapping HISOBLANMAYDI). REPEATED_MISTAKE qo'shimcha: evidenceRefs.triggerActivityIds (strict `repeated-mistake-signal/v1` parser) → trigger Activity (har qanday historical revision) → logical Lesson direct provenance (encountered+visible+same-subject bo'lsa directTrigger=true, current mapping shart EMAS §57). Malformed/cross-subject/missing evidence → safe skip. Text/title/AI inference YO'Q. Candidate = logical Lesson (revision pin EMAS, activity replay EMAS).
### TD-125 — Review Session v1 (ACCEPTED, Phase 1.9B-2)
- **Decision:** Dedicated `LearnerReviewSession` aggregate (LearningSession/LessonProgress/LessonCompletion EMAS). Explicit learner start; server-side ReviewCandidate revalidation (prior GET'ga ishonmaydi); bitta target Skill + logical Lesson; **encountered LessonRevision pinned** (repin YO'Q); immutable Activity membership snapshot; ReviewSessionStatus ACTIVE→COMPLETED (terminal; recurrence = yangi row). Fields: userId/skillId/lessonId/lessonRevisionId/status/provenance(JSONB {schemaVersion,signalTypes})/startedAt/completedAt; subjectId duplicate YO'Q (Skill.subjectId'dan). One-ACTIVE per (user,skill,lesson) partial unique (RS-DB-01).
### TD-126 — Review Activity Selection v1 (ACCEPTED, Phase 1.9B-2)
- **Decision:** Pinned encountered revision'da faqat supported objective Activity (MINI_QUESTION/PRACTICE/MASTERY_TEST). Select A iff ActivitySkill(A,target) OR (A'da ActivitySkill YO'Q AND LessonSkill(lesson,target)). LessonSkill fallback FAQAT ActivitySkill'i umuman yo'q activity uchun (explicit boshqa-skill attribution override QILINMAYDI). Direct-trigger (ACTIVE REPEATED_MISTAKE trigger activity ∩ selected ∩ pinned revision) ordering birinchi, keyin Activity.position, keyin id. Relational selected-Activity snapshot (`LearnerReviewSessionActivity`, position) — JSON selectedActivityIds authority EMAS. Zero reviewable → REVIEW_SESSION_NO_REVIEWABLE_ACTIVITY (empty session YO'Q).
### TD-127 — Review Attempt Isolation (ACCEPTED, Phase 1.9B-2)
- **Decision:** `ActivityAttempt.reviewSessionId` = review provenance discriminator (nullable FK). Normal evidence = reviewSessionId **NULL**; review = session id (learningSessionId discriminator EMAS). Review attempt LearnerLessonProgress'ni mutate QILMAYDI (recordActivityStep chaqirilmaydi). Normal LessonCompletion (`submittedActivityIds`) + lesson-mastery-v1 (`bestScores`) query'lari `reviewSessionId IS NULL` filtrladi (RS-08/09) — review attempt normal completion/mastery'ni SATISFY/ALTER QILMAYDI. ReviewSession completion faqat reviewSession-linked attemptlarni consume qiladi. attemptNo shared unique(user,activity,attemptNo) ketma-ketligi (reviewAttemptNo YO'Q).
### TD-128 — Dedicated Review Provenance Aggregate (ACCEPTED, Phase 1.9B-2)
- **Decision:** Generic LearningSession time-tracking aggregate bo'lib qoladi va review-domain FK/context bilan KENGAYTIRILMAYDI. Review provenance alohida historical aggregate oladi (coupling'ni oldini oladi: review semantics ↔ mission/reward/time-session). Bu Phase 1.9B ARCHITECTURE GAP'ining owner-approved yechimi.
### TD-129 — Review Mastery Milestone v1 (ACCEPTED, Phase 1.9C)
- **Decision:** source **REVIEW_MASTERY**, derivationVersion `review-mastery-v1` (immutable). Faqat COMPLETED ReviewSession'dan; bitta target Skill; evidence = session snapshot (LearnerReviewSessionActivity) + reviewSessionId-linked SUBMITTED attemptlar (normal/other-session attempt EMAS). Best attempt per selected Activity (max deterministicScore); scoreBp = clamp(round(arithmetic mean(best per selected Activity)),0,10000); confidenceBp=10000 (snapshot to'liq coverage); evidenceCount = distinct selected Activity soni (retry count EMAS); observedAt = session.completedAt; displayLevel=null. Source re-attribution YO'Q (target Skill authority). Review completion != review mastery score (correctness gate EMAS).
### TD-130 — Review SkillMeasurement Provenance (ACCEPTED, Phase 1.9C)
- **Decision:** `SkillMeasurement.reviewSessionId` nullable FK (Restrict, history-safe). REVIEW_MASTERY uchun reviewSessionId MAJBURIY; lessonId = NULL (recurring review conflict oldini olish — lesson-backed idempotency BLOCK qilmaydi). Review idempotency partial unique `(review_session_id, skill_id, source, derivation_version)` WHERE review_session_id IS NOT NULL (derivationVersion ishtirok etadi → review-mastery-v2 coexist). Recurring Review Session'lar distinct historical measurement. Existing rows reviewSessionId=NULL (backfill YO'Q).
### TD-131 — Learning Progress Merge v2 (ACCEPTED, Phase 1.9C)
- **Decision:** `learning-progress-merge-v2` = CURRENT engine. v1 IMMUTABLE (frozen contract/whitelist, docs/tests saqlanadi). Ikkalasi shared pure core (merge-core.ts)'ga delegate qiladi, faqat source config farq qiladi. Anchor UNCHANGED (DIAGNOSTIC/CHECKPOINT); incremental = LESSON_MASTERY + **REVIEW_MASTERY** (REVIEW_MASTERY hech qachon anchor EMAS §20/25). Formula/weight UNCHANGED. REVIEW_MASTERY window'ni reset qilmaydi (incremental). REVIEW_MASTERY'siz history uchun v2 == v1 byte-for-byte (§27/62). LearnerSkillState mergeVersion column YO'Q (rebuildable projection).
### TD-132 — Review Evidence Signal Recovery (ACCEPTED, Phase 1.9C)
- **Decision:** Normalized review evidence current Skill state'ni recompute qiladi (merge-v2); MAVJUD WEAK_SKILL/REVIEW_DUE policy natijaviy state'ni consume qiladi (recovery); MAVJUD REPEATED_MISTAKE detector review Activity outcome'larini consume qiladi. Review module signal policy'ni O'ZI IMPLEMENT QILMAYDI va learnerSignal'ni to'g'ridan-to'g'ri YOZMAYDI — LearnerSignalsService orqali. Review completion workflow: A completion → B REVIEW_MASTERY measurement → C recompute (merge-v2 + state-signal hook) → D repeated-mistake evaluate; C/D fail → A/B rollback YO'Q (reconcile repair). REVIEW_DUE poor review'da ham resolve bo'ladi (yangi evidence, improvement shart EMAS).
### TD-133 — Daily Review EXTRA v1 (ACCEPTED, Phase 2.0A)
- **Decision:** `daily-review-extra-v1` — NEW DailyPlan generatsiyasida ReviewCandidate'lardan ko'pi bilan BITTA optional review EXTRA qo'shiladi. **Core DailyPlan (TD-103/104/105) o'zgarmaydi** (1 Topic, MUST_DO/RECOMMENDED). EXTRA core Topic bilan BIR XIL Topic'dan (same-topic §11); optional (section=EXTRA, hech qachon MUST_DO/RECOMMENDED); immutable snapshot; ReviewSession auto-create YO'Q; core'da bor lesson duplicate qilinmaydi (§12). Representation: section=EXTRA + itemType=**REVIEW** + roadmapItemId=NULL + lessonId + skillId (mavjud enum/nullable — SCHEMA O'ZGARMAYDI; RoadmapItem talab qilinmaydi). Core done/progress denominator EXTRA'ni O'Z ICHIGA OLMAYDI (§27/59). ReviewCandidate (1.9A) eligibility authority; DailyPlan signal/threshold qayta hisoblamaydi.
### TD-134 — Daily Review Selection Priority v1 (ACCEPTED, Phase 2.0A)
- **Decision:** Same-topic, non-core-duplicate candidate'lar orasidan deterministik lexicographic key bilan BITTA tanlanadi: (1) directTrigger true birinchi, (2) strongest reason **REPEATED_MISTAKE > REVIEW_DUE > WEAK_SKILL** (1.9A serialization order endi PLANNING policy), (3) exposure COMPLETED before IN_PROGRESS, (4) accepted Skill order (1.9A group order), (5) 1.9A Lesson hierarchy (candidate index), (6) lessonId final tie-break. Numeric urgency score YO'Q. Pure `selectReviewExtra`.
### TD-135 — Review Daily Focus Boundary (ACCEPTED, Phase 2.0A)
- **Decision:** Auto-review core Topic'dan CHIQMAYDI; manual ReviewSession start cross-topic qoladi (1.9B o'zgarmaydi); review EXTRA core DailyPlan completion'ga TA'SIR QILMAYDI; same-day mavjud plan yangi review qo'shish uchun MUTATE QILINMAYDI (§29/56/57). Stale EXTRA snapshot'da qoladi; start-da 1.9B ReviewCandidate revalidation reject qilishi mumkin (DailyPlanItem execution authority EMAS).
### TD-136 — Daily Mission Foundation v1 (ACCEPTED, Phase 2.0B)
- **Decision:** `daily-missions-v1` — deterministik Daily Mission policy foundation: append-only, evidence-backed mission completion learner-local kun bo'yicha. Mission code registry (kod bilan yopiq): **LEARN_TODAY**, **MASTERY_TEST_90** (v1 aynan IKKITA). Har biri **account-level** (per-Subject EMAS, DailyPlan item EMAS). Completion `DailyMissionCompletion` (mission_code + policy_version + local_date + timezone_snapshot NOT NULL) + `DailyMissionCompletionEvidence` (qualifying ActivityAttempt'ga Restrict FK). **RewardGrant/XP/IZL YARATILMAYDI** (2.0B'da reward bridge YO'Q — 2.0C). Policy versiyalar immutable: `daily-missions-v1`, `learn-today-mission-v1`, `mastery-test-90-mission-v1`.
### TD-137 — Mission Completion Idempotency & Provenance (ACCEPTED, Phase 2.0B)
- **Decision:** Bitta completion per `(user_id, mission_code, local_date)` — partial UNIQUE index `uq_daily_mission_completion_day` (DM-DB-01) idempotency authority. `local_date` = qualifying `ActivityAttempt.submittedAt` → `UserProfile.timezone` (IANA; DailyPlan bilan bir xil authority) local kunga aylantirilib olinadi; `timezone_snapshot` **muzlatiladi** (keyingi tz o'zgarishi tarixiy completion'ni KO'CHIRMAYDI — TD-91). `completed_at` = qualifying attempt'ning `submittedAt`'i. **First/earliest qualifying evidence wins**: reconcile eng erta `submittedAt` ASC, keyin `id` ASC oladi. Append-only (UPDATE/DELETE YO'Q). Concurrency/retry: P2002 → skip (mavjud completion authority).
### TD-138 — Mission Evaluation Boundary (ACCEPTED, Phase 2.0B)
- **Decision:** Mission baholash — `ActivityAttempt` persist bo'lgandan KEYIN avtomatik advisory hook (lesson-execution + review-session submit ikkalasida). Downstream mission yozuvi FAIL bo'lsa **ActivityAttempt ROLLBACK QILINMAYDI** (hook try/catch; reconcile keyin repair qiladi). `GET /daily-missions/me/today` read-only (hech narsa yozmaydi); `POST .../reconcile` mavjud evidence'dan yetishmagan joriy-kun completion'larni yaratadi (idempotent). Mission module FAQAT `DailyMissionCompletion` + `DailyMissionCompletionEvidence` yozadi — reward/skillState/skillMeasurement/signal/plan/roadmap/reviewSession/notification/AI YOZMAYDI. Own-user only (cross-user IDOR yo'q); raw answer/answerKey/payload leak YO'Q.
### TD-139 — LEARN_TODAY + MASTERY_TEST_90 Policy v1 (ACCEPTED, Phase 2.0B)
- **Decision:** **LEARN_TODAY** (`learn-today-mission-v1`): local kun ichida ≥1 **SUBMITTED** objective ActivityAttempt (MINI_QUESTION/PRACTICE/MASTERY_TEST — lesson yoki review); **correctness ahamiyatsiz** (noto'g'ri ham hisoblanadi); placement (AssessmentResponse — ActivityAttempt EMAS) va view-only (attempt yaratmaydi) ISTISNO. **MASTERY_TEST_90** (`mastery-test-90-mission-v1`): type=MASTERY_TEST, SUBMITTED, `deterministicScore ≥ 9000` bp (8999 → yo'q, 9000 → ha, 10000 → ha). Review attempt (reviewSessionId != null) ikkala missiyaga ham hisoblanadi. Bitta 10000 MASTERY_TEST attempt IKKALA missiyani yakunlashi mumkin (ikki alohida completion satri). Pure `qualifiesLearnToday` / `qualifiesMasteryTest90`.
### TD-140 — Daily Mission XP Reward v1 (ACCEPTED, Phase 2.0C-2)
- **Decision:** Vehicle = **`XpGrant`** (TD-45 accepted XP model; RewardGrant IZL uchun, XP uchun EMAS — Phase 2.0C gap). Policy `daily-mission-xp-reward-v1`, immutable mapping: LEARN_TODAY+learn-today-mission-v1 → **10 XP**, MASTERY_TEST_90+mastery-test-90-mission-v1 → **20 XP**, boshqa → grant YO'Q (default amount YO'Q). Authority = **DailyMissionCompletion** (raw ActivityAttempt EMAS — mission semantikasi 2.0B'da normalize qilingan). Account-level; avtomatik (mission completion → XP bridge), manual claim YO'Q. Natural daily max 30 XP (mission uniqueness bound; alohida cap YO'Q). Pure `evaluateDailyMissionXp(missionCode, missionPolicyVersion)`.
### TD-141 — XpGrant Mission Provenance (ACCEPTED, Phase 2.0C-2)
- **Decision:** `XpGrant`'ni harden: `daily_mission_completion_id` typed FK(Restrict) = load-bearing mission XP provenance (`source_refs` loose JSONB reward authority EMAS — TD-92); `policy_version_code` immutable snapshot (`daily-mission-xp-reward-v1`); `reason_code`=`DAILY_MISSION`. Idempotency: partial UNIQUE `uq_xp_grant_mission_completion` (daily_mission_completion_id) WHERE NOT NULL (XP-DB-01) — bir XP grant per completion; **policy version uniqueness'ga KIRMAYDI** (v1+v2 bitta tarixiy completion'ni ikki marta to'lamasligi uchun — completionId=entitlement identity, policyVersion=provenance only). Mission XP append-only (create only; UPDATE/DELETE YO'Q). Non-mission XpGrant satrlar (daily_mission_completion_id NULL) accepted ±/aggregate semantikani saqlaydi.
### TD-142 — XP / IZL Model Separation (ACCEPTED, Phase 2.0C-2)
- **Decision:** `XpGrant` = XP history (gamification, non-monetary); `RewardGrant` = IZL reward vehicle (financial, subscription_cycle_id NOT NULL, amount=IZL, 1:1 IZLLedgerEntry). Shared `rewardKind` YO'Q; XP→IZL conversion/ratio YO'Q; RewardGrant XP uchun MISUSE QILINMAYDI (cycle nullable/amount redefine RAD). `RewardPolicyVersion` (Int-versioned, cycle-linked, staff config) IZL-specific qoladi — XP `policy_version_code` code string ishlatadi. `DailyMissionCompletion.rewardGrants`(RewardGrantMission) IZL uchun reserved (TD-84, Phase 2.1A); `DailyMissionCompletion.xpGrants`(XpGrantMission) XP uchun — schema'da alohida ko'rinadi. Phase 2.0C-2 IZL/Subscription/Payment/wallet/ledger'ga ZERO write.
### TD-143 — XP Source-of-Truth / Balance Boundary (ACCEPTED, Phase 2.0C-2)
- **Decision:** `XpGrant` = joriy XP accounting authority; `totalXp = SUM(XpGrant.amount)` (indexed, full append-only history — correction satrlarni ham o'z ichiga oladi, faqat mission grantlar EMAS). `XpBalance` (total_xp/current_level cache) = **deferred projection** — Phase 2.0C-2'da YOZILMAYDI (create/increment/currentLevel YO'Q). XpBalance activation/rebuild + level curve = kelajakdagi XP Progression fazasi (2.0D). Ikkinchi XP authority yaratilmaydi.
### TD-144 — XP Materialization Failure Boundary (ACCEPTED, Phase 2.0C-2)
- **Decision:** Authoritative order: ActivityAttempt → DailyMissionCompletion → XpGrant. XP downstream; XpGrant create FAIL bo'lsa ActivityAttempt/DailyMissionCompletion **ROLLBACK QILINMAYDI** (bridge try/catch log). Replay (clientRequestId → mavjud completion → same grant) + `POST /api/xp/me/reconcile` (barcha tarixiy rewardable completion'lar, today-only EMAS) missing XP'ni idempotent repair qiladi. Delayed reconcile frozen DailyMissionCompletion.{missionCode, policyVersion}'dan deterministik (processing-time semantikasi YO'Q).
### TD-145 — XP Progression v1 (ACCEPTED, Phase 2.0D)
- **Decision:** `xp-progression-v1` (immutable; keyingi o'zgarish `xp-progression-v2`). Level 1'dan boshlanadi (Level 0 YO'Q). Cumulative threshold: **threshold(L) = 100·(L-1)·L/2 = 50·(L-1)·L** (L1→0, L2→100, L3→300, L4→600, L5→1000, L6→1500, L10→4500; per-level L→L+1 = 100·L). Max level YO'Q (matematik derived). `currentLevel` = max L≥1 with threshold(L) ≤ progressionXp (exact threshold darhol yangi levelga kiradi). Integer-exact (binary search — float boundary error YO'Q). Level-up NO reward/unlock/entitlement/event. Pure `computeXpProgression(totalXp)` → {totalXp, progressionXp, currentLevel, currentLevelStartXp, nextLevelXp, xpIntoLevel, xpToNextLevel, progressBp[0..9999], progressionVersion}.
### TD-146 — XpBalance Projection Authority (ACCEPTED, Phase 2.0D)
- **Decision:** `XpGrant` XP source of truth bo'lib qoladi; `XpBalance` = **rebuildable mutable projection/cache** (authoritative EMAS). `totalXp` = signed `SUM(XpGrant.amount)` mirror; `currentLevel` = faqat xp-progression-v1'dan derived; `progressionVersionCode` = projection provenance (xp-progression-v1; NULL=stale/unversioned). Repair direction **har doim XpGrant → XpBalance** (hech qachon teskari). Recompute = FULL history SUM (incremental `balance += amount` EMAS — signed correction/import/drift uchun). Per-user advisory lock (§48) concurrent recompute'ni serialize qiladi. Phase 2.0C zero-XpBalance-write boundary shu projector bilan ATAYLAB superseded.
### TD-147 — XP Correction / Level Semantics (ACCEPTED, Phase 2.0D)
- **Decision:** `totalXp` signed bo'lishi mumkin (XpGrant ± correction history sensor qilinmaydi). `progressionXp = max(totalXp, 0)` — level calculation faqat shundan; API `totalXp` haqiqiy signed authoritative total qoladi. `currentLevel` minimum 1. Accepted negative correction total'ni threshold ostiga tushirsa `currentLevel` **KAMAYISHI mumkin** (projection, `highestLevelEver` EMAS — permanent-level semantics YO'Q, v1'da highestLevelEver future scope).
### TD-148 — XP Projection Failure Boundary (ACCEPTED, Phase 2.0D)
- **Decision:** `XpGrant` authoritative; projection recompute FAIL bo'lsa XpGrant/DailyMissionCompletion/ActivityAttempt **ROLLBACK QILINMAYDI** (downstream try/catch, tryRecomputeProjection non-throwing). `GET /api/xp/me` canonical state'ni **XpGrant SUM'dan** (engine orqali) hisoblaydi — XpBalance cache'ga BLINDLY ISHONMAYDI (stale/missing bo'lsa ham to'g'ri). `POST reconcile` grant repair + projection rebuild qiladi (alohida projection-repair endpoint YO'Q).
### TD-149 — XP Progress Read v1 (ACCEPTED, Phase 2.0D)
- **Decision:** `GET /api/xp/me` **read-only** (XpBalance'ni UPSERT/mutate QILMAYDI, hidden write YO'Q); canonical progression'ni XpGrant'dan derive qiladi (cache vaqtincha stale bo'lishi mumkin). Own-user only; client `totalXp`/`currentLevel`/`progressionVersion`/thresholds YUBORA OLMAYDI. Level rewards/badges/streaks/titles/leaderboards/entitlements YO'Q; XpGrant.sourceRefs/evidence/cache internal leak YO'Q.
### TD-150 — IZL Economic Reward Authority (ACCEPTED, Phase 2.1A)
- **Decision:** `RewardGrant` = IZL reward eligibility/provenance; **`IZLLedgerEntry` = canonical IZL accounting truth** (append-only signed ledger; balance = SUM(amount)). DailyMissionCompletion normalized learning evidence — LEKIN **economic entitlement EMAS**; final entitlement faqat `RewardGrant.status=GRANTED` + mos EARN ledger credit'dan keyin mavjud. RewardGrant + IZLLedgerEntry bitta DB transaction'da **ATOMIK** yaratiladi (biri ikkinchisisiz YO'Q). `IZLWallet` YOZILMAYDI (ledger canonical, wallet deferred §48/§49). **SCHEMA O'ZGARMADI** — mavjud finance schema (RewardGrant.missionCompletionId/subscriptionCycleId/rewardPolicyVersionId, F-5 unique(userId,dedupKey), ledger.rewardGrantId @unique 1:1, ledger unique(userId,entryNo)) yetarli.
### TD-151 — Daily Mission IZL Reward v1 (ACCEPTED, Phase 2.1A)
- **Decision:** Policy config schema `izl-reward-policy/v1` (RewardPolicyVersion.config; RewardPolicyVersion.version Int DB identity'dan alohida). **MASTERY_TEST_90 + mastery-test-90-mission-v1 → 1 IZL**; **LEARN_TODAY → 0 IZL** (bitta objective attempt, noto'g'ri ham — XP uchun valid, real-value uchun juda zaif). Unknown mission code / producer version / config schema → grant YO'Q (default amount YO'Q). Strict TD-92 parser (schemaVersion, mission code, producer version, amount>0 integer, caps>=0 integer, amount<=daily cap); malformed → config error. Generic reward DSL YO'Q. Pure `evaluateDailyMissionIzl` / `parseIzlRewardPolicyConfig`.
### TD-152 — Subscription Cycle Reward Context (ACCEPTED, Phase 2.1A)
- **Decision:** IZL grant `DailyMissionCompletion.completedAt`'ni **historical covering SubscriptionCycle** (periodStart ≤ completedAt < periodEnd) bilan bog'laydi; cycle'ning **snapshotted `rewardPolicyVersionId`** = policy authority. Processing-time / current-ACTIVE policy / current cycle ISHLATILMAYDI (delayed reconcile deterministik). Cycle status ACTIVE va COMPLETED ikkalasi ham valid (historical context; §11). Covering cycle YO'Q → grant YO'Q (mission/XP valid qoladi). Bir nechta covering cycle (corruption) → integrity error, grant YO'Q (§13).
### TD-153 — IZL Cap / Anti-Farming v1 (ACCEPTED, Phase 2.1A)
- **Decision:** Daily DAILY_MISSION cap = **1 IZL** per (user, cycle, mission localDate [frozen snapshot, current tz'dan qayta hisoblanmaydi]); cycle DAILY_MISSION cap = **30 IZL** per SubscriptionCycle. Cap consumption = allaqachon **GRANTED RewardGrant** (category=DAILY_MISSION) amountlari (ActivityAttempt/completion count/XpGrant/wallet EMAS). Per-user advisory lock cap check + posting'ni serialize qiladi (§29/§69). **All-or-nothing** — cap yetmasa grant YO'Q, partial YO'Q, "pending entitlement" YO'Q. Posting-time semantics — reconcile mavjud tarixiy grantlarni revoke/reorder QILMAYDI (retroactive rebalance YO'Q §43). Anti-farming v1: mission qualification + one-completion-per-mission/day + MASTERY_TEST_90-only + exact version + dedup + caps + valid cycle (device/IP/AI fraud YO'Q).
### TD-154 — IZL Failure Boundary (ACCEPTED, Phase 2.1A)
- **Decision:** Upstream: ActivityAttempt → DailyMissionCompletion. Downstream **mustaqil branchlar**: XpGrant/XpBalance va IZL RewardGrant+ledger — biri ikkinchisini gate QILMAYDI (§37). IZL posting FAIL → ActivityAttempt/DailyMissionCompletion/XpGrant/XpBalance **ROLLBACK QILINMAYDI** (bridge try/catch). Economic posting'ning O'ZI atomik: RewardGrant+ledger birga commit yoki birga rollback (ledger create fail → RewardGrant ham rollback §39 — bu upstream evidence rollback'idan FARQLI). `POST /api/izl/me/reconcile` missing economic posting'larni idempotent repair qiladi.
### TD-155 — IZL Earning-Only Boundary v1 (ACCEPTED, Phase 2.1A)
- **Decision:** Phase 2.1A faqat **earn + read**. IZLWallet activation YO'Q (mavjud arxitektura majbur qilmaydi — ledger canonical); IZLReservation YO'Q; redemption/withdraw/cashout/spend/transfer/gift YO'Q; XP↔IZL conversion/rate YO'Q; PaymentOrder/Click/Payme YO'Q; Subscription/cycle/entitlement mutation YO'Q; cash value API YO'Q. `GET /api/izl/me` → {balanceIzl} (ledger SUM), read-only. Wallet/reservation/redemption/rate = kelajakdagi fazalar (2.1B+).
### TD-156 — IZL Wallet Projection v1 (ACCEPTED, Phase 2.1B)
- **Decision:** `izl-wallet-projection-v1`. `IZLLedgerEntry` accounting truth bo'lib qoladi; `IZLWallet` = **rebuildable mutable projection/cache** (authority EMAS). Recompute = FULL canonical history (SUM(ledger) + SUM(ACTIVE reservation), incremental `balance += amount` EMAS) per-user IZL advisory lock ostida. `projection_version_code` provenance (NULL=stale). GET canonical'ni ledger+reservation'dan derive qiladi — wallet cache'ga BLINDLY ISHONMAYDI. Repair direction har doim ledger+reservation → wallet. **Recon topildi: IZLWallet'da `reserved`/`projection_version` bilan bog'liq — `chk_wallet_balance_nonneg` + `chk_wallet_reserved_le_balance` (F-2/F-17) signed projection'ga ZID → migration 14'da DROP** (wallet endi signed canonical state'ni mirror qiladi; `chk_wallet_reserved_nonneg` saqlanadi).
### TD-157 — IZL Available Balance Semantics (ACCEPTED, Phase 2.1B)
- **Decision:** `balanceIzl = SUM(IZLLedgerEntry.amount)` (signed); `reservedIzl = SUM(ACTIVE IZLReservation.amount)` (≥0); `availableIzl = balance − reserved` (signed, **clamp QILINMAYDI** — accounting correction'dan keyin negative bo'lishi mumkin, §6/§42). Faqat ACTIVE reservation reserved'ga hisoblanadi (RELEASED terminal). Yangi reservation `amount>0 && amount <= max(available,0)` talab qiladi (negative availability'ni yangi hold CHUQURLASHTIRMAYDI §7). Pure `computeIzlBalance` / `canReserve`.
### TD-158 — IZL Reservation v1 (ACCEPTED, Phase 2.1B)
- **Decision:** **Recon: dedicated `IZLReservation` model MAVJUD EMAS edi** (faza premissasi noto'g'ri; IZLWallet.reservedAmount cache + `IZLRedemption` [redemption/payment entity, RedemptionType/RedemptionStatus/valueUzs/paymentOrder] bor). `IZLRedemption` reservation uchun QAYTA ISHLATILMADI (redemption/payment/fiat — scope EMAS §18/§60). Migration 14 **yangi `IZLReservation` jadval** yaratdi (izl-reservation-v1): temporary hold against available IZL, kelajakdagi trusted spend/redemption workflow uchun — ledger DEBIT QILMAYDI, RewardGrant EMAS, redemption EMAS. Fields: userId, amountIzl (CHECK>0), status (IzlReservationStatus ACTIVE/RELEASED), idempotencyKey (unique(userId,idempotencyKey), CHECK nonempty), purposeCode (server-owned), createdAt, releasedAt. Idempotent (same key+amount+purpose → same row; farqli → conflict). Concurrency: per-user IZL advisory lock. Immutable provenance (amount/user/purpose/idempotency o'zgarmaydi). **Learner-facing arbitrary reservation endpoint YO'Q** — internal trusted service only (§16/§88).
### TD-159 — Reservation Lifecycle Boundary v1 (ACCEPTED, Phase 2.1B)
- **Decision:** Phase 2.1B ACTIVE reservation yaratadi + ACTIVE→RELEASED (idempotent) qo'llaydi. Release ledger movement YARATMAYDI (reservation hech qachon debit qilmagan). **CONSUMED transition YO'Q** (v1'da runtime producer yo'q — consumed hold atomik SPEND ledger debit bilan bog'lanishi shart, kelajakdagi faza §29; IzlReservationStatus enum'da CONSUMED qiymati ham qo'shilmadi — kelakda ADD VALUE). Runtime DELETE YO'Q (history saqlanadi §33). Expiration scheduler YO'Q.
### TD-160 — Wallet / Reservation Failure Boundary (ACCEPTED, Phase 2.1B)
- **Decision:** `IZLLedgerEntry` + `IZLReservation` authoritative; `IZLWallet` downstream cache. Wallet projection FAIL → ledger/reservation ROLLBACK QILINMAYDI (tryRecompute non-throwing). Reservation create/release authoritative tx; wallet recompute alohida downstream (atomik EMAS). Earning (2.1A) atomik RewardGrant+ledger o'zgarmadi — wallet recompute uning authoritative tx'iga KIRMAYDI. GET canonical'ni ledger+reservation'dan derive qiladi (wallet stale bo'lsa ham to'g'ri); `POST /api/izl/me/reconcile` earning postinglar + wallet projection'ni repair qiladi (reservation statuslarni O'ZGARTIRMAYDI §45).
### TD-167 — PaymentOrder Purchase Authority v1 (ACCEPTED, Phase 2.1C-PO)
- **Decision:** `PaymentOrder` = concrete subscription purchase'ning internal authority (`subscription-purchase-order-v1`). v1 runtime purpose = **SUBSCRIPTION_PURCHASE** only (RENEWAL/upgrade/downgrade/proration deferred). Order `status=CREATED`da yaratiladi (provider execution EMAS). **Subscription/SubscriptionCycle payment'dan OLDIN yaratilmaydi** (DATA_MODEL_FINANCE l.269 — verified payment'dan keyin; to'lovgacha PaymentOrder'da). Client faqat `planId + clientRequestId` beradi; purpose/price/discount/payable/currency/provider server-derived.
### TD-168 — Provider / Order Separation (ACCEPTED, Phase 2.1C-PO)
- **Decision:** `PaymentOrder.provider` **nullable** qilindi (migration 15 DROP NOT NULL). Subscription purchase intent provider talab QILMAYDI (creation'da provider=NULL, PaymentTransaction YO'Q); `PaymentTransaction.provider` = execution authority (CLICK/PAYME). `PaymentOrder.provider` o'chirilmaydi/redefine qilinmaydi — kelajakdagi provider faza uning semantikasini (selected-provider snapshot / convenience / deprecate) ko'rib chiqadi; hozir hech qanday authority unga bog'liq emas.
### TD-169 — PaymentOrder Pricing Snapshot (ACCEPTED, Phase 2.1C-PO)
- **Decision:** Creation time T'da deterministik PlanPrice: `planId + currency=UZS + effectiveFrom ≤ T`, `effectiveFrom DESC, id DESC` — eng yangi eligible immutable price (TD-85). `planPriceId` FK + `grossAmount` snapshot purchase narxini muzlatadi (double protection); `izlDiscountAmount=0`, `payableAmount=grossAmount` (CHECK payable=gross−discount). Keyingi PlanPrice o'zgarishlari mavjud order'ni REPRICE QILMAYDI. Eligible price yo'q → PAYMENT_PLAN_PRICE_NOT_AVAILABLE (order yaratilmaydi).
### TD-170 — PaymentOrder Idempotency v1 (ACCEPTED, Phase 2.1C-PO)
- **Decision:** Create API `clientRequestId` talab qiladi; DB partial UNIQUE `uq_payment_order_client_request` (user_id, client_request_id) WHERE NOT NULL (PO-DB-01). Bir xil (user, clientRequestId, planId, purpose) replay → mavjud order qaytariladi (**REPRICE QILINMAYDI** — original snapshot g'olib); bir xil key + farqli planId/purpose → PAYMENT_ORDER_REQUEST_CONFLICT (ikkinchi order YO'Q). Concurrent identical → P2002 winner-reload → bitta order. Farqli userlar bir xil clientRequestId ishlatishi mumkin (unique per-user).
### TD-171 — Subscription Discount Ceiling Authority (ACCEPTED, Phase 2.1C-PO)
- **Decision:** Spending/discount ceiling **concrete PaymentOrder'ga** tegishli (SubscriptionCycle.reward_ceiling_* EMAS — u earning-only, TD-50). v1 max discount: `maxDiscountUzs = floor(grossAmount × 2000 / 10000)` (max 20% of concrete purchase gross; integer-safe, float YO'Q). SubscriptionCycle'ga discount ceiling field QO'SHILMAYDI. Bu rule Phase 2.1C-2 redemption'da validation authority bo'ladi (2.1C-PO'da order'ni mutate qilmaydi). **NO PaymentOrder → NO subscription-discount redemption.**
### TD-172 — Subscription Discount Redemption Intent v1 (ACCEPTED, Phase 2.1C-2)
- **Decision:** Aggregate = mavjud **`IZLRedemption`** (yangi RedemptionRequest model YO'Q). Policy `subscription-discount-redemption-v1`. Concrete authority = **PaymentOrder** (SubscriptionCycle EMAS). v1 type = **SUBSCRIPTION_DISCOUNT** only. **Reserve-only faza:** muvaffaqiyatli create → IZLRedemption RESERVED + IZLReservation ACTIVE; ledger debit/APPLIED/CONSUMED YO'Q. Client faqat `paymentOrderId + amountIzl + clientRequestId` beradi.
### TD-173 — Redemption Economic Quote v1 (ACCEPTED, Phase 2.1C-2)
- **Decision:** Ceiling base = `PaymentOrder.grossAmount`; `maxDiscountUzs = floor(gross × 2000/10000)` (BigInt integer-safe). Effective rate = **ACTIVE IzlRateVersion** (effectiveFrom ≤ now; DB one-ACTIVE) server-side resolve. `valueUzs = amountIzl × rateUzsPerIzl` (BigInt exact, **rounding YO'Q** — ikkalasi integer). `izlRateSnapshot`/`valueUzs` IZLRedemption'ga muzlatiladi; keyingi rate o'zgarish mavjud redemption'ni REPRICE QILMAYDI. Validation: `valueUzs ≤ maxDiscountUzs && ≤ grossAmount` (partial adjustment YO'Q — oshsa butun request rad). Value overflow ceiling check bilan ushlanadi (value > ceiling ≤ gross ≤ Int max). earning reward ceiling ISHLATILMAYDI.
### TD-174 — Redemption / Reservation Typed Binding (ACCEPTED, Phase 2.1C-2)
- **Decision:** `IZLRedemption.paymentOrderId` FK **Restrict** (SetNull'dan — load-bearing financial provenance; purchase authority redemption ostidan o'chirilmaydi). `IZLReservation.redemptionId` typed FK(Restrict) **UNIQUE** = 1:1 hold↔redemption provenance (RD-DB-02/03; purposeCode/idempotencyKey authority EMAS, supplemental). RESERVED redemption + ACTIVE reservation bitta transaction'da ATOMIK yaratiladi (biri ikkinchisisiz commit BO'LMAYDI). Reservation purposeCode = `SUBSCRIPTION_DISCOUNT_REDEMPTION`, idempotencyKey = `subscription-discount-redemption:<redemptionId>` (server-derived).
### TD-175 — Redemption Idempotency / Open Intent v1 (ACCEPTED, Phase 2.1C-2)
- **Decision:** Create API `clientRequestId` talab qiladi; DB partial UNIQUE `uq_izl_redemption_client_request` (user_id, client_request_id) WHERE NOT NULL (RD-DB-01). Identical replay (same user+key+order+amount) → mavjud snapshot [RE-RESOLVE/REPRICE YO'Q]; farqli order/amount → REDEMPTION_REQUEST_CONFLICT. **One OPEN SUBSCRIPTION_DISCOUNT redemption per PaymentOrder** — partial UNIQUE `uq_izl_redemption_open_per_order` (payment_order_id) WHERE type=SUBSCRIPTION_DISCOUNT AND status IN (REQUESTED,RESERVED) (RD-DB-04). RELEASED'dan keyin yangi clientRequestId bilan yangi intent mumkin (eski key → eski RELEASED qaytariladi). Concurrent → P2002 winner-reload.
### TD-176 — Redemption Release v1 (ACCEPTED, Phase 2.1C-2)
- **Decision:** `POST /api/izl/redemptions/:id/release` (own-user, no body). Bitta transaction + per-user lock: redemption RESERVED→RELEASED + linked reservation ACTIVE→RELEASED **ATOMIK** (split state taqiqlangan). Ledger movement YO'Q (reservation hech qachon debit qilmagan). Idempotent (RELEASED → terminal qaytariladi). **Release order CREATED bo'lishini talab QILMAYDI** (authority = redemption RESERVED + reservation ACTIVE + ownership) — mablag'ni bo'shatish xavfsiz.
### TD-177 — Redemption Spend Boundary (ACCEPTED, Phase 2.1C-2)
- **Decision:** Reserve/release `PaymentOrder`'ni MUTATE QILMAYDI (grossAmount/izlDiscountAmount=0/payableAmount=gross o'zgarmaydi); `PaymentOrder.izlRedemptionId` **NULL qoladi** (kelajakdagi APPLIED discount provenance uchun reserved, 2.1D). SPEND/REDEEM IZLLedgerEntry YO'Q; reservation CONSUMED YO'Q; redemption APPLIED YO'Q (IzlReservationStatus'da CONSUMED, runtime APPLIED producer YO'Q). Bular Phase 2.1D (atomik real-benefit application).
### TD-178 — Subscription Discount Commit v1 (ACCEPTED, Phase 2.1D)
- **Decision:** `subscription-discount-commit-v1`. RESERVED redemption o'z **CREATED** PaymentOrder'iga commit qilinishi mumkin: `POST /api/izl/redemptions/:id/commit-discount` (own, no body). PaymentOrder → `izlDiscountAmount = redemption.valueUzs`, `payableAmount = grossAmount − valueUzs`, `izlRedemptionId = redemption.id`; order **CREATED qoladi**, provider o'zgarmaydi. Frozen quote qayta-narxlanmaydi (current rate re-resolve YO'Q) — faqat deterministik integrity (valueUzs>0 && ≤ floor(gross×2000/10000) && ≤ gross) revalidate. Idempotent, per-user IZL lock, concurrency-safe.
### TD-179 — Reserved-vs-Spent Economic Boundary (ACCEPTED, Phase 2.1D)
- **Decision:** Discount commit **IZL SARFLAMAYDI**. Redemption RESERVED qoladi; reservation ACTIVE qoladi; IZLLedgerEntry O'ZGARMAYDI (byte-for-byte); canonical balance/reserved/available O'ZGARMAYDI (commit faqat PaymentOrder pricing'ni o'zgartiradi). "Discounted but unpaid order" hali realized subscription benefit EMAS. Spend faqat kelajakdagi verified-payment finalization'da (REDEEM ledger + reservation CONSUMED + redemption APPLIED + PaymentOrder PAID + Subscription activation).
### TD-180 — PaymentOrder IZL Redemption Binding (ACCEPTED, Phase 2.1D)
- **Decision:** `PaymentOrder.izlRedemptionId` = committed discount redemption pointer; partial UNIQUE `uq_payment_order_izl_redemption` WHERE NOT NULL (DC-DB-01 — bir redemption bitta orderni narxlaydi, discount stacking YO'Q). App invariant (§20, service tx-enforced): `PaymentOrder.izlRedemptionId = R ⟹ R.paymentOrderId = PaymentOrder.id`. Order boshqa redemption'ga point qilsa → REDEMPTION_COMMIT_CONFLICT (overwrite YO'Q). izlRedemptionId typed FK EMAS (scalar) — cross-consistency FK bilan ifodalab bo'lmaydi, service transaction enforce qiladi.
### TD-181 — Discount Commit Release v1 (ACCEPTED, Phase 2.1D) — AMENDS TD-176
- **Decision:** **Uncommitted** redemption release = mavjud 2.1C-2 xatti-harakat (order pointer NULL → order mutation YO'Q, order status'dan mustaqil). **Committed** redemption (order.izlRedemptionId = redemption.id) faqat order **CREATED** bo'lganda unwind qilinadi: bitta transaction + lock'da PaymentOrder restore (izlDiscountAmount=0, payableAmount=gross, izlRedemptionId=NULL) + reservation ACTIVE→RELEASED + redemption RESERVED→RELEASED **ATOMIK** (split state taqiqlangan); ledger movement YO'Q; idempotent. **TD-176 amended:** release endi committed pricing'ni ham unwind qiladi (avval order tegilmagani uchun release order status'dan mustaqil edi); committed release order CREATED talab qiladi (provider execution boshlansa unbind xavfli). expiresAt release'ni bloklamaydi (mablag' bo'shatish/order restore xavfsiz).
### TD-182 — Discount / Payment Sequencing Boundary (ACCEPTED, Phase 2.1D)
- **Decision:** Kelajakdagi provider execution `PaymentOrder.payableAmount` (committed, gross EMAS) bo'yicha charge qilishi shart. SPEND/APPLIED/CONSUMED verified payment'dan OLDIN sodir bo'lmaydi. Payment/cancellation/reversal semantikasi alohida keyingi faza. Commit hech qanday PaymentTransaction/PAID/Subscription ishini boshlamaydi.
### TD-183 — Payment Execution Attempt v1 (ACCEPTED, Phase 2.1E)
- **Decision:** `payment-execution-attempt-v1`. Learner o'z **CREATED** PaymentOrder'i uchun provider payment execution attempt boshlaydi: `POST /api/payments/orders/:id/initiate` (own, faqat `provider + clientRequestId`). Bitta transaction + per-user IZL lock'da **PaymentTransaction PENDING** (providerTransactionId=NULL, amount=payableAmount snapshot) yaratiladi + order **CREATED → PENDING**. Bu **payment execution INTENT, payment SUCCESS EMAS** — SUCCEEDED/PAID/IZL spend/Subscription YO'Q. Eligibility: purpose SUBSCRIPTION_PURCHASE && status CREATED && not-expired && payableAmount>0 (aks holda PAYMENT_ORDER_NOT_ELIGIBLE 409). Committed discount bo'lsa integrity revalidate (RESERVED redemption + ACTIVE reservation + payable=gross−value).
### TD-184 — Payment Provider Port Boundary (ACCEPTED, Phase 2.1E)
- **Decision:** Provider init injectable **`PAYMENT_PROVIDER_PORT`** orqali (SMS_PORT modeli). Production default = **UnavailablePaymentProviderAdapter** (real Click/Payme YO'Q → throws). Provider `initiate()` chaqiruvi **DB transaction'dan TASHQARIDA** (attempt PENDING persist bo'lgandan keyin), qaytgan `providerTransactionId` alohida qisqa transaction'da attach qilinadi (idempotent). **Ambiguous/failed provider init** (transport error) attempt'ni **PENDING qoldiradi** (id attach QILINMAYDI, order PENDING qoladi) — bir xil clientRequestId bilan retry stable txId'ga id attach qiladi. Test adapter deterministik (bir xil txId → bir xil providerTransactionId).
### TD-185 — PaymentTransaction Provider Authority (ACCEPTED, Phase 2.1E)
- **Decision:** `PaymentTransaction.provider` (CLICK/PAYME) = yagona **execution authority**. `PaymentOrder.provider` **NULL qoladi / YOZILMAYDI** (TD-168 provider-agnostik purchase authority — non-authoritative). Initiate PaymentOrder.provider'ni set QILMAYDI. Bir order uchun bir vaqtda bitta PENDING attempt (provider unga bog'langan); farqli provider bilan bir xil idempotency key → PAYMENT_ATTEMPT_REQUEST_CONFLICT (409).
### TD-186 — Payment Charge Amount Authority (ACCEPTED, Phase 2.1E)
- **Decision:** `PaymentTransaction.amount = PaymentOrder.payableAmount` snapshot (committed discount post-value; **grossAmount EMAS**). Server-derived; client amount/currency/status/providerTransactionId beролmaydi (DTO whitelist → 400). currency PaymentTransaction'da alohida ustun EMAS — `PaymentOrder.currency`'dan derive qilinadi va response'da qaytariladi. payableAmount ≤ 0 → attempt YO'Q (409).
### TD-187 — Payment Attempt Idempotency v1 (ACCEPTED, Phase 2.1E)
- **Decision:** Initiate `clientRequestId` talab qiladi; DB partial UNIQUE `uq_payment_transaction_client_request` (payment_order_id, client_request_id) WHERE NOT NULL (PT-DB-01). Identical replay (same order+key) → mavjud attempt qaytariladi (needsProviderInit = providerTransactionId hali NULL bo'lsa); farqli provider bilan bir xil key → PAYMENT_ATTEMPT_REQUEST_CONFLICT. **One PENDING attempt per order** — partial UNIQUE `uq_payment_transaction_pending` (payment_order_id) WHERE status='PENDING' (PT-DB-02); yangi key bilan PENDING order → PAYMENT_ORDER_NOT_ELIGIBLE (order endi CREATED emas). Concurrent identical → P2002 winner-reload → bitta attempt.
### TD-188 — Payment Finalization Sequencing Boundary (ACCEPTED, Phase 2.1E)
- **Decision:** Bu faza faqat execution INTENT yaratadi. **YO'Q:** verified callback/PaymentCallbackEvent, SUCCEEDED producer, PaymentOrder PAID, IZL REDEEM ledger / reservation CONSUMED / redemption APPLIED, Subscription/SubscriptionCycle activation, real provider API. Bularning barchasi Phase 2.1F (Verified Payment Finalization) — bitta atomik transaction: PaymentOrder PAID + IZL REDEEM + reservation CONSUMED + redemption APPLIED + Subscription + Cycle (F-9/F-10 invariant). PaymentTransaction faqat payments modulida yoziladi.
### TD-189 — Verified Payment Evidence v1 (ACCEPTED, Phase 2.1F)
- **Decision:** `payment-verified-evidence-v1`. Durable trusted-success marker = **`PaymentTransaction.status = SUCCEEDED`**, faqat adapter-verified provider callback + business integrity check'lardan keyin ishlab chiqiladi (boshqa producer YO'Q; client trusted success YARATOLMAYDI — learner route yo'q, §45). **SUCCESS-ONLY v1:** runtime faqat verified SUCCESS materializes; FAILED/CANCELLED/REFUNDED production transition YO'Q (enum qiymatlari qoladi, producer yo'q — definitive failure/retry/late-success/refund alohida owner-reviewed faza). "SUCCEEDED evidence ≠ finalized" (§2/§11).
### TD-190 — Provider Verification Port (ACCEPTED, Phase 2.1F)
- **Decision:** `PaymentProviderPort.verifyCallback(input)` provider-neutral — adapter signature/authentication + payload parsing + provider status mapping + identity extraction'ni EGALLAYDI; business layer Click/Payme payload semantikasini PARSE QILMAYDI. Callback envelope (`PaymentCallbackInput {provider, payload, headers?, query?}`) business uchun **opaque** (§5, raw header/secret business jadvalga yozilmaydi). Verification **DB transaction'dan TASHQARIDA** (§39). Real Click/Payme adapter + webhook route DEFERRED (§8/§46) — faqat deterministik TestPaymentProviderAdapter + throwing UnavailablePaymentProviderAdapter.
### TD-191 — Payment Verification Integrity (ACCEPTED, Phase 2.1F)
- **Decision:** Business-accepted ≠ provider-authenticated (§11). Verified event qabul qilinishidan oldin: (a) merchant PT identity (`merchantPaymentTransactionId = PaymentTransaction.id`) exact resolve qilishi shart (amount/user/order guessing YO'Q, §7; non-UUID = malformed → verification reject no-write §10; valid-but-unknown = IDENTITY_MISMATCH reject event); (b) `PaymentTransaction.provider = verified.provider`; (c) external `providerTransactionId` NULL bo'lsa attach, mavjud bo'lsa equality (boshqa id → EXTERNAL_ID_CONFLICT, overwrite YO'Q; boshqa PT egasi → conflict, @@unique(provider,provider_transaction_id)); (d) `verified.amount = PaymentTransaction.amount = PaymentOrder.payableAmount` (ikkinchi tenglik post-init corruption'ni ushlaydi §59); (e) `verified.currency = PaymentOrder.currency` (PT'da currency ustuni YO'Q — order authority). Har qanday mismatch → SUCCEEDED YO'Q, PAID YO'Q, downstream YO'Q; provider-authenticated business-rejected event REJECTED evidence sifatida yoziladi (§13).
### TD-192 — Payment Callback Idempotency v1 (ACCEPTED, Phase 2.1F)
- **Decision:** `@@unique(provider, provider_event_id)` (F-19) = callback dedup authority. Oqim: adapter verify (DB tx tashqarisida) → qisqa DB tx (payment-scoped lock `pg_advisory_xact_lock('pay', payment_order_id)`, IZL lock EMAS §40) → callback insert. Accepted callback insert + PENDING→SUCCEEDED transition **ATOMIK** (§14 — accepted-but-still-PENDING yoki SUCCEEDED-without-provenance taqiqlangan). Exact (provider, providerEventId) replay → mavjud natija qayta quriladi, mutation takrorlanmaydi (§29/§65). Bir successful provider transaction uchun bir necha DISTINCT event id kelsa (matching provider id/amount/currency) → terminal DUPLICATE no-op evidence (confirmedAt/transaction o'zgarmaydi §28/§66); farqli data → SUCCESS_DATA_CONFLICT (§30, accepted history mutate qilinmaydi).
### TD-193 — One Successful Payment Authority (ACCEPTED, Phase 2.1F)
- **Decision:** Bir PaymentOrder uchun ko'pi bilan **BITTA SUCCEEDED** PaymentTransaction — partial UNIQUE `uq_payment_transaction_succeeded` (payment_order_id) WHERE status='SUCCEEDED' (PV-DB-01, load-bearing financial invariant). Historical multiple attempt qoladi. Ikkinchi success → financial integrity conflict (SUCCESS_CONFLICT proactive check + PV-DB-01 DB backstop P2002); timestamp bo'yicha silent winner tanlash YO'Q; ikkinchisini silent FAILED qilish YO'Q (serialized/DB-constrained first accepted success g'olib). confirmedAt = trusted `verified.confirmedAt` (server now emas, client input emas, §20/§21).
### TD-194 — Verified-vs-Finalized Payment Boundary (ACCEPTED, Phase 2.1F)
- **Decision:** `PaymentTransaction.status = SUCCEEDED` PaymentOrder PAID'ni ANGLATMAYDI. Order **PENDING qoladi** (§31 — callback path PaymentOrder'ni umuman yozmaydi). IZL reserved/unspent qoladi (ledger/reservation/redemption o'zgarmaydi §35/§72); Subscription/SubscriptionCycle YARATILMAYDI (§73). Internal economic finalization alohida faza (Phase 2.1G): bitta atomik tx — PaymentOrder PAID + IZL REDEEM ledger −amountIzl + reservation CONSUMED + redemption APPLIED + Subscription + SubscriptionCycle (reward-policy selection + earning-ceiling snapshot cycle creation'da, F-9 frontend hech activate qilmaydi / F-10 one payment=one cycle). Trusted evidence yo'qolmasligi uchun order PENDING recoverable saqlanadi (§32).
### TD-195 — Subscription Billing Period Authority v1 (ACCEPTED, Phase 2.1G-D)
- **Decision:** Commercial billing duration = **immutable `PlanPrice.billingPeriodMonths`** (amount + duration bir tarixiy offerga tegishli; PaymentOrder.planPriceId muzlatadi). Existing v1 PlanPrice rows = **1 calendar month** (migration 20 backfill, NOT NULL, CHECK > 0 FP-DB-01). Future price/duration change = yangi PlanPrice version (in-place mutation YO'Q). Future cycle `periodStart = PaymentTransaction.confirmedAt` (§35); `periodEnd = addCalendarMonths(periodStart, billingPeriodMonths)` end-of-month clamping bilan (Jan 31 +1 → Feb 28/29). Pure `addCalendarMonths` helper + unit testlar (deterministik infra); cycle creation YO'Q (2.1G).
### TD-196 — Paid Order Finalization Time Boundary (ACCEPTED, Phase 2.1G-D)
- **Decision:** `PaymentOrder.expiresAt` faqat NEW payment initiation'ni gate qiladi (2.1E). Trusted `PaymentTransaction.SUCCEEDED` mavjud bo'lgach order expiry internal finalization'ni **BLOKLAMAYDI** — to'langan pul finalizable/recoverable qoladi (§25). Payment/time authority = unique SUCCEEDED PT.confirmedAt; `PaymentOrder.paidAt` ustuni QO'SHILMAYDI (§31).
### TD-197 — Cycle Reward Economic Basis v1 (ACCEPTED, Phase 2.1G-D)
- **Decision:** `rewardBasisUzs = PaymentOrder.payableAmount` (net/actually-paid, **grossAmount EMAS** — IZL-discounted qiymat fiat'da to'lanmagan, qo'shimcha earning capacity yaratmaydi). `rewardCeilingUzs = floor(rewardBasisUzs × 2000/10000)` (20%, integer-safe). `rewardCeilingIzl = floor(rewardCeilingUzs / izlRateSnapshot)` (rate = ACTIVE IzlRateVersion.rateUzsPerIzl; ceilingUzs < rate → 0; rounding-up YO'Q §8). Pure `rewardCeilingUzs`/`rewardCeilingIzl` helperlar + testlar; cycle creation YO'Q.
### TD-198 — Reward Configuration Failure Boundary (ACCEPTED, Phase 2.1G-D)
- **Decision:** Reward/gamification config missing/malformed bo'lsa verified paid subscription'ni **BLOKLAMAYDI**. Usable policy+rate (status ACTIVE, effectiveFrom ≤ periodStart, parse OK, rate>0) → **reward-ENABLED** cycle (policy+rate snapshot). Aks holda → **reward-DISABLED** cycle: `rewardPolicyVersionId=NULL, izlRateSnapshot=NULL, rewardCeilingUzs=0, rewardCeilingIzl=0, earnedIzl=0`, rewardBasisUzs=payable audit uchun. Fake policy/fake rate/zero-rate-pretending YO'Q. Coherence CHECK (FP-DB-02/03): enabled ⟺ policy+rate present (rate>0); disabled ⟺ ikkalasi NULL + zero ceilings. Cycle reward fields nullable qilindi (migration 20).
### TD-199 — Cycle Economic Reward Ceiling Enforcement (ACCEPTED, Phase 2.1G-D)
- **Decision:** 2.1A earning effective cycle cap = `min(policyDefinedCycleCap, SubscriptionCycle.rewardCeilingIzl)` (cycle economic ceiling endi load-bearing). Authoritative consumption = SUM(GRANTED RewardGrant per cycle) per-user IZL lock ostida (§16/§17); `earnedIzl` NON-authoritative qoladi (incremental += YO'Q, future projection §19). Reward-disabled cycle (policy NULL) → grant YO'Q, ledger YO'Q, exception YO'Q (mission/XP branch rollback QILINMAYDI, §14). Daily cap (1/day) + policy cycle cap (30) mavjud; economic ceiling qo'shimcha upper bound.
### TD-200 — Subscription Purchase Activation Boundary v1 (ACCEPTED, Phase 2.1G-D)
- **Decision:** Future SUBSCRIPTION_PURCHASE finalization: (A) nonterminal subscription YO'Q → yangi ACTIVE Subscription; (B) EXPIRED nonterminal → **SHU Subscription episode'ni reactivate** (EXPIRED→ACTIVE) + keyingi cycle (F-14 ux_nonterminal_subscription bir episode kafolatlaydi; startedAt tarixiy episode timestamp saqlanadi); (C) ACTIVE Subscription → **silent renewal YO'Q** — recoverable subscription-conflict (PT SUCCEEDED qoladi, order PENDING qoladi, partial effect YO'Q); (D) faqat CANCELLED tarix → yangi Subscription episode. SUBSCRIPTION_RENEWAL alohida future scope (§26/§27).
### TD-201 — IZL Reservation Consumption Provenance (ACCEPTED, Phase 2.1G-D)
- **Decision:** `IzlReservationStatus + CONSUMED` (migration 20 enum ADD VALUE). CONSUMED = ACTIVE hold REDEEM ledger debit bilan fulfilled; RELEASED (freed, no spend) dan DISTINCT (audit §34). Reserved SUM faqat ACTIVE (RELEASED + CONSUMED ikkalasi ham exclude — mavjud query o'zgarmadi, §22). **One REDEEM per redemption** DB-enforced: partial UNIQUE `uq_izl_ledger_redeem_per_redemption` (redemption_id) WHERE redemption_id IS NOT NULL AND entry_type='REDEEM' (FP-DB-04; global redemption_id UNIQUE EMAS — future REVERSAL/ADJUSTMENT provenance saqlanadi §23). CONSUMED/REDEEM runtime producer YO'Q — Phase 2.1G finalizer birinchi (§21/§24).
### TD-202 — Verified Payment Finalization Lock Order (ACCEPTED, Phase 2.1G-D)
- **Decision:** Global multi-lock order (2+ lock kerak bo'lganda authoritative): **`sub(userId) → pay(paymentOrderId) → izl(userId)`** (izl faqat discounted). Finalizer verification transaction commit'idan KEYIN alohida tx'da ishlaydi (callback tx ichida nest QILINMAYDI, §55). Finalization tx ichida provider/external call YO'Q. Bitta lock oladigan mavjud oqimlar (reward/reservation/redemption=izl; callback=pay) o'zgarmaydi — hech biri reverse multi-lock order olmaydi. `sub(userId)` helper qo'shilishi mumkin lekin 2.1G-D subscription mutate qilmaydi (§33).
### TD-203 — Held IZL Finalization Authority (ACCEPTED, Phase 2.1G-D)
- **Decision:** Discounted finalization'da ACTIVE reservation = prior financial authorization. Finalization `amount ≤ current availableIzl` ni QAYTA TEKSHIRMAYDI. Keyingi ledger correction'lar `balance < reserved` qilgan bo'lsa ham held amount CONSUMED bo'ladi; REDEEM debit signed negative ledger balance qilishi mumkin (honest signed exposure). Already-paid purchase current balance harakati sabab strand QILINMAYDI / auto-release/resize/reject YO'Q (§28/§40).
### TD-204 — Payment Finalization Provenance v1 (ACCEPTED, Phase 2.1G-D)
- **Decision:** `PaymentTransaction.provider` = provider authority; `PaymentOrder.provider` non-authoritative/nullable qoladi (finalization'da snapshot QILINMAYDI, §30/§44). Payment timestamp authority = unique SUCCEEDED PT.confirmedAt (PaymentOrder.paidAt YO'Q). Future redemption `resolvedAt = confirmedAt` (§29/§42). PaymentOrder→SUCCEEDED transactions (PV-DB-01) + SubscriptionCycle.paymentOrderId UNIQUE = audit chain (redundant successfulPaymentTransactionId pointer QO'SHILMAYDI, §45).
### TD-205 — Verified Payment Economic Finalization v1 (ACCEPTED, Phase 2.1G)
- **Decision:** `verified-payment-finalization-v1`. Authority = persisted `PaymentTransaction.SUCCEEDED` (client trusted qiymat BEROLMAYDI — finalizer faqat internal `paymentTransactionId` oladi, qolgani DB'dan; learner 'mark paid'/'activate' route YO'Q §3/§67/§101). Provider call YO'Q (§69 — trusted evidence consume qiladi, Click/Payme CHAQIRMAYDI). Bitta atomik replay-safe DB transaction: PaymentOrder PENDING→PAID + Subscription + SubscriptionCycle + SubscriptionCycleEntitlement snapshot (+ discounted: IZL REDEEM + reservation CONSUMED + redemption APPLIED). PaymentOrder PAID faqat to'liq business finalization bilan commit bo'ladi (§26/§43/§44). PaymentTransaction/PaymentCallbackEvent finalizer tomonidan MUTATE QILINMAYDI (§70/§71). Post-migration-20 schema yetarli — **YANGI MIGRATION YO'Q (count 20)**.
### TD-206 — Subscription Activation Finalization v1 (ACCEPTED, Phase 2.1G)
- **Decision:** SUBSCRIPTION_PURCHASE only (renewal deferred §5/§73). Activation cases (F-14 ux_nonterminal_subscription serialize under sub-lock): nonterminal YO'Q → yangi ACTIVE Subscription (startedAt=confirmedAt); EXPIRED nonterminal → **SHU episode reactivate** (EXPIRED→ACTIVE, planId=yangi order plan §20, **startedAt O'ZGARMAYDI**, currentCycleId=yangi cycle, eski cycle'lar tegilmaydi §24); ACTIVE → **SubscriptionPurchaseActiveConflictError** (recoverable, silent renewal/extension YO'Q, partial effect YO'Q, PT SUCCEEDED + order PENDING qoladi §21); faqat CANCELLED tarix → yangi episode (CANCELLED resurrect QILINMAYDI §22).
### TD-207 — Subscription Cycle Snapshot v1 (ACCEPTED, Phase 2.1G)
- **Decision:** `periodStart = PaymentTransaction.confirmedAt` (finalizer now EMAS, §16 — delayed recovery access qisqartirmaydi); `periodEnd = addCalendarMonths(periodStart, PlanPrice.billingPeriodMonths)` (frozen order.planPriceId, current price re-lookup YO'Q §15/§17). Commercial snapshotlar FAQAT PaymentOrder'dan: planId/planPriceId/grossPriceUzs(=grossAmount)/discountUzs(=izlDiscountAmount)/paidAmountUzs(=payableAmount)/rewardBasisUzs(=payableAmount), earnedIzl=0 (§26, reprice YO'Q). sequenceNo: yangi sub=1, EXPIRED reactivation=MAX(existing)+1 sub-lock ostida (§23). One order → one cycle (paymentOrderId UNIQUE). PlanEntitlement → SubscriptionCycleEntitlement immutable snapshot (deterministik, §31/§32); UsageCounter YO'Q (deferred). Reward-enabled/disabled cycle 2.1G-D contract bo'yicha (TD-198/199).
### TD-208 — Paid Discount IZL Consumption v1 (ACCEPTED, Phase 2.1G)
- **Decision:** Committed discount (order.izlRedemptionId != NULL) FAQAT paid finalization'da consume qilinadi. Provenance exact (§35): redemption RESERVED + type SUBSCRIPTION_DISCOUNT + paymentOrderId=order + valueUzs=izlDiscountAmount, reservation ACTIVE + redemptionId=redemption + amount match (loose matching YO'Q). Atomik: IZLLedgerEntry REDEEM `amount=-redemption.amountIzl` (entryNo=MAX+1, balanceAfter=SUM−amount, **signed negative MUMKIN** §37/§42, FP-DB-04 one-REDEEM) + reservation ACTIVE→**CONSUMED** (RELEASED emas §39) + redemption RESERVED→**APPLIED** resolvedAt=confirmedAt (§40). **Available-balance recheck YO'Q** (ACTIVE reservation=prior authorization; balance<reserved bo'lsa ham consume, already-paid strand qilinmaydi §36). Available invariant: reserve→consume da available O'ZGARMAYDI (§41). izlRedemptionId NULL → izlDiscountAmount=0 && payable=gross talab (§34); discount>0 && pointer NULL → integrity error (§33).
### TD-209 — Finalization Idempotency / Recovery (ACCEPTED, Phase 2.1G)
- **Decision:** Replay authorities = PaymentOrder.status=PAID + SubscriptionCycle.paymentOrderId UNIQUE + FP-DB-04 one-REDEEM. Order PAID → reconstruct+validate (yangi write YO'Q; discounted uchun REDEEM/CONSUMED/APPLIED provenance mavjudligini tekshiradi; inconsistent PAID → integrity error, silent repair YO'Q §7). Idempotent + concurrency-safe (sub→pay→izl lock, concurrent same-order → 1 effect §50; two paid orders same user → 1 activate, ikkinchi ACTIVE conflict §51/§96). Finalizer failure → butun tx rollback (partial activation YO'Q §62/§98); PT SUCCEEDED + order PENDING recoverable qoladi. Recoverable backlog = SUCCEEDED PT + PENDING order (§68; scheduler YO'Q). IZLWallet downstream cache (finalization tx'dan tashqarida tryRecompute non-throwing §60, projection fail PAID/cycle/REDEEM'ni rollback QILMAYDI).
### TD-210 — Post-Verification Finalization Bridge (ACCEPTED, Phase 2.1G)
- **Decision:** 2.1F callback verification transaction COMMIT bo'lgach (§63 — nested EMAS), PaymentCallbackService trusted SUCCEEDED transaction uchun finalizer'ni **best-effort alohida tx'da** chaqiradi (`tryFinalizeAfterVerification`, non-throwing). Bridge failure verified-payment evidence'ni (PT SUCCEEDED / PaymentCallbackEvent) rollback QILMAYDI / o'chirmaydi (§64) — recoverable SUCCEEDED+PENDING qoladi. Matching callback replay (DUPLICATE, PT allaqachon SUCCEEDED lekin order PENDING) stuck finalization'ni QAYTA urinishi mumkin (§65), provider re-verification YO'Q. Provider-facing callback success = provider evidence accepted (internal finalization fail bo'lsa ACCEPTED→failure qilinmaydi §66). Learner finalization authority YO'Q.
### TD-211 — Verified Payment Finalization Recovery v1 (ACCEPTED, Phase 2.1H)
- **Decision:** `verified-payment-finalization-recovery-v1`. Backlog authority = **SUCCEEDED PaymentTransaction + PENDING PaymentOrder** (PV-DB-01 → har PENDING order bitta trusted PT'ga resolve bo'ladi; CallbackEvent/providerTransactionId/IZL'dan inference YO'Q). Recovery mavjud **yagona** `PaymentFinalizationService.finalizeVerifiedPayment`'ni qayta ishlatadi — ikkinchi finalization implementation YO'Q (§3); recovery moduli hech qanday business mutation OWN QILMAYDI (PaymentOrder PAID/Subscription/Cycle/REDEEM/CONSUMED/APPLIED writer yo'q, grep-verified). Provider call/recharge YO'Q — provider success 2.1F'da hal qilingan, qayta verify qilinmaydi. Verified evidence immutable (PT SUCCEEDED/confirmedAt/provider/amount o'zgarmaydi). YANGI MIGRATION YO'Q (count 20).
### TD-212 — Payment Finalization Reconciliation v1 (ACCEPTED, Phase 2.1H)
- **Decision:** Bounded (limit default 50, max 200), deterministik **oldest-verified-first** (confirmedAt ASC, id ASC) SERIAL processing (§7 — worker pool/Promise.all yo'q, low volume). Item isolation: har item alohida finalizer transaction'ida (outer transaction spanning items YO'Q); bitta item fail/blocked bo'lsa keyingilarni to'xtatMAYDI (§30/§31 no starvation). Outcomes: **FINALIZED** (PENDING→PAID shu run'da, replay=false) / **ALREADY_FINALIZED** (finalizer replay path already-PAID validate qildi, replay=true) / **BLOCKED** (deterministik domain condition, masalan SUBSCRIPTION_PURCHASE_ACTIVE_CONFLICT) / **FAILED** (transient/integrity, INTERNAL_FINALIZATION_ERROR). Stack trace/SQL/secret leak YO'Q; corrupted confirmedAt=NULL SUCCEEDED finalize qilinmaydi (FAILED).
### TD-213 — Finalization Operational Access Boundary (ACCEPTED, Phase 2.1H)
- **Decision:** Reconciliation internal/admin only. Yangi permissionlar `payments.finalization.read` (GET backlog read-only) + `payments.finalization.reconcile` (POST bounded reconcile) — global AuthGuard + PermissionsGuard (ADMIN role-name bypass YO'Q, effective permission tekshiriladi). Learner permission'siz → 403; unauth → 401. Learner mark-paid/activate/reconcile authority YO'Q. `/api/admin/payments/...` (loyihada birinchi admin route). **Permission matrix ops-managed** (bootstrap grant YO'Q, mavjud "matrix OPEN" pozitsiyasi bilan mos; production admin role'ga RolePermission grant qiladi — data operation, migration EMAS). Backlog read-only + response'da provider/callback/OTP/SQL/policy secret YO'Q (§19/§62).
### TD-214 — Finalization Recovery Concurrency (ACCEPTED, Phase 2.1H)
- **Decision:** 2.1G callback bridge va operational reconciliation ikki recovery path — bir xil idempotent finalizer orqali converge bo'ladi. Mavjud finalizer locklari (sub→pay→izl) + DB uniqueлar (cycle-per-order, one-REDEEM, PV-DB-01) = yagona serialization/idempotency authority. Bir necha reconciler overlap bo'lishi mumkin (global reconciliation mutex YO'Q). Backlog SELECT'dan keyin PAID bo'lgan item → ALREADY_FINALIZED (false FAILED emas, §23). Duplicate cycle/REDEEM/subscription effect YO'Q.
### TD-215 — Finalization Retry Failure Boundary (ACCEPTED, Phase 2.1H)
- **Decision:** Domain-blocked paid purchase (ACTIVE-subscription conflict) → **BLOCKED**, FAILED emas: PT SUCCEEDED/order PENDING qoladi, redemption RESERVED/reservation ACTIVE, PaymentTransaction/PaymentOrder FAILED qilinmaydi, refund/release/cancel/IZL-consume YO'Q; repeated reconcile BLOCKED qoladi (mutation yo'q). Transient finalizer failure → **FAILED**, lekin 2.1G atomik rollback tufayli SUCCEEDED+PENDING recoverable qoladi; provider call yo'q, avtomatik refund yo'q. Bitta blocked/failed item boshqa backlog itemlarni starve QILMAYDI. Scheduler/cron/queue/worker YO'Q (operational primitive birinchi; automation future owner decision).
### TD-216 — Verified Non-Success Payment Evidence v1 (ACCEPTED, Phase 2.1I)
- **Decision:** `payment-verified-non-success-v1`. Trusted (adapter-verified) DEFINITIVE provider terminal evidence PaymentTransaction'ni **PENDING → FAILED / CANCELLED** transition qilishi mumkin. Faqat provider truth — **PaymentOrder PENDING QOLADI** (§10/§27), order reopen/retry/IZL release/refund/finalizer YO'Q (§13/§29/§31 — Phase 2.1J). Ambiguous/untrusted evidence hech qachon terminalize qilmaydi (PT PENDING qoladi). Evidence = order retry policy'dan alohida (success arxitekturasidagi evidence/finalization ajratilishiga simmetrik). Discounted order: redemption RESERVED/reservation ACTIVE/ledger o'zgarmaydi (§12/§28). YANGI MIGRATION YO'Q (PaymentTransactionStatus FAILED/CANCELLED + PaymentCallbackEvent.result free String + F-19 + pay lock yetarli; count 20).
### TD-217 — Definitive Payment Failure Contract (ACCEPTED, Phase 2.1I)
- **Decision:** **FAILED** = provider transaction endi normal semantikada muvaffaqiyatli bo'la olmaydi (definitive terminal). Adapter bu kafolatni bera olmasa FAILED normalize QILMAYDI — PT PENDING qoladi (§4/§5/§24). Ambiguous transport/timeout/no-response/unknown/processing/browser-abandonment hech qachon FAILED EMAS (2.1E ambiguous-init printsipiga mos). Provider-side definitive expiry `status=FAILED + terminal=true + reasonCode=PROVIDER_EXPIRED` ga normalize bo'ladi (PT EXPIRED enum QO'SHILMAYDI v1'da). Normalized status canonical union SUCCEEDED/FAILED/CANCELLED (provider-specific status business'ga leak qilmaydi).
### TD-218 — Provider-Confirmed Cancellation Contract (ACCEPTED, Phase 2.1I)
- **Decision:** **CANCELLED** = trusted provider-confirmed terminal cancellation. Learner/browser abandonment / local UX Cancel CANCELLED authority EMAS (§9) — faqat provider evidence. Purchase-order cancellation (PaymentOrder CANCELLED) alohida future operation, attempt-level PT CANCELLED'dan farqli (§61).
### TD-219 — Payment Terminal State Immutability (ACCEPTED, Phase 2.1I)
- **Decision:** SUCCEEDED / FAILED / CANCELLED = immutable accepted terminal truth. Taqiqlangan transitionlar: FAILED→SUCCEEDED, CANCELLED→SUCCEEDED, SUCCEEDED→FAILED, SUCCEEDED→CANCELLED, FAILED→CANCELLED, CANCELLED→FAILED (arrival-order accepted history'ni qayta yozMAYDI). Contradictory later terminal evidence → **TERMINAL_STATUS_CONFLICT** callback evidence (financial-integrity incident, operator review uchun), PT mutation YO'Q, PaymentOrder/Subscription mutation YO'Q, finalizer chaqirilMAYDI. Matching same-terminal distinct event → DUPLICATE terminal no-op (§20).
### TD-220 — Non-Success Callback Integrity v1 (ACCEPTED, Phase 2.1I)
- **Decision:** F-19 (provider, providerEventId) callback dedup reused. Merchant PT identity (merchantPaymentTransactionId=PaymentTransaction.id) exact resolve (guessing yo'q); provider identity match; external providerTransactionId attach-if-null/equality/no-overwrite (§11/§12/§13). **Amount/currency asimmetriyasi (§14):** SUCCESS exact amount/currency equality talab qiladi; non-success event amount/currency BERMASLIGI mumkin (provider contract'da mavjud emas/non-applicable), berilsa validate qilinadi (mismatch → reject), berilmasa OK. Merchant/provider/external identity har doim shart. Callback event + PT transition ATOMIK (§17). Invalid verification (2.1F security) hech qanday write QILMAYDI; verified non-success invalid'dan ajratilgan (§10 — "unsupported verified status" generic handling'ga tushmaydi).
### TD-221 — Failure / Retry Separation (ACCEPTED, Phase 2.1I)
- **Decision:** 2.1I FAQAT provider truth yozadi (PT FAILED/CANCELLED + callback evidence). Order reopen (PENDING→CREATED) QILMAYDI, IZL release QILMAYDI, yangi attempt YARATMAYDI, finalizer chaqirMAYDI. Phase 2.1J retry eligibility/reopen'ni egallaydi (definitively terminal PT + PENDING order + no PENDING/SUCCEEDED PT → CREATED, pricing/committed-discount/RESERVED/ACTIVE saqlab). Failure timestamp (failedAt/cancelledAt) va failure reason column v1'da QO'SHILMAYDI — audit authority = PaymentCallbackEvent.receivedAt/processedAt/result + PT status (§25/§26); confirmedAt success-only qoladi.
### TD-222 — Payment Order Reopen v1 (ACCEPTED, Phase 2.1J)
- **Decision:** `payment-order-reopen-retry-v1`. Definitively terminal (FAILED/CANCELLED) PaymentTransaction o'z **PENDING** PaymentOrder'ini qayta retryable qilishi mumkin — retryability = **PaymentOrder PENDING → CREATED** (yagona yoziladigan field, §3/§19). Reopen internal/server-owned — learner reopen route YO'Q (§4/§61, faqat internal paymentTransactionId authority; client status/provider/failure/order-state BEROLMAYDI). Order expiry reopen'ni BLOKLAMAYDI (§18) — mavjud initiate expiry rule yangi attempt'ni rad qiladi, CREATED esa 2.1D committed-discount release'ni qayta yoqadi. `pay(order)` lock ostida bitta short tx (izl/sub lock yo'q — faqat order.status o'zgaradi). Terminal PT MUTATE QILINMAYDI (§4/§36). YANGI MIGRATION YO'Q (count 20).
### TD-223 — Payment Retry Eligibility v1 (ACCEPTED, Phase 2.1J)
- **Decision:** Reopen preconditions (lock ostida reload): target PT FAILED/CANCELLED + **NO PENDING PT for order** (§10 — live charge path bo'lsa reopen yo'q, duplicate-charge himoyasi) + **NO SUCCEEDED PT for order** (§11 — SUCCEEDED bo'lsa 2.1G/2.1H finalization territory). PAID order → hech qachon reopen (§12/ALREADY_PAID). Outcomes: REOPENED / ALREADY_REOPENED (CREATED + no live/success, idempotent) / RETRY_ALREADY_IN_PROGRESS (PENDING PT mavjud) / PAYMENT_SUCCESS_PENDING_FINALIZATION / ALREADY_PAID / NOT_REOPENABLE (PT terminal emas / order state / integrity). CREATED+PENDING PT → integrity error (silent repair yo'q §14). Terminal historical PT'lar yolg'iz reopen'ni bloklamaydi (§51).
### TD-224 — Payment Retry Attempt Identity v1 (ACCEPTED, Phase 2.1J)
- **Decision:** Reopen'dan keyin yangi attempt = mavjud **initiate flow** (yangi retry implementation YO'Q, §29). Fresh retry yangi clientRequestId talab qiladi → yangi PT (§30/§76); eski clientRequestId replay eski terminal PT'ni qaytaradi (resolveOrCreateAttempt prior-check eligibility'dan oldin, PT-DB-01, §31/§77 — yangi PT yo'q); eski key + farqli provider → PaymentAttemptRequestConflictError (§32). Retry same yoki different provider tanlashi mumkin (PaymentTransaction.provider per-attempt authority, PaymentOrder.provider non-authoritative/NULL §33/§78). PT-DB-02 (one PENDING/order) + PV-DB-01 (one SUCCEEDED/order) o'zgarmaydi.
### TD-225 — Discount Hold Across Payment Retry (ACCEPTED, Phase 2.1J)
- **Decision:** Attempt failure/cancellation committed discount'ni AVTOMATIK RELEASE QILMAYDI: reopen'da redemption RESERVED / reservation ACTIVE / ledger o'zgarmaydi, order pricing (gross/discount/payable/pointer) frozen (§8/§9/§19/§20/§79). Sabab: FAILED/CANCELLED bitta provider attempt'ni tugatdi, purchase intent'ni EMAS — retry aynan shu frozen discounted payable bo'yicha darhol boshlanishi mumkin (auto-release snapshot'ni buzardi §21). Reopen'dan keyin order CREATED bo'lgani uchun mavjud 2.1D explicit release (POST /api/izl/redemptions/:id/release) discount'ni unwind qila oladi (2.1J-specific release implementation YO'Q, §22/§80).
### TD-226 — Post-Non-Success Reopen Bridge (ACCEPTED, Phase 2.1J)
- **Decision:** 2.1I terminal callback transaction COMMIT bo'lgach (§24 — nested EMAS), PaymentCallbackService accepted/matching terminal (FAILED/CANCELLED) uchun `tryReopenAfterTerminal`'ni **best-effort alohida tx'da** chaqiradi (non-throwing, success-path finalization bridge'iga simmetrik). Reopen failure accepted provider evidence'ni (PaymentCallbackEvent / PT FAILED/CANCELLED) rollback QILMAYDI / o'zgartirmaydi (§25/§26 — result failure'ga qayta yozilmaydi, terminal PT + PENDING order recoverable qoladi). Matching callback replay (DUPLICATE, PT allaqachon terminal, order PENDING) stuck reopen'ni QAYTA urinishi mumkin (§27/§72), ikkinchi PT mutation yo'q. TERMINAL_STATUS_CONFLICT / identity/provider mismatch reopen'ni chaqirMAYDI (§28).
### TD-227 — Stale Terminal Retry Protection (ACCEPTED, Phase 2.1J)
- **Decision:** Eski terminal callback/reopen: (a) yangi PENDING PT mavjud bo'lsa reopen QILMAYDI (§48/§73 — order CREATED + active charge path duplicate-charge yaratardi); (b) SUCCEEDED evidence mavjud bo'lsa reopen QILMAYDI (§49/§74); (c) PAID order reopen QILMAYDI (§50/§75, subscription/cycle tegilmaydi). Retry authority = **order-wide PENDING/SUCCEEDED PT yo'qligi**, "qaysi terminal attempt oxirgi edi" EMAS — redundant `latestAttemptId` pointer QO'SHILMAYDI (§51/§52, relational authority yetarli: PT-DB-02 live attempt, PV-DB-01 paid evidence). Concurrent reopen `pay(order)` lock bilan serialize (one REOPENED, one ALREADY_REOPENED §46/§83).
### TD-228 — Terminal Payment Reopen Recovery v1 (ACCEPTED, Phase 2.1K)
- **Decision:** `payment-order-reopen-recovery-v1`. Operational stuck state = terminal (FAILED/CANCELLED) PaymentTransaction + hali **PENDING** PaymentOrder (post-callback reopen bridge fail bo'lgan + callback replay kelmagan). Recovery mavjud **yagona** `PaymentOrderReopenService.reopenAfterTerminalAttempt`'ni qayta ishlatadi — ikkinchi PENDING→CREATED writer YO'Q (§41/§60); recovery moduli hech qanday mutation OWN QILMAYDI (faqat read + reopen delegate). Provider re-verification YO'Q (§17 — FAILED/CANCELLED allaqachon trusted 2.1I evidence, provider qayta so'ralmaydi), provider initiate YO'Q (§18 — user keyin initiate qiladi, silent re-charge yo'q). Actionable backlog = order PENDING + FAILED/CANCELLED PT bor + PENDING PT yo'q + SUCCEEDED PT yo'q; eligibility `pay(order)` lock ostida reopen service tomonidan qayta validate qilinadi (SELECT mutation authority EMAS §4). YANGI MIGRATION YO'Q (count 20).
### TD-229 — Reopen Reconciliation v1 (ACCEPTED, Phase 2.1K)
- **Decision:** Bounded (limit default 50, max 200 — 2.1H bilan bir xil), deterministik oldest-first (order.createdAt ASC, id ASC) SERIAL processing (§9, worker pool yo'q, outer transaction yo'q). Item isolation: har item mustaqil, bitta blocked/failed keyingilarni to'xtatMAYDI (§55). Outcomes = **2.1J reopen outcome model reuse** (REOPENED / ALREADY_REOPENED / RETRY_ALREADY_IN_PROGRESS / PAYMENT_SUCCESS_PENDING_FINALIZATION / ALREADY_PAID / NOT_REOPENABLE) + unexpected error → FAILED (INTERNAL_REOPEN_ERROR); stack/SQL/secret leak yo'q. Har item deterministik terminal PT (eng oxirgi FAILED/CANCELLED by createdAt DESC) reopen-service input sifatida oladi — retry safety order-wide (latest lifecycle authority EMAS §6).
### TD-230 — Reopen Recovery Access Boundary (ACCEPTED, Phase 2.1K)
- **Decision:** Reconciliation internal/admin only. Yangi permissionlar **payments.reopen.read** (GET backlog) + **payments.reopen.reconcile** (POST) — finalization permissionlaridan ATOMIY ajratilgan (§25, payments.finalization.reconcile reopen'ni AVTORIZE QILMAYDI, cross-authority yo'q). Global AuthGuard + PermissionsGuard (ADMIN role-name bypass yo'q); learner → 403; unauth → 401. Learner reopen/reconcile/force-status authority YO'Q; learner faqat CREATED bo'lgach mavjud initiate orqali retry qiladi (§26). Permission matrix ops-managed (bootstrap grant yo'q, migration emas). Response'da provider metadata/callback payload/OTP/SQL/policy secret YO'Q (§23/§59). `/api/admin/payments/reopen-backlog` + `/reopen-reconcile`.
### TD-231 — Reopen Recovery Concurrency (ACCEPTED, Phase 2.1K)
- **Decision:** Callback bridge (2.1J) va admin reconciliation bir xil idempotent reopen service orqali converge (§27 — one REOPENED, other ALREADY_REOPENED). Bir necha reconciler overlap bo'lishi mumkin (global recovery mutex YO'Q §28); mavjud `pay(order)` lock + reopen invariantlari yagona authority. Backlog SELECT'dan keyin newer PENDING PT (§29 → RETRY_ALREADY_IN_PROGRESS) yoki SUCCEEDED (§30 → PAYMENT_SUCCESS_PENDING_FINALIZATION) paydo bo'lsa reopen service revalidate qiladi va reopen QILMAYDI. Actionable backlog PENDING/SUCCEEDED-PT orderlarni allaqachon exclude qiladi (extra safety).
### TD-232 — Payment Recovery Domain Separation (ACCEPTED, Phase 2.1K)
- **Decision:** To'lov recovery domenlari qat'iy ajratilgan: **SUCCEEDED + PENDING → 2.1H finalization recovery** (PaymentFinalizationService); **FAILED/CANCELLED + PENDING → 2.1K reopen recovery** (PaymentOrderReopenService); ambiguous PENDING attempt → **hech biri** (age'dan terminality infer QILINMAYDI, timeout threshold yo'q §35); REFUNDED → hech biri (refund lifecycle future §36). Recovery domenlari bir-birini CHAQIRMAYDI (2.1K finalizer chaqirmaydi §14/§34). TERMINAL_STATUS_CONFLICT callback recovery authority yaratmaydi — recovery persisted PT status'ga bog'liq, callback result stringni PT state'ni override qilish uchun inspect QILMAYDI (§37).
### TD-233 — Provider-Specific Protocol Persistence (ACCEPTED, Phase 2.1L-D)
- **Decision:** Real provider protocol state **provider-specific** typed jadvallarda saqlanadi (generic ProviderProtocolState JSON YO'Q §2): `PaymeMerchantTransaction` (Payme Merchant API) + `ClickShopTransaction` (CLICK Shop API), har biri PaymentTransaction bilan 1:1 (unique FK, onDelete Restrict). Bu qatorlar iqtisodiy authority EMAS — pul haqiqati core PaymentTransaction/PaymentOrder/PaymentCallbackEvent/Subscription/IZL'da qoladi (§25); `providerMetadata` JSONB supplemental, protocol authority emas. Maqsad: provider protokolini to'g'ri gapirish + idempotent native javoblarni process restart'dan keyin reconstruct qilish + provider identity bind. BigInt (13-digit ms / tiyin) hech qachon JS Number'ga coerce QILINMAYDI. Migration 21 (count 20→21, named CHECK 40→45).
### TD-234 — Non-Terminal Provider Transaction Binding (ACCEPTED, Phase 2.1L-D)
- **Decision:** Provider-native NON-terminal qadam (CLICK Prepare / Payme CreateTransaction) provider transaction id'ni mavjud **PENDING** PaymentTransaction'ga bog'laydi — faqat `provider_transaction_id` yoziladi, `pay(order)` lock ostida; status transition YO'Q, PaymentOrder/IZL/Subscription mutation YO'Q, provider call YO'Q. Idempotency/conflict: external id NULL→attach (BOUND), bir xil id→ALREADY_BOUND (idempotent replay), boshqa id yoki provider mos emas yoki non-PENDING attempt→CONFLICT/NOT_BINDABLE; cross-attempt external-id uniqueness PT-DB-03 (`@@unique([provider, providerTransactionId])`). 2.1F terminal evidence writer QAYTA ISHLATILMAYDI (binding qat'iy pre-terminal, terminal transition + callback event yozmaydi).
### TD-235 — Provider Payment Time Normalization (ACCEPTED, Phase 2.1L-D)
- **Decision:** Barcha merchant-assigned timestamp injected `Clock.now()` → integer Unix ms (BigInt). Payme: birinchi qabul qilingan CreateTransaction → `create_time_ms`, birinchi PerformTransaction → `perform_time_ms` (= kelajakda PaymentTransaction.confirmedAt bo'ladigan instant), birinchi pre-success cancel → `cancel_time_ms`; har biri BIR MARTA yoziladi, replay original instant'ni saqlaydi (repo qayta yozmaydi). Payme `time` (provider creation time) verbatim saqlanadi — GetStatement range authority (§8), local createdAt bilan ALMASHTIRILMAYDI. CLICK: birinchi accepted Complete → confirmedAt=Clock.now() (2.1L-C'da), CLICK `sign_time` iqtisodiy timestamp authority EMAS (§12, faqat signature/protocol input).
### TD-236 — Payme Native State Mapping v1 (ACCEPTED, Phase 2.1L-D)
- **Decision** (official developer.help.paycom.uz Merchant API'dan VERIFIED): native state 1=created (protocol pending, core PT PENDING), 2=performed (kelajakdagi SUCCEEDED evidence), -1=cancelled-before-perform, -2=cancelled-after-perform. state -1 reason semantik: reason 4=PROVIDER_EXPIRED (timeout) va boshqa definitive reasonlar → kelajakda FAILED (core CANCELLED emas — native "-1"ni avtomatik CANCELLED'ga map QILMASLIK, core status semantik reason'ga ergashadi); reason enumeration exact ro'yxati 2.1L-PM'da re-verify qilinadi. state -2 va refund reason 5 = kelajakdagi **Refund/Reversal domain** — 2.1L-D uni HECH QACHON yaratmaydi (post-success cancel REFUND_DOMAIN_UNSUPPORTED bilan rad etiladi). state 1/2/-1/-2 DB CHECK bilan himoyalangan (mapping/normalization 2.1L-PM adapter'da).
### TD-237 — Provider Native Controller Boundary (ACCEPTED, Phase 2.1L-D)
- **Decision:** Kelajakda CLICK Shop controller Prepare/Complete (form-urlencoded), Payme controller Merchant JSON-RPC gapiradi; native protocol vocabulary (CLICK action raqamlari/sign_string/merchant_prepare_id, Payme JSON-RPC error/state/reason integerlari) core finance'ga OQIB O'TMAYDI. `verifyCallback` har bir non-terminal provider method'ni modellashtirisha MAJBUR emas — terminal financial evidence bo'lgandagina normalized evidence services (2.1F/2.1I) chaqiriladi. 2.1L-D'da provider HTTP route OCHILMAYDI (controllerlar hali yaratilmaydi, live endpoint enable qilinmaydi §18/§19).
### TD-238 — Payme Refund Readiness Boundary (ACCEPTED, Phase 2.1L-D)
- **Decision:** Post-success CancelTransaction oddiy non-success EMAS — SUCCEEDED→CANCELLED YO'Q, subscription/cycle/IZL/order rollback YO'Q. Izlan'da refund/reversal iqtisodiy modeli hali yo'q; Payme sandbox performed transaction cancellation'ni sinaydi, shuning uchun Payme real integration strukturaviy tayyorlanishi mumkin, lekin refund/reversal contract hal qilinmaguncha (yoki merchant agreement uni rasmiy istisno qilmaguncha) **production-ready deb E'LON QILINMAYDI**. Hard boundary: post-success cancel/reversal → REFUND_DOMAIN_UNSUPPORTED (soxta muvaffaqiyatli refund javobi IXTIRO QILINMAYDI; Payme -31007 "order completed" semantikasi).
### TD-239 — Real Provider Rollout Order (ACCEPTED, Phase 2.1L-D)
- **Decision:** Ketma-ketlik: (1) 2.1L-D contract/persistence hardening → (2) **2.1L-C CLICK Shop API** (owner tanlovi: CLICK birinchi) → (3) Refund/Reversal architecture recon+impl → (4) 2.1L-PM Payme Merchant API. Har provider alohida, o'z STOP + full-regression boundary bilan; ikkitasi bir fazada implement QILINMAYDI. CLICK real signature/amount/error/native-type constantlari (docs.click.uz Shop API) 2.1L-C boshida majburiy re-verify qilinadi — **CLICK PROTOCOL VERIFICATION BLOCKER** hali OCHIQ (2.1L-D ularni memory'dan implement QILMAGAN; `ClickShopTransaction` provider-neutral shell, native-value CHECK yo'q). Payme protokoli official docs'dan verified.

### TD-240 — Content Revision Lifecycle v1 (ACCEPTED, Phase 2.2A-P owner review; formalized Phase 2.2A-D)
- **Decision:** LessonRevision editorial lifecycle = **DRAFT → REVIEW → PUBLISHED → ARCHIVED** (mavjud `RevisionStatus` enum). Review rejection: **REVIEW → DRAFT**. Almashtrilgan (superseded) published revision → **ARCHIVED**. MVP'da **SUPERSEDED/REJECTED enum YO'Q** (ARCHIVED + return-to-DRAFT yetarli). Lifecycle enforcement (state transitions, draft mutability, published immutability) = kelajakdagi authoring/publishing service (2.2A/2.2B), 2.2A-D EMAS. Manba: [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a.
### TD-241 — Content Publish Authority v1 (ACCEPTED, Phase 2.2A-P owner review; formalized Phase 2.2A-D)
- **Decision:** Publish uchun **`content.publish` permission + subject scope** talab qilinadi. MVP'da **self-publish RUXSAT** (to'g'ri permission/scope bilan — bir kishi author+publish bo'lishi mumkin). **ADMIN role-name bypass YO'Q** (mavjud PermissionsGuard konvensiyasi). Enforcement (SubjectAssignment + permission guard) = 2.2A, 2.2A-D EMAS.
### TD-242 — Published Revision / Runtime Version Authority v1 (ACCEPTED, Phase 2.2A-P owner review; formalized Phase 2.2A-D)
- **Decision:** Joriy learner-visible revision authority = **`Lesson.publishedRevisionId`** (deterministik pointer, "latest by updatedAt" EMAS). Runtime version-selection (allaqachon implement qilingan, o'zgarmaydi): Roadmap/DailyPlan **logical lesson** saqlaydi; lesson START'da aynan revision pin qilinadi (`LearnerLessonProgress.lessonRevisionId`, har `ActivityAttempt`); boshlangan/tugallangan learner **original revision**'da qoladi; boshlanmagan logical lesson start'da **joriy published revision**'ni oladi. Retroaktiv mutation YO'Q.
### TD-243 — Content Lifecycle Safety v1 (ACCEPTED, Phase 2.2A-P owner review; formalized Phase 2.2A-D)
- **Decision:** Urgent takedown = **Lesson `ARCHIVED`** (MVP; revision immutable qoladi, alohida HIDDEN state YO'Q). Referenced/published content **hard-delete QILINMAYDI** (FK Restrict himoya qiladi); referenced bo'lmagan DRAFT o'chirilishi mumkin. Published content MVP'da **erkin reparent QILINMAYDI** (Topic ko'chirish o'rniga clone/new logical content).
### TD-244 — Content Identity / Format / Concurrency v1 (ACCEPTED, Phase 2.2A-P owner review; **contentKey implemented Phase 2.2A-D**)
- **Decision:** **`Lesson.contentKey`** = immutable stable business/import identity (NOT NULL + globally UNIQUE) — `id` (internal), `slug` (routing/SEO), `title` (mutable) dan DISTINCT; **title identity EMAS; slug identity EMAS** (alohida routing/SEO concern, OPEN). Authoring rich-text format = **restricted Markdown** (raw HTML authoring authority EMAS). Edit concurrency = **`updatedAt` optimistic concurrency** (kelajakdagi writer'da enforce, 2.2A). **Phase 2.2A-D contentKey column + UNIQUE'ni implement qildi** (immutability = authoring-service contract, DB trigger EMAS; generation service YO'Q). Deferred: skill merge, media transcript/captions.
### TD-245 — Prerequisite Cycle Invariant v1 (ACCEPTED, Phase 2.2A-P owner review; **self-loop CHECK implemented Phase 2.2A-D**)
- **Decision:** DB row-level CHECK **FAQAT self-loop**'ni oldini oladi (`lesson_id <> prerequisite_lesson_id`, `chk_lesson_prerequisite_no_self_loop`, Phase 2.2A-D). Multi-node DAG cycle (A→B→C→A) **row CHECK bilan aniqlanib BO'LMAYDI** — to'liq DAG cycle prevention kelajakdagi Phase 2.2A authoring service'ning transactional write-time validation'iga tegishli, DB EMAS. Eski "cycle prevention = CHECK" comment tuzatildi.

### TD-246 — Canonical Lesson Activity Registry v1 (ACCEPTED, implemented Phase 2.2A-R)
- **Decision:** ONE exhaustive canonical registry (`src/content/activity/activity-registry.ts`, `activity-registry-v1`) owns Lesson `ActivityType` **runtime capability classification** — executionKind (OBJECTIVE / VIEW_ONLY / UNSUPPORTED), completionEvidence, scoring, learnerProjection, payloadContract. It replaces the objective/view-only `Set`s that were duplicated across the payload parser, completion eligibility, daily-mission policy/repo, learner-signals, and review-session. Exhaustiveness is enforced at **compile time** (`Record<ActivityType, …>`) and by a **runtime test** (every Prisma `ActivityType` classified EXACTLY once; no "unknown → probably unsupported" fallback). Classification mirrors current runtime-v1: objective = MINI_QUESTION/PRACTICE/MASTERY_TEST; view-only = TEXT/EXPLANATION/IMAGE/AUDIO/EXAMPLE; deferred = SPEAKING/WRITING/LISTENING/AI_INTERACTION/VIDEO (**VIDEO stays deferred, not view-only**).
- **One Lesson objective payload authority:** `lesson-activity-objective/v1` (`objective-activity-payload.ts`) remains the single validator + learner-safe projection for objective Lesson activities. LessonExecution / completion / eligibility / daily-mission / review-session **consume the registry** for classification; they hold no local literal.
- **Domain boundary preserved:** AssessmentItem `placement-item/v1` stays a **separate versioned contract** — NOT collapsed into the Lesson Activity domain. The two parsers share ONLY a neutral low-level structural primitive (`src/common/payload/choice-question-payload.ts`, `validateChoiceQuestionBody`): identical choice/true_false structural rules, no shared schemaVersion, error class, `open_ended` policy, projection, or lifecycle — each wrapper passes its own `fail` so domain error identity is preserved. This removes duplicated structural logic without coupling the domains.
- **View-only payload contracts are NOT defined here:** view-only/deferred types carry `payloadContract = NONE_DEFINED`; TEXT/EXPLANATION/IMAGE/AUDIO/EXAMPLE JSON shapes remain a future Phase 2.2A authoring concern (restricted-Markdown format decision, TD-244, does not imply an Activity JSON shape). Registry exposes learner-render capability flags (OBJECTIVE_SAFE / METADATA_ONLY) as backend metadata only — no renderer/CMS/frontend.
- **Scope:** behavior-preserving refactor — **no Prisma schema/migration change**; the future authoring backend (2.2A) consumes this registry for write-time validation. Old import paths kept as thin re-export shims (`objective-activity-payload.ts` re-exports `OBJECTIVE_ACTIVITY_TYPES`/`isObjectiveActivityType` from the registry).
- **Status:** ACCEPTED (implemented Phase 2.2A-R)

### TD-247 — Content Authoring Authorization & Concurrency v1 (ACCEPTED, implemented Phase 2.2A-1)
- **Permissions (frozen for this slice):** `content.author` — author content INSIDE a Subject, ONLY with an active `SubjectAssignment` for it; `content.subject.manage` — create/manage top-level Subjects and manage SubjectAssignments (global capability). Registered in the application permission registry (TD-26/90). No `Permission` table.
- **No role-name bypass:** the generic `PermissionsGuard` is the permission authority; there is NO `role === 'ADMIN'` shortcut anywhere. ADMIN succeeds only because the **system bootstrap** seeds explicit `RolePermission` rows (idempotent, additive, never destructive): METHODIST→`content.author`; ADMIN→`content.author`+`content.subject.manage`; LEARNER/MODERATOR→none. An ADMIN authoring inside a Subject still needs a SubjectAssignment.
- **Permission plane split (exact final semantics):** top-level **Subject create AND Subject metadata PATCH**, plus **all SubjectAssignment management**, require `content.subject.manage` — a **global** capability, so no per-subject assignment is required for them. Only **child content** (Track/Level/Module/Topic/Lesson) authoring uses `content.author` + a SubjectAssignment for the resolved Subject. (A `content.author`-only methodist with an assignment therefore CANNOT edit top-level Subject metadata — that is `content.subject.manage`.)
- **Subject scope = second authorization dimension:** for every child mutation the Subject is resolved from DB relationships (never trusted from the client) and the actor must hold a SubjectAssignment for it. Subject create runs in ONE transaction: DRAFT Subject + creator self-assignment + StaffAudit. IDOR-safe (§17): unknown or out-of-scope resolve to `CONTENT_NOT_FOUND` — existence in another Subject is never revealed.
- **Scope check inside the mutation transaction:** child-create parent resolution + the SubjectAssignment check run INSIDE the same Prisma transaction as the create + audit (using the tx client) — no pre-transaction authorization read. This removes the obvious authorization TOCTOU window. The residual READ COMMITTED race (an assignment revoked concurrently with the same transaction) is a **documented future hardening** (would need `SELECT … FOR UPDATE` / SERIALIZABLE) — deliberately NOT added here to avoid changing the project's transaction policy.
- **updatedAt optimistic concurrency:** every mutating PATCH/move requires `expectedUpdatedAt`; the write is a conditional `updateMany` gated on `id` + `updatedAt` + `status = DRAFT`. Stale token → `CONTENT_EDIT_CONFLICT` (409), no write, no audit. No dedicated version column (formalizes the accepted §13a `updatedAt` decision).
- **PATCH null semantics:** non-null fields (slug/title/code/sortOrder) are "optional by presence but NOT nullable" (`@OptionalPresent` = `ValidateIf(present)`), so an explicit `null` is rejected 400 at the DTO — it never reaches Prisma/PostgreSQL; intentionally nullable fields (`description`, `Lesson.slug`) accept `null` to clear. A missing key means "do not change".
- **StaffAudit same-transaction requirement:** every successful mutation writes its StaffAudit row inside the SAME Prisma transaction as the business mutation (commit/rollback together) — proven by a test that forces the audit write to fail and asserts the business mutation is rolled back and no audit row persists. Rejected mutations write no audit. metadata carries only safe provenance/diff — never tokens/raw bodies.
- **Create/immutability invariants:** authoring creates are DRAFT (client cannot choose status); `createdBy` is always the principal; `Lesson.contentKey` is set on create and is NOT a mutable field (whitelist + forbidNonWhitelisted reject it — no DB trigger). Only the accepted DRAFT Lesson → Topic move (same Subject) reparents; no other reparenting; no status transitions; no hard delete except SubjectAssignment removal.
- **Scope:** **no Prisma schema/migration change** (existing hierarchy/contentKey/SubjectAssignment/StaffAudit/updatedAt suffice). LessonRevision/Activity/prerequisite/publish authoring are NOT in this slice (2.2A-2 / 2.2A-3).
- **Status:** ACCEPTED (implemented Phase 2.2A-1)

### TD-248 — Draft Revision & Activity Authoring v1 (ACCEPTED, implemented Phase 2.2A-2)
- **Revision version authority:** `LessonRevision.version` is **backend-generated, monotonic per Lesson** (`max+1`),
  never client-supplied; concurrency is the existing `@@unique([lessonId, version])` with a bounded retry around the
  race (no advisory locks / schema). **Multiple DRAFT revisions per Lesson are NOT prohibited.** A new DRAFT revision may
  be created under a **DRAFT or PUBLISHED** Lesson (never ARCHIVED); creating one does **not** mutate `Lesson.status`.
- **Mutability:** only `RevisionStatus.DRAFT` content is mutable. Revision PATCH edits only `title`/`description`
  (`updatedBy` = actor, server-owned); `version`/`status`/reviewer/publisher/`publishedAt`/`estimatedDurationMin` are not
  ordinary fields. No review/publish/archive transition endpoints in this phase.
- **Revision = Activity concurrency aggregate:** every Activity mutation (create/patch/delete/reorder) carries
  `expectedRevisionUpdatedAt`; in one transaction it resolves + scopes the revision, conditionally claims the DRAFT
  revision on `updatedAt` (setting `updatedBy` = actor), mutates the Activity, writes StaffAudit, and returns the new
  revision `updatedAt`. Stale token → 409 before any business write. `Activity.updatedAt` alone is NOT the token.
- **Activity type/position:** `type` is **not** an ordinary PATCH field (wrong type = delete + recreate); `position`
  changes **only** through the atomic reorder endpoint (collision-safe two-step rewrite honoring `@@unique([revision,
  position])`). Manual staff authoring sets `source = HUMAN`, `aiMetadata = null` (server-owned).
- **Payload contracts (closed v1, registry-driven `payloadContract`):** OBJECTIVE (MINI_QUESTION/PRACTICE/MASTERY_TEST) →
  `lesson-activity-objective/v1` (the canonical learner parser is reused as the authoring validator; its learner error is
  adapted to a safe staff error; full payload incl. answerKey stored server-side). TEXT/EXPLANATION/EXAMPLE →
  `lesson-activity-markdown/v1` = strict `{ schemaVersion, markdown }` (restricted Markdown; **no rawHtml/html field** —
  storage contract only). IMAGE/AUDIO → `lesson-activity-media/v1` = strict `{ schemaVersion }` marker; **media identity
  stays relational** (`ActivityMedia → MediaAsset`) — never mediaAssetId/URL/storageKey in the payload; DRAFT VALIDITY ≠
  PUBLISH READINESS. SPEAKING/WRITING/LISTENING/AI_INTERACTION/VIDEO → NONE_DEFINED = **not authorable v1**. One canonical
  authoring dispatcher consumes the registry; no ActivityType classification list is duplicated.
- **Secrets / learner runtime:** staff reads expose the full objective payload (author edits the answer key); the LEARNER
  runtime is UNCHANGED (still strips answerKey, view-only stays metadata-only). StaffAudit stores only safe metadata
  (ids, activityType, position, schemaVersion, changed field names) — never payload/markdown/answerKey; payloads are never
  logged. **No Prisma schema/migration change.**
- **OCC token precision (2.2A-2 review clarification — applies to TD-247 writers too):** the optimistic-concurrency
  token is the row's `updatedAt`, stored as PostgreSQL **TIMESTAMP(3)** (millisecond precision). Every content-authoring
  conditional writer (Subject/Track/Level/Module/Topic/Lesson update + lesson move, and revision update + the revision
  aggregate `touch`) now **explicitly** sets `updatedAt = max(now, expected + 1ms)` in the SAME conditional `updateMany`
  (one DB write, inside the existing transaction) so a successful write **strictly advances** the token
  (`new updatedAt > expected`) even at the same-millisecond boundary or if the wall clock moved backward — closing a
  theoretical stale-token re-use at TIMESTAMP(3) precision. No version column / lock / extra write. Proven by a
  deterministic same-millisecond test (freeze `Date.now()` to the stored token's exact ms). This hardens the
  already-accepted TD-247 concurrency contract; it is not a new decision.
- **Status:** ACCEPTED (implemented Phase 2.2A-2)

### TD-249 — Skill Mapping & Prerequisite DAG Authoring v1 (ACCEPTED, implemented Phase 2.2A-3)
- **Skill:** Subject-scoped (`Skill.subjectId`); authored via content.author + SubjectAssignment. **No hard delete /
  merge / archive lifecycle** in this phase; new Skills are ACTIVE; only ACTIVE skills are metadata-editable (ARCHIVED →
  safe 409). Skill PATCH uses strict-monotonic `updatedAt` OCC; `subjectId`/`status` immutable (no cross-Subject move).
  Duplicate `(subjectId,name)`/`(subjectId,code)` → safe 409.
- **Aggregate concurrency tokens:** LessonSkill + LessonPrerequisite mutations use **`Lesson.updatedAt`** (DRAFT logical
  Lesson only); ActivitySkill mutations use **`LessonRevision.updatedAt`** (DRAFT revision only), NOT `Activity.updatedAt`.
  Idempotent add/remove with a CURRENT token is a no-op (no token advance, no audit); a STALE token is 409 even when the
  requested final state already exists. All strict-monotonic (TD-247/248 hardening).
- **Same-Subject mapping invariant:** the Skill's Subject must equal the Lesson/Activity's Subject; cross-subject targets
  resolve to `CONTENT_NOT_FOUND` (IDOR-safe). Only ACTIVE skills may be newly assigned.
- **Lifecycle gates (protect live learner authorities before publishing exists):** LessonSkill mutates only a DRAFT
  logical Lesson; ActivitySkill only a DRAFT LessonRevision; prerequisite mutates only a DRAFT source Lesson. These write
  the EXISTING learner authorities (roadmap `LessonSkill`/`LessonPrerequisite`, review-session `ActivitySkill`) — no shadow
  tables, no learner-runtime change, no historical roadmap snapshot rewrite.
- **Prerequisites are same-Subject** (both Lessons resolved from DB; Subjects must match); DRAFT source, DRAFT-or-PUBLISHED
  target (ARCHIVED target rejected). Direct self-loop rejected at the service layer (the `chk_lesson_prerequisite_no_self_loop`
  DB CHECK remains defense-in-depth — it does NOT detect multi-node cycles).
- **Transactional full-DAG cycle prevention:** before adding `A → B` there must be no existing path `B → … → A`; the check
  uses the **WHOLE Subject graph** (all edges whose source Lesson is in the Subject), via an iterative (non-recursive) path
  search. Graph mutations for a Subject are **serialized by locking the Subject row (`SELECT … FOR UPDATE`)** inside the
  transaction before reading the graph + validating + writing — so two concurrent inverse-edge writers cannot both commit a
  cycle. ADD and REMOVE both acquire the same Subject lock. No app-mutex / Redis / advisory-lock / lock-table.
- **Atomicity:** actual relation write + aggregate `touch` + StaffAudit are ONE transaction (audit failure rolls all back);
  cycle rejection / no-op / stale writes touch nothing and audit nothing. **No Prisma schema/migration change.**
- **Status:** ACCEPTED (implemented Phase 2.2A-3)

### TD-250 — Review, Publication & Learner Visibility v1 (ACCEPTED, implemented Phase 2.2B)
- **Authority:** `content.publish` + SubjectAssignment (inside the transaction; no role-name / ADMIN bypass). MVP
  **self-publish**: METHODIST holds `content.author` + `content.publish` (ADMIN also `content.subject.manage`); the same
  actor may be createdBy = reviewedBy = publishedBy.
- **Lifecycle (frozen):** revision `DRAFT → REVIEW → PUBLISHED → ARCHIVED`; review rejection is `REVIEW → DRAFT`
  (mandatory reason in `StaffAudit.reason`; no SUPERSEDED/REJECTED enum). `reviewedBy` is stamped at publication approval.
  Explicit **top-down hierarchy publication** (Subject→Track→Level→Module→Topic; a container publishes only when its
  ancestors are PUBLISHED; no auto-publish). No status transitions via ordinary authoring PATCH.
- **`Lesson.publishedRevisionId` is the current publication authority.** Actual publication is ONE transaction serialized
  by the **Lesson row (`SELECT … FOR UPDATE`)**: validate revision + Lesson OCC tokens, rerun publish-readiness, archive
  the old current revision (PUBLISHED → ARCHIVED, coherence-checked), stamp the new revision (PUBLISHED + reviewedBy +
  publishedBy + publishedAt + duration cache), move the Lesson pointer + status, StaffAudit — no externally visible
  intermediate state. Two concurrent REVIEW revisions of the same Lesson cannot both state-change publish (loser 409).
- **Idempotent republish:** re-publishing the ALREADY-current coherent revision is a success no-op (no status/timestamp
  change, no second audit) even with now-stale tokens; a stale token for a DIFFERENT actual publication still 409s;
  publishing an ARCHIVED or non-current-PUBLISHED revision is invalid.
- **Readiness (canonical, metadata-only):** review blockers = zero activities / non-`0..N-1` positions / unauthorable type
  / invalid payload (registry + authoring dispatcher; no duplicated type lists) / ActivitySkill→archived skill. Publish
  blockers add parent-not-PUBLISHED, Lesson-ARCHIVED, prerequisite-not-PUBLISHED, LessonSkill→archived skill, media
  readiness (IMAGE/AUDIO require a ready, non-BLOCKED, MIME-matching ActivityMedia→MediaAsset; UNREVIEWED = warning), and
  current-pointer coherence. Warnings: no LessonSkill, objective without ActivitySkill, DURATION_INCOMPLETE. Never leaks
  payload/answerKey/markdown/storageKey. Media identity stays relational; media delivery remains out of scope.
- **Learner projection & visibility:** ONE shared learner-safe Activity projector (registry `learnerProjection`) used by
  BOTH staff preview and LessonExecution — objective answerKey stripped; TEXT/EXPLANATION/EXAMPLE → validated markdown
  body (MARKDOWN_SAFE); IMAGE/AUDIO metadata-only; never storageKey. ONE canonical visibility authority requiring the
  FULL hierarchy (Subject→Track→Level→Module→Topic→Lesson PUBLISHED) + current-pointer coherence, reconciling the
  previously-incomplete `LessonExecutionRepository.publishedRevisionId` and `RoadmapRepository.lessonMeta` (which stopped
  at Level).
- **History & takedown:** a replacement keeps the old PUBLISHED revision as ARCHIVED history — a pinned learner keeps
  resuming the OLD pinned revision (never repinned); a NEW learner pins the new current revision. Urgent **takedown**
  (`Lesson.status = ARCHIVED`, audited reason) removes learner access (new start + resume + roadmap eligibility) WITHOUT
  moving the pointer, archiving the revision, or deleting progress/completion/attempt history. No Prisma schema/migration change.
- **Status:** ACCEPTED (implemented Phase 2.2B)

> Bu hujjat D-04'dagi "hali tanlanmagan" ro'yxatini bosqichma-bosqich yopib boradi.
> Faqat haqiqatan qabul qilingan qarorlar ACCEPTED; product tasdig'ini kutayotganlar bu yerda yozilmaydi ([OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) va [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md) §23).

## Platform stack

### TD-01 — Frontend: Next.js + TypeScript
- **Status:** ACCEPTED (Phase 0)

### TD-02 — Backend: Node.js + TypeScript
- **Status:** ACCEPTED (Phase 0)

### TD-03 — Backend framework: NestJS
- **Why:** Modular Monolith uchun modul tizimi, DI, guards/decorators kabi strukturaviy imkoniyatlar.
- **Status:** ACCEPTED (Phase 1.1)

### TD-04 — HTTP adapter: Fastify
- **Why:** NestJS ostida Express'ga nisbatan yuqoriroq throughput.
- **Status:** ACCEPTED (Phase 1.1)

### TD-05 — Database: PostgreSQL
- **Status:** ACCEPTED (Phase 0)

### TD-06 — ORM: Prisma
- **Why:** TypeScript-first, migration tooling, tez development.
- **Status:** ACCEPTED (Phase 1.1)

### TD-07 — Architecture: Modular Monolith
- **Why:** Greenfield'da tezlik; modul chegaralari keyingi o'sishga (kerak bo'lsa ajratishga) tayyorlaydi.
- **Status:** ACCEPTED (Phase 0)

## Authentication & session architecture (Phase 1.1)

Batafsil dizayn: [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md).

### TD-08 — Token model: short-lived access + rotating refresh + server-side sessions
- **Decision:** Qisqa muddatli access JWT (~10–15 min, tuning) + opaque rotating refresh token + server-side AuthSession recordlar. Long-lived (masalan 30 kunlik) JWT ishlatilmaydi.
- **Why:** Revocation (logout, logout-all, suspension) server tomonda boshqarilishi kerak; long-lived JWT'ni revoke qilib bo'lmaydi.
- **Consequences:** Refresh endpoint har doim DB'ga boradi; oddiy endpointlarda revocation kechikishi access TTL bilan chegaralanadi; sensitive endpointlarda live session check.
- **Status:** ACCEPTED

### TD-09 — Refresh rotation + reuse detection
- **Decision:** Har refresh'da eski token invalidated, yangisi beriladi. Ishlatilgan token qayta kelsa — butun session/family revoke + security event.
- **Why:** O'g'irlangan refresh token'ning umri bitta ishlatishgacha qisqaradi; o'g'irlik aniqlanadi.
- **Status:** ACCEPTED

### TD-10 — Secretlar hech qachon plaintext saqlanmaydi
- **Decision:** OTP — hash (server-side pepper bilan HMAC yoki argon2); refresh token — hash (masalan SHA-256). OTP va refresh token hech qanday logga yozilmaydi.
- **Why:** DB/log leak'da ishlaydigan credential qolmasligi kerak; 6 raqamli OTP maydoni kichik — pepper + server-side attempt limit shart.
- **Status:** ACCEPTED

### TD-11 — OTP lifecycle qoidalari
- **Decision:** Single-use; muddatli (taxminan 3–5 min); resend cooldown; resend'da oldingi challenge invalidation; challenge boshiga attempt limit; per-phone va per-IP limitlar. Exact raqamlar — tuning parameter (final emas).
- **Why:** Brute-force, SMS bombing va xarajat hujumlariga qarshi minimal production-grade to'plam.
- **Status:** ACCEPTED (qoidalar to'plami); raqamlar — TUNING

### TD-12 — Web token storage: HttpOnly Secure cookies
- **ACCEPTED:**
  - Refresh token web'da HttpOnly + Secure + SameSite cookie orqali saqlanadi (refresh endpoint path'iga scope qilingan);
  - localStorage/sessionStorage token storage sifatida ishlatilmaydi;
  - state-changing endpointlarda CSRF protection bo'lishi shart.
- **STILL OPEN:**
  - Access token aynan HttpOnly cookie'da bo'ladimi yoki client memory'da;
  - exact CSRF mechanism (SameSite + custom header / CSRF token varianti).
- **Why:** XSS orqali token exfiltration'ni yopish; CSRF alohida mexanizm bilan boshqariladi.
- **Status:** ACCEPTED (faqat yuqoridagi ACCEPTED ro'yxati doirasida; OPEN bandlari implementation'da final qilinadi)

### TD-13 — Transport-agnostic token issuance (mobile-ready)
- **Decision:** Core auth "session + token juftligi" bilan ishlaydi; cookie'ga o'rash — web adapter qatlami. Kelajak mobile clientlar tokenlarni body orqali olib OS secure storage'da saqlaydi; session'da client/platform turi qayd etiladi.
- **Why:** Android/iOS keyin qo'shilganda auth core o'zgarmasligi kerak.
- **Status:** ACCEPTED

### TD-14 — Authorization: permission-based, token yagona manba emas
- **Decision:** `ROLE → PERMISSIONS` modeli; NestJS'da AuthGuard (authentication) va PermissionsGuard + decorator (authorization) ajratiladi; `if (role === "ADMIN")` tarqoq tekshiruvlari taqiqlanadi. Staff/sensitive amallarda permissionlar server-side tekshiriladi.
- **Why:** Product qarori D-33; staff permission o'zgarganda eski token bilan davom etib bo'lmasligi kerak.
- **Consequences:** Exact permission schema keyingi bosqichda.
- **Status:** ACCEPTED

### TD-15 — Account states + alohida onboarding holati
- **Decision:** User status: ACTIVE / SUSPENDED / DEACTIVATED. Onboarding completion — account state emas, ACTIVE userning alohida belgisi (incomplete onboarding'li user authenticated bo'la oladi, core featurelarga kira olmaydi).
- **Why:** Onboardingni davom ettirish uchun login kerak; state'lar minimal qolsin.
- **Consequences:** DEACTIVATED semantikasi (kim/qanday/qaytish) — OPEN product savoli.
- **Status:** ACCEPTED (state to'plami); DEACTIVATED policy — OPEN

### TD-16 — Phone canonical format: E.164 (+998...)
- **Decision:** DB'da faqat canonical `+998XXXXXXXXX`; uniqueness shu shakl bo'yicha; normalization faqat backendda; frontend faqat display formatting.
- **Why:** Yagona identity kalitida format xilma-xilligi bo'lmasligi kerak; kelajakda boshqa davlatlarga E.164 kengayadi.
- **Status:** ACCEPTED

### TD-17 — SMS provider abstraction
- **Decision:** Auth logic SMS'ni faqat conceptual port orqali ishlatadi (standart natija semantikasi bilan); provider adapteri almashtiriladigan. Provider tanlanmagan — OPEN.
- **Why:** Provider tanlovi/almashishi auth business logic'ka ta'sir qilmasligi kerak.
- **Status:** ACCEPTED (abstraction); provider — OPEN

### TD-18 — Security events ≠ financial audit
- **Decision:** Auth/security eventlar (OTP requested/failed, login, reuse detected, revoke, suspension...) financial audit ledger'dan (D-35) alohida tizimda; loglarda PII minimal, OTP/refresh token hech qachon yozilmaydi.
- **Why:** Har xil maqsad, har xil retention/access talablari.
- **Consequences:** Retention va to'liq telefon saqlash siyosati — OPEN.
- **Status:** ACCEPTED

### TD-19 — Rate limiting: layered, Redis'siz MVP
- **Decision:** Per-IP + per-identity limitlar; OTP limitlari DB-backed (challenge yozuvlari asosida, instance'dan mustaqil); yengil limitlar in-memory; limiter abstraction ortida — keyinchalik Redis distributed limiting sifatida qo'shilishi mumkin. Redis hozir dependency emas.
- **Why:** MVP'da qo'shimcha infra'siz ishlash; kengayish yo'li ochiq.
- **Status:** ACCEPTED

### TD-20 — Auth module boundaries
- **Decision:** Conceptual chegara: AuthModule (OTP/token/session), UsersModule (user/profile/onboarding), AuthorizationModule (permissions/guards), SmsModule (provider port), SecurityModule (events, rate limiting). Auth ichiga profile/SMS detali/permission tizimi tiqilmaydi.
- **Why:** Separation of concerns; Modular Monolith intizomi.
- **Status:** ACCEPTED (conceptual; exact struktura implementation'da)

## Phase 1.2A — Core data model

Batafsil tahlil: [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md). Product Owner review yakunlandi: **TD-21..TD-30 hammasi ACCEPTED** (TD-21..25 — birinchi review; TD-26..30 — final review). **Phase 1.2A technical decision review nuqtai nazaridan COMPLETE.**

### TD-21 — Activity storage: bitta jadval + type enum + JSONB payload
- **Decision:** Barcha block turlari bitta Activity jadvalida: cross-cutting maydonlar (type, position, duration) — column; content — JSONB payload. Subtype/per-type jadvallar rad etildi (Prisma polymorphism og'irligi, migration pain).
- **Status:** ACCEPTED (2026-08-22 review)

### TD-22 — Activity payload validation: application-level discriminated union
- **Decision:** Har ActivityType uchun payload schema; create/update'da validatsiya + publish transition'da strict qayta tekshiruv. Exact validation library — implementation'da.
- **Status:** ACCEPTED (2026-08-22 review)

### TD-23 — ID strategy: UUIDv7
- **Decision:** Barcha PK'lar UUIDv7 (native uuid): time-ordered index locality + public expose xavfsizligi. bigint (enumerable) va CUID (string) rad etildi.
- **Note:** UUIDv7 qiymati PostgreSQL tomonidami yoki Prisma/application tomonida generatsiya qilinishi — architecture decision EMAS, implementation detail sifatida keyin hal qilinadi.
- **Status:** ACCEPTED (2026-08-22 review; format bo'yicha)

### TD-24 — Lesson versioning: draft revision + immutable published snapshot
- **Decision:** Lesson (logical id) + LessonRevision; publish = transactional pointer swap (`published_revision_id`); eski published revision immutable ARCHIVED — learner history barqaror referencelarda qoladi. In-place edit rad etildi.
- **Status:** ACCEPTED (2026-08-22 review)

### TD-25 — MediaAsset: provider-neutral media entity
- **Decision:** Alohida MediaAsset (storage_key, mime, size, duration, uploaded_by, status); Activity payload media'ni `media_asset_id` orqali reference qiladi (raw URL emas). Community kelajakda shu infradan foydalanadi.
- **Status:** ACCEPTED (2026-08-22 review)

### TD-26 — Authorization relational model
- **Decision:** Role — seeded jadval (enum emas); UserRole N:M (default LEARNER + qo'shimcha staff rollari); RolePermission(role_id, permission_code) — permission kodlari application registry'da, mapping DB'da.
- **Status:** ACCEPTED (final review)

### TD-27 — Hierarchy: to'liq explicit zanjir + default structural node
- **Decision:** Subject → Track → Level → Module → Topic → Lesson hierarchy database/domain modelda **explicit va to'liq** qoladi. Generic CMS tree ishlatilmaydi. Level — data-driven entity (code/title/sort_order), English-only enum emas.
- **Clarification (owner):** Agar kelajakdagi Subject uchun Track yoki Level semantik jihatdan kerak bo'lmasa, **system/default structural node** ishlatilishi mumkin; bunday node UI'da Learner'dan yashirilishi mumkin. Default node product hierarchy'ni buzmaydi — faqat structural compatibility uchun.
- **Status:** ACCEPTED (final review)

### TD-28 — Lifecycle granularity (final wording)
- **Decision:**
  - Subject / Track / Level / Module / Topic: `DRAFT → PUBLISHED → ARCHIVED`
  - Lesson (logical entity): `DRAFT / PUBLISHED / ARCHIVED`
  - LessonRevision: `DRAFT → REVIEW → PUBLISHED → ARCHIVED`
  - Activity: alohida lifecycle status yo'q — parent LessonRevision lifecycle'iga ergashadi
  - Skill: `ACTIVE → ARCHIVED`
- **Muhim:** REVIEW state logical Lesson'da emas, **LessonRevision'da** — shunday qilib hozirgi PUBLISHED revision mavjud bo'lgan paytda yangi revision REVIEW holatida bo'lishi mumkin. AI-assisted content human review'dan o'tmasdan PUBLISHED bo'la olmaydi.
- **Status:** ACCEPTED (final review)

### TD-29 — Prerequisites va ordering
- **Decision:** Gating — explicit LessonPrerequisite junction (DAG); sort_order faqat presentation uchun. **Cycle prevention application/service layerda transaction doirasida tekshiriladi.** Activity ordering simple integer position bilan boshlanadi (reorder — transactional renumber; gaps/fractional rank rad etildi).
- **Status:** ACCEPTED (final review)

### TD-30 — Timestamps / deletion policy
- **Decision:** Global `deleted_at`/soft-delete ishlatilmaydi — semantic lifecycle/account states ishlatiladi. Published/referenced content uchun normal removal = **ARCHIVE**. Referential integrity destructive hard-delete'ni cheklashi kerak (RESTRICT). Hard delete faqat xavfsiz ephemeral (masalan muddati o'tgan OtpChallenge) yoki hech qachon publish qilinmagan disposable data uchun. created_at/updated_at — standard.
- **Status:** ACCEPTED (final review)

## Phase 1.2B — Learning domain data model

Batafsil tahlil: [DATA_MODEL_LEARNING.md](DATA_MODEL_LEARNING.md). Product Owner review yakunlandi: **TD-31..TD-44 hammasi ACCEPTED** (TD-35/37/40/42 — quyidagi clarificationlar bilan). **Phase 1.2B COMPLETE.**

### TD-31 — Evidence + materialized current state (hybrid)
- **Decision:** Append-only raw evidence (ActivityAttempt, AssessmentResponse, AiEvaluation, SkillMeasurement) + engine yozadigan mutable current state (LearnerSkillState, LearnerLessonProgress, LearnerSignal). Progress/scoring formulalari DB'da emas — engine'da; formula o'zgarsa evidence'dan recompute.
- **Status:** ACCEPTED (owner review)

### TD-32 — Assessment content: AssessmentDefinition + assessment-owned immutable AssessmentItem
- **Decision:** Assessment itemlari lesson Activity'laridan reuse qilinmaydi (calibration metadata, mustaqillik, o'lchov sifati); alohida AssessmentItem — TD-21 falsafasi bilan (type + JSONB + strict validation) + skill_id + difficulty. PUBLISHED item immutable (o'zgarish = yangi item). AI-sourced item human review'siz publish bo'lmaydi.
- **Status:** ACCEPTED (owner review)

### TD-33 — Adaptive assessment state: normalized responses + engine_state JSONB
- **Decision:** Har taqdim etilgan item + javob = AssessmentResponse qatori (deterministic replay/audit manbai); joriy adaptive pozitsiya = AssessmentAttempt.engine_state JSONB + engine_version (resume + reproducibility). Obyektiv itemlar faqat deterministic scoring bilan.
- **Status:** ACCEPTED (owner review)

### TD-34 — AiEvaluation: alohida async entity
- **Decision:** AI baholash response ichidagi JSONB emas — alohida entity (PENDING/COMPLETED/FAILED lifecycle, re-evaluation imkoni, future cost linkage): score + rubric JSONB + feedback + provider_metadata + evaluation_version. AssessmentResponse XOR ActivityAttempt'ga bog'lanadi.
- **Status:** ACCEPTED (owner review)

### TD-35 — Skill state: current + milestone history
- **Decision:** LearnerSkillState (unique user+skill; mastery_score ichki normalized + display_level derived string — content Level'ga FK emas, English hard-code yo'q) + append-only SkillMeasurement faqat mazmunli chegaralarda (diagnostic/checkpoint/mastery/engine recalc) — har mini savolda emas.
- **Clarification (owner):** Authoritative mathematical state — **mastery_score + confidence + evidence_count**. `display_level` authoritative emas — derived representation (masalan mastery_score=0.63, confidence=0.81 → proficiency mapping → "B1"); mapping keyinchalik o'zgarishi mumkin va content `Level` entity'siga hard FK bilan bog'lanmaydi.
- **Status:** ACCEPTED (owner review)

### TD-36 — ActivityAttempt evidence + JSONB answers
- **Decision:** Attempt faqat javob talab qiladigan bloklar uchun (view-only bloklar LearnerLessonProgress ichida — alohida event jadvalisiz). Answer — JSONB (TD-21/22 bilan consistent; speaking → MediaAsset reference). unique(user, activity, attempt_no); SUBMITTED'dan keyin append-only.
- **Status:** ACCEPTED (owner review)

### TD-37 — Lesson progress + revision-pinning policy
- **Decision:** LearnerLessonProgress unique(user, lesson); boshlanmagan lesson → latest PUBLISHED revision; boshlashda revision pin qilinadi va lesson shu revisionda tugatiladi; COMPLETED progress revision reference'ini abadiy saqlaydi.
- **Clarification (owner):** Yangi revision publish bo'lganda boshlagan learner **silent migration qilinmaydi** — pinned revision bilan davom etadi; yangi/boshlamagan learnerlar yangi revisionni oladi. Jiddiy content xatosi uchun kelajakda **explicit force-migration mechanism** bo'lishi mumkin — bu ordinary publish flow emas va hozir implement qilinmaydi.
- **Status:** ACCEPTED (owner review; completed-lesson qayta ko'rish display savoli OPEN)

### TD-38 — Roadmap: LearnerRoadmap + typed RoadmapItem, logical Lesson reference
- **Decision:** Bitta (user, subject)da bitta ACTIVE roadmap; RoadmapItem typed (LESSON/REVIEW/PRACTICE/CHECKPOINT) + nullable FK'lar + params JSONB + reason (explainability). Roadmap logical Lesson'ga bog'lanadi — revision execution paytida progress'da pin bo'ladi.
- **Status:** ACCEPTED (owner review)

### TD-39 — Recommendation/Change audit modeli
- **Decision:** LearnerRecommendation (AI-specific nomlanmagan; reason + proposed_change + signal_refs; PROPOSED/ACCEPTED/REJECTED/EXPIRED; content immutable) + append-only RoadmapChange. Recommendation-driven o'zgarish faqat ACCEPTED recommendation orqali (acceptance-gated mutation). Full roadmap versioning yo'q — change log yetarli.
- **Status:** ACCEPTED (owner review)

### TD-40 — LearnerSignal: generic derived signal modeli
- **Decision:** Bitta signal jadvali: kichik type enum + moslashuvchan category_code string (mistake taxonomy OPEN — enum emas) + evidence_refs + lifecycle. Persisted (explainability/lifecycle uchun), lekin evidence'dan recomputable. Review tarixi alohida jadvalsiz — signal→recommendation→roadmap item→attempts zanjiridan derived.
- **Clarification (owner):** Signal — har qanday metric emas, **actionable learning insight** (REPEATED_MISTAKE, REVIEW_DUE, CONSISTENCY_RISK kabi). Current score/normal progress — LearnerSkillState'da, signal emas. Lifecycle: ACTIVE → RESOLVED yoki ACTIVE → EXPIRED. Exact taxonomy OPEN.
- **Status:** ACCEPTED (owner review)

### TD-41 — Checkpoint: entity → AssessmentDefinition; attempt = purpose
- **Decision:** Checkpoint (module_id unique, assessment_definition_id, lifecycle) — boolean emas; checkpoint attempt alohida jadval emas — AssessmentAttempt(purpose=CHECKPOINT, checkpoint_id). Oldin/keyin taqqoslash SkillMeasurement tarixidan derived.
- **Status:** ACCEPTED (owner review)

### TD-42 — DailyPlan: versioned/supersedable snapshot + items
- **Decision:** DailyPlan persisted snapshot + DailyPlanItem (section MUST_DO/RECOMMENDED/EXTRA, typed, roadmap_item bog'lanishi). Daily Recap — persisted emas, derived.
- **Clarification (owner):** `unique(user, date)` **mutable singleton EMAS**. DailyPlan — versioned/supersedable snapshot: uniqueness `(user, local_date, generation_no)`; bir vaqtda faqat bitta CURRENT plan; regeneratsiya ("bugun 30 min") → eski snapshot SUPERSEDED bo'ladi, **rewrite/delete qilinmaydi**. DailyPlanItem aynan o'z generation snapshot'iga tegishli. Bu kelajakdagi IZL eligibility audit uchun muhim.
- **Status:** ACCEPTED (owner review)

### TD-43 — Session/vaqt/event intizomi
- **Decision:** LearningSession aggregate active_seconds saqlaydi; raw heartbeat forever saqlanmaydi (qisqa retention telemetry — implementation detail); generic LearningEvent jadvali va event sourcing YO'Q — domain jadvallar evidence'ni qamraydi; AI context — alohida memory table emas, domain jadvallardan Context Assembler; tutor suhbat persistence — AI bosqichiga (faqat structured natijalar 1.2B'da).
- **Status:** ACCEPTED (owner review)

### TD-44 — Idempotency: unique constraints + state-machine no-op
- **Decision:** Submit/complete/accept operatsiyalari idempotent: unique constraintlar (attempt_no, sequence_no, plan_date...) + terminal statuslarga qayta o'tish no-op + submit yo'llarida optional client_request_id. Alohida idempotency jadvali yo'q.
- **Status:** ACCEPTED (owner review)

## Phase 1.2C-1 — Financial domain data model

Batafsil tahlil: [DATA_MODEL_FINANCE.md](DATA_MODEL_FINANCE.md). Product Owner review yakunlandi: **TD-45..TD-60 hammasi ACCEPTED** (TD-47/50/52/57 — quyidagi clarificationlar bilan). **Phase 1.2C-1 COMPLETE.**

### TD-45 — XP ≠ IZL: alohida modellar
- **Decision:** XP — XpBalance (aggregate + level cache) + yengil append-only XpGrant (reason_code, yig'ma grant mumkin); IZL — financial-style ledger. Bir jadval/ledger'da aralashtirilmaydi — har xil qat'iylik darajasi.
- **Status:** ACCEPTED (owner review)

### TD-46 — Pul representatsiyasi: integer bigint, float yo'q
- **Decision:** IZL — integer units (bigint, kasrsiz); UZS — integer so'm (bigint). Hech qayerda floating-point yo'q. Provider amount formati (tiyin va h.k.) — adapter mapping. `1 IZL = X UZS` — data (IzlRateVersion), schema emas.
- **Status:** ACCEPTED (owner review)

### TD-47 — IZL: append-only signed ledger + wallet cache
- **Decision:** IZLLedgerEntry (EARN/REDEEM/ADJUSTMENT/REVERSAL, signed amount, balance_after, referencelar) — UPDATE/DELETE hech qachon; korreksiya faqat REVERSAL/ADJUSTMENT entry. IZLWallet — cached balance (ledger bilan bir tranzaksiyada) + concurrency lock anchor. Double-entry rad etildi — closed-loop bir valyutali reward tizimi uchun overengineering.
- **Clarification (owner):**
  - **Wallet-local ordering:** ledger entrylar wallet ichida aniq monotonic tartibga ega (conceptual: `entry_no`/`wallet_version`). Har tranzaksiya: wallet row lock → current version → next entry number → ledger entry → balance update → commit. UUIDv7 vaqt bo'yicha yaxshi ordering beradi, lekin financial ledger ordering'ining **yagona authority'si sifatida UUID timestamp'ga tayanilmaydi**. Exact field nomi — implementation detail.
  - **Reservation:** wallet conceptual ravishda total/current balance va **reserved_amount**ni ajratadi: `spendable = total_balance − reserved_amount`. Temporary payment reservation — ledgerdagi actual financial balance movement **EMAS**.
- **Status:** ACCEPTED (owner review)

### TD-48 — RewardGrant boundary + evidence FK + dedup_key idempotency
- **Decision:** Learning → Reward Engine → RewardGrant → Ledger oqimi; grant kategoriyaga mos dedicated evidence FK'ga ega (polymorphic source rad — FK integrity); unique(user, dedup_key) — bir evidence uchun bir marta IZL (DB-darajali anti-farming). Har grant policy versiyasiga bog'lanadi.
- **Status:** ACCEPTED (owner review)

### TD-49 — RewardPolicyVersion
- **Decision:** Reward konfiguratsiyasi (kategoriya taqsimoti, mastery threshold, qiymatlar — exact sonlar OPEN) versioned config yozuvi; grantlar va cycle'lar version'ga FK — tarixiy iqtisod izohlanadi; Admin o'zgarishi eski cycle'ga ta'sir qilmaydi.
- **Status:** ACCEPTED (owner review)

### TD-50 — 20% ceiling: SubscriptionCycle snapshot
- **Decision:** SubscriptionCycle reward ceiling economics'ni snapshot qiladi; earned_izl cached, lock ostida tekshiriladi.
- **Clarification (owner):** ceiling **gross narxdanmi yoki actual paid/net narxdanmi — PRODUCT QUESTION OPEN**; data model policy'ga bog'lanmaydi. Cycle conceptual snapshotlari: `gross_price_uzs`, `discount_uzs`, `paid_amount_uzs`, **`reward_basis_uzs`** (20% aynan qaysi qiymatga qo'llanganini tarixiy ko'rsatadi), `reward_ceiling_uzs`, `izl_rate_snapshot`, `reward_ceiling_izl`. Product keyin gross yoki net tanlaydi; oldingi cycle keyingi policy change bilan **qayta hisoblanmaydi**.
- **Status:** ACCEPTED (owner review; basis tanlovi OPEN)

### TD-51 — Plan identity + PlanPrice versioning
- **Decision:** SubscriptionPlan (code seedable) narxsiz identity; PlanPrice — effective_from/to bilan versioned, tarixiy qator hech qachon UPDATE qilinmaydi; narx o'zgarishi = yangi versiya + staff audit. Har cycle ishlatgan price'ga FK + amount snapshot.
- **Status:** ACCEPTED (owner review)

### TD-52 — Entitlements: generic PlanEntitlement + normalized cycle snapshot + resolver
- **Decision:** Plan jadvaliga limit ustunlar emas — PlanEntitlement(plan, feature_code registry, limit_value); Entitlement Resolver policy bo'yicha snapshot yoki live'dan o'qiydi.
- **Clarification (owner):** cycle entitlement snapshot'i **opaque JSON blob EMAS** — normalized immutable model: `SubscriptionCycleEntitlement (cycle, feature_code, granted mode/limit)` yoki teng keluvchi. Sabab: usage checks, audit, feature lookup, historical entitlement query'lari oson. Plan entitlement o'zgarsa oldingi cycle snapshot rewrite qilinmaydi. Entitlement change joriy subscriberga darholmi yoki keyingi cycle'dami — **PRODUCT POLICY OPEN**. Exact schema — implementation bosqichida.
- **Status:** ACCEPTED (owner review; qo'llash policy'si OPEN)

### TD-53 — Usage metering: UsageCounter + domain-entity-bound consumption
- **Decision:** UsageCounter unique(cycle, feature_code); consumption tegishli domain yozuvi (masalan AiEvaluation) bilan bir tranzaksiyada — retry-safe. Alohida UsageRecord jadvali va reserve/release apparati MVP'da yo'q (kichik overshoot — qabul qilinadigan trade-off, monitoring bilan). Provider cost tracking user entitlement bilan aralashtirilmaydi.
- **Status:** ACCEPTED (owner review)

### TD-54 — Subscription + SubscriptionCycle markaziy modeli
- **Decision:** Subscription minimal statuslar (ACTIVE/EXPIRED/CANCELLED; PENDING yo'q — activation faqat verified payment bilan); bitta user'da bitta non-terminal subscription (partial unique). SubscriptionCycle — markaziy iqtisod entity'si: unique(subscription, sequence_no), price/ceiling/rate/policy/entitlement snapshotlari, earned_izl, UsageCounter'lar, payment_order bog'lanishi. SubscriptionChange — append-only plan o'zgarish audit'i (upgrade/downgrade policy OPEN). Expiry behavior — Entitlement Resolver policy'si (OPEN), model bog'lanmagan.
- **Status:** ACCEPTED (owner review)

### TD-55 — PaymentOrder ≠ PaymentTransaction + provider port
- **Decision:** PaymentOrder — business intent (purpose, gross/izl_discount/payable snapshot, status); PaymentTransaction — provider tranzaksiyasi (unique(provider, provider_transaction_id), canonical status, sanitized provider_metadata; raw payload faqat cheklangan-retention operational logda). Provider-specific narsalar faqat payment jadvallarida — Click/Payme adapterlar port ortida.
- **Status:** ACCEPTED (owner review)

### TD-56 — Callback idempotency + trusted-path activation
- **Decision:** PaymentCallbackEvent unique(provider, provider_event_id) — duplicate callback DB'da to'xtaydi; processing bitta atomik tranzaksiya: verify → event insert → transaction/order transition → subscription/cycle activation (+ redemption APPLIED). Frontend success sahifasi hech qachon activate qilmaydi — faqat status poll.
- **Status:** ACCEPTED (owner review)

### TD-57 — Redemption: reservation-first model (owner tomonidan o'zgartirilgan)
- **Decision (accepted model — dastlabki "darhol REDEEM + avto-REVERSAL" proposal'i o'rniga):**
  1. **Reservation:** checkout boshlanishida IZLRedemption `REQUESTED → RESERVED`; wallet'da spendable IZL reservation miqdoriga kamayadi (reserved_amount), lekin **actual ledger debit hali yozilmaydi** (misol: total 500, reserved 100 → spendable 400).
  2. **Payment success:** trusted confirmation tranzaksiyasi ichida `RESERVED → APPLIED`: reserved kamayadi + **REDEEM ledger entry** yaratiladi + total balance kamayadi.
  3. **Payment failure/expiry/cancellation:** `RESERVED → RELEASED/CANCELLED` — reserved bo'shatiladi; ledger REVERSAL **kerak emas** (actual REDEEM bo'lmagan).
  4. **Already-applied reversal:** REDEEM final bo'lgach business reversal/refund kerak bo'lsa — yangi REVERSAL ledger entry; oldingi entry o'zgartirilmaydi.
  - Bu pending checkout va real financial movementni ajratadi. IZLRedemption type enum — MVP: SUBSCRIPTION_DISCOUNT (cash-out/game currency — kelajak kengayishi). Order/reservation expiry exact policy va cleanup — implementation/queue bosqichida.
- **Status:** ACCEPTED (owner review)

### TD-58 — IzlRateVersion + snapshot strategiyasi
- **Decision:** `1 IZL = X UZS` — versioned config (IzlRateVersion); rate ishlatilgan har joy (SubscriptionCycle, IZLRedemption, PaymentOrder summalari) o'z snapshot'ini saqlaydi — rate o'zgarsa tarix buzilmaydi. Rate qiymati va o'zgarishi mumkinligi — OPEN product qarori (model ikkalasiga mos).
- **Status:** ACCEPTED (owner review)

### TD-59 — Financial concurrency: row lock + atomik tranzaksiya + unique backstop
- **Decision:** Grant/redeem/adjustment — wallet qatori lock; ceiling tekshiruvi — cycle qatori lock; RewardGrant+Ledger+Wallet+Cycle bitta tranzaksiyada; callback processing atomik; unique constraintlar (dedup_key, provider ids, sequence_no) backstop.
- **Status:** ACCEPTED (owner review)

### TD-60 — Financial retention: append-only, destructive delete yo'q
- **Decision:** Ledger/grants/redemptions/payments/cycles hech qachon hard-delete qilinmaydi; refund history qo'shadi, o'chirmaydi; user erasure — anonymization/pseudonymization yo'nalishi (legal policy OPEN, invent qilinmaydi); minors parental-consent jadvallari qo'shilmaydi (extension point).
- **Status:** ACCEPTED (owner review)

## Phase 1.2C-2 — Community/Announcements/Notifications

Batafsil tahlil: [DATA_MODEL_COMMUNITY.md](DATA_MODEL_COMMUNITY.md). Product Owner review yakunlandi: **TD-61..TD-80 hammasi ACCEPTED** (TD-65/67/74/77 — quyidagi clarificationlar bilan). **Phase 1.2C-2 COMPLETE.**

### TD-61 — Post/Reply: alohida entitylar, flat replies
- **Decision:** CommunityPost va CommunityReply alohida (self-referencing bitta jadval emas — maydonlar farqli, moderation aniqroq); replies flat (nested tree yo'q — parent_reply_id keyin additive); post type — DB enum (5 accepted type stable).
- **Status:** ACCEPTED (owner review)

### TD-62 — Body format: plain text
- **Decision:** Post/reply body — plain text, render'da escape (XSS yo'q); markdown-subset kelajakda rendering qatlamida (storage o'zgarmaydi); HTML/rich JSON rad.
- **Status:** ACCEPTED (owner review)

### TD-63 — Subject/Topic association: optional 1+1
- **Decision:** Post'da optional bitta subject_id + optional topic_id (topic → subject invariant); N:M tagging va freeform hashtag yo'q (MVP).
- **Status:** ACCEPTED (owner review)

### TD-64 — Accepted Answer: post'da accepted_reply_id
- **Decision:** Alohida entity emas — CommunityPost.accepted_reply_id + invariantlar (faqat QUESTION, faqat shu post reply'si, faqat author, hidden reply emas); almashtirish tarixi saqlanmaydi (reputation korreksiyasi event modelида).
- **Status:** ACCEPTED (owner review)

### TD-65 — Reactions: seeded ReactionType + XOR-target
- **Decision:** ReactionType seeded/configured jadval (ro'yxat final emas — enum'ga lock qilinmaydi); CommunityReaction post_id XOR reply_id (polymorphic va alohida ikki jadval rad); unique(user, target, type) — har turdan bittadan; olib tashlash = delete.
- **Clarification (owner):** reaction **code/history semantikasi immutable** — mavjud ReactionType qatorining semantic ma'nosi o'zgartirilmaydi; **display label** o'zgarishi mumkin; yangi reaction qo'shilishi mumkin; eski reaction `is_active=false` orqali yangi foydalanishdan chiqariladi; existing reaction history saqlanadi.
- **Status:** ACCEPTED (owner review)

### TD-66 — Community media: MediaAsset reuse + junction
- **Decision:** CommunityPostMedia(post, media_asset, position); mime allowlist — faqat image/audio, video strukturaviy rad; limitlar config (OPEN); MediaAsset Community'ga hard-code qilinmaydi; replies media MVP'da yo'q.
- **Status:** ACCEPTED (owner review)

### TD-67 — Visibility/removal
- **Decision:** VISIBLE / AUTHOR_REMOVED / MODERATOR_HIDDEN semantic removal model; normal operationda hard delete yo'q (bog'liq tarix buzilmaydi); report paytida content_snapshot moderation kontekstini himoya qiladi.
- **Clarification (owner):** post/reply **editing capability va exact editing policy — PRODUCT OPEN**, ACCEPTED emas. Model `edited_at`/`content_snapshot` kabi future supportga tayyor bo'lishi mumkin, lekin bu editing policy qabul qilingani degani emas. Full revision history MVP uchun talab qilinmaydi.
- **Status:** ACCEPTED (owner review — faqat visibility/removal qismi; editing OPEN)

### TD-68 — Report modeli
- **Decision:** CommunityReport: XOR target + category_code (string registry — taksonomiya final emas) + free_text + content_snapshot + OPEN/ACTIONED/DISMISSED; unique(reporter, target) — duplicate report yo'q.
- **Status:** ACCEPTED (owner review)

### TD-69 — Moderation: append-only ModerationAction, Case'siz MVP
- **Decision:** ModerationAction append-only domain tarixi (actor, action_type, target, report ref, reason); ModerationCase MVP'da yo'q (reportlar bevosita, content bo'yicha guruhlab ko'rsatiladi); StaffAudit bilan ajratilgan — bitta jadvalga tiqilmaydi.
- **Status:** ACCEPTED (owner review)

### TD-70 — CommunityRestriction
- **Decision:** Temporary restriction entity (user, starts/expires, reason, created_by, revoked) — account SUSPENDED'dan alohida; active restriction community write'larini bloklaydi.
- **Status:** ACCEPTED (owner review)

### TD-71 — Reputation: balance cache + append-only event
- **Decision:** ReputationBalance (1:1 cache) + ReputationEvent (amount ±, reason_code registry, source refs, unique(user, dedup_key)); formula engine/config'da (OPEN); korreksiya manfiy event bilan; IZL darajasidagi locking/reversal apparati yo'q.
- **Status:** ACCEPTED (owner review)

### TD-72 — Community → XP/IZL boundary
- **Decision:** Community domain XP'ni faqat Gamification service orqali beradi (bevosita XpBalance mutate yo'q); oddiy community faoliyati IZL bermaydi — hech qachon; mission holatida CommunityPost faqat evidence, reward faqat Reward Engine (DailyPlanItem + evidence) orqali; community wallet/ledgerga yozmaydi.
- **Status:** ACCEPTED (owner review)

### TD-73 — Feed/counters
- **Decision:** Persisted Feed jadvali yo'q — query/ranking view (subject/topic + recency + visibility); post'da cached reply_count/reaction_count (tranzaksiyada, davriy reconcile bilan — financial emas); event-sourced/social-graph feed rad.
- **Status:** ACCEPTED (owner review)

### TD-74 — Media moderation: processing ≠ moderation status
- **Decision:** MediaAsset'da ikki alohida o'lchov: **processing_status** (upload/process readiness) va **moderation_status** (safety/moderation decision); public = ready + not blocked; automated moderation natijasi provider-neutral metadata'da.
- **Clarification (owner):** pre-moderation vs post-moderation policy, automated moderator authority chegarasi va **exact state nomlari — OPEN/implementation detail** (hujjatdagi PENDING/READY/UNREVIEWED/BLOCKED — conceptual misollar). **AI/provider hech qachon business authority bo'lmaydi** — final moderation qarori human authority'da.
- **Status:** ACCEPTED (owner review — ajratish prinsipi; nomlar/policy OPEN)

### TD-75 — Announcement modeli
- **Decision:** Announcement: DRAFT/PUBLISHED/ARCHIVED + publish_at (scheduled-ready; queue OPEN) + expires_at + audience_type=ALL (segmentatsiya — additive kelajak, rules engine yo'q); created_by + publish audit.
- **Status:** ACCEPTED (owner review)

### TD-76 — AnnouncementUserState: lazy sparse junction
- **Decision:** unique(announcement, user) + read_at/dismissed_at; row faqat birinchi interaksiyada yaratiladi — pre-fanout yo'q.
- **Status:** ACCEPTED (owner review)

### TD-77 — Notification core: in-app record
- **Decision:** Notification: type (string code registry — minimal set OPEN), rendered title/body snapshot, read_at; channel-independent core hozir modellanadi.
- **Clarification (owner):** source model ko'p typed nullable FK ustunlari bilan **kengaytirilmaydi** — lightweight polymorphic **`source_type + source_id`** reference ishlatiladi. Notification financial/audit source of truth **emas**; original source yo'qolsa/archived bo'lsa ham rendered title/body snapshot notificationni historical UX record sifatida tushunarli saqlaydi. Retry-safe dedup_key bilan ishlaydi (TD-78).
- **Status:** ACCEPTED (owner review)

### TD-78 — Notification dedup + delivery abstraction
- **Decision:** unique(user, dedup_key) — retry duplicate yaratmaydi, aggregation'ga tayyor kalit; NotificationDelivery entity channel tanlangach additive qo'shiladi (hozir yaratilmaydi); provider hard-code yo'q; SMS OTP notification tizimi emas.
- **Status:** ACCEPTED (owner review)

### TD-79 — Community idempotency/concurrency
- **Decision:** Unique constraintlar (reaction/report/user-state/notification/reputation dedup) + status transition guardlar (report resolve — birinchi yutadi; accepted answer — owner last-write; hidden content'ga edit blok); full optimistic-lock framework yo'q.
- **Status:** ACCEPTED (owner review)

### TD-80 — Community retention/deletion
- **Decision:** Hard delete o'rniga visibility model; reports/ModerationAction audit uchun saqlanadi (destructive delete yo'q); user erasure — anonymization/pseudonymization extension (legal OPEN, exact retention invent qilinmaydi).
- **Status:** ACCEPTED (owner review)

## Phase 1.2D — Pre-Schema Integrity Review

Batafsil tahlil: [PRE_SCHEMA_REVIEW.md](PRE_SCHEMA_REVIEW.md), [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md), [JSONB_GOVERNANCE.md](JSONB_GOVERNANCE.md). Product Owner review yakunlandi: **TD-81..TD-92 hammasi ACCEPTED** (TD-82/83/84/86/87/89/91 — quyidagi clarificationlar bilan). Amendment mapping: TD-83→TD-32/33, TD-84→TD-48, TD-85→TD-51, TD-86→TD-52, TD-87→TD-54, TD-88→TD-37 — asl TD matnlari tarix sifatida saqlanadi, bu bo'lim ularning refinement'i. **Phase 1.2D COMPLETE.**

### TD-81 — StaffAudit: append-only accountability entity (yangi; P0)
- **Decision:** StaffAudit: actor_user_id, action_code (registry), target_type+target_id (lightweight polymorphic — business truth emas, shuning uchun to'g'ri), reason?, metadata JSONB, created_at; domain amal bilan bir tranzaksiyada. SecurityEvent/Ledger/ModerationAction/Payment bilan QO'SHILMAYDI; universal log emas. D-35'ning yopilmagan qismini (masalan rol o'zgarishlar tarixi) yopadi.
- **Status:** ACCEPTED (owner review)

### TD-82 — Media relational FK qatlami (yangi; P0; TD-21/25 saqlanadi, TD-25 refine qilinadi)
- **Decision:** ActivityMedia(activity, media_asset, role_code?, position), AssessmentItemMedia(item, media_asset), CommunityPostMedia (mavjud) — relational truth; ActivityAttempt/AssessmentResponse'ga `response_media_asset_id` nullable FK (MVP: bitta canonical response media; ko'p asset kerak bo'lsa junction additive). MediaAsset referenced → hard delete RESTRICT.
- **Clarification (owner) — source-of-truth rule:** JSONB va relational jadval bir xil media_asset_id'ni **ikki authoritative source sifatida dublikat qilmaydi**. Final semantika: payload — faqat rendering/configuration (caption, displayMode, **role/slot semantikasi** — raw MediaAsset UUID emas); actual asset association — faqat junction. Junction: unique(parent, media_asset, role_code?) — exact uniqueness schema bosqichida; position int; role_code — closed registry (kerak bo'lsa).
- **Status:** ACCEPTED (owner review)

### TD-83 — AMENDMENT TO TD-32/TD-33: AssessmentDefinitionVersion + AssessmentVersionItem (P0)
- **Decision:** AssessmentDefinition — logical stable identity + `current_version_id` (published pointer). **AssessmentDefinitionVersion** — append-only: version_no, config, lifecycle, created_by, published_at; publish bo'lgach immutable. AssessmentItem — immutable item identity (logical definition scope'ida). **AssessmentAttempt → definition_version_id**; AssessmentResponse → exact item.
- **Clarification (owner) — original proposal yetarli emas edi:** faqat config emas, **ITEM POOL MEMBERSHIP ham versionlanadi**: **AssessmentVersionItem** (assessment_definition_version_id, assessment_item_id, unique(version, item), faqat chindan version-specific bo'lsa ordering/calibration override). Reproducibility zanjiri: attempt → version → exact config + exact eligible item pool + response sequence + engine_version. Misol: 2026 v3 = {A,B,C,D}, 2027 v4 = {A,C,E,F} — eski attempt qaysi pooldan ishlagani aniq. Draft version membership tahrirlanadi; published bo'lgach membership qatorlari ham immutable.
- **Status:** ACCEPTED (owner review)

### TD-84 — AMENDMENT TO TD-48: DailyMissionCompletion + Evidence child records (P0)
- **Decision:** Ajratish: DailyPlanItem = mission assignment/plan snapshot; **DailyMissionCompletion** = append-only "mission bajarildi" business evidence (user, daily_plan_item_id **unique** — one-shot semantika, completed_at, kerak bo'lsa completion_type); **RewardGrant** = financial decision (`daily_mission_completion_id` FK — endi CommunityPost'ni to'g'ridan-to'g'ri bilishi shart emas). Oqim: PlanItem → Completion → Reward Engine → Grant → Ledger.
- **Clarification (owner) — evidence bitta nullable FK to'plamiga cheklanmaydi:** **DailyMissionCompletionEvidence** (1:N child, append-only): completion_id + typed nullable FK'lar (community_post_id / activity_attempt_id / learning_session_id; kelajak turlari additive) — **har evidence qatorida roppa-rosa bitta typed FK (XOR)**. Polymorphic source_type/id ISHLATILMAYDI — financial rewardga boradigan evidence FK integrity talab qiladi. MVP'da bitta evidence = bitta qator; "15 daqiqa" tipidagi mission bir nechta session/attempt evidence qatori bilan isbotlanishi mumkin. Exact completion rule — Reward/Learning engine'da.
- **Status:** ACCEPTED (owner review)

### TD-85 — AMENDMENT TO TD-51: PlanPrice faqat effective_from (P1)
- **Decision:** effective_to olib tashlanadi; PlanPrice = id, plan_id, currency, amount, effective_from, created_by, created_at — row'lar 100% immutable, UPDATE umuman yo'q. Joriy narx = effective_from ≤ now bo'lgan eng yangi row; unique(plan, currency, effective_from). Cycle baribir plan_price_id + gross snapshot saqlaydi — current lookup tarixni o'zgartirmaydi. Future scheduled-price correction/cancellation admin policy'si — implementation bosqichida; ishlatilgan tarixiy rowlar hech qachon mutable bo'lmaydi. Sotuvni to'xtatish — plan status orqali.
- **Status:** ACCEPTED (owner review)

### TD-86 — AMENDMENT TO TD-52: Entitlement mode semantikasi (P1)
- **Decision:** EntitlementMode enum: DISABLED / ENABLED / LIMITED / UNLIMITED; PlanEntitlement va SubscriptionCycleEntitlement — **bir xil kontrakt** (plan/cycle_id, feature_code, mode, limit_value nullable).
- **Clarification (owner) — CHECK semantika:** LIMITED → limit_value NOT NULL va ≥ 0; DISABLED/ENABLED/UNLIMITED → limit_value IS NULL. Feature registry (application-side): feature_code + value_kind (BOOLEAN / COUNT_LIMIT); **registry semantikasi immutable** — feature_code ma'nosi keyin boshqa featurega aylantirilmaydi. Mode↔value_kind mosligi (BOOLEAN → ENABLED/DISABLED; COUNT_LIMIT → LIMITED/UNLIMITED/DISABLED) — application validation. Generic enterprise feature-flag platforma yaratilmaydi.
- **Status:** ACCEPTED (owner review)

### TD-87 — AMENDMENT TO TD-54: Subscription identity semantikasi (P1)
- **Decision (owner explicit lifecycle):** Subscription = har payment uchun record EMAS — **userning bitta davomli subscription episode/container'i**; har verified payment/renewal → yangi SubscriptionCycle. Semantika:
  - **ACTIVE** — joriy paid cycle/access mavjud;
  - **EXPIRED** — paid cycle tugagan, lekin episode **terminal EMAS**; renewal → **SHU Subscription ichida** yangi cycle → ACTIVE;
  - **CANCELLED** — terminal episode; keyin yangidan obuna → **NEW Subscription record**.
  - Bir userda ko'pi bilan bitta non-terminal (ACTIVE yoki EXPIRED) Subscription; tarixiy CANCELLED subscriptionlar saqlanadi.
- **Status:** ACCEPTED (owner review)

### TD-88 — AMENDMENT TO TD-37: LearnerLessonCompletion history (P1)
- **Decision:** **LearnerLessonProgress** — unique(user, lesson), CURRENT learning state: current pinned revision, progress/resume point, current mastery cache (mutable/derived). **LearnerLessonCompletion** — append-only history: user_id, lesson_id, lesson_revision_id, **completion_no** (unique(user, lesson, completion_no)), completed_at, foydali bo'lsa started_at/mastery snapshot. Oqim: 2026'da v3 complete → Completion #1 (v3); keyin relearn → Progress reset (v8 pin) → complete → Completion #2 (v8) — v3 tarixi yo'qolmaydi. ActivityAttempt'lar raw evidence bo'lib qolaveradi.
- **Status:** ACCEPTED (owner review)

### TD-89 — Numeric width / score representation policy (yangi; TD-46 doirasida)
- **Decision (owner final policy):** Prinsip — **"smallest safe integer width; BIGINT by habit emas"**. Bounded per-user/per-order qiymatlar (UZS amounts, IZL, XP, counters, active_seconds, usage) — Prisma Int / PG INTEGER, **schema'da domain CHECK/max bounds bilan**; 2^31'ga real yaqinlashishi mumkin bo'lgan field bo'lsa — BIGINT (field-by-field, system-wide high-volume counterlar alohida ko'riladi). Score/mastery/confidence (pul emas): normalized fixed integer — **basis points 0..10000** (`mastery_score_bp`, `confidence_bp`; 63.25% → 6325); AI rubric chindan decimal precision talab qilsa AiEvaluation.score uchun bounded Decimal/numeric mumkin (exact scale schema'da consistency bilan). **Float pul uchun hech qachon.**
- **Status:** ACCEPTED (owner review)

### TD-90 — Enum vs seeded-table vs code-registry policy (yangi)
- **Decision:** DB enum — faqat yopiq, kod bilan o'zgaradigan state'lar (statuslar, ActivityType, LedgerEntryType...); seeded jadval — runtime boshqariladigan + FK kerak (Role, ReactionType, SubscriptionPlan); code registry string — o'suvchi kodlar (permission, feature, reason/category, notification type, audit action). Product-tunable narsa hech qachon DB enum emas.
- **Status:** ACCEPTED (owner review)

### TD-91 — local_date/timezone snapshot semantikasi (yangi)
- **Decision:** UserProfile.timezone — current preference/authority; **core learning boshlanishidan oldin tizimda resolved valid IANA timezone bo'lishi shart** (bo'lmasa system default: Asia/Tashkent). DailyPlan: local_date + `timezone_snapshot`; WeeklyProgress: week_start_local_date (+kerak bo'lsa timezone_snapshot). Daily reward eligibility — tarixiy DailyPlan local_date/tz snapshot orqali. Tz keyin o'zgarsa (masalan Asia/Tashkent → Europe/Berlin) eski yozuvlar **qayta hisoblanmaydi** — yangi plan/period yangi tz ishlatadi.
- **Clarification (owner):** tz o'zgarishi midnight/day-boundary abuse yoki duplicate daily eligibility yaratmasligi uchun Reward Engine joriy cycle/day eligibility'ni mavjud tarixiy DailyPlan/RewardGrant evidence bilan tekshiradi. Exact timezone-change product policy — implementation'da.
- **Status:** ACCEPTED (owner review)

### TD-92 — JSONB governance (yangi)
- **Decision:** [JSONB_GOVERNANCE.md](JSONB_GOVERNANCE.md) qoidalari: JSONB FK o'rnini bosmaydi; A-sinf payloadlar strict validation + schema_version; evidence JSONB parent bilan muzlaydi; katta blob/raw provider payload biznes jadvalida emas; GIN indexlar faqat isbotlangan ehtiyoj bilan.
- **Status:** ACCEPTED (owner review)

## Ochiq texnik qarorlar (hali ACCEPTED emas)

- SMS provider tanlovi.
- JWT signing algoritmi va key management.
- Access token web'da cookie vs client-memory — final tanlov.
- CSRF mexanizmining aniq shakli.
- Exact TTL/limit qiymatlari (tuning).
- Object storage, queue/background jobs, AI provider strategy, deployment, mobile stack, analytics, notification provider — oldingidek OPEN ([OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)).
