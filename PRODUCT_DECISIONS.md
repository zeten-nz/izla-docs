# Izlan — Product Decision Log

> Status: Phase 0.2 (2026-08-19). Bu hujjat qabul qilingan product qarorlarining log'i.
> Conflict bo'lsa priority: accepted decisions > `loyiha.md` > inference.
> `OPEN QUESTION` deb belgilanganlar bu yerda yozilmaydi — ular [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)da.

## Product & scope

### D-01 — Personalized self-study platforma
- **Decision:** Izlan kurs katalogi emas; Learner'ning darajasi, maqsadi, kuchli/zaif tomonlari, vaqti va learning history'siga qarab individual experience yaratadi.
- **Why:** Tayyor kurslar shaxsiy maqsad va darajaga mos kelmaydi; personalization — platformaning asosiy differentsiatori.
- **Consequences:** Assessment, Skill Profile, Roadmap, Daily Plan va adaptive learning — core arxitektura elementlari bo'ladi.
- **Status:** ACCEPTED

### D-02 — MVP: Web; mobile kelajakda
- **Decision:** MVP faqat Web. Android va iOS — future.
- **Why:** Scope discipline; bitta platformada core loop'ni sifatli chiqarish.
- **Consequences:** API va arxitektura keyinchalik mobile client qo'shishga mos bo'lishi kerak. `loyiha.md`dagi "web + ilova (iOS/Android)" launch g'oyasi bosqichlarga bo'lindi (qarang S-04).
- **Status:** ACCEPTED

### D-03 — Birinchi Subject: English
- **Decision:** MVP English bilan boshlanadi; architecture multi-subject bo'lishi kerak.
- **Why:** Til o'rgatish — eng aniq demand; bitta fan bilan pedagogik model sinovdan o'tadi.
- **Consequences:** Content model, Skill model va assessment English'ga bog'lanmagan, umumiy bo'lishi kerak (masalan, Mathematics skill'lari boshqa).
- **Status:** ACCEPTED

## Tech direction

