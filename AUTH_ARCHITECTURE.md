# Izlan — Authentication & Session Architecture

> Status: Phase 1.1 technical architecture (2026-08-20). Bu design hujjati — implementation EMAS.
> Bog'liq hujjatlar: [TECH_DECISIONS.md](TECH_DECISIONS.md), [USER_ROLES.md](USER_ROLES.md), [PRODUCT_DECISIONS.md](PRODUCT_DECISIONS.md) (D-30..D-33).
> Barcha aniq raqamlar (TTL, limitlar) — **tuning parameter**: qiymatlar tavsiya, final emas.

## 1. Goals

1. Phone + SMS OTP asosidagi passwordless authentication (product qarori D-31).
2. Web (Next.js) uchun ishlaydigan, kelajakda Android/iOS clientlarga o'zgarishsiz xizmat qiladigan token arxitekturasi.
3. Server tomondan boshqariladigan sessiyalar: logout, logout-all, suspension darhol yoki chegaralangan vaqt ichida kuchga kiradi.
4. OTP va refresh token'lar hech qachon plaintext saqlanmaydi.
5. Auth endpointlari abuse'ga (SMS bombing, brute-force) chidamli.
6. Authentication va Authorization aniq ajratilgan; authorization permission-based modelga tayyor.
7. SMS provider'ga hard-coded bog'liqlik yo'q.

**Anti-goal:** 30 kunlik long-lived JWT modeli ishlatilmaydi — revoke qilib bo'lmaydigan token bilan session boshqaruvi mumkin emas.

## 2. Accepted requirements (product hujjatlaridan)

- **Guest** — landing, platform overview, limited demo; account'siz. Assessment, roadmap, learning, progress, rewards, community participation, subscription — faqat registration bilan.
- **Primary identity:** telefon raqam. **Verification:** SMS OTP. Email majburiy emas.
- **Profile onboarding:** registrationdan keyin kamida name/display name va `date_of_birth`. Age fixed integer sifatida saqlanmaydi — `date_of_birth`dan hisoblanadi.
- **Minors:** mavjudligi hisobga olinadi; exact legal/parental-consent qoidalari product/legal review'da — bu hujjat ularni invent qilmaydi.
- Keyingi loginlar ham SMS OTP orqali; lekin har page refresh / har API request SMS talab qilmaydi — buni session/token modeli hal qiladi.

## 3. Authentication flow (umumiy model)

**Model: Short-lived Access Token + Rotating Refresh Token + Server-side Session/Device Records.**

```
Phone number
    ↓
Request OTP  ──────────────── rate limit, cooldown, abuse checks
    ↓
SMS verification (OTP)  ───── attempt limit, expiry, single-use
    ↓
Account lookup/create  ────── canonical phone bo'yicha
    ↓
AuthSession yaratiladi  ───── server-side record (device/client info bilan)
    ↓
Access token (short-lived) + Refresh token (rotating, session'ga bog'langan)
```

Kundalik ishlash:

- API requestlar **access token** bilan (~10–15 min TTL, tuning).
- Access token muddati tugasa client **refresh token** bilan yangi juftlik oladi (rotation, §8).
- Refresh ham tugagan/revoke qilingan bo'lsa — qayta SMS OTP login.

Shu tarzda SMS faqat login paytida ishlatiladi; sessiya davomida userga SMS kerak emas.

## 4. Registration flow (yangi telefon)

```
1. Guest telefon kiritadi → normalize (§14) → OTP request
2. OTP SMS yuboriladi (provider abstraction orqali, §20)
3. User OTP kiritadi → verify (§6)
4. Canonical phone bo'yicha user topilmadi → yangi User yaratiladi:
   - status: ACTIVE
   - onboarding: INCOMPLETE (§13 account states bilan aralashtirilmaydi)
5. AuthSession + tokenlar beriladi (user darhol authenticated)
6. Client onboarding'ga yo'naltiriladi: display name, date_of_birth, ...
7. Onboarding tugagach onboarding state COMPLETE bo'ladi
```

**Incomplete onboarding:** user onboardingni tugatmay chiqib ketsa, account va session saqlanadi; keyingi loginda onboarding davom etadi. Onboarding holati server-side saqlanadi (masalan `onboarding_completed_at` yoki step marker — exact shakl implementation bosqichida). Onboarding tugamagan userga core learning featurelari ochilmaydi (masalan assessment `date_of_birth`siz boshlanmaydi) — bu authorization darajasida "onboarding required" tekshiruvi bilan amalga oshadi, alohida account state bilan emas.

