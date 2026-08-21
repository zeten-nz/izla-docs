# Izlan — Core Data Model Architecture (User + Content)

> Status: Phase 1.2A (2026-08-21; review 2026-08-22). Prisma-ready **architecture** — schema EMAS, migration EMAS.
> Qarorlar holati ([TECH_DECISIONS.md](TECH_DECISIONS.md)): **TD-21..TD-30 hammasi ACCEPTED** — Phase 1.2A technical decision review nuqtai nazaridan **COMPLETE**. Qolgan mayda bandlar — implementation detail (§30).
> Bog'liq: [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md), [CONTENT_MODEL.md](CONTENT_MODEL.md), [USER_ROLES.md](USER_ROLES.md).

## 1. Goals

1. User/Profile/Authorization va Content domainlari uchun keyin Prisma schema'ga deyarli to'g'ridan-to'g'ri o'giriladigan darajada aniq model.
2. Phase 1.1 auth entitylari (User, AuthSession, RefreshToken, OtpChallenge, SecurityEvent) bilan zid kelmaslik.
3. Phase 1.2B (assessment/roadmap/progress) va 1.2C uchun extension pointlarni ochiq qoldirish.
4. Overengineering'siz: generic CMS tree emas, enterprise identity platform emas.

## 2. Scope / non-scope

**Scope (NOW):** User, UserProfile, Role/Permission modeli, Methodist subject assignment, Subject→...→Activity hierarchy, Skill, lifecycle, versioning, media, IDs/slugs, constraints/integrity.

**Non-scope (bu bosqichda entity darajasida modellashtirilmaydi):** AssessmentAttempt, AssessmentAnswer, LearnerSkillProfile, Roadmap/RoadmapItem, LearningSession, ActivityAttempt, Mistake, Progress, XP transaction, IZL ledger, Subscription/Payment, Community, Notification. Ular faqat future FK/extension sifatida zikr qilinadi.

## 3. Design principles

1. **Explicit relational chain** — har content darajasi alohida jadval, aniq FK; generic parent/child tree yo'q.
2. **Bitta jadval — bitta mas'uliyat:** identity (User) ≠ personal data (UserProfile) ≠ authorization (Role/Permission).
3. **Learner history hech qachon singan referencega qolmasin** — delete o'rniga ARCHIVE, published content immutable.
4. **Enum faqat kod bilan birga o'zgaradigan yopiq to'plamlar uchun**; content sifatida o'zgaradigan narsalar (Level, Skill) — data.
5. **Optional field ≠ invented requirement:** product tasdiqlamagan maydonlar nullable va optional (masalan `preferred_language`).
6. AI content **hech qachon** human review'siz PUBLISHED bo'lmaydi — model shu invariantni qo'llab-quvvatlaydi.

## 4. User/Profile model

### User (identity + account core)

| Field | Turi (conceptual) | Izoh |
|---|---|---|
| id | UUID (v7) | PK |
| phone | string, **unique** | Canonical E.164 `+998...` (TD-16) |
| status | enum `UserStatus`: ACTIVE / SUSPENDED / DEACTIVATED | TD-15 |
| last_login_at | timestamp, nullable | Recommended (security/analytics) |
| created_at / updated_at | timestamp | Standard |

### UserProfile (1:1, alohida jadval)

| Field | Turi | Izoh |
|---|---|---|
| user_id | PK + FK → User | 1:1 |
| display_name | string, nullable | Onboardinggacha bo'sh; public identity |
| date_of_birth | date, nullable | Onboardinggacha bo'sh; age shu yerdan hisoblanadi |
| onboarding_completed_at | timestamp, nullable | Null = onboarding INCOMPLETE (account state emas — TD-15) |
| preferred_language | string, nullable, **optional** | UI tili product'da OPEN — required qilinmaydi |
| timezone | string, nullable, optional | Schedule/notification uchun keyin foydali |
| created_at / updated_at | timestamp | |

**Nega alohida jadval:** (1) User — auth/authorization/ledger'lar reference qiladigan "hot" identity jadvali, lean qolishi kerak; (2) profil vaqt o'tishi bilan kengayadi (avatar, community handle...) — migration churn identity jadvaliga tegmaydi; (3) UsersModule ichida auth-identity va personal-data chegarasi aniq bo'ladi.

