# Izlan — Auth & Users Core Foundation (Phase 1.4B)

> Status: Phase 1.4B COMPLETE (2026-08-27). Auth **core primitives** — HTTP flow'siz.
> Kod: `backend/src/{users,auth,authorization,security,sms,bootstrap}`.
> Manba: [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md), [PRISMA_SCHEMA_V1.md](PRISMA_SCHEMA_V1.md), TD-08..TD-20.
> HTTP endpoint / AuthGuard / JWT / cookie / SMS integration — YO'Q (Phase 1.4C).

## 1. Scope

Implement qilingan: phone normalization, OTP generate/hash/lifecycle + DB-backed per-phone limits, session/refresh crypto + rotation + reuse detection, session revocation, SecurityEvent (append-only), authorization effective-permission lookup, system role bootstrap, SMS port, per-IP rate limiter foundation. Barchasi HTTP'dan mustaqil, DB integration + concurrency testlar bilan.

## 2. Module boundaries (TD-20)

```
src/
  users/           UsersRepository, UsersService, phone.util
  auth/
    otp/           OtpCodeService (crypto), OtpRepository, OtpService, otp-purpose
    sessions/      refresh-token.crypto, RefreshTokenRepository, AuthSessionRepository, SessionsService
    rate-limit/    InMemoryAuthRateLimiter (IP foundation)
    auth.module.ts
  authorization/   AuthorizationRepository, AuthorizationService, permission-registry
  security/        SecurityEventsRepository/Service, phone-mask.util
  sms/             sms.port (port only — production adapter YO'Q)
  bootstrap/       system-roles, seed-system entrypoint
  common/          errors (domain errors)
```

Boundary'lar: OtpService SMS provider'ni bilmaydi; SecurityModule StaffAudit'dan alohida; PrismaService faqat infrastructure. Generic `common/services` dumping ground yo'q.

## 3. Users foundation (§5/6)

`UsersService.createLearnerAfterVerifiedPhone(phone)` — OTP'ni O'ZI verify qilmaydi (contract: caller phone verification muvaffaqiyatiga mas'ul). Atomik `$transaction`: User + UserProfile + default LEARNER UserRole. Phone unique race-safe — P2002 → `DuplicatePhoneError` (silent duplicate yo'q). Boshqa rol avtomatik berilmaydi.

`assertAuthAllowed(userId)`: ACTIVE → allowed; SUSPENDED/DEACTIVATED → `AccountUnavailableError`. Onboarding holati auth'ga ta'sir qilmaydi (§37 — INCOMPLETE user ham session ola oladi). DEACTIVATED reactivation OPEN.

## 4. Phone normalization (§11, TD-16)

`normalizeUzPhone()` — bitta backend authority. Canonical `+998XXXXXXXXX`. Qabul: `+998...`, `998...`, mahalliy 9-raqam, separatorlar. Strukturaviy: `+998` + roppa-rosa 9 raqam. Rad: bo'sh, 8/10 raqam, `+997`, harf, malformed `+`, juda uzun. O'zgaruvchi operator-prefiks ro'yxati hard-code qilinmaydi. Xato: `PhoneInvalidError`; phone mavjudligi oshkor etilmaydi.

## 5–7. OTP generation / hashing / lifecycle