## 5. Login flow (mavjud telefon)

```
1. User telefon kiritadi → normalize → OTP request
2. OTP verify muvaffaqiyatli
3. Canonical phone bo'yicha user topildi → account state tekshiriladi (§13):
   - ACTIVE → yangi AuthSession + tokenlar
   - SUSPENDED → login rad etiladi (generic xabar), security event
   - DEACTIVATED → policy OPEN (§13)
4. Onboarding INCOMPLETE bo'lsa → onboarding'ga qaytariladi
```

Muhim: **account enumeration himoyasi** — OTP request bosqichida telefon ro'yxatdan o'tgan/o'tmaganligi response'da bilinmasligi kerak (ikkala holatda ham bir xil "OTP yuborildi" javobi). Mavjudlik faqat muvaffaqiyatli OTP verify'dan keyin ma'lum bo'ladi.

## 6. OTP lifecycle

### Challenge modeli

Har OTP so'rovi server-side **OtpChallenge** yozuvi bilan ifodalanadi (§18): phone, code hash, purpose (login / phone_change), expiry, attempt counter, consumed/invalidated holati.

### Qoidalar

| Qoida | Tavsiya (tuning) | Izoh |
|---|---|---|
| Code format | 6 raqam | UX/security balans |
| Expiration | 3 min (3–5 min oralig'i) | Muddati o'tgan challenge verify qilinmaydi |
| Single-use | ✅ | Muvaffaqiyatli verify → challenge `consumed`, qayta ishlatib bo'lmaydi |
| Resend cooldown | 60 s | Cooldown ichida yangi OTP so'rab bo'lmaydi |
| Resend → eski OTP invalidation | ✅ | Yangi challenge yaratilganda shu phone'ning oldingi active challenge'lari invalidated — bir vaqtda faqat bitta amal qiluvchi OTP |
| Max verification attempts | 5 / challenge | Limitdan keyin challenge invalidated; yangi OTP so'rash kerak |
| Per-phone limit | masalan 5 OTP/soat, 10/kun | SMS bombing + xarajat himoyasi |
| Per-IP limit | masalan 10 OTP request/soat | Ko'p raqamga bitta manbadan hujumni to'sish |

### Storage va hashing

- OTP code **hech qachon plaintext saqlanmaydi** va **hech qachon log qilinmaydi**.
- Saqlash: code'ning hash'i. 6 raqamli kod maydoni kichik (10^6) bo'lgani uchun oddiy hash offline brute-force'ga ojiz — shuning uchun **server-side secret (pepper) bilan HMAC** yoki argon2 tavsiya qilinadi. Qo'shimcha himoya baribir server-side attempt limit (5 ta urinish)dan keladi.
- Taqqoslash constant-time bo'lishi kerak.
- Verify muvaffaqiyatli → challenge darhol consumed; parallel takroriy verify ishlamaydi.

> **Phase 1.4B implementation (realized):** hashing sifatida **HMAC-SHA-256 + server pepper** tanlandi; 6-raqam `crypto.randomInt`; constant-time verify; atomik one-shot consume + atomik attempt increment; DB-backed per-phone cooldown/hourly/daily. Batafsil: [AUTH_FOUNDATION_IMPLEMENTATION.md](AUTH_FOUNDATION_IMPLEMENTATION.md). OPEN qolganlar (SMS provider, JWT signing/key, access transport, CSRF, recovery, phone-change, retention) — o'zgarishsiz.

### Provider failure handling

- SMS yuborish natijasi standartlashtirilgan: `SENT / TEMPORARY_FAILURE / PERMANENT_FAILURE` kabi semantika (conceptual).
- Temporary failure → cheklangan retry/backoff (provider abstraction ichida); userga neytral "qayta urinib ko'ring" xabari.
- Provider down bo'lsa — bu holat monitoring/alert hodisasi; auth logic o'zgarmaydi (§20).
- Failure responselari attacker'ga raqam mavjudligi haqida signal bermasligi kerak.

## 7. Access token model

- **Format:** imzolangan JWT (stateless verify uchun). Signing algoritm/key management — implementation bosqichida (key rotation concept hisobga olinadi).
- **TTL:** qisqa — 10–15 min (tuning).
- **Minimal claims:**

| Claim | Vazifa |
|---|---|
| `sub` | user id |
| `sid` | session id (AuthSession'ga bog'laydi) |
| `typ` | token turi (`access`) — refresh bilan chalkashmasligi uchun |
| `iat` / `exp` | issued/expiry |
| `ver` (optional) | auth/permission version — bump qilinsa eski tokenlar sensitive joylarda rad etiladi |

- Token ichiga user/profile object, to'liq permission ro'yxati **joylanmaydi**.
- **Role/permissions:** token permissionlarning yagona source of truth EMAS. Learner-level oddiy endpointlarda token claims yetarli bo'lishi mumkin; staff (METHODIST/MODERATOR/ADMIN) va sensitive amallarda guard permissionni server-side (DB/cache) tekshiradi — staff permission o'zgartirilganda eski token bilan ishlashda davom etib bo'lmaydi (§12, §18-revocation).

## 8. Refresh token rotation

- **Format:** opaque random token (masalan 256-bit), JWT emas.
- **Storage:** DB'da faqat **hash** (masalan SHA-256) — plaintext yo'q. Token qiymatini faqat client ko'radi.
- **Bog'lanish:** har refresh token bitta AuthSession'ga tegishli; session = token family.

### Rotation flow

```
Refresh token used
    ↓ (DB: hash bo'yicha topiladi, session ACTIVE, muddati o'tmagan, ishlatilmagan)
Old token marked used (invalidated)
    ↓
New refresh token issued (shu session ichida)
    +
New access token issued
```

### Reuse detection

- Ishlatilgan (used/revoked) refresh token qayta taqdim etilsa → **session compromised deb qaraladi**:
  1. Butun session/token family revoke qilinadi;
  2. `refresh_token_reuse_detected` security event yoziladi (§16);
  3. Ikkala tomon (legitimate user va attacker) qayta SMS login qilishga majbur bo'ladi.
- Shu detection ishlashi uchun ishlatilgan token hashlari session doirasida tarixiy saqlanadi (§18).

**Implementation note — parallel refresh (Phase 1.2D qo'shimchasi; qarorlar o'zgarmagan):** bitta userning ikki tab/so'rovi bir vaqtda refresh qilsa, ikkinchisi hozirgina rotate qilingan eski token bilan kelib reuse-detection'ni yolg'ondan (theft bo'lmasa ham) trigger qilishi mumkin. Implementatsiyada hisobga olinadi: rotation qat'iy atomic tranzaksiyada; client tomonda single-flight refresh tavsiya etiladi; ixtiyoriy juda qisqa grace strategiyasi (eski token soniyalar ichida qayta kelsa o'sha yangi juftlikni qaytarish) — security trade-off bilan; accidental-parallel vs theft farqlash policy'si — OPEN/implementation detail.

