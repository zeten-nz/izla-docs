# Izlan — Project State (current)

> **Mutable.** Represents only the current state and changes every phase. Historical records live in
> [PHASE_HISTORY.md](PHASE_HISTORY.md) and [checkpoints/](checkpoints/). Adopted 2026-08-21.

## Repository pointers (verified 2026-08-21)
| Repo | Role | Branch | HEAD SHA (at verification) | Working tree |
|---|---|---|---|---|
| `zeten-nz/izlan` | code / schema / migrations / tests | `main` | `281ca4159e3bfe08ce9bb1e6a26f865f04cd5017` | clean |
| `zeten-nz/izla-docs` | product/architecture decisions, checkpoints | `main` | `db8dfdbc92e66d2c73952cb92797a3e2cbb19a5d` | clean |

\* Inspected for the **2.2T-P Telegram recon**: **izlan `main` @ `281ca415`** — this differs from `19461eb` only by
`CLAUDE.md` (workflow rules, PR #1 merged), so the **runtime code/schema/tests are unchanged** and `281ca415` is the
current verified code↔docs match. This recon runs on docs branch `phase/2.2T-P` (off docs `main` @ `db8dfdbc`, which
merged the OPEN_QUESTIONS cleanup, PR #3) and advances docs `main` after merge.

> **Governance note:** phases before 2026-08-21 were committed coarsely to `main` (izlan has only 2 commits total,
> izla-docs 3). There are **no per-phase SHAs or phase branches for historical phases** — they are all contained in
> code `19461eb` / docs `92cadce`. Per-phase SHA recording + `phase/<id>` branches begin now.

## Current position
- **Last completed:** Phase **2.2T-P** — Telegram Integration Architecture Reconnaissance (NO CODE). Result: PASS WITH
  ARCHITECTURE GAPS. See [TELEGRAM_INTEGRATION_RECON.md](TELEGRAM_INTEGRATION_RECON.md) + [checkpoints/2.2T-P.md](checkpoints/2.2T-P.md).
  (Prior recon: 2.2A-P content authoring — [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md).)
- **Telegram integration:** **architecture CANDIDATE — NOT STARTED**, not approved for implementation. Recon found the
  codebase is already identity-agnostic under the phone layer; a generic `UserIdentity` + nullable phone (Option B) is
  recommended, plus a same-site refresh-cookie limitation for Mini Apps and a pre-existing suspension-revocation gap.
  **12 owner decisions surfaced (none accepted)** — [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) §3 / recon §16.
- **Content implementation: NOT STARTED** — the schema models the full authoring/publishing lifecycle, but no CMS /
  authoring backend / content permissions exist yet. The content-lifecycle decisions are **ACCEPTED** (2026-08-21,
  [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a); **Phase 2.2A-D can proceed independently of Telegram.**
- **Payment provider track:** **PAUSED** (no CLICK/Payme merchant application, merchant docs, sandbox, or test
  credentials). Completed payment architecture is intact and must not be modified. (Telegram Stars is a *future*
  PaymentProvider behind the existing boundary — it does not resume the CLICK/Payme track.)
- **Workflow:** the two-repo phase/checkpoint/SHA workflow is adopted (rules in `izlan/CLAUDE.md`).
- **No future phase is marked complete.** No implementation phase starts until the owner supplies its specific prompt.

## Baseline (izlan @ 281ca415 — runtime unchanged from 19461eb; only CLAUDE.md added)
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
Phase **2.2A-D** — Content Lifecycle / Schema Hardening (**minimal, schema-only**): exactly two schema changes — a
**LessonPrerequisite self-loop CHECK** (`lesson_id <> prerequisite_lesson_id`) and an **immutable stable Lesson
`contentKey`** (import/business identity; title is not identity — Lesson **slug** stays a separate routing/SEO concern,
NOT content identity and NOT part of this hardening). Service-layer enforcement of the accepted contracts —
**full-DAG cycle prevention** and **`updatedAt` optimistic-concurrency** — belongs to **Phase 2.2A** (Content Authoring
Backend), not 2.2A-D; **bulk-import format/infrastructure** is **Phase 2.2D**. Decisions are accepted (§13a); **do NOT
start without the owner's phase prompt.**
