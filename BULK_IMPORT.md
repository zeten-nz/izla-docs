# Izlan — Bulk Content Import (living doc)

> **Mutable living doc** for the Topic-scoped JSON bulk content import pipeline (introduced Phase 2.2D, **TD-253**).
> The *decision* is [TD-253](TECH_DECISIONS.md); the phase record is [checkpoints/2.2D.md](checkpoints/2.2D.md). Keep this
> current as import evolves. Related: [CONTENT_MODEL.md](CONTENT_MODEL.md), [METHODIST_CMS.md](METHODIST_CMS.md),
> [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md), [JSONB_GOVERNANCE.md](JSONB_GOVERNANCE.md).

## What it is

A production-safe pipeline that lets a Methodist load **many lessons at once** into an **existing** Topic from a local
JSON file, as **DRAFT** content — with the exact same authorization, lifecycle, payload, and prerequisite-DAG invariants
the single-item authoring path (2.2A/2.2B) already enforces. It is bulk *creation only*: it never publishes, overwrites,
updates, or deletes, and it adds no learner-visible content.

Flow: **local JSON → strict validation → dry-run plan → human confirmation → atomic DRAFT import.**

## Scope

**Imported authorities:** Skills, Lessons + one initial `LessonRevision` (v1), Activities, `LessonSkill`, `ActivitySkill`,
`LessonPrerequisite`.

**Hard out of scope (v1):** Subject/Track/Level/Module/Topic creation; CSV/Excel/ZIP; MediaAsset/ActivityMedia/image/audio;
remote URLs; auto-publication or REVIEW transitions; bulk update/overwrite/delete; AI imports; background queue/Redis;
import-history/import-batch table; any learner surface. Everything imports as DRAFT (`publishedRevisionId=null`).

**No schema change, no migration.** Reuses existing tables + domain/repo primitives. Migrations stay **23**, named CHECK
constraints stay **46**.

## Document format — `izlan-topic-content/v1`

```jsonc
{
  "schemaVersion": "izlan-topic-content/v1",   // exact match; unknown → IMPORT_SCHEMA_UNSUPPORTED
  "provenance": { "source": "AI_ASSISTED" },   // OPTIONAL (TD-254); omitted → HUMAN. Every Activity inherits it.
  "skills": [
    { "code": "DEMO-BE", "name": "…", "description": "…", "sortOrder": 0 }
  ],
  "lessons": [
    {
      "contentKey": "DEMO-A1-0001",             // create-only, GLOBAL unique
      "slug": "optional-slug",
      "sortOrder": 0,
      "skillCodes": ["DEMO-BE"],                // → LessonSkill (same-Subject, ACTIVE)
      "prerequisiteContentKeys": [],            // → LessonPrerequisite (new-in-doc or existing same-Subject)
      "revision": {
        "title": "…", "description": "…",
        "activities": [                          // NO position field — order = array index 0..N-1
          { "type": "EXPLANATION", "payload": { /* markdown/objective contract */ },
            "estimatedDurationMin": 3, "skillCodes": ["DEMO-BE"] }  // → ActivitySkill
        ]
      }
    }
  ]
}
```

**Strict shape.** Unknown top-level *and* nested fields, wrong types, duplicate `contentKey`s, duplicate skill `code`s, and
malformed payloads are rejected. **Server-owned fields are forbidden** in the document: `id`, `topicId`, `status`,
`createdBy`, timestamps, `publishedRevisionId`, and revision `id`/`version`/`status`/`reviewedBy`/`publishedBy`/
`publishedAt`. Every lesson receives exactly one revision `version=1 status=DRAFT createdBy=actor`; Activities are
`source=HUMAN`, `aiMetadata=null`.

**Supported activity types:** markdown (`TEXT`/`EXPLANATION`/`EXAMPLE`) and objective (`MINI_QUESTION`/`PRACTICE`/
`MASTERY_TEST`). Every other type → `IMPORT_ACTIVITY_TYPE_UNSUPPORTED`. Payloads are validated by the same
`validateActivityPayloadForAuthoring` used by single-item authoring (no second parser).

**Provenance (TD-254).** The optional root `provenance.source` — `HUMAN` / `AI_ASSISTED` / `AI_GENERATED` — records the
origin of the package's Activities. **Omitted → HUMAN** (backward compatible with 2.2D documents). STRICT: the only
field is `source`, and its value must be an exact enum; an unknown field inside `provenance` or an invalid `source` →
`IMPORT_INVALID_DOCUMENT`. Every Activity in the package persists with this `source`; **`aiMetadata` is not accepted in
v1 and stays null**. Provenance is part of the `documentHash` (same content under HUMAN vs AI_ASSISTED hashes
differently). Human review/import does **not** rewrite provenance — AI content still flows through the DRAFT → REVIEW →
PUBLISHED workflow.

**Per-item limits:** ≤200 skills, ≤250 lessons, ≤5000 activities, ≤100 skillRefs/lesson, ≤50 skillRefs/activity,
≤50 prereqs/lesson → `IMPORT_LIMIT_EXCEEDED`.

