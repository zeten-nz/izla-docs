# Izlan — Learner Onboarding + Profile API (Phase 1.5A)

> Status: Phase 1.5A COMPLETE + **Phase 1.5A-2 COMPLETE (2026-08-19)** — architecture gap **RESOLVED** via `LearnerLearningIntent` (TD-93). Profile + onboarding foundation + learner subject/track selection persistence.
> Kod: `backend/src/profile`, `backend/src/onboarding`. Manba: [PRISMA_SCHEMA_V1.md](PRISMA_SCHEMA_V1.md), [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md), [AUTH_HTTP_IMPLEMENTATION.md](AUTH_HTTP_IMPLEMENTATION.md).
> Assessment / roadmap / daily plan / skill profile — YO'Q (keyingi fazalar).

## 1. Scope

Authenticated learner: o'z profilini o'qish/tahrirlash (displayName, dateOfBirth, timezone, preferredLanguage), onboarding holati/completion, published Subject/Track discovery. **Subject/Track SELECTION persistence — schema modeli yo'q → implement qilinmadi (§13, architecture gap).**

## 2–3. Profile model & editable fields

UserProfile (Phase 1.3, o'zgarmagan): displayName, dateOfBirth (`@db.Date`, nullable), timezone, preferredLanguage, onboardingCompletedAt. Self-editable: **displayName, dateOfBirth (onboarding'gacha), timezone, preferredLanguage**. Tahrirlab BO'LMAYDIGAN: phone, status, role, onboardingCompletedAt, id, permissions (DTO whitelist + explicit repository field map — mass assignment yo'q).

## 4–5. DOB / date semantics & age

- Format: **YYYY-MM-DD** (date-only). `parseDobOrThrow` → UTC midnight `Date`; **timezone shift YO'Q** (2007-05-12 doim 2007-05-12). Validatsiya: format, valid kalendar (Feb 30 rad), kelajakda emas.
- **Age SAQLANMAYDI** (TD/D-32) — `date_of_birth`dan runtime'da hisoblanadi (hozir response faqat dateOfBirth qaytaradi; age cache yo'q).
- **Minors:** age chegarasi/parental consent INVENT QILINMADI (§10 — legal policy OPEN). Faqat strukturaviy DOB validatsiya.

## 6. Timezone

`isValidIanaTimezone` — Node built-in (`Intl.supportedValuesOf('timeZone')` + `Intl.DateTimeFormat` runtime fallback). Qabul: Asia/Tashkent, Europe/Berlin, UTC. Rad: GMT+5, bo'sh, noma'lum. Canonical IANA string saqlanadi. **TD-91:** profil timezone o'zgarishi tarixiy DailyPlan/WeeklyProgress snapshotlarini QAYTA YOZMAYDI (bu modellar keyinroq) — 1.5A faqat joriy profil timezone'ni yangilaydi.

## 7. Preferred language

UI locale registry `['uz','en']` (§15). **UI tili ≠ learning subject tili.** Schema'da mavjud (`preferredLanguage`) — implement qilindi. To'liq locale strategiyasi OPEN.

## 8–10. Onboarding readiness & completion

- **Authority:** `UserProfile.onboardingCompletedAt` (qo'shimcha boolean yo'q). `completed = onboardingCompletedAt != null`.
- **Required (Phase 1.5A-2 yangilangan):** profile fields (displayName, dateOfBirth, timezone) **+ kamida bitta COMPLETE learning intent** — trackId to'ldirilgan va Subject+Track hozir PUBLISHED (`countCompleteVisible > 0`, aks holda `missing` ichida `learningIntent`). `canComplete = missing.length === 0`.
- **Completion:** conditional `updateMany where onboardingCompletedAt=null` → **first-write** timestamp saqlanadi; idempotent (qayta chaqiruv → o'sha timestamp). Mavjud `completedAt` **hech qachon qayta yozilmaydi/uncomplete qilinmaydi** (§25).
- **Resumable:** holat DB'dan derived; in-memory state yo'q. `next` field YO'Q (§24 — assessment entry route aniqlanmagan).

## 11–12. Subject / Track discovery

Read-only, faqat **PUBLISHED** (DRAFT/ARCHIVED yashirin). Deterministik tartib (sortOrder, title). Track — parent subject PUBLISHED bo'lishi shart; boshqa subject track'i sizib chiqmaydi. Draft/mavjud emas subject → 404 (yashirin content oshkor etilmaydi). Bo'sh katalog → `[]` (500 emas). Narrow `OnboardingContentRepository` (methodist CRUD yo'q).

## 13. ✅ Learner subject/track selection persistence — RESOLVED (TD-93, Phase 1.5A-2)

Phase 1.5A'da aniqlangan gap owner review'dan o'tdi → **`LearnerLearningIntent`** modeli ACCEPTED ([TECH_DECISIONS.md](TECH_DECISIONS.md) TD-93). Fake persistence/schema invent qilinmagan qaror to'g'ri chiqdi — owner haqiqiy modelni tasdiqladi.

**Model:** `learner_learning_intent` (id UUIDv7, userId, subjectId, trackId **nullable**, createdAt, updatedAt). `@@unique(userId, subjectId)`, `@@index(subjectId)`, `@@index(trackId)`. FK'lar `onDelete: Restrict` (content archive intentni destructive delete qilmaydi — LI-06). Migration: `20260819141741_add_learner_learning_intent`.

