# Izlan — Phase History (append-only index)

> Chronological index of phases. **Authority map:** this file = the **concise historical index**; the historical
> `*_IMPLEMENTATION.md` docs = the **detailed per-phase records**; [checkpoints/](checkpoints/) = **immutable SHA-bearing
> checkpoints from the new workflow (2026-08-21) onward**; [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) = **unresolved
> decisions only** (the old per-phase "HOLATI" blocks were removed from OPEN_QUESTIONS on 2026-08-21 — they no longer
> exist there). Phases before 2026-08-21 were committed coarsely to `main` and have **no per-phase SHA** — all are
> contained in code `19461eb`. Test counts are as reported at each phase; treat counts as informational, not a quality
> target.

## SHA-recorded phases (new workflow, 2026-08-21+)
| Phase | Title | Result | Mig | Unit | E2E | Code SHA | Docs SHA | Checkpoint |
|---|---|---|---|---|---|---|---|---|
| 2.2T-P | Telegram integration architecture recon (NO CODE) | PASS w/ gaps | 21 | 397 | 432 | `281ca415` | `phase/2.2T-P` (final SHA in checkpoint/PR) | [2.2T-P.md](checkpoints/2.2T-P.md) |
| 2.2A-D | Content lifecycle / schema hardening (Lesson.contentKey + prereq self-loop CHECK) | PASS | 22 | 397 | 436 | `7dec7bff` | `phase/2.2A-D` (SHA in checkpoint/PR) | [2.2A-D.md](checkpoints/2.2A-D.md) |
| 2.2A-R | Canonical Activity registry + shared payload validation (behavior-preserving refactor; no schema) | PASS | 22 | 417 | 436 | `bd83c99` | `phase/2.2A-R` (SHA in checkpoint/PR) | [2.2A-R.md](checkpoints/2.2A-R.md) |
| _next_ | (awaiting owner phase prompt) | — | — | — | — | — | — | — |

## Historical phases (pre-per-phase-SHA — all in code `19461eb`, docs `92cadce`)
| Phase | Title | Mig | Unit | E2E | TDs |
|---|---|---|---|---|---|
| 1.1–1.2C-2 | Auth architecture + data-model reviews (core/learning/finance/community) | — | — | — | TD-01..80, IN-01..06 |
| 1.3 | Prisma Schema v1 (`20260819100830_init`) — 83 model, 49 enum | 1 | 21 verif | — | — |
| 1.4A/B/C | Backend foundation; auth core; auth HTTP + access JWT + web security | 1 | 62 | 46 | IN-07..26 |
| 1.5A/A-2 | Learner onboarding + profile; learning-intent persistence | 2 | 77 | 62 | TD-93 |
| 1.5B/B-2 | Placement/diagnostic assessment foundation + adaptive contract hardening | 3 | 130 | 82 | TD-94/95/96, IN-32..45 |
| 1.5C… | Skill measurement derivation + versioning | 4–5 | — | — | TD-98/113 |
| 1.6/1.7/1.8/1.9 | Roadmap; daily plan; lesson execution/completion; learner signals; review candidates/sessions; mastery; learning-progress merge | 6–10 | — | — | TD-108..130 |
| 2.0B/C-2/D | Daily missions; XP mission reward + provenance; XP progression projection | 11–13 | — | — | TD-136/141/146 |
| 2.1A/B | IZL reward; IZL wallet projection + reservation safety | 13–14 | 355 | 288 | TD-156..160 |
| 2.1C-PO/C-2/D | PaymentOrder purchase intent; redemption intent; discount commit | 15–17 | 361 | 310 | TD-167..182 |
| 2.1E | Payment execution foundation | 18 | — | — | TD-183..188 |
| 2.1F | Verified payment evidence (PENDING→SUCCEEDED; order stays PENDING) | 19 | 361 | 345 | TD-189..194 |
| 2.1G-D | Finalization contract + schema hardening | 20 | 378 | 351 | TD-195..204 |
| 2.1G | Verified payment economic finalization (atomic PAID+Subscription+Cycle+REDEEM) | 20 | 378 | 369 | TD-205..210 |
| 2.1H | Finalization recovery / reconciliation | 20 | 384 | 378 | TD-211..215 |
| 2.1I | Verified non-success evidence (PENDING→FAILED/CANCELLED) | 20 | 384 | 391 | TD-216..221 |
| 2.1J | Payment order reopen / retry foundation | 20 | 386 | 405 | TD-222..227 |
| 2.1K | Terminal payment reopen recovery / reconciliation | 20 | 390 | 414 | TD-228..232 |
| 2.1L-D | Real provider contract / persistence hardening (Payme verified; CLICK shell/blocker) | 21 | 397 | 432 | TD-233..239 |
| 2.1L-C0 | CLICK protocol verification closure (NO CODE) — blocker NOT closed | 21 | 397 | 432 | — |
| 2.2A-P | Content authoring/publishing/methodist workflow recon (NO CODE) | 21 | 397 | 432 | — |