- **Generate (§14):** `crypto.randomInt` → 6 o'nlik raqam, leading zero saqlanadi (`004281`). Math.random/timestamp/counter YO'Q.
- **Hash (§15):** **HMAC-SHA-256 + server pepper**; input `HMAC(pepper, purpose \x1f canonicalPhone \x1f code)`. Verify — constant-time (`timingSafeEqual`, teng-uzunlik digest). Raw OTP hech qachon saqlanmaydi/log qilinmaydi; DB faqat `codeHash`.
- **Issue (§17):** normalize → cooldown → DB-backed hourly/daily limit → `$transaction`(invalidate prior active + create) → SecurityEvent `otp_requested` (masked phone). Return `{ challengeId, canonicalPhone, code (raw, SMS uchun), expiresAt }`. Account mavjudligini TEKSHIRMAYDI (§21 enumeration-safe).
- **Verify (§20):** har step atomik standalone commit (fail'da attempt increment transaction rollback bilan yo'qolmaydi). Tekshiruv: exists/context, not consumed/invalidated, not expired, attempts < max, constant-time hash. Fail → atomik `attemptCount++`; max'da `invalidatedAt` + `otp_challenge_locked` event. Success → atomik one-shot consume (`updateMany where consumedAt=null` → count=1); parallel'da faqat bittasi succeed.

**Tuning (implementation, product qarori emas):** TTL 180s, cooldown 60s, max 5, hourly 5, daily 10 — env orqali (`env.validation.ts`).

## 8. Rate limiting foundation (§22/23, TD-19)

- **Per-phone (DB-backed, implemented):** cooldown (latest challenge), hourly/daily (`countSince` OtpChallenge tarixidan). Cleanup'ga bog'liq emas — eski row'lar history sifatida qoladi.
- **Per-IP (foundation):** `InMemoryAuthRateLimiter` (sliding window). Redis YO'Q. **HTTP/IP binding → 1.4C** (real client IP endpoint bilan keladi).

## 9. SMS port (§18/19)

`SmsPort` interface + `SMS_PORT` token; natija `SENT | TEMPORARY_FAILURE | PERMANENT_FAILURE`. Production adapter (Eskiz/PlayMobile/Twilio) YO'Q; OTP'ni logga chiqaradigan dev adapter YO'Q. OTP challenge creation SMS yubormaydi (§18) — real yuborish 1.4C. Test'lar in-memory test double.

## 10–14. Sessions / refresh / rotation / reuse / revocation

- **Refresh crypto (§26):** 32 random bayt (256-bit), base64url opaque (JWT EMAS). DB faqat SHA-256 hash (yuqori entropiya → HMAC shart emas). Plaintext faqat caller'ga bir marta.
- **Session (§28):** `createSession` ACTIVE user talab qiladi (`assertAuthAllowed`); atomik AuthSession + initial RefreshToken. Idle TTL 30d (token), absolute TTL 90d (session). Access token YO'Q (§62). Invasive fingerprint yo'q — faqat platform/clientInfo.
- **Rotation (§30):** atomik `$transaction` + conditional `markUsedIfActive` (`updateMany where usedAt=null AND revokedAt=null` → count). count=1 → yangi token + link + session activity. count=0 (parallel) → strict reuse. **Bir eski token'dan IKKITA aktiv child CHIQMAYDI** (concurrency test bilan tasdiqlangan).
- **Reuse detection (§31):** used/revoked token qayta kelsa → butun session + token family revoke + `refresh_token_reuse_detected` event → `RefreshReuseDetectedError`. Unknown random token → `RefreshTokenInvalidError` (reuse EMAS, §32 — reuse event yaratilmaydi).
- **Concurrency (§33):** grace window YO'Q — strict TD-09/33. Parallel false-positive (transport single-flight, cross-tab) → **1.4C**.
- **Revocation (§34):** `revokeSession`/`revokeAllUserSessions` idempotent; row delete YO'Q (history). `session_revoked`/`all_sessions_revoked` event.

## 15. Security events (§24/25, TD-18/81)

Append-only `SecurityEventsRepository` (insert only). Kodlar: otp_requested, otp_verify_failed, otp_challenge_locked, session_created, session_revoked, all_sessions_revoked, refresh_token_reuse_detected, rate_limit_triggered. **StaffAudit'dan alohida** (§60). metadata'ga OTP/OTP-hash/refresh/refresh-hash/JWT/secret/DATABASE_URL HECH QACHON (§25). Pre-user OTP event'larda **masked phone** (`+99890*****67`); userId ma'lum bo'lganda userId. Faqat haqiqatan sodir bo'lgan hodisa yoziladi.

