# Izlan — Community

> Bog'liq qarorlar: D-36..D-41 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).

## Purpose

Community oddiy Twitter/Reddit clone **emas**. U learning systemning bir qismi:

> O'rgan → savol ber → tushuntir → boshqalarga yordam ber → bilimingni mustahkamla.

Asos: o'rgatish — o'rganilgan narsani mustahkamlashning eng zo'r yo'li (`loyiha.md`ning saqlangan g'oyasi).

## Post types (ACCEPTED)

1. **Question** — savol berish.
2. **Learned** — bugun o'rgangan narsasini ulashish.
3. **Explanation** — biror mavzuni boshqalarga tushuntirish.
4. **Discussion** — muhokama.
5. **Other** — fallback.

Kelajakda automatic classification/tavsiya bo'lishi mumkin — hozir shart emas.

## Media

| Media | MVP |
|---|---|
| Text | ✅ |
| Image | ✅ |
| Audio | ✅ (language learning uchun ayniqsa muhim — talaffuz, gapirish namunalari) |
| Video | ❌ **kerak emas** (community uchun) |

- Exact media limits (image count/size, audio duration/size) — OPEN.
- Media binary data PostgreSQL'da saqlanishi shart emas; storage texnik bosqichda.
- Media moderation hisobga olinadi.

## Subject/Topic linking

Postlar Subject/Topic bilan bog'lanishi mumkin:

```
English   #PresentPerfect   #Grammar
```

Bu lesson ↔ community integratsiyasi uchun ishlatiladi: masalan lesson sahifasidan shu Topic'ga oid postlarga chiqish, yoki "bugun o'rgangan narsangni community'da tushuntir" daily mission'i ([DAILY_PLAN.md](DAILY_PLAN.md)).

## Features (MVP concept)

- **Feed**
- **Post** (types + media yuqorida)
- **Replies** — javoblar
- **Reactions** — generic emas, learning-oriented: Helpful, Clear, Great Explanation (exact ro'yxat final emas)
- **Accepted Answer** — savol egasi eng yaxshi javobni belgilaydi
- **Report** — foydalanuvchilar muammoli contentni report qiladi
- **Reputation** — community contribution ko'rsatkichi

## Reputation

Uch tizim aralashtirilmaydi:

```
XP         → learning gamification
IZL        → economic reward
Reputation → community contribution
```

- Community activity uchun asosiy reward: **reputation, XP, title, badge**.
- Real-value IZL community'da ehtiyotkorlik bilan — spam/farming riski bor. (Daily mission orqali "o'rganganini tushuntirish" IZL berishi mumkin, lekin bu mission tizimi orqali nazorat qilinadi, ochiq community farming emas.)
- Exact reputation mexanikasi — OPEN.

## Moderation

- **Moderator** roli: reported posts/replies, spam, inappropriate content, moderation actions ([USER_ROLES.md](USER_ROLES.md)).
- Moderation actions audit qilinadi (D-35).
- Moderation policy/guidelines — OPEN (MVP'dan oldin kerak).

## Minor safety (MVP)

Auditoriyada voyaga yetmaganlar bor, shuning uchun:

- **Private DM yo'q** (future backlog);
- report mavjud;
- block capability ko'rib chiqiladi;
- moderation mavjud;
- phone/email public qilinmaydi;
- media moderation hisobga olinadi.

## Future (MVP emas)

- Private DM
- Video content
- Advanced social features (follow/group systems)
- Automatic post classification

Open savollar: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
