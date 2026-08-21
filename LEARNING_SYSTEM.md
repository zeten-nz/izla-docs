# Izlan — Learning System

> Learner'ning onboardingdan kundalik o'rganishgacha bo'lgan to'liq learning flow'i.
> Bog'liq qarorlar: D-10..D-20 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)). Daily Plan alohida batafsil: [DAILY_PLAN.md](DAILY_PLAN.md).

## Umumiy oqim

```
Onboarding (context intake)
→ Adaptive Assessment
→ Skill Profile
→ Roadmap (Track asosida)
→ Weekly schedule
→ Daily Plan  →  Lesson  →  Practice  →  Feedback  →  Progress
       ↑                                                  |
       └───────── adaptive learning (review, weak skills) ┘
```

## 1. Onboarding learning flow

1. Registration (phone + SMS OTP) va profil (kamida `date_of_birth`) — qarang [USER_ROLES.md](USER_ROLES.md).
2. Interaktiv "nimani o'rganishni istaysiz?" — Subject tanlash (MVP: English).
3. Track / learning goal tanlash (General English, IELTS, Speaking Focus, Travel/Hobby English).
4. **Context intake** — assessmentdan avval Learner'dan so'raladi:
   - learning goal;
   - o'zini qanday baholashi ("umuman bilmayman" / "boshlaganman" / ...);
   - qancha vaqt ajrata olishi;
   - nimada qiynalayotgani.
5. Adaptive Assessment (quyida).
6. Skill Profile natijasi ko'rsatiladi.
7. Shaxsiy Roadmap taklif qilinadi.
8. Weekly schedule tanlanadi.
9. Learning boshlanadi — birinchi Daily Plan hosil bo'ladi.

## 2. Adaptive Diagnostic Assessment

Assessment "test → foiz → level" modeli **emas**.

- Savollar Learner javoblariga qarab adaptiv moslashadi:

```
B1 savol → yaxshi javob → B2 savol → qiynalish → B1/B2 chegarasini aniqlash
```

- Context intake boshlang'ich nuqtani belgilashga yordam beradi ("umuman bilmayman" → eng boshidan; "B1'dayman deb o'ylayman" → B1 atrofidan boshlash).
- Aniq javobli savollar deterministic baholanadi; open-ended qismlar (writing/speaking bo'lsa) AI evaluation bilan — qarang [AI_SYSTEM.md](AI_SYSTEM.md).

**Exact scoring algorithm hali final emas** (OPEN QUESTION).

## 3. Skill Profile

Assessment natijasi bitta "English: B1" emas — skill-level profil:

```
Grammar     — B1
Vocabulary  — B1
Reading     — B2
Listening   — A2
Writing     — A2
Speaking    — A2
```

Skill Profile:

- Roadmap engine'ning asosiy kirishi;
- progress tracking'ning asosi (skill bo'yicha o'sish ko'rinadi);
- adaptive learning uchun weak/strong skills manbai.

## 4. Roadmap

Model: **Approved Content + Pedagogical Rules + Assessment + AI Personalization** (100% AI generatsiyasi emas).

Pipeline:

```
Assessment → Skill Profile → Gap Detection → Learning Goal
→ Approved Content Pool → Roadmap Engine → AI Personalization → Learner Recommendation
```

Qoidalar:

- AI faqat PUBLISHED content va Methodist belgilagan pedagogik qoidalar chegarasida ishlaydi.
- AI yordami: kerakli mavzular, ko'proq practice kerak bo'lgan skilllar, reviewlar, prioritization.
- AI Roadmap'ni Learner'dan yashirincha o'zgartirmaydi. Model:

```
AI recommendation → Learner accepts → roadmap adjustment
```

## 5. Learning sequence

- Roadmapdagi **prerequisite ketma-ketlik** saqlanadi: Lesson 1 tugatilmasdan Lesson 3'ga sakrab bo'lmaydi.
- Prerequisite talab qilmaydigan joylar ochiq: Library, Extra Practice, Review, Community.
- Learning'ning o'zi sun'iy limitlanmaydi — sequence pedagogik izchillik (va anti-farming) uchun.

## 6. Weekly schedule — Weekly Learning Target

- Onboardingda Learner kunlar (masalan Mon/Wed/Fri) va session davomiyligini tanlaydi (30 min, 1 hour, 1.5 hour, ...; umumiy oraliq: min 30 min — max 6 soat).
- Schedule'ni istalgan payt o'zgartirish mumkin.
- Schedule **jazo mexanizmi emas**. Asosiy model — haftalik target:

```
Target: 3 sessions/week
Mon ✓   Wed ✓   Fri ✗   Sat ✓
Natija: 3/3 completed
```

Wednesday darsini Saturday bajarsa ham weekly goal hisoblanadi.

