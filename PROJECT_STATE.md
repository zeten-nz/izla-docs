# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests | `phase/2.2A-3` | `beb31d5a113156a5e4c09ab4b4b3a87750cb70f9` (base `main` `d8e6b69`) | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `phase/2.2A-3` | base `main` `a11b214` | clean |

\* Phase **2.2A-3** (Skill authoring + Skill mapping + prerequisite DAG) implemented on branch `phase/2.2A-3` — izlan base
`main` @ `d8e6b69` (which merged the 2.2A-2 PR #5); docs base `main` @ `a11b214` (which merged the 2.2A-2 docs, PR #8).
**Code SHA `beb31d5`** is this phase's implementation; izlan `main` stays `d8e6b69` until the PR merges. The Baseline below
reflects the `phase/2.2A-3` branch state, OWNER REVIEW PENDING.

> **Governance note:** before the 2026-08-21 workflow adoption, historical phases were committed coarsely to `main` and
> do **not** have per-phase SHAs or phase branches — they are all contained in code `19461eb` / docs `92cadce`
> (historical authority). Per-phase branch/SHA recording begins with the adopted workflow.

## Current position
- **Last completed:** Phase **2.2A-3** — Skill authoring + Skill mapping + prerequisite DAG. Result: PASS — **complete on
  branch `phase/2.2A-3` (code `beb31d5`), OWNER REVIEW PENDING (not merged)**. Staff `/api/staff/content` now authors:
  subject-scoped **Skill** (read/create/update; no delete/archive/merge; ACTIVE-only edits; strict-monotonic OCC),
  **LessonSkill** (DRAFT logical Lesson, Lesson.updatedAt token), **ActivitySkill** (DRAFT revision, Revision.updatedAt
  token), and **LessonPrerequisite** with **transactional full-DAG cycle prevention** (whole-Subject graph + Subject-row
  `SELECT … FOR UPDATE` serialization so concurrent inverse edges cannot both commit; DB self-loop CHECK stays
  defense-in-depth). Same-Subject invariant everywhere (cross-subject → IDOR-safe 404); every mutation touches its
  aggregate + audits in one transaction; these write the EXISTING learner authorities with **no learner-runtime change**
  (roadmap/review-session suites green). No schema/migration. TD-249 added. See [checkpoints/2.2A-3.md](checkpoints/2.2A-3.md).
  (Prior: 2.2A-2 revision/activity — [checkpoints/2.2A-2.md](checkpoints/2.2A-2.md); 2.2A-1 authz/hierarchy —
  [checkpoints/2.2A-1.md](checkpoints/2.2A-1.md); 2.2A-R registry — [checkpoints/2.2A-R.md](checkpoints/2.2A-R.md).)
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
- **Content authoring backend: Phase 2.2A FUNCTIONALLY COMPLETE for the text/objective MVP scope** (2.2A-1 + 2.2A-2 +
  2.2A-3) — authorization + subject scope + hierarchy + logical Lesson (2.2A-1); DRAFT LessonRevision + Activity + closed
  objective/markdown/media payload contracts (2.2A-2); Skill + LessonSkill + ActivitySkill + LessonPrerequisite with
  transactional full-DAG cycle prevention (2.2A-3). SubjectAssignment enforcement, StaffAudit same-transaction wiring, and
  strict-monotonic aggregate optimistic concurrency throughout; learner runtime unchanged (writes the existing
  roadmap/review authorities). **Still NOT built:** review/publish/preview + publish-readiness incl. required media for
  IMAGE/AUDIO (→ 2.2B); ActivityMedia management (deferred); CMS (→ 2.2C); bulk import (→ 2.2D); pilot content (→ 2.2E).
  Accepted content-lifecycle decisions are formalized as
  **TD-240..245** ([CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a); **TD-246** formalizes the 2.2A-R
  Canonical Activity Registry decision; **TD-247** the 2.2A-1 content-authoring authorization & concurrency decision;
  **TD-248** the 2.2A-2 draft revision & activity authoring decision; **TD-249** the 2.2A-3 skill mapping & prerequisite
  DAG decision. Content track is independent of Telegram.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (phase/2.2A-3 @ `beb31d5`; izlan `main` still `d8e6b69` until the PR merges)
| Metric | Value |
|---|---|
| migrations | 22 (last: `20260821110000_content_schema_hardening`; **no new migration in 2.2A-3**) |
| unit tests | 458 (2.2A-3 +9: prerequisite-graph DAG-01..07 + hasPath + deep-chain) |
| e2e tests | 519 (2.2A-3 +20: SA3-01..23 + PR-01..09 + DAG-E1/E2/E3 + audit rollback) |
| total tests | 977 |
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
Phase **2.2B** — Review + Publishing + Preview + Readiness: revision `DRAFT→REVIEW→PUBLISHED→ARCHIVED` transitions
(`content.publish` + subject scope, no ADMIN bypass), atomic publish (publishedRevisionId pointer move + supersede +
publishedAt/By), publish-time readiness validation (incl. required-media readiness for IMAGE/AUDIO markers), preview,
idempotent republish, takedown, centralized learner-visibility gate. Then `2.2C` CMS, `2.2D` bulk import, `2.2E` English A1
pilot. **Do NOT start without the owner's phase prompt.**
