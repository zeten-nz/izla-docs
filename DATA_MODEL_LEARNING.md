# Izlan — Learning Domain Data Model Architecture

> Status: Phase 1.2B — **COMPLETE**; Phase 1.2D refinementlari kiritilgan (TD-82 response media FK, TD-83 AssessmentDefinitionVersion+VersionItem, TD-84 DailyMissionCompletion, TD-88 LearnerLessonCompletion — hammasi ACCEPTED).
> **TD-31..TD-44 ACCEPTED** ([TECH_DECISIONS.md](TECH_DECISIONS.md)); TD-35/37/40/42 owner clarificationlari shu hujjatga kiritilgan.
> Foundation: [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md) (TD-21..TD-30 ACCEPTED). Product asos: [LEARNING_SYSTEM.md](LEARNING_SYSTEM.md), [DAILY_PLAN.md](DAILY_PLAN.md), [AI_SYSTEM.md](AI_SYSTEM.md).

## 1. Goals

1. Assessment, Skill Profile, Roadmap, Learning Session, Attempts, Mistake/Signal, Progress va Checkpoint uchun Prisma-ready model.
2. Adaptive/personalization engine'ga ishonchli **structured learning state** berish (D-14: AI memory'si LLM conversation'ga tayanmaydi).
3. Progress/scoring formulalari OPEN bo'lgani uchun — formula DB strukturasiga hard-code qilinmaydi; evidence'dan recompute mumkin bo'ladi.
4. 1.2A foundationni buzmaslik: immutable LessonRevision, JSONB Activity, UUIDv7, ARCHIVE-first deletion.
5. Reward (XP/IZL), subscription, community — 1.2C; bu yerda faqat extension pointlar.

## 2. Scope / non-scope

**Scope (NOW):** AssessmentDefinition/Item/Attempt/Response, AiEvaluation, LearnerSkillState, SkillMeasurement, LearningSession, ActivityAttempt, ActivitySkill (1.2A'dan ko'chirilgan NEXT), LearnerLessonProgress, LearnerRoadmap/RoadmapItem, LearnerRecommendation/RoadmapChange, LearnerSignal, Checkpoint, DailyPlan/DailyPlanItem, LearningSchedulePreference, WeeklyProgress.

**Non-scope (1.2C):** XP ledger, IZL ledger/balance, reward transactions/redemption, subscription, pricing, payment (Click/Payme), Community, reputation, notifications. Learning activity reward tizimiga **signal/event beradi** — extension point sifatida ko'rsatiladi (§26, §31), lekin reward jadvallari yaratilmaydi.

## 3. Core principles

1. **Evidence append-only, state mutable-derived** (§4) — hech bir formula evidence'ni almashtirmaydi.
2. **Content reference hech qachon yo'qolmaydi:** har attempt qaysi Activity/LessonRevision/AssessmentItem asosida bajarilganini saqlaydi (TD-24 immutability shuni kafolatlaydi).
3. **AI — yozuvchi emas, o'quvchi + taklifchi:** AI recommendation yozadi, Learner acceptance'siz roadmap o'zgarmaydi (§15); skill state'ni faqat engine yozadi.
4. **Deterministic birinchi:** aniq javobli itemlar backend deterministic scoring bilan; AI evaluation alohida async entity (§8).
5. **Volume discipline:** eng katta jadvallar (ActivityAttempt, AssessmentResponse) tor va JSONB-intizomli; view-only bloklar attempt yaratmaydi (§11); generic event table yo'q (§22-rad).
6. TD-21/22 falsafasi javoblarga ham: heterogeneous answer = JSONB + strict app-level validation.

## 4. Evidence vs current state

| Variant | Baho |
|---|---|
| A — faqat raw attempts, state har safar hisoblanadi | Har Daily Plan/AI chaqiruvda minglab attempt scan — qimmat; latency; AI context assembly og'ir |
| B — faqat current aggregate state | Explainability yo'qoladi ("nega 61%?"), formula o'zgarsa recompute qilib bo'lmaydi, debugging imkonsiz, checkpoint oldin/keyin taqqoslash sinadi |
| **C — raw immutable evidence + materialized current state** | Evidence (attempts, responses, measurements) append-only; engine ulardan mutable current state (skill state, progress, signals) hisoblab yozadi. Formula o'zgarsa evidence'dan recompute. ✅ |

**PROPOSED (TD-31): Option C.** Oqim:

```
Evidence (ActivityAttempt, AssessmentResponse, AiEvaluation)
        ↓  (Progress/Skill Engine — formulalar shu yerda, DB'da emas)
Derived milestones (SkillMeasurement)  +  Current state (LearnerSkillState,
LearnerLessonProgress, LearnerSignal, WeeklyProgress)
```