## Notes
- **Real-provider implementation track PAUSED** after 2.1L-D / 2.1L-C0 (no merchant application / docs / sandbox /
  credentials). The completed payment architecture is intact and untouched.
- The **CLICK PROTOCOL VERIFICATION BLOCKER is NOT closed**; **2.1L-C (real CLICK integration) has not started** and will
  not begin until the blocker is cleared and the track resumes (see PROJECT_STATE).
- **2026-08-21 cleanup:** the per-phase "HOLATI" status blocks were **removed from OPEN_QUESTIONS.md** and consolidated
  into this history (the table above is their concise record; full per-phase detail lives in the `*_IMPLEMENTATION.md`
  docs). OPEN_QUESTIONS now holds only genuinely unresolved owner decisions. Accepted content-lifecycle decisions moved
  to [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md) §13a. Historical phases remain `pre-per-phase-SHA`
  (all in code `19461eb`); no old SHA was fabricated. Complete SHA-bearing checkpoints (from the next phase on) are not
  duplicated here — they live in [checkpoints/](checkpoints/).
- **Content-lifecycle decisions ACCEPTED** 2026-08-21 (§13a of the recon doc); to be formalized as TDs at Phase 2.2A-D.
- **Telegram** is an architecture candidate only (not approved). **Phase 2.2T-P recon COMPLETE** (2026-08-21, code
  inspected @ `281ca415`, runtime unchanged) — see [TELEGRAM_INTEGRATION_RECON.md](TELEGRAM_INTEGRATION_RECON.md) +
  [checkpoints/2.2T-P.md](checkpoints/2.2T-P.md); **12 owner decisions** in [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) §3, plus
  **technical verification gates** (cross-surface identity key; Mini App transport VERIFY-LATER) in the recon doc §16a.
- **Phase 2.2A-D COMPLETE on branch** (2026-08-21, code `7dec7bff`): content schema hardening — `Lesson.contentKey`
  (NOT NULL + UNIQUE) + `lesson_prerequisite` self-loop CHECK; migration 22, CHECK 46, e2e 436. Accepted content
  decisions formalized as **TD-240..245**. Content authoring application layer still NOT STARTED (next: 2.2A-R).
  **Phase 2.2A-D (content schema hardening) can proceed independently of Telegram.** *(2.2A-D merged to `main`:
  izlan `2e1c9e32`, izla-docs `fecbfad9`.)*
- **Phase 2.2A-R COMPLETE on branch** (2026-08-21, code `bd83c99`): canonical Lesson Activity registry
  (`src/content/activity/activity-registry.ts`) is the one source of truth for objective/view-only/unsupported
  classification (was duplicated across the payload parser, completion eligibility, daily-mission policy/repo,
  learner-signals, review-session); a neutral shared choice-question primitive
  (`src/common/payload/choice-question-payload.ts`) backs BOTH the Lesson objective and AssessmentItem placement
  parsers **without collapsing the two domains**. Behavior-preserving: **no schema/migration** (migrations 22, CHECK 46);
  unit 397→417 (+20), e2e 436 unchanged. **TD-246**. Authoring backend still NOT STARTED (next: 2.2A).