**Aggregate relationship caps** (safety bounds, counted as UNIQUE references after structural validation, rejected with
`IMPORT_LIMIT_EXCEEDED` **before** the write transaction): ≤10 000 LessonSkill, ≤25 000 ActivitySkill, ≤10 000
prerequisites — because the per-item limits can otherwise multiply into pathological totals (5000 activities × 50 skill
refs = 250 000 rows).

**Body boundary:** the ordinary API keeps the small default JSON ceiling (**1 MiB**); ONLY the two import routes accept
up to **5 MiB**, raised as a native Fastify route `bodyLimit` via ONE shared adapter factory
(`src/bootstrap/http-adapter.ts`, used by production and e2e alike). An oversized body is refused with **413 at the
Fastify body-parser boundary — before it is buffered or parsed** (not a post-parse or Content-Length-only check).

Sample that validates cleanly: [`examples/izlan-topic-content.v1.json`](../izlan/examples/izlan-topic-content.v1.json)
(in the runtime repo).

## Identity & resolution semantics

| Entity | Reference | Rule |
|---|---|---|
| Skill | Subject-scoped `code` | Reuse existing **ACTIVE** by code; declared code with conflicting name → `IMPORT_SKILL_CONFLICT`; ARCHIVED → `IMPORT_SKILL_ARCHIVED`; referenced-but-not-declared-nor-existing → `IMPORT_SKILL_NOT_FOUND`. **Two declared skills with different codes but the same normalized name → `IMPORT_SKILL_DUPLICATE`** (the DB enforces `@@unique([subjectId, name])`). No cross-Subject lookup. |
| Lesson | global `contentKey` (`CONTENT_KEY_RE`, ≤200) | **Create-only.** Existing key → HARD `IMPORT_CONTENT_KEY_EXISTS` (never overwrite/adopt). Duplicate within the document → `IMPORT_CONTENT_KEY_DUPLICATE`. |
| Prerequisite | `contentKey` (SAME `CONTENT_KEY_RE`, ≤200 syntax as Lesson.contentKey — NOT skill-code rules) | Target may be a new lesson in the same document OR an existing same-Subject lesson (DRAFT/PUBLISHED ok, ARCHIVED → `IMPORT_PREREQUISITE_ARCHIVED`, missing → `IMPORT_PREREQUISITE_NOT_FOUND`). Cross-Subject → `IMPORT_PREREQUISITE_SUBJECT_MISMATCH`. Cycle over existing + batch edges → `IMPORT_PREREQUISITE_CYCLE`. |
| Skill mapping | `skillCodes` | `lesson.skillCodes` → LessonSkill, `activity.skillCodes` → ActivitySkill; resolved in the destination Subject, ACTIVE only. |
| Reference lists | `skillCodes` / `prerequisiteContentKeys` | **Strict: duplicate items are rejected** (`IMPORT_INVALID_DOCUMENT`) — the parser guarantees de-duplicated lists, so no junction unique conflict surfaces at apply. Silent `Set()` dedup is not the contract. |

## Endpoints

Both under `POST /api/staff/content/topics/:topicId/import/…` and both take the **same** versioned document (no
server-side import session):

- **`/validate`** → dry-run. Returns `{ schemaVersion, documentHash, valid, summary, errors, warnings }`. Never writes.
- **`/apply`** → commit. Re-runs the FULL validation against the current DB, then creates everything atomically. Returns
  `{ schemaVersion, documentHash, summary, lessons }`.

**Authorization (both):** `content.author` **+** a `SubjectAssignment` for the Topic's **server-resolved** Subject
(Topic→Module→Level→Track→Subject). No ADMIN bypass, no role-name check, no client-supplied Subject. `content.publish` is
NOT required. An out-of-scope / non-existent Topic returns **`CONTENT_NOT_FOUND` (404, IDOR-safe)**, never 403.

## Apply — atomicity & serialization

`apply()` runs as ONE Prisma transaction, all-or-nothing:

0. **Preliminary** cheap Topic→Subject + `SubjectAssignment` check BEFORE the expensive parse (so an unassigned actor
   cannot trigger deep validation of a large foreign-Topic document) — then the authoritative check re-runs in the tx.
1. Resolve + authorize the Topic's Subject (`SubjectScopeService.requireScope`).
2. Take the **destination Subject row `FOR UPDATE`** — the SAME graph-serialization authority as the 2.2A-3 prerequisite
   writer, so a concurrent prerequisite write and an import cannot interleave into a cycle.
3. **Re-run full validation** against the current DB snapshot (the dry-run is never trusted).
4. **Batched persistence** (`persistBatch`): Skills (new only) → Lessons → Revisions (v1) → Activities (position = array
   index) → LessonSkill → ActivitySkill → LessonPrerequisite edges, each written with `createMany` / `createManyAndReturn`
   — **no per-row round trips**. Ids are correlated by **stable business keys** (skill code, contentKey, lessonId+version,
   revisionId+position), never by database return ordering. Large junction inserts are **chunked** (1000 rows/insert),
   all inside this transaction.