**Public vs private identity:** public — `display_name` (kelajakda avatar/handle profilga qo'shiladi); private — `phone`, `date_of_birth`. Community/public kontekstda faqat public maydonlar chiqadi (invariant I-12). Alohida "CommunityIdentity" jadvali hozir kerak emas — premature.

**Soft delete:** yo'q. Account state (DEACTIVATED/SUSPENDED) yetarli; `deleted_at` qo'shilmaydi. Privacy bo'yicha o'chirish/anonimlashtirish policy'si — OPEN (§30).

**Learning preferences/schedule:** bu jadvallarga qo'shilmaydi — Phase 1.2B'da alohida entity (extension point: `user_id` FK).

## 5. Authorization model

```
User ─N:M─ Role ─N:M─ Permission(code)
      (UserRole)   (RolePermission)
```

**Tahlillar va tanlov:**

- **Bitta rol vs ko'p rol:** ko'p rol (N:M). Real hayotda Methodist ayni paytda Moderator bo'lishi mumkin; hamma userga registratsiyada default LEARNER roli beriladi, staff rollari qo'shimcha. Single-role enum bu holatlarda darhol sinadi.
- **Role: enum vs table → table.** Rollar seed qilinadi (LEARNER, METHODIST, MODERATOR, ADMIN, `code` unique), lekin jadval bo'lishi: (1) RolePermission mapping DB'da yashashiga, (2) kelajakda yangi rol migration'siz qo'shilishiga imkon beradi.
- **Permission: table vs code registry → code registry + DB mapping.** Valid permission kodlari **application code'dagi typed registry**da yashaydi (guards baribir kodga bog'liq — alohida Permission jadvali kod bilan drift qiladi). DB'da faqat `RolePermission(role_id, permission_code)` mapping — Admin deploy'siz mappingni sozlay oladi; kod registry validatsiya manbai.

### Entitylar