Recompute: evidence saqlanib turgani uchun engine yangi formula bilan state'ni qayta qura oladi; SkillMeasurement'lar "o'sha paytdagi o'lchov" sifatida tarixiy qoladi (recompute ularni o'chirmasligi kerak — yangi source bilan yangi measurement yoziladi).

## 5. Assessment definition

**Savol:** assessment savollari Lesson Activity'laridan reuse qilinsinmi?

- Reuse foydasi: bitta content mexanizmi, bitta builder/validation.
- Reuse xavfi: (1) lesson activity pedagogik kontekstga (theory oqimiga) bog'langan — diagnostic item mustaqil bo'lishi kerak; (2) diagnostic itemga **calibration metadata** (skill, difficulty) kerak — lesson activity'da yo'q va bo'lmasligi kerak; (3) Methodist lessonni tahrirlasa assessment sifati yashirin o'zgaradi; (4) lesson content'i learnerga o'quv jarayonida ko'ringan — test itemi sifatida qayta ishlatish o'lchov sifatini buzadi.

**PROPOSED (TD-32): assessment content alohida, lekin TD-21 falsafasi bilan.**

### AssessmentDefinition

| Field | Izoh |
|---|---|
| id | UUIDv7 — **logical stable identity** (TD-83) |
| subject_id | FK |
| purpose_scope | enum: DIAGNOSTIC / CHECKPOINT (fan-neytral) |
| title, description? | |
| status | DRAFT / PUBLISHED / ARCHIVED (TD-28 container pattern) |
| current_version_id | nullable FK → AssessmentDefinitionVersion — joriy published pointer |
| created_by | FK → User (Methodist) |
| timestamps | |

### AssessmentDefinitionVersion (TD-83, ACCEPTED — append-only)

| Field | Izoh |
|---|---|
| id, assessment_definition_id | FK |
| version_no | int; unique(definition, version_no) |
| config | JSONB — adaptive engine parametrlari (scoring algorithm OPEN — config data, schema emas); **publish bo'lgach immutable** |
| status | DRAFT / PUBLISHED / ARCHIVED |
| created_by, published_at, timestamps | |

### AssessmentVersionItem (TD-83 owner clarification — item pool membership snapshot)

| Field | Izoh |
|---|---|
| assessment_definition_version_id + assessment_item_id | **unique pair** |
| ordering/calibration override | faqat chindan version-specific bo'lsa |

Draft versionda membership tahrirlanadi; **published bo'lgach membership qatorlari ham immutable**. Reproducibility zanjiri: attempt → version → exact config + exact eligible item pool; + Response sequence + engine_version. (2026 v3 = {A,B,C,D}, 2027 v4 = {A,C,E,F} — eski attempt qaysi pooldan ishlagani aniq.)

### AssessmentItem

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| assessment_definition_id | FK — MVP'da item pool definition'ga tegishli |
| type | Activity type enum'ining assessment'ga mos subseti (MINI_QUESTION uslubidagi obyektiv itemlar, SPEAKING, WRITING, LISTENING...) |
| payload | JSONB + strict discriminated-union validation (TD-22 bilan bir xil mexanizm) |
| skill_id | FK → Skill — item qaysi skillni o'lchaydi |
| difficulty | int/rank — adaptive tanlov uchun (fan-neytral ichki shkala; English'da CEFR'ga map qilinadi, lekin schema CEFR emas) |
| status | DRAFT / PUBLISHED / ARCHIVED |
| source, ai_metadata | AI provenance — Activity'dagi kabi (human review'siz publish yo'q) |
| timestamps | |

**Immutability:** PUBLISHED AssessmentItem tahrirlanmaydi — o'zgarish = yangi item + eskisi ARCHIVED. Shu tufayli AssessmentResponse `item_id`ga ishora qilishi yetarli — snapshot kerak emas. AssessmentSection MVP'da yo'q (adaptive oqim section'larga bo'linmaydi); kerak bo'lsa keyin qo'shiladi.

## 6. Assessment attempts

### AssessmentAttempt

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| user_id | FK |
| assessment_definition_id | FK (denormalized) |
| definition_version_id | **FK → AssessmentDefinitionVersion (TD-83)** — reproducibility |
| subject_id | FK (denormalized — definition'dan; user+subject querylari uchun) |
| track_id | nullable FK — diagnostic qaysi track kontekstida |
| checkpoint_id | nullable FK → Checkpoint (purpose=CHECKPOINT bo'lsa) |
| purpose | enum: INITIAL_DIAGNOSTIC / CHECKPOINT / REASSESSMENT (fan-neytral) |
| status | IN_PROGRESS / COMPLETED / ABANDONED |
| engine_state | JSONB — adaptive joriy holat (§7) |
| engine_version | string/int — reproducibility uchun |
| result_summary | JSONB, nullable — tugagach display uchun cache (haqiqiy natija — SkillMeasurement qatorlari) |
| started_at, completed_at | |
| timestamps | |

Natija ikki joyda: (1) **SkillMeasurement** qatorlari (normalized, source=attempt) — engine/tarix uchun haqiqat; (2) attempt.result_summary — UI cache. Context intake javoblari (learning goal, o'z bahosi...) — onboarding'da UserProfile/roadmap generation kirishlariga boradi; kerak bo'lsa attempt.engine_state ichida boshlang'ich kontekst sifatida qayd etiladi.

## 7. Adaptive state

| Variant | Baho |
|---|---|
| A — faqat attempt JSONB state | Resume oson, lekin qadamlar tarixi yo'q — audit/replay yo'q |
| B — faqat normalized decision records | To'liq audit, lekin engine joriy pozitsiyasini har safar qayta qurishi kerak |
| **C — hybrid** | Har taqdim etilgan item + javob = **AssessmentResponse** qatori (normalized evidence, deterministic replay manbai); joriy pozitsiya/daraja oynasi = attempt.**engine_state** JSONB (resume uchun kichik snapshot); engine_version bilan reproducibility ✅ |

**PROPOSED (TD-33): C.** Resume after disconnect: engine_state'dan davom; state buzilsa — response'lardan deterministic qayta qurish mumkin. Obyektiv itemlar scoring'i to'liq deterministic (AI'siz).

## 8. Responses / evaluations

### AssessmentResponse

| Field | Izoh |
|---|---|
| id, attempt_id | FK |
| item_id | FK → AssessmentItem (immutable — content reference yo'qolmaydi) |
| sequence_no | int — taqdim etilish tartibi; unique(attempt_id, sequence_no) |
| answer | JSONB (type'ga mos; §12 bilan bir xil qoidalar) |
| response_media_asset_id | nullable FK → MediaAsset (TD-82) — speaking/audio javob (canonical bitta media) |
| status | PRESENTED / DRAFT / SUBMITTED / SKIPPED |
| is_correct | nullable bool (deterministic itemlar) |
| deterministic_score | nullable |
| response_time_ms | nullable |
| presented_at, submitted_at | |

SUBMITTED bo'lgach answer append-only — o'zgartirilmaydi (§24 idempotency, I-1).

### AiEvaluation (alohida entity — response ichidagi JSONB emas)

**Nega alohida (PROPOSED, TD-34):** (1) evaluation **async** — speaking/writing baholash vaqt oladi, o'z lifecycle'i bor (PENDING/COMPLETED/FAILED); (2) re-evaluation mumkin (bir response'ga bir nechta evaluation, oxirgisi amal qiladi); (3) kelajak AI cost tracking shu entitylarga bog'lanadi; (4) assessment response ham, lesson ActivityAttempt ham bitta mexanizmdan foydalanadi.

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| assessment_response_id / activity_attempt_id | nullable FK'lar — **roppa-rosa bittasi to'ldirilgan** (invariant I-8) |
| status | PENDING / COMPLETED / FAILED |
| score | nullable numeric |
| rubric | JSONB — categories/mezonlar bo'yicha structured natija (D-12 talabi) |
| feedback | text — learnerga ko'rsatiladigan tushuntirish |
| provider_metadata | JSONB, nullable — model/provider identifikatori (provider OPEN — shuning uchun JSONB) |
| evaluation_version | rubric/prompt versiyasi — consistency tahlili uchun |
| created_at, completed_at | |

## 9. Skill state / history

### LearnerSkillState (current, mutable — faqat engine yozadi)

| Field | Izoh |
|---|---|
| user_id + skill_id | unique pair |
| mastery_score | numeric (normalized ichki shkala, masalan 0–100) — engine'ning haqiqiy qiymati |
| display_level | string/code, nullable — taqdimot yorlig'i (English'da "B1"); **content Level jadvaliga FK EMAS** (Level track strukturasiga tegishli; skill yorlig'i subject-scale mapping orqali engine'da hisoblanadi — mapping OPEN) |
| confidence | numeric, nullable — o'lchov ishonchliligi |
| evidence_count | int — nechta evidence hisobga olingan |
| last_measurement_at, updated_at | |

**Owner clarification (TD-35):** authoritative mathematical state — **mastery_score + confidence + evidence_count**. `display_level` authoritative emas — derived representation (masalan `mastery_score=0.63, confidence=0.81 → proficiency mapping → B1`); mapping keyinchalik o'zgarishi mumkin va content `Level` entity'siga hard FK bilan bog'lanmaydi. Formula/mapping o'zgarsa display qayta hisoblanadi, evidence o'zgarmaydi. Englishga hard-code yo'q.

### SkillMeasurement (append-only history)

| Field | Izoh |
|---|---|
| id, user_id, skill_id | |
| source | enum: DIAGNOSTIC / CHECKPOINT / LESSON_MASTERY / AI_EVALUATION / ENGINE_RECALC |
| assessment_attempt_id / lesson_id | nullable source referencelari |
| score, display_level, confidence | o'sha paytdagi qiymatlar |
| created_at | |

**Chegara (TD-35):** SkillMeasurement **har mini savolda yozilmaydi** — faqat mazmunli chegaralarda: diagnostic tugashi, checkpoint tugashi, lesson mastery tugashi, engine'ning davriy qayta hisoblashi. Mayda signallar raw ActivityAttempt'da qoladi; measurement = milestone snapshot. "Checkpoint'da oldin/keyin" savoli shu jadvaldan derived (alohida comparison jadvali yo'q, §17).

## 10. Learning sessions

### LearningSession

| Field | Izoh |
|---|---|
| id, user_id | |
| status | ACTIVE / ENDED |
| started_at, ended_at | |
| active_seconds | int — engine hisoblagan real faol vaqt (aggregate) |
| daily_plan_id | nullable FK — qaysi kun rejasi kontekstida |
| timestamps | |

- Session ≠ app ochiq vaqt — real learning activity davri. Idle'ni qanday hisoblash — implementation detail; model faqat **aggregate** saqlaydi, timer mexanizmiga bog'lanmagan.
- Raw heartbeat'lar bu modelda YO'Q (§22-time-tracking).
- 1.2C extension: reward engine "real learning session" tekshiruvida shu aggregate + attempt evidence'dan foydalanadi (reward columnlar bu yerda yo'q).

## 11. Activity attempts

**Qaysi type attempt yaratadi (TD-36):** faqat javob talab qiladigan bloklar — MINI_QUESTION, PRACTICE, SPEAKING, WRITING, LISTENING, MASTERY_TEST, AI_INTERACTION. **Text/Explanation/Image/Audio/Example attempt yaratmaydi** — ularning viewed/completed holati LearnerLessonProgress ichida kompakt saqlanadi (§13). Alohida ActivityProgress/Event jadvali yo'q — volume himoyasi.

### ActivityAttempt

| Field | Izoh |
|---|---|
| id | UUIDv7 |
| user_id, activity_id | FK |
| lesson_revision_id | FK (activity'dan denormalized — user+revision querylari uchun) |
| learning_session_id | nullable FK |
| roadmap_item_id | nullable FK — qaysi plan kontekstida (review/practice attribution) |
| attempt_no | int; **unique(user_id, activity_id, attempt_no)** |
| status | IN_PROGRESS / SUBMITTED / EVALUATED |
| answer | JSONB (§12) |
| response_media_asset_id | nullable FK → MediaAsset (TD-82) — speaking audio (canonical bitta media; ko'p asset kerak bo'lsa junction additive) |
| is_correct | nullable bool |
| deterministic_score | nullable |
| response_time_ms | nullable |
| client_request_id | nullable — retry dedup (§24) |
| started_at, submitted_at | |
| created_at | |

Eng katta jadval nomzodi — tor saqlanadi; katta media javoblar MediaAsset reference orqali (§12). SUBMITTED'dan keyin answer immutable. AI-evaluated attempt'lar AiEvaluation bilan bog'lanadi (status=EVALUATED evaluation tugagach).

### ActivitySkill (1.2A'da NEXT deb belgilangan — endi kiritiladi)

`activity_id + skill_id` junction, unique(pair) — mistake→skill attributioni uchun (Methodist/builder to'ldiradi; bo'lmasa LessonSkill fallback). Invariant: skill activity'ning Subject'iga tegishli (I-9).

## 12. Answer storage

| Variant | Baho |
|---|---|
| **A — attempt.answer JSONB** | TD-21/22 bilan to'liq consistent: activity type discriminator allaqachon bor; bitta validation mexanizmi ✅ |
| B — type-specific response jadvallar | TD-21'da rad etilgan polymorphism muammosining aynan o'zi |
| C — hybrid | Cross-cutting maydonlar (is_correct, score, response_time) allaqachon column — qolgani payload |

**ACCEPTED (TD-36 + TD-82 refinement): A (aslida to'g'ri qilingan C — TD-21'dagi kabi).** Shakllar: MCQ → `{selected_option_id}`; Writing → `{text}`; Speaking → audio **`response_media_asset_id` FK ustunida** (TD-82 — relational truth; payload'da raw MediaAsset UUID saqlanmaydi; MediaAsset — TD-25 reuse, ownership/moderation bor); Listening → type'ga mos. Har biri activity type schema'siga qarshi strict validate qilinadi.

## 13. Lesson progress

### LearnerLessonProgress (mutable current state)

| Field | Izoh |
|---|---|
| user_id + lesson_id | unique pair (logical Lesson) |
| lesson_revision_id | FK — **boshlaganda pin qilingan revision** |
| status | IN_PROGRESS / COMPLETED |
| completed_activities | JSONB — tugatilgan activity id'lar/pozitsiya (view-only bloklar ham shu yerda) |
| last_activity_id | nullable FK — resume nuqtasi |
| completion_pct | cached int — engine hisoblaydi (formula OPEN — shuning uchun cache, haqiqat evidence'da) |
| mastery_best_score | nullable cached — mastery test eng yaxshi natijasi (evidence: attempts) |
| started_at, completed_at, updated_at | |

**Versioning policy (TD-37, ACCEPTED — owner clarification bilan):**

- Boshlanmagan lesson → har doim **latest PUBLISHED revision**.
- Learner lessonni boshlaganda joriy PUBLISHED revision **pin qilinadi** va u lessonni **shu revisionda tugatadi** — v4 chiqsa ham v3'da davom etadi; **silent migration qilinmaydi**.
- Yangi/boshlamagan learnerlar yangi PUBLISHED revisionni oladi.
- COMPLETED progress o'z revision reference'ini abadiy saqlaydi (L-5).
- Jiddiy content xatosi uchun kelajakda **explicit force-migration mechanism** bo'lishi mumkin — bu ordinary publish flow emas va hozir implement qilinmaydi.
- Tugatilgan lessonni qayta ochganda qaysi versiya ko'rsatilishi — OPEN (display policy).

### LearnerLessonCompletion (TD-88, ACCEPTED — append-only history)

Current state va tarix ajratiladi: **LearnerLessonProgress = CURRENT state** (relearn boshlanganda yangi PUBLISHED revision pin qilinib reset bo'la oladi); **LearnerLessonCompletion = historical fact**:

| Field | Izoh |
|---|---|
| id, user_id, lesson_id | |
| lesson_revision_id | Qaysi revision tugatilgan — abadiy saqlanadi (eski L-5 kafolati endi shu yerda) |
| completion_no | int; **unique(user, lesson, completion_no)** |
| completed_at (+foydali bo'lsa started_at/mastery snapshot) | |

Oqim: 2026 — v3 complete → Completion #1(v3); 2027 — relearn → Progress reset (v8 pin) → Completion #2(v8). v3 tarixi yo'qolmaydi; ActivityAttempt'lar raw evidence bo'lib qolaveradi.

## 14. Roadmap

### LearnerRoadmap

| Field | Izoh |
|---|---|
| id, user_id, subject_id, track_id | FK'lar |
| status | ACTIVE / COMPLETED / ARCHIVED |
| source_assessment_attempt_id | nullable FK — qaysi diagnostic asosida generatsiya qilingan (explainability) |
| generated_at | |
| timestamps | |

Invariant: bitta (user, subject) uchun bir vaqtda ko'pi bilan bitta ACTIVE roadmap (I-6). Track almashtirish → eski ARCHIVED, yangi ACTIVE.

### RoadmapItem

| Field | Izoh |
|---|---|
| id, roadmap_id | FK |
| item_type | enum: LESSON / REVIEW / PRACTICE / CHECKPOINT |
| lesson_id | nullable FK — LESSON/REVIEW uchun (**logical Lesson**, §14-reference qarori quyida) |
| checkpoint_id | nullable FK — CHECKPOINT uchun |
| skill_id | nullable FK — skill-targeted PRACTICE/REVIEW uchun |
| position | int — roadmap ichidagi tartib |
| status | PENDING / IN_PROGRESS / COMPLETED / SKIPPED |
| source | enum: INITIAL_GENERATION / RECOMMENDATION / SYSTEM |
| reason | qisqa matn/kod — "nega bu item bor" (explainability) |
| roadmap_change_id | nullable FK — qaysi o'zgarish qo'shgan (§15) |
| params | JSONB, nullable — type'ga xos qo'shimchalar |
| timestamps | |

Type-specific alohida jadvallar yo'q — bitta typed item + nullable FK + params (overengineering'siz flexible).

**Content reference qarori (TD-38):** Roadmap **logical Lesson**ga bog'lanadi (plan darajasi); learner itemni boshlaganda revision **LearnerLessonProgress'da** pin qilinadi (§13). Plan va execution ajratilgan: plan barqaror lesson identity bilan ishlaydi, tarix revision bilan.

## 15. Roadmap adjustments

Accepted qoida: `AI recommendation → Learner accepts → roadmap adjustment` — model buni **isbotlay olishi** kerak.

### LearnerRecommendation (AI-specific nomlanmagan — system recommendation ham bo'ladi)

| Field | Izoh |
|---|---|
| id, user_id | |
| type | enum: ROADMAP_ADJUSTMENT / REVIEW_SUGGESTION / SCHEDULE_ADJUSTMENT / ... |
| source | enum: AI_TUTOR / SYSTEM_RULE (kelajakda kengayadi) |
| roadmap_id | nullable FK |
| reason | text — learnerga ko'rsatilgan asos |
| proposed_change | JSONB — nima taklif qilinyapti |
| signal_refs | JSONB, nullable — qaysi LearnerSignal'lar asos bo'ldi (explainability zanjiri) |
| status | PROPOSED / ACCEPTED / REJECTED / EXPIRED |
| responded_at | nullable |
| created_at | |

Kontent (reason/proposed_change) immutable; faqat status + responded_at o'zgaradi.

### RoadmapChange (append-only audit)

| Field | Izoh |
|---|---|
| id, roadmap_id | |
| recommendation_id | nullable FK — recommendation orqali kelgan bo'lsa **majburiy ACCEPTED bo'lishi shart** (I-7) |
| change_type | enum/kod: ITEM_ADDED / ITEM_REMOVED / ITEM_REORDERED / REGENERATED ... |
| change_payload | JSONB — nima o'zgardi |
| applied_by | SYSTEM / USER (actor turi) |
| applied_at | |

**Roadmap versioning (TD-39):** mutable current itemlar + append-only RoadmapChange log = minimal audit. "Roadmap nega o'zgardi?" — change log'dan javob. Full git-like snapshot versioning — rad (overengineering).

## 16. Mistakes / signals

**Savol:** maxsus MistakePattern entity'mi yoki generic signal model?

Taxonomy hali final emas ("since/for confusion", "article omission"...) — qattiq mistake jadvali erta. AI/engine'ga esa attemptlarni har safar qayta o'qimasdan ishlaydigan darajada tayyor signal kerak.

**PROPOSED (TD-40): bitta generic `LearnerSignal` (derived, engine yozadi, persisted — explainability/lifecycle uchun):**

| Field | Izoh |
|---|---|
| id, user_id, subject_id | |
| type | enum (kichik, boshlang'ich to'plam — tuning): REPEATED_MISTAKE / REVIEW_DUE / CONSISTENCY_RISK / ... |
| skill_id | nullable FK |
| lesson_id / topic_id | nullable FK — lokalizatsiya |
| category_code | nullable **string** — moslashuvchan mistake taxonomy (masalan `present_perfect.since_for`); registry application/config'da rivojlanadi, enum EMAS (taxonomy OPEN) |
| strength | numeric — signal kuchi (masalan takror soni/og'irligi) |
| evidence_refs | JSONB — asos bo'lgan attempt id'lari/oynasi (chegaralangan ro'yxat) |
| status | ACTIVE / RESOLVED / EXPIRED |
| first_detected_at, last_seen_at, resolved_at | |

**Owner clarification (TD-40):** signal — har qanday metric emas, **actionable learning insight**. Current score/normal progress LearnerSkillState'da yashaydi — signal sifatida yozilmaydi. Lifecycle: ACTIVE → RESOLVED yoki ACTIVE → EXPIRED. Exact taxonomy OPEN.

Nega persisted (pure derived emas): (1) AI "bu xato 3-marta qaytdi" deya olishi kerak — explainability; (2) RESOLVED/EXPIRED lifecycle (review qilingandan keyin signal yopiladi); (3) recommendation generation kirishi. Recompute mumkin — evidence saqlangan. Unique'lik: (user, type, skill/category kombinatsiyasi) bo'yicha bitta ACTIVE signal — dedup engine qoidasi (app-level).

## 17. Reviews

Alohida ReviewHistory jadvali **kerak emas** — review to'liq mavjud entitylar bilan ifodalanadi:

```
LearnerSignal (REVIEW_DUE / REPEATED_MISTAKE)
→ LearnerRecommendation (REVIEW_SUGGESTION, reason, signal_refs)
→ [Learner accepts]
→ RoadmapChange → RoadmapItem (type=REVIEW, reason, skill/lesson ref)
→ ActivityAttempt'lar (roadmap_item_id bilan)
→ item COMPLETED + signal RESOLVED
```

"Nima sababdan, qaysi lesson/skill, qachon, natija" — shu zanjirdan to'liq javob beriladi. Checkpoint oldin/keyin kabi, bu ham derived tarix.

## 18. Checkpoints

`has_checkpoint` boolean EMAS — to'liq entity:

### Checkpoint

| Field | Izoh |
|---|---|
| id, subject_id | |
| module_id | FK → Module (MVP boundary; unique(module_id) — bitta module'ga bitta checkpoint) |
| assessment_definition_id | FK → AssessmentDefinition (Methodist tuzgan test — I-10) |
| status | DRAFT / PUBLISHED / ARCHIVED |
| created_by, timestamps | |

Kelajakda boshqa pedagogical boundary kerak bo'lsa — module_id nullable qilinib boundary reference kengayadi (hozir qilinmaydi — premature).

**Checkpoint attempt (TD-41):** alohida jadval YO'Q — `AssessmentAttempt (purpose=CHECKPOINT, checkpoint_id)`. DRY: bir xil response/evaluation/scoring mexanizmi. **Oldin/keyin taqqoslash:** saqlanmaydi — SkillMeasurement tarixidan derived (checkpoint measurement vs oldingi measurementlar); attempt.result_summary'da UI cache bo'lishi mumkin.

## 19. Daily Plan persistence

| Variant | Baho |
|---|---|
| A — har requestda dynamic generate | Ertalab ko'rgan plan kunduzi o'zgarib qolishi mumkin; IZL eligibility keyin "o'sha kungi target"ga bog'lanadi — audit yo'q; explainability yo'q |
| B — faqat snapshot | Adjustment ("bugun 30 min") va item holatlari uchun structure kerak |
| **C — persisted snapshot + items** | Kunlik plan barqaror; adjustment nazorat ostida regeneratsiya; 1.2C IZL eligibility aynan plan itemlariga bog'lana oladi ✅ |

**PROPOSED (TD-42): C.**

### DailyPlan — versioned/supersedable snapshot (owner clarification, TD-42)

`unique(user, date)` mutable singleton **EMAS** — har regeneratsiya yangi snapshot:

| Field | Izoh |
|---|---|
| id, user_id | |
| local_date | date — learner'ning lokal kuni |
| generation_no | int; **unique(user_id, local_date, generation_no)** |
| status | CURRENT / SUPERSEDED |
| available_time_min | nullable — "bugun qancha vaqtingiz bor?" javobi |
| context | JSONB — generation kirishlari xulosasi (explainability) |
| generated_at, timestamps | |

Qoidalar: bir (user, local_date) uchun bir vaqtda faqat **bitta CURRENT** plan; regeneratsiya ("bugun 30 min") → yangi generation CURRENT, eskisi SUPERSEDED; oldingi snapshotlar **rewrite/delete qilinmaydi**; DailyPlanItem aynan o'z generation snapshot'iga tegishli. Bu 1.2C IZL eligibility auditi uchun muhim.

Misol:

```
2026-08-20:  generation 1 · 90 min · SUPERSEDED
             generation 2 · 30 min · CURRENT
```

### DailyPlanItem

| Field | Izoh |
|---|---|
| id, daily_plan_id | |
| section | enum: MUST_DO / RECOMMENDED / EXTRA |
| item_type | enum: LESSON / REVIEW / PRACTICE / MISSION / COMMUNITY / OTHER |
| roadmap_item_id | nullable FK — roadmap bilan bog'lanish |
| lesson_id / skill_id | nullable FK'lar |
| params | JSONB — type'ga xos (mission tavsifi va h.k.) |
| position, status | PENDING / COMPLETED / SKIPPED |
| completed_at | |

Reward logic YO'Q — 1.2C reward engine "shu kuni target nima edi va bajarildimi"ni shu jadvallardan o'qiydi (extension point). Daily mission katalogi/reward mapping — 1.2C.

### DailyMissionCompletion + Evidence (TD-84, ACCEPTED — append-only)

Ajratish: **DailyPlanItem** = mission assignment (plan snapshot); **DailyMissionCompletion** = "mission bajarildi" business evidence; **RewardGrant** (finance) = financial decision.

| Entity | Fieldlar |
|---|---|
| DailyMissionCompletion | id, user_id, daily_plan_item_id (**unique** — one-shot), completed_at, kerak bo'lsa completion_type |
| DailyMissionCompletionEvidence (1:N, append-only) | id, completion_id, typed nullable FK'lar: community_post_id / activity_attempt_id / learning_session_id (**har qatorda roppa-rosa bittasi — XOR**; kelajak turlari additive), created_at |

Oqim: `DailyPlanItem → DailyMissionCompletion (+Evidence) → Reward Engine → RewardGrant → IZL Ledger`. Polymorphic source ISHLATILMAYDI — financial evidence FK integrity talab qiladi. Ko'p-evidence mission (masalan "15 daqiqa") — bir nechta evidence qatori. Exact completion rule — engine'da. Completion reward bo'lmasa ham yoziladi (masalan ceiling tugagan) — audit to'liq.

## 20. Weekly schedule

### LearningSchedulePreference (current, mutable; User domain bilan 1:1)

| Field | Izoh |
|---|---|
| user_id | unique |
| selected_weekdays | kichik to'plam (masalan bitmask/array) |
| target_sessions_per_week | int |
| preferred_session_minutes | int (30–360 oralig'i — product qoidasi) |
| updated_at | |

(`timezone` — UserProfile'da, dublikat qilinmaydi.)

### WeeklyProgress

| Field | Izoh |
|---|---|
| user_id + week_start_date | unique |
| target_sessions | int — **hafta boshida snapshot** (keyin schedule o'zgarsa o'sha haftaning adolati buzilmaydi) |
| completed_sessions | int — cached hisoblagich (haqiqat: LearningSession'lar) |
| status | IN_PROGRESS / MET / MISSED |

Schedule change history alohida jadval sifatida MVP'da YO'Q — WeeklyProgress snapshotlari tarixiy adolatni beradi; AI schedule takliflari LearnerRecommendation (SCHEDULE_ADJUSTMENT) orqali — bu o'zi minimal change iziga aylanadi.

## 21. Daily recap

**PROPOSED: alohida persisted DailyRecap entity YO'Q — derived.** Manbalar: LearningSession (vaqt), ActivityAttempt/progress (lesson/review soni), SkillMeasurement/State (skill o'sish), DailyPlan (missionlar), LearnerRecommendation (AI tavsiya); XP/IZL qiymatlari 1.2C ledgerlaridan keladi. Sabab: duplicate derived data — sync xavfi; underlying evidence immutable bo'lgani uchun tarixiy recap qayta chiqarsa bo'ladi. Agar keyin "o'sha kuni aynan ko'rsatilgan recap" (o'sha paytdagi formula bilan) product talabi bo'lsa — DailyPlan.context uslubida snapshot JSONB qo'shiladi (extension point, hozir emas).

## 22. Time tracking

- Model saqlaydi: session started/ended, **active_seconds aggregate**, last activity vaqtlari (attempt timestamplari orqali).
- **Raw heartbeat'lar forever saqlanMAYdi** — privacy + storage; ular qisqa muddatli operational telemetry (implementation detail, retention OPEN).
- Future fraud detection (reward uchun anti-cheat): session anomaliyalari LearnerSignal/SecurityEvent uslubidagi derived yozuvlar sifatida qo'shilishi mumkin — extension point, hozir modellashtirilmaydi.

**Generic LearningEvent jadvali (savol §31): YO'Q (TD-43).** Event sourcing qilinmaydi; domain jadvallar (attempts, responses, progress, sessions, changes) evidence'ni allaqachon to'liq qamraydi — generic event table duplicate data + egasiz ma'lumot bo'lardi. Product analytics stream kerak bo'lsa — analytics pipeline (provider OPEN), core DB emas.

## 23. AI context / feedback

**Context assembly (TD-31/D-14 davomi):** alohida "AI memory table" YO'Q. **Context Assembler** (application service) domain jadvallardan o'qiydi:

```
LearnerSkillState (current) + ACTIVE LearnerSignals + so'nggi N ActivityAttempt
+ joriy LearnerLessonProgress + ACTIVE Roadmap (+ so'nggi RoadmapChange'lar)
+ so'nggi LearnerRecommendation'lar  →  AI chaqiruv konteksti
```

LLM conversation memory source of truth emas — accepted principle saqlanadi.

**AI feedback storage ajratmasi (TD-43):**

- **Evaluation feedback** (speaking/writing/mistake analysis) — **persisted** (AiEvaluation) — evidence'ning qismi, learner tarixida ko'rinadi.
- **Conversational tutor xabarlari** — 1.2B core learning modeli EMAS. Suhbat persistence dizayni (UX history, moderation/safety, cost, privacy) — AI architecture bosqichida; retention policy OPEN. 1.2B faqat suhbatning **structured natijalarini** saqlaydi: LearnerRecommendation, AiEvaluation, LearnerSignal.

## 24. Idempotency

Double click / mobile retry uchun (TD-44):

- **Unique constraintlar guard sifatida:** (user, activity, attempt_no); (attempt, sequence_no); (user, local_date, generation_no); (user, week_start); (user, skill); (user, lesson).
- **State machine idempotency:** SUBMITTED response/attempt'ga qayta submit → no-op (xato emas, mavjud natija qaytadi); COMPLETED lesson'ni qayta complete → no-op; ACCEPTED recommendation'ni qayta accept → no-op.
- **client_request_id** (nullable) — submit yo'llarida retry dedup uchun yengil column (ActivityAttempt, AssessmentResponse'da); exact API semantikasi implementation'da.

Data modelga ta'siri: yuqoridagi unique'lar + status maydonlari yetarli — alohida idempotency jadvali kerak emas.

## 25. Statuses

| Entity | Statuslar |
|---|---|
| AssessmentAttempt | IN_PROGRESS / COMPLETED / ABANDONED |
| AssessmentResponse | PRESENTED / DRAFT / SUBMITTED / SKIPPED |
| ActivityAttempt | IN_PROGRESS / SUBMITTED / EVALUATED |
| AiEvaluation | PENDING / COMPLETED / FAILED |
| LearningSession | ACTIVE / ENDED |
| LearnerLessonProgress | IN_PROGRESS / COMPLETED |
| LearnerRoadmap | ACTIVE / COMPLETED / ARCHIVED |
| RoadmapItem | PENDING / IN_PROGRESS / COMPLETED / SKIPPED |
| LearnerRecommendation | PROPOSED / ACCEPTED / REJECTED / EXPIRED |
| LearnerSignal | ACTIVE / RESOLVED / EXPIRED |
| DailyPlan | CURRENT / SUPERSEDED |
| DailyPlanItem | PENDING / COMPLETED / SKIPPED |
| WeeklyProgress | IN_PROGRESS / MET / MISSED |

Draft writing javobi: DRAFT holatida mutable, SUBMITTED'dan keyin locked (§24, I-1).

## 26. IDs / deletion / indexes

- **IDs:** UUIDv7 (TD-23) — barcha yangi entitylar.
- **Timestamps:** created_at (+ updated_at mutable'larda); append-only jadvallarda faqat created_at.
- **Deletion (TD-30):** learning history (attempts, responses, measurements, changes, evaluations) **hech qachon destructive hard-delete qilinmaydi** — reward evidence, adaptive engine va checkpoint tarixi bunga tayanadi. User erasure → anonymization strategiyasi OPEN (mavjud OQ; learning history'ga ta'siri — user reference'larni anonimlashtirish, evidence strukturani saqlash yo'nalishida ko'riladi). Hard delete faqat: ABANDONED/expired attempt draftlari kabi hech narsa reference qilmaydigan disposable data (policy bilan).
- **Indexes (obvious, premature emas):** unique'lar (§24); (user_id, subject_id, status) — roadmap; (roadmap_id, position) — items; (user_id, started_at) — sessions, assessment attempts; (user_id, activity_id) va (user_id, submitted_at) — activity attempts; (user_id, skill_id, created_at) — measurements; (user_id, status) — signals, recommendations; (daily_plan_id, position) — plan items; (attempt_id, sequence_no) — responses.
- **Data volume (§41):** MVP'da partitioning YO'Q — to'g'ri indexlangan PostgreSQL o'n millionlab attempt qatorini bemalol ko'taradi; time-based partitioning/archival — kelajak extension point (ActivityAttempt, AssessmentResponse uchun), hozir qo'shilmaydi.

## 27. Entity Catalog

| Entity | Purpose | Mutable/Immutable | Key relations | Phase |
|---|---|---|---|---|
| AssessmentDefinition | Logical assessment identity | Mutable (lifecycle, current_version pointer) | Subject; 1:N Version; 1:N Item | NOW |
| AssessmentDefinitionVersion | Immutable config snapshot (TD-83) | Published → immutable | N:1 Definition; 1:N VersionItem; 1:N Attempt | NOW |
| AssessmentVersionItem | Version item-pool membership (TD-83) | Published version bilan immutable | unique(Version, Item) | NOW |
| AssessmentItem | Kalibrlangan test itemi (type+JSONB) | Published → immutable | N:1 Definition (logical); N:1 Skill | NOW |
| AssessmentAttempt | Learner test sessiyasi (+adaptive state) | Mutable → COMPLETED'da yopiladi | User, Definition, Checkpoint?; 1:N Response | NOW |
| AssessmentResponse | Taqdim etilgan item + javob | SUBMITTED → append-only | N:1 Attempt, Item; 0..N AiEvaluation | NOW |
| AiEvaluation | Async AI baholash (structured) | Append-only (status'dan tashqari) | Response XOR ActivityAttempt | NOW |
| LearnerSkillState | Joriy skill holati (engine yozadi) | Mutable derived | unique(User, Skill) | NOW |
| SkillMeasurement | Milestone skill snapshot | Append-only | User, Skill, source refs | NOW |
| LearningSession | Real learning davri (aggregate vaqt) | Mutable → ENDED | User; DailyPlan? | NOW |
| ActivityAttempt | Learner javob evidence'i | SUBMITTED → append-only | User, Activity, Revision, Session?, RoadmapItem? | NOW |
| ActivitySkill | Activity↔Skill mapping | Mutable (content) | Activity, Skill | NOW |
| LearnerLessonProgress | Lesson bo'yicha CURRENT state (pinned revision; relearn'da reset) | Mutable | unique(User, Lesson); Revision | NOW |
| LearnerLessonCompletion | Tugatishlar tarixi (TD-88) | **Append-only** | unique(User, Lesson, completion_no); Revision | NOW |
| DailyMissionCompletion | Mission bajarilganlik evidence'i (TD-84) | **Append-only** | unique(DailyPlanItem); 1:N Evidence | NOW |
| DailyMissionCompletionEvidence | Typed evidence qatorlari (XOR FK) | **Append-only** | N:1 Completion; Post/Attempt/Session | NOW |
| LearnerRoadmap | Personal plan instance | Mutable (status) | User, Subject, Track; 1:N Item | NOW |
| RoadmapItem | Plan elementi (typed) | Mutable (status) | Roadmap; Lesson?/Checkpoint?/Skill? | NOW |
| LearnerRecommendation | AI/system taklif + learner qarori | Content immutable, status mutable | User, Roadmap?; signal refs | NOW |
| RoadmapChange | Roadmap o'zgarishlari audit'i | Append-only | Roadmap, Recommendation? | NOW |
| LearnerSignal | Derived learning signal (mistake/weak/review) | Mutable lifecycle, recomputable | User, Subject, Skill?, Lesson? | NOW |
| Checkpoint | Methodist-configured checkpoint | Mutable (lifecycle) | Module (unique), AssessmentDefinition | NOW |
| DailyPlan | Kunlik versioned plan snapshot | Snapshot — supersedable, rewrite yo'q | unique(User, local_date, generation_no); 1:N Item | NOW |
| DailyPlanItem | Plan elementi (section+type) | Mutable (status) | Plan; RoadmapItem?/Lesson?/Skill? | NOW |
| LearningSchedulePreference | Joriy haftalik schedule | Mutable | unique(User) | NOW |
| WeeklyProgress | Haftalik target snapshot + natija | Mutable (hafta ichida) | unique(User, week) | NOW |
| XPTransaction, IZLLedger, RewardEligibility, DailyMission katalogi | Reward domain | — | Learning evidence'ni o'qiydi | NEXT (1.2C) |
| Subscription, Payment | Billing | — | — | NEXT (1.2C) |
| TutorConversation/Message | AI suhbat persistence | — | — | LATER (AI phase) |

## 28. Relationship Map

```
User
 ├──1:1── LearningSchedulePreference
 ├──1:N── WeeklyProgress
 ├──1:N── LearningSession ──N:1── DailyPlan?
 ├──1:N── AssessmentAttempt ──1:N── AssessmentResponse ──0..N── AiEvaluation
 ├──1:N── ActivityAttempt ──0..N── AiEvaluation
 ├──1:N── LearnerSkillState (unique per Skill)
 ├──1:N── SkillMeasurement (append-only)
 ├──1:N── LearnerLessonProgress (unique per Lesson; pinned LessonRevision)
 ├──1:N── LearnerSignal
 ├──1:N── LearnerRecommendation
 ├──1:N── LearnerRoadmap (bitta ACTIVE per Subject)
 └──1:N── DailyPlan (versioned per local_date; bitta CURRENT) ──1:N── DailyPlanItem

AssessmentDefinition ──1:N── AssessmentItem (skill_id, difficulty)
        ▲                          ▲
        │                          └── AssessmentResponse ──N:1── AssessmentAttempt
        └──1:1?── Checkpoint ──N:1── Module        (purpose=CHECKPOINT → checkpoint_id)

LearnerRoadmap ──1:N── RoadmapItem ──▶ Lesson (logical) / Checkpoint / Skill
        │                   ▲
        └──1:N── RoadmapChange ──N:1── LearnerRecommendation (ACCEPTED bo'lsa)

ActivityAttempt ──N:1── Activity ──N:1── LessonRevision   (content reference doim saqlanadi)
Activity ──N:M── Skill (ActivitySkill)
Speaking answer payload ──ref──▶ MediaAsset
```

## 29. Invariants

1. **L-1:** SUBMITTED AssessmentResponse/ActivityAttempt javobi hech qachon o'zgartirilmaydi (silently modified bo'lmaydi); DRAFT faqat submit'gacha mutable.
2. **L-2:** ActivityAttempt qaysi Activity + LessonRevision asosida bajarilganini doim saqlaydi; AssessmentResponse — qaysi (immutable) AssessmentItem asosida.
3. **L-3:** LearnerSkillState/SkillMeasurement faqat o'z Subject'iga tegishli Skill bilan ishlaydi (skill scope chalkashmaydi).
4. **L-4:** Roadmap recommendation orqali o'zgarganda RoadmapChange majburiy ACCEPTED LearnerRecommendation'ga bog'lanadi — user acceptance'siz recommendation-driven mutation yo'q.
5. **L-5:** COMPLETED LearnerLessonProgress o'z lesson_revision_id reference'ini abadiy saqlaydi.
6. **L-6:** Bitta (user, subject) uchun bir vaqtda ko'pi bilan bitta ACTIVE LearnerRoadmap.
7. **L-7:** Checkpoint har doim Methodist-configured AssessmentDefinition'ga ishora qiladi; checkpoint attempt = AssessmentAttempt(purpose=CHECKPOINT, checkpoint_id).
8. **L-8:** AiEvaluation roppa-rosa bitta parent'ga bog'lanadi (assessment_response XOR activity_attempt).
9. **L-9:** ActivitySkill faqat activity Subject'iga tegishli Skill bilan bo'ladi.
10. **L-10:** SkillMeasurement append-only — recompute eski measurementlarni o'chirmaydi/o'zgartirmaydi.
11. **L-11:** ARCHIVED content yangi RoadmapItem/DailyPlanItem sifatida tanlanmaydi (1.2A I-9 ijrosi).
12. **L-12:** attempt_no (user, activity) ichida unique va monotonic — "birinchi eligible completion" evidence'i yo'qolmaydi (1.2C mastery reward uchun).
13. **L-13:** DailyPlan (user, local_date, generation_no) unique; bir (user, local_date)da bitta CURRENT; SUPERSEDED snapshotlar rewrite/delete qilinmaydi — IZL eligibility (1.2C) auditi shu snapshotlarga tayanadi.
14. **L-14:** LearnerSignal evidence_refs faqat mavjud attempt evidence'ga ishora qiladi; signal recompute qilinganda evidence o'zgarmaydi.
15. **L-15:** Learning history jadvallari (attempts, responses, measurements, changes, evaluations) destructive hard-delete qilinmaydi; user erasure — anonymization yo'li bilan (policy OPEN).
16. **L-16 (TD-83):** Published AssessmentDefinitionVersion va uning AssessmentVersionItem membership qatorlari immutable; attempt har doim definition_version_id saqlaydi.
17. **L-17 (TD-84):** Har DailyMissionCompletionEvidence qatorida roppa-rosa bitta typed FK; completion daily_plan_item bo'yicha unique; completion/evidence append-only.
18. **L-18 (TD-88):** LearnerLessonCompletion append-only — relearn eski completion qatorlarini o'zgartirmaydi; progress reset faqat CURRENT state'ga tegadi.
19. **L-19 (TD-82):** Speaking javob media'si faqat response_media_asset_id FK orqali — answer JSONB'ida raw MediaAsset UUID saqlanmaydi.

## 30. Open questions

**Product qarori kerak (yangi):**

1. ~~Lesson version policy~~ — **TD-37 ACCEPTED** (pinning; silent migration yo'q; force-migration — future explicit mechanism, hozir implement qilinmaydi). Qolgan OPEN: tugatilgan lessonni qayta ko'rishda qaysi versiya ko'rsatiladi (display policy).
2. **Mistake taxonomy:** category_code registry'sini kim boshqaradi (Methodist? AI taklif qiladi?) va boshlang'ich ro'yxat.
3. **Signal lifecycle qoidalari:** REVIEW_DUE qachon paydo bo'ladi, signal qachon EXPIRED bo'ladi (product tuning).
4. **Reassessment:** learner qayta diagnostic topshira oladimi, qachon (purpose=REASSESSMENT modelda tayyor, policy yo'q).
5. **Daily Plan regeneratsiya chegaralari:** kuniga necha marta, kechagi bajarilmagan MUST_DO nima bo'ladi.

**Texnik/implementation (keyingi bosqichlarda):**

6. Skill display_level mapping — subject-level scale definition (English CEFR mapping data sifatida).
7. Adaptive engine config schema (AssessmentDefinition.config ichi) — scoring algorithm OPEN'ligiga bog'liq.
8. AiEvaluation re-evaluation siyosati (qachon qayta baholanadi, qaysi natija amal qiladi).
9. Tutor conversation persistence + retention — AI architecture bosqichi.
10. Heartbeat/anti-cheat telemetry retention — fraud design bilan (1.2C+).
11. ActivityAttempt/AssessmentResponse uchun kelajak partition/archive strategiyasi.
12. Anonymization'ning learning history'ga aniq ta'siri (mavjud OQ bilan birga).

**Mavjud OPEN bo'lib qolganlar (bu model ularni hal qilmaydi, faqat chegaralaydi):** assessment scoring algorithm; progress calculation formulalari; mastery threshold; estimated duration tizimi.

## 31. Recommended model summary

1. **Evidence + state (TD-31):** append-only evidence (attempts/responses/evaluations/measurements) + engine yozadigan mutable current state (skill state, progress, signals) — formulalar engine'da, recompute mumkin.
2. **Assessment (TD-32/33):** reusable AssessmentDefinition + immutable, kalibrlangan AssessmentItem (TD-21 falsafasi; lesson activity reuse rad etildi); adaptive holat = normalized Response'lar + attempt.engine_state JSONB + engine_version.
3. **AI evaluation (TD-34):** alohida async AiEvaluation entity — score/rubric/feedback structured, provider-neutral, response XOR attempt'ga bog'lanadi.
4. **Skill Profile (TD-35):** LearnerSkillState (mastery_score ichki + display_level derived, English hard-code'siz) + SkillMeasurement milestone tarixi (har mini savolda emas).
5. **Learning (TD-36):** ActivityAttempt faqat javobli bloklar uchun (JSONB answer, unique attempt_no, MediaAsset'li speaking); view-only bloklar LearnerLessonProgress ichida.
6. **Lesson progress (TD-37):** unique(user, lesson) + boshlashda pinned revision; versioning policy ACCEPTED (silent migration yo'q; force-migration — future explicit mechanism).
7. **Roadmap (TD-38/39):** LearnerRoadmap (bitta ACTIVE per subject) + typed RoadmapItem (logical Lesson reference); recommendation→acceptance→change zanjiri LearnerRecommendation + RoadmapChange bilan isbotlanadi; full versioning yo'q.
8. **Signals (TD-40):** bitta generic LearnerSignal — kichik type enum + moslashuvchan category_code; review tarixi mavjud zanjirdan derived.
9. **Checkpoint (TD-41):** Checkpoint entity → AssessmentDefinition; attempt = AssessmentAttempt purpose; oldin/keyin — SkillMeasurement'dan derived.
10. **Daily Plan (TD-42):** versioned/supersedable snapshotlar — (user, local_date, generation_no), bitta CURRENT, SUPERSEDED'lar rewrite qilinmaydi + itemlar generation'ga tegishli (1.2C IZL eligibility uchun audit-tayyor); recap — derived, saqlanmaydi.
11. **Session/vaqt (TD-43):** LearningSession aggregate active_seconds; heartbeat saqlanmaydi; generic LearningEvent jadvali yo'q; AI context — jadvallardan assembler, alohida memory table yo'q.
12. **Idempotency (TD-44):** unique constraintlar + state-machine no-op + optional client_request_id.
