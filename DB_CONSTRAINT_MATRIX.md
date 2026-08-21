# Izlan — Database Constraint Matrix

> Phase 1.2D deliverable — **owner tomonidan architecture artifact sifatida ACCEPTED** (2026-08-27). Prisma Schema v1 yozuvchisi uchun **majburiy hamroh hujjat** — schema.prisma o'zi bu ro'yxatning hammasini ifodalay olmaydi.
> Mexanizm belgilari: PK, FK, UNIQUE, PARTIAL UNIQUE, CHECK, TXN/LOCK (transaction+row lock), APP (application validation), SM (state machine), PRIV (DB privilege: UPDATE/DELETE revoke — ixtiyoriy qattiqlashtirish).
> **Partial index/unique bo'yicha aniqlik (owner correction):** "hamma partial unique custom SQL talab qiladi" degan categorical gap noto'g'ri — hozirgi Prisma partial index'larni schema tilida `where` bilan qo'llay olishi mumkin, lekin bu capability preview bo'lishi mumkin. **Izlan production policy:** core integrity uchun preview feature'ga avtomatik tayanilmaydi; Phase 1.3'da Prisma versiyasi pin qilinadi; agar partial index o'sha pinned versionda preview bo'lsa — **default: kritik partial unique'lar custom PostgreSQL migration'da**; GA/stable bo'lsa — Prisma-native ishlatish mumkin. CHECK va boshqa PG-only invariantlar alohida qoladi.
> "Prisma" ustuni: ✅ native; ⚠️ pinned versiyada tekshiriladi — preview bo'lsa custom SQL (default), GA bo'lsa native mumkin; CHECK'lar uchun — custom SQL; ❌ faqat application.
> **Phase 1.3 majburiyati (owner):** har invariant qayta map qilinadi (A Prisma-native / B PG migration constraint / C txn+lock / D app state machine / E app validation) — **hech bir invariant silently dropped bo'lmaydi**; ayniqsa: XOR'lar, one CURRENT/ACTIVE/non-terminal qoidalari, reward dedup, ledger order, reserved≤balance, earned≤ceiling, accepted-reply parent mosligi, media FK, VersionItem membership, prerequisite DAG.
> † belgili qatorlar — Phase 1.2D'da qo'shilgan, endi ACCEPTED.

## 1. Auth

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| A1 | User.phone unique (canonical E.164) | UNIQUE | ✅ | Normalization backendda | P0 | I-13 |
| A2 | UserProfile 1:1 User | PK=FK | ✅ | — | P0 | |
| A3 | RefreshToken.token_hash unique | UNIQUE | ✅ | — | P0 | |
| A4 | Ishlatilgan/revoked refresh qayta kelsa → session revoke | SM + TXN | ❌ | Reuse detection | P0 | Parallel-refresh false-positive note: AUTH §8 |
| A5 | Bir phone(+purpose)da bitta faol OTP challenge | TXN (invalidate→insert) | ❌ | App txn; ixtiyoriy PARTIAL UNIQUE | P1 | Resend eski challenge'ni invalidatsiya qiladi |
| A6 | OTP attempt limit (5) → challenge invalidated | SM | ❌ | Counter + lock | P0 | Brute-force himoya |
| A7 | Revoked/expired session refresh ololmaydi | SM + FK | ❌ | Refresh path DB check | P0 | |
| A8 | SecurityEvent append-only | PRIV | ❌ | Update path yo'q | P1 | |

## 2. Authorization

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| R1 | Role.code unique | UNIQUE | ✅ | Seeded | P1 | |
| R2 | UserRole unique(user, role) | UNIQUE | ✅ | — | P0 | Idempotent grant |
| R3 | RolePermission unique(role, permission_code) | UNIQUE | ✅ | Kod registry validatsiyasi | P1 | |
| R4 | SubjectAssignment unique(user, subject) | UNIQUE | ✅ | — | P0 | Methodist scope |
| R5 | permission_code registry'da mavjud | — | ❌ | APP | P1 | TD-26 |
| R6 | SUSPENDED user permission bilan amal bajarmaydi | — | ❌ | Guard: state check permissiondan oldin | P0 | I-14 |

## 3. Content

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| C1 | Subject.slug global unique | UNIQUE | ✅ | — | P1 | |
| C2 | Track (subject, slug) unique | UNIQUE | ✅ | — | P1 | |
| C3 | Level (track, code) va (track, sort_order) unique | UNIQUE ×2 | ✅ | — | P1 | |
| C4 | (parent, sort_order) unique — Module/Topic/Lesson | UNIQUE | ✅ | Reorder = txn renumber | P2 | Renumber to'qnashuvi: DEFERRABLE (raw SQL) yoki two-phase app renumber |
| C5 | LessonRevision unique(lesson, version) | UNIQUE | ✅ | — | P0 | TD-24 |
| C6 | published_revision_id shu lesson'ning revisioni | (ixtiyoriy composite FK) | ⚠️ | APP (default) | P0 | PG texnika: unique(id, lesson_id) + composite FK; default — app |
| C7 | Bitta lessonda bitta PUBLISHED revision | PARTIAL UNIQUE (lesson_id WHERE status='PUBLISHED') | ⚠️ | Publish txn | P0 | I-2 |
| C8 | PUBLISHED revision + activitylari immutable | PRIV + SM | ❌ | Edit → yangi draft revision | P0 | I-3 |
| C9 | LessonPrerequisite unique(pair) + self-ref taqiqi | UNIQUE + CHECK (lesson_id <> prerequisite_lesson_id) | ⚠️ | — | P1 | |
| C10 | Prerequisite DAG (cycle yo'q) | — | ❌ | Saqlash txn'ida cycle check (TD-29) | P1 | DB enforce qilmaydi |
| C11 | Activity unique(revision, position) | UNIQUE | ✅ | Renumber txn | P2 | C4 kabi |
| C12 | LessonSkill unique(pair); skill lesson Subject'iga tegishli | UNIQUE + APP | ✅/❌ | Cross-table subject check | P1 | I-7 |
| C13 | ActivitySkill unique(pair); subject mosligi | UNIQUE + APP | ✅/❌ | L-9 | P1 | |
| C14 | Activity type↔payload strict validity (publishda) | — | ❌ | TD-22 discriminated union | P0 | I-4 |
| C15 | AI-sourced content human review'siz PUBLISHED emas | SM | ❌ | Publish transition guard | P0 | I-5 |
| C16 | MediaAsset.storage_key unique | UNIQUE | ✅ | — | P1 | |
| C17† | Published content ishlatgan MediaAsset o'chirilmaydi | FK RESTRICT (TD-82 junctionlar orqali) | ✅ (junction bo'lsa) | Hozircha APP-only — TD-82 shuni tuzatadi | P0 | I-15 |
| C17b† | Payload'dagi har media_asset_id uchun junction qator bor | — | ❌ | TD-22 validation (publish/submit) | P0 | Drift himoyasi |
| C18 | Skill (subject, name) unique; (subject, code) unique | UNIQUE ×2 | ✅ | code nullable — PG'da NULL'lar to'qnashmaydi | P1 | |