### Muddatlar (tuning)

- **Idle expiry:** refresh ~30 kun ishlatilmasa session tugaydi.
- **Absolute lifetime:** session maksimal ~90 kun — undan keyin qayta SMS login.
- Mobile clientlar uchun kelajakda boshqa qiymatlar bo'lishi mumkin — arxitektura buni session/client turi darajasida sozlashga qo'yadi.

## 9. Session / device model

Har login **AuthSession** yaratadi (server-side record):

| Maydon (conceptual) | Izoh |
|---|---|
| session id | Unique; access token `sid` claim shu yerga ishora qiladi |
| user | Egasi |
| client/platform | `web` / kelajakda `android` / `ios`; qo'pol client info (user-agent, app version) |
| created time | Login vaqti |
| last activity | Oxirgi refresh/foydalanish — idle expiry uchun |
| expiry | Absolute lifetime |
| revoked state | revoked_at + sabab (logout, reuse_detected, suspension, ...) |
| refresh-token lifecycle | Sessionga bog'langan token zanjiri (§8) |

**User imkoniyatlari (MVP):** current device'dan logout; all devices'dan logout.
**Future:** active sessions/devices ro'yxatini ko'rish, specific device revoke — model bunga tayyor (session recordlar allaqachon bor), faqat UI/endpoint keyin qo'shiladi.

