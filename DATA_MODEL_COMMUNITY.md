# Izlan — Community, Moderation, Announcements & Notifications Data Model

> Status: Phase 1.2C-2 — **COMPLETE** (owner review o'tdi). Prisma-ready **architecture** — schema EMAS, migration EMAS, code EMAS.
> **TD-61..TD-80 hammasi ACCEPTED** ([TECH_DECISIONS.md](TECH_DECISIONS.md)); TD-65/67/74/77 owner clarificationlari shu hujjatga kiritilgan.
> Product asos: [COMMUNITY.md](COMMUNITY.md), [ANNOUNCEMENTS_NOTIFICATIONS.md](ANNOUNCEMENTS_NOTIFICATIONS.md). **Barcha data architecture fazalari (1.2A, 1.2B, 1.2C-1, 1.2C-2) — COMPLETE.**

## 1. Goals

1. Learning-oriented community (post/reply/reaction/accepted answer/report/reputation) uchun minimal, evolvable model.
2. Minors safety: DM yo'q, private ma'lumot expose qilinmaydi, moderation/report birinchi kundan modelda.
3. Reputation ≠ XP ≠ IZL chegarasi modelda qat'iy; community oddiy faoliyati IZL bermaydi.
4. Announcement va Notification — alohida, provider/channel-independent tizimlar.
5. Social-network overengineering yo'q: DM, follow, groups, video, ML feed — scope'dan tashqarida.

## 2. Scope / non-scope

**Scope (NOW):** CommunityPost, CommunityReply, ReactionType, CommunityReaction, CommunityPostMedia, CommunityReport, ModerationAction, CommunityRestriction, ReputationBalance, ReputationEvent, Announcement, AnnouncementUserState, Notification.

**Non-scope:** private DM, group/live chat, follow/groups/spaces, community video, ranking/recommendation ML, game currency, cash-out, payment, reward ledger redesign (1.2C-1 qayta ochilmaydi). UserBlock — OPEN (§35), NotificationDelivery — channel tanlangach (§26).

## 3. Community principles (accepted)

```
O'rgan → savol ber → tushuntir → boshqalarga yordam ber → bilimingni mustahkamla
```

5 post type (Question/Learned/Explanation/Discussion/Other); media: text/image/audio, **video yo'q**; feed, replies, reactions, accepted answer, report, reputation, subject/topic association; minors bor — DM yo'q, phone/email public emas.

## 4. Posts

### CommunityPost (TD-61)

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| author_user_id | FK → User |
| type | enum: QUESTION / LEARNED / EXPLANATION / DISCUSSION / OTHER — 5 type accepted va stable, DB enum yetarli (yangi type = product qarori + migration, normal) |
| title | nullable — Question uchun foydali; majburiylik UX bosqichida |
| body | text (§5) |
| subject_id | nullable FK → Subject (§6) |
| topic_id | nullable FK → Topic (§6) |
| visibility | enum: VISIBLE / AUTHOR_REMOVED / MODERATOR_HIDDEN (§11) |
| accepted_reply_id | nullable FK → CommunityReply (§8) |
| reply_count, reaction_count | cached int (§17) |
| edited_at | nullable (§11) |
| created_at, updated_at | |

**Post vs Reply — alohida entitylar** (bitta self-referencing jadval emas): maydonlar farqli (type/subject/accepted answer faqat postda), moderation/query'lar aniqroq, tasodifiy deep-threading strukturaviy imkonsiz.

## 5. Body format

| Variant | Baho |
|---|---|
| **A — plain text** | Xavfsiz (render'da escape — XSS yo'q), sodda editing, mobile-friendly ✅ |
| B — sanitized markdown subset | Foydali, lekin sanitizer xatosi = XSS xavfi; MVP uchun shart emas |
| C — rich JSON document | Overengineered editor talab qiladi — rad |
| D — HTML | XSS xavfi eng yuqori — rad |

**PROPOSED (TD-62): plain text** (satr ko'chishi saqlanadi, render'da escape). Storage `text` bo'lgani uchun keyinchalik markdown-subset **rendering** darajasida qo'shilsa storage o'zgarmaydi (future enhancement). Media — attachment orqali (§10), body ichida emas.

## 6. Subject/topic association

**PROPOSED (TD-63): optional bitta Subject + optional bitta Topic** (topic bo'lsa subject majburiy va topic o'sha subject zanjiriga tegishli — invariant C-3). N:M tagging va freeform hashtag tizimi — rad (MVP scope; lesson↔community integratsiya uchun bitta topic yetarli: lesson sahifasidan o'z topic'idagi postlarga chiqiladi, "Learned" mission postlari topic bilan bog'lanadi).

## 7. Replies / nesting

### CommunityReply

| Field | Izoh |
|---|---|
| id, post_id | FK |
| author_user_id | FK |
| body | text (§5 bilan bir xil format) |
| visibility | VISIBLE / AUTHOR_REMOVED / MODERATOR_HIDDEN |
| edited_at | nullable |
| created_at, updated_at | |

**Nesting (TD-61): flat replies.** Tahlil: unlimited tree — moderation/UX/minors murakkabligi, accepted requirement emas; one-level reply-to-reply — hozircha keraksiz murakkablik. Flat model eng sodda; `parent_reply_id` keyin qo'shilsa — additive migration (evolvable). Reply'da media MVP'da yo'q (product docs postlar uchun belgilagan) — kengaytirish additive.

## 8. Accepted Answer

**PROPOSED (TD-64): `CommunityPost.accepted_reply_id`** — alohida AcceptedAnswer entity YO'Q (audit/history talab financial darajada emas; almashtirish tarixini saqlash MVP uchun ortiqcha — reputation korreksiyasi event modelida hal bo'ladi, §15).

Invariantlar: faqat `type=QUESTION` postda; reply o'sha postga tegishli bo'lishi shart; faqat post author o'rnatadi/almashtiradi (application guard); hidden reply accepted bo'la olmaydi.

## 9. Reactions

**Reaction turlari (TD-65, ACCEPTED):** exact ro'yxat FINAL emas (Helpful, Clear, Great Explanation — misollar) → DB enum'ga lock qilinmaydi. **Seeded/configured `ReactionType` jadvali:** id, code (unique), label, sort_order, is_active (bool).

**Owner clarification:** reaction **code/history semantikasi immutable** — mavjud qatorning semantic ma'nosi o'zgartirilmaydi; faqat **display label** o'zgarishi mumkin; yangi reaction qo'shiladi; eski reaction `is_active=false` bilan yangi foydalanishdan chiqariladi — **existing reaction history saqlanadi** (o'chirilmaydi, remap qilinmaydi).

### CommunityReaction

| Field | Izoh |
|---|---|
| id, user_id | |
| post_id / reply_id | nullable FK'lar — **roppa-rosa bittasi** (XOR — AiEvaluation'dagi qabul qilingan pattern; polymorphic target_type/id rad — FK integrity yo'qoladi; alohida PostReaction+ReplyReaction jadvallari — dublikat mexanizm) |
| reaction_type_id | FK → ReactionType |
| created_at | |

**Uniqueness:** `unique(user_id, post_id, reaction_type_id)` va `unique(user_id, reply_id, reaction_type_id)` (partial) — user bir target'ga **har turdan bittadan** reaction bera oladi (Helpful + Clear birga mumkin; bir tur ikki marta — yo'q). Olib tashlash = row delete (bu financial emas — tarix shart emas); double-click → unique constraint no-op (§29).

## 10. Media attachments

**PROPOSED (TD-66):** TD-25 **MediaAsset reuse** — Community'ga hard-code qilinmaydi.

### CommunityPostMedia

| Field | Izoh |
|---|---|
| post_id + media_asset_id | unique pair |
| position | int |
| created_at | |

- Ruxsat etilgan turlar: **image, audio** — mime allowlist attach paytida tekshiriladi; **video attach strukturaviy rad etiladi** (C-1).
- Audio language learning uchun muhim (talaffuz, speaking sample, javob) — duration/size limitlar **OPEN**, model policy'ni hard-code qilmaydi (MediaAsset metadata'da duration bor, limit — application config).
- Ko'rinish sharti: asset READY + moderation BLOCKED emas (§20).

