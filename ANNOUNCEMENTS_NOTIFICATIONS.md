# Izlan — Announcements & Notifications

> Bog'liq qaror: D-42 ([PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md)).
> Ikkalasi conceptual jihatdan **alohida** tizimlar.

## Announcement vs Notification

| | Announcement | Notification |
|---|---|---|
| Kim yaratadi | Admin | Tizim (event asosida) |
| Auditoriya | Umumiy (platforma foydalanuvchilari) | Shaxsiy (bitta user) |
| Qayerda | Platformada saqlanadi va ko'rsatiladi | Userga yetkaziladi |
| Holat | ACCEPTED (MVP) | To'liq final emas |

Kerak bo'lsa announcementdan notification hosil qilinishi mumkin (masalan muhim e'lon → userga bildirishnoma).

## Announcements

Admin platforma e'lonlarini yaratadi. Ishlatilish holatlari:

- platform events;
- updates / o'zgarishlar;
- competitions;
- votes;
- important information.

Misol:

> 12-sentabr Telegram kanalimizda online Zakovat bo'ladi. Qatnashib o'zingizni sinab ko'ring va sovrinlar yuting.

`loyiha.md`dagi "e'lonlar, vote'lar, zakovat baxslari" g'oyasi shu tizimga kiradi.

## Notifications (hali to'liq final emas)

Kelajakdagi ehtimoliy notificationlar:

- announcement (muhim e'lonlar);
- community reply (postiga javob kelganda);
- accepted answer (javobi qabul qilinganda);
- learning reminder (schedule asosida);
- subscription expiry;
- important AI recommendation.

## Anti-spam prinsipi

> Userni har bir mayda action bilan spam qilmaslik kerak.

Notification tizimi dizaynida "kam, lekin qimmatli" yondashuv — har reply/reaction uchun alohida push emas, muhim va foydali hodisalar uchun.

## SMS haqida eslatma

SMS registration verification (OTP) uchun **alohida** ishlatiladi ([USER_ROLES.md](USER_ROLES.md)) — bu notification tizimining qismi emas.

## Unresolved

- Delivery channels: in-app / push / SMS / email — tanlanmagan.
- Notification provider — tanlanmagan.
- MVP'da notificationning minimal to'plami qaysi bo'ladi.
- Learning reminder'ning aniq dizayni (qachon, qanchalik tez-tez).

To'liq ro'yxat: [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
