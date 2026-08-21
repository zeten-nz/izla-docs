# Izlan — Phase 1.2D: Pre-Schema Integrity & Constraint Review

> Status: Phase 1.2D — **COMPLETE** (owner review 2026-08-27 yakunlandi). **TD-81..TD-92 hammasi ACCEPTED** — owner modifikatsiyalari bilan: AssessmentVersionItem membership snapshot (TD-83), media junction'da role/slot semantikasi — payload'da raw UUID yo'q (TD-82), DailyMissionCompletionEvidence 1:N child qatorlar (TD-84), subscription episode semantikasi (TD-87), completion_no (TD-88), "smallest safe integer width" (TD-89), resolved IANA tz talabi (TD-91).
> Bu hujjatdagi tahlil/variantlar tarix sifatida saqlanadi; final holat — [TECH_DECISIONS.md](TECH_DECISIONS.md) va domain data-model hujjatlari.
> Yo'ldosh hujjatlar: [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) (ACCEPTED artifact), [JSONB_GOVERNANCE.md](JSONB_GOVERNANCE.md) (TD-92 ACCEPTED).

## 1. Executive summary

Barcha 4 domain data modeli (CORE, LEARNING, FINANCE, COMMUNITY) yonma-yon ko'rib chiqildi. Arxitektura umumiy holda izchil va schema'ga yaqin, LEKIN **4 ta P0** (schema'dan oldin majburiy) va **5 ta P1** muammo topildi:

**P0:** (1) StaffAudit entity kataloglarning birortasida yo'q — D-35 talabi to'liq qoplanmagan; (2) MediaAsset referencelari 3 domainda faqat JSONB ichida — FK integrity yo'q; (3) AssessmentDefinition config'i mutable — attempt reproducibility sinadi; (4) Daily Mission evidence zanjiri (ayniqsa community missionlar uchun) uzuq.

**P1:** PlanPrice "immutable" vs `effective_to` update ziddiyati; PlanEntitlement `limit_value=null` semantikasi noaniq; Subscription identity (per-purchase vs long-lived) hujjatda explicit emas; lesson relearning tarixi unique(user, lesson) bilan bloklanadi; auth parallel-refresh false-positive reuse.

**Verdict (owner review'dan keyin yangilangan): READY** — TD-81..TD-92 ACCEPTED, barcha P0/P1 yopildi (§23).

## 2. Review scope

TD-21..TD-30 (core), TD-31..TD-44 (learning), TD-45..TD-60 (finance), TD-61..TD-80 (community) + barcha product/architecture hujjatlari. Tekshiruv o'qlari: relational integrity, immutable history, auditability, constraint enforceability, Prisma feasibility, documentation consistency.

## 3. Source-of-truth rules

Owner clarificationlari > ACCEPTED TD > ACCEPTED product decisions > data model docs > conceptual docs > inference. Stale conceptual matn accepted TD'ga zid bo'lsa — TD g'olib, matn cleanup ro'yxatiga kiradi (§ hujjat oxirida, bajarilgan cleanuplar bilan).

## 4. P0 issues

| # | Issue | TD |
|---|---|---|
| P0-1 | StaffAudit entity mavjud emas (§7) | TD-81 (yangi) |
| P0-2 | Media referencelari JSONB ichida — FK yo'q (§8) | TD-82 (yangi) |
| P0-3 | Assessment config reproducibility (§9) | TD-83 (AMENDMENT TD-32/33) |
| P0-4 | Daily Mission completion evidence (§10) | TD-84 (AMENDMENT TD-48) |

Nega P0: har biri production data kelgandan keyin tuzatilsa — JSONB'dan backfill, tarixiy yozuvlarni qayta bog'lash yoki isbotlab bo'lmaydigan moliyaviy grantlar degani.

## 5. P1 issues

| # | Issue | TD |
|---|---|---|
| P1-1 | PlanPrice immutability ziddiyati (§11) | TD-85 (AMENDMENT TD-51) |
| P1-2 | Entitlement null semantikasi (§12) | TD-86 (AMENDMENT TD-52) |
| P1-3 | Subscription identity semantikasi (§13) | TD-87 (AMENDMENT TD-54) |
| P1-4 | Lesson relearning/completion history (§14) | TD-88 (AMENDMENT TD-37) |
| P1-5 | Parallel refresh false-positive reuse (§ quyida, auth note) | Implementation note (TD emas) |

**P1-5 — Auth refresh concurrency:** ikki tab parallel refresh qilsa: A eski tokenni rotate qiladi; B o'sha eski token bilan keladi → reuse-detection sessionni revoke qiladi (false positive, theft emas). Bu schema qarori emas — AUTH_ARCHITECTURE.md §8'ga implementation note qo'shildi (accepted qarorlar o'zgartirilmagan): rotation atomic transaction; client single-flight refresh tavsiyasi; qisqa grace strategiyasi varianti (security trade-off bilan) — exact policy OPEN/implementation. Schema'ga yengil ta'sir: RefreshToken'da `replaced_by` reference grace uchun foydali bo'lishi mumkin — implementation detail.

## 6. P2 implementation notes

1. C4/C11 renumbering: `(parent, sort_order)` unique constraint reorder paytida to'qnashadi — yoki DEFERRABLE unique (raw SQL), yoki app-side two-phase renumber. Schema'da hal qilinadi, blocker emas.
2. UsageCounter overshoot monitoring (TD-53da qayd etilgan) — implementation.
3. OtpChallenge/expired RefreshToken TTL cleanup joblari — queue tanloviga bog'liq.
4. Reconciliation joblar: wallet vs ledger SUM; community counterlar.
5. AiEvaluation → UsageCounter bog'liqligi FINANCE'da bor, LEARNING hujjatida eslatilmagan — hujjat izohi (schema ta'siri yo'q).

## 7. StaffAudit review (P0-1 → TD-81)

**Topilma:** D-35 "sensitive staff actions audit qilinadi" — lekin StaffAudit hech bir entity katalogida yo'q. FINANCE §31 va COMMUNITY §28 unga *tizim sifatida* ishora qiladi, model esa yo'q. Hozirgi qoplama qismli: IZL ADJUSTMENT (actor+reason ledgerda), ModerationAction, SubscriptionChange, PlanPrice.created_by, LessonRevision.published_by, UserRole.granted_by, SubjectAssignment.assigned_by, SecurityEvent(account_suspended). Lekin: payment correction, pricing change, entitlement change, user suspension (staff nuqtai nazaridan), role/permission o'zgarishi tarixiy log sifatida — yagona accountability manba yo'q; domain maydonlari (granted_by...) faqat oxirgi holatni ko'rsatadi, tarixni emas.

**Concrete failure:** Admin userning rolini METHODIST'dan olib tashladi, keyin qaytardi — UserRole faqat joriy holat; "kim, qachon, nega olib tashlagan" hech qayerda yo'q.

**Proposal (TD-81):** append-only **StaffAudit**: id, actor_user_id (FK → User), action_code (string registry — masalan `pricing.change`, `user.suspend`, `role.grant`), **target_type + target_id (lightweight polymorphic)**, reason nullable, metadata JSONB (minimal), created_at. Domain harakat bilan **bir tranzaksiyada** yoziladi.

- **Polymorphic bu yerda to'g'ri:** targetlar barcha domainlarni qamraydi (FK ustunlar to'plami enumeration'ga sig'maydi); StaffAudit business truth EMAS (domain jadvallar truth bo'lib qoladi) — Notification (TD-77) bilan bir xil asos; target keyin arxivlansa ham audit yozuvi yashashi kerak — FK RESTRICT aksincha zarar qilardi.
- **Chegara:** SecurityEvent (auth/security), IZL Ledger (qiymat), ModerationAction (community domain tarixi), PaymentTransaction (provider truth) bilan QO'SHILMAYDI; universal "everything log" emas — faqat staff administrative amallar.
- Schema impact: bitta yangi jadval; USER_ROLES.md'dagi "exact audit architecture keyin" matni shu proposalga bog'landi (accepted deb yozilmadi).

## 8. Media relation integrity (P0-2 → TD-82)

**Topilma:** TD-25 MediaAsset bor, lekin referencelar: Activity payload JSONB (`media_asset_id`), AssessmentItem payload (implied), ActivityAttempt/AssessmentResponse speaking answer payload. Faqat CommunityPostMedia — haqiqiy junction. JSONB ichidagi UUID: FK enforce qilmaydi, RESTRICT/CASCADE ishlamaydi, orphan detection va reverse-usage query og'ir, C-17/I-15/F darajasidagi "referenced media o'chirilmaydi" invarianti faqat application'da.

**Concrete failure:** Methodist audio'ni almashtirdi, eski MediaAsset'ni kim ishlatayotganini bilish uchun BARCHA activity payloadlarni scan qilish kerak; storage cleanup jobi xato asset o'chirsa published lesson sinadi — DB hech narsa demaydi.

**Variantlar:** A (faqat JSONB) — zaif; C (generic polymorphic MediaReference) — FK yo'qoladi; **B/D (tanlangan)** — TD-21 payload saqlanadi, relational qatlam qo'shiladi:

**Proposal (TD-82):**

- **ActivityMedia** (activity_id, media_asset_id, role nullable, position nullable; unique(activity_id, media_asset_id)) — content bloklari uchun.
- **AssessmentItemMedia** (item_id, media_asset_id; unique pair).
- **Learner javob media:** junction ortiqcha — 1:0..1: ActivityAttempt va AssessmentResponse'ga **`response_media_asset_id` nullable FK** ustuni (speaking audio).
- CommunityPostMedia — o'zgarishsiz (allaqachon to'g'ri model).

**Drift himoyasi (dual source-of-truth):** relational qator — "bu content shu asset'ni ishlatadi" faktining truth'i; payload — placement/rendering. Yozish bitta tranzaksiyada; TD-22 validation payload'dagi har media_asset_id uchun mos junction qator borligini publish/submitda tekshiradi (invariant M-C17b). MediaAsset delete endi haqiqiy FK RESTRICT bilan himoyalanadi. TD-21 bekor qilinmaydi — payload strukturasi joyida.

## 9. Assessment reproducibility (P0-3 → TD-83, AMENDMENT TO TD-32/TD-33)

**Topilma:** AssessmentDefinition'da `version` int + `config` JSONB mutable row'da. Attempt definition'ga FK qiladi. 2026'da version=3/config=X bilan topshirilgan attempt, 2027'da row version=4/config=Y bo'lsa — attempt qaysi config bilan ishlagani **yo'qoladi**. Bu accepted TD-33 reproducibility maqsadiga zid. Itemlar allaqachon immutable (yaxshi), presented itemlar Response'larda (yaxshi) — uzilish faqat **config**da.

**Variantlar:** A (definition row immutable, har versiya yangi row) — item pool har versiyada qayta bog'lanishi kerak, og'ir; B (to'liq Lesson-style Definition+Revision, itemlar revisionga) — item pool dublikatsiyasi, overengineering; C (attempt'da config snapshot JSONB) — ishlaydi, lekin config auditi/versiyalar ro'yxati yo'qoladi, har attemptda dublikat; **D (tanlangan) — B-lite:**

**Proposal (TD-83):** append-only **AssessmentDefinitionVersion**: id, definition_id, version (unique per definition), config JSONB (immutable), status, created_by, created_at. Definition (logical) `current_version_id` pointer'iga ega; publish = yangi version row + pointer swap (LessonRevision patterni, lekin itemlar logical definitionda qoladi — dublikatsiz). **AssessmentAttempt `definition_version_id` FK oladi** (definition_id denormalized qoladi). Reproducibility to'liq: config (version row) + itemlar (immutable, Response'larda) + engine_version (mavjud). AssessmentSection qo'shilmadi (kerak emas).

## 10. Reward / Daily Mission evidence (P0-4 → TD-84, AMENDMENT TO TD-48)

**Topilma:** RewardGrant(DAILY_MISSION) → daily_plan_item_id. Lekin: (1) DailyPlanItem.status mutable — "bajarilgan" fakti uchun immutable evidence emas; (2) community mission holatida qaysi CommunityPost missionni qanoatlantirgani hech qayerda yozilmaydi (COMMUNITY C-8 "post faqat evidence" deydi, lekin bog' modelda yo'q).

**Concrete failure:** 6 oy o'tib audit: "15 IZL nima uchun?" → grant plan itemga ishora qiladi; item COMPLETED — lekin qaysi post bilan, mission sharti haqiqatan bajarilganmi — isbotlab bo'lmaydi.

**Variantlar:** A (faqat plan item) — yetarli emas; B (RewardGrant'ga community_post_id qo'shish) — mission kelajak turlariga (session, review, speaking) har safar ustun; D (generic polymorphic) — FK yo'qoladi; **C (tanlangan):**

**Proposal (TD-84):** append-only **DailyMissionCompletion**: id, user_id, daily_plan_item_id FK (**unique** — bir itemga bitta completion), evidence typed nullable FK'lar: community_post_id / activity_attempt_id / learning_session_id (kamida bittasi, mission turiga mos — CHECK/app), completed_at. **RewardGrant(DAILY_MISSION) evidence FK'si endi `daily_mission_completion_id`** (TD-48 mapping amendmenti). Semantika: **completion = immutable bajarilganlik evidence'i** (reward bo'lmasa ham yoziladi — masalan ceiling tugagan); **RewardGrant = eligibility/decision record** (TD-48 bo'yicha). Idempotency: unique(daily_plan_item_id) + mavjud dedup_key — double-processing imkonsiz. Kelajak mission turlari FK ustun qo'shish bilan kengayadi (product qarori bilan birga — normal).

## 11. Pricing immutability (P1-1 → TD-85, AMENDMENT TO TD-51)

**Ziddiyat (o'zim yozgan):** "tarixiy qator hech qachon UPDATE qilinmaydi" VS "yangi versiya chiqqanda eski qatorda effective_to yopiladi" — effective_to yozish ham UPDATE.

**Variantlar:** B (effective_to bir-martalik close field, iqtisodiy maydonlar immutable) — ishlaydi, lekin "qisman mutable" semantika chalkash; C (superseded_by) — qo'shimcha relation; **A (tanlangan):** faqat `effective_from`; joriy narx = eng so'nggi effective_from ≤ now (plan+currency bo'yicha); kelajak narx = kelajak effective_from'li row; **row 100% immutable, UPDATE umuman yo'q**. unique(plan_id, currency, effective_from). Cycle snapshotlari tarixni allaqachon himoya qiladi — effective_to hech narsa uchun kerak emas. Sotuvni to'xtatish — plan status orqali.

## 12. Entitlement semantics (P1-2 → TD-86, AMENDMENT TO TD-52)

**Muammo:** `limit_value = null` — unlimited'mi, boolean-enabled'mi, disabled'mi, not-configured'mi? Resolver bug manbai.

**Proposal (TD-86):** ikki qatlamli aniq semantika: (1) **feature registry** (kod) har feature'ning `value_kind`ini e'lon qiladi: BOOLEAN / COUNT; (2) entitlement qatorida **mode enum: DISABLED / ENABLED / LIMITED / UNLIMITED** + `limit_value` (**faqat LIMITED'da majburiy** — CHECK: `(mode='LIMITED') = (limit_value IS NOT NULL)`). Row yo'qligi = granted emas (DISABLED default). PlanEntitlement va SubscriptionCycleEntitlement **bir xil kontrakt**. JSON value (D) va ko'p ustun (C) — rad.

## 13. Subscription identity semantics (P1-3 → TD-87, AMENDMENT TO TD-54)

**Muammo:** Subscription har purchase'gami yoki long-lived containermi — hujjatda implicit.

**Proposal (TD-87), explicit:** Subscription = **user'ning long-lived subscription relationship container'i**: User 1→0..1 non-terminal Subscription; har verified payment → yangi SubscriptionCycle (shu subscription ichida); plan change → shu subscription + SubscriptionChange; EXPIRED — **non-terminal** (manual renewal EXPIRED→ACTIVE, yangi cycle); CANCELLED — terminal. **Keyin qayta obuna: yangi Subscription record (default tavsiya — revive emas: tarix toza, partial unique buzilmaydi)**; revive vs new — arzon product tasdig'i, defaultni owner tasdiqlasin. Partial unique (user, status non-terminal) ikkala javob bilan ham ishlaydi.

## 14. Lesson relearning / completion history (P1-4 → TD-88, AMENDMENT TO TD-37)

**Muammo:** unique(user, lesson) yagona row + pinned revision. 2026: v3 COMPLETED; 2027: learner v8'ni qayta o'rganmoqchi — yagona row'da yoki v3 tarixini yo'qotamiz (L-5 buziladi), yoki relearning imkonsiz. Adaptive vision (review/spaced repetition) uchun relearning — kutilayotgan yo'nalish; keyin jadval bo'lish = og'riqli migration.

**Variantlar:** A (bitta lifetime row) — relearningni bloklaydi; C (Enrollment/Run entity) — og'irroq; D (attemptlardan derive) — "completion" query'lari qimmat/noaniq; **B (tanlangan):**

**Proposal (TD-88):** **LearnerLessonProgress = joriy holat** (unique(user, lesson) saqlanadi; relearn boshlaganda yangi PUBLISHED revision pin qilinib IN_PROGRESS'ga qaytadi) + append-only **LearnerLessonCompletion**: id, user_id, lesson_id, lesson_revision_id, completed_at, mastery_best_score nullable. Har tugatish (birinchi va relearn) — yangi completion row; **"revision reference abadiy" kafolati (eski L-5) endi completion tarixida yashaydi**. MVP'da relearning UX bo'lmasa ham schema tayyor — bu extension path emas, arzon hozirgi qaror.

## 15. Cross-domain relationship audit

| Bog' | Holati |
|---|---|
| Community mission → finance | ❌ uzuq edi → TD-84 yopadi |
| Activity/AssessmentItem/Attempt → MediaAsset | ❌ JSONB-only → TD-82 yopadi |
| AiEvaluation → UsageCounter | ✔ FINANCE'da bor; LEARNING'da eslatilmagan — hujjat izohi (P2) |
| SubscriptionCycle → RewardPolicyVersion / IzlRateVersion / PlanPrice | ✔ ikkala tomonда izchil |
| RewardGrant → attempt/session/plan item | ✔ (mission qismi TD-84 bilan aniqlashadi) |
| Notification → source | ✔ intentionally loose (TD-77 polymorphic — owner accepted) |
| XpGrant.source_refs / LearnerSignal.evidence_refs (JSONB) | ✔ intentionally loose — pul emas, recomputable ([JSONB_GOVERNANCE.md](JSONB_GOVERNANCE.md)) |
| StaffAudit → staff amallar | ❌ entity yo'q edi → TD-81 |
| SecurityEvent → User/AuthSession | ✔ nullable FK, to'g'ri |
| ReputationEvent → post/reply/reaction | ✔ typed FK |

## 16. Denormalized cache audit

| Cache | Source of truth | Update | Reconcile | Divergence impact | Criticality |
|---|---|---|---|---|---|
| IZLWallet.balance, reserved_amount | Ledger + RESERVED redemptions | Bir tranzaksiya (lock) | Davriy SUM job | Moliyaviy noto'g'rilik | **CRITICAL** |
| SubscriptionCycle.earned_izl | Cycle RewardGrant'lari | Bir tranzaksiya (lock) | SUM job | Ceiling buzilishi | **CRITICAL** |
| XpBalance.total | XpGrant'lar | Bir tranzaksiya | Recompute mumkin | Gamification xatosi | MEDIUM |
| ReputationBalance.total | ReputationEvent'lar | Bir tranzaksiya | Recompute | Ko'rinish xatosi | MEDIUM |
| Subscription.current_cycle_id | Cycles | Activation tranzaksiyasi | — | Noto'g'ri routing | MEDIUM |
| LearnerSkillState (butun row) | Evidence (dizayn bo'yicha derived) | Engine | Recompute — dizaynning o'zi | Noto'g'ri personalization | MEDIUM |
| LearnerLessonProgress.completion_pct / mastery_best_score | Attempts/progress payload | Engine tranzaksiyasi | Recompute | UX xatosi | LOW-MED |
| WeeklyProgress.completed_sessions | LearningSession'lar | Engine | Recompute | Target ko'rinishi | LOW-MED |
| CommunityPost.reply_count/reaction_count | Rows | Bir tranzaksiya | Davriy reconcile (TD-73) | Ko'rinish xatosi | LOW |
| LessonRevision.estimated_duration_min | Activity durationlar | Builder | Recompute | Reja aniqligi | LOW |
| AssessmentAttempt.result_summary | SkillMeasurement'lar | Write-once (completion) | — | Display xatosi | LOW |
| display_level | mastery_score + mapping | Engine | Recompute | Display xatosi | LOW |

Qoida: CRITICAL cache'lar faqat lock ostidagi bir tranzaksiyada; LOW'lar drift'ga chidamli + reconcile.

## 17. Timezone / local-date integrity

- **Muammo:** UserProfile.timezone nullable; DailyPlan.local_date, WeeklyProgress.week_start, daily IZL eligibility — timezone'ga bog'liq; authority aniqlanmagan.
- **Recommendation (TD-91):** (1) **local_date/week_start — tarixiy snapshot**: yozilish paytidagi effective timezone bilan hisoblanadi va **hech qachon qayta hisoblanmaydi** (tz o'zgarsa faqat kelajak yozuvlarga ta'sir); (2) effective timezone = profile.timezone, bo'lmasa **system default (MVP: Asia/Tashkent — bitta bozor)**; (3) DailyPlan'ga `timezone_snapshot` ustuni (yengil, audit uchun — qaysi tz bilan hisoblangan). Exact UX policy (tz o'zgarganda foydalanuvchiga nima ko'rsatiladi) — invent qilinmadi, OPEN.
- Daily IZL eligibility chegarasi = plan'ning local_date'i (snapshot) — tz o'yinlari bilan ikki "bugun" olish mumkin emas (bir local_date'da bitta CURRENT plan + dedup_key'lar).

## 18. Numeric / type policy (TD-89)

TD-46 ("integer money, float yo'q") buzilmaydi — faqat DB width/mapping:

| Ma'lumot | Tavsiya | Sabab |
|---|---|---|
| UZS summalar (narx, ceiling, to'lov) | **INTEGER (int4)** | Row-level max ~2.147 mlrd so'm — yetarli headroom; Prisma Int → JS number — BigInt/JSON serialization frictionsiz |
| IZL miqdorlar/balans | **INTEGER** | Qiymatlar kichik (ceiling ~ yuzlab-minglab IZL) |
| XP, reputation, counters, entry_no, attempt_no, position | **INTEGER** | Real diapazonlar kichik |
| mastery_score, confidence | **INTEGER basis points (0–10000)** | Deterministik, float drift yo'q, JSON-safe; "float taqiqi" pulga tegishli, lekin bp learning score uchun ham eng sodda aniq yechim; display % = bp/100 |
| response_time_ms, active_seconds | INTEGER | |
| BIGINT | Faqat chindan unbounded o'sish isbotlansa (hozircha nomzod yo'q) | Prisma BigInt ergonomika narxini faqat zarurat oqlaydi |

Agar kelajakda biror maydon int4 chegarasiga yaqinlashsa — PG'da int4→int8 migration arzon (table rewrite, lekin to'g'ri vaqtda rejalashtiriladi).

## 19. Enum / config registry policy (TD-90)

| Sinf | Mexanizm | Misollar |
|---|---|---|
| **DB enum** (yopiq, faqat kod bilan o'zgaradi) | Prisma/PG enum | UserStatus, ContentStatus, ActivityType, AssessmentPurpose, attempt/response statuslari, LedgerEntryType, payment canonical statuslari, RedemptionStatus, SubscriptionStatus, DailyPlan CURRENT/SUPERSEDED, DailyPlanSection, RoadmapItem type/status, SignalType, visibility (VISIBLE/...), RewardCategory (accepted 4), EntitlementMode (TD-86) |
| **Seeded jadval** (runtime boshqariladi, FK kerak) | Table | Role, ReactionType, SubscriptionPlan |
| **Code registry (string ustun)** (o'sadi, FK keraksiz) | App registry + validation | permission_code, feature_code, reason/category kodlar (report, mistake, reputation, XP), notification type, StaffAudit action_code, mission/reward rule kodlari |

Qoida: product-tunable narsa hech qachon DB enum bo'lmaydi (PG enumdan qiymat olib tashlash og'riqli); yopiq state machine'lar hech qachon erkin string bo'lmaydi.

## 20. Prisma relation risk review

High-risk juftliklar (schema yozuvchi uchun oldindan):

1. **Circular pointer'lar (nullable FK + named relations):** Lesson↔LessonRevision (`published_revision_id`), Subscription↔SubscriptionCycle (`current_cycle_id`), CommunityPost↔CommunityReply (`accepted_reply_id`), AssessmentDefinition↔AssessmentDefinitionVersion (`current_version_id`, TD-83). Prisma'da mumkin — relation nomlari majburiy, migration'da pointer nullable bo'lishi shart.
2. **XOR nullable relations (CHECK raw SQL):** AiEvaluation (response XOR attempt), CommunityReaction, CommunityReport, DailyMissionCompletion (TD-84), RewardGrant evidence (kategoriya-mos CHECK).
3. **Partial unique (Prisma-native EMAS → custom SQL migration):** bitta PUBLISHED revision/lesson; bitta ACTIVE roadmap/(user,subject); bitta CURRENT DailyPlan/(user,local_date); bitta non-terminal Subscription/user; bitta ACTIVE RewardPolicyVersion; reaction/report partial unique'lar; client_request_id WHERE NOT NULL.
4. **Self-relation:** LessonPrerequisite (ikki FK → Lesson) — named relations.
5. **User'ga ko'p relation:** created_by/updated_by/actor/granted_by/reporter/... — har biriga alohida relation nomi kerak (schema'da eng ko'p mexanik ish shu yerda).
6. **Composite-FK texnikasi (ixtiyoriy):** "published_revision shu lesson'ga tegishli" kabi parent-consistency — PG composite FK bilan enforce qilish mumkin (unique(id, lesson_id) + FK juftlik) — Prisma-native emas; default: app validation, xohlansa raw SQL.
7. **JSONB:** Prisma `Json` — typed emas; TD-22/validation qatlamiga tayanish (governance hujjati).
8. **Prisma bilim chegarasi:** partial unique/CHECK/DEFERRABLE constraintlar `prisma migrate` custom SQL bloklarida yashaydi — schema.prisma o'zi to'liq manzarani bermaydi; [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) shu gap'ni yopish uchun majburiy hamroh hujjat.

## 21. Proposed decisions / amendments

| TD | Mavzu | Turi |
|---|---|---|
| TD-81 | StaffAudit append-only accountability entity | Yangi |
| TD-82 | Media relational FK qatlami (ActivityMedia, AssessmentItemMedia, response_media_asset_id) | Yangi (TD-21/25 saqlanadi) |
| TD-83 | AssessmentDefinitionVersion (immutable config) + attempt version FK | AMENDMENT TO TD-32/TD-33 |
| TD-84 | DailyMissionCompletion + RewardGrant mission evidence mapping | AMENDMENT TO TD-48 |
| TD-85 | PlanPrice: faqat effective_from, to'liq immutable rows | AMENDMENT TO TD-51 |
| TD-86 | Entitlement mode semantikasi (DISABLED/ENABLED/LIMITED/UNLIMITED + registry value_kind) | AMENDMENT TO TD-52 |
| TD-87 | Subscription = long-lived container; re-subscribe = yangi record (default) | AMENDMENT TO TD-54 |
| TD-88 | LearnerLessonCompletion append-only history + progress = current state | AMENDMENT TO TD-37 |
| TD-89 | Numeric width policy (INTEGER default, basis points) | Yangi (TD-46 doirasida) |
| TD-90 | Enum vs seeded-table vs code-registry policy matritsasi | Yangi |
| TD-91 | local_date/week_start — tz snapshot semantikasi + DailyPlan.timezone_snapshot | Yangi |
| TD-92 | JSONB governance qoidalari | Yangi |

**Hammasi ACCEPTED (owner review 2026-08-27)** — [TECH_DECISIONS.md](TECH_DECISIONS.md); yuqoridagi 4 clarification bilan.

## 22. Blocking owner review list — ✅ YAKUNLANDI

TD-81..TD-92 to'liq tasdiqlandi. **Schema-blocking owner qarorlari qolmadi.** Schema blocker bo'lmagan infra tanlovlari (SMS provider, object storage, queue, AI provider, deployment, CSRF exact mexanizmi, recurring payment protokoli) — implementation OPEN sifatida [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)da qoladi.

## 23. Schema-readiness verdict

**READY (owner review'dan keyin).** Final validation — barcha savollarga HA:

- **Historical reproducibility:** qaysi LessonRevision (progress/completion), qaysi AssessmentDefinitionVersion + eligible item pool (VersionItem) + presented itemlar (Response) + engine_version, qaysi reward/mission evidence (RewardGrant → Completion → Evidence FK'lari), qaysi cycle iqtisodi (snapshotlar) + entitlement (CycleEntitlement) + ledger tartibi (entry_no) — hammasi aniqlanadi. ✅
- **Referential integrity:** referenced MediaAsset delete (FK RESTRICT), orphan published revision (partial unique + pointer), mission evidence'siz grant (FK + CHECK), duplicate reward (dedup unique), duplicate callback (event unique), multiple CURRENT plan / ACTIVE roadmap / non-terminal subscription (partial unique'lar) — enforcement strategiyasi [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md)da hujjatlashtirilgan. ✅
- **Current vs history ajratilgan:** Progress↔Completion, SkillState↔Measurement, Wallet↔Ledger, ReputationBalance↔Event, current price↔cycle snapshot. ✅
- **Audit chegaralari distinct:** StaffAudit, SecurityEvent, ModerationAction, IZLLedgerEntry, PaymentTransaction. ✅

**Phase 1.3 — Prisma Schema v1 boshlanishi mumkin** (alohida faza promptida).

---

### Bajarilgan documentation cleanup (accepted qarorlarga moslash)

1. PRODUCT.md — "texnik arxitektura hali tanlanmagan" stale header → accepted stack reference.
2. CONTENT_MODEL.md — "Exact technical storage OPEN" (2 joy) → TD-21/22 + DATA_MODEL_CORE reference (product semantika saqlangan).
3. AI_SYSTEM.md — "structured state texnik bosqichda" → DATA_MODEL_LEARNING reference (provider OPEN qoladi).
4. PRODUCT_DECISIONS.md D-31 — session/token OPEN wording → Phase 1.1 accepted reference (SMS provider OPEN qoladi).
5. USER_ROLES.md — "exact audit architecture keyin" → TD-81 PROPOSED reference (ACCEPTED deb yozilmagan).
6. AUTH_ARCHITECTURE.md — parallel refresh concurrency implementation note qo'shildi (qarorlar o'zgartirilmagan).
