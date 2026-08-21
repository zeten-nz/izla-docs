# Izlan — Content Model

> Bu conceptual product/content model — database schema EMAS. Texnik storage keyingi bosqichda hal qilinadi.
> Bog'liq qarorlar: D-05..D-09, D-34 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).

## Hierarchy

```
Subject → Track → Level → Module → Topic → Lesson → Activity / Content Block
```

| Daraja | Ta'rif |
|---|---|
| **Subject** | Fan: English, Mathematics, Japanese... MVP birinchi Subject — English. |
| **Track** | Learner'ning learning maqsadini ifodalovchi yo'nalish. Tracklar har Subject uchun bir xil bo'lishi shart emas. |
| **Level** | Track ichidagi daraja (English uchun masalan A1..C1 uslubida; boshqa fanlarda boshqacha bo'lishi mumkin). |
| **Module** | Level ichidagi mantiqiy bo'lim. Checkpoint'lar Module chegaralariga Methodist tomonidan qo'yiladi. |
| **Topic** | Module ichidagi mavzu. |
| **Lesson** | Topic ichidagi o'quv birligi — Learner'ning asosiy ish birligi. |
| **Activity / Content Block** | Lesson ichidagi modular blok (quyida). |

### English misoli

```
English (Subject)
└── General English (Track)          ← boshqa tracklar: IELTS, Speaking Focus, Travel/Hobby English
    └── B1 (Level)
        └── Module: Past & Present Connections
            └── Topic: Present Perfect
                └── Lesson: Present Perfect — since/for
                    ├── Explanation (text + audio)
                    ├── Example
                    ├── Mini Question
                    ├── Practice
                    ├── Speaking
                    └── Mastery Test
```

### Boshqa Subject misoli — Mathematics

```
Mathematics (Subject)
└── School Math Foundation (Track — misol, final emas)
    └── Level
        └── Module: Tenglamalar
            └── Topic: Chiziqli tenglamalar
                └── Lesson: Bir noma'lumli tenglamalar
                    ├── Explanation
                    ├── Example
                    ├── Practice
                    └── Mastery Test
```

Mathematics'da Speaking/Listening bloklari ma'nosiz — struktura fanga moslashadi. Hamma fanni English modeliga majburan tiqishtirish mumkin emas.

## Track

Track — Learner maqsadining ifodasi. English uchun accepted misollar:

- General English
- IELTS
- Speaking Focus
- Travel/Hobby English

Onboardingda Learner Subject tanlagach, unga Track variantlari ko'rsatiladi va assessment natijasi bilan birga shaxsiy Roadmap shu Track asosida quriladi.

## Skill model

Hierarchy'dan tashqari har Subject o'zining **Skill** to'plamiga ega:

| Subject | Skills |
|---|---|
| English | Grammar, Vocabulary, Reading, Listening, Speaking, Writing, Pronunciation |
| Mathematics (misol) | Arithmetic, Algebra, Geometry, Problem Solving, Logical Reasoning |

Qoidalar:

- Bitta Lesson bir yoki bir nechta Skill'ga ta'sir qiladi (masalan "Present Perfect" → Grammar + Speaking).
- Assessment natijasi skill-level **Skill Profile** hosil qiladi ([LEARNING_SYSTEM.md](LEARNING_SYSTEM.md)).
- Progress ham skill-level yuritiladi.
- Universal (barcha fanlar uchun majburiy) skill ro'yxati yo'q — har Subject o'ziniki.

## Lesson composition — Content Blocks

Lesson bitta katta HTML/text blob emas. U modular bloklardan tuziladi:

| Block | Izoh |
|---|---|
| Text | Oddiy matn |
| Explanation | Nazariy tushuntirish (slide uslubida, audio bilan bo'lishi mumkin) |
| Image | Rasm |
| Audio | Audio material (tillar uchun muhim) |
| Video | Lesson content uchun **kelajakda** mumkin (MVP emas) |
| Example | Misol |
| Mini Question | Nazariyadan keyingi qisqa tekshiruv savoli |
| Practice | Mashq |
| Speaking | Gapirish mashqi |
| Writing | Yozish mashqi |
| Listening | Tinglash mashqi |
| AI Interaction | AI tutor bilan interaktiv qism |
| Mastery Test | Lesson yakunidagi o'zlashtirish testi |

Qoidalar:

- Har blokda **estimated duration** concept bo'ladi (masalan Theory ~12 min, Practice ~8 min) — Daily Plan va time-based personalization uchun ([LEARNING_SYSTEM.md](LEARNING_SYSTEM.md)). Exact estimation system final emas.
- Methodist kelajakdagi **lesson builder** orqali bloklarni yaratadi va tartiblaydi.
- Technical storage — ACCEPTED: TD-21/TD-22 (bitta Activity jadvali + JSONB + strict validation), batafsil [DATA_MODEL_CORE.md](DATA_MODEL_CORE.md). Bu hujjat product/content semantikasini tasvirlaydi.

## Content lifecycle

```
DRAFT → REVIEW → PUBLISHED → ARCHIVED
```

- **DRAFT** — Methodist yoki AI assistance yaratgan xom content.
- **REVIEW** — human review bosqichi. AI yaratgan content **hech qachon avtomatik publish qilinmaydi**.
- **PUBLISHED** — Learner'ga ko'rinadigan va Roadmap engine ishlatadigan yagona holat.
- **ARCHIVED** — muomaladan chiqarilgan content.

Lifecycle keyinchalik batafsil role/permission qarorlari bilan kengayishi mumkin. Publish/unpublish harakatlari audit qilinadi (D-35).

## Hybrid creation model: Methodist + AI

**Methodist** — pedagogik struktura va sifat uchun asosiy javobgar (authority).

**AI** Methodist'ga yordam beradi:

- lesson draft
- example
- exercise
- test variantlari
- explanation
- content adaptation
- boshqa yordamchi content generation

AI hech qachon o'zi content publish qilmaydi va curriculum authority emas.

### Scope assignment

Methodist faqat o'ziga biriktirilgan Subject'lar bilan ishlaydi:

```
Methodist X:  English ✓   Japanese ✓   Mathematics ✗
```

MVP uchun subject-level assignment yetarli. Granular scope (English → Grammar, English → Speaking) — FUTURE.

## Open points (bu hujjat doirasida)

- ~~Lesson/block'larning exact technical storage formati~~ — hal qilindi: TD-21/TD-22 ([DATA_MODEL_CORE.md](DATA_MODEL_CORE.md)).
- Lesson builder UX/tafsilotlari — keyinroq.
- Estimated duration'ni aniqlash tizimi — final emas.
- Level nomenclature har Subject uchun (English CEFR'ga o'xshash, boshqa fanlarda nima?) — OPEN QUESTION.

To'liq ro'yxat: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
