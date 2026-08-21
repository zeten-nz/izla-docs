# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests | `main` | `19461eb236b20829c226e6931b96d3032b65027` | clean* |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `main` | `188c5f74ca81d6f39573a85d0ea8d4f03981d59b` | clean* |

\* izla-docs `main` @ `188c5f7` merged the workflow-governance scaffolding (PR #1). This OPEN_QUESTIONS cleanup runs on
branch `chore/open-questions-cleanup` and will advance docs `main` again after merge. **The code SHA `19461eb` is the
implementation inspected for the 2.2A-P recon** (it already contains all of 2.1E→2.1L-D) and is unchanged by this
docs-only cleanup. Per the code↔docs SHA rule, `19461eb` is the last verified code↔docs match.

> **Governance note:** phases before 2026-08-21 were committed coarsely to `main` (izlan has only 2 commits total,
> izla-docs 3). There are **no per-phase SHAs or phase branches for historical phases** — they are all contained in
> code `19461eb` / docs `92cadce`. Per-phase SHA recording + `phase/<id>` branches begin now.

## Current position
- **Last completed:** Phase **2.2A-P** — Content Authoring / Publishing / Methodist Workflow Reconnaissance (NO CODE).
  Result: PASS WITH ARCHITECTURE GAPS. See [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md).
- **Content implementation: NOT STARTED** — the schema models the full authoring/publishing lifecycle, but no CMS /
  authoring backend / content permissions exist yet. The content-lifecycle decisions are now **ACCEPTED** (2026-08-21;
  see [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a), to be formalized as TDs when Phase 2.2A-D is built.
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified.
- **Telegram integration:** **architecture CANDIDATE only** — not approved for implementation; open decisions tracked in
  [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) §3.
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (izlan @ 19461eb)
| Metric | Value |
|---|---|
| migrations | 21 (last: `20260821100000_real_provider_protocol_persistence`) |
| unit tests | 397 |
| e2e tests | 432 |
| total tests | 829 |
| named CHECK constraints | 45 |
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
Phase **2.2A-D** — Content Lifecycle / Schema Hardening: prerequisite **self-loop CHECK (DB) + full-DAG cycle validation
at service/transaction level** (a DB CHECK cannot enforce a multi-node cycle), immutable stable Lesson `contentKey` +
slug uniqueness, `updatedAt` optimistic-concurrency, and the accepted revision-lifecycle model (`DRAFT→REVIEW→PUBLISHED→
ARCHIVED`, no SUPERSEDED/REJECTED enum). Decisions are accepted (§13a); **do NOT start without the owner's phase prompt.**
