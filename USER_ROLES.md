# Izlan — User Roles & Authorization

> Conceptual permission model — exact backend permission nomlari bu hujjatda YARATILMAYDI (texnik bosqich).
> Bog'liq qarorlar: D-30..D-35 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).

## Authentication konteksti

- Learning boshlash uchun **registration majburiy**.
- Primary identity: **telefon raqam**; verification: **SMS OTP**. Email majburiy emas.
- Onboardingda kamida `date_of_birth` olinadi (age fixed value emas, shu asosda hisoblanadi).
- **Phase 1.1da ACCEPTED** ([AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md), [TECH_DECISIONS.md](TECH_DECISIONS.md)): short-lived access token + rotating refresh token + server-side AuthSession modeli; permission-based authorization.
- **Hali OPEN:** SMS provider tanlovi; auth implementation detallari (JWT signing/key management, CSRF mexanizmining aniq shakli, access token joylashuvi, TTL/limit tuning); account recovery policy.

### Guest (rol emas — unauthenticated holat)

Guest ko'rishi mumkin: landing, product explanation, feature preview, limited demo experience.

Account talab qiladi: assessment, personalized roadmap, learning progress, rewards, subscription, community participation.

## MVP rollari — 4 ta

### LEARNER

Ishlaydigan sohalari: profile, assessment, roadmap, lessons, AI tutor, progress, XP, IZL, subscription, payment history, reward history, community, personal analytics.

Cheklov: boshqa userlarning private learning data'siga kira olmaydi.

### METHODIST (Content Creator)

- Faqat **o'ziga biriktirilgan Subject'lar** doirasida ishlaydi (masalan English ✓, Japanese ✓, Mathematics ✗).
- Yaratishi/tahrirlashi mumkin: tracks, levels, modules, topics, lessons, slides/content blocks, examples, exercises, assessments, audio, learning objectives.
- AI unga content creation'da yordam beradi; AI content avtomatik publish bo'lmaydi ([CONTENT_MODEL.md](CONTENT_MODEL.md)).
- Checkpoint joylashuvini pedagogik struktura asosida belgilaydi.
- FUTURE: subject'dan granular scope (English → Grammar, English → Speaking). MVP uchun subject-level yetarli.

### MODERATOR

Community uchun: reported posts, reported replies, spam, inappropriate content, moderation actions.

Cheklov: payment, reward balance, subscription, learning private data, content creation kabi funksiyalarga **avtomatik access olmaydi**.

### ADMIN

Platformani deyarli to'liq boshqaradi:

- statistics/dashboard
- users, methodists, moderators
- subject assignments
- learning content
- subscriptions, pricing, payments
- rewards
- announcements
- community
- reports, analytics
- AI usage/cost monitoring
- system configuration

## Sensitive action restrictions

Admin bo'lsa ham, sensitive financial actions **unrestricted edit emas**:

- ❌ `User IZL balance = input field` orqali to'g'ridan-to'g'ri overwrite — tavsiya etilmaydi.
- ✅ Adjustment ledger/audit entry orqali: **amount + reason + actor + timestamp**.

## Permission-based authorization prinsipi

Authorization role nomlariga qattiq bog'langan hard-coded model emas:

```
ROLE → PERMISSIONS
```

- Rol — permission to'plamining nomi; tekshiruvlar permission darajasida bo'ladi.
- Bu kelajakda granular scope'lar va yangi rollarni og'riqsiz qo'shishga imkon beradi.
- Relational model (Role/UserRole/RolePermission) — ACCEPTED (TD-26, [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md)); exact permission catalog hali final emas.

## Audit talabi

Sensitive staff actions audit qilinadi:

- IZL adjustment
- payment correction
- subscription change
- user suspension
- role/permission change
- content publish/unpublish
- moderation action
- creator assignment

Audit arxitekturasi — **ACCEPTED**: domain-level yozuvlar (IZL ledger ADJUSTMENT, ModerationAction, SubscriptionChange va h.k.) business truth; cross-cutting **StaffAudit** (TD-81) — append-only accountability log: actor + action_code + lightweight polymorphic target + reason, sensitive amal bilan bir tranzaksiyada ([DATA_MODEL_CORE.md](DATA_MODEL_CORE.md), [PRE_SCHEMA_REVIEW.md](PRE_SCHEMA_REVIEW.md) §7). SecurityEvent/Ledger/ModerationAction bilan birlashtirilmaydi.

## Minors bo'yicha eslatma

Auditoriyada voyaga yetmaganlar bor. Privacy, community safety, payment/reward restrictions va parental consent masalalari launchdan oldin alohida huquqiy/product review talab qiladi. Exact legal rules hozir invent qilinmaydi — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
