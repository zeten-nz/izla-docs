# Izlan — Product Overview

> Status: Phase 0.2 product documentation (2026-08-19).
> Source of truth tartibi: (1) accepted product decisions — [PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md), (2) `loyiha.md`.
> Bu hujjat product hujjati. Texnik arxitektura qarorlari [TECH_DECISIONS.md](TECH_DECISIONS.md)da (stack va data architecture ACCEPTED); qolgan ochiq infra tanlovlari (storage, queue, AI provider va h.k.) — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).

## Vision

Izlan — oddiy kurs katalogi emas. Platformaning maqsadi:

> Foydalanuvchining hozirgi bilim darajasi, maqsadi, kuchli va zaif tomonlari, ajrata oladigan vaqti va learning history'siga qarab individual self-study experience yaratish.

Foydalanuvchi platformaga kirganda unga kurslar ro'yxati emas, "nimani o'rganishni istaysiz?" degan interaktiv boshlanish ochiladi. Undan keyin uning darajasi aniqlanadi, shaxsiy Roadmap tuziladi va har kuni unga moslashtirilgan Daily Plan beriladi.

## Problem

Mustaqil o'rganuvchining asosiy muammolari:

- nimadan boshlashni va qanday davom etishni bilmaslik;
- o'z darajasini aniq bilmaslik (umumiy "B1" yorlig'i skill'lar orasidagi farqni ko'rsatmaydi);
- tayyor kurslar shaxsiy maqsadga (IELTS, gapirish, hobby) mos kelmasligi;
- feedback yo'qligi — xato qilganda nima uchun xato ekanini hech kim tushuntirmaydi;
- motivatsiya va izchillikni ushlab turish qiyinligi.

Izlan bularni personalized roadmap, adaptive assessment, faol feedback, gamification (XP) va real qiymatli reward (IZL) orqali hal qiladi.

## Target users

- Yosh oralig'i: taxminan 11–12 yoshdan 40 yoshgacha.
- Auditoriyada voyaga yetmaganlar (minors) mavjud — bu privacy, community safety, payment/reward cheklovlari va parental consent bo'yicha launchdan oldin alohida huquqiy/product review talab qiladi (hal qilinmagan, qarang: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)).
- Yo'nalishlar: xorijiy tillar (ingliz, rus, yapon va h.k.) va amaliy/school-related fanlar (matematika, tarix, adabiyot, biologiya, kimyo va h.k.).

## Product principles

1. **Personalization over catalog.** Kurs katalogi emas — har bir Learner uchun individual traektoriya.
2. **Verified content — asosiy authority.** Pedagogik struktura va sifat uchun Methodist javob beradi. AI yordamchi, curriculum authority emas.
3. **AI recommendation → Learner accepts.** AI Roadmap va rejalarni Learner'dan yashirincha o'zgartirmaydi.
4. **Learning unlimited, reward controlled.** O'rganish hech qachon cheklanmaydi; faqat IZL earning'i nazorat qilinadi.
5. **Core learning loop barcha tariflarda.** Arzon tarifdagi Learner'ga ataylab yomon learning experience berilmaydi.
6. **Deterministic scoring where possible.** Aniq javobli mashqlarda AI final scoring authority emas.
7. **Community — learning'ning davomi.** "O'rgan → savol ber → tushuntir → yordam ber → mustahkamla."
8. **Minors safety.** Private DM yo'q (MVP), moderation bor, shaxsiy kontaktlar public qilinmaydi.
9. **Honest reward economics.** Login-only reward yo'q, farming yo'q, financial/reward harakatlar audit qilinadi.

## Core learning loop

```
Assessment → Skill Profile → Roadmap → Daily Plan → Lesson → Practice → Feedback → Progress
     ↑                                                                        |
     └──────────────── adaptive learning (review, weak skills) ←──────────────┘
```

Batafsil: [LEARNING_SYSTEM.md](LEARNING_SYSTEM.md).

## High-level modules

| Modul | Qisqacha | Hujjat |
|---|---|---|
| Authentication | Phone + SMS OTP registration, guest preview | [USER_ROLES.md](USER_ROLES.md) |
| User/Profile | Profil, date_of_birth, schedule, sozlamalar | [USER_ROLES.md](USER_ROLES.md) |
| Content | Subject→Track→Level→Module→Topic→Lesson→Activity, Methodist+AI hybrid | [CONTENT_MODEL.md](CONTENT_MODEL.md) |
| Assessment | Adaptive diagnostic assessment, Skill Profile | [LEARNING_SYSTEM.md](LEARNING_SYSTEM.md) |
| Roadmap | Approved content + pedagogical rules + AI personalization | [LEARNING_SYSTEM.md](LEARNING_SYSTEM.md) |
| Daily Plan | Kundalik learning workspace: Must Do / Recommended / Extra | [DAILY_PLAN.md](DAILY_PLAN.md) |
| Progress | Skill-level progress, checkpoints, daily recap | [LEARNING_SYSTEM.md](LEARNING_SYSTEM.md) |
| AI | Roadmap personalization, tutor/feedback, speaking/writing evaluation, adaptive learning | [AI_SYSTEM.md](AI_SYSTEM.md) |
| Rewards | XP (gamification) + IZL (real qiymatli currency) | [REWARDS.md](REWARDS.md) |
| Subscription | 3 tarif: START / PRO / MAX | [SUBSCRIPTIONS.md](SUBSCRIPTIONS.md) |
| Payments | Click / Payme yo'nalishi | [SUBSCRIPTIONS.md](SUBSCRIPTIONS.md) |
| Community | Learning-oriented postlar, javoblar, reputation | [COMMUNITY.md](COMMUNITY.md) |
| Announcements | Platforma e'lonlari, events, competitions | [ANNOUNCEMENTS_NOTIFICATIONS.md](ANNOUNCEMENTS_NOTIFICATIONS.md) |
| Notifications | Hali final emas — kelajakdagi personal notificationlar | [ANNOUNCEMENTS_NOTIFICATIONS.md](ANNOUNCEMENTS_NOTIFICATIONS.md) |
| Admin | Boshqaruv paneli, statistics, audit | [USER_ROLES.md](USER_ROLES.md) |

## MVP direction

- **Platforma:** Web (Android/iOS kelajakda).
- **Birinchi Subject:** English. Architecture boshqa fanlarni keyinchalik qo'shishga mos bo'lishi kerak.
- **MVP tarkibi (CORE):** phone+OTP auth, onboarding + adaptive assessment, Skill Profile, Roadmap, Daily Plan, lesson learning (block-based), deterministic + AI evaluation, progress, XP + IZL, 3-tier subscription, Click/Payme, learning-oriented Community (text/image/audio), announcements, admin/methodist/moderator asosiy funksiyalari.
- **Tech direction (tanlangan):** Next.js + TypeScript (frontend); Node.js + TypeScript + NestJS + Fastify (backend); PostgreSQL + Prisma; Modular Monolith. To'liq log: [TECH_DECISIONS.md](TECH_DECISIONS.md).
- Hali ochiq tanlovlar (SMS provider, object storage, queue, AI provider, deployment, mobile stack, analytics, notification provider): [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).

## Future direction (MVP'dan tashqarida, accepted)

- Android/iOS full app
- Private DM
- Community video
- Advanced social features (follow/group systems)
- Complex/granular creator scope (masalan English→Grammar darajasidagi assignment)
- O'yin valyutasi vendor integratsiyalari (Steam, PUBG va h.k.)
- Advanced AI features
- Sophisticated notification segmentation

Bu ro'yxatdagi narsalar MVP'ga qaytadan qo'shilmaydi.
