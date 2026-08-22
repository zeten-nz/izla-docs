# Izlan — English A1 Pilot Content (Phase 2.2E)

> **Mutable pilot record.** The first real educational content pack for Izlan. Implemented Phase **2.2E** on branch
> `phase/2.2E`. **TECHNICAL PASS — pedagogical owner/Methodist review pending.** No new architecture decision (latest
> remains **TD-253**). Related: [BULK_IMPORT.md](BULK_IMPORT.md), [CONTENT_MODEL.md](CONTENT_MODEL.md),
> [METHODIST_CMS.md](METHODIST_CMS.md), [checkpoints/2.2E.md](checkpoints/2.2E.md).

## Goal

Prove that the existing content model + skills + prerequisites + bulk import + review/publication workflow + learner-safe
execution can carry genuinely useful learning content — using **only** the currently supported text (Markdown) and
objective Activity contracts. This is a **pilot**: it does not freeze the full future English curriculum.

## Status (read this first)

- **Technical:** PASS. Validates through the canonical importer (`npm run content:pilot:a1:validate` → VALID) and imports
  + publishes structurally in the test database via the existing workflow with zero readiness blockers.
- **Pedagogical:** **AI-ASSISTED DRAFT — OWNER / METHODIST REVIEW PENDING.** Not pedagogically approved, not
  "CEFR certified", not "production content finalized". A human must review the actual lessons before real import/publish.
- **Provenance:** AI-assisted authoring drafts. Import v1 records `ContentSource = HUMAN` (the importing staff member
  takes responsibility); v1 accepts no provenance field and real publication is always a manual human step after review.
  This is consistent with TD-20 (`ContentSource`) because nothing auto-publishes — see *Provenance* below.

## Languages

- **Teaching / explanation:** Uzbek (Latin). Learner-facing instructional text is natural Uzbek, not literal translation.
- **Target:** English. Every English sentence self-reviewed for A1-appropriate grammar/spelling/naturalness.

Content language ≠ UI i18n. No Russian educational-content copy; no localized content DB fields.

## Hierarchy (pilot only)

| Level | Name |
|---|---|
| Subject | English |
| Track | General English |
| Level | A1 |
| Module | A1 Foundations |
| Topic 1 | Tanishuv va asosiy gaplar |
| Topic 2 | Savollar va shaxsiy ma'lumot |
| Topic 3 | Oila va egalik |
| Topic 4 | Kundalik hayot |

The bulk importer does **not** create this hierarchy — the Methodist creates it in the CMS, then imports each Topic
package into its Topic.

## Topic packages & import order

Import order matters (later packages reference earlier lessons):

| # | File | Topic | Lessons |
|---|---|---|---|
| 1 | `01-introductions-and-be.json` | Tanishuv va asosiy gaplar | 001–003 |
| 2 | `02-personal-information.json` | Savollar va shaxsiy ma'lumot | 004–006 |
| 3 | `03-family-and-possession.json` | Oila va egalik | 007–009 |
| 4 | `04-daily-routines.json` | Kundalik hayot | 010–012 |

`manifest.json` is a repository-level manifest (humans / tests / import order) — never sent to the import endpoint.

## Lessons (12)