## 11. Editing / removal

**Editing — capability ham, exact policy ham PRODUCT OPEN (owner clarification, TD-67)** — bu bo'limda hech narsa editing bo'yicha ACCEPTED emas:

- Model **future supportga tayyor** bo'lishi mumkin (`edited_at`, report'dagi `content_snapshot`) — lekin bu editing ruxsat etilgani degani emas.
- **Full revision history MVP uchun talab qilinmaydi.**
- **Moderation context himoyasi:** report yaratilganda content'ning o'sha paytdagi matni report ichida **snapshot** qilinadi (§12) — kelajakda edit ruxsat etilsa ham report konteksti yo'qolmaydi.

**Removal — semantic visibility model (TD-67), hard delete YO'Q:**

| Holat | Ma'no |
|---|---|
| VISIBLE | Normal |
| AUTHOR_REMOVED | Muallif o'chirdi — feed/public'da ko'rinmaydi; DB'da qoladi (replies, reports, reputation, moderation konteksti buzilmaydi) |
| MODERATOR_HIDDEN | Moderation yashirdi — faqat moderation kontekstida ko'rinadi |

Global `deleted_at` copy qilinmaydi — visibility semantikasi community'ga mos. User erasure/anonymization — OPEN legal policy bilan (§30).

## 12. Reports

