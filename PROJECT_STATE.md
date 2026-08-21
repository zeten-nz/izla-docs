# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests | `phase/2.2A-R` | `bd83c99a6eaeb9b21e1f42897953cbc0eeb7a890` (base `main` `2e1c9e32`) | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `phase/2.2A-R` | base `main` `fecbfad9` | clean |

\* Phase **2.2A-R** (canonical Activity registry + shared payload validation) implemented on branch `phase/2.2A-R` —
izlan base `main` @ `2e1c9e32` (which merged the 2.2A-D content-schema-hardening PR #2); docs base `main` @ `fecbfad9`
(which merged the 2.2A-D docs, PR #5). **Code SHA `bd83c99`** is this phase's implementation; izlan `main` stays
`2e1c9e32` until the PR merges. The Baseline below reflects the `phase/2.2A-R` branch state, OWNER REVIEW PENDING.

> **Governance note:** phases before 2026-08-21 were committed coarsely to `main` (izlan has only 2 commits total,
> izla-docs 3). There are **no per-phase SHAs or phase branches for historical phases** — they are all contained in
> code `19461eb` / docs `92cadce`. Per-phase SHA recording + `phase/<id>` branches begin now.

## Current position
- **Last completed:** Phase **2.2A-R** — Canonical Activity Registry + Shared Payload Validation (IMPLEMENTATION,
  behavior-preserving refactor). Result: PASS — **complete on branch `phase/2.2A-R` (code `bd83c99`), OWNER REVIEW
  PENDING (not merged)**. ONE canonical Lesson Activity registry (`src/content/activity/activity-registry.ts`) now owns
  objective/view-only/unsupported classification (was duplicated across 5+ runtime files); a neutral shared choice-question
  primitive (`src/common/payload/choice-question-payload.ts`) backs BOTH the Lesson objective and AssessmentItem placement
  parsers without collapsing the two domains. No Prisma schema/migration change; learner/scoring/completion behavior
  unchanged. TD-246 added. See [checkpoints/2.2A-R.md](checkpoints/2.2A-R.md).
  (Prior: 2.2A-D content schema hardening — [checkpoints/2.2A-D.md](checkpoints/2.2A-D.md); 2.2T-P Telegram recon —
  [TELEGRAM_INTEGRATION_RECON.md](TELEGRAM_INTEGRATION_RECON.md); 2.2A-P content recon —
  [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md).)
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
  (AssessmentItem stays a separate versioned contract). Behavior-preserving; no schema/migration. **Content authoring
  application layer is still NOT STARTED** (no CMS / authoring backend / permissions / publish workflow / write-time
  validation / full-DAG cycle service / bulk import). Accepted content-lifecycle decisions are formalized as
  **TD-240..246** ([CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a). Content track is independent of Telegram.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (phase/2.2A-R @ `bd83c99`; izlan `main` still `2e1c9e32` until the PR merges)
| Metric | Value |
|---|---|
| migrations | 22 (last: `20260821110000_content_schema_hardening`; **no new migration in 2.2A-R**) |
| unit tests | 417 (2.2A-R +20: registry AR-01..07, primitive CQ-01..12) |
| e2e tests | 436 (unchanged — refactor needed no new scenario) |
| total tests | 853 |
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
  pointer, revision states, skill/prereq mapping, media, subject scoping, audit) — **but the authoring application layer
  is not built** (no CMS/controllers/services/permissions). Runtime version-selection already matches the ideal policy.
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
Phase **2.2A** — Content Authoring Backend: subject-scoped Activity/LessonRevision CRUD + permissions + StaffAudit +
**write-time validation consuming the 2.2A-R canonical registry** (executionKind + `lesson-activity-objective/v1`) +
full-DAG prerequisite cycle prevention (transactional, building on the 2.2A-D self-loop CHECK) + `updatedAt`
optimistic-concurrency enforcement. After that: `2.2B` publishing workflow, `2.2C` CMS, `2.2D` bulk import,
`2.2E` English A1 pilot. **Do NOT start without the owner's phase prompt.**