### D-04 — Boshlang'ich tech yo'nalish
- **Decision:** Frontend: Next.js + TypeScript. Backend: Node.js + TypeScript. Database: PostgreSQL. Architecture: Modular Monolith.
- **Why:** Team yo'nalishi; monolith greenfield uchun tezroq, modullarga bo'lish keyingi o'sishga tayyorlaydi.
- **Consequences:** Keyingi texnik tanlovlar [TECH_DECISIONS.md](TECH_DECISIONS.md)da yuritiladi. Phase 1.1 (2026-08-20)da NestJS (framework), Fastify (HTTP adapter), Prisma (ORM) va auth/session arxitekturasi qabul qilindi; SMS provider, storage, queue, AI provider, deployment, mobile stack, analytics, notification provider — hali ochiq.
- **Status:** ACCEPTED (partial — tanlovlar bosqichma-bosqich TECH_DECISIONS.md'da yopiladi)

## Content

### D-05 — Hybrid content model: Methodist + AI assistance
- **Decision:** Pedagogik struktura va sifat uchun Methodist javob beradi. AI lesson draft, example, exercise, test variantlari, explanation, adaptation kabi ishlarda yordam beradi. AI yaratgan content avtomatik publish qilinmaydi — human review majburiy.
- **Why:** Sifat nazorati va pedagogik ishonchlilik; AI'ning xato content chiqarish riskini yopish.
- **Consequences:** Content lifecycle (DRAFT→REVIEW→PUBLISHED→ARCHIVED) va Methodist tooling (kelajakda lesson builder) kerak bo'ladi.
- **Status:** ACCEPTED

### D-06 — Content hierarchy
- **Decision:** Subject → Track → Level → Module → Topic → Lesson → Activity/Content Block.
- **Why:** Roadmap, progress va checkpoint'lar uchun aniq struktura; Track learning maqsadini ifodalaydi (General English, IELTS, Speaking Focus, Travel/Hobby English).
- **Consequences:** Tracklar har Subject uchun bir xil bo'lishi shart emas. Batafsil: [CONTENT_MODEL.md](CONTENT_MODEL.md).
- **Status:** ACCEPTED

### D-07 — Skill model
- **Decision:** Hierarchy'dan tashqari Subject'ga xos Skill concept mavjud (English: Grammar, Vocabulary, Reading, Listening, Speaking, Writing, Pronunciation). Lesson bir yoki bir nechta Skill'ga ta'sir qiladi. Assessment va progress skill-level ishlaydi.
- **Why:** "English: B1" yorlig'i yetarli emas — skill'lar orasida farq katta bo'ladi.
- **Consequences:** Skill modeli har Subject uchun boshqacha bo'lishi mumkin (Mathematics: Arithmetic, Algebra, Geometry, Problem Solving, Logical Reasoning). Universal majburiy skill ro'yxati yo'q.
- **Status:** ACCEPTED

### D-08 — Lesson = modular content blocks
- **Decision:** Lesson bitta katta HTML/text blob emas; Text, Explanation, Image, Audio, Example, Mini Question, Practice, Speaking, Writing, Listening, AI Interaction, Mastery Test kabi bloklardan tuziladi. Video — lesson content uchun kelajakda mumkin.
- **Why:** Personalizatsiya, vaqt hisoblash (estimated duration) va lesson builder uchun granular struktura kerak.
- **Consequences:** Technical storage — ACCEPTED: TD-21/TD-22 ([DATA_MODEL_CORE.md](DATA_MODEL_CORE.md)). Methodist kelajakda lesson builder orqali bloklarni tartiblaydi.
- **Status:** ACCEPTED

### D-09 — Content lifecycle
- **Decision:** DRAFT → REVIEW → PUBLISHED → ARCHIVED.
- **Why:** AI-assisted content uchun human review nuqtasi shart.
- **Consequences:** Faqat PUBLISHED content Learner'ga va Roadmap engine'ga ko'rinadi. Lifecycle keyinchalik role/permission qarorlari bilan kengayishi mumkin.
- **Status:** ACCEPTED

## Assessment & learning

### D-10 — Adaptive Diagnostic Assessment
- **Decision:** Assessment "test → foiz → level" emas. Avval kontekst olinadi (learning goal, o'z bahosi, vaqt, qiynalayotgan joylari), keyin savollar Learner darajasiga adaptiv moslashadi (B1 yaxshi → B2 → qiynalsa chegara aniqlanadi).
- **Why:** Aniq boshlang'ich nuqta — personalizatsiyaning fundamenti.
- **Consequences:** Exact scoring algorithm hali final emas (OPEN). `loyiha.md`dagi "oddiy variantli daraja testi" modeli superseded (S-05).
- **Status:** ACCEPTED

### D-11 — Assessment natijasi = Skill Profile
- **Decision:** Natija bitta "English: B1" emas, skill-level profil (masalan Grammar B1, Reading B2, Listening A2 ...).
- **Why:** Roadmap va review'lar skill darajasida ishlashi kerak.
- **Consequences:** Progress tracking ham skill-level saqlanadi.
- **Status:** ACCEPTED

### D-12 — Deterministic vs AI evaluation boundary
- **Decision:** Aniq javobli testlarda (multiple choice, grammar, standard quiz, deterministic exercises) final scoring — backend deterministic logic. AI: writing/speaking evaluation, explanation, mistake analysis, personalized feedback, learning recommendation. AI evaluation natijalari iloji boricha structured (score, rubric, reasoning categories) saqlanadi.
- **Why:** Aniq javobli mashqda AI'ga scoring berish — keraksiz xarajat va nondeterminizm.
- **Consequences:** Assessment engine ikki yo'lni ajratadi. Exact implementation texnik bosqichda.
- **Status:** ACCEPTED

### D-13 — Roadmap modeli
- **Decision:** Roadmap 100% AI generatsiyasi emas. Model: **Approved Content + Pedagogical Rules + Assessment + AI Personalization**. Pipeline: Assessment → Skill Profile → Gap Detection → Learning Goal → Approved Content Pool → Roadmap Engine → AI Personalization → Recommendation. AI recommendation → Learner accepts → roadmap adjustment.
- **Why:** Pedagogik sifat kafolati; AI'ning yashirin o'zgartirishiga yo'l qo'ymaslik.
- **Consequences:** AI faqat PUBLISHED content va pedagogik qoidalar chegarasida ishlaydi.
- **Status:** ACCEPTED

### D-14 — Adaptive learning: structured learning state
- **Decision:** AI faqat joriy javobga emas, learning history'ga qaraydi (correct/wrong answers, repeated mistakes, weak/strong skills, lesson progress, mastery results, consistency, review history). AI "memory"si LLM conversation memory'ga tayanmaydi — structured learning state backendda saqlanadi.
- **Why:** Ishonchli, reproducible personalizatsiya; provider'ga bog'lanmaslik.
- **Consequences:** Learning event/state modeli texnik bosqichda loyihalanadi.
- **Status:** ACCEPTED

### D-15 — Checkpoint model
- **Decision:** Checkpoint qayerda bo'lishini Methodist pedagogik struktura asosida belgilaydi (Module A — 6 lesson → checkpoint; Module B — 10 lesson → checkpoint ...).
- **Why:** Universal "12 lesson" qoidasi turli fanlarga mos emas.
- **Consequences:** `loyiha.md`dagi "har 12 mavzudan keyin daraja testi" SUPERSEDED (S-01). "Oldin/keyin" progress ko'rsatish g'oyasi saqlanadi — endi checkpoint'larga bog'lanadi.
- **Status:** ACCEPTED

### D-16 — Weekly Learning Target schedule
- **Decision:** Learner onboardingda kunlar va session vaqtini tanlaydi, keyin o'zgartira oladi. Schedule jazo mexanizmi emas — asosiy model haftalik target (masalan 3 sessions/week; Wed o'rniga Sat bajarsa ham hisoblanadi).
- **Why:** Real hayotga mos moslashuvchanlik; streak-anxiety o'rniga barqaror odat.
- **Consequences:** `loyiha.md`dagi min 30 min / max 6 soat oralig'i saqlanadi; exact vaqt variantlari UI bosqichida aniqlanadi.
- **Status:** ACCEPTED

### D-17 — Time-based personalization
- **Decision:** Roadmap lesson soniga emas, Learner'ning available learning time'iga ham qaraydi (masalan 3 kun × 1.5 soat ≈ 18 soat/oy). Lesson/Activity'larda estimated duration concept bo'ladi.
- **Why:** Realistic plan — bitmaydigan roadmap motivatsiyani o'ldiradi.
- **Consequences:** Exact estimation system hali final emas (OPEN).
- **Status:** ACCEPTED

### D-18 — Daily Plan
- **Decision:** Daily Plan — kundalik learning workspace: **MUST DO** (roadmap asosiy learning), **RECOMMENDED** (weak areas/recent mistakes asosidagi review), **EXTRA** (library, speaking, community va h.k.). "Bugun qancha vaqtingiz bor?" quick adjustment bo'lishi mumkin. Lesson bir sessiyada tugashi majburiy emas — progress saqlanib davom ettiriladi.
- **Why:** "Lesson 4 ni bajaring" modeli personalization vision'iga zid.
- **Consequences:** Batafsil: [DAILY_PLAN.md](DAILY_PLAN.md).
- **Status:** ACCEPTED

### D-19 — Daily Recap
- **Decision:** Session/kun yakunida qisqa recap: vaqt, tugatilgan lesson/activity, mastery result, skill progress, XP, IZL, AI recommendation.
- **Why:** Progress ko'rinishi — motivatsiya va adaptive loop'ning kirishi.
- **Consequences:** Exact progress calculation hali final emas (OPEN).
- **Status:** ACCEPTED

### D-20 — Learning sequence
- **Decision:** Roadmapdagi prerequisite ketma-ketlik saqlanadi (Lesson 1'siz Lesson 3'ga o'tilmaydi). Library, Extra Practice, Review, Community kabi joylar ochiq. Learning'ning o'zi sun'iy limitlanmaydi.
- **Why:** Pedagogik izchillik + erkinlik balansi.
- **Consequences:** Sequence bypass anti-farming bilan ham bog'liq (D-27).
- **Status:** ACCEPTED

## AI

### D-21 — AI rollari va chegaralari
- **Decision:** AI kamida 4 rolda: (1) Roadmap Personalization, (2) Tutor/Feedback, (3) Speaking/Writing Evaluation, (4) Adaptive Learning recommendations. AI curriculum authority emas — verified content + pedagogical rules asosiy authority.
- **Why:** AI kuchli yordamchi, lekin pedagogik javobgarlik insonda.
- **Consequences:** Batafsil: [AI_SYSTEM.md](AI_SYSTEM.md). Provider/model strategy OPEN.
- **Status:** ACCEPTED

## Rewards

### D-22 — XP + IZL ajratilishi
- **Decision:** Ikki alohida tizim. **XP** — gamification (level, achievements, badges, titles, streak), real pul qiymati yo'q. **IZL** — real qiymatga ega reward currency.
- **Why:** Gamification cheklanmasligi kerak; real-value reward esa qat'iy nazorat talab qiladi.
- **Consequences:** `loyiha.md`dagi yagona "ball" tizimi SUPERSEDED (S-03).
- **Status:** ACCEPTED

### D-23 — IZL qiymati universal
- **Decision:** 1 IZL qiymati barcha foydalanuvchi va tariflarda bir xil. Tariflar farqi maximum earnable IZL, feature limits va boshqa imkoniyatlar orqali.
- **Why:** Currency ishonchliligi; tarifga qarab qiymat farqi — adolatsiz va chalkash.
- **Consequences:** Exact `1 IZL = X so'm` hali final emas (OPEN).
- **Status:** ACCEPTED

### D-24 — IZL reward ceiling: 20%
- **Decision:** Har subscription cycle uchun maksimal redeemable reward — subscription narxining 20%igacha (300,000 UZS → max 60,000 UZS equivalent IZL). Learner 20%ni avtomatik olmaydi — learning activity orqali ishlab topadi.
- **Why:** Iqtisodiy barqarorlik + haqiqiy learning'ni rag'batlantirish.
- **Consequences:** `loyiha.md`dagi "20% avtomatik ball qilib qaytariladi, 12 mavzuga bo'linadi" modeli SUPERSEDED (S-02).
- **Status:** ACCEPTED

### D-25 — IZL earning kategoriyalari
- **Decision:** (1) Real Learning Session (login-only reward yo'q), (2) Lesson Attention (mini practice'larni to'g'ri bajarish), (3) Mastery (masalan mastery testda 90%+; bir activity uchun bir marta), (4) Daily Missions (learning'ga foydali missionlar).
- **Why:** Reward faqat haqiqiy o'rganishga bog'lanadi.
- **Consequences:** Exact threshold/qiymatlar keyin tuning qilinadi (OPEN).
- **Status:** ACCEPTED

### D-26 — Learning unlimited, IZL controlled
- **Decision:** Learner istagancha o'qiy oladi (XP, progress, achievements cheklanmaydi). IZL uchun controlled eligibility: bir kunda faqat asosiy reward-eligible learning target IZL beradi. Cycle eligibility tugasa — learning davom etadi, IZL to'xtaydi.
- **Why:** O'rganishni cheklamaslik prinsipi + reward iqtisodiyotini himoya qilish.
- **Consequences:** Exact daily/weekly allowance keyingi economic design bosqichida (OPEN).
- **Status:** ACCEPTED

### D-27 — Anti-farming prinsiplari
- **Decision:** Login-only reward yo'q; bir testni qayta ishlab IZL farm qilib bo'lmaydi; completed reward activity qayta bajarilganda IZL bermasligi mumkin; sequence bypass orqali easy lesson farm yo'q; monthly/cycle ceiling; financial/reward actions audit qilinadi.
- **Why:** IZL real qiymatga ega — farming to'g'ridan-to'g'ri moliyaviy zarar.
- **Consequences:** Exact anti-fraud engine keyinchalik alohida design qilinadi.
- **Status:** ACCEPTED

## Subscription & payment

### D-28 — 3-tier subscription: START / PRO / MAX
- **Decision:** 3 tarif. START — bitta maqsad/fanga fokus; PRO — asosiy recommended plan; MAX — ko'proq fanlar va yuqoriroq usage/features. Farqlanish: available subjects, AI usage, speaking/writing evaluation usage, extra practice, advanced analytics va h.k. Core learning loop (Assessment → Roadmap → Lesson → Practice → Feedback → Progress) barcha tariflarda. Literal "unlimited AI" promise'dan ehtiyot bo'linadi.
- **Why:** Turli segmentlar; arzon tarifga ataylab yomon experience bermaslik prinsipi.
- **Consequences:** Exact nomlar, narxlar, feature matritsa — OPEN. `loyiha.md`dagi yagona oylik narx modeli SUPERSEDED (S-07).
- **Status:** ACCEPTED (3-tier model final; nomlar conceptual)

### D-29 — Payment providers: Click / Payme
- **Decision:** Rejalashtirilgan to'lov providerlari — Click va Payme.
- **Why:** O'zbekiston bozorining asosiy to'lov tizimlari.
- **Consequences:** Exact integratsiya va recurring payment modeli OPEN.
- **Status:** ACCEPTED (yo'nalish sifatida)

## Authentication, profile & roles

### D-30 — Registration majburiy, guest preview
- **Decision:** Learning uchun registration majburiy. Guest: landing, product explanation, feature preview, limited demo. Assessment, roadmap, progress, rewards, subscription, community participation — faqat account bilan.
- **Why:** Personalizatsiya account'siz ishlamaydi; guest preview — conversion uchun.
- **Status:** ACCEPTED

### D-31 — Primary identity: phone + SMS OTP
- **Decision:** Registration identity — telefon raqam, verification — SMS OTP. Email majburiy emas.
- **Why:** O'zbekistonda telefon — universal identifikator.
- **Consequences:** Session/token arxitektura Phase 1.1da ACCEPTED ([AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md), TD-08..TD-13 — short-lived access + rotating refresh + server-side sessions, passwordless). SMS provider va implementation detallari (JWT signing, CSRF shakli, TTL tuning) — OPEN.
- **Status:** ACCEPTED

### D-32 — date_of_birth va minors
- **Decision:** Onboardingda kamida tug'ilgan sana olinadi; age fixed value emas, `date_of_birth` asosida hisoblanadi. Minors bo'yicha privacy/community safety/payment-reward restrictions/parental consent launchdan oldin alohida huquqiy/product review talab qiladi — exact legal rules hozir invent qilinmaydi.
- **Why:** Auditoriyada 11–12 yoshlilar bor.
- **Status:** ACCEPTED

### D-33 — 4 role + permission-based authorization
- **Decision:** MVP rollari: LEARNER, METHODIST, MODERATOR, ADMIN. Authorization role nomlariga hard-coded emas — permission-based (ROLE → PERMISSIONS).
- **Why:** Kelajakda granular scope'lar (D-34) va yangi rollar uchun moslashuvchanlik.
- **Consequences:** Exact permission schema texnik bosqichda. Batafsil: [USER_ROLES.md](USER_ROLES.md).
- **Status:** ACCEPTED

### D-34 — Methodist subject assignment
- **Decision:** Methodist faqat o'ziga biriktirilgan Subject'lar doirasida ishlaydi. MVP uchun subject-level assignment yetarli; granular scope (English→Grammar) — future.
- **Why:** Content scope nazorati.
- **Status:** ACCEPTED

### D-35 — Admin sensitive actions: ledger/audit orqali
- **Decision:** Admin platformani boshqaradi, lekin sensitive financial actions unrestricted edit emas. Masalan IZL balance to'g'ridan-to'g'ri overwrite qilinmaydi — adjustment amount/reason/actor/timestamp bilan ledger/audit entry orqali. Audit qilinadigan harakatlar: IZL adjustment, payment correction, subscription change, user suspension, role/permission change, content publish/unpublish, moderation action, creator assignment.
- **Why:** Moliyaviy ishonchlilik va accountability.
- **Consequences:** Exact audit architecture OPEN (texnik bosqich).
- **Status:** ACCEPTED

## Community & announcements

### D-36 — Community maqsadi
- **Decision:** Community Twitter/Reddit clone emas — learning systemning qismi: "O'rgan → savol ber → tushuntir → boshqalarga yordam ber → bilimingni mustahkamla."
- **Why:** O'rgatish — o'rganilganni mustahkamlashning eng zo'r yo'li (`loyiha.md` g'oyasi saqlanib, aniqlashtirildi).
- **Status:** ACCEPTED

### D-37 — Post types
- **Decision:** Question, Learned, Explanation, Discussion, Other (fallback). Automatic classification — kelajakda mumkin, hozir shart emas.
- **Status:** ACCEPTED

### D-38 — Community media: text/image/audio, video yo'q
- **Decision:** Postlarda Text, Image, Audio. Video community uchun kerak emas. Audio language learning uchun ayniqsa muhim. Media binary data PostgreSQL'da saqlanishi shart emas — storage texnik bosqichda.
- **Consequences:** Exact media limits (count/size/duration) OPEN.
- **Status:** ACCEPTED

### D-39 — Community features (MVP concept)
- **Decision:** Feed, Subject/Topic association (masalan English #PresentPerfect #Grammar), Post, Replies, Reactions (learning-oriented: Helpful, Clear, Great Explanation — ro'yxat final emas), Accepted Answer, Report, Reputation.
- **Consequences:** Lesson ↔ Community integratsiya subject/topic linking orqali.
- **Status:** ACCEPTED

### D-40 — Reputation ≠ XP ≠ IZL
- **Decision:** Community contribution alohida Reputation conceptga ega. XP → learning gamification, IZL → economic reward, Reputation → community contribution. Community activity uchun asosiy reward: reputation, XP, title, badge. Real-value IZL community'da ehtiyotkorlik bilan (spam/farming riski).
- **Status:** ACCEPTED

### D-41 — Community safety (minors)
- **Decision:** MVP'da private DM yo'q; report mavjud; block capability ko'rib chiqiladi; moderation mavjud; phone/email public qilinmaydi; media moderation hisobga olinadi. Private messaging — future backlog.
- **Status:** ACCEPTED

### D-42 — Announcements vs Notifications
- **Decision:** Announcement (platform events, updates, competitions, votes, important information — admin yaratadi, platformada saqlanadi) va personal notification — conceptual jihatdan alohida tizimlar. Kerak bo'lsa announcementdan notification hosil qilinadi. Userni mayda actionlar bilan spam qilmaslik prinsipi. Notification channels OPEN.
- **Status:** ACCEPTED

## Scope discipline

### D-43 — MVP scope discipline
- **Decision:** Featurelar CORE MVP / IMPORTANT LATER / FUTURE deb ajratiladi. Accepted future ro'yxati: Android/iOS full app, private DM, community video, advanced social features, advanced follow/group systems, complex creator scope, vendor game currency integrations, advanced AI features, sophisticated notification segmentation.
- **Why:** Product katta — scope intizomisiz MVP chiqmaydi.
- **Status:** ACCEPTED

---

## Superseded decisions

`loyiha.md`dagi quyidagi g'oyalar yangi qarorlar bilan almashtirildi (tarix sifatida saqlanadi):

### S-01 — "Har 12 mavzudan keyin level test"
- **Eski:** 12 mavzudan keyin bitta daraja testi.
- **Yangi:** Checkpoint joylashuvini Methodist pedagogik struktura asosida belgilaydi (D-15).
- **Status:** SUPERSEDED

### S-02 — "Obunaning 20%i avtomatik ball qilib qaytariladi, 12 mavzuga bo'linadi"
- **Eski:** 20% avtomatik taqsimlanadi; misol: 500k obuna → 100k → 12 mavzu → mavzu boshiga ~8,300 so'm → 1 ball = 166 so'm.
- **Yangi:** 20% — bu **ceiling** (maksimal redeemable qiymat); Learner uni activity orqali ishlab topadi (D-24, D-25). 1 IZL qiymati hali final emas.
- **Status:** SUPERSEDED

### S-03 — Yagona "ball" currency
- **Eski:** Bitta ball tizimi ham gamification, ham pul qiymati vazifasini bajarardi.
- **Yangi:** XP (gamification, pul qiymati yo'q) + IZL (real qiymat) ajratildi (D-22).
- **Status:** SUPERSEDED

### S-04 — Launch'da web + iOS + Android
- **Eski:** Platforma web va ilova (iOS/Android) ko'rinishida bo'lishi kerak.
- **Yangi:** MVP — Web; mobile — future (D-02). Mobile g'oyasi bekor emas, bosqichga ko'chirildi.
- **Status:** SUPERSEDED (launch scope bo'yicha)

### S-05 — Daraja testi = oddiy variantli test
- **Eski:** Daraja aniqlash testi oddiy multiple-choice, to'g'ri/xato.
- **Yangi:** Adaptive Diagnostic Assessment + kontekst intake + Skill Profile (D-10, D-11).
- **Status:** SUPERSEDED

### S-06 — Community = X/Twitter/Reddit'ga o'xshash
- **Eski:** Umumiy social feed g'oyasi.
- **Yangi:** Learning-oriented community: post types, subject/topic linking, accepted answer, reputation (D-36..D-40). Asl "o'rgatish orqali mustahkamlash" maqsadi saqlanadi.
- **Status:** SUPERSEDED (format bo'yicha; maqsad saqlangan)

### S-07 — Yagona oylik obuna narxi
- **Eski:** Bitta oylik obuna, narxini admin qo'yadi.
- **Yangi:** 3-tier model: START / PRO / MAX (D-28). Narxlarni admin boshqarishi saqlanadi.
- **Status:** SUPERSEDED

### Legacy g'oyalar (superseded emas, lekin hali qaror ham emas)

- **Ball/IZL'ni pulga chiqarish (support ariza orqali):** `loyiha.md`da "recommend qilinmaydi, feedback sifatida qabul qilinadi" deb yozilgan. Yangi qarorlarda cash-out policy belgilanmagan — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)da.
- **O'yin valyutasiga almashtirish (Steam, PUBG, Brawl Stars, MLBB):** g'oya saqlanadi, lekin vendor integratsiyalari FUTURE (D-43). MVP redemption ro'yxati OPEN.