## 7. Time-based personalization

Roadmap faqat lesson soniga emas, learning capacity'ga qaraydi:

```
3 kun × 1.5 soat ≈ 4.5 soat/hafta ≈ 18 soat/oy
```

Shu capacity asosida realistic plan tuziladi. Buning uchun har Lesson/Activity'da **estimated duration** bo'ladi:

```
Theory ~12 min + Examples ~5 min + Practice ~8 min + Speaking ~10 min + Mastery test ~10 min ≈ 45 min
```

Exact estimation system hali final emas (OPEN).

## 8. Daily Plan (qisqacha)

Har kun Learner uchun MUST DO / RECOMMENDED / EXTRA bo'limlaridan iborat kundalik workspace hosil bo'ladi. Vaqti kam bo'lsa "Bugun qancha vaqtingiz bor?" quick adjustment ishlaydi. Lesson bir sessiyada tugashi majburiy emas — progress saqlanadi.

Batafsil: [DAILY_PLAN.md](DAILY_PLAN.md).

## 9. Lesson flow

Lesson modular bloklardan iborat ([CONTENT_MODEL.md](CONTENT_MODEL.md)). Tipik oqim:

1. **Explanation** — nazariya (slide/matn + audio).
2. **Example** — misollar.
3. **Mini Question** — nazariya tushunilganini tekshiruvchi qisqa savollar ("Lesson Attention" — IZL earning kategoriyasi, qarang [REWARDS.md](REWARDS.md)).
4. **Practice / Speaking / Writing / Listening** — mashqlar:
   - to'g'ri bo'lsa — ijobiy feedback (maqtov);
   - xato bo'lsa — nima xato ekani va qanday tuzatish tushuntiriladi. O'rganish testlarida Learner'dan "nega bu variantni tanladingiz?" deb so'ralishi va uning fikri asosida xatosi tushuntirilishi mumkin (`loyiha.md` g'oyasi; AI tutor roli — [AI_SYSTEM.md](AI_SYSTEM.md)).
5. **Mastery Test** — lesson yakunida o'zlashtirish tekshiruvi (masalan 90%+ → mastery reward; threshold tuning qilinadi).
6. Lesson recap / progress yangilanishi.

Deterministic vs AI scoring chegarasi: aniq javobli bloklar — backend deterministic; open-ended (speaking/writing) — AI evaluation, structured natija bilan (D-12).

## 10. Checkpoints

- Checkpoint joylashuvini **Methodist** pedagogik struktura asosida belgilaydi:

```
Module A — 6 lesson  → checkpoint
Module B — 10 lesson → checkpoint
Module C — 14 lesson → checkpoint
```

- Universal "12 lesson" qoidasi yo'q (S-01, superseded).
- Checkpoint natijasi progress'da "oldin / keyin" taqqoslash sifatida ko'rinadi (`loyiha.md`ning saqlangan g'oyasi).

## 11. Progress

Progress ko'rinishlari:

- **Skill Profile dinamikasi** — har skill bo'yicha o'sish (masalan Grammar +3%).
- **Roadmap progress** — Module/Topic/Lesson bo'yicha bajarilganlik.
- **Checkpoint natijalari** — oldin/keyin.
- **Consistency** — weekly target bajarilishi, streak (XP tizimi bilan bog'liq).

Exact progress calculation formulalari final emas (OPEN).

## 12. Daily Recap

Session/kun yakunida qisqa recap:

```
Today:
68 min learning
1 main lesson
2 reviews
1 daily mission

Grammar +3%
Speaking +1%

+420 XP
+18 IZL

AI: "Present Perfect yaxshilandi, lekin since/for hali review talab qiladi."
```

Tarkibi: vaqt, tugatilgan lesson/activity, mastery result, skill progress, XP, IZL, AI recommendation.

## 13. Adaptive learning loop

Tizim kuzatadigan signallar (structured learning state, backendda — LLM conversation memory'ga tayanilmaydi):

- correct answers / wrong answers
- repeated mistakes
- weak skills / strong skills
- lesson progress
- mastery results
- learning consistency
- review history

Misol:

> Learner `Present Perfect` bo'yicha bir necha marta bir xil turdagi xatoni qaytardi.
> AI: "Bu mavzuda bir xil xato bir necha marta qaytaryapti. Balki bu qism yodingdan chiqqandir. Qisqa review qilib olamizmi?"

Adaptive recommendations Daily Plan'ning RECOMMENDED bo'limiga tushadi va Learner accept qilgandagina Roadmap'ga ta'sir qiladi (D-13).

## Open points (bu hujjat doirasida)

- Assessment scoring algorithm.
- Estimated duration estimation system.
- Progress calculation formulalari.
- Mastery threshold exact qiymati.

To'liq ro'yxat: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