- **Role:** id, code (unique), name, created_at. Seeded.
- **UserRole:** user_id, role_id, granted_at, granted_by (nullable — staff bergan bo'lsa audit reference), **unique(user_id, role_id)**.
- **RolePermission:** role_id, permission_code (string, kod registry'ga qarshi validatsiya), **unique(role_id, permission_code)**.

Permission misollari (katalog EMAS, faqat namuna): `content.lesson.publish`, `community.post.moderate`, `billing.price.manage`. Exact katalog — keyingi bosqich.

## 6. Methodist subject assignment

**SubjectAssignment** (junction, N:M):

| Field | Izoh |
|---|---|
| user_id | FK → User (METHODIST rolidagi user) |
| subject_id | FK → Subject |
| assigned_at | timestamp |
| assigned_by | FK → User (Admin) — D-35 audit talabi |
| unique(user_id, subject_id) | |

- Authorization guard: content mutation'da `SubjectAssignment` tekshiriladi (invariant I-6).
- **Future granular scope** (English→Grammar): jadvalga keyin nullable scope maydoni (masalan `skill_id`) yoki alohida scope jadvali qo'shish orqali evolyutsiya qiladi — **hozir qo'shilmaydi**.

## 7. Content hierarchy

Har daraja alohida jadval, umumiy pattern: `id (UUIDv7) + parent FK + title + description? + sort_order + status + created_at/updated_at`.

| Entity | Parent FK | Qo'shimcha | Izoh |
|---|---|---|---|
| **Subject** | — | slug (global unique), created_by | Content container + Skill owner + assignment scope |
| **Track** | subject_id | slug (subject ichida unique), created_by | Learning goal (General English, IELTS...) |
| **Level** | track_id | **code** (display: "B1"; track ichida unique), sort_order (track ichida unique) | Enum EMAS — §12 |
| **Module** | level_id | — | Checkpoint maydoni YO'Q — §13 |
| **Topic** | module_id | — | |
| **Lesson** | topic_id | §9'da batafsil | Learner'ning asosiy unit'i |
| **Activity** | lesson_revision_id | §10–11'da batafsil | Content shu yerda |

**Subject maydonlari — essential vs recommended (§10 talabi):** essential: id, slug, title, status; recommended: description, sort_order (bosh sahifa tartibi), created_by; invented emas: icon/color kabi UI maydonlari hozir qo'shilmaydi.

**Sort/order:** har darajada `sort_order` (parent ichidagi tartib) — default ko'rsatish tartibi. **Active/lifecycle:** §17.

## 8. Hierarchy flexibility decision

| Variant | Baho |
|---|---|
| **A — har node mandatory** | Aniq FK zanjiri, oddiy querylar, roadmap engine uchun predictable. Kamchilik: "Track kerak bo'lmagan" fan ham Track yaratadi. |
| B — optional layerlar (nullable parentlar) | Har query'da branch'lar, Prisma relationlarda murakkablik, engine'da maxsus holatlar — xavfli o'rta yo'l. |
| C — generic content tree | Maksimal moslashuvchan, lekin FK integrity, type safety va aniq semantika yo'qoladi — **generic CMS overengineering, rad etiladi**. |

**ACCEPTED (TD-27): Variant A — hierarchy database/domain modelda explicit va to'liq qoladi; generic CMS tree ishlatilmaydi.**

Owner clarification: agar kelajakdagi Subject uchun Track yoki Level semantik jihatdan kerak bo'lmasa, **system/default structural node** ishlatilishi mumkin; bunday node UI'da Learner'dan yashirilishi mumkin. Default node product hierarchy'ni buzmaydi — faqat structural compatibility uchun. Level semantikasi fanga qarab data orqali o'zgaradi (§12) — schema o'zgarmaydi.

## 9. Lesson model

**Lesson** ikki qismga ajraladi: **Lesson (logical)** — barqaror identity (roadmap, skills, prerequisites shunga bog'lanadi) va **LessonRevision** — content versiyasi (§18 versioning qarori shu yerga olib keladi).

### Lesson (logical)

| Field | Izoh |
|---|---|
| id | UUIDv7 — learner history/roadmap uchun barqaror reference |
| topic_id | FK |
| slug | nullable (URL kerak bo'lsa; topic ichida unique) |
| sort_order | Topic ichidagi default tartib |
| status | DRAFT / PUBLISHED / ARCHIVED (§17 — REVIEW faqat LessonRevision'da) |
| published_revision_id | nullable FK → LessonRevision — learnerlar ko'radigan versiya |
| created_by | FK → User (Methodist) |
| created_at / updated_at | |

### LessonRevision

| Field | Izoh |
|---|---|
| id, lesson_id | FK |
| version | int, lesson ichida increment (unique(lesson_id, version)) |
| title, description? | Content maydonlari revision'da (tahrirlash uchun) |
| estimated_duration_min | nullable; activity durationlardan hisoblangan cache (builder yangilaydi) |
| status | DRAFT / REVIEW / PUBLISHED / ARCHIVED — revision workflow |
| created_by, updated_by | Authorship (§19) |
| reviewed_by?, published_by?, published_at? | nullable — review/publish audit referencelari |
| created_at / updated_at | |

**Lesson type maydoni:** hozir kerak emas — lesson xarakteri activitylardan kelib chiqadi; ehtiyoj tug'ilsa keyin qo'shiladi (premature field emas).

## 10. Activity/block storage analysis

| Variant | Baho |
|---|---|
| **A — bitta Activity jadvali + `type` + JSONB payload** | Prisma bilan sodda (bitta model, `Json` field); lesson builder bitta ordered ro'yxat o'qiydi; yangi block type = enum qiymati + payload schema (migration minimal); kamchilik — DB payload strukturasini enforce qilmaydi → application validation shart; payload ichiga relational query cheklangan (JSONB operatorlar bilan mumkin, lekin asosiy analytics 1.2B attempt jadvallarida bo'ladi). |
| B — base Activity + subtype jadvallar | DB-level type safety, lekin Prisma'da polymorphic relation yo'q — har o'qishda N ta join/lookup, builder'da union assembling, har yangi type = yangi jadval + migration. 12+ type uchun og'ir. |
| C — har type mustaqil jadval | Ordering/ro'yxat bitta lessonda 12 jadvaldan yig'iladi — builder va querying uchun eng yomon. |
| D — hybrid (A + cross-cutting maydonlar column sifatida) | Aslida to'g'ri qilingan A: query qilinadigan umumiy maydonlar (type, position, duration) column, content detali payload'da. |

**ACCEPTED (TD-21): Option A/D — bitta Activity jadvali, cross-cutting maydonlar column, content payload JSONB.** Payload validation qoidalari (§12) ham ACCEPTED (TD-22).

## 11. Final Activity model recommendation

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| lesson_revision_id | FK → LessonRevision (activity revisionga tegishli — publish immutability uchun) |
| type | enum `ActivityType`: TEXT, EXPLANATION, IMAGE, AUDIO, EXAMPLE, MINI_QUESTION, PRACTICE, SPEAKING, WRITING, LISTENING, AI_INTERACTION, MASTERY_TEST (VIDEO — future qiymat sifatida keyin qo'shiladi) |
| position | int — revision ichidagi tartib (§13) |
| estimated_duration_min | nullable int |
| payload | JSONB — type'ga bog'liq content (matn, savol/variantlar; media — TD-82: asset identity ActivityMedia junctionda, payload'da faqat role/config) |
| source | enum: HUMAN / AI_GENERATED / AI_ASSISTED (§20) |
| ai_metadata | JSONB, nullable (§20) |
| created_at / updated_at | |

Activity'da alohida lifecycle status YO'Q — revision statusiga ergashadi (§17). 1.2B'da `ActivityAttempt` shu `id`ga FK qiladi (extension point).

## 12. Activity validation

DB JSONB strukturani enforce qilmaydi → **application-level qat'iy validation**:

- Har `ActivityType` uchun payload schema — **discriminated union**: `type` discriminator, payload shu type'ning schema'siga to'liq mos bo'lishi shart (NestJS DTO/pipe qatlamida; exact library — schema-validation vositasi — implementation'da tanlanadi, hozir tanlanmaydi).
- Ikki tekshiruv nuqtasi: (1) create/update'da — asosiy validatsiya; (2) **publish transition'da qayta, strict** — PUBLISHED revision faqat valid payloadlardan iborat (invariant I-4). Draft'da qisman to'ldirilgan payloadga yumshoqroq qarash mumkin, publish'da emas.
- Payload schemalari versiyalanadi degan concept (schema o'zgarsa eski payloadlarni migratsiya qilish yo'li) — implementation bosqichida detallashtiriladi.

## 13. Ordering (Activity)

| Variant | Baho |
|---|---|
| **Oddiy int position (1,2,3...), reorder'da transaction ichida renumber** | Lesson ichida bloklar soni kichik (o'nlab) — renumber arzon; eng sodda va ishonchli. ✅ |
| Gaps (100, 200...) | Insert osonlashadi, lekin baribir vaqti-vaqti bilan rebalance kerak — kichik ro'yxat uchun keraksiz murakkablik |
| Fractional rank | Precision/rebalance muammolari — kerak emas |
| Linked-list | O'qish/tartiblash query'lari og'ir — rad |

**ACCEPTED (TD-29):** oddiy int `position`, reorder butun ro'yxatni bitta transactionda qayta yozadi. Concurrent editing core requirement emas (keyin optimistic lock qo'shsa bo'ladi). Xuddi shu yondashuv hierarchy `sort_order`lari uchun ham.

## 14. Prerequisites (Lesson)

- `Lesson.sort_order` — **taqdimot tartibi** (topic ichida), gating semantikasi emas.
- Gating uchun **explicit `LessonPrerequisite`** junction: `lesson_id`, `prerequisite_lesson_id`, unique(pair), self-reference taqiqlangan.

**Nega faqat linear order yetmaydi:** personalized Roadmap (1.2B) lessonlarni kesib tanlaydi — "roadmapda oldingi lesson" har learnerda boshqacha; gating haqiqati content-level bog'liqlik bo'lishi kerak. **Cycle prevention (TD-29):** application/service layerda, saqlash transactioni doirasida tekshiriladi; DB buni enforce qilmaydi. **MVP soddaligi:** Methodist tooling default sifatida "oldingi lesson prerequisite" ni avto-taklif qiladi — jadval murakkabligi Methodist'ga ko'rinmaydi. Cross-topic prerequisite'lar ruxsat etiladi (bir Subject ichida — invariantga kiritilgan emas, lekin tavsiya: bir Subject doirasida).

## 15. Skill model

**Skill:**

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| subject_id | FK — Skill har doim bitta Subject'ga tegishli (Subject 1:N Skill) |
| name | subject ichida unique |
| code | nullable, subject ichida unique — recommended (analytics/AI uchun barqaror identifikator, rename'dan himoya) |
| description? | nullable |
| sort_order | |
| status | ACTIVE / ARCHIVED |
| created_at / updated_at | |

Skill full content lifecycle talab qilmaydi (DRAFT/REVIEW ma'nosiz) — ACTIVE/ARCHIVED yetarli.

## 16. Lesson/Activity skill mapping

**LessonSkill** junction: lesson_id (logical Lesson — revision emas), skill_id, unique(pair).

- **`weight`/`primary`/`impact`:** hozir **premature** — ularni iste'mol qiladigan engine (skill profile hisoblash) 1.2B'da; qaysi ko'rinish kerakligi o'shanda aniq bo'ladi. Junction jadvalga keyin column qo'shish arzon.
- **Activity↔Skill (A/B/C tahlili):** A (faqat Lesson↔Skill) — MVP progress/roadmap lesson granularity'da ishlaydi; B (faqat Activity) — roadmap uchun baribir lesson-level kerak; C (ikkalasi) — to'g'ri yakuniy holat, lekin Activity↔Skill'ning iste'molchisi (mistake→skill attribution, adaptive analytics) 1.2B attempt-tracking bilan birga keladi. **Model qarori (1.2A): MVP'da faqat LessonSkill (NOW); `ActivitySkill` junction 1.2B'da attempt modeli bilan birga qo'shiladi (NEXT).** Model bunga tayyor (activity id'lar barqaror).

## 17. Content lifecycle

**ACCEPTED (TD-28, final wording):**

| Entity | Status to'plami | Sabab |
|---|---|---|
| **LessonRevision** | `DRAFT → REVIEW → PUBLISHED → ARCHIVED` | Review workflow'ning haqiqiy obyekti — Methodist/AI content shu yerda |
| **Lesson (logical)** | `DRAFT / PUBLISHED / ARCHIVED` — REVIEW **yo'q** (PUBLISHED ⟺ published_revision_id mavjud) | Roadmap engine "PUBLISHED lesson" filtri uchun; workflow revisionda |
| Subject, Track, Level, Module, Topic | `DRAFT → PUBLISHED → ARCHIVED` (REVIEW'siz) | Strukturaviy containerlar — review workflow ma'nosiz, lekin ko'rinish nazorati kerak (qurilayotgan Track learnerga chiqmasin) |
| **Activity** | Alohida lifecycle status yo'q — parent LessonRevision lifecycle'iga ergashadi | Har blokka alohida workflow — overengineering |
| Skill | `ACTIVE → ARCHIVED` | §15 |

**Muhim (owner clarification):** REVIEW state logical Lesson'da emas, LessonRevision'da — shu tufayli hozirgi PUBLISHED revision learnerlarga xizmat qilayotgan paytda yangi revision REVIEW holatida bo'lishi mumkin. Bitta umumiy `ContentStatus` enum ishlatilishi mumkin (REVIEW faqat LessonRevision darajasida ishlatiladi). Learner ko'radigan content sharti: butun parent zanjiri PUBLISHED + lesson PUBLISHED revision'ga ega. **AI content:** `source = AI_*` bo'lgan activity'li revision human review bosqichisiz PUBLISHED bo'la olmaydi (invariant I-5).

## 18. Versioning strategy

| Variant | Baho |
|---|---|
| A — in-place edit | Learner history published contentga bog'langach sinadi (attempt qaysi savolga edi — noma'lum bo'lib qoladi); rad |
| B — full immutable versions (har edit yangi version) | Har saqlashda snapshot — og'ir, draft workflow yo'q |
| **C — draft revision + published snapshot (revision model)** | Methodist draft revision ustida ishlaydi; publish → shu revision PUBLISHED bo'ladi, `Lesson.published_revision_id` unga ko'chadi; eski published revision **immutable ARCHIVED** bo'lib qoladi (activitylari bilan) — learner history eski revision id'lariga bog'lanib qolaveradi. ✅ |

**ACCEPTED (TD-24): C.** Oqim: PUBLISHED lessonni tahrirlash → published revisiondan yangi DRAFT revision copy-on-edit → REVIEW → publish (transactional pointer swap). Bitta lessonda bir vaqtda ko'pi bilan bitta PUBLISHED va bitta DRAFT/REVIEW revision (invariant I-2, I-3). Versioning faqat Lesson+Activity darajasida — containerlar versiyalanmaydi (title tahriri history uchun kritik emas). 1.2B'da attempt'lar `lesson_revision_id`/`activity_id`ga bog'lanadi — barqaror.

## 19. Authorship / audit

- **Authorship (column sifatida, faqat kerakli joyda):** `created_by` — barcha content jadvallarida; `updated_by` — tahrirlanadigan content'da (LessonRevision); `reviewed_by`/`published_by` + `published_at` — faqat LessonRevision'da (workflow nuqtalari).
- **Audit trail (kim-qachon-nima tarixi) column emas** — publish/unpublish, assignment o'zgarishlari kabi harakatlar D-35 bo'yicha alohida audit tizimiga event sifatida yoziladi. Har jadvalga to'liq audit column to'plamini tiqish — rad.
- Methodist scope nazorati storage'da emas, authorization qatlamida (SubjectAssignment guard, I-6).

## 20. AI provenance

**Minimal model (1.2A qabul qilingan model qismi):** Activity darajasida:

- `source`: HUMAN / AI_GENERATED / AI_ASSISTED — review filtri ("AI content ko'rilishi shart") va sifat tahlili uchun yetarli;
- `ai_metadata` JSONB, nullable: provider/model identifikatori, generation vaqti kabi qisqa provenance. Provider/model hali OPEN — shuning uchun structured columnlar emas, JSONB.

**Full prompt saqlash — default emas:** hajm katta, content/PII bo'lishi mumkin; debugging uchun kerak bo'lsa alohida (retention'li) joyda — bu AI observability masalasi, content jadvalining ishi emas. LessonRevision darajasida qo'shimcha AI flag kerak emas — activitylardan derive qilinadi.

## 21. Media model

**ACCEPTED (TD-25): alohida `MediaAsset` entity** (payload ichida faqat URL emas):

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| storage_key | unique — provider-neutral object key (URL EMAS; URL runtime'da provider config bilan quriladi) |
| mime_type, size_bytes | |
| duration_seconds | nullable (audio uchun) |
| width / height | nullable (image uchun) |
| uploaded_by | FK → User |
| status | UPLOADED / READY / BLOCKED — minimal moderation/processing holati (exact to'plam OPEN) |
| created_at | |

- **TD-82 (Phase 1.2D, ACCEPTED):** asset association'ning relational truth'i — **ActivityMedia junction** (activity_id, media_asset_id, role_code?, position; unique(parent, asset, role_code?)); payload'da **raw MediaAsset UUID saqlanmaydi** — faqat rendering/config (caption, displayMode, role/slot semantikasi). Provider migration'da payloadlar o'zgarmaydi (asset identity junctionda).
- Nega entity: ownership, moderation, size/cost accounting, FK RESTRICT bilan orphan himoyasi va **Community keyin xuddi shu infratuzilmadan foydalanishi**.
- Boshqa domain bog'lari: AssessmentItemMedia, CommunityPostMedia, response_media_asset_id — tegishli domain hujjatlarida.
- MediaAsset referenced bo'lsa hard delete — **real FK RESTRICT** (I-15).

## 22. IDs / slugs

**ID strategy (ACCEPTED, TD-23): UUIDv7, native `uuid` column.** Generatsiya joyi (PostgreSQL'dami yoki Prisma/application tomonidami) — implementation detail, architecture qarori emas.

| Variant | Baho |
|---|---|
| UUIDv4 | Global unique, lekin random — index locality yomon |
| **UUIDv7** | Time-ordered (insert/index friendly) + global unique + public expose xavfsiz (enumeration yo'q) ✅ |
| CUID | O'xshash foyda, lekin string storage; Postgres native uuid'dan foyda yo'qoladi |
| bigint auto-increment | Eng tez/kichik, lekin enumerable (public API'da leak) va future distribution'ga noqulay |

**Slugs:** DB PK — ichki/API identity; slug — URL/SEO uchun public identifikator. Scope'lar: Subject.slug — global unique; Track.slug — subject ichida; Level.code — track ichida (slug rolini bajaradi); Lesson.slug — nullable, topic ichida unique. Module/Topic uchun MVP'da id-based routing yetarli — slug keyin qo'shilishi mumkin (OPEN, §30). **Slug change:** texnik ruxsat etiladi, published contentda discourage qilinadi (SEO/link breakage); old-slug redirect — FUTURE.

## 23. Timestamps / deletion

- `created_at` + `updated_at` — barcha jadvallar (append-only'larda faqat `created_at`: SecurityEvent, UserRole kabi).
- **Global `deleted_at` YO'Q.** Content'da o'chirish semantikasi = **ARCHIVED status** (kerak bo'lsa `archived_at` bilan); User'da — account states. `deleted_at`ning hamma joyda bo'lishi ikkita parallel lifecycle yaratadi — rad.
- Haqiqiy hard delete faqat: ephemeral auth data (muddati o'tgan OtpChallenge cleanup), hech qayerdan reference qilinmagan DRAFT content, orphan MediaAsset.
- Privacy erasure (userni o'chirish talabi) → anonimlashtirish strategiyasi — OPEN (§30).

## 24. Constraints / indexes

**Unique constraints (asosiylari):**

- User.phone; UserProfile.user_id
- Role.code; UserRole(user_id, role_id); RolePermission(role_id, permission_code)
- SubjectAssignment(user_id, subject_id)
- Subject.slug; Track(subject_id, slug); Level(track_id, code); Level(track_id, sort_order)
- Lesson(topic_id, slug) — slug null bo'lmaganda; LessonRevision(lesson_id, version)
- Skill(subject_id, name); Skill(subject_id, code) — code null bo'lmaganda
- LessonSkill(lesson_id, skill_id); LessonPrerequisite(lesson_id, prerequisite_lesson_id)
- MediaAsset.storage_key

Ortiqcha global unique'lar (masalan title'lar bo'yicha) — qo'yilmaydi (business flexibility).

**Indexes (obvious relational, premature tuning'siz):**

- Barcha FK columnlar;
- `(parent_id, sort_order)` — har hierarchy darajasida (tartiblangan ro'yxat query'si);
- `(lesson_revision_id, position)` — Activity;
- status filtrlari uchun `(topic_id, status)` kabi kombinatsiyalar — kerak bo'lganda;
- slug lookup'lar (unique constraintlar o'zi index beradi);
- SubjectAssignment(user_id) — Methodist scope lookup.

## 25. Referential integrity / deletion policy

| Holat | Policy |
|---|---|
| Container (Subject...Topic) o'chirish | **RESTRICT** — bolalari borligida delete yo'q; operatsion yo'l — ARCHIVE |
| Lesson/LessonRevision o'chirish | PUBLISHED bo'lgan yoki bo'lgan revision — **hech qachon delete emas, ARCHIVE** (1.2B learner history FK'lari RESTRICT bilan keladi); faqat hech qachon publish bo'lmagan DRAFT revision hard delete qilinishi mumkin (activitylari **CASCADE**) |
| Activity | Draft revision ichida erkin delete; published revision ichida — immutable (I-3) |
| MediaAsset | Referenced bo'lsa **FK RESTRICT** (TD-82 junctionlari orqali — endi real DB constraint); reference'siz bo'lsa delete OK |
| User o'chirish | Hard delete yo'q — states + kelajak anonimlashtirish (OPEN) |
| UserRole/SubjectAssignment | Oddiy delete OK (grant/revoke) — tarix audit tizimida |
| OtpChallenge / eski RefreshToken | TTL cleanup — hard delete normal |

## 26. Entity Catalog

| Entity | Domain | Purpose | Key relationships | Lifecycle | Phase |
|---|---|---|---|---|---|
| User | Identity | Account + identity core | 1:1 Profile; 1:N AuthSession; N:M Role | UserStatus | NOW |
| UserProfile | Identity | Personal data, onboarding | 1:1 User | — | NOW |
| AuthSession | Auth (1.1) | Server-side session | N:1 User; 1:N RefreshToken | revoked/expiry | NOW |
| RefreshToken | Auth (1.1) | Rotation zanjiri | N:1 AuthSession | used/revoked | NOW |
| OtpChallenge | Auth (1.1) | OTP lifecycle | phone bo'yicha (FK'siz — pre-account) | consumed/expired | NOW |
| SecurityEvent | Auth (1.1) | Security logging | N:1 User (nullable) | append-only | NOW |
| Role | Authorization | Rol registry (seeded) | N:M User; 1:N RolePermission | — | NOW |
| UserRole | Authorization | User↔Role junction | — | — | NOW |
| RolePermission | Authorization | Role↔permission_code mapping | — | — | NOW |
| SubjectAssignment | Authorization | Methodist↔Subject scope | User N:M Subject | — | NOW |
| Subject | Content | Fan; container/skill owner/scope | 1:N Track, Skill | D/P/A | NOW |
| Track | Content | Learning goal yo'nalishi | N:1 Subject; 1:N Level | D/P/A | NOW |
| Level | Content | Daraja (data, enum emas) | N:1 Track; 1:N Module | D/P/A | NOW |
| Module | Content | Bo'lim; checkpoint scope (1.2B) | N:1 Level; 1:N Topic | D/P/A | NOW |
| Topic | Content | Mavzu | N:1 Module; 1:N Lesson | D/P/A | NOW |
| Lesson | Content | Logical learning unit | N:1 Topic; 1:N Revision; N:M Skill; N:M Lesson (prereq) | D/P/A (REVIEW'siz) | NOW |
| LessonRevision | Content | Content versiyasi + workflow | N:1 Lesson; 1:N Activity | D/R/P/A (to'liq) | NOW |
| Activity | Content | Content block (type + JSONB) | N:1 LessonRevision; payload→MediaAsset | Revision'ga ergashadi | NOW |
| Skill | Content | Subject skill'i | N:1 Subject; N:M Lesson | ACTIVE/ARCHIVED | NOW |
| LessonSkill | Content | Lesson↔Skill junction | — | — | NOW |
| LessonPrerequisite | Content | Gating DAG | Lesson↔Lesson | — | NOW |
| MediaAsset | Media | Provider-neutral media reference | Junctionlardan FK (TD-82); uploaded_by→User | status | NOW |
| ActivityMedia | Media | Activity↔MediaAsset relational truth (TD-82) | unique(Activity, Asset, role?); position | — | NOW |
| StaffAudit | Audit | Staff accountability append-only log (TD-81) | actor→User; target_type/id (loose polymorphic) | Append-only | NOW |
| ActivitySkill | Content | Activity-level skill mapping | Activity↔Skill | — | NEXT (1.2B) |
| Checkpoint | Assessment | Module checkpoint config | →Module | — | NEXT (1.2B) |
| AssessmentAttempt, ActivityAttempt, LearnerSkillProfile, Roadmap(+Item), LearningSession, Mistake, Progress, XPTransaction, IZLLedger | Learning/Rewards | 1.2B domainlari | →User, Lesson/Revision, Activity, Skill | — | NEXT |
| Subscription, Payment | Billing | 1.2B/1.2C | →User | — | NEXT/LATER |
| CommunityPost/Reply, Notification | Community/Comms | 1.2C+ | →User, Subject/Topic, MediaAsset | — | LATER |

## 27. Relationship Map

```
User ──1:1── UserProfile
 │
 ├──1:N── AuthSession ──1:N── RefreshToken
 ├──1:N── SecurityEvent (nullable user)
 ├──N:M── Role ──(UserRole)          Role ──1:N── RolePermission (permission_code)
 └──N:M── Subject ──(SubjectAssignment: Methodist scope)

Subject
 ├──1:N── Skill
 └──1:N── Track
            └──1:N── Level
                       └──1:N── Module          ←── (1.2B: Checkpoint)
                                  └──1:N── Topic
                                             └──1:N── Lesson
                                                        ├──N:M── Skill (LessonSkill)
                                                        ├──N:M── Lesson (LessonPrerequisite, DAG)
                                                        └──1:N── LessonRevision
                                                                   │  (Lesson.published_revision_id → bittasi)
                                                                   └──1:N── Activity
                                                                              ├── payload JSONB ──ref──▶ MediaAsset
                                                                              └── (1.2B: ActivitySkill, ActivityAttempt)

MediaAsset ──N:1── User (uploaded_by)
OtpChallenge — mustaqil (phone bo'yicha, account'dan oldin ham mavjud)
```

## 28. Cardinalities

| Relationship | Cardinality |
|---|---|
| User — UserProfile | 1:1 |
| User — AuthSession | 1:N |
| AuthSession — RefreshToken | 1:N |
| User — Role | N:M (UserRole) |
| Role — permission_code | 1:N (RolePermission) |
| User(Methodist) — Subject | N:M (SubjectAssignment) |
| Subject — Track / Skill | 1:N / 1:N |
| Track — Level | 1:N |
| Level — Module | 1:N |
| Module — Topic | 1:N |
| Topic — Lesson | 1:N |
| Lesson — LessonRevision | 1:N (published pointer: 1:0..1) |
| LessonRevision — Activity | 1:N |
| Lesson — Skill | N:M (LessonSkill) |
| Lesson — Lesson | N:M (LessonPrerequisite, DAG) |
| Activity — MediaAsset | N:M (payload reference orqali, relational junction'siz — MVP) |
| User — MediaAsset | 1:N (uploaded_by) |

## 29. Invariants

1. **I-1:** Har content node roppa-rosa bitta parentga tegishli (Activity → bitta LessonRevision; Track → bitta Subject; ...).
2. **I-2:** Bitta Lesson'da bir vaqtda ko'pi bilan bitta PUBLISHED revision; `published_revision_id` faqat shu revisionga ishora qiladi.
3. **I-3:** PUBLISHED (yoki bir paytlar PUBLISHED bo'lgan) revision va uning activitylari immutable — o'zgartirish faqat yangi draft revision orqali.
4. **I-4:** Revision PUBLISHED bo'lishi uchun barcha activitylari `type`↔`payload` validatsiyasidan o'tgan bo'lishi shart.
5. **I-5:** `source = AI_GENERATED/AI_ASSISTED` activity'li revision human review bosqichisiz PUBLISHED bo'la olmaydi.
6. **I-6:** Methodist faqat SubjectAssignment'ida bor Subject subtree'sida create/update/publish qila oladi.
7. **I-7:** LessonSkill faqat lesson'ning Subject'iga tegishli Skill bilan bog'lanadi (Skill.subject_id == Lesson zanjiridagi Subject).
8. **I-8:** LessonPrerequisite — DAG: self-reference yo'q, cycle yo'q (application-level tekshiruv).
9. **I-9:** ARCHIVED content yangi Roadmap/Daily Plan tanloviga tushmaydi (1.2B engine qoidasi; boshlagan learner uchun ko'rinish policy'si — 1.2B).
10. **I-10:** Learner faqat butun parent zanjiri PUBLISHED bo'lgan va PUBLISHED revision'ga ega lessonni ko'radi.
11. **I-11:** Onboarding tugamagan user (profile incomplete) assessment/learning flow'ga kira olmaydi (application guard).
12. **I-12:** Public/community kontekstda faqat public identity (display_name) chiqadi; phone/date_of_birth hech qachon expose qilinmaydi.
13. **I-13:** User.phone har doim canonical E.164 va unique.
14. **I-14:** SUSPENDED user hech qanday rol/permission bilan amal bajara olmaydi — state tekshiruvi permissiondan oldin.
15. **I-15:** MediaAsset reference qilinganda o'chirilmaydi — TD-82 junctionlari/FK ustunlari orqali **real FK RESTRICT** (Phase 1.2D'gacha application-check edi).
16. **I-16 (TD-81):** Har sensitive staff action StaffAudit yozuvi bilan bir tranzaksiyada; StaffAudit append-only, domain business truth'ning o'rnini bosmaydi.

## 30. Open questions

1. ~~TD-21..TD-30 review~~ — **yakunlandi, hammasi ACCEPTED**. Qolgan sof implementation detaillar (architecture qarori emas): UUIDv7 generatsiya joyi (DB vs application), payload validation library tanlovi, `ContentStatus` enum'ni bitta umumiy yoki per-entity qilish, Prisma relation naming.
2. Module/Topic uchun slug kerakmi (URL chuqurligi/SEO policy) — UI/routing bosqichida.
3. `preferred_language` semantikasi — UI tili product savoli ochiq (profil maydoni optional turadi).
4. MediaAsset `status` to'plamining aniq shakli — media moderation flow bilan birga.
5. AI provenance uchun full prompt/observability storage — AI arxitektura bosqichida.
6. Privacy erasure/anonymization policy (user "o'chirilganda" nima bo'ladi) — legal review bilan.
7. Level semantikasi til bo'lmagan fanlarda (Mathematics'da "Level" nima) — content dizayni bilan; model (data-driven Level) bunga tayyor.
8. LessonSkill'ga `weight/primary` qo'shish — 1.2B skill-profile engine dizayni bilan.
9. Cross-subject prerequisite'ga ruxsat berilishi kerakmi (hozir tavsiya: bitta Subject ichida).

## 31. Recommended model summary

1. **User + 1:1 UserProfile**; soft delete yo'q — states; public/private identity ajratilgan.
2. **Authorization:** Role jadvali (seeded) + UserRole (N:M) + RolePermission (permission_code, kod registry validatsiyasi); Methodist scope — SubjectAssignment junction.
3. **Content:** to'liq explicit zanjir Subject→Track→Level→Module→Topic→Lesson (Variant A; semantik kerak bo'lmagan qatlam uchun UI'da yashiriladigan system/default structural node); Level — data, enum emas.
4. **Lesson = logical id + LessonRevision** (versioning C: draft revision + immutable published snapshot, pointer swap).
5. **Activity:** bitta jadval + `type` enum + JSONB payload + application-level discriminated-union validation (publish'da strict); oddiy int position.
6. **Prerequisites:** explicit LessonPrerequisite DAG; sort_order faqat taqdimot.
7. **Skill:** Subject 1:N Skill; MVP'da LessonSkill (N:M), ActivitySkill — 1.2B.
8. **Lifecycle:** REVIEW bosqichi faqat LessonRevision'da; Lesson (logical) va containerlar DRAFT/PUBLISHED/ARCHIVED; Skill ACTIVE/ARCHIVED; AI content human review'siz publish bo'lmaydi.
9. **MediaAsset** — provider-neutral (storage_key), payload'dan id orqali reference.
10. **IDs:** UUIDv7; slugs — Subject/Track/Level(code)/Lesson; timestamps hamma joyda, global soft-delete yo'q; delete o'rniga ARCHIVE + RESTRICT.
