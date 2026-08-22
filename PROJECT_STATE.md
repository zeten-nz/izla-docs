# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-22)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests + `web/` | `phase/3.0` | `5c772d7c4897b60529b5c9eae0a5f1cdded01678` (base `main` `b1bfd80`, which merged 2.2E) | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `phase/3.0` | `phase/3.0` (final SHA in PR; base `main` `4d28b8c`, which merged 2.2E docs) | clean |

**Phase 2.2E is MERGED / CLOSED** — runtime `main` `b1bfd80575202df544bdd1e8438dfacb35e5df66`, docs `main`
`4d28b8c6394d1804cde78eb975913987cfb6712a`.

**Phase 2.2D is MERGED / CLOSED** — runtime `main` `a977c35816d5642a93eb071b1ad56d71aba6400d`, docs `main`
`2bdd590bdd7cae88e19d94ecbfed6569d16e30ab`.

\* Phase **2.2B** (review + publishing + preview + readiness + learner visibility) implemented on branch `phase/2.2B` —
izlan base `main` @ `9ebd90a` (which merged the 2.2A-3 PR #6); docs base `main` @ `4b8b89f` (which merged the 2.2A-3 docs,
PR #9). **Code SHA `85ddb10`** is this phase's implementation; izlan `main` stays `9ebd90a` until the PR merges. The
Baseline below reflects the `phase/2.2B` branch state, OWNER REVIEW PENDING.

> **Governance note:** before the 2026-08-21 workflow adoption, historical phases were committed coarsely to `main` and
> do **not** have per-phase SHAs or phase branches — they are all contained in code `19461eb` / docs `92cadce`
> (historical authority). Per-phase branch/SHA recording begins with the adopted workflow.

## Current position
- **Last completed:** Phase **3.0** — Learner Web Foundation. Result: **TECHNICAL PASS — implementation complete on branch
  `phase/3.0` (code `5c772d7`); OWNER UI REVIEW PENDING (not merged)**. The **first learner-facing Izlan web experience**,
  in the SAME Next.js app as the staff CMS (no second frontend): a public landing at **`/`** (Izlan positioned as
  personalized self-study; the staff redirect removed), learner auth (**`/login`** phone+password, **`/register`** phone→
  OTP→password, **`/forgot-password`**), a resumable 4-step **`/onboarding`** wizard (Profil→Fan→Yo'nalish→Boshlash), and
  the learner app (**`/learn`** read-only foundation dashboard, **`/learn/subjects`**, **`/learn/profile`**) behind a
  learner shell (top nav + mobile bottom nav) distinct from the staff sidebar. Branding: learner = **Izlan**, staff =
  **Izlan Studio**. **ONE shared auth/session authority** is reused (access token memory-only, single-flight refresh); a
  `LearnerGuard` waits for bootstrap and routes to `/login`; post-login intent is **per login page** (learner →
  `/onboarding`|`/learn`; staff → `/staff/content`), never role-gated, with **safe-local `?next` only** (no open redirect).
  Onboarding authority is backend state (profile + LearningIntent + `canComplete`); the dashboard is **read-only** (never
  `POST /daily-plans/today`, no fabricated XP/streak/progress). Friendly, enumeration-safe, **code-free error UX**
  (the generic message no longer prints the machine code) applied to staff login too. **API base hardening**: a missing
  `NEXT_PUBLIC_API_BASE_URL` fails visibly instead of silently hitting same-origin. UI i18n uz/ru/en. **Backend changes are
  minimal**: `preferredLanguage` registry → uz/ru/en (string field) and a LEARNER demo-seed account (+998900000003,
  LEARNER-only, no assignment, incomplete onboarding) — the English A1 pilot is NOT auto-published. **No schema change,
  no migration** (migrations **23**, CHECK **46**). TD-255 added; living doc [LEARNER_WEB_FOUNDATION.md](LEARNER_WEB_FOUNDATION.md);
  see [checkpoints/3.0.md](checkpoints/3.0.md). The lesson player (→ 3.1) and diagnostic UI (→ 3.2) are NOT built.
- **Prior:** Phase **2.2E** — English A1 Pilot Content v1. Result: **TECHNICAL PASS — MERGED to `main` (`b1bfd80`);
  PEDAGOGICAL OWNER/METHODIST REVIEW still recommended before real publication**. The **first real
  educational content pack**: English → General English → A1 → A1 Foundations, **4 Topics / 12 Lessons / 98 Activities /
  13 Skills** (~176 min), teaching language **Uzbek**, target **English**. Content lives in the runtime repo at
  `content/pilots/english-a1/v1/` (4 `izlan-topic-content/v1` packages + `manifest.json` + `README.md`) — authoring source
  files that carry server-only answerKey and are **never** delivered to the browser. Markdown + objective activities only
  (no media/speaking/writing/listening/AI). Linear prerequisite chain 001→012; 13 Subject-scoped Skills reused across
  packages. A `npm run content:pilot:a1:validate` command + PILOT unit tests validate the pilot through the
  EXISTING importer parser (no second format/parser). A real e2e imports all 12 lessons (4 import audits, DRAFT-only),
  publishes the whole pilot top-down + in prerequisite order via the existing review→publish workflow with **zero
  readiness blockers**, and smoke-tests learner-safe projection (no answerKey leak) + deterministic objective scoring
  (10000/0, no AI). An **owner-review correction** (`016400c`) added **TD-254** (bulk-import Activity provenance: optional
  `provenance.source` HUMAN/AI_ASSISTED/AI_GENERATED, omitted→HUMAN, part of documentHash, no schema) so the pilot
  persists **`AI_ASSISTED`** (not HUMAN) while human review remains required, and fixed the pedagogy (Lesson 06 full
  0–100/phone digit-reading + both-skill mastery; Lessons 07/08 no longer pre-use have/has; Lesson 12 real objective
  cumulative retrieval). A **final surgical reconciliation** (`e1112e0`, current HEAD) made `provenance: null` reject as
  `IMPORT_INVALID_DOCUMENT` (never silent HUMAN; IMP-PROV-06) and pinned the Lesson 12 cumulative ActivitySkill mapping.
  **No schema change, no migration** (migrations **23**, CHECK **46**); nothing auto-imports into
  dev/prod and publication remains a manual CMS step after human review. The content is an **AI-assisted draft** — not
  pedagogically approved, not CEFR-certified. See [ENGLISH_A1_PILOT.md](ENGLISH_A1_PILOT.md) and
  [checkpoints/2.2E.md](checkpoints/2.2E.md). Latest TD is **TD-254** (amends TD-253). The learner web app is a 3.x concern.
- **Earlier:** Phase **2.2D** — Topic-Scoped JSON Bulk Content Import v1. Result: PASS — **MERGED to `main` (`a977c358`)**.
  The first bulk-authoring pipeline: local JSON
  document → strict validation → dry-run plan → human confirmation → **atomic DRAFT import** into an **existing** Topic.
  Imports Skills (by Subject-scoped `code`, reuse ACTIVE), Lessons (create-only `contentKey`) + one initial LessonRevision
  (v1), Activities (**markdown + objective only**), LessonSkill/ActivitySkill mappings, and LessonPrerequisite edges — all
  as DRAFT (`publishedRevisionId=null`, zero learner visibility). Two endpoints
  (`POST /api/staff/content/topics/:topicId/import/{validate,apply}`) take the **same** versioned
  `izlan-topic-content/v1` document (no server import session); both require `content.author` + a `SubjectAssignment` for
  the Topic's **server-resolved** Subject (no ADMIN bypass, no role-name check, no client Subject), out-of-scope Topic →
  `CONTENT_NOT_FOUND` (404, IDOR-safe), `content.publish` NOT required. **apply()** = ONE all-or-nothing transaction that
  takes the destination **Subject-row `FOR UPDATE`** (same DAG serialization authority as the 2.2A-3 prerequisite writer),
  **re-runs full validation against the current DB** (dry-run never trusted), creates everything, and writes **ONE
  `content.import.apply` StaffAudit** with **safe metadata only** (subjectId/topicId/documentHash/counts — never titles/
  Markdown/payload/`answerKey`/contentKeys/raw document). `documentHash` = SHA-256 canonical serialization (correlation,
  **not** authorization). **No Prisma schema change, no migration** (migrations stay **23**, CHECK **46**). The CMS
  (`izlan/web`) adds a polished 3-step importer ("Import qilish", author-gated) — `.json` only, ≤5 MiB, **safe JSON.parse,
  no eval/HTML, no localStorage/sessionStorage/IndexedDB persistence, never renders `answerKey`**, dry-run first, apply
  re-runs the server authority with duplicate-submit protection and no fake progress. An **owner-review correction**
  (`8defe5b`, same branch, TD-253 clarified — no new TD/schema) hardened three boundaries: the **5 MiB limit is now
  import-route-only** (ordinary API back to 1 MiB, enforced at the Fastify body-parser boundary via one shared adapter
  factory); **dry-run now rejects every deterministic package-local conflict** (duplicate declared skill name, duplicate
  reference-list items, and prerequisite keys correctly following `contentKey` — not skill-code — syntax); and **apply
  persistence is batched/chunked** (`createMany`/`createManyAndReturn`, stable-key correlation, 1000-row chunks) with new
  **aggregate relationship caps** (LessonSkill 10k / ActivitySkill 25k / prerequisites 10k) rejected before the write tx.
  TD-253 added; living doc [BULK_IMPORT.md](BULK_IMPORT.md); see [checkpoints/2.2D.md](checkpoints/2.2D.md). (The English
  A1 pilot content that exercises this importer end-to-end is now built in 2.2E; ActivityMedia upload and the learner web
  app remain NOT built.)
- **Earlier:** Phase **2.2C** — Methodist CMS Web Application. Result: PASS — **merged to `main` (PR #8, `42d0b79`)**. The
  **first Izlan web app** lives at **`izlan/web`** (Next.js
  App Router + TypeScript + React + Tailwind), a professional Methodist/Admin content CMS consuming the 2.2A/2.2B staff
  APIs: subject/hierarchy/lesson/revision/activity/skill/prerequisite authoring, readiness, learner preview, and the
  review→publish→takedown workflow. **Auth:** access token **memory-only**, HttpOnly rotating refresh cookie, **single-flight
  refresh** (one refresh for concurrent 401s, retry-once, no loop), no 409/OCC auto-retry. **Authorization:** a narrow
  **`GET /api/staff/content/session`** returns capability booleans (`author`/`publish`/`subjectManage`) from effective
  permission codes — **no role-name hard-coding**; the backend stays the final authority (SubjectAssignment scope per
  mutation). **OCC:** one centralized `Revision.updatedAt` authority + per-aggregate tokens; conflicts surface a banner,
  never a silent retry. **Content safety:** canonical markdown/objective serializers; learner preview via an allowlist safe
  view model (never `answerKey`/`correctOptionIds`/`storageKey`); Markdown raw HTML disabled. **Only backend change** = the
  capability endpoint + tests; **no schema/migration**. An **owner design + language override** (same branch, in-place)
  added a 2026 Izlan visual identity (refined tokens, `next/font` Inter with Cyrillic), a reduced-motion-aware Framer-Motion
  system, a collapsible animated sidebar + ⌘K command palette, and **UI i18n (uz default / ru / en — chrome only; the
  content model is unchanged, no schema/migration)**; functional/OCC/security behavior preserved. TD-251 added (design +
  i18n folded in). A further **owner auth amendment** (TD-252, same branch) made **phone + PASSWORD the primary login**
  (OTP demoted to registration / password-reset), adding a 1:1 `PasswordCredential` (Argon2id, separate from identity) via
  **migration 23** while preserving the entire session architecture (RS256 JWT, rotating refresh + reuse detection, HttpOnly
  cookie, revoke, status checks, RBAC); login is enumeration-safe with **DB-backed / cross-process** rate limiting
  (SecurityEvent-backed, advisory-locked, HMAC phone fingerprint — no new schema); password reset is **atomic** (credential
  + revoke-all sessions/tokens + events in one transaction); staff use the same login. See
  [checkpoints/2.2C.md](checkpoints/2.2C.md), [AUTH_ARCHITECTURE.md](AUTH_ARCHITECTURE.md) (amendment banner), and the
  living [METHODIST_CMS.md](METHODIST_CMS.md). ActivityMedia upload, bulk import (→ 2.2D), and the learner web app (→ 3.x)
  are NOT built.
- **Earlier:** Phase **2.2B** — Review + Publishing + Preview + Readiness + Learner Visibility (PASS; `content.publish` +
  self-publish, top-down hierarchy publish, atomic Lesson-serialized publication + pointer switch, idempotent republish,
  canonical readiness, centralized visibility, urgent takedown gating every learner surface, `PREREQUISITE_SUBJECT_MISMATCH`).
  See [checkpoints/2.2B.md](checkpoints/2.2B.md). (Earlier: 2.2A-3 [checkpoints/2.2A-3.md](checkpoints/2.2A-3.md); 2.2A-2
  [checkpoints/2.2A-2.md](checkpoints/2.2A-2.md); 2.2A-1 [checkpoints/2.2A-1.md](checkpoints/2.2A-1.md).)
- **Telegram integration:** **architecture CANDIDATE — NOT STARTED**, not approved for implementation. Recon found the
  codebase is already identity-agnostic under the phone layer; a generic `UserIdentity` + nullable phone (Option B) is
  recommended — but there is a **cross-surface identity verification gate** (a technical external-contract fact, **NOT an
  owner decision**: OIDC `sub` is not documented as equal to the Bot/Mini App numeric `user_id`; it gates the identity
  model and must be verified before 2.2T-D), the Mini App session **transport is VERIFY-LATER** (accept the
  converge-onto-Izlan-session invariant only), and a pre-existing suspension-revocation gap.
  **12 owner decisions surfaced (none accepted)** plus those technical gates — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) §3 /
  recon §16 & §16a.
- **Content schema hardening: DONE** (2.2A-D) — `Lesson.contentKey` + prerequisite self-loop CHECK.
- **Canonical Activity registry: DONE** (2.2A-R) — one exhaustive source of truth for Lesson `ActivityType` runtime
  capability classification + one Lesson objective payload authority + a neutral shared choice-question primitive
  (AssessmentItem stays a separate versioned contract). Behavior-preserving; no schema/migration.
- **Content authoring + publication backend: END-TO-END COMPLETE for the text/objective MVP scope** (2.2A-1..3 + 2.2B) —
  authoring (hierarchy + Lesson + revision + activity + skill + prerequisite DAG, 2.2A) PLUS the **publication workflow**
  (2.2B): `content.publish` + self-publish, top-down hierarchy publish, `DRAFT→REVIEW→PUBLISHED→ARCHIVED`, atomic
  Lesson-serialized publication + pointer switch, idempotent republish, canonical publish-readiness, learner-safe preview,
  centralized learner visibility, Markdown learner projection, urgent takedown. A text/objective Lesson is genuinely
  publishable + learner-executable end-to-end. The **Methodist CMS frontend is now built (2.2C, `izlan/web`)** and drives
  this whole flow from a browser. **Still NOT built:** ActivityMedia management/upload/delivery (deferred — media identity
  relational, readiness enforced but no storage); bulk import (→ 2.2D); English A1 pilot content (→ 2.2E); learner web app
  (→ 3.x). Accepted content-lifecycle decisions are formalized as
  **TD-240..245** ([CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a); **TD-246** the 2.2A-R registry;
  **TD-247** 2.2A-1 authz/concurrency; **TD-248** 2.2A-2 draft revision/activity; **TD-249** 2.2A-3 skill mapping &
  prerequisite DAG; **TD-250** the 2.2B review/publication/visibility decision; **TD-251** the 2.2C Methodist CMS web
  architecture (`izlan/web`, memory-only token, single-flight refresh, capability endpoint, OCC save model, safe preview);
  **TD-252** phone + password primary authentication (Argon2id `PasswordCredential`, OTP → registration/reset, migration 23).
  Content track is independent of Telegram.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (phase/3.0 @ `5c772d7`; izlan `main` `b1bfd80`)
| Metric | Value |
|---|---|
| migrations | 23 (unchanged; last: `20260822120000_password_credential`, TD-252; 3.0 adds NO schema/migration) |
| backend unit tests | 505 (unchanged in 3.0) |
| backend e2e tests | 601 (3.0: +LEARNER-LANG preferredLanguage ru, +DEMO-LEARNER seed) |
| backend total tests | **1106** (505 + 601) |
| web tests (Vitest) | 92 (3.0: +40 WEB-AUTH/REG/REC/ONB/HOME/SUB/PROFILE learner tests; staff tests intact) |
| named CHECK constraints | 46 (unchanged; no schema change) |
| drift | clean (empty diff / exit 0 on izlan_dev + izlan_test) |
| learner web | `izlan/web` — public `/` landing + `/login` `/register` `/forgot-password` `/onboarding` `/learn/**`; staff `/staff/**` intact; one shared auth authority; UI uz/ru/en |
| web app | `izlan/web` — Next.js 15.5.23 · typecheck/lint clean · `next build` ok · `npm ci` reproduces |

## What is implemented (high level)
- **Auth & users** (1.1–1.4C): phone+OTP, RS256 access JWT, refresh-cookie rotation, RBAC (permission-code, no ADMIN
  bypass), roles LEARNER/METHODIST/MODERATOR/ADMIN.
- **Learning loop** (1.5–2.0): placement/diagnostic assessment, skill profile, roadmap, daily plan, lesson execution,
  lesson completion, learner signals, review candidates/sessions, mastery, daily missions, XP.
- **Finance** (2.1A–2.1L-D): IZL wallet/reservation, reward, subscription-discount redemption/commit, PaymentOrder
  purchase intent, payment execution/verification/finalization + recovery, non-success evidence, reopen/retry + recovery,
  real-provider protocol persistence hardening (Payme verified; CLICK provider-neutral shell under blocker).
- **Content schema**: full authoring/publishing lifecycle **modeled** (Subject→…→LessonRevision→Activity, publish
  pointer, revision states, skill/prereq mapping, media, subject scoping, audit). Runtime version-selection already
  matches the ideal policy.
- **Content authoring application layer: PARTIALLY IMPLEMENTED / STARTED (2.2A-1)** — staff content
  controllers/services/repositories under `/api/staff/content`; permissions `content.author` + `content.subject.manage`;
  SubjectAssignment enforcement (child content); Subject/Track/Level/Module/Topic + logical Lesson authoring; StaffAudit
  mutation wiring; `updatedAt` optimistic concurrency. Subsequent slices completed the authoring/publishing lifecycle
  (2.2A-2/3, 2.2B), the Methodist CMS frontend (2.2C), and **Topic-scoped JSON bulk content import (2.2D)**. **Still NOT
  built:** ActivityMedia upload/delivery, English A1 pilot content (→ 2.2E), and the learner web app (→ 3.x).
  The **canonical Lesson Activity registry** (2.2A-R, `src/content/activity/activity-registry.ts`) is the runtime-side
  classification authority the future authoring backend will validate writes against; view-only Activity payload shapes
  remain undefined (payloadContract = NONE_DEFINED).

## Active blockers
1. **CLICK PROTOCOL VERIFICATION BLOCKER** (gates Phase 2.1L-C): docs.click.uz Shop API detail is a client-rendered SPA;
   most constants are OFFICIAL-CORROBORATED by 2024 official repos but not raised to current-documentation authority;
   `merchant_trans_id` UUID compatibility has no positive evidence. See
   [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md) §15.

(The former "12 content lifecycle owner decisions" blocker is **RESOLVED** — decisions accepted 2026-08-21; Phase 2.2A-D
is now an implementation step, not a blocker.)

## Recommended next build step (subject to owner prompt)
After the owner accepts the Phase 3.0 learner web foundation: Phase **3.1 — Learner Lesson Player** (lesson execution /
activity attempts / completion UI over the existing runtime); then Phase **3.2 — Diagnostic/Placement UX**. Separately,
ActivityMedia management/upload/delivery remains deferred (readiness is enforced but there is no media storage). **Do NOT
start without the owner's phase prompt.**
