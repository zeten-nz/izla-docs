# Izlan — AI System

> AI'ning product'dagi roli va chegaralari. AI provider/model strategy TANLANMAGAN va bu hujjatda tanlanmaydi.
> Bog'liq qarorlar: D-05, D-10..D-14, D-21 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).

## Asosiy pozitsiya

> AI — kuchli yordamchi. **Curriculum authority emas.**
> Asosiy authority: verified (PUBLISHED) content + Methodist belgilagan pedagogical rules.

## AI rollari (kamida 4 ta)

### 1. Roadmap Personalization

Assessment natijasi (Skill Profile) va learning goal asosida:

- kerakli mavzular tanlovi va prioritization;
- ko'proq practice kerak bo'lgan skilllar;
- review takliflari.

Chegara: faqat Approved Content Pool ichida; o'zgarish faqat `AI recommendation → Learner accepts` orqali.

### 2. Tutor / Feedback

- Learner bilan gaplashish (learning support);
- xatolarni tushuntirish — jumladan Learner "nega bu variantni tanladim" deb yozgan fikri asosida qayerda adashganini ko'rsatish;
- to'g'ri javobni tushuntirish;
- maslahat berish.

### 3. Speaking / Writing Evaluation

Open-ended skill'larni baholashda yordam. Natija iloji boricha structured saqlanadi:

- score
- rubric
- reasoning/explanation categories

### 4. Adaptive Learning

Learner behavior va learning history asosida:

- review recommendation;
- extra practice recommendation;
- schedule adjustment recommendation;
- weak topic recommendation.

## Deterministic vs AI boundary

| Baholash turi | Kim final scoring authority |
|---|---|
| Multiple choice, grammar, standard quiz, deterministic exercises | **Backend deterministic scoring** |
| Writing, Speaking, open-ended javoblar | AI evaluation (structured natija bilan) |
| Explanation, mistake analysis, personalized feedback, recommendations | AI (scoring emas — support) |

Aniq javobli mashqlarda AI hech qachon final scoring authority bo'lmaydi.

## Structured learning state

AI'ning "memory"si LLM conversation memory'ga tayanmaydi. Backendda structured learning state saqlanadi:

- correct/wrong answers
- repeated mistakes (turi bilan)
- weak/strong skills
- lesson progress
- mastery results
- learning consistency
- review history

AI har chaqiruvda shu structured state'dan kontekst oladi. Bu provider'dan mustaqillik va reproducible personalizatsiya beradi. State modeli — ACCEPTED: [DATA_MODEL_LEARNING.md](DATA_MODEL_LEARNING.md) (TD-31 evidence+state, TD-43 Context Assembler — alohida "AI memory table" yo'q). AI provider/model strategy — OPEN bo'lib qoladi.

## AI content assistance (Methodist uchun)

AI Methodist'ga lesson draft, example, exercise, test variantlari, explanation va content adaptation'da yordam beradi. AI yaratgan content avtomatik publish qilinmaydi — har doim human review (content lifecycle: [CONTENT_MODEL.md](CONTENT_MODEL.md)).

## Guardrails (accepted principles)

1. AI curriculum authority emas — content va pedagogik qoidalar chegarasida ishlaydi.
2. AI Roadmap'ni Learner'dan yashirincha o'zgartirmaydi (recommendation → accept modeli).
3. Aniq javobli baholashda AI final scorer emas.
4. AI content avtomatik publish qilinmaydi.
5. AI usage tarifga qarab limitlanishi mumkin; literal "unlimited AI" promise berilmaydi ([SUBSCRIPTIONS.md](SUBSCRIPTIONS.md)).
6. Admin uchun AI usage/cost monitoring bo'lishi kerak ([USER_ROLES.md](USER_ROLES.md)).

## OPEN TECHNICAL CONCERNS

Quyidagilar hal qilinmagan va texnik arxitektura bosqichida ko'riladi:

- **Provider/model strategy** — qaysi provider(lar), qaysi model(lar), fallback strategiyasi.
- **Cost** — har Learner javobiga AI chaqiruvlar narxi; tarif limitlari bilan bog'lash; cost monitoring.
- **Latency** — practice feedback real-time bo'lishi kerak; speaking evaluation pipeline'ining kechikishi.
- **Quality** — o'zbek auditoriya konteksti: o'zbekcha tushuntirish sifati, o'zbek aksentli speech'ni baholash aniqligi.
- **Speech pipeline** — audio yozish, STT, pronunciation assessment (texnik yechim tanlanmagan).
- **Safety** — minors bilan muloqotda AI tutor xavfsizligi; moderation.
- **Consistency** — AI evaluation'ning barqarorligi (bir xil javobga bir xil baho).
- **Usage limits enforcement** — tarif bo'yicha AI limitlarini hisoblash va cheklash mexanikasi.

To'liq ro'yxat: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