## 4. Learning

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| L1 | AiEvaluation target XOR (response XOR attempt) | CHECK (num_nonnulls(...)=1) | ⚠️ | — | P0 | L-8 |
| L2 | AssessmentResponse unique(attempt, sequence_no) | UNIQUE | ✅ | — | P0 | |
| L3 | SUBMITTED response/attempt javobi immutable | SM + PRIV | ❌ | Submit'dan keyin update yo'q | P0 | L-1 |
| L4 | ActivityAttempt unique(user, activity, attempt_no) | UNIQUE | ✅ | attempt_no monotonic — app | P0 | L-12 |
| L5 | client_request_id dedup | PARTIAL UNIQUE (user, client_request_id WHERE NOT NULL) | ⚠️ | — | P1 | TD-44 |
| L6 | LearnerSkillState unique(user, skill) | UNIQUE | ✅ | Faqat engine yozadi | P0 | |
| L7 | Skill state/measurement subject scope mosligi | — | ❌ | Engine check | P1 | L-3 |
| L8 | SkillMeasurement append-only | PRIV | ❌ | Recompute o'chirmaydi | P0 | L-10 |
| L9 | Bitta ACTIVE roadmap per (user, subject) | PARTIAL UNIQUE (user, subject WHERE status='ACTIVE') | ⚠️ | — | P0 | L-6 |
| L10 | RoadmapItem type↔required refs mosligi | CHECK (per-type) | ⚠️ | APP ham | P1 | LESSON→lesson_id, CHECKPOINT→checkpoint_id... |
| L11 | Recommendation-driven change faqat ACCEPTED recommendation bilan | FK + APP | ❌ | Change yozish guard'i | P0 | L-4 |
| L12 | RoadmapChange append-only | PRIV | ❌ | — | P1 | |
| L13 | LearnerLessonProgress unique(user, lesson) | UNIQUE | ✅ | — | P0 | TD-88† bilan: current state |
| L14† | Har completion LearnerLessonCompletion append-only qatorida (revision bilan) | FK + PRIV | ✅/❌ | Completion txn | P0 | TD-88; eski L-5 kafolati shu yerda |
| L15 | DailyPlan unique(user, local_date, generation_no) | UNIQUE | ✅ | — | P0 | TD-42 |
| L16 | Bitta CURRENT plan per (user, local_date) | PARTIAL UNIQUE (WHERE status='CURRENT') | ⚠️ | Supersede txn | P0 | |
| L17 | SUPERSEDED snapshotlar rewrite qilinmaydi | PRIV + SM | ❌ | — | P0 | L-13 |
| L18 | Checkpoint unique(module) | UNIQUE | ✅ | — | P1 | |
| L19 | Checkpoint.assessment_definition_id NOT NULL | FK NOT NULL | ✅ | — | P0 | L-7 |
| L20 | WeeklyProgress unique(user, week_start) | UNIQUE | ✅ | Target hafta boshida snapshot | P1 | |
| L21 | Bitta ACTIVE signal per (user, type, dimensiya) | — | ❌ | Engine dedup (nullable dimensiyalar partial unique'ni noqulay qiladi) | P2 | |
| L22† | Attempt definition_version + engine_version saqlaydi | FK NOT NULL | ✅ | — | P0 | TD-83 |
| L23 | AssessmentItem PUBLISHED bo'lgach immutable | SM + PRIV | ❌ | Edit = yangi item | P0 | TD-32 |
| L24† | AssessmentVersionItem unique(version, item); published version membership immutable | UNIQUE + SM/PRIV | ✅/❌ | Draft'da tahrir, publishdan keyin yo'q | P0 | TD-83 clarification |
| L25† | DailyMissionCompletion unique(daily_plan_item_id) | UNIQUE | ✅ | One-shot mission | P0 | TD-84 |
| L26† | MissionCompletionEvidence: har qatorda roppa-rosa bitta typed FK; completion'da ≥1 evidence | CHECK (XOR) + APP | ⚠️/❌ | Engine completion rule | P0 | TD-84 |
| L27† | LearnerLessonCompletion unique(user, lesson, completion_no); append-only | UNIQUE + PRIV | ✅/❌ | — | P0 | TD-88 |
| L28† | ActivityMedia unique(activity, asset, role?); AssessmentItemMedia unique(item, asset) | UNIQUE | ✅ | Payload↔junction sinxron — TD-22 validation | P0 | TD-82; exact uniqueness schema'da |
| L29† | DailyPlan.timezone_snapshot yozilgach o'zgarmaydi; local_date qayta hisoblanmaydi | SM/PRIV | ❌ | Snapshot qoidasi | P1 | TD-91 |

## 5. Finance

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| F1 | IZLWallet 1:1 user | PK=FK | ✅ | — | P0 | |
| F2 | Ledger unique(user, entry_no), monotonic | UNIQUE + TXN/LOCK | ✅/⚠️ | Wallet lock → next entry_no | P0 | F-18; monotoniklik — lock bilan |
| F3 | Ledger append-only (UPDATE/DELETE yo'q) | PRIV | ❌ | Repositoryda update path yo'q | P0 | F-1; PRIV tavsiya etiladi |
| F4 | wallet.balance == SUM(ledger) | TXN + reconcile job | ❌ | Bir tranzaksiya qoidasi | P0 | F-2 |
| F5 | reserved_amount ≥ 0; balance ≥ 0 | CHECK ×2 | ⚠️ | — | P0 | Negative balance policy OPEN — CHECK hozircha ≥0 |
| F6 | reserved_amount ≤ balance (spendable ≥ 0) | CHECK + TXN/LOCK | ⚠️ | Reservation lock ostida | P0 | F-12/F-17 |
| F7 | RewardGrant unique(user, dedup_key) | UNIQUE | ✅ | — | P0 | F-5 — anti-farming DB himoyasi |
| F8 | Grant kategoriya↔evidence FK mosligi | CHECK (per-category) | ⚠️ | APP ham | P0 | F-4; TD-84† bilan MISSION → completion FK |
| F9 | earned_izl ≤ reward_ceiling_izl | CHECK + TXN/LOCK (cycle lock) | ⚠️ | Grant txn | P0 | F-6 — ikkala qatlam ham |
| F10 | Grant ↔ EARN ledger entry 1:1 | UNIQUE FK + TXN | ✅ | Bir tranzaksiya | P0 | |
| F11 | Bitta ACTIVE RewardPolicyVersion | PARTIAL UNIQUE (WHERE status='ACTIVE') | ⚠️ | — | P1 | |
| F12† | PlanPrice rows to'liq immutable (effective_to YO'Q); unique(plan, currency, effective_from) | UNIQUE + PRIV | ✅/❌ | Yangi narx = yangi row (TD-85) | P1 | F-8 |
| F27† | RewardGrant(DAILY_MISSION) → daily_mission_completion_id FK | FK + CHECK | ✅/⚠️ | Kategoriya-mos evidence | P0 | TD-84 |
| F13† | Entitlement unique(plan/cycle, feature) + mode↔limit CHECK | UNIQUE + CHECK ((mode='LIMITED')=(limit_value IS NOT NULL)) | ✅/⚠️ | Registry value_kind | P1 | TD-86 |
| F14 | Bitta non-terminal Subscription per user | PARTIAL UNIQUE (user WHERE status IN ('ACTIVE','EXPIRED')) | ⚠️ | — | P0 | F-13; TD-87† semantika |
| F15 | Cycle unique(subscription, sequence_no) | UNIQUE | ✅ | — | P0 | |
| F16 | Cycle iqtisod snapshotlari immutable | PRIV + SM | ❌ | earned/status'dan boshqasi yozilmaydi | P0 | F-7 |
| F17 | UsageCounter unique(cycle, feature); used ≥ 0 | UNIQUE + CHECK | ✅/⚠️ | Consume domain entity bilan bir txn | P1 | F-14 |
| F18 | PaymentTransaction unique(provider, provider_tx_id) | UNIQUE | ✅ | — | P0 | |
| F19 | Callback unique(provider, provider_event_id) | UNIQUE | ✅ | Atomic processing | P0 | F-10 idempotency |
| F20 | Bitta verified payment → bitta cycle | UNIQUE(cycle.payment_order_id) + TXN | ✅ | Activation txn | P0 | F-10 |
| F21 | Frontend activation yo'q — faqat server-verified yo'l | — | ❌ | Trust boundary (API dizayni) | P0 | F-9 |
| F22 | REDEEM entry faqat RESERVED→APPLIED'da; ledger_entry_id ⟺ APPLIED | SM + CHECK ((status='APPLIED')=(ledger_entry_id IS NOT NULL)) | ⚠️ | Redemption txn | P0 | TD-57 |
| F23 | Payment/order state machine transitionlari | SM | ❌ | Canonical mapping adapterda | P1 | |
| F24 | payable = gross − discount | CHECK | ⚠️ | Order yaratish | P1 | Snapshot arifmetikasi |
| F25 | XP va IZL strukturaviy alohida | Jadval dizayni | ✅ | — | P0 | F-3 |
| F26 | ADJUSTMENT har doim reason + actor bilan | CHECK (entry_type='ADJUSTMENT' → reason/actor NOT NULL) | ⚠️ | + StaffAudit† | P0 | F-15 |

## 6. Community / Announcements / Notifications

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| K1 | Reaction target XOR (post XOR reply) | CHECK | ⚠️ | — | P0 | |
| K2 | Reaction unique(user, post, type) / (user, reply, type) | PARTIAL UNIQUE ×2 | ⚠️ | Double-click no-op | P0 | C-5 |
| K3 | Report target XOR | CHECK | ⚠️ | — | P0 | |
| K4 | Report unique(reporter, post) / (reporter, reply) | PARTIAL UNIQUE ×2 | ⚠️ | — | P1 | C-5 |
| K5 | accepted_reply shu postga tegishli + faqat QUESTION | (ixtiyoriy composite FK) + APP | ❌ | Set-accepted guard | P0 | C-4; C6 texnikasi qo'llanishi mumkin |
| K6 | Hidden reply accepted bo'lmaydi | — | ❌ | Guard | P1 | |
| K7 | CommunityPostMedia unique(post, asset); faqat image/audio | UNIQUE + APP (mime allowlist) | ✅/❌ | Attach guard | P0 | C-1 — video taqiqi |
| K8 | Post topic → subject mosligi | — | ❌ | APP | P1 | C-3 |
| K9 | ModerationAction append-only | PRIV | ❌ | — | P0 | C-6 |
| K10 | Restriction expires_at > starts_at | CHECK | ⚠️ | — | P1 | |
| K11 | ReputationEvent unique(user, dedup_key) | UNIQUE | ✅ | — | P1 | C-16 |
| K12 | AnnouncementUserState unique(announcement, user) | UNIQUE | ✅ | Lazy insert no-op | P1 | |
| K13 | Notification unique(user, dedup_key) | UNIQUE | ✅ | — | P0 | C-11 |
| K14 | Hidden content feed'da chiqmaydi | — | ❌ | Query invariant (visibility filter) | P1 | C-7 |
| K15 | Community wallet/XP'ni bevosita mutate qilmaydi | — | ❌ | Module boundary (arxitektura) | P0 | C-8/C-14 |
| K16 | Blocked media serve qilinmaydi | — | ❌ | Serving guard | P0 | C-9 |
| K17 | Announcement faqat publish oynasida ko'rinadi | — | ❌ | Query invariant | P1 | C-10 |

## 7. StaffAudit (TD-81†)

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| S1 | StaffAudit append-only | PRIV | ❌ | Update path yo'q | P0 | |
| S2 | Har sensitive staff amal audit yozuvi bilan bir tranzaksiyada | TXN | ❌ | Service-layer qoida | P0 | D-35 |
| S3 | target_type/target_id — loose reference (FK emas, ataylab) | — | — | Registry validatsiya | P2 | Target arxivlansa ham audit yashaydi |

## 8. FK delete policy matrix

Umumiy qoida (TD-30/TD-60): **default — RESTRICT + archive-first**; CASCADE faqat "bola ota'siz ma'nosiz va tarixga kirmagan" holatlarda.

| FK | Policy | Izoh |
|---|---|---|
| User → har qanday financial/learning/community tarix | **RESTRICT**; erasure = anonymization (OPEN) | Hard delete yo'q |
| User → UserProfile | CASCADE nazariy, amalda user o'chirilmaydi | — |
| AuthSession → RefreshToken | CASCADE OK (session o'chirilsa) — amalda TTL cleanup | Retention: xavfsiz muddatdan keyin |
| OtpChallenge | Hard delete OK (ephemeral TTL cleanup) | TD-30 |
| Subject/Track/Level/Module/Topic (children mavjud) | **RESTRICT** — operatsion yo'l ARCHIVE | 1.2A §25 |
| Lesson → LessonRevision | **RESTRICT**; faqat hech qachon publish bo'lmagan DRAFT revision hard delete | |
| LessonRevision (draft) → Activity | CASCADE (draft delete scenariysi) | Published — immutable, delete yo'q |
| Lesson/Revision/Activity ← learner history (attempts, completions, progress) | **RESTRICT** — content ARCHIVE bo'ladi, o'chmaydi | L-2 |
| MediaAsset ← ActivityMedia/AssessmentItemMedia/CommunityPostMedia/response FK† | **RESTRICT** | TD-82 bilan real FK |
| AssessmentDefinition/Version/Item ← attempts/responses | **RESTRICT** | TD-83 |
| CommunityPost → CommunityReply/Reaction/Report | **Semantic remove (visibility)** — CASCADE EMAS | TD-67 |
| CommunityPost (hech reply/report/reactionsiz, author's own draft-like) | App-managed delete ruxsati — policy OPEN (editing bilan) | |
| ReactionType ← CommunityReaction | **RESTRICT** (retirement = is_active=false) | TD-65 clarification |
| Subscription/Cycle/Payment/Ledger/Grant/Redemption o'zaro | **RESTRICT** — hech biri o'chmaydi | TD-60 |
| DailyPlan/Item ← RewardGrant/MissionCompletion† | **RESTRICT** — snapshot audit zanjiri | TD-42 |
| Announcement ← AnnouncementUserState | CASCADE qabul qilinishi mumkin (state — ephemeral), lekin default RESTRICT + ARCHIVE | P2 |
| Notification | Retention cleanup delete OK (policy bilan) — financial emas | TD-77 |
| SecurityEvent / StaffAudit† / ModerationAction | Delete yo'q; retention policy OPEN | Append-only; StaffAudit target — loose polymorphic (FK yo'q, ataylab); actor erasure/legal — alohida OPEN policy |
| ActivityMedia / AssessmentItemMedia† | Unpublished parent draft delete'ida CASCADE mumkin; published referencelar — RESTRICT/archive-first | TD-82 |
| AssessmentDefinitionVersion / AssessmentVersionItem† | Published → immutable, RESTRICT | TD-83 |
| DailyMissionCompletion / Evidence† | Append-only; delete yo'q | TD-84 |
| RewardGrant ← DailyMissionCompletion† | RESTRICT — completion referenced bo'lsa o'chmaydi | TD-84 |
| LearnerLessonCompletion† | Append-only; lesson/revision tomoni RESTRICT/archive-first | TD-88 |

## 9. Xulosa

- **Prisma-native (✅):** ~40 invariant — oddiy UNIQUE/FK/NOT NULL.
- **Custom SQL yoki pinned-version tekshiruvi (⚠️):** ~22 — PARTIAL UNIQUE'lar (owner policy: pinned Prisma versiyasida preview bo'lsa — custom PostgreSQL migration, GA bo'lsa native mumkin) va barcha CHECK'lar (XOR, arifmetika, mode) — CHECK'lar har doim custom SQL. **Phase 1.3'da schema.prisma bilan birga custom constraint migration birinchi kundan yoziladi** — keyin qo'shish emas.
- **Faqat application (❌):** ~25 — state machine'lar, cross-table mosliklar, DAG, trust boundary, module boundary. Bular uchun service-layer guard'lar 1.3+ implementatsiyasida constraint matritsaga reference bilan yoziladi.
- **PRIV qattiqlashtirish:** append-only jadvallarda (ledger, measurements, audit, changes) app-role'dan UPDATE/DELETE'ni revoke qilish — arzon himoya, tavsiya etiladi (implementation detail).

## 9b. Learner Learning Intent (TD-93, Phase 1.5A-2)

| ID | Invariant | DB mechanism | Prisma | App check | Severity | Notes |
|---|---|---|---|---|---|---|
| LI-01 | unique(user, subject) — bir user+subject → bitta current intent | UNIQUE | ✅ | Upsert | P0 | migration'da |
| LI-02 | trackId nullable (subject-only resumable) | nullable column | ✅ | — | P1 | §5 |
| LI-03 | track mavjud bo'lsa → track.subjectId == intent.subjectId | — | ❌ | Service validation (`saveIntent`) | P0 | cross-table — DB CHECK emas, app (§9) |
| LI-04 | Yangi/update selection → Subject PUBLISHED | — | ❌ | Service (`findPublishedSubject`) | P0 | DRAFT/ARCHIVED yashirin |
| LI-05 | Yangi/update track → Track PUBLISHED | — | ❌ | Service (`findPublishedTrack`) | P0 | |
| LI-06 | ARCHIVED content mavjud intentni destructive delete QILMAYDI | FK RESTRICT | ✅ | Archive-first | P1 | Subject/Track onDelete Restrict; readiness `countCompleteVisible` visibility'ni FIRST-completion'da tekshiradi |

**Silently dropped: 0.** LI-03/04/05 — application/service validation (cross-table/lifecycle; DB CHECK cross-table qila olmaydi). Onboarding readiness (`learningIntent` missing) = kamida bitta trackId-to'ldirilgan + hozir PUBLISHED intent (§23).

## 9c. Placement (diagnostic) assessment (Phase 1.5B)

Runtime invariants — schema O'ZGARMADI; hammasi application/service qatlamida (assessment reproducibility Phase 1.3'da qurilgan).

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PA-01 | Attempt exact `AssessmentDefinitionVersion`ga pin (TD-83) | scalar FK (version relation) | createAttempt | P0 | current pointer o'zgarsa ham o'zgarmaydi |
| PA-02 | Engine faqat pinned `AssessmentVersionItem` pooldan tanlaydi | — (query scope) | AssessmentItemRepository | P0 | global item lookup YO'Q; §51 test |
| PA-03 | Resume/submit pinned versiyani id bo'yicha o'qiydi (status filtri YO'Q) | — | findVersionConfig | P0 | ARCHIVED versiya ham resumable (§38) |
| PA-04 | Objective scoring backend authority; answer key HTTP'da YO'Q | — | projectItemForLearner / scorer | P0 | §52 leak test |
| PA-05 | Attempt own-user (IDOR) | userId FK filter | findOwnAttempt | P0 | boshqa user → 404 |
| PA-06 | Bir presented itemga ko'pi bilan bitta submit (concurrency) | `unique(attempt, sequence_no)` + guarded updateMany | submitPresented count===1 | P0 | §48; loser idempotent replay/conflict |
| PA-07 | **Bir subject uchun bitta PUBLISHED DIAGNOSTIC definition** (TD-94) | **partial UNIQUE** `uq_def_published_diagnostic_per_subject` WHERE purpose_scope=DIAGNOSTIC AND status=PUBLISHED | resolveCurrentDefinition (>1 → CONFIGURATION_INVALID, defense-in-depth) | P0 | migration `20260819160000_harden_initial_diagnostic_constraints` |
| PA-08 | Config (`config`) publish'dan keyin immutable + validated | immutable (TD-83) | parsePlacementConfig | P1 | malformed → fail-safe, infinite attempt YO'Q |
| PA-09 | Completed attempt terminal (yangi response YO'Q) | — | status guard | P1 | §13 |
| PA-10 | **Bir (user, subject) uchun bitta IN_PROGRESS INITIAL_DIAGNOSTIC** (TD-95) | **partial UNIQUE** `uq_attempt_inprogress_initial_diagnostic_user_subject` WHERE purpose=INITIAL_DIAGNOSTIC AND status=IN_PROGRESS | start resume-by-(user,subject) + P2002 catch/resume | P0 | concurrent start → 1 attempt; completed ta'sirlanmaydi |
| PA-11 | Objective-only pool (start'dan oldin butun pool validated) | — | resolveCurrentDefinition | P0 | open_ended/unsupported/bad difficulty → CONFIGURATION_INVALID, attempt yaratilmaydi (§22/23) |
| PA-12 | Immutable response — replay/conflict | `unique(attempt, sequence_no)` | canonical answer compare | P1 | same → 200 idempotent; different → 409 RESPONSE_CONFLICT (§5) |
| PA-13 | Coverage feasibility | — | distinctSkills×itemsPerSkill ≤ maxItems | P1 | impossible plan → CONFIGURATION_INVALID (§19/34) |

**Silently dropped: 0.** PA-07/PA-10 endi **DB partial-unique** bilan himoyalangan (runtime fail-safe defense-in-depth qoladi). Side-effect boundary (§40): assessment LearnerSkillState/SkillMeasurement/Roadmap/XP/IZL YOZMAYDI — testda absent tasdiqlanadi. Migration: `20260819160000_harden_initial_diagnostic_constraints` (2 partial unique; old migrations tegilmadi).

## 9d. Skill profile derivation (Phase 1.5C)

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| SP-01 | LearnerSkillState unique(user, skill) | `@@unique(user_id, skill_id)` | — | P0 | current state — bitta qator per user+skill |
| SP-02 | Measurement/state Skill = attempt Subject (L-3) | — | ensureDiagnosticDerived (cross-subject → CONFIGURATION_INVALID, no writes) | P0 | §54 test |
| SP-03 | SkillMeasurement append-only | — | faqat create (createMany skipDuplicates); UPDATE/DELETE YO'Q | P0 | §51 test |
| SP-04 | Assessment-backed measurement idempotency | **partial UNIQUE** `uq_skill_measurement_assessment_idempotency` (attempt, skill, source, derivation_version) WHERE attempt IS NOT NULL | createMany skipDuplicates | P0 | migration `20260819170000`; §52 test; derivation_version NOT NULL bo'lgach semantikasi explicit |
| SP-05 | masteryScoreBp 0..10000 | CHECK `chk_lss_mastery_bp` + `chk_sm_score_bp` | round+clamp | P0 | mavjud (Phase 1.3) |
| SP-06 | confidenceBp 0..10000 (nullable) | CHECK `chk_lss_confidence_bp` + `chk_sm_confidence_bp` | round+clamp | P0 | mavjud (Phase 1.3) |
| SP-07 | evidenceCount ≥ 0 | — (CHECK yo'q) | app (real evidence soni ≥ 0) | P2 | construction bo'yicha; CHECK qo'shilmadi |
| SP-08 | Eski measurement yangi current state'ni overwrite qilmaydi | — | guarded updateMany (lastMeasurementAt ≤ completedAt OR null) + createMany skipDuplicates | P0 | chronology, concurrency-safe; §53 test |
| SP-09 | SkillMeasurement derivation identity mavjud + bo'sh emas (TD-98) | **NOT NULL** + CHECK `chk_sm_derivation_version_nonempty` (`length(trim)>0`) | caller explicit beradi (default YO'Q) | P0 | migration `20260819180000`; NULL/''/whitespace → violation (§15 test); dev+test empty → backfill yo'q |
| SP-10 | Lesson-backed measurement idempotency (TD-111) | **partial UNIQUE** `uq_skill_measurement_lesson_idempotency` (user_id, lesson_id, skill_id, source, derivation_version) WHERE lesson_id IS NOT NULL | createMany skipDuplicates | P0 | migration `20260820120000`; Phase 1.7C; complete replay/concurrent → measurement duplicate YO'Q (§59/60 test) |
| LP-01 | SkillMeasurement evidenceCount > 0 (TD-113) | **CHECK** `chk_sm_evidence_count_positive` (`evidence_count > 0`) + NOT NULL column | producer explicit beradi; pure engine defensive validate | P0 | migration `20260820130000`; Phase 1.8A; zero/negative-evidence milestone YO'Q; pure engine test §38 |
| LP-02 | SkillMeasurement observedAt NOT NULL (TD-113) | **NOT NULL** column `observed_at` | producer = attempt/completion.completedAt | P0 | migration `20260820130000`; logical evidence time (materialization time EMAS); dev+test bo'sh → backfill YO'Q |
| LP-03 | Merge source Skill user-requested Subject'ga tegishli | — | app (subjectSkillIdsForRecompute WHERE skill.subjectId = subject) | P1 | §59 subject-scope test; cross-subject state corruption YO'Q |
| LP-04 | LearnerSkillState unique(user, skill) | **UNIQUE** `learner_skill_state_user_id_skill_id_key` (mavjud) | upsert | P0 | Phase 1.3 |
| LP-05 | Faqat merge module current state yozadi (TD-115) | — | application invariant (grep gate) | P0 | §53 single-writer grep — yagona yozuvchi learning-progress.repository.upsertState |
| LP-06 | Measurement history append-only (§61) | — | createMany only; runtime UPDATE/DELETE YO'Q | P0 | grep gate; merge read-only against skill_measurement |
| LP-07 | Current state = normalized current merge window'dan hisoblanadi (§30) | — | recompute FROM SCRATCH (incremental mutation EMAS) | P0 | §53/§30 projection-rebuild test |
| LP-08 | Eng oxirgi calibration anchor reset semantics | — | pure engine anchor+window (learning-progress-merge-v1) | P0 | §45/46 pure + e2e test |
| LP-09 | Per user+skill merge serialization (§31) | — | `pg_advisory_xact_lock(hashtext(user), hashtext(skill))` transaction-scoped (bound params) | P0 | §50 concurrent-same-skill converge test |
| SG-01 | One ACTIVE LearnerSignal per (user, skill, type) (TD-117) | **partial UNIQUE** `uq_learner_signal_active` (user_id, skill_id, type) WHERE status = 'ACTIVE' | create + P2002 catch; per (user,skill) signal advisory-lock serialization | P0 | migration `20260820140000`; Phase 1.8B; concurrent trigger → 1 ACTIVE (§60 test); RESOLVED/EXPIRED unconstrained (recurrence §51); per-type (WEAK/REVIEW/REPEATED coexist §56/57) |
| SG-02 | WEAK_SKILL v1 activation/resolution hysteresis (TD-119) | — | pure weak-skill-signal-v1 (activate mastery<5000 & conf>=7000 & evidence>=3; resolve mastery>=6500 & conf>=7000; 5000..6499 hold) | P0 | Phase 1.8C; unit §37-44 + e2e |
| SG-03 | REVIEW_DUE interval faqat current Skill state'dan (TD-120) | — | pure review-due-signal-v1 (confidence-first, mastery bands → 1/3/7/14d) | P0 | LearnerSkillState only; login/completion/roadmap EMAS; unit §45-49 |
| SG-04 | Review basis = lastMeasurementAt (elapsed duration) | — | dueAt = lastMeasurementAt + intervalDays×24h (Clock.now()); server/local timezone EMAS | P0 | DST-safe; unit §50 + e2e |
| SG-05 | Newer lastMeasurementAt eski REVIEW_DUE'ni resolve qiladi | — | current lastMeasurementAt STRICTLY > basisLastMeasurementAt | P0 | same-timestamp hold (§53); unit/e2e §52 |
| SG-06 | Signal evaluation authoritative Skill state'ni rollback QILMAYDI | — | LearningProgress recompute → evaluateStateSignals tx tashqarisida, try/catch (fail → deferred, reconcile repair) | P0 | §31/62/63 test |
| SG-07 | GET signal API read-only (vaqt uchun mutate YO'Q) | — | GET faqat activeSignalsForSubject o'qiydi; REVIEW_DUE eligibility faqat reconcile | P0 | §33 GET-no-mutation test |
| RV-01 | Review candidates faqat ACTIVE supported signal consume qiladi (TD-122) | — | whitelist REPEATED_MISTAKE/WEAK_SKILL/REVIEW_DUE; RESOLVED/EXPIRED ignore | P0 | Phase 1.9A; §48/49 test |
| RV-02 | Candidate Lesson avval encountered (TD-123) | — | LearnerLessonProgress OR LearnerLessonCompletion | P0 | §50 unseen-excluded test |
| RV-03 | Candidate Lesson hozir learner-visible | — | Lesson PUBLISHED + published revision + Topic/Module/Level/Track PUBLISHED, subject scope | P0 | §64 hidden-content test |
| RV-04 | General Skill relevance explicit relational mapping talab qiladi (TD-124) | — | LessonSkill OR current revision ActivitySkill (inference YO'Q) | P0 | §55 no-inference + §59 current-mapping test |
| RV-05 | REPEATED_MISTAKE direct trigger historical provenance'dan logical Lesson aniqlaydi | — | strict evidenceRefs parser → trigger activity → revision.lessonId | P1 | §56/57 test; malformed → skip (§70) |
| RV-06 | Candidate projection WRITE YO'Q | — | read-only module (grep) | P0 | §75 side-effect grep + count-unchanged test |
| RV-07 | Review read signal RESOLVE QILMAYDI | — | GET faqat o'qiydi; signal lifecycle LearnerSignals'da | P0 | §68 test |
| RV-08 | Review candidate executable authority EMAS | — | start/execution endpoint YO'Q; lesson start hali DailyPlanItem talab qiladi | P0 | §34/67/78 route review |
| RS-DB-01 | one ACTIVE review session per (user, skill, lesson) (TD-125) | **partial UNIQUE** `uq_review_session_active` (user_id, skill_id, lesson_id) WHERE status = 'ACTIVE' | create + P2002 winner-resume; review advisory-lock | P0 | migration `20260820150000`; Phase 1.9B-2; concurrent start → 1 ACTIVE (§84) |
| RS-DB-02 | selected Activity unique per session | **UNIQUE** (review_session_id, activity_id) | createMany atomic | P0 | immutable membership §29 |
| RS-DB-03 | selected Activity position unique per session | **UNIQUE** (review_session_id, position) | createMany atomic (1-based) | P0 | stable order §26 |
| RS-DB-04 | selected Activity position positive | **CHECK** `chk_review_session_activity_position_positive` (position > 0) | 1-based assignment | P1 | migration `20260820150000` |
| RS-01 | NEW review session current ReviewCandidate talab qiladi (TD-125) | — | assertCandidateAvailable server revalidation (§16) | P0 | §63 test |
| RS-02 | encountered revision pinned (TD-125) | — | Completion.lessonRevisionId else Progress.lessonRevisionId; repin YO'Q | P0 | §65/66/67 test |
| RS-03 | one target Skill + Lesson per session | — | session shape (skillId+lessonId) | P0 | §21 |
| RS-04 | selected Activities pinned revision + attribution policy (TD-126) | — | pure selectReviewActivities (ActivitySkill→LessonSkill fallback, objective only) | P0 | §68 unit+e2e |
| RS-05 | session Activity snapshot immutable | **UNIQUE** membership (RS-DB-02/03) | atomic create; never re-select on resume | P0 | §73 test |
| RS-06 | review attempts session provenance (TD-127) | `ActivityAttempt.review_session_id` FK + index | createReviewAttempt sets reviewSessionId | P0 | §74 test |
| RS-07 | review attempts normal Progress'ni mutate QILMAYDI | — | recordActivityStep chaqirilmaydi | P0 | §74 test (progress unchanged) |
| RS-08 | normal LessonCompletion `reviewSessionId IS NULL` filtrlaydi (TD-127) | — | submittedActivityIds where reviewSessionId=null | P0 | **§78 mandatory** — review attempt normal complete SATISFY QILMAYDI |
| RS-09 | lesson-mastery-v1 `reviewSessionId IS NULL` filtrlaydi (TD-127) | — | bestScores where reviewSessionId=null | P0 | **§79 mandatory** — review score LESSON_MASTERY'ni ALTER QILMAYDI |
| RS-10 | Review complete har selected Activity same-session attempt talab qiladi | — | attemptSummary(reviewSessionId) coverage | P0 | §80/81 test |
| RS-11 | Review complete SkillMeasurement YARATMAYDI | — | review module SkillMeasurement YOZMAYDI (grep) | P0 | §81 test |
| RS-12 | Review module LearnerSignal YOZMAYDI | — | signal service chaqirilmaydi (grep) | P0 | §90 grep + test |
| RM-DB-01 | REVIEW_MASTERY reviewSession provenance (TD-130) | `SkillMeasurement.review_session_id` FK (Restrict) + index | createReviewMeasurement reviewSessionId beradi | P0 | migration `20260820160000`; Phase 1.9C |
| RM-DB-02 | review measurement idempotency (TD-130) | **partial UNIQUE** `uq_skill_measurement_review_idempotency` (review_session_id, skill_id, source, derivation_version) WHERE review_session_id IS NOT NULL | createMany skipDuplicates | P0 | migration `20260820160000`; idempotent+concurrent complete → 1 measurement (§66/67) |
| RM-01 | REVIEW_MASTERY reviewSession provenance talab qiladi | (RM-DB-01) | ensureDerived reviewSessionId + observedAt=completedAt | P0 | §77 test |
| RM-02 | review mastery faqat COMPLETED session'dan | — | ensureDerived status=COMPLETED guard | P0 | §16 |
| RM-03 | har selected Activity bitta best-score evidence unit | — | bestReviewScores per selected activity | P0 | §8 |
| RM-04 | review evidenceCount = selected Activity soni | — | deriveReviewMastery evidenceCount = n | P0 | §64 test (retry count EMAS) |
| RM-05 | review observedAt = ReviewSession.completedAt | — | ensureDerived observedAt | P0 | §14 test |
| RM-06 | review measurement append-only/idempotent per session+skill+version | (RM-DB-02) | createMany skipDuplicates | P0 | §66 test |
| RM-07 | merge-v2 REVIEW_MASTERY'ni incremental sifatida qo'shadi (TD-131) | — | mergeSkillV2 incremental sources | P0 | §59/60 test |
| RM-08 | Review calibration anchor EMAS | — | anchorSources = {DIAGNOSTIC, CHECKPOINT} only | P0 | §20/25 test |
| RM-09 | Review module LearnerSkillState to'g'ridan YOZMAYDI (TD-115) | — | LearningProgress single writer (grep) | P0 | §75 grep |
| RM-10 | signals faqat LearnerSignals service orqali (TD-132) | — | review module signal.create/update chaqirmaydi (grep) | P0 | §83 grep |
| DP-RV-01 | auto Review EXTRA current ReviewCandidate'dan (TD-133) | — | ReviewService.getCandidates authority | P0 | Phase 2.0A; §47/48 test |
| DP-RV-02 | EXTRA core DailyPlan bilan same Topic | — | pure selector lessonTopicId == coreTopicId | P0 | §49 cross-topic excluded |
| DP-RV-03 | ko'pi bilan bitta Review EXTRA | — | selector one-item cap | P0 | §51 test |
| DP-RV-04 | core'dagi review Lesson duplicate qilinmaydi | — | selector coreLessonIds dedup | P0 | §50 test |
| DP-RV-05 | Review EXTRA optional (section=EXTRA) | — | section EXTRA, itemType REVIEW | P0 | §9/26 |
| DP-RV-06 | plan generation ReviewSession YARATMAYDI | — | daily-plan module reviewSession YOZMAYDI (grep) | P0 | §60 test |
| DP-RV-07 | same-day mavjud plan immutable | — | findCurrentPlan → resume; EXTRA faqat NEW generation | P0 | §56/57 test |
| DP-RV-08 | ReviewSession start current candidate revalidate qiladi | — | 1.9B start authority o'zgarmadi | P0 | §61 (1.9B revalidation) |
| DP-RV-09 | manual review cross-topic bo'lishi mumkin | — | ReviewSession start same-topic filtri YO'Q | P0 | §62 test |
| DP-RV-10 | review EXTRA core completion denominator'ni o'zgartirmaydi | — | read projector core = non-EXTRA only | P0 | §59 mandatory test |

**Silently dropped: 0.** SP-04 endi DB partial-unique; SP-09 derivation identity NOT NULL + non-empty; SP-10 lesson-backed measurement idempotency (Phase 1.7C); LP-01/LP-02 merge metadata (evidence_count > 0 CHECK + observed_at NOT NULL, Phase 1.8A); SG-01 one-active signal partial-unique (Phase 1.8B); SG-02..07 WEAK_SKILL/REVIEW_DUE application invariants (Phase 1.8C — SCHEMA O'ZGARMADI); RV-01..08 review-candidate read invariants (Phase 1.9A — SCHEMA O'ZGARMADI, read-only module, WRITE YO'Q); RS-DB-01..04 review-session DB constraints + RS-01..12 review-execution invariants (Phase 1.9B-2, migration `20260820150000`: LearnerReviewSession + LearnerReviewSessionActivity + ActivityAttempt.reviewSessionId + one-active partial unique + position CHECK). Side-effect boundary (Phase 1.9B-2 §90): ReviewSession faqat learnerReviewSession + learnerReviewSessionActivity + activityAttempt(reviewSessionId) YOZADI; LessonProgress/LessonCompletion/SkillMeasurement/SkillState/Roadmap/DailyPlan/LearnerSignal/Reward/Notification/AI YOZMAYDI — grep + testda tasdiqlangan. Normal completion/mastery endi reviewSessionId IS NULL filtrlaydi (isolation). RM-DB-01/02 + RM-01..10 review mastery (Phase 1.9C, migration `20260820160000`: REVIEW_MASTERY source + SkillMeasurement.reviewSessionId FK + review idempotency partial unique). Side-effect boundary (Phase 1.9C §82): review completion REVIEW_MASTERY SkillMeasurement + LearnerSkillState (LearningProgress single writer) + LearnerSignal (LearnerSignals only) YOZADI; LessonProgress/LessonCompletion/Roadmap/DailyPlan/Reward/Notification/AI YOZMAYDI; SkillMeasurement append-only (runtime UPDATE/DELETE YO'Q) — grep+testda tasdiqlangan. merge-v1 IMMUTABLE, merge-v2 current engine. DP-RV-01..10 daily review EXTRA integration (Phase 2.0A — SCHEMA O'ZGARMADI; DailyPlanItemType.REVIEW + nullable roadmapItemId/lessonId/skillId mavjud). Side-effect boundary (Phase 2.0A §69): DailyPlan generation faqat dailyPlan + dailyPlanItem YOZADI; LearnerSignal/LearnerReviewSession/ActivityAttempt/LearnerSkillState/SkillMeasurement/Roadmap/RoadmapItem/LessonProgress/LessonCompletion/Reward/Notification/AI YOZMAYDI — grep+testda tasdiqlangan. Core done/progress = roadmap-backed non-EXTRA only. Side-effect boundary (Phase 1.8A §72): LearningProgress faqat learnerSkillState yozadi (upsert, yagona writer TD-115); Roadmap/RoadmapItem/DailyPlan/LearnerSignal/RewardGrant/DailyMission/XP/IZL/AiEvaluation YOZMAYDI, SkillMeasurement runtime UPDATE/DELETE YO'Q — grep + testda tasdiqlangan. Side-effect boundary (Phase 1.8B/1.8C §66/72): LearnerSignals faqat learnerSignal yozadi (create + conditional resolve updateMany, yagona writer — REPEATED_MISTAKE + WEAK_SKILL + REVIEW_DUE); SkillMeasurement/LearnerSkillState/Roadmap/DailyPlan/LessonCompletion/Reward/DailyMission/Notification/AiEvaluation/ActivityAttempt YOZMAYDI — grep + testda tasdiqlangan. Side-effect boundary (Phase 1.5C §64/65/66): skill-profile Roadmap/RoadmapItem/LearnerSignal/XP/IZL/AiEvaluation/DailyPlan YOZMAYDI; AssessmentService o'zi LearnerSkillState/SkillMeasurement YOZMAYDI (faqat skill-profile domeni) — grep + testda tasdiqlangan. Side-effect boundary (Phase 1.7C §36/65): LessonCompletion faqat learnerLessonCompletion + learnerLessonProgress + skillMeasurement YOZADI; LearnerSkillState/Signal/reward/XP/IZL/AI/DailyPlan/RoadmapItem YOZMAYDI (grep + §65 testda tasdiqlangan).

## 9e. Roadmap foundation (Phase 1.6A)

Schema O'ZGARMADI — Phase 1.3 invariantlari qayta ishlatildi.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| RM-01 | Bir (user, subject) uchun bitta ACTIVE roadmap | **partial UNIQUE** `ux_active_roadmap` (user_id, subject_id) WHERE status=ACTIVE (L9, mavjud) | idempotency + P2002 catch | P0 | §50 concurrent; §51 conflict; §52 idempotent |
| RM-02 | Roadmap source = exact diagnostic snapshot | `sourceAssessmentAttemptId` FK-scalar | SkillMeasurement (attempt, DIAGNOSTIC, derivationVersion) | P0 | §53; current state EMAS |
| RM-03 | Content→Skill = LessonSkill (yagona mapping) | `unique(lesson, skill)` | mappedLessons | P0 | title/AI inference YO'Q; §43 |
| RM-04 | Faqat learner-visible published lesson | `ux_lesson_published_revision` (C7, mavjud) + status filtrlari | lessonMeta (Lesson+chain PUBLISHED) | P0 | §42 |
| RM-05 | Completed lesson exclude | — | completedLessonIds | P1 | in-progress qoladi; §44 |
| RM-06 | Prerequisite before dependent, cycle-safe | `unique(lesson, prereq)` | topological sort + cycle → ROADMAP_CONFIGURATION_INVALID | P0 | §15/45/46 |
| RM-07 | Roadmap header + items atomic | FK Cascade (item→roadmap) | `$transaction` | P0 | half/empty ACTIVE roadmap YO'Q; §55 |
| RM-08 | Roadmap read-only against learning state | — | grep + test | P0 | SkillState/Measurement/Progress/DailyPlan/Signal/XP/IZL YOZILMAYDI |
| RM-09 | Completion truth = LearnerLessonCompletion (RoadmapItem dublikat EMAS) | `unique(user, lesson, completionNo)` | derived read model; RoadmapItem.status yozilmaydi | P0 | Phase 1.6B (TD-101); derived state persist YO'Q |
| RM-10 | ACTIVE→COMPLETED iff har item lesson completed | `RoadmapStatus` enum + `ux_active_roadmap` | conditional updateMany (status=ACTIVE), idempotent/concurrent-safe | P0 | Phase 1.6B (TD-102); completedAt field yo'q → status only; COMPLETED→ACTIVE YO'Q |

**Silently dropped: 0.** RM-01/RM-04 mavjud DB partial-unique'lar (L9/C7). RM-09/RM-10 (Phase 1.6B) — schema o'zgarmadi; derived states/progress persist QILINMAYDI, reconciliation mavjud enum bilan.

## 9f. Daily plan foundation (Phase 1.7A)

Schema O'ZGARMADI — mavjud DailyPlan invariantlari qayta ishlatildi.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| DP-01 | Bir (user, localDate) uchun bitta CURRENT DailyPlan | **partial UNIQUE** `ux_current_daily_plan` (user_id, local_date) WHERE status=CURRENT (L16, mavjud) | idempotency + P2002 catch/resume | P0 | §35 concurrent; §39 same-day |
| DP-02 | unique(user, localDate, generationNo) | `@@unique` | nextGenerationNo (max+1) | P0 | mavjud |
| DP-03 | localDate = profile timezone (server UTC EMAS) | `@db.Date` | localDateInTimezone(clock.now, profile tz) | P0 | §36/37 |
| DP-04 | timezoneSnapshot/localDate immutable (TD-91) | — | generation'da yoziladi, keyin o'zgarmaydi | P1 | §38 |
| DP-05 | Bitta Topic per DailyPlan | — | selectPlanItems + topic invariant (fail-safe) | P0 | §20/44 |
| DP-06 | Header + items atomic | FK Cascade (item→plan) | `$transaction` | P0 | partial CURRENT plan YO'Q |
| DP-07 | DailyPlan read-only against roadmap/progress/skill/reward | — | grep + test | P0 | dailyPlan/dailyPlanItem'dan boshqa YOZILMAYDI |

**Silently dropped: 0.** DP-01/DP-02 mavjud DB constraint'lar (L16). DailyPlan = roadmap snapshot; live item state derived (1.6B reuse), persist QILINMAYDI.

## 9g. Lesson execution foundation (Phase 1.7B)

Schema O'ZGARMADI — mavjud invariantlar qayta ishlatildi.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| LE-01 | Bir (user, lesson) uchun bitta current progress | `@@unique(user_id, lesson_id)` (mavjud) | start idempotency + P2002 resume | P0 | §10/46 |
| LE-02 | Execution entry = today's CURRENT DailyPlanItem | — | findTodayPlanItem (dailyPlan userId+localDate+CURRENT) | P0 | arbitrary lesson start YO'Q; §42 |
| LE-03 | Revision pinning (first start), resume pinned | `lesson_revision_id` (LearnerLessonProgress, TD-37) | createProgress + revision id read | P0 | §8/45; archive-first republish'dan keyin ham |
| LE-04 | Bitta PUBLISHED revision per lesson (pin manbai) | `ux_lesson_published_revision` (C7, mavjud) | publishedRevisionId chain check | P0 | pin faqat published + visible chain |
| LE-05 | Faqat AVAILABLE/IN_PROGRESS start | — | deriveItemState (1.6B reuse) | P0 | COMPLETED/BLOCKED/UNAVAILABLE rad |
| LE-06 | LessonCompletion/roadmap/skill/reward YOZILMAYDI | — | grep + test | P0 | faqat learnerLessonProgress + activityAttempt (1.7B-2) |
| LE-07 | Answer-key/payload leak YO'Q | — | learner-safe projection (objective payload strip; body qaytarilmaydi) | P0 | §28/39/49 |
| LE-08 | ActivityAttempt attemptNo server-assigned | **unique(user_id, activity_id, attempt_no)** (mavjud) | maxAttemptNo + retry-on-unique | P0 | §20; concurrency-safe |
| LE-09 | ActivityAttempt durable idempotency (clientRequestId) | **partial UNIQUE** `ux_attempt_client_request` (user_id, client_request_id) WHERE NOT NULL (L5, mavjud) | replay/conflict + P2002 resume | P0 | §16/17/18; same reqId → 1 attempt |
| LE-10 | ActivityAttempt append-only (SUBMITTED) | — | faqat create; UPDATE/DELETE YO'Q | P0 | §24/49 |

**Silently dropped: 0.** LE-01/LE-04/LE-08/LE-09 mavjud DB constraint'lar (Phase 1.7B-2'da ishlatildi — schema o'zgarmadi). Side-effect boundary (§51): lesson execution roadmap/dailyplan/skill/reward/AI YOZMAYDI (faqat learnerLessonProgress + activityAttempt).

## 9h. Daily mission foundation (Phase 2.0B)

Schema O'ZGARDI — `DailyMissionCompletion` hardened: `mission_code` / `policy_version` / `local_date` (`@db.Date`) /
`timezone_snapshot` (barchasi NOT NULL) qo'shildi, `daily_plan_item_id` → nullable, `@@index(user_id, local_date)`.
Migration 11 (`20260820170000_harden_daily_mission_completion`): Prisma DDL + custom partial UNIQUE + 3 CHECK.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| DM-DB-01 | Bitta completion per (user, mission_code, local_date) | **partial UNIQUE** `uq_daily_mission_completion_day` (user_id, mission_code, local_date) WHERE local_date IS NOT NULL | createCompletion + P2002 skip | P0 | idempotency + first-evidence-wins (TD-137) |
| DM-DB-02 | mission_code bo'sh emas | **CHECK** `chk_dmc_mission_code_nonempty` (length(trim)>0) | policy registry kod | P1 | |
| DM-DB-03 | policy_version bo'sh emas | **CHECK** `chk_dmc_policy_version_nonempty` (length(trim)>0) | immutable versiya | P1 | |
| DM-DB-04 | timezone_snapshot bo'sh emas | **CHECK** `chk_dmc_timezone_snapshot_nonempty` (length(trim)>0) | resolved IANA tz | P1 | frozen provenance |
| DM-DB-05 | Evidence → qualifying ActivityAttempt | FK `daily_mission_completion_evidence_activity_attempt_id_fkey` **onDelete: Restrict** | createCompletion tx | P0 | evidence attempt'ni bloklaydi; test reset avval evidence o'chiradi |
| DM-DB-06 | Completion + evidence append-only | — | faqat create; UPDATE/DELETE YO'Q | P0 | §… reconcile ham faqat yangi satr |
| DM-01 | LEARN_TODAY = ≥1 SUBMITTED objective attempt | — | `qualifiesLearnToday` (MINI_QUESTION/PRACTICE/MASTERY_TEST + SUBMITTED) | P0 | correctness ahamiyatsiz; noto'g'ri ham hisoblanadi |
| DM-02 | MASTERY_TEST_90 = MASTERY_TEST SUBMITTED score≥9000 | — | `qualifiesMasteryTest90` | P0 | 8999 yo'q / 9000 ha / 10000 ha |
| DM-03 | Placement ISTISNO | — | AssessmentResponse ActivityAttempt EMAS (construction) | P0 | |
| DM-04 | View-only ISTISNO | — | view attempt yaratmaydi (construction) | P0 | |
| DM-05 | local_date + completedAt = qualifying attempt.submittedAt | `local_date` @db.Date | localDateInTimezone(submittedAt, profile tz) | P0 | DailyPlan bilan bir xil tz authority |
| DM-06 | tz/date/policy snapshot frozen | persisted NOT NULL ustunlar | keyingi tz o'zgarishi tarixiy satrni ko'chirmaydi | P0 | TD-91/137; e2e tz-immutability |
| DM-07 | Reconcile earliest evidence | — | eligibleAttempts submittedAt ASC, id ASC | P0 | first qualifying wins |
| DM-08 | Hook post-attempt; downstream fail attempt'ni rollback qilmaydi | — | try/catch mission hook (lesson-exec + review-session) | P0 | reconcile repair |
| DM-09 | GET read-only; own-user; answer leak YO'Q | — | controller CurrentPrincipal + learner-safe projection | P0 | cross-user IDOR yo'q; 401 |
| DM-10 | Reward/skill/signal/plan/roadmap/session/notification/AI YOZMAYDI | — | grep + e2e side-effect counts | P0 | faqat 2 mission jadval |

**Silently dropped: 0.** DM-DB-01..04 migration 11'da CREATE + verified; DM-DB-05 mavjud Restrict FK; append-only (DM-DB-06) repository write-path (mission completion — §10 "append-only" ro'yxatiga qo'shildi). **NO RewardGrant/XP/IZL** (2.0B'da reward bridge YO'Q — Phase 2.0C). Boundary: mission module faqat `daily_mission_completion` + `daily_mission_completion_evidence` yozadi.

## 9i. Daily mission → XP reward (Phase 2.0C-2)

Schema O'ZGARDI — `XpGrant` hardened (mission-XP vehicle, TD-45 accepted model): +`daily_mission_completion_id`
(nullable typed FK Restrict), +`policy_version_code` (nullable). **RewardGrant / IZL / XpBalance TEGILMADI.**
Migration 12 (`20260820180000_add_xp_grant_mission_provenance`): Prisma DDL + custom partial UNIQUE + 2 CHECK.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| XP-DB-01 | Bitta XP grant per mission completion | **partial UNIQUE** `uq_xp_grant_mission_completion` (daily_mission_completion_id) WHERE daily_mission_completion_id IS NOT NULL | createMissionXpGrant + P2002 skip | P0 | policy version uniqueness'ga KIRMAYDI (v1+v2 double-pay oldini olish, TD-141) |
| XP-DB-02 | Mission provenance = typed FK | FK `xp_grant_daily_mission_completion_id_fkey` **onDelete: Restrict** → DailyMissionCompletion | createMissionXpGrant | P0 | source_refs JSONB reward authority EMAS (TD-92); XpGrant → completion o'chirilishini bloklaydi |
| XP-DB-03 | policy_version_code bo'sh emas (present bo'lsa) | **CHECK** `chk_xp_grant_policy_version_nonempty` (NULL OR btrim<>'') | mission producer har doim `daily-mission-xp-reward-v1` | P1 | non-mission XpGrant NULL qoladi |
| XP-DB-04 | Mission-backed XP strictly positive | **CHECK** `chk_xp_grant_mission_amount_positive` (daily_mission_completion_id IS NULL OR amount>0) | policy amount 10/20 | P1 | non-mission ±/correction satrlar buzilmaydi (broad amount>0 CHECK YO'Q) |
| XP-01 | Mission XP faqat DailyMissionCompletion'dan | — | XpRepository.missionCompletion; ActivityAttempt to'g'ridan XP bermaydi | P0 | ActivityAttempt→Completion→XpGrant chain |
| XP-02 | Completion user = XpGrant user | — | ensureMissionXpGranted cross-user guard → XpRewardConfigurationInvalidError | P0 | grant.userId = completion.userId |
| XP-03 | LEARN_TODAY v1 → 10 | — | evaluateDailyMissionXp | P0 | |
| XP-04 | MASTERY_TEST_90 v1 → 20 | — | evaluateDailyMissionXp | P0 | |
| XP-05 | Unsupported mission producer version → grant YO'Q | — | policy (code+version pair) | P0 | learn-today-mission-v2 v1 amount OLMAYDI |
| XP-06 | policyVersionCode = daily-mission-xp-reward-v1 | — | policy result | P0 | immutable provenance |
| XP-07 | Mission XpGrant append-only | — | faqat create; UPDATE/DELETE YO'Q | P0 | grep |
| XP-08 | Retry/concurrency → bir grant | XP-DB-01 | P2002 skip | P0 | |
| XP-09 | Historical reconcile frozen identity/version | — | reconcile completion.{missionCode,policyVersion}'dan | P0 | processing-time YO'Q |
| XP-10 | RewardGrant/IZL TEGILMAYDI | — | grep + e2e count | P0 | XP≠IZL (TD-142) |
| XP-11 | XpGrant joriy XP source of truth | — | totalXp = SUM(XpGrant.amount) | P0 | XpBalance authority EMAS |
| XP-12 | XpBalance YOZILMAYDI | — | grep + e2e count unchanged | P0 | deferred cache (TD-143) |
| XP-13 | XP failure mission'ni rollback qilmaydi | — | bridge try/catch (tryEnsureMissionXpGranted) | P0 | reconcile repair (TD-144) |

**Silently dropped: 0.** XP-DB-01 migration 12'da CREATE + verified; XP-DB-02 typed FK Restrict; append-only (XP-07)
repository write-path (mission XP — §10 "append-only" ro'yxatiga qo'shildi). **NO IZL / RewardGrant / XpBalance
writes.** Boundary: XP module faqat `xp_grant` yozadi (single writer — xp.repository).

## 9j. XP progression / XpBalance projection (Phase 2.0D)

Schema O'ZGARDI — `XpBalance` activated as rebuildable projection: +`progression_version_code` (nullable),
`current_level` default 0→**1** (level floor). `XpGrant` XP source of truth bo'lib qoladi. Migration 13
(`20260820190000_activate_xp_balance_projection`): Prisma DDL + 2 custom CHECK. **RewardGrant/IZL/XpGrant TEGILMADI.**

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| XPP-DB-01 | progression_version_code bo'sh emas (present bo'lsa) | **CHECK** `chk_xp_balance_progression_version_nonempty` | projector har doim `xp-progression-v1` | P1 | NULL = stale/unversioned |
| XPP-DB-02 | Level floor 1 (Level 0 YO'Q) | **CHECK** `chk_xp_balance_current_level_min` (current_level ≥ 1) + default 1 | levelForXp min 1 | P1 | progressionXp=max(total,0) → level≥1 |
| XPP-01 | XpGrant SUM = canonical XP total | — | totalXp = SUM(XpGrant.amount) | P0 | XpBalance authority EMAS |
| XPP-02 | XpBalance XP YARATMAYDI | — | projector faqat SUM'dan derive | P0 | XpGrant→XpBalance only |
| XPP-03 | XpBalance total = signed canonical sum (recompute'dan keyin) | — | recomputeProjection | P0 | |
| XPP-04 | currentLevel faqat xp-progression-v1'dan | — | computeXpProgression | P0 | |
| XPP-05 | Progression basis negative total'ni 0'ga clamp | — | progressionXp = max(totalXp,0) | P0 | totalXp signed qoladi |
| XPP-06 | Level ≥ 1 | XPP-DB-02 | levelForXp | P0 | |
| XPP-07 | Projection negative correction'dan keyin kamayishi mumkin | — | full recompute | P0 | highestLevelEver YO'Q |
| XPP-08 | GET XP projection'ni mutate QILMAYDI | — | getProgression (read-only, no upsert) | P0 | canonical from SUM |
| XPP-09 | Projection failure XpGrant'ni rollback qilmaydi | — | tryRecomputeProjection non-throwing | P0 | TD-148 |
| XPP-10 | Reconcile to'liq history'dan rebuild | — | recomputeProjection full SUM | P0 | |
| XPP-11 | Level change zero reward/entitlement side effect | — | grep + e2e | P0 | no event/grant/notification |
| XPP-12 | XpBalance single writer = XP projection module | — | grep (xp.repository yagona xpBalance.upsert) | P0 | |

**Silently dropped: 0.** XPP-DB-01/02 migration 13'da CREATE + verified. **NO RewardGrant / IZL / new XpGrant from
progression.** Boundary: XP module `xp_grant` (create) + `xp_balance` (upsert) YOZADI — yagona writer (xp.repository).

## 9k. IZL economic reward (Phase 2.1A)

**SCHEMA O'ZGARMADI** — mavjud finance invariantlar qayta ishlatildi (RewardGrant/IZLLedgerEntry Phase 1.3'da
tayyor). First real-value earning producer: DailyMissionCompletion → RewardGrant → IZLLedgerEntry (atomik).
IZLWallet YOZILMAYDI (ledger canonical, wallet deferred). Migration count o'zgarmaydi (13).

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| IZL-DB-01 | RewardGrant dedup idempotency | **unique** `reward_grant_user_id_dedup_key_key` (F-5, mavjud) | dedupKey=`daily-mission-izl:<completionId>` + findUnique + P2002 | P0 | bir IZL grant per completion (§18/§19) |
| IZL-DB-02 | Ledger 1:1 RewardGrant | **unique** `izl_ledger_entry_reward_grant_id_key` (mavjud) | atomik post; existing → return | P0 | orphan ledger/double-credit YO'Q (§34) |
| IZL-DB-03 | Ledger entryNo per-user monotonic | **unique** `izl_ledger_entry_user_id_entry_no_key` (mavjud) | MAX(entryNo)+1 lock ostida | P0 | wallet-local ordering (TD-47) |
| IZL-DB-04 | Mission RewardGrant provenance FK | FK `missionCompletionId` **Restrict** + `subscriptionCycleId`/`rewardPolicyVersionId` NOT NULL (mavjud) | createGrant | P0 | audit chain load-bearing |
| IZL-DB-05 | Reward/ledger amount positive (mission) | — (blanket CHECK YO'Q; ledger signed) | policy parser amount>0; producer 1 IZL | P1 | REQUIRES_APP (signed ledger REDEEM/ADJUSTMENT buzilmaydi) |
| IZL-01 | Mission IZL faqat DailyMissionCompletion'dan | — | RewardRepository.missionCompletion | P0 | ActivityAttempt to'g'ridan IZL bermaydi |
| IZL-02 | MASTERY_TEST_90 v1 yagona reward (v1) | — | IZL_SUPPORTED_MISSION_CODES + policy | P0 | |
| IZL-03 | LEARN_TODAY → 0 IZL | — | policy'da absent → not eligible | P0 | |
| IZL-04 | Historical covering cycle shart | — | periodStart≤completedAt<periodEnd | P0 | yo'q → grant yo'q |
| IZL-05 | Cycle snapshot policy authority | FK cycle.rewardPolicyVersionId | parse cycle.policyVersion.config | P0 | current ACTIVE EMAS (§15) |
| IZL-06 | Daily cap 1 IZL | — | SUM(GRANTED) per localDate+cycle < cap (lock) | P0 | |
| IZL-07 | Cycle DAILY_MISSION cap 30 IZL | — | SUM(GRANTED) per cycle < cap (lock) | P0 | |
| IZL-08 | Cap serialized economic tx | advisory lock (`pg_advisory_xact_lock('izl', userId)`) | materializeMissionReward | P0 | concurrent final-cap race safe |
| IZL-09 | RewardGrant + ledger atomik | one `$transaction` | grant+ledger birga | P0 | orphan YO'Q |
| IZL-10 | Bir completion double-grant YO'Q | IZL-DB-01 | dedup + existing check | P0 | |
| IZL-11 | Mission/XP IZL fail'da rollback QILINMAYDI | — | tryEnsureMissionReward non-throwing | P0 | independent branches |
| IZL-12 | Ledger canonical IZL truth | — | balanceIzl=SUM(ledger.amount) | P0 | RewardGrant SUM EMAS |
| IZL-13 | GET read-only | — | getBalance (no write) | P0 | wallet ham yozilmaydi |
| IZL-14 | Redemption/reservation/payment YO'Q | — | grep + test | P0 | earn+read only (TD-155) |

**Silently dropped: 0.** IZL-DB-01..04 mavjud constraint'lar (Phase 1.3). IZL-DB-05 REQUIRES_APP (signed ledger uchun
blanket CHECK xavfli). Boundary: Finance module `reward_grant` + `izl_ledger_entry` (create, atomik) YOZADI — yagona
writer; IZLWallet/XpGrant/XpBalance/Subscription/cycle/Payment/reservation/redemption YO'Q.

## 9l. IZL wallet projection + reservation (Phase 2.1B)

Schema O'ZGARDI — migration 14 (`20260820200000_activate_izl_wallet_reservation`): **new `IZLReservation` table**
(hold primitive; recon: dedicated reservation model YO'Q edi — IZLRedemption redemption/payment entity, qayta
ishlatilmadi), `IZLWallet` +`projection_version_code`, `chk_wallet_balance_nonneg` + `chk_wallet_reserved_le_balance`
DROP (wallet endi signed projection). `IZLLedgerEntry` accounting authority bo'lib qoladi.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| RES-DB-01 | Reservation amount > 0 | **CHECK** `chk_izl_reservation_amount_positive` | canReserve amount>0 | P0 | zero/negative hold YO'Q |
| RES-DB-02 | Idempotency identity | **unique** `izl_reservation_user_id_idempotency_key_key` | findUnique + P2002 | P0 | bir hold per (user, key) |
| RES-DB-03 | Idempotency key non-empty | **CHECK** `chk_izl_reservation_idempotency_nonempty` | server-generated key | P1 | |
| RES-DB-04 | Provenance immutable | — | app: amount/user/purpose/key create'dan keyin o'zgarmaydi | P0 | faqat status→RELEASED update |
| RES-DB-05 | Status lifecycle | enum `IzlReservationStatus` (ACTIVE/RELEASED) | ACTIVE→RELEASED only | P0 | CONSUMED enum'da YO'Q (§29) |
| WL-01 | Ledger SUM = canonical balance | — | ledgerBalance aggregate | P0 | TD-150 authority |
| WL-02 | ACTIVE reservation SUM = reserved | — | activeReservedTotal | P0 | |
| WL-03 | available = balance − reserved | — | computeIzlBalance | P0 | signed |
| WL-04 | Wallet rebuildable projection | IZLWallet (signed; nonneg CHECKs dropped) | recomputeWallet | P0 | authority EMAS |
| WL-05 | GET source-derived + read-only | — | getBalances (no write) | P0 | stale cache'ga ishonmaydi |
| WL-06 | Wallet fail authoritative'ni rollback qilmaydi | — | tryRecompute non-throwing | P0 | TD-160 |
| WL-07 | Full canonical recompute | — | SUM ledger + ACTIVE reservation | P0 | incremental EMAS |
| RES-01 | New reservation amount>0 | RES-DB-01 | canReserve | P0 | |
| RES-02 | amount <= max(available,0) | — | canReserve (lock ostida) | P0 | over-reserve/negative-deepen YO'Q |
| RES-03 | Reservation wallet cache'ga ishonmaydi | — | ledger+reservation SUM under lock | P0 | §23 |
| RES-04 | Serialized cap/create | advisory lock `pg_advisory_xact_lock('izl',userId)` | createReservation | P0 | §50 shared namespace |
| RES-05 | Idempotent replay double-reserve QILMAYDI | RES-DB-02 | findUnique/P2002 | P0 | |
| RES-06 | Reservation ledger DEBIT QILMAYDI | — | grep + test | P0 | §35 |
| RES-07 | Release ledger CREDIT QILMAYDI | — | grep + test | P0 | §36 |
| RES-08 | ACTIVE→RELEASED only (v1) | RES-DB-05 | release | P0 | |
| RES-09 | CONSUMED transition YO'Q (v1) | enum yo'q | — | P0 | §29 |
| RES-10 | Runtime DELETE YO'Q | — | grep (create/update only) | P0 | §33 |

**Silently dropped: 0.** Migration 14: +3 CHECK (2 reservation + 1 wallet projection), **−2 CHECK** (wallet nonneg +
reserved≤balance DROP — signed projection §41/§43), +1 table (izl_reservation), +1 unique (idempotency). CHECK 33→34.
Boundary: reservation/wallet module `izl_reservation` (create/update) + `izl_wallet` (upsert) YOZADI — yagona writerlar;
ledger/RewardGrant/XP/Subscription/Payment/Redemption YO'Q.

## 9m. PaymentOrder subscription purchase intent (Phase 2.1C-PO)

Schema O'ZGARDI — migration 15 (`20260820210000_payment_order_purchase_intent`): `PaymentOrder.provider` NOT NULL →
**nullable** (provider-agnostic purchase authority); +partial UNIQUE idempotency. Pricing/discount/payable semantics
o'zgarmadi (mavjud chk_order_amounts saqlanadi). Subscription/Cycle/PaymentTransaction/IZL TEGILMADI.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PO-DB-01 | Bir order per (user, clientRequestId) | **partial UNIQUE** `uq_payment_order_client_request` (user_id, client_request_id) WHERE NOT NULL | orderByClientRequest + P2002 replay | P0 | network idempotency (TD-170) |
| PO-DB-— | payable = gross − discount | **CHECK** `chk_order_amounts` (mavjud) | create discount=0/payable=gross | P0 | reused, o'zgarmadi |
| PO-01 | v1 purpose = SUBSCRIPTION_PURCHASE only | — | server-fixed | P0 | RENEWAL/upgrade deferred |
| PO-02 | PlanPrice server-resolved | — | currentPlanPrice (effectiveFrom≤now DESC) | P0 | client price EMAS |
| PO-03 | Price snapshot immutable | planPriceId FK + grossAmount | create only, update yo'q | P0 | reprice YO'Q |
| PO-04 | Initial IZL discount = 0 | — | createOrder izlDiscountAmount=0 | P0 | |
| PO-05 | Initial payable = gross | chk_order_amounts | payable=gross | P0 | |
| PO-06 | Provider not required | provider nullable | provider=NULL | P0 | TD-168 |
| PO-07 | No PaymentTransaction on create | — | grep + test | P0 | |
| PO-08 | No Subscription/Cycle before payment | — | grep + test | P0 | l.269 accepted |
| PO-09 | Replay preserves original price | PO-DB-01 | replayId | P0 | reprice YO'Q |
| PO-10 | Conflicting key rejected | — | replayId planId/purpose check → conflict | P0 | |
| PO-11 | No IZL redemption/reservation | — | grep + test | P0 | 2.1C-2 |
| PO-12 | Reward ceiling never spending authority | — | discount ceiling = PaymentOrder.grossAmount (TD-171) | P0 | earning≠spending |

**Silently dropped: 0.** Migration 15: provider DROP NOT NULL + 1 partial unique (PO-DB-01); 0 yangi CHECK (mavjud
chk_order_amounts reused). Boundary: payments module `payment_order` (create) YOZADI — yagona writer (payments.repository);
PaymentTransaction/Subscription/SubscriptionCycle/PlanPrice/Plan/IZL/XP YO'Q.

## 9n. Subscription discount redemption intent (Phase 2.1C-2)

Schema O'ZGARDI — migration 16 (`20260820220000_subscription_discount_redemption_intent`): IZLRedemption
+clientRequestId/policyVersionCode, paymentOrderId FK **SetNull→Restrict**; IZLReservation +redemptionId (typed
UNIQUE FK Restrict). IZLLedgerEntry TEGILMADI. Reserve-only (ledger debit / APPLIED / CONSUMED YO'Q).

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| RD-DB-01 | Bir redemption per (user, clientRequestId) | **partial UNIQUE** `uq_izl_redemption_client_request` WHERE client_request_id IS NOT NULL | replay + P2002 | P0 | network idempotency (TD-175) |
| RD-DB-02 | Reservation→redemption typed FK | FK `izl_reservation_redemption_id_fkey` **Restrict** | createRedemption tx | P0 | load-bearing 1:1 provenance |
| RD-DB-03 | Bir reservation per redemption | **UNIQUE** `izl_reservation_redemption_id_key` (nullable → non-null unique) | 1:1 | P0 | |
| RD-DB-04 | One OPEN SUBSCRIPTION_DISCOUNT per order | **partial UNIQUE** `uq_izl_redemption_open_per_order` (payment_order_id) WHERE type=SUBSCRIPTION_DISCOUNT AND status IN (REQUESTED,RESERVED) | open check + P2002 | P0 | RELEASED → yangi mumkin |
| RD-DB-05 | policyVersionCode nonempty (present bo'lsa) | **CHECK** `chk_izl_redemption_policy_version_nonempty` | producer subscription-discount-redemption-v1 | P1 | |
| RD-DB-06/07/08 | amount/rate/value > 0 | **CHECK** amount_positive / rate_positive / value_positive | policy + producer | P1 | jadval bo'sh edi → safe |
| RD-DB-— | PaymentOrder provenance | FK `izl_redemption_payment_order_id_fkey` **Restrict** (SetNull'dan) | createRedemption | P0 | purchase authority o'chirilmaydi |
| RD-01 | v1 type SUBSCRIPTION_DISCOUNT only | — | fixed | P0 | |
| RD-02..04 | own PaymentOrder, SUBSCRIPTION_PURCHASE + CREATED, not expired | — | createRedemption eligibility | P0 | §14/15/17 |
| RD-05/06 | ceiling base = order gross; floor(gross×2000/10000) | — | maxDiscountUzs | P0 | reward ceiling EMAS |
| RD-07 | earning ceiling never spending authority | — | grep | P0 | |
| RD-08/09 | rate server-resolved + snapshot immutable | — | ACTIVE IzlRateVersion; replay no-reprice | P0 | |
| RD-10 | available = ledger − ACTIVE holds (not wallet) | — | canonical SUM under lock | P0 | §26/§40 |
| RD-11/12 | atomic RESERVED + ACTIVE, typed 1:1 | one `$transaction` | createRedemption | P0 | |
| RD-13/14 | replay preserves quote; one-open per order | RD-DB-01/04 | replayId | P0 | |
| RD-15/16 | reserve zero ledger; order price unchanged | — | grep + test | P0 | izlRedemptionId NULL |
| RD-17/18 | release atomic RELEASED+RELEASED, zero ledger | one `$transaction` | releaseRedemption | P0 | |
| RD-19/20/21 | no APPLIED / no CONSUMED / izlRedemptionId NULL | — | grep + test | P0 | 2.1D |
| RD-22 | wallet fail never rolls back intent/hold | — | tryRecompute non-throwing | P0 | |

**Silently dropped: 0.** Migration 16: +2 partial unique (RD-DB-01/04) +1 plain unique (RD-DB-03) +4 CHECK
(RD-DB-05/06/07/08); paymentOrder FK SetNull→Restrict; reservation.redemptionId typed FK. CHECK 34→38, partial unique
21→23, plain unique +1. Boundary: redemption module `izl_redemption` + `izl_reservation` (create/update) YOZADI;
ledger/RewardGrant/PaymentOrder/PaymentTransaction/Subscription/XP YO'Q.

## 9o. Subscription discount commit (Phase 2.1D)

Schema O'ZGARDI — migration 17 (`20260820230000_subscription_discount_commit`): custom SQL only (izl_redemption_id
ustuni mavjud edi) — +1 partial unique. IZLRedemption lifecycle / IZLReservation enum / ledger / PaymentTransaction
TEGILMADI. Commit faqat PaymentOrder pricing'ni bog'laydi (IZL spend YO'Q).

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| DC-DB-01 | Non-null izlRedemptionId bir redemption'ni narxlaydi | **partial UNIQUE** `uq_payment_order_izl_redemption` (izl_redemption_id) WHERE NOT NULL | commit + conflict check | P0 | discount stacking YO'Q (§19) |
| DC-DB-— | payable = gross − discount | **CHECK** `chk_order_amounts` (mavjud) | commit payable=gross−value | P0 | reused |
| DC-01 | Commit faqat RESERVED redemption'ni consume qiladi | — | createRedemption status check | P0 | |
| DC-02 | Typed reservation ACTIVE | — | commit reservation validate | P0 | |
| DC-03 | redemption.paymentOrderId == committed order | — | service tx | P0 | §20 |
| DC-04 | Commit order CREATED talab qiladi | — | commit order status | P0 | |
| DC-05 | Frozen valueUzs, current rate re-resolve YO'Q | — | commit reads redemption.valueUzs | P0 | §12 |
| DC-06 | Ceiling frozen gross'dan revalidate | — | maxDiscountUzs(gross) | P0 | corruption guard |
| DC-07/08/09 | order discount=value, payable=gross−value, pointer=redemption | — | paymentOrder.update | P0 | |
| DC-10/11 | redemption RESERVED / reservation ACTIVE saqlanadi | — | commit no lifecycle update | P0 | §4/§5 |
| DC-12/13 | commit zero ledger; reserved/available o'zgarmaydi | — | grep + test | P0 | §6/§7 |
| DC-14 | commit idempotent | DC-DB-01 | already-committed detect | P0 | |
| DC-15 | committed release (CREATED) order restore + hold release atomik | one `$transaction` | releaseRedemption | P0 | §25 |
| DC-16 | release zero ledger | — | grep + test | P0 | |
| DC-17/18/19/20 | no APPLIED / no CONSUMED / no PaymentTransaction / no Subscription | — | grep + test | P0 | 2.1D+ |

**Silently dropped: 0.** Migration 17: +1 partial unique (DC-DB-01); 0 yangi CHECK (chk_order_amounts reused). Boundary:
redemption module `payment_order`(update commit/release) + `izl_redemption`/`izl_reservation`(release update) YOZADI;
IZLLedgerEntry/RewardGrant/PaymentTransaction/Subscription/XP YO'Q. PaymentOrder writer endi payments(create) +
redemption(commit/release update).

## 9p. Payment execution intent (Phase 2.1E)

Schema O'ZGARDI — migration 18 (`20260820240000_payment_execution_intent`): `PaymentTransaction.provider_transaction_id`
**DROP NOT NULL** (internal PENDING attempt external init'dan OLDIN yaratiladi → id keyin attach) + `client_request_id`
ustuni qo'shildi + custom SQL 2 partial unique (PT-DB-01/02). PaymentTransaction faqat payments modulida yoziladi
(create + provider-init attach). PaymentCallbackEvent/PAID/SUCCEEDED/IZL/Subscription TEGILMADI.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PT-DB-01 | Bir attempt per (order, clientRequestId) | **partial UNIQUE** `uq_payment_transaction_client_request` (payment_order_id, client_request_id) WHERE NOT NULL | resolveOrCreateAttempt + P2002 replay | P0 | network idempotency (TD-187) |
| PT-DB-02 | One PENDING attempt per order | **partial UNIQUE** `uq_payment_transaction_pending` (payment_order_id) WHERE status='PENDING' | one-PENDING race + P2002 | P0 | second concurrent provider YO'Q (TD-187) |
| PT-DB-03 | Bir (provider, providerTransactionId) global | **UNIQUE** `@@unique([provider, provider_transaction_id])` (NULLs distinct) | attachProviderInit (P2002 → leave PENDING) | P0 | mavjud; NULL-many pending mumkin |
| PT-DB-— | payable = gross − discount | **CHECK** `chk_order_amounts` (mavjud) | amount=payableAmount snapshot | P0 | reused (order) |
| PT-01 | Attempt faqat CREATED order uchun | — | eligibility status=CREATED | P0 | §7 |
| PT-02 | purpose SUBSCRIPTION_PURCHASE | — | eligibility purpose | P0 | §6 |
| PT-03 | expired order rad | — | now < expiresAt | P0 | §8/§75 |
| PT-04 | payableAmount > 0 | — | eligibility payable | P0 | §11/§77 |
| PT-05 | amount = payableAmount (gross EMAS) | — | create amount snapshot | P0 | §66 (TD-186) |
| PT-06 | currency = PaymentOrder.currency (derive) | — | initiation view | P1 | PT'da currency ustuni YO'Q |
| PT-07 | order CREATED → PENDING atomik | one `$transaction` | resolveOrCreateAttempt | P0 | §33 |
| PT-08 | PaymentOrder.provider NULL qoladi | — | grep + test | P0 | §32 (TD-185) |
| PT-09 | provider = PaymentTransaction.provider authority | — | create provider | P0 | TD-185 |
| PT-10 | committed discount integrity revalidate | — | assertCommittedDiscountIntegrity | P0 | §44/§64 |
| PT-11 | RESERVED redemption + ACTIVE hold + ledger unchanged | — | grep + test | P0 | §64 (no spend) |
| PT-12 | provider.initiate DB tx'dan TASHQARIDA | — | service (tx#1 → call → tx#2) | P0 | §24/§26 |
| PT-13 | provider id attach idempotent (PENDING+NULL only) | PT-DB-03 | attachProviderInit no-op guard | P0 | §27/§29 |
| PT-14 | ambiguous init → PENDING, no id, retryable | — | service catch + retry | P0 | §30/§72/§73 |
| PT-15 | idempotent replay → bitta attempt | PT-DB-01 | replayAttempt | P0 | §67 |
| PT-16 | different provider + same key → conflict | — | replayAttempt provider check | P0 | §68 (409) |
| PT-17 | per-user IZL lock (commit/release bilan serialize) | `pg_advisory_xact_lock` | both tx | P0 | §25 |
| PT-18 | no PAID / SUCCEEDED / callback / IZL / subscription | — | grep + test | P0 | §86 (2.1F) |

**Silently dropped: 0.** Migration 18: 1 column nullable (provider_transaction_id) + 1 column add (client_request_id) +
2 partial unique (PT-DB-01/02); 0 yangi CHECK. PT-DB-03 mavjud unique reused. Boundary: payments module
`payment_transaction`(create + attach) + `payment_order`(create + CREATED→PENDING) YOZADI; PaymentCallbackEvent /
PAID / SUCCEEDED / IZLLedgerEntry / IZLReservation / IZLRedemption / Subscription / Cycle / XP YO'Q. PaymentTransaction
yagona writer = payments.repository.

## 9q. Verified payment evidence (Phase 2.1F)

Schema O'ZGARDI — migration 19 (`20260820250000_verified_payment_evidence`): custom SQL only (confirmed_at /
provider_transaction_id / payment_callback_event allaqachon mavjud) — +1 partial unique PV-DB-01. Trusted-success marker
= PaymentTransaction.status=SUCCEEDED (adapter-verified callback + integrity check'lardan keyin). PaymentOrder PENDING
qoladi; IZL/Subscription TEGILMADI.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PV-DB-01 | Bir order per SUCCEEDED transaction | **partial UNIQUE** `uq_payment_transaction_succeeded` (payment_order_id) WHERE status='SUCCEEDED' | SUCCESS_CONFLICT proactive + P2002 backstop | P0 | one successful charge authority (TD-193) |
| PV-DB-— | callback dedup | **UNIQUE** `@@unique(provider, provider_event_id)` (F-19, mavjud) | dedup findUnique + insert | P0 | reused |
| PV-DB-— | external identity | **UNIQUE** `@@unique(provider, provider_transaction_id)` (mavjud) | attach/equality/idOwner check | P0 | reused |
| PV-01 | provider adapter callback verify egallaydi | — | port.verifyCallback | P0 | §4 (TD-190) |
| PV-02 | invalid/unverified callback zero business write | — | service throws pre-DB + test | P0 | §10/§57 |
| PV-03 | merchant PT identity exact resolve | — | recordVerifiedCallback findUnique(merchantId) | P0 | §7/§61 (guessing yo'q) |
| PV-04 | provider = PaymentTransaction.provider | — | provider match | P0 | §62 |
| PV-05 | external provider tx id immutable once attached | PV-DB-— | attach-if-null / equality / no overwrite | P0 | §16/§64 |
| PV-06 | verified amount = PaymentTransaction.amount | — | amount check | P0 | §22/§58 |
| PV-07 | transaction amount = PaymentOrder.payableAmount | — | order-payable check | P0 | §22/§59 corruption guard |
| PV-08 | verified currency = PaymentOrder.currency | — | currency check (PT'da currency yo'q) | P0 | §23/§60 |
| PV-09 | first accepted success PENDING→SUCCEEDED | — | conditional transition | P0 | §9/§20 |
| PV-10 | confirmedAt = trusted provider evidence | — | set verified.confirmedAt (server now/client emas) | P0 | §20/§21/§68 |
| PV-11 | callback event + transition ATOMIK | one `$transaction` | recordVerifiedCallback | P0 | §14 no split state |
| PV-12 | provider event replay idempotent | PV-DB-— (F-19) | dedup reconstruct | P0 | §29/§65 |
| PV-13 | one SUCCEEDED per order | PV-DB-01 | proactive + backstop | P0 | §25/§27 |
| PV-14 | matching repeated success = terminal no-op | — | already-SUCCEEDED DUPLICATE | P0 | §28/§66 |
| PV-15 | PaymentOrder PENDING qoladi | — | callback path order write yo'q | P0 | §31 grep+test |
| PV-16 | no IZL mutation | — | grep + test | P0 | §35/§72 |
| PV-17 | no Subscription/Cycle | — | grep + test | P0 | §73 |
| PV-18 | no client success authority | — | learner route yo'q + test | P0 | §45 |

**Silently dropped: 0.** Migration 19: +1 partial unique (PV-DB-01); 0 column, 0 yangi CHECK. F-19 + external-id unique
reused. Boundary: payments module `payment_callback_event`(create) + `payment_transaction`(update PENDING→SUCCEEDED +
confirmedAt + external-id attach) YOZADI; PaymentOrder(status/pricing/provider) / IZLLedgerEntry / IZLReservation /
IZLRedemption / Subscription / Cycle / XP / Notification YO'Q. FAILED/CANCELLED/REFUNDED producer YO'Q (§74). Callback
path yagona SUCCEEDED writer + yagona PaymentCallbackEvent writer = payments.repository.

## 9r. Finalization contract + schema hardening (Phase 2.1G-D)

Schema O'ZGARDI — migration 20 (`20260821000000_finalization_schema_hardening`): PlanPrice +`billing_period_months`
(backfill 1, NOT NULL); SubscriptionCycle `reward_policy_version_id` + `izl_rate_snapshot` NOT NULL→**nullable**
(reward-disabled cycle); `IzlReservationStatus` enum +CONSUMED; +2 CHECK + 1 partial unique custom SQL. NO finalizer,
NO PaymentOrder PAID, NO Subscription/Cycle producer, NO REDEEM/CONSUMED/APPLIED producer. 2.1A earning: reward-disabled
no-op + economic ceiling enforcement.

| ID | Invariant | DB mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| FP-DB-01 | Billing duration > 0 | **CHECK** `chk_plan_price_billing_period_positive` | producer (2.1G) | P0 | immutable PlanPrice (TD-195) |
| FP-DB-02/03 | reward-config coherence | **CHECK** `chk_cycle_reward_config_coherent` | cycle producer (2.1G) | P0 | enabled ⟺ policy+rate(>0); disabled ⟺ NULL+zero ceilings (TD-198) |
| FP-DB-04 | One REDEEM per redemption | **partial UNIQUE** `uq_izl_ledger_redeem_per_redemption` (redemption_id) WHERE NOT NULL AND entry_type='REDEEM' | REDEEM producer (2.1G) | P0 | global redemption_id UNIQUE EMAS (TD-201) |
| FP-01 | Billing duration = immutable PlanPrice | PlanPrice column | — | P0 | TD-195 |
| FP-02 | v1 price duration = 1 calendar month | backfill | — | P1 | migration |
| FP-03 | future periodStart = confirmedAt | — | 2.1G | P0 | §35 |
| FP-04 | future periodEnd = calendar-month add | — | addCalendarMonths (pure) | P0 | end-of-month clamp §48 |
| FP-05 | reward basis = payable (net) | — | 2.1G cycle producer | P0 | TD-197 |
| FP-06 | reward ceiling UZS = floor(net×20%) | — | rewardCeilingUzs (pure) | P0 | §49 |
| FP-07 | reward ceiling IZL = floor(uzs/rate) | — | rewardCeilingIzl (pure) | P0 | §50, no round-up |
| FP-08 | reward config never blocks paid access | FP-DB-02/03 nullable | reward-disabled cycle | P0 | TD-198 |
| FP-09 | reward-disabled cycle issues no IZL | — | reward.repository disabled no-op | P0 | §14/§52 |
| FP-10 | 2.1A respects min(policy cap, cycle ceiling) | — | effectiveCycleCapIzl | P0 | §17/§53/§54 (TD-199) |
| FP-11 | RewardGrant SUM = earning authority | — | cycle GRANTED aggregate | P0 | earnedIzl non-authoritative §16/§19 |
| FP-12 | CONSUMED ≠ RELEASED | enum | — | P0 | audit §34 (TD-201) |
| FP-13 | no runtime CONSUMED producer | — | grep + test | P0 | §21 |
| FP-14 | one REDEEM/redemption DB authority | FP-DB-04 | — | P0 | §56 |
| FP-15 | no runtime REDEEM producer | — | grep + test | P0 | §24 |
| FP-16 | expiry does not block SUCCEEDED finalization | — | 2.1G | P0 | TD-196 |
| FP-17 | ACTIVE conflict ≠ silent renewal | — | 2.1G | P0 | TD-200 |
| FP-18 | EXPIRED episode reactivatable | ux_nonterminal_subscription (mavjud F-14) | 2.1G | P0 | TD-200 |
| FP-19 | finalization never re-authorizes ACTIVE hold | — | 2.1G | P0 | TD-203 |
| FP-20 | lock order sub → pay → izl | advisory locks | 2.1G | P0 | TD-202 |

**Silently dropped: 0.** Migration 20: +1 column (billing_period_months) + 2 columns nullable + 1 enum value (CONSUMED)
+ 2 CHECK (FP-DB-01/02-03) + 1 partial unique (FP-DB-04). CHECK 38→40, partial unique 27→28. Reserved SUM query
(`status=ACTIVE`) o'zgarmadi (RELEASED + CONSUMED exclude). Boundary: schema + 2.1A earning cap YOZADI; PaymentOrder PAID
/ Subscription / SubscriptionCycle / IZL REDEEM / reservation CONSUMED / redemption APPLIED runtime producer YO'Q.

## 9s. Verified payment economic finalization (Phase 2.1G)

Schema O'ZGARMADI — **YANGI MIGRATION YO'Q** (post-migration-20 schema yetarli). Finalizer mavjud DB authoritylarni
reuse qiladi (yangi constraint qo'shmaydi). Single finalization writer: PaymentOrder(PAID) + Subscription + Cycle +
CycleEntitlement (+ discounted: REDEEM + CONSUMED + APPLIED). PaymentTransaction/PaymentCallbackEvent finalizer
tomonidan yozilmaydi; provider call yo'q.

| ID | Invariant | DB / mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PF-01 | SUCCEEDED PT = finalization authority | PV-DB-01 (one SUCCEEDED/order) | finalize precondition | P0 | §4 |
| PF-02 | no provider call | — | grep + test | P0 | §69 |
| PF-03 | first finalization needs PENDING order | — | status check | P0 | §6 |
| PF-04 | expiry does not block trusted paid finalization | — | no expiry check | P0 | §7 (TD-196) |
| PF-05 | PT.amount = order.payableAmount | — | final re-check | P0 | §12 |
| PF-06 | periodStart = confirmedAt | — | finalize | P0 | §16 |
| PF-07 | periodEnd = purchased billingPeriodMonths | PlanPrice.billingPeriodMonths | addCalendarMonths | P0 | §17 |
| PF-08 | commercial snapshots from PaymentOrder only | — | cycle create | P0 | §26 no reprice |
| PF-09 | no nonterminal → new ACTIVE episode | F-14 | activation decision | P0 | §18 |
| PF-10 | EXPIRED → same episode reactivated | F-14 | activation decision | P0 | §19 |
| PF-11 | reactivation may update planId, not startedAt | — | subscription update | P0 | §19/§20 |
| PF-12 | ACTIVE → recoverable conflict | F-14 | SubscriptionPurchaseActiveConflictError | P0 | §21 rollback |
| PF-13 | CANCELLED never reactivated | F-14 (excludes CANCELLED) | activation decision | P0 | §22 |
| PF-14 | one paid order → one cycle | SubscriptionCycle.paymentOrderId UNIQUE | create | P0 | §14/§48 |
| PF-15 | cycle entitlements snapshotted | SubscriptionCycleEntitlement unique(cycle,feature) | createMany | P0 | §31/§32 |
| PF-16 | reward config cannot block paid access | FP-DB-02/03 | reward-disabled fallback | P0 | §28 |
| PF-17 | reward basis = payable/net | — | cycle create | P0 | §17 (TD-197) |
| PF-18 | reward ceilings use floor rules | — | reward-ceiling helpers | P0 | §29 |
| PF-19 | discounted path requires exact RESERVED+ACTIVE provenance | reservation.redemptionId UNIQUE | assert provenance | P0 | §35 |
| PF-20 | no available-IZL reauthorization | — | no availability check | P0 | §36 (TD-203) |
| PF-21 | REDEEM amount = −redemption.amountIzl | — | ledger create | P0 | §37 |
| PF-22 | REDEEM + CONSUMED + APPLIED atomic | one `$transaction` | finalize | P0 | §22–§24 |
| PF-23 | redemption resolvedAt = confirmedAt | — | redemption update | P0 | §24/§40 |
| PF-24 | available invariant across consume | ACTIVE-only reserved SUM | — | P0 | §41 |
| PF-25 | signed negative ledger allowed | — | balanceAfter | P0 | §25/§42 |
| PF-26 | PAID commits only with complete finalization | one `$transaction` | order update last | P0 | §43/§44 |
| PF-27 | duplicate finalizer → no duplicate effects | order PAID + cycle unique + FP-DB-04 | reconstruct | P0 | §48/§49 |
| PF-28 | wallet is downstream cache only | — | post-commit tryRecompute | P0 | §60 |
| PF-29 | PT SUCCEEDED survives finalizer failure | tx rollback | grep + test | P0 | §62/§98 |
| PF-30 | lock order sub → pay → optional izl | advisory locks | finalize | P0 | §9 (TD-202) |

**Silently dropped: 0.** No new DB object. Reused authorities: PV-DB-01, SubscriptionCycle.paymentOrderId UNIQUE, F-14
(ux_nonterminal_subscription), FP-DB-04, izl_reservation.redemption_id UNIQUE, izl_ledger_entry (user,entry_no) UNIQUE.
Boundary (§102 grep+test): finalizer writes payment_order(PAID) + subscription + subscription_cycle +
subscription_cycle_entitlement (+ discounted izl_ledger_entry REDEEM / izl_reservation CONSUMED / izl_redemption APPLIED)
+ izl_wallet (downstream post-commit); NEVER payment_transaction / payment_callback_event / reward_grant / xp / usage_counter
/ plan* / notification. No provider call.

## 9t. Verified payment finalization recovery (Phase 2.1H)

Schema O'ZGARMADI — **YANGI MIGRATION YO'Q** (count 20). Operational recovery: SUCCEEDED PT + PENDING order backlog'ni
mavjud finalizer orqali retry qiladi. Recovery moduli **hech qanday direct writer emas** (faqat findMany/count read +
finalizer delegate); provider call yo'q. Yangi permissionlar (RolePermission data, migration emas): payments.finalization.read/reconcile.

| ID | Invariant | Mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| FR-01 | backlog = SUCCEEDED PT + PENDING order | PV-DB-01 (one SUCCEEDED/order) | BACKLOG_WHERE query | P0 | §4 |
| FR-02 | reconcile payment success'ni establish qilmaydi | — | 2.1F authority | P0 | §2 |
| FR-03 | mavjud finalizer yagona business writer | — | grep + test | P0 | §3 (TD-211) |
| FR-04 | reconcile provider'ni chaqirmaydi | — | grep + spy test | P0 | §20/§60 |
| FR-05 | batch bounded (default 50, max 200) | — | clampLimit + DTO @Max | P0 | §6 |
| FR-06 | deterministik oldest-confirmed-first | — | orderBy confirmedAt ASC, id ASC | P1 | §5 |
| FR-07 | bitta item failure boshqasini rollback qilmaydi | — | per-item try/catch, no outer tx | P0 | §30/§31 |
| FR-08 | BLOCKED ≠ FAILED | — | SubscriptionPurchaseActiveConflictError→BLOCKED | P0 | §10 |
| FR-09 | ACTIVE-sub conflict recoverable | — | 2.1G rollback | P0 | §29 |
| FR-10 | transient failure SUCCEEDED/PENDING saqlaydi | 2.1G atomik tx | test | P0 | §11 |
| FR-11 | bridge/reconcile race finalizer idempotency orqali converge | finalizer locks + uniques | test | P0 | §21/§57 |
| FR-12 | overlapping reconcilerlar safe | finalizer authority | test | P0 | §22 (no mutex) |
| FR-13 | learner recovery authority yo'q | AuthGuard + PermissionsGuard | 403/401 test | P0 | §18 |
| FR-14 | backlog read read-only | — | findMany/count only + test | P0 | §13 |
| FR-15 | avtomatik refund/release/cancel yo'q | — | grep + test | P0 | §44 |
| FR-16 | PaymentTransaction mutation yo'q | — | grep | P0 | §27 |
| FR-17 | PaymentCallbackEvent mutation yo'q | — | grep | P0 | §28 |
| FR-18 | duplicate wallet projection yo'q | — | finalizer owns recompute | P1 | §40 |

**Silently dropped: 0.** No new DB object, no migration. Reused: PV-DB-01, SubscriptionCycle.paymentOrderId UNIQUE, F-14,
FP-DB-04 (finalizer idempotency). Boundary (§63 grep+test): recovery module writes NOTHING directly (read + finalizer
delegate); no provider call. New permission codes (payments.finalization.read/reconcile) = RolePermission data,
ops-granted (no bootstrap seed, no migration).

## 9u. Verified non-success payment evidence (Phase 2.1I)

Schema O'ZGARMADI — **YANGI MIGRATION YO'Q** (count 20). Mavjud PaymentTransactionStatus FAILED/CANCELLED +
PaymentCallbackEvent.result (free String) + F-19 unique + pay lock yetarli. Trusted definitive provider terminal
evidence PT PENDING→FAILED/CANCELLED qiladi; PaymentOrder PENDING qoladi; IZL/Subscription/finalizer TEGILMADI.

| ID | Invariant | Mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PN-01 | faqat verified provider evidence PT terminalize qiladi | — | verifyCallback + assertVerifiedEvent | P0 | §PN-01 |
| PN-02 | ambiguous evidence terminalize qilmaydi | — | terminal!==true → reject (no write) | P0 | §5/§43 |
| PN-03 | FAILED = definitive terminal unsuccessful | — | adapter terminal=true kafolat | P0 | §4 (TD-217) |
| PN-04 | CANCELLED = provider-confirmed terminal | — | adapter authority | P0 | §7 (TD-218) |
| PN-05 | provider expiry → FAILED + PROVIDER_EXPIRED | — | reasonCode → result | P1 | §8/§42 |
| PN-06 | invalid callback ≠ trusted non-success | — | adapter throw vs normalized | P0 | §10 |
| PN-07 | merchant PT identity exact | — | findUnique(merchantId) | P0 | §11 |
| PN-08 | provider identity exact | — | provider match | P0 | §12 |
| PN-09 | external PT identity immutable | @@unique(provider, provider_transaction_id) | attach/equality/idOwner | P0 | §13/§54 |
| PN-10 | callback dedup F-19 | @@unique(provider, provider_event_id) | dedup findUnique | P0 | §15 |
| PN-11 | callback evidence + PT transition ATOMIK | one `$transaction` | recordTerminalNonSuccess | P0 | §17 |
| PN-12 | terminal PT → different terminal YO'Q | — | pt.status!==PENDING → conflict/no-op | P0 | §19 (TD-219) |
| PN-13 | contradictory late event = conflict evidence only | — | TERMINAL_STATUS_CONFLICT record | P0 | §21/§22/§47-49 |
| PN-14 | PaymentOrder PENDING qoladi | — | order write yo'q + test | P0 | §27 |
| PN-15 | discount hold RESERVED/ACTIVE qoladi | — | grep + test | P0 | §28/§51 |
| PN-16 | ledger movement yo'q | — | grep + test | P0 | §28 |
| PN-17 | order reopen (PENDING→CREATED) yo'q | — | grep + test (initiate rejects) | P0 | §29/§53 |
| PN-18 | retry attempt (new PT) yo'q | — | grep + test | P0 | §30 |
| PN-19 | non-success finalizer chaqirmaydi | — | bridge SUCCEEDED-only + test | P0 | §31 |

**Silently dropped: 0.** No new DB object, no migration. Reused: PaymentTransactionStatus FAILED/CANCELLED, F-19,
@@unique(provider, provider_transaction_id), pay advisory lock. amount/currency asimmetriyasi: SUCCESS exact equality,
non-success optional-when-present (§14). Boundary (§56 grep+test): non-success path `payment_callback_event`(create) +
`payment_transaction`(PENDING→FAILED/CANCELLED + external-id attach) YOZADI; payment_order / subscription* / izl* /
reward_grant / xp / notification / finalizer YO'Q; confirmedAt success-only; failedAt/cancelledAt/reason column YO'Q.

## 9v. Payment order reopen / retry (Phase 2.1J)

Schema O'ZGARMADI — **YANGI MIGRATION YO'Q** (count 20). Terminal FAILED/CANCELLED PT o'z PENDING order'ini PENDING→CREATED
qiladi (retryable), keyin mavjud initiate flow yangi attempt yaratadi. Reopen faqat `PaymentOrder.status` yozadi;
provider call yo'q. Mavjud DB authoritylarni reuse qiladi (PT-DB-02 one-PENDING, PV-DB-01 one-SUCCEEDED).

| ID | Invariant | Mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| PR-01 | faqat FAILED/CANCELLED PT reopen authorize qiladi | — | reload + status check | P0 | §5 |
| PR-02 | reopen internal/server-owned | — | learner route yo'q | P0 | §4/§61 |
| PR-03 | reopen faqat PaymentOrder.status yozadi | — | grep + count test | P0 | §3/§19/§85 |
| PR-04 | first transition PENDING→CREATED | — | reopen | P0 | §9 |
| PR-05 | reopen commit'da PENDING PT bo'lmasligi | — | pendingCount check under lock | P0 | §10 duplicate-charge himoyasi |
| PR-06 | reopen commit'da SUCCEEDED PT bo'lmasligi | — | succeeded findFirst | P0 | §11 |
| PR-07 | PAID order hech qachon reopen | — | status check | P0 | §12/§50 |
| PR-08 | CREATED replay idempotent (no live/success) | — | ALREADY_REOPENED | P0 | §13 |
| PR-09 | stale old terminal ≠ newer PENDING overwrite | pay(order) lock | RETRY_ALREADY_IN_PROGRESS | P0 | §48/§73 |
| PR-10 | stale old terminal ≠ SUCCEEDED/PAID overwrite | — | precedence checks | P0 | §49/§50 |
| PR-11 | order expiry reopen'ni bloklamaydi | — | expiresAt tekshirilmaydi | P0 | §18/§81 |
| PR-12 | retry writer = mavjud initiate | — | reopen new PT yaratmaydi | P0 | §29/§42 |
| PR-13 | fresh retry yangi clientRequestId | PT-DB-01 | initiate | P0 | §30/§76 |
| PR-14 | eski clientRequestId eski terminal PT | PT-DB-01 | resolveOrCreateAttempt prior-check | P0 | §31/§77 |
| PR-15 | provider per fresh retry o'zgarishi mumkin | — | PaymentTransaction.provider | P1 | §33/§78 |
| PR-16 | commercial pricing frozen | — | reopen faqat status | P0 | §19 |
| PR-17 | discount hold RESERVED/ACTIVE qoladi | — | grep + test | P0 | §20/§79 |
| PR-18 | ledger movement yo'q | — | grep + test | P0 | §79 |
| PR-19 | CREATED-state redemption release reuse | 2.1D | release after reopen test | P0 | §22/§80 |
| PR-20 | terminal callback commit reopen'dan oldin | — | separate tx bridge | P0 | §24 |
| PR-21 | reopen bridge failure evidence'ni saqlaydi | — | non-throwing + test | P0 | §25/§71 |
| PR-22 | avtomatik new charge yo'q | — | reopen PT yaratmaydi | P0 | §42 |

**Silently dropped: 0.** No new DB object, no migration. Reused: PT-DB-02 (one PENDING/order), PV-DB-01 (one SUCCEEDED/
order), pay advisory lock, PT-DB-01 (initiate idempotency). Boundary (§91 grep+test): reopen module FAQAT
`payment_order`(PENDING→CREATED status) yozadi; payment_transaction / payment_callback_event / izl* / subscription* /
reward / xp / notification YO'Q; provider call YO'Q. PaymentOrder FAILED/CANCELLED/EXPIRED producer YO'Q.

## 9w. Terminal payment reopen recovery (Phase 2.1K)

Schema O'ZGARMADI — **YANGI MIGRATION YO'Q** (count 20). Terminal FAILED/CANCELLED PT + PENDING order stuck state'ni
mavjud PaymentOrderReopenService orqali recover qiladi. Recovery moduli **hech qanday direct writer emas** (read + reopen
delegate); provider call yo'q. Yangi permissionlar (RolePermission data, migration emas): payments.reopen.read/reconcile.

| ID | Invariant | Mechanism | App check | Severity | Notes |
|---|---|---|---|---|---|
| RR-01 | backlog = terminal PT + PENDING order (no live/success) | — | BACKLOG_WHERE some FAILED/CANCELLED + none PENDING/SUCCEEDED | P0 | §5 |
| RR-02 | reconcile provider failure'ni establish qilmaydi | — | 2.1I authority | P0 | §2 |
| RR-03 | mavjud PaymentOrderReopenService yagona reopen writer | — | grep + test | P0 | §41 (TD-228) |
| RR-04 | provider call yo'q | — | grep + spy test | P0 | §17/§18 |
| RR-05 | automatic new attempt yo'q | — | PT count test | P0 | §18/§57 |
| RR-06 | batch bounded (default 50/max 200) | — | clampLimit + DTO @Max | P0 | §8 |
| RR-07 | deterministik oldest-first | — | order.createdAt ASC, id ASC | P1 | §7 |
| RR-08 | item failure keyingilarni to'xtatmaydi | — | per-item try/catch | P0 | §55 |
| RR-09 | bridge/reconcile race safe | pay(order) lock | test | P0 | §27/§50 |
| RR-10 | overlapping reconcilerlar safe | reopen idempotency | test | P0 | §28 |
| RR-11 | newer PENDING PT stale reopen'ni to'xtatadi | reopen revalidate | RETRY_ALREADY_IN_PROGRESS | P0 | §29/§52 |
| RR-12 | SUCCEEDED PT reopen'ni to'xtatadi | reopen revalidate | PAYMENT_SUCCESS_PENDING_FINALIZATION | P0 | §30/§53 |
| RR-13 | PAID order reopen'ni to'xtatadi | reopen revalidate | ALREADY_PAID | P0 | §54 |
| RR-14 | expiry reopen'ni bloklamaydi | — | test | P0 | §31/§49 |
| RR-15 | discount hold unchanged | — | grep + test | P0 | §32/§48 |
| RR-16 | ambiguous PENDING PT candidate emas | — | backlog none PENDING | P0 | §35 |
| RR-17 | learner reopen/reconcile authority yo'q | AuthGuard + PermissionsGuard | 403/401 test | P0 | §26/§58 |
| RR-18 | recovery moduli PT/callback/IZL/subscription mutation yo'q | — | grep + test | P0 | §60 |

**Silently dropped: 0.** No new DB object, no migration. Reused: PaymentOrderReopenService (yagona reopen writer),
PT-DB-02, PV-DB-01, pay lock. Domain separation (TD-232): SUCCEEDED+PENDING→2.1H finalization recovery,
FAILED/CANCELLED+PENDING→2.1K reopen recovery (bir-birini chaqirmaydi). Boundary (§60 grep+test): recovery module direct
write YO'Q (read + reopen delegate); provider call yo'q. New permission codes (payments.reopen.read/reconcile) =
RolePermission data, ops-granted, finalization permissionlaridan ajratilgan.

## 9x. Real provider protocol persistence + binding (Phase 2.1L-D)

**Migration 21** (`20260821100000_real_provider_protocol_persistence`, count 20→21, named CHECK 40→45). Provider-specific
typed protocol persistence (TD-233) — adapter durable state, iqtisodiy authority EMAS (core money core PT/order/IZL'da
qoladi §25). Payme facts official developer.help.paycom.uz'dan VERIFIED; CLICK provider-neutral shell — **CLICK PROTOCOL
VERIFICATION BLOCKER** (§0), native constant yo'q. NO real adapter/route/provider call/refund/terminal transition.

New tables: `payme_merchant_transaction` (1:1 PaymentTransaction), `click_shop_transaction` (1:1 PaymentTransaction).

| ID | Invariant | Mechanism | Severity | Notes |
|---|---|---|---|---|
| PMT-DB-01 | one Payme protocol row per PaymentTransaction | `@unique payment_transaction_id` | P0 | 1:1 core attempt |
| PMT-DB-02 | Payme transaction id (params.id) globally unique | `@unique payme_transaction_id` | P0 | external identity dedup |
| PMT-DB-03 | native state ∈ {1,2,-1,-2} | CHECK chk_payme_mt_state_valid | P0 | verified states; -2 forward-compat only |
| PMT-DB-04 | amount tiyin > 0 (BigInt, no coercion) | CHECK chk_payme_mt_amount_positive | P0 | verified tiyin unit |
| PMT-DB-05 | perform/cancel times coherent with state | CHECK chk_payme_mt_time_coherent | P0 | GetStatement/CheckTransaction reconstruction |
| PMT-DB-06 | reason present ⟺ cancel state (-1/-2) | CHECK chk_payme_mt_reason_coherent | P1 | provider reason snapshot |
| PMT-DB-07 | GetStatement range over Payme creation time | index (provider_created_time_ms) | P1 | §8 — not local createdAt |
| PMT-FK | protocol row → PaymentTransaction | FK onDelete Restrict | P0 | append-only protocol state |
| CST-DB-01 | one CLICK protocol row per PaymentTransaction | `@unique payment_transaction_id` | P0 | 1:1 core attempt |
| CST-DB-02 | click_trans_id unique when present | partial UNIQUE uq_click_shop_click_trans_id WHERE NOT NULL | P0 | external identity dedup (NULLs distinct) |
| CST-DB-03 | ACCEPTED Complete requires ACCEPTED Prepare | CHECK chk_click_shop_complete_requires_prepare | P0 | Izlan phase model, NOT a CLICK native constant |
| CST-FK | protocol row → PaymentTransaction | FK onDelete Restrict | P0 | append-only protocol state |
| PB-01 | binding is non-terminal (only provider_transaction_id) | pay(order) lock; no status/order/IZL write | P0 | TD-234; grep + count test |
| PB-02 | external-id attach idempotent / conflict-safe | PT-DB-03 unique + attach-or-verify | P0 | BOUND/ALREADY_BOUND/CONFLICT |

**Silently dropped: 0.** No CLICK native-value CHECK (BLOCKER §0 — signature/amount/error/native-type deferred to 2.1L-C).
No secret / auth header / raw callback body column (§24). state -2 (post-success reversal) allowed by CHECK for forward
compatibility but never produced (REFUND_DOMAIN_UNSUPPORTED, TD-238). Reused: pay(order) advisory lock, PT-DB-03.

## 10. Phase 1.3 — actual implementation mapping

> Prisma Schema v1 ([PRISMA_SCHEMA_V1.md](PRISMA_SCHEMA_V1.md)) yozildi va `izlan_dev`ga migratsiya qilindi. Har invariant klassi quyidagicha map qilindi. **Silently dropped: 0.**

| Invariant klassi | Mexanizm | Holat | Izoh |
|---|---|---|---|
| PK, oddiy UNIQUE (phone, dedup_key, sequence_no, token_hash, storage_key, code...) | `@id` / `@unique` / `@@unique` | **IMPLEMENTED_PRISMA** | 151 unique index DB'da |
| FK + onDelete (RESTRICT/CASCADE/SET NULL) | `@relation onDelete` | **IMPLEMENTED_PRISMA** | 170 FK; delete policy §8 bo'yicha |
| Composite index (parent+order, user+time...) | `@@index` | **IMPLEMENTED_PRISMA** | access-path indexlar |
| Basis points 0..10000 (L-*), reserved≤balance, earned≤ceiling, amounts≥0, period ordering, entitlement mode↔limit, schedule 30..360, payment arithmetic, ADJUSTMENT reason+actor, prereq self-ref | PostgreSQL CHECK | **IMPLEMENTED_DB** | 23 CHECK; verification 21/21 PASS |
| XOR (AiEvaluation L1, Reaction K1, Report K3, MissionEvidence L26) | PostgreSQL CHECK (`(a IS NOT NULL)::int + ... = 1`) | **IMPLEMENTED_DB** | verified |
| One PUBLISHED revision / ACTIVE roadmap / CURRENT plan / non-terminal subscription / ACTIVE policy / ACTIVE rate | partial UNIQUE index (custom SQL) | **IMPLEMENTED_DB** | 11 partial unique; verified |
| Reaction/report dedup (nullable-target NULL semantics K2/K4), client_request_id (L5) | partial UNIQUE index (custom SQL) | **IMPLEMENTED_DB** | to'g'ri NULL semantics |
| Append-only (ledger F-3, measurement L-8, completion L-14, roadmap change L-12, moderation K9, reputation C-16, staff audit S-1, mission completion) | repository write-path discipline; ixtiyoriy DB PRIV | **REQUIRES_APP + deferred PRIV** | PRIV = deployment role dizayni (hozir yo'q) |
| Cross-row parent-consistency (published revision C6, current version, current cycle, accepted reply K5) | service validation (PG composite-FK — ixtiyoriy keyin) | **REQUIRES_APP_VALIDATION** | Prisma FK cross-row equality bermaydi |
| Prerequisite DAG cycle (C10) | saqlash transaction'ida cycle check | **REQUIRES_APP_VALIDATION** | + self-ref CHECK (DB) |
| Submit/publish immutability (L-1, L-3, C8, F-16, cycle snapshot F-7) | state machine + repository | **REQUIRES_STATE_MACHINE** | |
| Recommendation-gated roadmap change (L-4) | service guard + FK | **REQUIRES_APP_VALIDATION** | |
| Wallet/cycle concurrency (ledger entry_no monotonic F-2/F-18, ceiling F-6, reserved F-17) | row lock + atomik transaction | **REQUIRES_TRANSACTION** | CHECK'lar backstop |
| Payment callback idempotency (F-19) | unique(provider, event_id) + atomik processing | **IMPLEMENTED_DB (unique) + REQUIRES_TRANSACTION** | verified unique |
| Frontend activation yo'q (F-9), module boundary (C-8/K-15) | API/trust boundary, module dizayni | **REQUIRES_APP_VALIDATION** | |
| Content payload validation (C14, TD-22), media↔junction sinxron (C17b) | discriminated-union validation (publish/submit) | **REQUIRES_APP_VALIDATION** | validation library — Phase 1.4+ |
| Timezone snapshot (TD-91) | app: resolved IANA tz core learning oldidan | **REQUIRES_APP_VALIDATION** | |

**Xulosa:** IMPLEMENTED_PRISMA + IMPLEMENTED_DB — barcha DB-enforceable invariantlar migratsiyada mavjud va verification'da tasdiqlangan. REQUIRES_* — application/service qatlamiga (Phase 1.4+), har biri sabab bilan hujjatlangan; **DEFERRED_WITH_REASON**: append-only PRIV (deployment role dizayni kerak). Hech bir invariant izsiz tashlanmagan.
