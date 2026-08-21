# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests | `phase/2.2A-D` | `7dec7bff4880fefbbb9698fd222b174a702977bd` (base `main` `281ca415`) | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `phase/2.2A-D` | base `main` `86ebc313` | clean |

\* Phase **2.2A-D** (content schema hardening) implemented on branch `phase/2.2A-D` — izlan base `main` @ `281ca415`;
docs base `main` @ `86ebc313` (which merged the 2.2T-P recon, PR #4). **Code SHA `7dec7bff`** is this phase's
implementation; izlan `main` stays `281ca415` until the PR merges. The Baseline below reflects the `phase/2.2A-D` branch
state, OWNER REVIEW PENDING.

> **Governance note:** phases before 2026-08-21 were committed coarsely to `main` (izlan has only 2 commits total,
> izla-docs 3). There are **no per-phase SHAs or phase branches for historical phases** — they are all contained in
> code `19461eb` / docs `92cadce`. Per-phase SHA recording + `phase/<id>` branches begin now.

## Current position
- **Last completed:** Phase **2.2A-D** — Content Lifecycle / Schema Hardening (IMPLEMENTATION). Result: PASS —
  **complete on branch `phase/2.2A-D` (code `7dec7bff`), OWNER REVIEW PENDING (not merged)**. Two schema changes:
  `Lesson.contentKey` (immutable business/import identity, NOT NULL + UNIQUE) + `lesson_prerequisite` self-loop CHECK
  (`chk_lesson_prerequisite_no_self_loop`). See [checkpoints/2.2A-D.md](checkpoints/2.2A-D.md). TDs 240–245 formalized.
  (Prior recon: 2.2T-P Telegram — [TELEGRAM_INTEGRATION_RECON.md](TELEGRAM_INTEGRATION_RECON.md); 2.2A-P content —
  [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md).)
- **Telegram integration:** **architecture CANDIDATE — NOT STARTED**, not approved for implementation. Recon found the
  codebase is already identity-agnostic under the phone layer; a generic `UserIdentity` + nullable phone (Option B) is
  recommended — but there is a **cross-surface identity verification gate** (a technical external-contract fact, **NOT an
  owner decision**: OIDC `sub` is not documented as equal to the Bot/Mini App numeric `user_id`; it gates the identity
  model and must be verified before 2.2T-D), the Mini App session **transport is VERIFY-LATER** (accept the
  converge-onto-Izlan-session invariant only), and a pre-existing suspension-revocation gap.
  **12 owner decisions surfaced (none accepted)** plus those technical gates — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) §3 /
  recon §16 & §16a.
- **Content schema hardening: DONE** (2.2A-D) — `Lesson.contentKey` + prerequisite self-loop CHECK. **Content authoring
  application layer is still NOT STARTED** (no CMS / authoring backend / permissions / publish workflow / Activity
  registry / import). Accepted content-lifecycle decisions are formalized as **TD-240..245**
  ([CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a). Content track was independent of Telegram.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (phase/2.2A-D @ `7dec7bff`; izlan `main` still `281ca415` until the PR merges)
| Metric | Value |
|---|---|
| migrations | 22 (last: `20260821110000_content_schema_hardening`) |
| unit tests | 397 |
| e2e tests | 436 |
| total tests | 833 |
| named CHECK constraints | 46 |
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

## Active blockers
1. **CLICK PROTOCOL VERIFICATION BLOCKER** (gates Phase 2.1L-C): docs.click.uz Shop API detail is a client-rendered SPA;
   most constants are OFFICIAL-CORROBORATED by 2024 official repos but not raised to current-documentation authority;
   `merchant_trans_id` UUID compatibility has no positive evidence. See
   [REAL_PROVIDER_CONTRACT_HARDENING.md](REAL_PROVIDER_CONTRACT_HARDENING.md) §15.

(The former "12 content lifecycle owner decisions" blocker is **RESOLVED** — decisions accepted 2026-08-21; Phase 2.2A-D
is now an implementation step, not a blocker.)

## Recommended next build step (subject to owner prompt)
Phase **2.2A-R** — Canonical Activity Registry + Shared Payload Validator: a single source of truth
(type → schema → scoring → renderer flags) replacing the runtime-only, duplicated objective-activity parsers/sets,
usable by both learner runtime AND the future authoring layer (behavior-preserving refactor). After that: `2.2A`
Content Authoring Backend (subject-scoped CRUD + permissions + StaffAudit + write-time validation + full-DAG cycle
prevention + `updatedAt` optimistic-concurrency enforcement), `2.2B` publishing workflow, `2.2C` CMS, `2.2D` bulk
import, `2.2E` English A1 pilot. **Do NOT start without the owner's phase prompt.**