### CommunityReport (TD-68)

| Field | Izoh |
|---|---|
| id, reporter_user_id | |
| post_id / reply_id | nullable FK — XOR (§9 pattern) |
| category_code | string (application registry — report sabab taksonomiyasi final emas, enum emas) |
| free_text | nullable |
| content_snapshot | text — report paytidagi body (edit'dan himoya, §11) |
| status | OPEN / ACTIONED / DISMISSED |
| resolved_by, resolved_at | nullable |
| created_at | |

**Duplicate himoya:** `unique(reporter_user_id, post_id)` / `unique(reporter_user_id, reply_id)` (partial) — bir user bir contentni bir marta report qiladi (spam himoyasi); qayta urinish → no-op.

## 13. Moderation

**PROPOSED (TD-69):** MVP'da **ModerationCase aggregation jadvali YO'Q** — reportlar to'g'ridan-to'g'ri ko'rib chiqiladi (bitta content'ning barcha reportlari query bilan guruhlangan holda ko'rsatiladi). Case entity — hajm oshsa additive.

### ModerationAction (append-only)

| Field | Izoh |
|---|---|
| id, actor_user_id | Moderator |
| action_type | enum/kod: HIDE_CONTENT / RESTORE_CONTENT / WARNING / COMMUNITY_RESTRICTION / DISMISS_REPORT ... (punishment policy OPEN — ro'yxat kengayadi) |
| post_id / reply_id / target_user_id | nullable — action nishoni |
| report_id | nullable FK — qaysi report asosida |
| reason | text |
| created_at | |

**Audit chegarasi:** ModerationAction = **community domain business history** ("bu content nega hidden"); **StaffAudit** (D-35) = staff accountability ("kim qachon nima qildi") — cross-cutting tizim. Bitta jadvalga tiqilmaydi; moderation amali ikkalasida ham iz qoldiradi (har biri o'z maqsadida). ModerationAction destructive edit qilinmaydi (C-6).

## 14. Community restrictions

### CommunityRestriction (TD-70)

| Field | Izoh |
|---|---|
| id, user_id | |
| starts_at, expires_at | expires nullable emas — temporary restriction (muddatsiz blok — Admin suspension domeni) |
| reason | |
| created_by | Moderator (audit) |
| revoked_at, revoked_by | nullable — muddatidan oldin bekor qilish |
| created_at | |

- Active restriction = revoke qilinmagan va expires_at > now — user post/reply/reaction yarata olmaydi (o'qish ochiq — UX bosqichida aniqlanadi).
- **Account SUSPENDED bilan aralashmaydi** (C-12): SUSPENDED — butun platforma, Admin domeni (D-35 audit); restriction — faqat community.

**Block (§21, OPEN):** UserBlock hozir modellashtirilmaydi — product qarori OPEN; kelajakda oddiy junction (blocker, blocked) sifatida additive qo'shiladi, feed/reply filtrlashga ulanish oson.

## 15. Reputation

| Variant | Baho |
|---|---|
| A — har safar history'dan hisoblash | Har profil ko'rsatishda og'ir query; formula OPEN bo'lsa ham ko'rsatish kerak |
| **B — ReputationBalance + append-only ReputationEvent** | Auditable, formula o'zgarsa recompute, spam correction (manfiy event) mumkin ✅ |
| C — UserProfile'da oddiy int | Auditability yo'q, correction izsiz — rad |

**PROPOSED (TD-71): B** — 1.2B evidence+state patterni, IZL'dan yengil (locking/reversal apparati yo'q):

- **ReputationBalance:** user_id (1:1), total (int), updated_at.
- **ReputationEvent:** id, user_id, amount (±), reason_code (string registry — formula/manbalar OPEN: accepted answer, useful reaction, helpful contribution, moderation correction...), post_id/reply_id/reaction_id nullable FK (source), dedup_key **unique(user_id, dedup_key)** (masalan `accepted_answer:{reply_id}` — idempotent, accepted answer almashtirilganda manfiy event bilan korreksiya), created_at. Append-only.

Formula table schema'ga kodlanmaydi — engine/config.

## 16. XP/IZL boundary

**PROPOSED (TD-72) — qat'iy chegara:**

- **Reputation ≠ XP ≠ IZL** (D-40). Community reward: reputation, XP, title, badge.
- **Community domain XP balance'ni bevosita mutate qilmaydi:** `Community event → Gamification/XP service → XpGrant` (1.2C-1 TD-45 modeli orqali).
- **Oddiy community faoliyati (post/reply/reaction) IZL BERMAYDI — hech qachon** (C-8). Daily Mission "bugun o'rganganingni tushuntir" bo'lsa: **CommunityPost faqat evidence** — zanjir (TD-84, Phase 1.2D): `DailyPlanItem → DailyMissionCompletion → DailyMissionCompletionEvidence(community_post_id) → Reward Engine → RewardGrant` ([DATA_MODEL_LEARNING.md](DATA_MODEL_LEARNING.md)). **Community domain wallet/ledgerga hech qachon yozmaydi.**

## 17. Feed / counters

**PROPOSED (TD-73):**

- **Persisted Feed jadvali YO'Q** — feed = query/ranking view: subject/topic filtri + recency (+ visibility=VISIBLE). "For You" personalization — kelajak; event-sourced/social-graph feed — rad.
- **Cached counters:** post'da `reply_count`, `reaction_count` — feed sahifasi har post uchun COUNT qilmasligi uchun; yozuvlar tegishli insert/delete bilan bir tranzaksiyada. Bu financial emas — drift bo'lsa davriy reconcile yetarli. Per-type reaction breakdown — post detail'da COUNT (bitta post uchun arzon).

## 18. Public profile / privacy

Community'da author sifatida ko'rinadi: **display_name** (+ kelajak avatar/badges), **reputation**. 1.2A public/private ajratish (I-12) kuchda:

- Hech qachon public bo'lmaydi: phone, date_of_birth, private learning state (skill profile, progress), payment/reward balance.
- Bu **application/DTO qatlami invarianti** — community query'lari User'dan faqat public maydonlarni tortadi (C-2).

## 19. Minor safety

- Private DM entity **mavjud emas** (C-13); group/live chat yo'q.
- Report + moderation + visibility hide + CommunityRestriction — birinchi kundan modelda.
- Media moderation (§20) — rasm/audio uchun.
- Parental consent/legal fields **invent qilinmaydi** — legal review OPEN (D-32); kerak bo'lsa additive.

## 20. Media moderation

**PROPOSED (TD-74):** MediaAsset'da (1.2A ochiq qolgan status to'plamini aniqlashtirish) **ikki alohida o'lchov**:

| Field | Ma'no | Conceptual qiymat misollari |
|---|---|---|
| processing_status | **Upload/process readiness** | PENDING / READY / FAILED |
| moderation_status | **Safety/moderation decision** | UNREVIEWED / APPROVED / BLOCKED |

- Ikki o'lchovning ajratilishi — **ACCEPTED**; **exact state nomlari — OPEN/implementation detail** (jadvaldagi qiymatlar conceptual misollar, owner clarification).
- Bitta enumga aralashtirilmaydi — ikki mustaqil lifecycle.
- Public ko'rinish sharti: ready **va** blocked emas (C-9). Pre-moderation vs post-moderation policy — **PRODUCT OPEN**.
- **Automated moderation (§31):** natijalar MediaAsset'ning provider-neutral `moderation_metadata` JSONB'ida (skorlar, flaglar) + kerak bo'lsa CommunityReport/moderation signal hosil qiladi; **AI/provider hech qachon business authority bo'lmaydi** — final qaror human moderator'da (automated faqat taklif/flag); automated authority chegarasi — OPEN. Provider-specific maydonlar business modelga kirmaydi. AI moderation provider — tanlanmagan (OPEN).

## 21. Announcements

### Announcement (TD-75)

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| title, body | body — plain text (post bilan bir xil format falsafasi) |
| status | DRAFT / PUBLISHED / ARCHIVED |
| publish_at | nullable — scheduled publishing (event e'lonlari uchun); queue texnologiyasi OPEN — model tayyor, mexanizm keyin |
| expires_at | nullable — muddati o'tgach ko'rsatilmaydi |
| audience_type | enum: ALL (MVP'da yagona qiymat) — kelajak segmentatsiya (subject learners, tier, role) additive kengayadi; generic audience rules engine YO'Q |
| created_by | FK (Admin) — publish audit D-35 bilan |
| created_at, updated_at | |

Public ko'rinish sharti: PUBLISHED + (publish_at ≤ now yoki null) + (expires_at > now yoki null) (C-10).

## 22. Announcement lifecycle / audience

- Lifecycle: DRAFT → PUBLISHED → ARCHIVED (TD-28 container patterni; REVIEW bosqichi kerak emas — admin content).
- Scheduled publishing: `publish_at` bilan model-ready; ishga tushirish mexanizmi (queue/cron) — texnik bosqich.
- Audience MVP: barcha userlar; segmentatsiya — kelajak (audience_type + params kengayishi), hozir qurulmaydi.

## 23. Announcement user state

**PROPOSED (TD-76): lazy sparse junction** — har announcement uchun barcha userga oldindan row **yaratilmaydi**:

### AnnouncementUserState

| Field | Izoh |
|---|---|
| announcement_id + user_id | unique pair |
| read_at | nullable |
| dismissed_at | nullable |

Row faqat user birinchi marta ko'rgan/dismiss qilganda yaratiladi; "o'qilmagan" = row yo'q yoki read_at null. Retry → unique no-op.

## 24. Notifications

**PROPOSED (TD-77): in-app Notification core record hozir modellanadi** (minimal type set — PRODUCT OPEN, model tayyor turadi; aks holda community reply notification kabi aniq kandidatlar uchun keyin butun model kutib qoladi):

### Notification

| Field | Izoh |
|---|---|
| id, user_id | |
| type | string code (application registry — minimal set OPEN, enum emas) |
| title, body | **rendered snapshot** — original source yo'qolsa/archived bo'lsa ham notification historical UX record sifatida tushunarli qoladi |
| source_type / source_id | **lightweight polymorphic reference** (owner clarification) — ko'p typed nullable FK ustunlari bilan kengaytirilmaydi; notification financial/audit source of truth emas, FK strictness talab qilinmaydi |
| params | JSONB, nullable — qo'shimcha display parametrlari |
| dedup_key | **unique(user_id, dedup_key)** (§25) |
| read_at | nullable |
| created_at | |

## 25. Notification sources / dedup

- **Manbalar (potentsial):** announcement, community reply, accepted answer, subscription expiry, learning reminder, muhim AI recommendation. Qaysilari MVP'da — PRODUCT OPEN; event → notification yaratish **decoupled** (hodisa bo'lishi notification bo'lishini majburlamaydi — policy hal qiladi, §27).
- **Source reference (owner clarification, TD-77):** lightweight polymorphic `source_type + source_id` — typed nullable FK ustunlar to'plami bilan kengaytirilmaydi. Notification financial/audit source of truth emas; rendered snapshot tufayli source archived/o'chirilgan bo'lsa ham record tushunarli qoladi.
- **Dedup (TD-78):** deterministic `dedup_key` (masalan `reply:{reply_id}`, `announcement:{announcement_id}`) + unique(user, dedup_key) — retry/background job duplicate yarata olmaydi (C-11). Aggregation kelajagi ham shu kalit orqali (masalan kunlik yig'ma kalit) — extension.

## 26. Delivery abstraction

```
Notification (fact, in-app o'qiladi)
      ↓
NotificationDelivery (channel urinishi: in-app'dan tashqari push/SMS/email)
      ↓
Provider adapter
```

**PROPOSED (TD-78):** NotificationDelivery entity **hozir yaratilmaydi** — channels OPEN (in-app/push/SMS/email tanlanmagan, provider tanlanmagan). In-app MVP Notification jadvalining o'zidan o'qiladi. Channel tanlangach Delivery qatlami additive qo'shiladi — core model o'zgarmaydi. Registration SMS OTP — notification tizimi EMAS (AUTH_ARCHITECTURE, SmsModule).

## 27. Anti-spam

Accepted prinsip: "kam, lekin qimmatli". Model tayyorligi:

- Har hodisa notification emas — yaratishni policy boshqaradi (masalan reaction default notification bermaydi — product OPEN).
- dedup_key — takror/spam himoyasi + kelajak aggregation kaliti.
- Complex preference center YO'Q — kelajakda oddiy user notification preference (additive) qo'shilishi mumkin.

## 28. Audit boundaries

| Tizim | Vazifa |
|---|---|
| Community business data (posts/replies/reactions/reports) | Domain haqiqati |
| ModerationAction | Community domain moderation tarixi |
| StaffAudit (D-35) | Staff accountability |
| SecurityEvent (1.1) | Auth/security |
| Notification | User-facing communication fakti |

Universal `logs` jadvali YO'Q.

## 29. Idempotency / concurrency

**PROPOSED (TD-79):**

- **Unique constraintlar:** reaction (user, target, type); report (reporter, target); AnnouncementUserState (announcement, user); Notification (user, dedup_key); ReputationEvent (user, dedup_key) — retry/double-click → no-op.
- **Report resolve race (ikki moderator):** report row status transition guard — birinchi resolver yutadi (row lock/`WHERE status=OPEN` uslubi), ikkinchisiga no-op/xabar.
- **Accepted answer tez almashtirish:** post row yangilanishi — last-write-wins (bitta owner — real konflikt yo'q); reputation korreksiyasi event modelida.
- **Edit + moderation hide race:** MODERATOR_HIDDEN holatdagi content'ga edit application qatlamida bloklanadi; hide har doim yutadi.
- Full optimistic-lock framework YO'Q — yuqoridagilar yetarli.

## 30. Retention / deletion

**PROPOSED (TD-80):**

- Community content financial darajada abadiy emas, lekin **hard delete o'rniga visibility model** (§11) — bog'liq tarix (replies, reports, reputation) buzilmaydi.
- **Reports va ModerationAction — audit uchun saqlanadi** (destructive delete yo'q).
- User erasure/legal — OPEN: yo'nalish anonymization/pseudonymization (author reference anonimlashtiriladi, content/moderation tarixi strukturaviy saqlanadi yoki policy bo'yicha tozalanadi) — exact retention invent qilinmaydi.

## 31. Indexes / volume

**Indexes (obvious):** posts — (subject_id, created_at), (topic_id, created_at), (author_user_id, created_at), (visibility); replies — (post_id, created_at), (author_user_id); reactions — (post_id)/(reply_id), unique'lar; reports — (status, created_at), target FK'lar; ModerationAction — target FK'lar, (created_at); notifications — (user_id, read_at, created_at); announcements — (status, publish_at); AnnouncementUserState/ReputationEvent — unique'lar; CommunityRestriction — (user_id, expires_at).

**Volume:** MVP'da partitioning YO'Q; o'sish bo'lsa reaction/notification jadvallari uchun archive/partition — extension. Full-text search — accepted emas, qo'shilmaydi (feed filtrlar bilan); search engine ehtiyoji — kelajak OPEN.

## 32. Entity Catalog

| Entity | Purpose | Mutable/Append-only | Key relations | Phase |
|---|---|---|---|---|
| CommunityPost | Post (5 type, subject/topic, visibility) | Mutable (body/status; edit policy OPEN) | Author→User; Subject?/Topic?; accepted_reply→Reply | NOW |
| CommunityReply | Flat javob | Mutable (body/status) | N:1 Post; Author→User | NOW |
| ReactionType | Seeded reaction ro'yxati | Mutable (admin) | 1:N Reaction | NOW |
| CommunityReaction | User reaksiyasi | Create/delete (tarixsiz) | User; Post XOR Reply; Type | NOW |
| CommunityPostMedia | Post↔MediaAsset junction | Mutable (attach/detach draftda) | unique(Post, MediaAsset) | NOW |
| CommunityReport | Content report + snapshot | Status lifecycle | Reporter; Post XOR Reply; unique(reporter, target) | NOW |
| ModerationAction | Moderation domain tarixi | **Append-only** | Actor; target refs; Report? | NOW |
| CommunityRestriction | Temporary community cheklovi | Mutable (revoke) | User; created_by | NOW |
| ReputationBalance | Reputation cache | Mutable (derived) | 1:1 User | NOW |
| ReputationEvent | Reputation manbai | **Append-only**, dedup_key | User; Post/Reply/Reaction refs | NOW |
| Announcement | Admin e'loni | Lifecycle (D/P/A) | created_by; publish_at | NOW |
| AnnouncementUserState | Lazy read/dismiss holati | Mutable sparse | unique(Announcement, User) | NOW |
| Notification | In-app notification fakti (rendered snapshot) | read_at mutable | User; polymorphic source_type/source_id; unique(user, dedup_key) | NOW |
| UserBlock | User-to-user block | — | OPEN product qarori | LATER (OPEN) |
| NotificationDelivery | Channel delivery urinishi | — | channel tanlangach | LATER |
| ModerationCase | Report aggregation | — | hajm oshsa | LATER |

## 33. Relationship Map

```
User
 ├──1:N── CommunityPost (type, subject?/topic?, visibility)
 │            ├──1:N── CommunityReply (flat)
 │            ├── accepted_reply_id ──▶ CommunityReply (faqat QUESTION)
 │            ├──1:N── CommunityReaction ◀──N:1── ReactionType (seeded)
 │            ├──1:N── CommunityPostMedia ──▶ MediaAsset (image/audio; video ❌)
 │            └──1:N── CommunityReport (content_snapshot bilan)
 │                         └──0..N── ModerationAction (append-only) ──▶ StaffAudit (alohida tizim)
 ├──1:N── CommunityReply / CommunityReaction / CommunityReport (author/actor sifatida)
 ├──1:N── CommunityRestriction (temporary; ≠ account SUSPENDED)
 ├──1:1── ReputationBalance ◀──derived── ReputationEvent (append-only, dedup_key)
 ├──1:N── Notification (dedup_key; read_at)         [kelajak: ──1:N── NotificationDelivery]
 └──1:N── AnnouncementUserState ──N:1── Announcement (DRAFT/PUBLISHED/ARCHIVED; publish_at)

Chegaralar:
Community event ──▶ Gamification/XP service ──▶ XpGrant   (bevosita XP mutate yo'q)
CommunityPost = faqat evidence ──▶ Reward Engine (DailyPlanItem bilan)   (wallet/ledgerga yozish yo'q)
```

## 34. Invariants

1. **C-1:** Community media faqat image/audio — video attach strukturaviy rad etiladi (mime allowlist).
2. **C-2:** Post/reply author DTO'sida phone, date_of_birth, private learning/payment data hech qachon expose qilinmaydi.
3. **C-3:** Post topic'i bo'lsa subject ham bo'ladi va topic o'sha subject zanjiriga tegishli.
4. **C-4:** Accepted Answer faqat QUESTION postda va faqat shu postning reply'si; faqat post author belgilaydi; hidden reply accepted bo'lmaydi.
5. **C-5:** unique(user, target, reaction_type) — duplicate reaction yo'q; unique(reporter, target) — duplicate report yo'q.
6. **C-6:** ModerationAction append-only — destructive edit/delete qilinmaydi.
7. **C-7:** AUTHOR_REMOVED / MODERATOR_HIDDEN content feed/public query'larda ko'rinmaydi.
8. **C-8:** Community oddiy faoliyati (post/reply/reaction) IZL ledgerga hech qachon bevosita yozmaydi; mission rewardlari faqat Reward Engine orqali (evidence sifatida).
9. **C-9:** moderation BLOCKED media asset public serve qilinmaydi; public ko'rinish = READY + not BLOCKED.
10. **C-10:** Announcement PUBLISHED (va publish_at yetgan, expire bo'lmagan) bo'lmasa public ko'rinmaydi.
11. **C-11:** Notification retry duplicate yaratmaydi — unique(user, dedup_key).
12. **C-12:** CommunityRestriction ≠ account suspension — faqat community write'larini cheklaydi.
13. **C-13:** Private DM entity mavjud emas — schema darajasida yo'q.
14. **C-14:** Community domain XpBalance/IZLWallet'ni bevosita mutate qilmaydi — faqat tegishli service/engine orqali.
15. **C-15:** Report content_snapshot report paytidagi matnni saqlaydi — keyingi edit moderation kontekstini o'zgartira olmaydi.
16. **C-16:** ReputationEvent append-only, dedup_key unique — bir manba bir marta hisoblanadi; korreksiya faqat yangi (manfiy) event bilan.

## 35. Open questions

**Product:**

1. Post/reply **editing policy** — ruxsatmi, muddat/chegara bormi (model tayyor: overwrite + edited_at + report snapshot).
2. **Reaction ro'yxati** final (seeded jadval — data).
3. **Reputation formulasi** va manba qiymatlari (reason_code registry — data).
4. **Block capability** MVP'ga kiradimi (UserBlock — LATER, OPEN).
5. **Media limits** — image count/size, audio duration/size (config).
6. **Media pre-moderation vs post-moderation** — UNREVIEWED default ko'rinadimi.
7. **Moderation policy/guidelines** va punishment turlari (action_type ro'yxati kengayadi).
8. **Notification minimal MVP set** va qaysi hodisa notification beradi (masalan reaction — bermaydi deb taxmin, OPEN).
9. **Delivery channels** (in-app/push/SMS/email) va providerlar.
10. **Learning reminder** dizayni (notification type sifatida).
11. Question postda **title majburiyligi** (UX).

**Texnik:**

12. Scheduled announcement publishing mexanizmi (queue OPEN'ga bog'liq).
13. Community uchun search yondashuvi (full-text — accepted emas; kelajak ehtiyoj).
14. Counter reconcile jobi.
15. Anonymization'ning community content'ga aniq ta'siri (legal bilan).

## 36. Recommended model summary

1. **Post/Reply (TD-61):** alohida entitylar; flat replies (nesting yo'q — additive kengayadi); 5 post type — DB enum.
2. **Body (TD-62):** plain text + render escape; markdown-subset — kelajak rendering qatlami.
3. **Association (TD-63):** optional bitta Subject + optional Topic; hashtag/N:M yo'q.
4. **Accepted Answer (TD-64):** post'da accepted_reply_id + invariantlar; alohida entity yo'q.
5. **Reactions (TD-65):** seeded ReactionType + XOR-target CommunityReaction + unique(user, target, type); reaction semantikasi immutable — label o'zgaradi, retirement `is_active=false` bilan, history saqlanadi.
6. **Media (TD-66):** MediaAsset reuse, CommunityPostMedia junction, image/audio only.
7. **Removal (TD-67):** VISIBLE/AUTHOR_REMOVED/MODERATOR_HIDDEN; normal operationda hard delete yo'q; editing capability va policy — PRODUCT OPEN (model faqat future supportga tayyor); report content_snapshot.
8. **Reports/moderation (TD-68/69):** XOR-target report + snapshot + duplicate unique; append-only ModerationAction; ModerationCase keyin; StaffAudit alohida.
9. **Restriction (TD-70):** temporary CommunityRestriction ≠ SUSPENDED.
10. **Reputation (TD-71):** balance cache + append-only dedup'li ReputationEvent; formula — engine/config.
11. **Boundary (TD-72):** community → XP faqat Gamification service orqali; IZL — hech qachon bevosita, faqat mission evidence.
12. **Feed/counters (TD-73):** persisted feed yo'q — query view; cached reply/reaction counters + reconcile.
13. **Media moderation (TD-74):** processing_status (readiness) ≠ moderation_status (safety decision); exact state nomlari — implementation detail; automated natija — provider-neutral metadata; AI/provider business authority emas — final qaror human'da.
14. **Announcements (TD-75/76):** DRAFT/PUBLISHED/ARCHIVED + publish_at (scheduled-ready) + audience ALL (extension); lazy AnnouncementUserState.
15. **Notifications (TD-77/78):** in-app core record (rendered snapshot + lightweight polymorphic source_type/source_id + dedup_key unique); financial/audit source of truth emas; NotificationDelivery — channel tanlangach; anti-spam policy-driven.
16. **Idempotency/retention (TD-79/80):** unique constraintlar + transition guardlar; reports/moderation saqlanadi; erasure — anonymization (OPEN).
