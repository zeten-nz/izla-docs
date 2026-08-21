# Telegram Integration — Architecture Reconnaissance (Phase 2.2T-P)

> **RECON ONLY — no implementation.** No runtime/schema/migration/endpoint/bot/Mini App/payment code. No TD accepted.
> Inspected `izlan` code SHA: **`281ca4159e3bfe08ce9bb1e6a26f865f04cd5017`** (main; differs from `19461eb` only by
> `CLAUDE.md`). Runtime baseline unchanged: migrations 21, unit 397, e2e 432, total 829, CHECK 45, drift clean.
> Docs branch: `phase/2.2T-P`. Telegram is an architecture **candidate**; nothing here is an accepted owner decision.

## 1. Official Telegram contract findings (current, core.telegram.org — 2026-08)
Every fact below is DOCUMENTED on the cited official page unless marked **VERIFY-LATER**.

### 1.1 Telegram Login = OpenID Connect (NOT the legacy hash widget)
Source: [bots/telegram-login](https://core.telegram.org/bots/telegram-login), [widgets/login](https://core.telegram.org/widgets/login), discovery [oauth.telegram.org/.well-known/openid-configuration](https://oauth.telegram.org/.well-known/openid-configuration). The legacy iframe/HMAC Login Widget is **archived**.
- **Flow:** OIDC **Authorization Code + PKCE** (`code_challenge_method` `S256` recommended; `plain` also listed). `response_types=code`, `grant_types=authorization_code`.
- **Endpoints:** auth `https://oauth.telegram.org/auth` · token `https://oauth.telegram.org/token` · **JWKS** `https://oauth.telegram.org/.well-known/jwks.json`. **No UserInfo endpoint** (all claims in the id_token — OIDC libs must be told to skip userinfo).
- **`iss` = `https://oauth.telegram.org`**; **`aud` = your Bot ID** (client_id). `id_token` = JWT; signing algs `RS256`(default)/`ES256`/`EdDSA`/`ES256K`.
- **Scopes:** `openid` (required → `sub`,`iss`,`iat`,`exp`) · `profile` (→ `id`,`name`,`preferred_username`,`picture`) · `phone` (→ `phone_number`,`phone_number_verified`, **requires user consent**) · `telegram:bot_access` (allows the bot to DM the user after login; no claim).
- **Stable identity = `sub`** (string). A numeric `id` claim is also present. **`preferred_username` is MUTABLE — never an identity key.**
- **Phone is NOT guaranteed** — present only when the `phone` scope is requested AND consented.
- **Client registration:** the client IS a Telegram bot (via @BotFather); BotFather issues **Client ID + Client Secret** and a **redirect_uri allowlist** ("Allowed URLs").

### 1.2 Mini App (Web App) authentication
Source: [bots/webapps](https://core.telegram.org/bots/webapps).
- `Telegram.WebApp.initData` (raw string — validate server-side) vs `initDataUnsafe` (parsed — **never trust**).
- **HMAC validation:** `secret_key = HMAC_SHA256(key="WebAppData", data=bot_token)`; `data_check_string` = all fields except `hash`, sorted alphabetically, `key=value` joined by `\n`; accept iff `hex(HMAC_SHA256(data_check_string, secret_key)) == hash`.
- **Newer Ed25519 `signature`** (validate WITHOUT the bot token): `data_check_string` prefixed with `{bot_id}:WebAppData\n…`; `Ed25519_verify(TelegramPublicKey, data_check_string, signature)`. Telegram publishes test + production Ed25519 public keys. (Useful if a separate auth service must validate without the bot token.)
- Fields: `query_id, user, auth_date, hash, signature, start_param, chat_type, chat_instance, receiver, chat, can_send_after`. `user` = `{id, is_bot?, first_name, last_name?, username?, language_code?, is_premium?, photo_url?, allows_write_to_pm?}`.
- `auth_date` = Unix timestamp for **freshness/replay** checks; **no fixed max-age is mandated** (Izlan must choose, e.g. reject stale `initData`).

### 1.3 Bot API identifiers & channel membership
Source: [bots/api](https://core.telegram.org/bots/api) (`getChatMember`, ID notes).
- **Telegram `user_id`/`chat_id` exceed 32 bits (≤52 bits) → MUST store as 64-bit** (`BIGINT` / signed 64-bit). Never 32-bit int.
- `getChatMember(chat_id, user_id)` returns a `ChatMember` (`creator`/`administrator`/`member`/`restricted`/`left`/`kicked`). **To query arbitrary members of a channel the bot generally must be an administrator of that channel** (VERIFY-LATER for the exact per-channel guarantee at implementation).

### 1.4 Telegram Stars (XTR) — digital goods
Source: [bots/payments-stars](https://core.telegram.org/bots/payments-stars).
- **Payments for digital goods/services inside a bot/Mini App MUST use Telegram Stars (currency `XTR`)** — mandatory (App Store / Play Store compliance).
- Flow: `sendInvoice` (currency `XTR`, empty `provider_token`) → `pre_checkout_query` (`answerPreCheckoutQuery` within 10s) → `successful_payment`.
- `successful_payment` carries `telegram_payment_charge_id` (refund key), `invoice_payload`, `total_amount`, `currency`, `provider_payment_charge_id`.
- **Refunds:** `refundStarPayment(telegram_payment_charge_id)`; the bot must handle `/paysupport`. **No idempotency/replay guidance is documented → Izlan must enforce it** (dedup on `invoice_payload` + `telegram_payment_charge_id`).

## 2. Existing Izlan auth findings (grounded in `izlan@281ca415`)
- **Identity = phone.** `User.id` = uuid7 PK; `User.phone String @unique` **NOT NULL** (E.164, TD-16) — the only unique columns are `id` and `phone` (`core.prisma:10-11`). `User.status UserStatus {ACTIVE,SUSPENDED,DEACTIVATED}` (`schema.prisma:23-27`).
- **User is created in exactly one place:** `UsersService.createLearnerAfterVerifiedPhone` (`users.service.ts:46-64`) — one `$transaction` → `User(phone)` + `UserProfile` + LEARNER role.
- **OTP verify = find-or-create** (login + silent auto-registration in one flow; only `OtpPurpose.LOGIN`). The OTP layer is identity-agnostic (returns `{canonicalPhone}`); branching is in `AuthController.verifyOtp` (`auth.controller.ts:84-109`).
- **Duplicate prevention** = `phone @unique` + P2002 catch (`DuplicatePhoneError`) + find-then-create race re-fetch. `OtpChallenge.phone` is a standalone string (no FK to User).
- **No provider abstraction exists** — no `UserIdentity`/`AuthProvider`/`oauth`/`externalId` model anywhere. Identity is strictly phone-on-User.
- **Below the phone layer the code is already identity-agnostic:** `SessionsService.createSession(userId,platform,clientInfo?)`, `issueAccessToken(userId,sessionId)` (claims `sub/sid/typ/iss/aud`, RS256, no phone/role), refresh rotation + reuse→family-revoke, logout/logout-all — **none depend on phone**; and **nothing dereferences `User.phone`** (bootstrap/profile omit it). So making phone nullable **breaks nothing that reads it** — only the phone write/lookup paths in `users.*` + the OTP login branch assume phone presence (that IS the phone provider).
- **`AuthSession.platform` is a free-form String** (comment `web/android/ios`, not an enum) — it **can already carry** `telegram_web_login`/`telegram_miniapp`/`bot` **without a schema change**; the sole issuer hardcodes `'web'` (`auth.controller.ts:100`). (Minor DOCUMENTATION DRIFT: the comment enumerates only web/android/ios.)
- **Gaps found (independent of Telegram, worth flagging):**
  - **Suspension does not revoke sessions.** `User.status` is enforced read-time only (`assertAuthAllowed` at login, `/me`, session-create); the global `AuthGuard` verifies only the JWT signature (no live session/account check); **refresh does not re-check account status** → a suspended user's tokens keep working until the session is explicitly revoked or expires. No admin suspend/deactivate endpoint exists.
  - **Refresh cookie is same-site only:** `izlan_refresh`, `Path=/api/auth/refresh`, `SameSite=Lax`, HttpOnly, host-only Domain; CSRF = `X-Izlan-CSRF` header + reject `Sec-Fetch-Site: cross-site` + exact `Origin` allowlist. **A Telegram Mini App runs on Telegram's origin (`web.telegram.org`/`*.t.me`) → cross-site to the Izlan API**, so `SameSite=Lax` blocks the cookie, `Sec-Fetch-Site:cross-site` trips CSRF, and the origin fails the allowlist. **The cookie-refresh design is not directly reusable from a Mini App/webview** without `SameSite=None; Secure` + allowlisting the Telegram origin, or a **non-cookie refresh transport** for that channel.

## 3. Identity architecture options (§6)
| | schema impact | auth-service impact | migration risk | dup-account risk | phone-OTP compat | TG-only signup | future Google/Apple | recovery | security | onboarding |
|--|--|--|--|--|--|--|--|--|--|--|
| **A** phone on User + `TelegramIdentity` side table | small (1 table) | small | low | low | unchanged | ✗ (TG-only still needs phone) | ✗ (per-provider tables proliferate) | ok | ok | phone-first stays |
| **B** provider-neutral `UserIdentity(userId, provider, providerSubject)` + phone nullable (partial-unique) | moderate (1 table + phone nullable + partial unique) | small (generalize 2 `users.*` methods + 1 auth branch) | **low–moderate** (nullable + partial unique; User.id stable) | low (unique `(provider, providerSubject)`) | unchanged (phone = one provider) | ✓ | ✓ (one table, N providers) | flexible | strong (single principal) | phone-first preserved |
| **C** phone stays sole identity; no linking (separate TG "profile") | none | none | none | — | — | ✗ | ✗ | — | risk of parallel/duplicate identity | — |

### Recommended identity model — **Option B** (grounded, not aesthetic)
The codebase is *already* identity-agnostic under the phone layer (sessions/JWT/profile key off `userId`; phone never dereferenced), so Option B is the smallest change that supports Telegram-only users and future Google/Apple without a parallel auth system. Concrete minimum shape (for a later 2.2T-D, **not built now**):
- Keep `User.id` (uuid7) as the **stable platform principal** — never changes.
- Make `User.phone` **nullable**, replace the column `@unique` with a **partial unique index** (`phone` unique WHERE NOT NULL).
- Add `UserIdentity(id, userId FK→User, provider, providerSubject, createdAt, …minimal metadata)` with `@@unique([provider, providerSubject])`. Telegram provider stores **`sub` (OIDC) as `providerSubject`** (string; the numeric `id` may be kept as non-identity metadata). Phone may remain on `User` for the phone provider (minimum migration) — a later normalization into `UserIdentity` is optional.
- Generalize `createLearnerAfterVerifiedPhone` → `createLearnerAfterVerifiedIdentity(provider, providerSubject, phone?)` reusing the same P2002 duplicate handling on the new unique index; add a Telegram auth branch that reuses `createSession` + `issueAccessToken` + `getAuthBootstrap` unchanged.

## 4. Telegram-only registration (§7) — OWNER DECISION
- **A. Allowed:** a User may exist with only a Telegram identity and no phone (requires Option B's nullable phone). Low friction/cost; broadens funnel; but phone-recovery is unavailable for those users (see §8).
- **B. Link/create only when a verified phone is available/accepted:** requires the `phone` scope consent (not guaranteed) or a follow-up SMS OTP. Higher friction/cost; keeps phone as universal recovery anchor.
- **Recommended MVP:** **A (Telegram-only allowed)** with a *soft* prompt to add a phone for recovery — because the `phone` scope is consent-gated and cannot be relied upon, and Option B already supports phone-less users cleanly. Final call is the owner's.

## 5. Duplicate-account / linking matrix (§8)
Rule: **never auto-merge on weak profile data** (name/username/avatar/photo are NEVER sufficient). Only a **verified phone match** or an **explicit authenticated link** may join identities.
| # | Situation | Safe default | Auto? | Re-auth? | Reject/conflict? | Owner decision |
|--|--|--|--|--|--|--|
| 1 | Phone user logs in via TG; TG returns **same verified** phone | link TG identity to the existing user | auto-link allowed **only** on verified-phone match | recommended (confirm) | — | policy on "auto vs confirm" |
| 2 | Phone user; TG identity already linked to **another** user | **CONFLICT** — do not move/merge | no | yes | yes | conflict UX |
| 3 | Phone user; TG returns a **different** verified phone | log in as the TG identity's own account (or none) — **do not link** to the phone user | no | — | no silent link | — |
| 4 | New TG user; phone scope accepted; **phone matches no** account | create new user (TG identity + optional phone) | auto-create | — | — | — |
| 5 | New TG user; phone scope **refused/absent** | create TG-only user (if §7=A) **or** prompt SMS (if §7=B) | per §7 | — | — | tied to §7 |
| 6 | TG identity exists; user later **adds/verifies** a phone | attach phone to same user unless that phone belongs to another account → CONFLICT | auto unless collision | recommended | on collision | — |
| 7 | Phone user **intentionally links** TG while authenticated | link via one-time nonce (§6 below) | auto after proof | **yes (authenticated session)** | on already-linked | — |
| 8 | TG account lost/changed | requires recovery (§8) — no automatic reattach | no | yes | — | recovery policy |
| 9 | Phone number **recycled** (new person gets old number) | phone match alone must not grant a pre-existing account without re-verification; consider phone-change audit | no | yes | — | recycling policy |
| 10 | Linking would **collide** two existing accounts | **CONFLICT** — never auto-merge; explicit account-merge is a separate future feature | no | yes | yes | merge policy (future) |

## 6. Account-linking security (§9)
Intentional link (authenticated Izlan user → prove Telegram identity):
1. authenticated user requests link → server mints a **high-entropy, single-use, short-TTL nonce**, bound to that user/session;
2. the Telegram flow (OIDC callback `state`/`nonce`, or a bot `/start=<nonce>` deep-link, or Mini App `start_param`) carries/echoes the nonce;
3. server validates the Telegram proof (OIDC id_token or initData), **consumes the nonce atomically**, and attaches the identity in one transaction.
Requirements: high entropy · short TTL · one-time (consume atomically) · server-generated · bound to the intended user/session · replay-safe · **never accept a client-provided `userId` as link authority**. Persistence: a short-TTL cache/table is sufficient (DB row with `usedAt` or a TTL store); no heavy persistence needed. For bot deep-links, the `start_param`/`/start` token must be the same kind of single-use nonce — a raw predictable value is unsafe.

## 7. Unlink, recovery & the last-auth-method invariant (§10) — OWNER DECISION
- Proposed **permanent invariant:** *a user may not remove their LAST usable authentication method unless an explicit recovery mechanism exists.* (Recommended to adopt.)
- Open recovery questions (owner): unlink Telegram; change phone; lose phone; lose Telegram; both lost; suspended/deactivated recovery. Account recovery policy remains an owner decision. **Note the existing suspension gap** (§2) should be fixed regardless (revoke sessions on status change).

## 8. Session architecture (§11)
Converge on the **existing** Izlan session system — do **not** create a parallel `TelegramSession` lifecycle. Target flow: **Telegram proof (OIDC id_token / validated initData) → resolve/create Izlan User → `createSession(userId, platform=telegram_*)` → normal access JWT → normal refresh rotation/revocation.**
- `AuthSession.platform` (free String) can represent `web` / `telegram_web_login` / `telegram_miniapp` / `bot` **without a schema change** (recommend later promoting it to an enum for integrity + a device/session list; not now).
- Reuse session-create, logout, logout-all, refresh reuse-detection, suspension revoke — all `userId`-keyed, phone-independent.
- **Mini App transport caveat (load-bearing):** the current same-site refresh cookie will not work cross-site from a Telegram Mini App. **Owner decision (§4 Mini App model):** (A) validate initData every request (stateless, no Izlan session) — simple but no unified session/logout; (B) **exchange validated initData once for a normal Izlan session + access token** (unified session model) — **recommended**, but requires a non-cookie refresh transport for the Telegram channel (return the refresh token in the response body for the Mini App to store and send as a header, or `SameSite=None; Secure` + Telegram-origin allowlist); (C) hybrid. Exact Telegram webview cookie behavior is **VERIFY-LATER** against current Telegram Mini App docs at implementation.

## 9. Bot auth / identity & notification consent (§5/§12/§13)
- **The bot is a client of the same Izlan backend — no learning/economic logic in the bot.** It calls Izlan application services / read models; it must NOT recompute roadmap/mastery/IZL.
- **Bot identity:** map the Telegram **user id** (64-bit) to an Izlan User via `UserIdentity` — the same principal as OIDC where possible. Do **not** treat a private-chat `chat_id` as the persistent user identity (chat_id ≠ user identity for architecture). Distinguish user_id vs chat_id vs OIDC `sub` vs `username` vs channel membership.
- **Notification consent is not one flag** — separate: (a) Telegram Login authorization, (b) `telegram:bot_access` scope (bot may DM), (c) the user having `/start`-ed the bot (bot can message), (d) the Izlan-side notification preference, (e) whether a Telegram identity is linked. Recommend a **minimal consent/state model** later: `linkedAt`, `botCanMessage` (bool), `notificationsOptIn` (per-category later) — **not built now**.
- **MVP bot read-model surface to evaluate:** `/start` (link/deep-link), `/today`, `/progress`, `/balance`, `/review`, `/settings`. **Read-model gaps a future bot would expose (do not build):** a compact "today's plan" projection, a streak/progress summary, an XP/IZL balance read — all likely need thin read endpoints over existing domain state.

## 10. Channel membership boundary (§14)
- Optional Telegram **channel** (marketing/community). `getChatMember` requires the bot's admin relationship (VERIFY-LATER exact guarantee). **Channel membership must NEVER be login authority, account identity, or subscription/payment authority.**
- Reward farming risk (join→reward→leave→rejoin). Options: **A** no economic reward · **B** one-time XP/badge/cosmetic mission · **C** IZL economic reward. **Recommended MVP: A or B** (never IZL for a repeatable, weakly-verifiable action). Owner decision; not implemented.

## 11. Mini App product boundary (§15)
Evaluate: **A** reuse the responsive Izlan web frontend where practical · **B** separate Mini App frontend · **C** shared components/domain client, separate shell. Recommend **C** (share the domain client + components, keep a Telegram-specific shell for the initData auth + Mini App UX), avoiding backend duplication. Future screens: Today / Roadmap / Lesson / Progress / Profile / Bot settings. **No frontend built now.**

## 12. Payments — Stars vs website vs IZL (§16/§17/§18/§19)
- **Hard boundary:** **izlan.uz website → future CLICK/Payme** (the existing, **PAUSED** provider track); **inside Telegram bot/Mini App → Telegram Stars (`XTR`)** for digital subscription/AI/course access (mandatory per Telegram rules). Do **not** design a CLICK/Payme/P2P workaround for digital purchases inside Telegram.
- A Telegram-linked user may **still** buy on izlan.uz via website providers (do not force Stars for website purchases). Flag the UX split: purchases made where the user is.
- **Stars ≠ IZL.** Stars = external Telegram payment rail; IZL = internal platform currency. **Never** model `1 Star = N IZL wallet balance` without an explicit future economic decision.
- **Future fit:** Telegram Stars should later become **another `PaymentProvider`** behind the *existing* provider/economic-finalization boundary — a Stars adapter emits verified success evidence (`telegram_payment_charge_id` as the external id, `invoice_payload` as the merchant reference) into the same `PaymentTransaction`/`PaymentOrder`/finalization pipeline; refunds (`refundStarPayment`) map to the **future refund domain** (still unbuilt). This preserves 2.1E–2.1L architecture. **No payment code now; the CLICK/Payme track stays PAUSED and untouched.**
- **P2P / personal-card top-up (§19): rejected for production.** No automatic balance credit from screenshots / SMS/push scraping / card-notification parsing / "I paid" / timestamp guessing. If a future manual beta is ever wanted, the only auditable shape is a `MANUAL_ADMIN_PAYMENT` provider type (explicit `PaymentTransaction` + evidence + actor audit + amount + reference + idempotency; never a raw `balance +=`). **Future / owner decision; not this phase.**

## 13. Threat model (§20)
| Threat | Current protection | Gap | Future mitigation |
|--|--|--|--|
| Forged OIDC callback / stolen auth code | none (no Telegram yet) | no OIDC client | strict code-for-token exchange server-side; never trust front-channel |
| Missing/weak PKCE | none | — | require `S256` PKCE; reject `plain` |
| `state`/`nonce` replay | existing CSRF only for cookie refresh | no OIDC state store | per-request `state`+`nonce`, single-use, bound to session |
| Wrong `iss`/`aud` | none | — | enforce `iss=https://oauth.telegram.org`, `aud=BotID` |
| Expired id_token | none | — | check `exp`/`iat`; small clock skew |
| JWKS/key rotation | none | — | cache JWKS with rotation/refresh by `kid` |
| Mini App forged `initData` | none | — | server-side HMAC (bot-token) or Ed25519 `signature`; never trust `initDataUnsafe` |
| Stale/replayed `initData` | none | no freshness bound | enforce `auth_date` max-age + one-time `query_id`/nonce |
| Link-nonce replay | none | no nonce store | high-entropy single-use short-TTL nonce, atomic consume |
| Arbitrary `userId` linking | N/A | — | link authority = server nonce bound to authenticated session, never client `userId` |
| Username takeover/change | N/A | — | identity = `sub`/user_id only; never `preferred_username` |
| Duplicate-identity race | phone `@unique`+P2002 today | no provider unique yet | `@@unique(provider, providerSubject)` + P2002 handling |
| Concurrent linking | pay/session locks exist | no link path | transactional link + unique constraint |
| Session fixation | rotation + reuse-detect | — | reuse existing rotation on Telegram sessions |
| Bot webhook spoofing | none | — | secret-token header on webhook; verify |
| Leaked bot token / OIDC client secret | .env, gitignored; masked logs | broad token power | secret store; rotation; never log |
| Log leakage | phone masking + no-secret-log rules | — | never log id_token/initData/tokens |
| Malicious `start_param` | none | — | validate/whitelist; treat as untrusted; nonce only |
| Channel reward farming | none | — | no IZL for join; one-time/cosmetic only |
| Stars payment replay | none | Telegram gives no idempotency | dedup on `telegram_payment_charge_id`+`invoice_payload` |
| Suspension not revoking sessions | **existing gap** (read-time only) | tokens live post-suspend | revoke sessions on status change + live check on refresh |

## 14. Schema-gap table (§23) — no models created
| Area | Current model | PASS/GAP | Minimum future change | Owner decision? | Target phase |
|--|--|--|--|--|--|
| `User.phone` required+unique | NOT NULL `@unique` | GAP for TG-only | nullable + partial unique (WHERE NOT NULL) | yes (identity model) | 2.2T-D |
| provider-neutral identity | none | GAP | `UserIdentity(userId,provider,providerSubject)` `@@unique(provider,providerSubject)` | yes | 2.2T-D |
| Telegram identity storage | none | GAP | provider='telegram', providerSubject=OIDC `sub`; store TG user_id as **BIGINT** metadata | — | 2.2T-D |
| Telegram-only user | impossible (phone required) | GAP | depends on nullable phone | yes (§7) | 2.2T-D |
| account linking | none | GAP | single-use nonce (short-TTL cache/table) | — | 2.2T-A |
| notification/bot consent | none | GAP | minimal `botCanMessage`/`notificationsOptIn`/`linkedAt` | yes (opt-in model) | 2.2T-B |
| session channel | `platform` free String | PASS (works) | later: enum + device/session list | soft | 2.2T-A/later |
| suspension revoke | read-time only | GAP (pre-existing) | revoke sessions on status change + refresh live-check | — | any (auth hardening) |
| Mini App refresh transport | cookie same-site only | GAP for Mini App | non-cookie refresh OR SameSite=None+allowlist | yes (§4 model) | 2.2T-M |
| Stars payment evidence | PaymentProvider enum (CLICK/PAYME) | GAP (future) | add STARS provider + charge-id external id via existing pipeline | yes | 2.2T-S (after refund domain) |
| channel mission state | none | GAP (optional) | one-time mission record if B chosen | yes (§14) | later |
| privacy/unlink | erasure policy OPEN | GAP | unlink + erasure semantics for TG metadata | yes | 2.2T-D/later |

## 15. Migration risk (§24)
An identity refactor (Option B) migrates **phone-centric** existing users **without touching `User.id`**: keep every `User.id` as the stable principal (all roles/progress/XP/IZL/subscriptions/relations reference `user_id` and are untouched); make `phone` nullable and swap the column `@unique` for a **partial unique index** (existing non-null phones remain unique — no duplicate risk); existing users need **no** `UserIdentity` row (phone stays their provider). New Telegram users get a `UserIdentity` row and null phone. No user-id change, no data loss, no historical-relation invalidation. (No migration SQL written here.)

## 16. Owner decisions required (§25) — none accepted here
1. **Identity model** — A (side table) vs **B (generic `UserIdentity` + nullable phone)** vs C. *Rec: B.*
2. **Telegram-only signup** — allowed (A) vs phone-required (B). *Rec: A with soft phone prompt.*
3. **Existing phone-account auto-link** — only on **verified-phone match**, auto vs confirm-step. *Rec: link only on verified-phone match, with a confirm step.*
4. **`phone` scope** — request by default vs optional later linking step. *Rec: optional (don't rely on it; it's consent-gated).*
5. **Recovery policy** when Telegram/phone is lost. *Owner.*
6. **Last-auth-method invariant** — adopt permanently? *Rec: yes.*
7. **Mini App session model** — A per-request initData vs **B one-time exchange for Izlan session** vs C. *Rec: B (needs non-cookie refresh transport).*
8. **Bot notification opt-in model** (which of the 5 consent facts gates messaging). *Owner; rec explicit opt-in.*
9. **Channel mission/reward** — A none / B one-time XP-cosmetic / C IZL. *Rec: A or B (never IZL).*
10. **Telegram Stars boundary** — confirm Stars-for-digital-in-Telegram + Stars-as-future-PaymentProvider behind existing finalization. *Rec: confirm.*
11. **Website vs Telegram payment UX** — website=CLICK/Payme, Telegram=Stars; both allowed per surface. *Rec: confirm.*
12. **Manual P2P/admin payment** — future auditable `MANUAL_ADMIN_PAYMENT` only, or reject entirely. *Owner; rec future-only, never scraping.*

## 17. Recommended rollout (§26) — not started
`2.2T-D` Telegram identity/auth schema hardening (UserIdentity + nullable phone + partial unique; BIGINT TG id metadata) → `2.2T-A` Telegram Web Login (OIDC) + account linking (nonce) → `2.2T-B` Bot foundation + notifications/read-models → `2.2T-M` Mini App auth + shell (resolves the refresh-transport decision) → `2.2T-S` Telegram Stars PaymentProvider (**after** the refund domain exists). **Stars/payment stays later than identity/auth; the CLICK/Payme track does NOT resume because Telegram recon exists.**

## 18. Content-track interaction (§27)
**Can Phase 2.2A-D proceed independently of Telegram? — YES.** 2.2A-D touches only content-domain schema (`LessonPrerequisite` self-loop CHECK + `Lesson.contentKey`); it has **zero** dependency on identity/auth/session/Telegram. Telegram decisions do **not** block 2.2A-D.
However, Telegram identity decisions **should be frozen BEFORE** building **frontend auth UI, account-settings, and mobile/Mini App** work — those surfaces embed the identity/session/linking model and are expensive to redo. Sequence content-schema (2.2A-D) freely; gate auth-facing UI on the §16 identity decisions.

## 19. Secrets / privacy (§21/§22)
Future config (values **never** committed/logged): bot token, OIDC client_id/client_secret, redirect_uri allowlist, webhook secret, channel id, Mini App URL — env/secret-store only. Minimal Telegram data to store: **identity authority** = OIDC `sub` / Telegram user_id (BIGINT); **display/cache** = current username/display name (refreshable, mutable — not identity); `linkedAt`, optional `lastSeenAt`, `botCanMessage`; **phone only if explicitly received/verified and actually needed**. Do not store profile-photo URLs or mutable profile data without a product reason. Unlink/erasure must remove Telegram metadata; identity authority participates in the (still-OPEN) privacy-erasure policy.
