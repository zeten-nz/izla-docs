# Izlan — Learner Web Foundation (Phase 3.0)

> **Mutable living doc** for the first learner-facing Izlan web experience (Phase 3.0, **TD-255**). The learner product
> and the staff CMS share ONE Next.js app (`izlan/web`). **TECHNICAL PASS — OWNER UI REVIEW PENDING.** Related:
> [METHODIST_CMS.md](METHODIST_CMS.md), [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md), [PRODUCT.md](PRODUCT.md),
> [DATA_MODEL_LEARNING.md](DATA_MODEL_LEARNING.md), [checkpoints/3.0.md](checkpoints/3.0.md).

## Goal

Turn the CMS-only web app into a shared Izlan web application with two clearly separated experiences — the **learner
product** (the primary Izlan experience) and the **staff CMS** — reusing the existing backend, auth, i18n and design
system. Scope: landing, learner auth (login/register/recovery), app shell, profile, onboarding, subject/track selection,
LearningIntent, a foundation dashboard, and a local demo learner. **No lesson player and no diagnostic UI yet.**

## Route map

| Route | Audience | Notes |
|---|---|---|
| `/` | public | Izlan landing (personalized self-study positioning). No longer redirects to staff. |
| `/login` | public | Learner phone + password login. |
| `/register` | public | Phone → OTP (REGISTRATION) → password. |
| `/forgot-password` | public | Phone → OTP (PASSWORD_RESET) → new password. |
| `/onboarding` | authenticated | Resumable 4-step setup wizard. |
| `/learn` | authenticated learner | Foundation dashboard. |
| `/learn/subjects` | authenticated learner | Manage LearningIntents (subject/track). |
| `/learn/profile` | authenticated learner | Safe profile edit + logout. |
| `/staff/login`, `/staff/content/**` | staff | **Unchanged** (Phase 2.2C). |
| `not-found` | any | Izlan-styled 404 (safe actions only). |

Branding: learner = **Izlan**; staff = **Izlan Studio**. A restrained footer link ("Metodistlar uchun") points to
`/staff/login` — never a primary CTA.

## Auth flows

- **One shared authority** (TD-251/252): access token **memory-only**; rotating HttpOnly refresh cookie; single-flight
  refresh; retry-once on 401; no token in localStorage/sessionStorage/URL. No separate learner/staff auth system — only
  route-specific wrappers.
- **Login** (`POST /api/auth/login`, phone+password, never OTP). On success routes by onboarding state:
  `onboardingCompleted=false → /onboarding`, else the **safe local** `?next` or `/learn`.
- **Register** (`POST /api/auth/otp/request` purpose `REGISTRATION` → `POST /api/auth/register`). Backend creates
  User+Profile+LEARNER+PasswordCredential+session atomically; the token is stored in memory; → `/onboarding`. Password
  8–128, never trimmed; resend respects the server `resendAfter`.
- **Recovery** (`/forgot-password`): `POST /api/auth/otp/request` purpose `PASSWORD_RESET` → `POST /api/auth/password/reset`.
  **Never auto-logs in** (server returns no token) → returns to `/login`.
- **Guard** (`LearnerGuard`): waits for auth bootstrap; unauthenticated → `/login?next=<local path>` (no content flash);
  guards are **not** role-name gated. Redirect targets are sanitized to local learner paths (no open redirect).
- **Error UX** (§17): backend `{code}` is mapped to safe, localized, enumeration-safe messages; the machine code is
  **never** shown (the generic fallback carries no code). The same improved mapping applies to the staff login.

## Onboarding flow (resumable)

Authority is backend state (`GET /profile/me`, `GET /onboarding/status`, `GET /onboarding/learning-intents`) — never a
local "complete" flag. Four steps: **Profil → Fan → Yo'nalish → Boshlash**.

1. **Profil** — `PATCH /api/profile/me` (displayName, dateOfBirth, timezone [suggested from `Intl`, server-authoritative],
   preferredLanguage). DOB becomes locked after completion.