5. Write **ONE `content.import.apply` StaffAudit** (counts derived from the exact rows written, matching the summary) and
   commit.

A bounded 30 s transaction timeout gives headroom for a near-limit batched package — it is not a substitute for batching.
A unique-constraint race (e.g. a contentKey created by someone else between dry-run and apply) surfaces as
`IMPORT_CONFLICT` (409). The import service uses domain/repo primitives only — it never calls HTTP or any publication
service.

## Audit & documentHash — safe by construction

- **One audit row** (`content.import.apply`), target = the Topic, `metadata` = **subjectId / topicId / documentHash /
  counts only**. It NEVER stores titles, Markdown, payloads, `answerKey`/`correctOptionIds`, the full contentKey array, or
  the raw document.
- **`documentHash`** = SHA-256 over a canonical, key-sorted serialization (arrays keep order), returned from both endpoints
  for operator correlation. It is **not** an authorization or idempotency token, and there is **no** import-batch table.

## Error codes → HTTP

| Code | HTTP | Meaning |
|---|---|---|
| `IMPORT_SCHEMA_UNSUPPORTED` | 400 | `schemaVersion` unknown |
| `IMPORT_INVALID_DOCUMENT` | 400 | unknown/wrong/malformed field or shape |
| `IMPORT_LIMIT_EXCEEDED` | 400 | size/count limit exceeded |
| `IMPORT_CONTENT_KEY_DUPLICATE` | 400 | duplicate contentKey in the document |
| `IMPORT_SKILL_DUPLICATE` | 400 | duplicate skill code in the document |
| `IMPORT_SKILL_NOT_FOUND` | 400 | referenced skill code neither declared nor existing |
| `IMPORT_SKILL_ARCHIVED` | 409 | referenced skill is ARCHIVED |
| `IMPORT_SKILL_CONFLICT` | 409 | declared code name-clashes an existing skill |
| `IMPORT_ACTIVITY_TYPE_UNSUPPORTED` | 400 | activity type not importable |
| `IMPORT_ACTIVITY_PAYLOAD_INVALID` | 400 | payload fails authoring validation |
| `IMPORT_CONTENT_KEY_EXISTS` | 409 | lesson contentKey already exists |
| `IMPORT_PREREQUISITE_NOT_FOUND` | 400 | prerequisite target not resolvable |
| `IMPORT_PREREQUISITE_ARCHIVED` | 409 | prerequisite target ARCHIVED |
| `IMPORT_PREREQUISITE_SUBJECT_MISMATCH` | 409 | prerequisite target in another Subject |
| `IMPORT_PREREQUISITE_CYCLE` | 409 | edges would form a cycle |
| `IMPORT_CONFLICT` | 409 | DB state changed concurrently (unique race) |
| `CONTENT_NOT_FOUND` | 404 | Topic out of scope / non-existent (IDOR-safe) |
| body too large | 413 | JSON body > 5 MiB (Fastify boundary) |

## CMS importer (izlan/web)

- Entry point **"Import qilish"** in the Topic workspace lessons column, shown only with the author capability for that
  Topic's Subject.
- 3 steps: **Fayl tanlash → Tekshirish → Natija**. `.json` only, ≤5 MiB, drag-and-drop or picker.
- **Safe `JSON.parse` only — no eval, no HTML render.** The parsed document lives in component memory and is **never**
  written to localStorage/sessionStorage/IndexedDB.
- Dry-run first (summary cards + localized error list); apply re-runs the server authority (dry-run not trusted), with a
  confirmation dialog, duplicate-submit protection, and **no fake progress percentage**. The success result lists created
  lessons with a "Qoralama" (Draft) badge and deep links; the summary/errors **never render `answerKey`**.
- i18n = chrome only (uz default / ru / en); content is unchanged.

## Source map (runtime repo `izlan/`)

| Concern | File |
|---|---|
| Route-scoped body limit (shared adapter factory) | `src/bootstrap/http-adapter.ts` (used by `src/main.ts` + e2e) |
| Contract, limits (incl. aggregate caps), hash | `src/content-import/import-contract.ts` |
| Strict parser | `src/content-import/import-parser.ts` (+ `import-parser.spec.ts`) |
| Pure resolution/validation | `src/content-import/import-validator.ts` |
| Read snapshot | `src/content-import/import.repository.ts` |
| Orchestration (validate/apply) | `src/content-import/import.service.ts` |
| HTTP | `src/content-import/http/import.controller.ts` |
| Error type | `src/common/errors.ts` (`ContentImportError`) → `src/auth/http/auth-exception.filter.ts` |
| Sample | `examples/izlan-topic-content.v1.json` |
| CMS | `web/src/components/import/ImportDialog.tsx`, wired in `web/src/components/hierarchy/LessonsColumn.tsx` |
| e2e | `test/content-import.e2e-spec.ts` |

## Future (NOT in v1)

Media/image/audio import, CSV/Excel/ZIP, update/overwrite semantics, cross-Topic or Subject/hierarchy import, a background
queue for very large files, an import-history table, and AI-assisted import are all explicitly deferred.