**Semantika:** learnerning muayyan Subject bo'yicha **hozirgi learning direction**. Multi-subject (unique per user+subject) → UserProfile.selectedSubjectId EMAS. trackId nullable → onboarding resumable. status/goal/versioning YO'Q.

**Rad etilgan variantlar (owner qarori):**
- `UserProfile.selectedSubjectId/selectedTrackId` — REJECTED (multi-subject cheklaydi, identity↔content coupling).
- `LearnerRoadmap` PROVISIONAL status — REJECTED (§30 "roadmap assessment'dan keyin" buziladi).
- `SubjectAssignment` (staff scope) — **ABSOLUTELY REJECTED** (learner selection uchun ishlatilmaydi).

**API:** `GET /api/onboarding/learning-intents` (own view), `PUT /api/onboarding/learning-intent {subjectId, trackId?}` (single resumable upsert). Validation: Subject PUBLISHED (LI-04), Track PUBLISHED (LI-05), track.subjectId==subjectId (LI-03) — [DB_CONSTRAINT_MATRIX.md](DB_CONSTRAINT_MATRIX.md) §9b. DELETE endpoint yo'q.

## 14. Content lifecycle filters

Har discovery query **explicit** `status = PUBLISHED` (filtrsiz `findMany` yo'q, §35). Test'lar PUBLISHED/DRAFT/ARCHIVED kiritib faqat PUBLISHED ko'rinishini tasdiqlaydi.

## 15. API endpoints

| Method | Path | Auth | Status |
|---|---|---|---|
| GET | /api/profile/me | Bearer | 200 |
| PATCH | /api/profile/me | Bearer | 200 |
| GET | /api/onboarding/status | Bearer | 200 |
| POST | /api/onboarding/complete | Bearer | 200 (409 agar incomplete) |
| GET | /api/onboarding/subjects | Bearer | 200 |
| GET | /api/onboarding/subjects/:subjectId/tracks | Bearer | 200 (404 unknown/hidden) |
| GET | /api/onboarding/learning-intents | Bearer | 200 (own view) |
| PUT | /api/onboarding/learning-intent | Bearer | 200 (404 subject/track, 400 mismatch) |

Learning intent persistence (TD-93, §13) — **IMPLEMENT QILINDI**. `PUT` — single resumable upsert (subjectId majburiy, trackId optional). DELETE endpoint yo'q. `/auth/me` minimal auth bootstrap sifatida o'zgarmadi (§6).

## 16. Error codes

`PROFILE_INVALID_TIMEZONE` (400), `PROFILE_INVALID_DOB` (400), `PROFILE_DOB_LOCKED` (409), `PROFILE_INCOMPLETE` (409, message=missing), `RESOURCE_NOT_FOUND` (404), `LEARNING_SUBJECT_NOT_AVAILABLE` (404), `LEARNING_TRACK_NOT_AVAILABLE` (404), `LEARNING_TRACK_SUBJECT_MISMATCH` (400). Phase 1.4C error konventsiyasi (`{statusCode, code, message}`); Prisma/internal leak yo'q.

## 17. PII / security

- **IDOR himoya:** barcha own-profile amallar `principal.userId` (AuthGuard) — target userId REQUEST'dan HECH QACHON olinmaydi.
- **Mass assignment:** DTO whitelist + repository explicit field map.
- **PII:** DOB/profile body log qilinmaydi; DOB JWT'da yo'q, SecurityEvent'da yo'q. Response'da phone/role/permission yo'q.
- **DOB post-onboarding (§25):** oddiy PATCH orqali DOB o'zgartirish onboarding'dan keyin **rad etiladi** (`PROFILE_DOB_LOCKED`) — xavfsizlik cheklovi (permanent product rule emas; policy OPEN). displayName/timezone tahrirlanaveradi.
- Cache-Control: no-store (profil PII).
- **StaffAudit / SecurityEvent:** user o'z profilini tahrirlashi uchun ishlatilmaydi (§42).

## 18. Tests

- **Unit:** dob.util (5 — valid/future/calendar/format/null), timezone.util (Asia/Tashkent/Berlin/UTC accept, GMT+5/bo'sh/noma'lum reject).
- **E2e (16):** GET profile (no phone/role leak), auth required, PATCH (trim/date-only/tz/lang), invalid tz/DOB → 400, mass assignment reject, empty displayName reject + Unicode accept, onboarding status flow (missing shrinks + `learningIntent`, complete idempotent first-write), DOB locked after onboarding, subject/track PUBLISHED-only + belong-to-subject + deterministic order, DRAFT/unknown subject → 404, empty catalog → []. **Learning intent (5):** resumable subject-only→track (upsert, 1 row), multi-subject coexist (2nd overwrite qilmaydi), DRAFT/ARCHIVED subject + cross-subject track + DRAFT track reject, same user+subject upsert / different users same subject allowed, auth required.
- Regression: 1.4A/1.4B/1.4C/1.5A PASS (unit 77, e2e 62).

## 19. Deferred assessment boundary

Subject/Track tanlash **hech qanday** AssessmentAttempt/question/roadmap/AI yaratmaydi (§33). Discovery faqat keyingi assessment fazasi uchun input beradi.

## 20. Schema gaps

**Learner subject/track selection persistence** (§13) — **RESOLVED** (`LearnerLearningIntent`, TD-93, Phase 1.5A-2). Boshqa blocking schema gap yo'q.

## OPEN (non-blocking)

- Minors legal/parental policy · post-onboarding DOB edit policy · full UI locale strategiyasi — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md). (learner selection persistence gap RESOLVED.)