**Device fingerprinting:** privacy-invasive yoki ishonchsiz fingerprinting ishlatilmaydi. Qo'pol client metadata (platforma, user-agent) faqat "Sessions" ro'yxatida userga ko'rsatish va security event kontekst uchun — unique tracking uchun emas.

## 10. Web token storage (Next.js)

### Taqqoslash

| Variant | XSS | CSRF | Izoh |
|---|---|---|---|
| **HttpOnly Secure SameSite cookie** | Token JS'dan o'qib bo'lmaydi — o'g'irlash sezilarli qiyinlashadi | Cookie avtomatik yuborilgani uchun CSRF himoya kerak (SameSite + qo'shimcha) | ✅ Tavsiya |
| localStorage | Har qanday XSS token'ni to'g'ridan-to'g'ri o'qiydi va exfiltrate qiladi | CSRF'ga chidamli (avtomatik yuborilmaydi) | ❌ Tavsiya etilmaydi |
| sessionStorage | localStorage bilan bir xil XSS zaifligi + tab yopilishi bilan yo'qoladi | — | ❌ Tavsiya etilmaydi |

### Recommended approach (final tavsiya)

- **Refresh token:** `HttpOnly; Secure; SameSite=Lax` cookie, **faqat refresh endpoint path'iga scope qilingan** — oddiy API requestlarda umuman yuborilmaydi.
- **Access token:** `HttpOnly; Secure; SameSite=Lax` cookie (yoki client memory'da — implementation bosqichida aniqlanadi; ikkalasida ham localStorage yo'q).
- **CSRF himoya:** SameSite=Lax + state-changing endpointlarda qo'shimcha himoya (custom header talabi yoki CSRF token — exact mexanizm implementation bosqichida). Refresh endpoint ham CSRF himoyasiz qolmaydi.
- **XSS haqida halol eslatma:** HttpOnly token *o'g'irlanishini* to'sadi, lekin XSS bor joyda attacker user brauzeri orqali requestlar yubora oladi. Shuning uchun cookie strategiyasi CSP, output escaping va dependency hygiene kabi umumiy XSS himoyasining o'rnini bosmaydi — to'ldiradi.

## 11. Future mobile compatibility