| # | contentKey | Title | Primary skill | Activities | ~min |
|---|---|---|---|---|---|
| 01 | ENG-A1-001-GREETINGS | Salomlashish va tanishuv | ENG-A1-GREETINGS | 8 | 14 |
| 02 | ENG-A1-002-SUBJECT-PRONOUNS | Kishilik olmoshlari | ENG-A1-SUBJECT-PRONOUNS | 8 | 12 |
| 03 | ENG-A1-003-BE-AFFIRMATIVE | To be: am, is, are | ENG-A1-BE-AFFIRMATIVE | 8 | 14 |
| 04 | ENG-A1-004-BE-NEGATIVE | To be: inkor shakli | ENG-A1-BE-NEGATIVE | 8 | 13 |
| 05 | ENG-A1-005-BE-QUESTIONS | To be bilan savollar | ENG-A1-BE-QUESTIONS | 8 | 14 |
| 06 | ENG-A1-006-NUMBERS-PERSONAL-INFO | Sonlar va shaxsiy ma'lumot | ENG-A1-NUMBERS / ENG-A1-PERSONAL-INFO | 8 | 15 |
| 07 | ENG-A1-007-POSSESSIVE-ADJECTIVES | Egalik sifatlari | ENG-A1-POSSESSIVE-ADJECTIVES | 8 | 14 |
| 08 | ENG-A1-008-FAMILY | Oila a'zolari | ENG-A1-FAMILY-VOCAB | 8 | 13 |
| 09 | ENG-A1-009-HAVE-HAS | Have va has | ENG-A1-HAVE-HAS | 8 | 13 |
| 10 | ENG-A1-010-PRESENT-SIMPLE-AFFIRMATIVE | Present Simple: tasdiq gaplar | ENG-A1-PRESENT-SIMPLE-AFFIRMATIVE | 8 | 15 |
| 11 | ENG-A1-011-PRESENT-SIMPLE-NEGATIVE | Present Simple: inkor gaplar | ENG-A1-PRESENT-SIMPLE-NEGATIVE | 8 | 15 |
| 12 | ENG-A1-012-PRESENT-SIMPLE-QUESTIONS | Present Simple: savollar va yakuniy takrorlash | ENG-A1-PRESENT-SIMPLE-QUESTIONS | 8 | 15 |

**96 activities total (48 objective).** Estimated total learning time ≈ **165 minutes**. contentKeys are immutable
business/import identities.

## Skills (13, Subject-scoped codes)

`ENG-A1-GREETINGS`, `ENG-A1-SUBJECT-PRONOUNS`, `ENG-A1-BE-AFFIRMATIVE`, `ENG-A1-BE-NEGATIVE`, `ENG-A1-BE-QUESTIONS`,
`ENG-A1-NUMBERS`, `ENG-A1-PERSONAL-INFO`, `ENG-A1-POSSESSIVE-ADJECTIVES`, `ENG-A1-FAMILY-VOCAB`, `ENG-A1-HAVE-HAS`,
`ENG-A1-PRESENT-SIMPLE-AFFIRMATIVE`, `ENG-A1-PRESENT-SIMPLE-NEGATIVE`, `ENG-A1-PRESENT-SIMPLE-QUESTIONS`.

A skill code declared in an earlier package is redeclared with the **identical** name in a later package where a lesson
retrieves it; the importer reuses the existing ACTIVE skill (never a name conflict). Every Lesson maps ≥1 primary skill;
later lessons map one or two prior skills only where they genuinely retrieve them.

## Prerequisite chain

Linear: `001 → 002 → 003 → 004 → 005 → 006 → 007 → 008 → 009 → 010 → 011 → 012`. Lesson 001 has no prerequisite. Direction
is Lesson → prerequisite Lesson. No forward references; cross-Topic prerequisites resolve to already-imported lessons,
which is why Topics are imported in order.

## Pedagogical template

Each Lesson has 8 activities following (adapted where useful):

1. `TEXT` — learning goal / orientation
2. `EXPLANATION` — concise Uzbek explanation
3. `EXAMPLE` — natural English examples
4. `MINI_QUESTION` — immediate comprehension check
5. `EXPLANATION` — common-mistake contrast (`To'g'ri:` / `Noto'g'ri:`)
6. `PRACTICE` — objective application
7. `PRACTICE` — a second context
8. `MASTERY_TEST` — independent check in a fresh context

Lesson 12 shifts two of these to additional `MASTERY_TEST` activities for a cumulative present-simple review plus a
Markdown recap of the whole pilot. Objective payloads use `single_choice` with exactly one defensible answer and
plausible beginner-mistake distractors; the objective schema has no feedback field, so explanations live in separate
`EXPLANATION` activities. Durations: TEXT 1–2, EXPLANATION 2–4, EXAMPLE 1–3, MINI_QUESTION 1, PRACTICE 1–2,
MASTERY_TEST 1–2 → ~12–15 min/lesson.

