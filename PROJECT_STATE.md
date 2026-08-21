# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests | `phase/2.2A-1` | `abea1c48e88701b6ac4ae9af670c326976328ef6` (runtime-approved; base `main` `8ef8205`; correction on `2b03b57`; branch tip `b3d526e` = trailing comment-only reconcile, no behavior change) | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `phase/2.2A-1` | base `main` `1331b05` | clean |

\* Phase **2.2A-1** (content authoring backend — Part 1: authorization + subject scope + hierarchy + logical Lesson)
implemented on branch `phase/2.2A-1` — izlan base `main` @ `8ef8205` (which merged the 2.2A-R registry PR #3); docs base
`main` @ `1331b05` (which merged the 2.2A-R docs, PR #6). **Code SHA `abea1c4`** is this phase's implementation (owner-review
correction on `2b03b57`); izlan `main` stays `8ef8205` until the PR merges. The Baseline below reflects the `phase/2.2A-1`
branch state, OWNER REVIEW PENDING.

> **Governance note:** before the 2026-08-21 workflow adoption, historical phases were committed coarsely to `main` and
> do **not** have per-phase SHAs or phase branches — they are all contained in code `19461eb` / docs `92cadce`
> (historical authority). Per-phase branch/SHA recording begins with the adopted workflow.

## Current position
- **Last completed:** Phase **2.2A-1** — Content Authoring Backend, Part 1 (authorization + subject scope + hierarchy +
  logical Lesson). Result: PASS — **complete on branch `phase/2.2A-1` (code `abea1c4`), OWNER REVIEW PENDING (not merged)**.
  Staff-only `/api/staff/content` authoring: permissions `content.author` (author child content inside a Subject —
  requires `SubjectAssignment`) + `content.subject.manage` (global: create Subjects, **Subject metadata PATCH**, and
  manage assignments), seeded idempotently by the bootstrap (no role-name bypass); DB-resolved SubjectAssignment scope on
  every child mutation, **resolved + checked inside the mutation transaction** (IDOR-safe not-found);
  Subject/Track/Level/Module/Topic + logical Lesson CRUD subset; DRAFT-only mutation; `createdBy` always the principal;
  `Lesson.contentKey` writer immutability; **PATCH null-on-required rejected 400**; `updatedAt` optimistic concurrency;
  StaffAudit written in the same transaction as each mutation (**audit-failure rollback proven**); accepted DRAFT
  Lesson→Topic move. No Prisma schema/migration change. TD-247 added. See [checkpoints/2.2A-1.md](checkpoints/2.2A-1.md).
  (Prior: 2.2A-R Activity registry — [checkpoints/2.2A-R.md](checkpoints/2.2A-R.md); 2.2A-D content schema hardening —
  [checkpoints/2.2A-D.md](checkpoints/2.2A-D.md); 2.2T-P Telegram recon — [TELEGRAM_INTEGRATION_RECON.md](TELEGRAM_INTEGRATION_RECON.md).)
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
- **Content authoring backend: STARTED (Part 1 of 2.2A done)** (2.2A-1) — authorization + subject scope + hierarchy
  (Subject/Track/Level/Module/Topic) + logical Lesson CRUD subset, with SubjectAssignment enforcement, StaffAudit wiring,
  `updatedAt` optimistic concurrency, and `Lesson.contentKey` immutability. **Still NOT built:** LessonRevision/Activity
  authoring + payload contract closure (→ 2.2A-2); LessonSkill/ActivitySkill + prerequisite writer + full-DAG cycle
  prevention (→ 2.2A-3); publish/review workflow (→ 2.2B); CMS (→ 2.2C); bulk import (→ 2.2D). Accepted content-lifecycle
  decisions are formalized as
  **TD-240..245** ([CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a); **TD-246** formalizes the 2.2A-R
  Canonical Activity Registry decision; **TD-247** formalizes the 2.2A-1 content-authoring authorization & concurrency
  decision. Content track is independent of Telegram.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (phase/2.2A-1 @ `abea1c4`; izlan `main` still `8ef8205` until the PR merges)
| Metric | Value |
|---|---|
| migrations | 22 (last: `20260821110000_content_schema_hardening`; **no new migration in 2.2A-1**) |
| unit tests | 423 (2.2A-1 +6: content-authoring permissions/bootstrap AUTH-01..05,07) |
| e2e tests | 472 (2.2A-1 +36: content-authoring CA-01..34 + REV-01..06 + AUTH-03..06 bootstrap) |
| total tests | 895 |
| named CHECK constraints | 46 (unchanged) |
| drift | clean (empty diff on izlan_dev + izlan_test) |

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
Phase **2.2A-2** — Draft LessonRevision + Activity authoring + Activity payload contract closure: subject-scoped
LessonRevision + Activity CRUD (DRAFT-only, audited, optimistic concurrency, reusing the 2.2A-1 authorization/scope
foundation) with **write-time validation consuming the 2.2A-R canonical registry** (executionKind +
`lesson-activity-objective/v1`), and closure of the view-only Activity payload contracts. Then **2.2A-3** (LessonSkill/
ActivitySkill + prerequisite writer + full-DAG cycle prevention, building on the 2.2A-D self-loop CHECK), `2.2B`
publishing workflow, `2.2C` CMS, `2.2D` bulk import, `2.2E` English A1 pilot. **Do NOT start without the owner's phase prompt.**
