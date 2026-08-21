# Izlan — Rewards: XP + IZL

> Reward tizimi ikki alohida qismdan iborat. Bog'liq qarorlar: D-22..D-27 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).
> Eski yagona "ball" modeli (1 ball = 166 so'm formulasi bilan) SUPERSEDED — qarang S-02, S-03.

## Ikki tizim

| | XP | IZL |
|---|---|---|
| Maqsad | Gamification | Real qiymatli reward currency |
| Pul qiymati | **Yo'q** | **Bor** (exact `1 IZL = X so'm` hali final emas) |
| Nimaga xizmat qiladi | Level, achievements, badges, titles, streak | Redeemable qiymat (keyingi obunaga chegirma va h.k.) |
| Cheklov | Learning davom etar ekan olinadi, cheklanmaydi | Controlled eligibility + cycle ceiling |

Community contribution uchun alohida **Reputation** concept bor — u XP ham, IZL ham emas ([COMMUNITY.md](COMMUNITY.md)).

## XP

- Learning davom etgani sari olinadi: lessonlar, practice, missionlar, izchillik.
- Level, achievements, badges, titles, streak kabi gamification elementlarini quvvatlaydi.
- Real pul qiymati yo'q — shuning uchun erkin va cheklovsiz berilishi mumkin.

## IZL

- Real qiymatga ega reward currency.
- **1 IZL qiymati barcha foydalanuvchi va tariflarda bir xil.** Qimmat tarifdagi Learner'ning 1 IZL'i arzon tarifdaginikidan qimmat bo'lmaydi. Tariflar farqi — maximum earnable IZL va feature limits orqali.
- Exact `1 IZL = X so'm` — OPEN (economic design bosqichi).

## IZL reward ceiling — 20%

Har subscription cycle uchun maksimal redeemable reward — **subscription narxining 20%igacha**:

```
Subscription = 300,000 UZS
Maximum reward value = 60,000 UZS equivalent IZL
```

Muhim: Learner 20%ni **avtomatik olmaydi**. Bu ceiling — u learning activity orqali ishlab topadi. (Eski "20% avtomatik qaytariladi" modeli superseded.)

## IZL earning kategoriyalari (accepted)

### 1. Real Learning Session
Faqat login qilib chiqib ketish reward bermaydi. Real learning activity talab qilinadi.

### 2. Lesson Attention
Theory/example'dan keyingi mini practice'larni to'g'ri bajarish.

### 3. Mastery
Lesson mastery testda yuqori natija (masalan 90%+ — threshold keyin tuning qilinadi).
Bir mastery reward bir activity uchun **bir martadan ortiq IZL bermaydi**.

### 4. Daily Missions
Masalan:
- library'da 15 min;
- speaking practice;
- weak topic review;
- bugun o'rgangan narsani community'da tushuntirish;
- boshqa learning missionlar.

Daily Missions imkon qadar learning'ga foydali bo'lishi kerak — sun'iy "click" missionlar emas.

## Learning unlimited — IZL controlled

Asosiy prinsip: **Learner'ning learning'i cheklanmaydi.**

- Bugungi asosiy lesson tugagach yana 4 lesson o'qimoqchimi — bemalol. XP, progress, achievements, badges, titles olaveradi.
- Lekin IZL uchun controlled eligibility:

> Bir kunda faqat asosiy reward-eligible learning target IZL olishi mumkin.

- Cycle IZL eligibility muddatidan oldin tugasa — learning davom etadi: **IZL to'xtaydi, XP va progress davom etadi.**
- Exact daily/weekly allowance — OPEN (economic design bosqichi).

## Anti-farming prinsiplari

IZL real qiymatga ega bo'lgani uchun:

- login-only reward yo'q;
- bir testni qayta-qayta ishlab IZL farm qilib bo'lmaydi;
- completed reward activity qayta bajarilganda IZL bermasligi mumkin;
- roadmap sequence bypass qilib easy lesson farm qilish mumkin emas;
- monthly/cycle reward ceiling mavjud;
- financial/reward actions audit qilinishi kerak.

Exact anti-fraud engine keyinchalik alohida design qilinadi.

## Ledger / audit talabi

- IZL balance to'g'ridan-to'g'ri overwrite qilinmaydi (hatto Admin tomonidan ham).
- Har qanday adjustment: **amount + reason + actor + timestamp** bilan ledger/audit entry orqali (D-35).
- Reward berilishi, redeem qilinishi va korreksiyalar — hammasi iz qoldiradi.

## IZL redemption (sarflash)

`loyiha.md`dan kelayotgan yo'nalishlar (final ro'yxat emas):

- **Keyingi obunaga chegirma** — asosiy ko'zda tutilgan yo'l.
- **O'yin valyutasi** (Steam, PUBG, Brawl Stars, MLBB kupyuralari) — g'oya saqlanadi, lekin vendor integratsiyalari **FUTURE** (D-43), MVP emas.
- **Pulga chiqarish (cash-out)** — `loyiha.md`da "support ariza orqali, recommend qilinmaydi" deb yozilgan; policy final emas — OPEN QUESTION.

MVP redemption ro'yxati hal qilinmagan — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).

## Unresolved economic questions

- `1 IZL = X UZS` qiymati.
- Daily/weekly IZL allowance (reward-eligible target hajmi).
- Har earning kategoriya uchun exact IZL qiymatlari.
- Mastery threshold (90%+ misol edi).
- IZL amal qilish muddati (expiry bormi?).
- MVP redemption ro'yxati va cash-out policy.
- Subscription bekor qilinganda / tarif o'zgarganda IZL taqdiri.

To'liq ro'yxat: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