## Activity types & safety

- Markdown (`lesson-activity-markdown/v1`): `TEXT` / `EXPLANATION` / `EXAMPLE` — restricted Markdown, **no raw HTML**.
- Objective (`lesson-activity-objective/v1`): `MINI_QUESTION` / `PRACTICE` / `MASTERY_TEST`.
- **No** media (`IMAGE`/`AUDIO`/`VIDEO`), `SPEAKING`, `WRITING`, `LISTENING`, `AI_INTERACTION`.

The JSON files carry server-only `answerKey` values: they are **authoring source files, not learner delivery files** —
never copied to `web/public`, never imported by frontend source, never served/logged, never referenced by browser code.
The learner runtime strips `answerKey`/`correctOptionIds` via the canonical learner projection before anything reaches a
client (verified by the e2e learner-safe smoke).

## Technical validation

- `npm run content:pilot:a1:validate` → **VALID** (Topics 4, Lessons 12, Activities 96 / objective 48, Skills 13). Uses
  the EXISTING `parseImportDocument` plus cross-file pilot invariants (`src/content-import/pilot/english-a1-pilot.ts`).
- Unit **PILOT-01..10** (+ aggregate) — manifest shape, zero structural issues, unique contentKeys, skill-name
  consistency, exact prereq chain, per-lesson coverage, objective skill mapping, supported types, no raw HTML, no
  answerKey in Markdown.
- e2e **PILOT-E2E-01** — real import of all 4 packages: 12 DRAFT lessons across 4 Topics, **4** `content.import.apply`
  audits (one per Topic), 13 skills reused, LessonSkill/ActivitySkill present, contiguous activity positions, exact
  prereq chain. **PILOT-IMPORT-SAFETY** — validate/apply responses contain no answerKey/correctOptionIds/Markdown bodies.

## Publication smoke result (test DB only)

- **PILOT-PUBLISH-01**: hierarchy published top-down (Subject→Track→Level→Module→4 Topics), then all 12 lessons published
  in prerequisite order `001→012` via the real `submit-review → readiness → publish` workflow. Every lesson reached
  `readiness.publishReady = true` with **zero blockers**; every Lesson ended `PUBLISHED` with the correct
  `publishedRevisionId` pointer and a `PUBLISHED` current revision. **Readiness was not weakened** to pass content.
- **PILOT-LEARNER-01**: a published lesson's activities project through the canonical learner projector with no
  `answerKey`/`correctOptionIds`/authoring metadata; objective prompt+options and Markdown are visible.
- **PILOT-SCORING-01**: a real pilot objective scores deterministically — correct = 10000, wrong = 0, no AI.

This publication happens **only** in `izlan_test`. Nothing auto-imports into dev/prod; `db:seed:system` / `db:seed:demo`
are unchanged.

## Provenance

The pilot files are AI-assisted drafts. The v1 importer records `ContentSource = HUMAN` and accepts no provenance field
(v1 intentionally supports only markdown + objective, no `source`/AI fields). This does **not** conflict with TD-20
(`ContentSource` — no publish without review) because in this pipeline a human staff member imports the content and a
human Methodist reviews and publishes it manually; nothing is auto-published. Real import/publication of this pilot
**requires** human review first. No `CONTENT_PROVENANCE_GAP`.

## Owner / Methodist review checklist

For each of the 12 lessons, confirm: one clear objective; explanation precedes independent mastery; Uzbek is natural and
concise; English is grammatically correct and A1-appropriate; vocabulary is not overloaded; objective questions are
unambiguous with plausible distractors; mastery uses a fresh context (not an exact repeat); skill mappings are accurate;
the prerequisite is justified; durations are credible. Wrong English must appear **only** as an intentional distractor or
an explicitly-labelled `Noto'g'ri` contrast — never as correct teaching text.

## Known limitations

No audio, no speaking, no writing, no listening, no images/video, no AI tutor, no learner web app yet. Placement/
checkpoint assessment content is out of scope. This pilot does not freeze the full A1 curriculum.
