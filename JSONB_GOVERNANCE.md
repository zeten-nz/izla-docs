# Izlan — JSONB Governance

> Phase 1.2D deliverable — **TD-92 ACCEPTED** (owner review 2026-08-27). Schema writer va service-layer uchun JSONB intizomi.
> Umumiy qoida: **JSONB — content/config/state uchun; relational identity (FK bo'lishi kerak narsalar) — ustunlarda.** Owner qoidalari: (1) JSONB business-critical FK integrity'ning yagona source of truth'i bo'lmaydi; (2) canonical payloadlar strict validation; (3) publish/submit'dan keyin immutable JSONB mutate qilinmaydi; (4) kerakli joyda version context saqlanadi; (5) provider metadata provider-neutral modelni buzmaydi; (6) JSONB dumping ground emas — application-level reasonable size caps.

## Klassifikatsiya

| Field | Sinf | Strict validation | Version field | Relational ID ichida? | Immutable qachon | Query/index | Size |
|---|---|---|---|---|---|---|---|
| Activity.payload | A — canonical content | ✅ TD-22 (publishda strict) | Payload schema versiyasi — tavsiya | Media — faqat junction (TD-82 ACCEPTED); payload'da raw MediaAsset UUID YO'Q — faqat role/slot/config | Revision PUBLISHED bo'lgach | Kerak emas | Media by-reference — katta blob yo'q |
| AssessmentItem.payload | A | ✅ (TD-22 mexanizmi) | Tavsiya | Media — AssessmentItemMedia† | Item PUBLISHED bo'lgach | Kerak emas | — |
| AssessmentDefinitionVersion.config† | A/E — engine config | ✅ (engine schema) | version = row'ning o'zi | Yo'q | Yaratilgach immutable | Kerak emas | Kichik |
| ActivityAttempt.answer / AssessmentResponse.answer | A — evidence | ✅ (activity type bo'yicha) | Payload schema versiyasi bilan birga | response_media_asset_id† ustunda; answer ichida faqat display detali | SUBMITTED'dan keyin | Kerak emas | Audio by-reference |
| AiEvaluation.rubric | A — structured natija | ✅ (evaluation_version schema'si) | evaluation_version ustuni bor | Yo'q | Yaratilgach | Kerak emas | O'rtacha |
| AiEvaluation.provider_metadata | C — provider metadata | Yengil | — | Yo'q | Yaratilgach | Yo'q | Kichik — full prompt YO'Q |
| AssessmentAttempt.engine_state | E — engine state | Engine o'zi | engine_version ustuni bor | item id'lar bo'lishi mumkin — lekin truth Response qatorlarida | COMPLETED'da muzlaydi | Yo'q | Kichik |
| AssessmentAttempt.result_summary | B — display cache | Yengil | — | Yo'q | Write-once | Yo'q | Kichik |
| LearnerLessonProgress.completed_activities | E — progress state | Yengil (id ro'yxati) | — | Activity id'lar — truth attempts + shu ro'yxat (view-only bloklar uchun yagona manba) | COMPLETED'da | Yo'q | Kichik |
| LearnerSignal.evidence_refs | D — loose refs (ataylab) | Yengil | — | Attempt id'lar — ataylab loose (recomputable, pul emas) | Yo'q (signal recompute) | Yo'q | Chegaralangan ro'yxat |
| LearnerRecommendation.proposed_change / RoadmapChange.change_payload | A — qaror mazmuni | O'rtacha (change type bo'yicha) | change_type discriminator | Lesson/skill id'lar bo'lishi mumkin — apply paytida validatsiya | Yaratilgach (content) | Yo'q | Kichik |
| RoadmapItem.params / DailyPlanItem.params | D — type params | Yengil (type bo'yicha) | — | Asosiy FK'lar ustunlarda (lesson_id, skill_id...) — params'da FK YO'Q | Item snapshot qoidalari bilan | Yo'q | Kichik |
| DailyPlan.context | B/D — explainability snapshot | Yengil | — | Yo'q | Snapshot — yozilgach | Yo'q | Kichik |
| RewardPolicyVersion.config | A/E — iqtisod config | ✅ (policy schema) | version = row | Yo'q | Yaratilgach immutable | Yo'q | Kichik |
| XpGrant.source_refs | D — loose refs (ataylab) | Yengil | — | Ataylab loose — pul emas | Yaratilgach | Yo'q | Kichik |
| PaymentTransaction.provider_metadata | C | Yengil (sanitized!) | — | Yo'q | Yaratilgach | Yo'q | **Sanitized minimal — raw payload emas** |
| MediaAsset.moderation_metadata | C | Yengil | — | Yo'q | Automated yozadi, human status'ga ta'sir qilmaydi | Yo'q | Kichik |
| Notification.params | D — display params | Yengil | — | source_type/source_id ustunlarda | Yaratilgach | Yo'q | Kichik |
| StaffAudit.metadata† | D — audit detali | Yengil | — | target ustunlarda | Yaratilgach | Yo'q | Kichik |

## Qoidalar (TD-92)

1. **FK o'rnini bosmaydi:** relational integrity kerak bo'lgan har reference — ustun (TD-82 media misoli). JSONB ichidagi id — faqat display/state detali yoki ataylab-loose sinf (D).
2. **A-sinf (canonical) — har doim strict validation** (discriminated union), publish/submit'da qattiq; B/C/D — yengil shape check yetarli.
3. **Immutability sinf bilan keladi:** evidence/config JSONB'lari parent row immutable bo'lganda birga muzlaydi — alohida mexanizm kerak emas, lekin service qatlamida yozish taqiqlanadi.
4. **Schema versiyalash:** A-sinf payloadlarda `schema_version` maydonini payload ichida saqlash tavsiya etiladi — eski yozuvlarni migratsiyasiz o'qish uchun (exact format — implementation).
5. **Hajm intizomi:** media/katta matn — hech qachon JSONB ichida emas, by-reference; provider raw payload — biznes jadvalida emas (cheklangan-retention operational log).
6. **Index:** JSONB ustunlariga GIN/expression index faqat isbotlangan query ehtiyoji bilan qo'shiladi — hozircha birortasi kerak emas.
