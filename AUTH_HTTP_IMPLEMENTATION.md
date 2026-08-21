# Izlan — Authentication HTTP Flow + Access JWT + Web Security (Phase 1.4C)

> Status: Phase 1.4C COMPLETE (2026-08-27). Auth core (1.4B) HTTP adapter bilan expose qilindi.
> Kod: `backend/src/auth/http`, `access-token`, `sms/adapters`. Manba: [AUTH_FOUNDATION_IMPLEMENTATION.md](AUTH_FOUNDATION_IMPLEMENTATION.md), [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md).
> **Production SMS delivery: NOT CONFIGURED** (§59 — provider tanlanmagan; HTTP arxitektura port + unavailable/test adapter bilan COMPLETE).

## 1. Owner decisions (implemented)

| | Qaror |
|---|---|
| Access token signing | **RS256** (asymmetric; public key verify; kid rotation-ready; Passport yo'q) |
| Access transport (web) | Response body → frontend **memory only** (localStorage/cookie YO'Q); `Authorization: Bearer` |
| Refresh transport (web) | **HttpOnly cookie** `izlan_refresh`, SameSite=Lax, Path=/api/auth/refresh, Domain omitted, Secure (prod) |
| CSRF | Header/origin-based: SameSite=Lax + custom `X-Izlan-CSRF: 1` + exact Origin allowlist + `Sec-Fetch-Site: cross-site` rad + credentialed CORS |
| Logout | Bearer access token talab qiladi; refresh cookie'ni exact path bilan tozalaydi |

## 2–4. JWT

- **Algorithm:** RS256 (`jsonwebtoken`; @nestjs/jwt/Passport yo'q — kid map uchun to'g'ridan nazorat).
- **Claims (minimal, §7):** `sub`=userId, `sid`=sessionId, `typ`=access, `iss`, `aud`, `iat`, `exp`. Phone/role/permission/IZL/DOB YO'Q.
- **TTL:** 900s (env `AUTH_ACCESS_TTL_SECONDS`).
- **kid:** har token header'ida `AUTH_JWT_ACTIVE_KID`; verifier `kid → public key` map'dan tanlaydi (unknown kid → reject).
- **Keys:** env base64 PEM — `AUTH_JWT_ACTIVE_KID`, `AUTH_JWT_PRIVATE_KEY_B64` (PKCS8), `AUTH_JWT_PUBLIC_KEYS_JSON` (`{kid: base64 SPKI}`). Startup'da decode+parse+RSA-validate (fail-fast); qiymatlar log qilinmaydi. Active kid public map'da bo'lishi shart.

## 5–6. Keys & rotation

- **Generation:** `npm run generate:jwt-keys [kid]` (RSA 3072, stdout env satrlar). Startup'da avtomatik key YARATILMAYDI. Private key hech qachon commit qilinmaydi; test keys — `test/fixtures/jwt-test-keys.ts` (test-only).
- **Rotation (§62, additive-ready):** `AUTH_JWT_PUBLIC_KEYS_JSON` bir necha kid saqlaydi. Pattern: (1) yangi public key qo'shish → deploy (verifier old+new); (2) `AUTH_JWT_ACTIVE_KID` + private key yangisiga o'tkazish; (3) max access TTL kutish; (4) eski public key olib tashlash. DB key jadvali YO'Q.

## 7–8. JWT verification (§8)

`AccessTokenService.verifyAccessToken`: header kid → public key; `jwt.verify` `algorithms: ['RS256']` (algorithm confusion/HS256 bloklanadi), issuer/audience/exp/signature; keyin `typ=access`, `sub`/`sid` UUID shape. `decode()`+trust YO'Q. DB query yo'q (live-check guard/controller'da).

## 9–10. AuthPrincipal & AccessTokenService

`AuthPrincipal { userId, sessionId }` — minimal (§10). `AccessTokenService` infrastructure: issue/verify only, DB yo'q, refresh logic yo'q.

## 11–12. AuthGuard & session revocation

Global `AuthGuard` (APP_GUARD): Bearer JWT verify → principal (baseline). `@Public()` chetlab o'tadi (health, otp/request, otp/verify, refresh). Passport yo'q. **Sensitive routelar** (logout, logout-all, me) controller'da **live-check** (§12): `assertSessionActive` (revoked/expired) + `assertAuthAllowed` (ACTIVE). Oddiy routelarda access token TTL bilan chegaralangan bounded revocation (§58 — global JWT blacklist/Redis YO'Q).

## 13–15. Decorators & PermissionsGuard

`@CurrentPrincipal()` (typed, `request.user as any` yo'q). `@RequirePermissions(...)`. Global `PermissionsGuard` (APP_GUARD, AuthGuard'dan keyin): server-side effective permissions, **ALL required** (default), no ADMIN bypass, no role-name hardcode, no cache. Metadata yo'q → o'tkazadi.

## 16. Guard strategy

Global AuthGuard + explicit `@Public()` (§16). Health/readiness/otp/refresh — `@Public`. Refresh JWT ma'nosida public, lekin cookie+CSRF bilan himoyalangan.

## 17. DTO validation

Global `ValidationPipe` (whitelist + forbidNonWhitelisted + transform). `RequestOtpDto` (phone), `VerifyOtpDto` (challengeId UUID, code `/^\d{6}$/` **string** — number'ga aylantirilmaydi, leading zero saqlanadi). Bu HTTP DTO validation TD-22 Activity JSONB validation qarorini belgilamaydi (alohida).

## 18–22. OTP request/verify

- **POST /api/auth/otp/request** (Public) → **202** `{ challengeId, expiresIn, resendAfter }`. LOGIN only (purpose client'dan olinmaydi). Oqim: DTO → per-IP limit → `issueChallenge` (account lookup YO'Q, enumeration-safe §22) → `SmsPort.sendOtp` → fail'da challenge invalidate (§21) + 503 → security event. Response'da OTP/phone-exists/user-status YO'Q.
- **POST /api/auth/otp/verify** (Public) → **200** `{ accessToken, tokenType, expiresIn, user: { id, onboardingCompleted } }`. Oqim: verify/consume challenge → canonical phone challenge'dan → findByPhone → yo'q bo'lsa `createLearnerAfterVerifiedPhone` (race → existing, §24) → `assertAuthAllowed` → createSession → issue JWT → set refresh cookie → login event. Response'da refresh/roles/permissions/phone YO'Q.

## 23–25. Login/registration

Verified phone → account lookup/create. Concurrent create race → phone unique constraint → existing user (duplicate profile/role yo'q). `onboardingCompleted` — routing uchun (yangi user false).

## 26–28. Refresh cookie

`izlan_refresh`; HttpOnly, SameSite=Lax, Path=/api/auth/refresh, Domain omitted (host-only), Max-Age = session absolute TTL. Secure — `AUTH_COOKIE_SECURE` (prod MAJBURIY true, env validation'da tekshiriladi). Value = opaque refresh token only. Cookie — minimal manual serialize/parse (`cookie.util.ts`, external dependency'siz — jest CommonJS-safe; signed cookie ISHLATILMAYDI, refresh o'zi yuqori entropiya + hash).

## 29–35. Refresh + CSRF + concurrency

- **POST /api/auth/refresh** (Public, cookie-auth) → **200** `{ accessToken, tokenType, expiresIn }` + rotated cookie. Oqim: `enforceRefreshCsrf` → cookie o'qish → yo'q bo'lsa 401 → `rotateRefreshToken` (1.4B strict) → issue JWT → set new cookie. Refresh JSON'da HECH QACHON.
- **CSRF (§31–33):** custom `X-Izlan-CSRF: 1` (yo'q/noto'g'ri → 403); `Sec-Fetch-Site: cross-site` → 403; Origin mavjud + allowlist'da emas → 403 (wildcard/regex YO'Q); header absent bo'lsa faqat shundan rad etilmaydi.
- **CORS (§34):** credentials=true + exact allowlist (`CORS_ORIGINS`); wildcard+credentials YO'Q.
- **Concurrency (§35):** 1.4B strict rotation o'zgarmagan — grace window yo'q; concurrent HTTP refresh → ≤1 succeed (e2e tasdiqlangan). Frontend contract: single-flight (quyida).

## 36–39. Logout / logout-all / me

- **POST /api/auth/logout** (Bearer) → **204**: revokeSession(principal.sessionId) idempotent + clear cookie.
- **POST /api/auth/logout-all** (Bearer) → **204**: revokeAllUserSessions(principal.userId) + clear cookie. Admin permission talab qilinmaydi (o'z qurilmalari).
- **GET /api/auth/me** (Bearer + live-check) → **200** `{ id, onboardingCompleted }`. Phone/DOB/permission/financial YO'Q. Live suspension → 403.

## 40–42. Error mapping

Domain error → HTTP `AuthExceptionFilter` boundary'da (core HTTP-independent). Shape `{ statusCode, code, message }` (machine code). Mapping: phone invalid → 400 AUTH_INVALID_INPUT; OTP invalid/expired/not-found → 400 AUTH_OTP_INVALID (enumeration-safe collapse); OTP locked → 429 AUTH_OTP_LOCKED; cooldown/rate → 429 AUTH_RATE_LIMITED; account → 403 AUTH_ACCOUNT_UNAVAILABLE; access/refresh/reuse/session invalid → 401 AUTH_UNAUTHORIZED (JWT sabablari collapse); CSRF → 403 AUTH_CSRF_REJECTED; duplicate → 409; noma'lum → 500 INTERNAL. Prisma/JWT/stack/token qiymatlari leak QILINMAYDI; safe kategoriya log qilinadi.

## 43–44. Rate limits

Per-IP hourly (`InMemoryAuthRateLimiter`, `otp:{ip}`) otp/request'da → 429 + security event. `request.ip` (trustProxy policy 1.4A authority — X-Forwarded-For mustaqil ishonilmaydi). Per-phone DB limitlar (1.4B) — asosiy identity himoya.

## 20, 47, 59. SMS

Port `SmsPort` (SENT/TEMPORARY/PERMANENT). **Production default: `UnavailableSmsAdapter`** (TEMPORARY_FAILURE — real provider tanlanmagan; OTP log/persist yo'q). Test: `test/test-sms.adapter.ts` `TestSmsAdapter` (xotirada capture, log yo'q) — test module override (`overrideProvider(SMS_PORT)`), NODE_ENV typo'ga bog'liq emas (§47). **No OTP debug bypass** (§60): dev-return/X-Debug-Otp/console.log/static code YO'Q.

## 65–67. Security events & cache

HTTP orchestration: otp_requested (issueChallenge ichida — dublikat yo'q), login_success (verify), rate_limit_triggered (IP). Reuse/session events core service'da (dublikat oldini olindi). Metadata'da OTP/token/Authorization/Cookie header/secret YO'Q; masked phone. Token-bearing javoblar (`otp/verify`, `refresh`, logout, me + xatolar) → `Cache-Control: no-store` (§67).

## Endpoints

| Method | Path | Auth | Status |
|---|---|---|---|
| POST | /api/auth/otp/request | Public + IP limit | 202 |
| POST | /api/auth/otp/verify | Public | 200 |
| POST | /api/auth/refresh | Cookie + CSRF | 200 |
| POST | /api/auth/logout | Bearer | 204 |
| POST | /api/auth/logout-all | Bearer | 204 |
| GET | /api/auth/me | Bearer + live-check | 200 |
| GET | /api/health, /api/ready | Public | 200 |

## 22. Tests

- **Unit (62):** AccessTokenService (RS256 roundtrip, expired/issuer/audience/kid/typ/tampered/**algorithm-confusion**/malformed), AuthGuard (missing/scheme/valid/invalid), PermissionsGuard (ALL/missing→403/no-ADMIN-bypass), + 1.4A/B.
- **E2e (46, --runInBand):** OTP request/verify (Learner create, access token, refresh cookie HttpOnly hash-only, no-leak, onboardingCompleted), duplicate login, bad flows (invalid phone/DTO field/invalid-expired-reused OTP), refresh (rotate + old→401 reuse), **CSRF** (missing/untrusted-origin/cross-site → 403), cookie attributes, me (bearer/401/refresh-as-bearer-rejected/live-suspension→403), logout (revoke+clear+old-refresh-fail+idempotent), logout-all, **concurrent HTTP refresh ≤1**.

## Database

Schema/migration **o'zgarmadi** (0 churn); drift yo'q. Test DB: izlan_test (guard, cleanup faqat auth jadvallar).

## 74. Frontend auth contract (Next.js)

1. **Login:** `POST /api/auth/otp/request` → `POST /api/auth/otp/verify` → `accessToken`ni **memory only** saqlash (localStorage/cookie EMAS). Refresh cookie brauzer avtomatik.
2. **API:** `Authorization: Bearer <accessToken>`.
3. **Reload (memory token yo'qoldi):** `POST /api/auth/refresh` `credentials: 'include'` + `X-Izlan-CSRF: 1` → yangi access token + rotated cookie.
4. **Concurrent 401:** **single-flight refresh** — bir vaqtda bitta refresh; boshqa failed requestlar shu promise'ni kutadi (strict backend reuse tufayli majburiy).
5. **Logout:** `POST /api/auth/logout` (Bearer) → memory token'ni tozalash.

## Deferred / OPEN (non-blocking)

- **Production SMS provider** — tanlanmagan (deployment implementation dependency).
- Account recovery, phone change, CAPTCHA/escalation, SecurityEvent retention, mobile token transport adapter, deployment secret manager, exact production origins/domain, refresh parallel-refresh grace (frontend single-flight bilan qoplanadi) — OPEN.
- Helmet/CSP (§46): frontend arxitekturasi noma'lum — CSP kiritilmadi (auth correctness blocker emas); deploy fazasida konservativ konfiguratsiya.