- Token **issuance** transport'dan ajratiladi: core auth logic "session yarat, token juftligini qaytar" bilan ishlaydi; cookie'ga o'rash — web adapter qatlamining ishi.
- Mobile clientlar (kelajakda) tokenlarni response body orqali oladi va OS secure storage'da saqlaydi (Android Keystore / iOS Keychain) — bu hujjat mobile'ni implement qilmaydi, faqat yo'lni ochiq qoldiradi.
- Session yaratishda client/platform turi qayd etiladi (§9) — per-platform lifetime/policy keyin sozlanishi mumkin.
- Hech bir flow browser-only mexanizmga (masalan faqat cookie'ga) qattiq bog'lanmaydi.

## 12. Authorization integration

Authentication ("kim?") va Authorization ("nimaga ruxsat?") ajratiladi.

- Rollar (product qarori D-33): LEARNER, METHODIST, MODERATOR, ADMIN.
- Model: `ROLE → PERMISSIONS` — permission-based. Exact permission ro'yxati hali final emas; authorization tizimi ro'yxat kengayishiga tayyor bo'ladi.

### NestJS'da conceptual ishlashi

- **AuthGuard (authentication):** access token'ni verify qiladi, principal (user id, session id, ...) request kontekstiga qo'shadi. Sensitive endpointlarda qo'shimcha server-side session/state tekshiruvi (§18).
- **PermissionsGuard + decorator (authorization):** endpoint deklarativ ravishda talab qilinadigan permission'ni e'lon qiladi (conceptual: `@RequirePermissions('content.publish')` uslubida); guard user'ning effective permissionlarini server-side manbadan (DB, kerak bo'lsa qisqa TTL'li cache) oladi va tekshiradi.
- `if (user.role === "ADMIN")` uslubidagi tarqoq hard-coded tekshiruvlar — anti-pattern; rol tekshiruvi faqat permission-mapping qatlamida yashaydi.
- **Staff permission o'zgarishi:** token permissionlarning yagona manbai emasligi uchun o'zgarish server-side darhol ta'sir qiladi (sensitive endpointlarda); qolgan joylarda ta'sir kechikishi access token TTL bilan chegaralangan (≤15 min) yoki `ver` bump orqali tezlashtiriladi.

## 13. Account states

Minimal model (overengineering'siz):

| State | Ma'no | Auth'ga ta'siri |
|---|---|---|
| **ACTIVE** | Normal account | Login/refresh ishlaydi |
| **SUSPENDED** | Staff tomonidan bloklangan | Login rad etiladi; barcha sessionlar revoke; refresh ishlamaydi; suspension audit qilinadi (D-35) |
| **DEACTIVATED** | Account faol emas (masalan user xohishi bilan) | Login/refresh ishlamaydi; qayta faollashtirish policy'si **OPEN QUESTION** (product qarori kerak — o'zimizcha final qilmaymiz) |

- **Onboarding holati account state EMAS** — ACTIVE account'ning alohida completion belgisi (§4). Sabab: onboarding tugamagan user ham authenticated bo'lishi kerak (aks holda onboardingni davom ettira olmaydi).
- State tekshiruvi ikki nuqtada: login/refresh paytida (majburiy) va sensitive endpointlarning server-side tekshiruvida. Oddiy endpointlarda suspended user'ning mavjud access tokeni maksimum TTL (~15 min) qadar yashashi mumkin — bu trade-off §18da.

## 14. Phone normalization

- **Canonical storage:** E.164 format — `+998XXXXXXXXX` (masalan `+998901234567`). Uniqueness shu canonical shakl bo'yicha.
- **Qabul qilinadigan inputlar:** `+998 90 123 45 67`, `998901234567`, `90 123 45 67`, turli separatorlar (bo'shliq, `-`, `(` `)`).
- **Normalization (backend, yagona joyda):**
  1. Raqam bo'lmagan belgilarni olib tashlash (`+`dan tashqari);
  2. `90...` (9 raqam, mahalliy) → `+998` prefiks qo'shish;
  3. `998...` → `+` qo'shish;
  4. Natija `+998` + 9 raqam shabloniga mosligini tekshirish; O'zbekiston mobil prefikslari bo'yicha validatsiya chuqurligi — implementation bosqichida.
- **Frontend:** faqat display formatting (masalan `+998 90 123 45 67`) va input yordamlari; **hech qachon o'zi canonicalization'ga tayanmaydi** — backend yagona authority.
- Kelajakda boshqa davlatlar kerak bo'lsa E.164 modeli o'zgarishsiz kengayadi (validatsiya qoidalari qo'shiladi).

## 15. Rate limiting

### Qatlamlar

| Endpoint | Per-IP | Per-identity (phone/session) | Izoh |
|---|---|---|---|
| Request OTP | ✅ qat'iy | ✅ phone bo'yicha (cooldown + soat/kun limitlari) | Eng qimmat endpoint (SMS xarajati) |
| Verify OTP | ✅ | ✅ challenge attempt limit (5) | Brute-force'ga qarshi asosiy himoya |
| Refresh | ✅ yumshoqroq | ✅ session bo'yicha anomal chastota | Reuse detection bilan birga ishlaydi |
| Login-related umumiy | ✅ | — | Umumiy abuse himoyasi |

- **Per-IP vs per-identity farqi:** per-IP bitta manbadan ko'p target'ga hujumni to'sadi (lekin NAT ortida ko'p legitimate user bo'lishi mumkin — juda qat'iy qilib bo'lmaydi); per-identity bitta accountga/raqamga taqsimlangan (ko'p IP'dan) hujumni to'sadi. Ikkalasi birga kerak.
- **Redis'siz MVP:** Redis hozir accepted dependency emas. OTP bilan bog'liq limitlar tabiiy ravishda DB-backed (OtpChallenge yozuvlari asosida hisoblanadi) — bu deterministic va instance sonidan mustaqil. Qolgan yengil limitlar uchun in-memory limiter yetarli (monolith dastlab bitta instance). 
- **Abstraction:** rate limiter conceptual interface ortida — keyin distributed limiting kerak bo'lsa Redis-backed implementatsiya auth logic'ni o'zgartirmasdan almashtiriladi.
- Escalation (masalan takroriy abuse'da CAPTCHA) — variant sifatida qayd etiladi, qaror OPEN.

## 16. Security logging (security events)

Financial audit ledger'dan (D-35) **alohida** security event tizimi:

Event turlari (kamida): `otp_requested`, `otp_verify_failed`, `otp_challenge_locked` (attempt limit), `login_success`, `session_created`, `refresh_token_reuse_detected`, `session_revoked`, `all_sessions_revoked`, `account_suspended`, `phone_change_requested/completed` (kelajak), `rate_limit_triggered`.

Yozuv tarkibi (conceptual): event type, vaqt, user id (bo'lsa), session id (bo'lsa), IP, qo'pol client info, minimal metadata.

**Qat'iy qoidalar:**

- OTP code va refresh token **hech qachon, hech qanday logga yozilmaydi** (hash ham shart emas).
- PII minimallashtiriladi: umumiy application loglarida telefon masked (`+99890*****67`); security event storage'da to'liq raqam saqlanishi kerakmi — retention/privacy policy bilan birga OPEN.
- Security eventlar monitoring/alerting manbai (masalan reuse detection spike, SMS xarajat anomaliyasi).
- Retention muddati — OPEN.

## 17. Recovery / change-phone concerns

### Account recovery — OPEN PRODUCT/SECURITY QUESTION

Telefon primary identity bo'lgani uchun "raqamga access yo'qoldi" — jiddiy holat. **Final policy bu hujjatda qabul qilinmaydi.** Xavflar va variantlar:

| Variant | Xavf/kamchilik |
|---|---|
| Support qo'lda raqam almashtiradi | Social engineering'ga eng ochiq yo'l; qat'iy verification protokolsiz xavfli — default deb qabul qilinmaydi |
| Optional email'ni recovery kanali qilish | Email majburiy emas (D-31) — hamma userda yo'q; email hygiene past bo'lishi mumkin |
| Hujjat (ID) verification | Og'ir jarayon, minors bilan murakkab, operatsion xarajat |
| Waiting period + account activity knowledge asosida progressive verification | Nisbatan balansli, lekin dizayn talab qiladi |
| Self-service recovery yo'q (yangi account) | Xavfsiz, lekin progress/subscription/IZL yo'qoladi — product uchun og'riqli |

Qo'shimcha real xavf: **O'zbekistonda operatorlar raqamlarni qayta muomalaga chiqaradi** (recycling) — eski egasining accountiga yangi egasi OTP bilan kira olishi mumkin. Bu recovery policy'dan tashqari, uzoq faol bo'lmagan accountlar uchun ham risk — mitigatsiya dizayni recovery policy bilan birga hal qilinishi kerak. **Status: OPEN QUESTION.**

### Phone number change (logged-in user) — high-risk action

Minimal baza: **current authenticated session + yangi raqamga OTP verification** (purpose: `phone_change` — login challenge'laridan ajratiladi).

Ko'rib chiqiladigan qo'shimcha steplar (final emas):

- Joriy (eski) raqamga ham OTP/xabar — eski raqam hali qo'lda bo'lsa;
- Fresh re-authentication talabi (session yaqinda yaratilgan bo'lishi);
- O'zgarishdan keyin barcha boshqa sessionlarni revoke qilish;
- Eski raqamga notification + cheklangan muddatli "bu men emasman" qaytarish imkoniyati;
- Cooldown (masalan tez-tez raqam almashtirishni cheklash).

Har holatda: `phone_change` security event + audit. **Exact policy: OPEN.**

## 18. Session revocation

### Holatlar matritsasi

| Trigger | Ta'sir |
|---|---|
| Logout (current) | Shu AuthSession revoked; uning refresh zanjiri o'ladi |
| Logout all | User'ning barcha sessionlari revoked |
| Account suspension | Barcha sessionlar revoked + login blok (§13) |
| Role/permission change (staff) | Sessionlar saqlanishi mumkin, lekin permissionlar server-side qayta o'qiladi (§12); kerak bo'lsa `ver` bump / forced re-auth |
| Refresh reuse detected | Shu session/family darhol revoked (§8) + security event |
| Phone number change | Tavsiya: boshqa barcha sessionlar revoked (policy OPEN, §17) |

### Access token trade-off (halol tushuntirish)

Access token stateless verify qilinadi — revoke qilingan sessionning access tokeni o'z TTL'i tugaguncha (~10–15 min) texnik jihatdan valid bo'lib qolishi mumkin. Balans:

- **Oddiy endpointlar:** faqat JWT verify — exposure oynasi TTL bilan chegaralangan (qisqa TTL shuning uchun muhim).
- **Sensitive endpointlar** (staff amallar, payment, phone change, session management): qo'shimcha server-side session/state tekshiruvi — revocation darhol ta'sir qiladi.
- **Refresh har doim DB'ga boradi** — revoked session hech qachon yangi token ololmaydi.

Bu hybrid model "har requestda DB check" (qimmat) va "faqat token" (revoke qilib bo'lmaydi) o'rtasidagi ongli kompromis.

## 19. NestJS module boundaries (conceptual)

```
AuthModule            — OTP flow, token issue/verify/rotate, AuthSession lifecycle
UsersModule           — User, profile, onboarding state, account states
AuthorizationModule   — role→permission mapping, PermissionsGuard, decoratorlar
SmsModule             — SMS provider abstraction + adapterlar (§20)
SecurityModule        — security events, rate limiter abstraction
```

Bog'lanishlar (conceptual):

- AuthModule → UsersModule (lookup/create), SmsModule (faqat abstraction orqali), SecurityModule (events/limits).
- AuthorizationModule AuthModule'dan mustaqil rivojlanadi (permission schema keyin kengayadi).
- User profile, SMS provider detali va permission tizimi **AuthModule ichiga tiqilmaydi** — separation of concerns.
- Bu papka/fayl strukturasi emas — implementation bosqichida moslashtiriladi.

## 20. Provider abstraction (SMS)

```
Auth Service → SMS abstraction (port) → Provider adapter(lar)
```

- Auth business logic faqat conceptual port bilan gaplashadi: "shu raqamga shu xabarni yubor" + standartlashtirilgan natija (yuborildi / vaqtinchalik xato / doimiy xato).
- Provider tanlanmagan (OPEN) — tanlov, almashtirish yoki multi-provider fallback auth logic'ni o'zgartirmaydi.
- Provider xarajat/delivery monitoring — SmsModule doirasida.
- Dev/test muhitda fake adapter (SMS yubormaydigan) ishlatilishi mumkin — bu ham xuddi shu port orqali.

## 21. Conceptual data model

Full schema EMAS — faqat entity'lar va vazifalari (minimal, evolvable):

| Entity | Vazifa | Asosiy conceptual maydonlar |
|---|---|---|
| **User** | Account + profil asosi | id, phone (canonical, unique), display name, date_of_birth, status (ACTIVE/SUSPENDED/DEACTIVATED), onboarding holati, timestamps |
| **AuthSession** | Server-side session/device record | id, user, platform/client info, created, last activity, expiry, revoked (vaqt+sabab) |
| **RefreshToken** | Rotation zanjiri + reuse detection tarixi | id, session, token hash, created, expires, used_at, revoked_at |
| **OtpChallenge** | OTP lifecycle | id, phone, purpose (login/phone_change), code hash, created, expires, attempt count, consumed/invalidated, request kontekst (IP) |
| **SecurityEvent** | Security logging (§16) | type, vaqt, user?, session?, IP, minimal metadata |

Dizayn izohlari:

- **Phone alohida UserIdentity jadvaliga chiqarilmaydi (MVP):** hozir yagona identity — telefon. Alohida identity jadvali kelajakda email/social login qo'shilsa evolyutsiya sifatida kiritiladi; hozir premature normalization bo'lardi.
- **RefreshToken alohida entity:** reuse detection uchun ishlatilgan tokenlar tarixi kerak — sessionda faqat "joriy hash" saqlash detection'ni yo'qotadi.
- **SecurityEvent financial audit ledger'dan alohida** (D-35 audit talablari bilan aralashtirilmaydi).
- Bularning hammasi Prisma schema YOZILMAGAN holda conceptual qoladi — implementation Phase'da aniqlanadi.

## 22. Threat model

| Tahdid | Mitigation (high-level) |
|---|---|
| OTP brute-force | 5 attempt/challenge → invalidation; qisqa TTL; hashed storage; verify rate limit |
| SMS bombing (raqamga spam / xarajat hujumi) | Per-phone cooldown + soat/kun limitlari; per-IP limitlar; xarajat monitoring/alert; escalation (CAPTCHA — OPEN) |
| Session theft (cookie o'g'irlash) | HttpOnly/Secure cookielar; qisqa access TTL; refresh rotation; reuse detection; logout-all |
| Refresh-token theft | Hash-only storage (DB leak'da token yo'q); rotation — o'g'irlangan token bir marta ishlaydi; reuse → family revoke |
| Token replay | Qisqa TTL; `typ` ajratish; refresh single-use; sensitive endpointlarda server-side session check |
| XSS | HttpOnly (token o'qilmaydi); localStorage'dan voz kechish; CSP/escaping umumiy himoya sifatida (§10) |
| CSRF | SameSite cookie + state-changing endpointlarda qo'shimcha CSRF himoya (§10) |
| Session fixation | Session faqat server tomonda, faqat OTP verify'dan keyin yaratiladi; login oldidan mavjud sessiya qayta ishlatilmaydi |
| Account enumeration | OTP request response'i mavjudlikni oshkor qilmaydi; xatolar generic; timing farqlarini minimallashtirish |
| Rate-limit bypass (distributed) | Per-identity limitlar IP'dan mustaqil ishlaydi; OTP limitlari DB-backed (instance'dan mustaqil); kelajakda Redis bilan distributed per-IP |
| Suspended user access | Login/refresh blok; sessionlar revoke; sensitive endpointlarda live state check; qolgan oynada TTL chegarasi (§18) |
| Admin/staff session compromise | Staff amallarda doimiy server-side permission+session check; audit (D-35); kelajakda qo'shimcha step-up himoya — OPEN |
| Phone recycling (operator raqamni qayta berishi) | OPEN — recovery policy bilan birga hal qilinadi (§17) |

## 23. Open questions

**Product qarori kerak:**

1. Account recovery final policy (§17) — variantlar berildi, qaror yo'q.
2. Phone change qo'shimcha security steplari (§17).
3. DEACTIVATED holati semantikasi (kim, qanday, qaytish yo'li).
4. CAPTCHA/escalation ishlatiladimi (UX savdosi).
5. Minors uchun parental consent auth flow'ga ta'sir qiladimi (legal review kutilmoqda, D-32).
6. Security event'larda to'liq telefon saqlash va retention muddati (privacy policy bilan).

**Texnik (implementation bosqichida):**

7. SMS provider tanlovi va fallback strategiyasi.
8. JWT signing algoritmi va key management/rotation.
9. Access token cookie'da vs client memory'da (web) — final tanlov.
10. CSRF mexanizmining aniq shakli.
11. Exact TTL/limit qiymatlari tuning (bu hujjatdagi barcha raqamlar — boshlang'ich tavsiya).
12. Cache strategiyasi (permission lookup uchun) — Redis'siz boshlanadi.

## 24. Recommended architecture summary

1. **Passwordless:** phone + SMS OTP; OTP hashed (peppered), single-use, attempt-limited, cooldown'li.
2. **Token model:** ~10–15 min access JWT (minimal claims: sub, sid, typ, iat/exp) + opaque rotating refresh token (hash-only storage) + server-side AuthSession recordlar. Long-lived JWT yo'q.
3. **Rotation + reuse detection:** har refresh'da yangi token; eski token reuse → session revoke + security event.
4. **Web storage:** HttpOnly Secure SameSite cookielar (refresh — path-scoped); localStorage ishlatilmaydi; CSRF himoya bilan.
5. **Mobile-ready:** token issuance transport-agnostic; cookie — web adapter detali.
6. **Authorization:** alohida qatlam, permission-based, NestJS guards + decoratorlar; token staff permissionlarning yagona manbai emas.
7. **Revocation:** logout / logout-all / suspension server-side; sensitive endpointlarda live check; oddiy endpointlarda exposure ≤ access TTL.
8. **Abuse himoyasi:** ko'p qatlamli rate limiting (per-IP + per-identity), Redis'siz MVP'da ishlaydi, keyin kengayadi.
9. **Boundaries:** Auth / Users / Authorization / Sms / Security modullari ajratilgan; SMS provider port ortida.
10. **Recovery — OPEN:** phone-loss policy ataylab qabul qilinmagan; variantlar va xavflar hujjatlashtirilgan.