2. **Fan** — `GET /api/onboarding/subjects` (PUBLISHED only). Selecting → `PUT /api/onboarding/learning-intent {subjectId}`
   (resumable subject-only). Empty state when no subject is published (valid 3.0 behavior — the pilot isn't auto-published).
3. **Yo'nalish** — `GET /api/onboarding/subjects/:id/tracks`. Selecting → `PUT .../learning-intent {subjectId, trackId}`.
4. **Boshlash** — review, then `POST /api/onboarding/complete` **only when `canComplete`**; refresh the in-memory auth
   user; → `/learn`. Placement assessment is **not** auto-started. Reloading mid-way resumes from backend state.

`/learn/subjects` reuses the same LearningIntent authority to add/change subjects and tracks after onboarding.

## Dashboard states (read-only)

Composed from existing READs; a page load **never** mutates learning state (no `POST /daily-plans/today`):

- **A — onboarding incomplete:** "Sozlashni davom ettiring" → `/onboarding`.
- **B — complete, no active roadmap:** `GET /roadmaps/me/subjects/:id/active` 404 → a foundation state; the diagnostic UX
  is Phase 3.2. No fake roadmap.
- **C — active roadmap, no today plan:** `GET /daily-plans/today` 404 → "Bugungi reja hali yaratilmagan." (not generated).
- **D — active roadmap + today plan:** real high-level plan metadata; lesson items show an **upcoming** (Phase 3.1) marker,
  never a broken link. No fabricated XP/streak/progress.

## Profile

`/learn/profile` edits only safe fields (displayName, dateOfBirth per backend policy, timezone, preferredLanguage) and
offers logout. It never exposes roles/permissions/internal IDs/sessions. DOB input is disabled after onboarding (no
mutation that would predictably fail). Saving `preferredLanguage` also reflects the chrome locale immediately; the browser
locale controls current chrome, the saved value is the profile preference.

## Locale & theme

UI i18n = chrome only, **uz (default) / ru / en**, via the existing `I18nProvider`/`useT` (dictionaries typed against
`uz`); locale persisted to localStorage + a non-auth cookie. Theme via the shared `ThemeProvider` (light/dark/system).
UI language ≠ teaching language ≠ target language.

## Demo learner (local)

The protected demo seed (`npm run db:seed:demo`, DEV/staging only, production-refused, `ALLOW_DEMO_SEED=true`, Argon2id,
idempotent) now also seeds a **LEARNER** account: `+998900000003`, role **LEARNER** only, **no SubjectAssignment**, and
**incomplete** onboarding (Asia/Tashkent timezone, no DOB, no LearningIntent) so the onboarding flow can be exercised from
a real incomplete account. Password from `DEMO_LEARNER_PASSWORD` (never committed). The English A1 pilot is **not**
auto-published for the demo — an empty subject step is valid. A dev-only login helper (`NEXT_PUBLIC_ENABLE_DEMO_ACCOUNTS=true`)
shows the learner on `/login` and fills the **phone only** (never the password, never auto-login); staff login keeps the
Admin/Methodist helpers.

## Env / local dev

- Backend `.env`: `PORT=3000`, `CORS_ORIGINS=http://localhost:4000` (credentialed CORS for the web dev origin),
  `AUTH_COOKIE_SECURE=false`, and (dev) `ALLOW_DEMO_SEED=true` + the demo passwords incl. `DEMO_LEARNER_PASSWORD`.
- Web `.env.local`: `NEXT_PUBLIC_API_BASE_URL=http://localhost:3000` (backend base). A **missing/empty** value now fails
  visibly as a config/network error — it does **not** silently degrade to same-origin. Changing any `NEXT_PUBLIC_*`
  requires restarting `npm run dev`. Web dev server runs on **:4000**.

## Backend changes (minimal)

- `preferredLanguage` registry `['uz','en']` → `['uz','ru','en']` (string field; no schema change) — LEARNER-LANG e2e.
- Demo seed adds the LEARNER account (DEMO-LEARNER e2e).
- **No** aggregate `/learner/dashboard` endpoint — the web composes existing reads. No schema change, no migration.

## Deferred

- **3.1** — Learner Lesson Player (lesson start / activity attempts / completion UI).
- **3.2** — Diagnostic/placement UX (questions / attempt progression / result).
- Later: roadmap/daily-plan generation UX, review sessions, XP/IZL, payments, community, Telegram, media, AI tutor.
