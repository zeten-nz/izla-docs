# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests + `web/` | `phase/2.2C` | `57530d5809831ef60fe338893b29ad973a08b3f3` (base `main` `14a6d5c`) | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `phase/2.2C` | `phase/2.2C` (final SHA in PR) | clean |

\* Phase **2.2B** (review + publishing + preview + readiness + learner visibility) implemented on branch `phase/2.2B` —
izlan base `main` @ `9ebd90a` (which merged the 2.2A-3 PR #6); docs base `main` @ `4b8b89f` (which merged the 2.2A-3 docs,
PR #9). **Code SHA `85ddb10`** is this phase's implementation; izlan `main` stays `9ebd90a` until the PR merges. The
Baseline below reflects the `phase/2.2B` branch state, OWNER REVIEW PENDING.

> **Governance note:** before the 2026-08-21 workflow adoption, historical phases were committed coarsely to `main` and
> do **not** have per-phase SHAs or phase branches — they are all contained in code `19461eb` / docs `92cadce`
> (historical authority). Per-phase branch/SHA recording begins with the adopted workflow.

## Current position
- **Last completed:** Phase **2.2C** — Methodist CMS Web Application. Result: PASS — **complete on branch `phase/2.2C`
  (code `57530d5`), OWNER REVIEW PENDING (not merged)**. The **first Izlan web app** lives at **`izlan/web`** (Next.js
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
  i18n folded in). See [checkpoints/2.2C.md](checkpoints/2.2C.md) and the living [METHODIST_CMS.md](METHODIST_CMS.md).
  ActivityMedia upload, bulk import (→ 2.2D), and the learner web app (→ 3.x) are NOT built.
- **Prior:** Phase **2.2B** — Review + Publishing + Preview + Readiness + Learner Visibility (PASS; `content.publish` +
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
  architecture (`izlan/web`, memory-only token, single-flight refresh, capability endpoint, OCC save model, safe preview).
  Content track is independent of Telegram.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (phase/2.2C @ `57530d5`; izlan `main` `14a6d5c` until the PR merges)
| Metric | Value |
|---|---|
| migrations | 22 (last: `20260821110000_content_schema_hardening`; **no new migration in 2.2B or 2.2C**) |
| backend unit tests | 474 (unchanged in 2.2C) |
| backend e2e tests | 547 (2.2C +5: CMS-SESSION-01..05 capability endpoint) |
| backend total tests | **1021** (474 + 547) |
| web tests (Vitest) | 33 (WEB-01..14 auth/OCC/serializers/preview/workflow + I18N-01..06 locale/chrome + UI-A11Y-01..05 modal focus/reorder) |
| named CHECK constraints | 46 (unchanged) |
| drift | clean (empty diff on izlan_dev + izlan_test) |
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
  mutation wiring; `updatedAt` optimistic concurrency. **Still NOT built:** LessonRevision authoring, Activity authoring +
  write-time payload validation, Skill-mapping writers, prerequisite writer / full-DAG validation, review/publishing, CMS
  frontend, bulk import.
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
Phase **2.2C** — Methodist CMS (frontend consuming the 2.2A/2.2B staff APIs: hierarchy/lesson/revision/activity/skill/
prerequisite authoring + review/publish/preview/readiness/takedown). Then `2.2D` bulk import, `2.2E` English A1 pilot
content; separately, ActivityMedia management/upload/delivery remains deferred (readiness is enforced but there is no media
storage). **Do NOT start without the owner's phase prompt.**
