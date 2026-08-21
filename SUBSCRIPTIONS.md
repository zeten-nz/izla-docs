# Izlan — Subscriptions

> Bog'liq qarorlar: D-24, D-28, D-29 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).
> Exact narxlar bu hujjatda YOZILMAYDI — pricing hal qilinmagan.

## 3-tier model (ACCEPTED)

| Tier | Conceptual positioning |
|---|---|
| **START** | Bitta maqsad/fanga fokuslangan foydalanuvchi uchun |
| **PRO** | Asosiy **recommended** plan — ko'pchilik uchun ideal |
| **MAX** | Ko'proq fanlar va yuqoriroq usage/features |

- 3-tier model — final qaror. Exact nomlar (START/PRO/MAX) hali conceptual, o'zgarishi mumkin.
- PRO — marketing/UI'da recommended sifatida ko'rsatiladigan asosiy plan.

## Tariflar nimada farqlanadi

Farqlanish quyidagi o'qlar orqali bo'lishi mumkin:

- available subjects (nechta fan);
- AI usage miqdori;
- speaking evaluation usage;
- writing evaluation usage;
- extra practice hajmi;
- advanced analytics;
- boshqa advanced features;
- maximum earnable IZL ([REWARDS.md](REWARDS.md)).

## Muhim prinsiplar

1. **Arzon tarifdagi Learner'ga ataylab yomon learning experience berilmaydi.**
2. **Core learning loop barcha tariflarda saqlanadi:**

```
Assessment → Roadmap → Lesson → Practice → Feedback → Progress
```

3. AI usage tarifga qarab limitlanishi mumkin, lekin **literal "unlimited AI" promise berilmaydi**.
4. **1 IZL qiymati barcha tariflarda bir xil** — tarif farqi earnable miqdorda, qiymatda emas.

## 20% reward bog'liqligi

Har subscription cycle'da Learner o'z tarif narxining **20%igacha** IZL sifatida ishlab topishi mumkin (ceiling; avtomatik emas). Batafsil: [REWARDS.md](REWARDS.md).

Bu qoida tarif narxiga bog'langani uchun, qimmatroq tarif → yuqoriroq maximum earnable IZL (lekin 1 IZL qiymati o'zgarmaydi).

## Payment

- Planned providers: **Click** va **Payme**.
- Exact integratsiya va recurring payment modeli hali final qilinmagan (Click/Payme'da auto-renew mexanikasi texnik bosqichda o'rganiladi).
- Pricing'ni Admin boshqaradi ([USER_ROLES.md](USER_ROLES.md)); payment correction'lar audit bilan.

## Unresolved pricing/subscription questions

- Har tarif narxi (UZS).
- Feature matritsaning aniq mazmuni (qaysi tarifda nechta subject, qancha AI usage va h.k.).
- Recurring/auto-renew modeli Click/Payme bilan qanday ishlaydi.
- Trial yoki free preview darajasi (guest demo'dan tashqari) bo'ladimi.
- Obuna muddati tugaganda learning holati nima bo'ladi (grace period? read-only?).
- Tarif o'zgartirish (upgrade/downgrade) qoidalari va IZL/progress'ga ta'siri.
- Minors uchun to'lov oqimi (parental consent bilan bog'liq — huquqiy review).

To'liq ro'yxat: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