## 16. Authorization foundation (§9, TD-26)

`AuthorizationService.getEffectivePermissions(userId)`: User → UserRole → Role → RolePermission, ko'p rol union + dedup. `hasPermission`/`hasAllPermissions`. `if (role === 'ADMIN')` hard-code YO'Q. Cache YO'Q. Permission canonical kodlar `permission-registry.ts` (minimal — himoyalangan endpoint yo'q, 100 permission invent qilinmaydi); RolePermission mapping DB. **PermissionsGuard/RequirePermissions decorator YO'Q** (§10 — HTTP principal yo'q).

## 17. System role bootstrap (§7)

`bootstrapSystemRoles()` — idempotent `upsert` (LEARNER/METHODIST/MODERATOR/ADMIN). Faqat role identity — demo user/content/permission-matrix YO'Q. Entrypoint: `npm run db:seed:system`. Qayta ishga tushirish xavfsiz; role code destructive overwrite qilinmaydi.

## 18. Transaction boundaries (§39/40)

Pattern: **service transaction boundary'ni ochadi** (`PrismaService.$transaction`), **query'lar repository'da** (har method optional `tx?: Prisma.TransactionClient`). UnitOfWork/generic abstraction YO'Q. Atomik flow'lar: user+profile+role, issue(invalidate+create), session+token, rotation, reuse+revoke, logout-all. Verify'ning attempt/consume — standalone atomik operatsiya (rollback muammosiz).

## 19. Test database safety (§43/44/45)

Izolyatsiyalangan **izlan_test** (TEST_DATABASE_URL); `setup-integration.ts` DATABASE_URL override + NODE_ENV=test guard (izlan_test bo'lmasa throw). `cleanupAuthTables` avval `assertTestDatabase` (current_database='izlan_test' + NODE_ENV='test', aks holda FAIL) — keyin faqat auth jadvallar (child→parent), content/finance/community TEGILMAYDI. izlan_dev casual ishlatilmaydi. Global truncate helper yo'q.

## 20. Tests

| Turi | Soni | Qamrov |
|---|---|---|
| Unit | 43 (jami, +1.4A) | phone normalization (14), OTP crypto (8), refresh crypto (4), env auth (6), health, env |
| Integration | 23 (auth-core) + 2 (health) | role bootstrap idempotent, user creation + default role + duplicate, effective permissions union/dedup/removal, OTP lifecycle (issue/resend/cooldown/hourly/attempts/lock/expire/consume/reuse) + parallel verify, session create + SUSPENDED/DEACTIVATED deny + refresh hash-only + rotation + reuse→revoke + unknown token + revoked/expired session + idempotent revoke + logout-all + **concurrent rotation** + security event hygiene |

Concurrency: parallel OTP verify → 1 success; **concurrent refresh rotation → ≤1 new active child token** (strict reuse).

## 21. Deferred to Phase 1.4C

HTTP endpoint'lar (request-otp/verify-otp/refresh/logout), access JWT (algorithm/key management — HS256 vs asymmetric OPEN), cookie transport (access memory vs cookie), CSRF, AuthGuard/request principal, PermissionsGuard/RequirePermissions decorator, SMS provider adapter + real yuborish, IP rate-limit HTTP binding, refresh parallel-refresh grace/single-flight, onboarding routing response, phone-change flow, account recovery.

## 22. Known OPEN product/security policies

- SecurityEvent retention + pre-user full-phone vs masked policy (hozir masked; retention OPEN).
- Refresh parallel false-positive strategiyasi (1.4C grace).
- Account recovery / phone recycling / phone change / DEACTIVATED reactivation — product OPEN.
- JWT signing/key management, access transport, CSRF mexanizmi, SMS provider — texnik OPEN.
- CAPTCHA/escalation.
