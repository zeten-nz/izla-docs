# Content Authoring / Publishing / Methodist Workflow — Reconnaissance (Phase 2.2A-P)

> **Recon result (2026-08-21): PASS WITH ARCHITECTURE GAPS.** A reconnaissance does not decide — at recon completion NO
> decision was accepted. **Post-recon owner review (2026-08-21): the content-lifecycle decisions are now ACCEPTED — see
> §13a.** This document preserves the original recon findings as written; §13a holds the accepted answers, and the
> stale "OWNER DECISION / OPEN" labels below are annotated "RESOLVED §13a". NO CODE / SCHEMA / MIGRATION / UI / SEED /
> IMPORT / AI / TD. Payment provider track PAUSED (no merchant docs / sandbox / credentials); completed architecture
> untouched. Baseline unchanged: migrations 21, unit 397, e2e 432, CHECK 45, drift clean.

## Headline
The content **schema** was designed in Phase 1.2/1.3 with a **complete authoring/publishing lifecycle already present**
(status enums, immutable revisions, published-revision pointer, authorship/review/publish FKs, subject scoping,
generalized audit, provider-neutral media, AI provenance). The **authoring application layer is entirely unbuilt** — no
CMS, no content controllers/services/repositories, no content permission codes, no write-time validation. And the
**learner runtime version-selection already matches the owner's ideal policy (§62 A–E)**. So the work ahead is
predominantly **service-layer on top of a ready schema**, plus a handful of small schema decisions.

---

## 1. Existing content architecture (grounded in `prisma/schema/content.prisma`, `core.prisma`, `learning.prisma`)

**Hierarchy** `Subject → Track → Level → Module → Topic → Lesson → LessonRevision → Activity`. Every container carries
`status`, `sortOrder`/`code`, `createdBy`, `createdAt`, `updatedAt`; child→parent FKs are `onDelete: Restrict`.

| Entity | PK | Status field | Ordering | Author fields | Version | Uniqueness | Classify |
|--|--|--|--|--|--|--|--|
| Subject | uuid7 | `ContainerStatus` DRAFT/PUBLISHED/ARCHIVED | sortOrder | createdBy | — | slug unique | AUTHORING-READY |
| Track | uuid7 | ContainerStatus | sortOrder | createdBy | — | (subjectId,slug) | AUTHORING-READY |
| Level | uuid7 | ContainerStatus | sortOrder | createdBy | — | (trackId,code),(trackId,sortOrder) | AUTHORING-READY |
| Module | uuid7 | ContainerStatus | sortOrder | createdBy | — | (levelId,sortOrder) | AUTHORING-READY |
| Topic | uuid7 | ContainerStatus | sortOrder | createdBy | — | (moduleId,sortOrder) | AUTHORING-READY |
| Lesson | uuid7 | `LessonStatus` DRAFT/PUBLISHED/ARCHIVED **+** `publishedRevisionId` unique pointer | sortOrder | createdBy | — | publishedRevisionId unique | AUTHORING-READY |
| LessonRevision | uuid7 | `RevisionStatus` DRAFT/REVIEW/PUBLISHED/ARCHIVED | (via Activity) | createdBy, updatedBy?, reviewedBy?, publishedBy?, publishedAt? | `version Int` | (lessonId,version) | AUTHORING-READY |
| Activity | uuid7 | — (revision owns lifecycle) | `position` | source HUMAN/AI_*, aiMetadata | — | (lessonRevisionId,position) | AUTHORING-READY |
| Skill | uuid7 | `SkillStatus` ACTIVE/ARCHIVED | sortOrder | — | — | (subjectId,name),(subjectId,code) | AUTHORING-READY |
| LessonSkill / ActivitySkill | uuid7 | — | — | — | — | (lesson,skill)/(activity,skill) | AUTHORING-READY |
| LessonPrerequisite | uuid7 | — | — | — | — | (lesson,prereq) | MISSING LIFECYCLE (no cycle/self CHECK) |
| MediaAsset | uuid7 | processing PENDING/READY/FAILED + moderation UNREVIEWED/APPROVED/BLOCKED | — | uploadedBy | — | storageKey unique | AUTHORING-READY (no transcript/captions) |
| ActivityMedia | uuid7 | — | position | — | — | (activity,asset), asset onDelete Restrict | AUTHORING-READY |
| SubjectAssignment | uuid7 | — | — | assignedBy? | — | (userId,subjectId) | AUTHORING-READY (unwired) |
| StaffAudit | uuid7 | — | — | actorUserId | — | — | AUTHORING-READY (unwired) |

**Enums:** `ContainerStatus{DRAFT,PUBLISHED,ARCHIVED}`, `LessonStatus{DRAFT,PUBLISHED,ARCHIVED}`,
`RevisionStatus{DRAFT,REVIEW,PUBLISHED,ARCHIVED}`, `SkillStatus{ACTIVE,ARCHIVED}`, `ContentSource{HUMAN,AI_GENERATED,AI_ASSISTED}`,
`ActivityType{TEXT,EXPLANATION,IMAGE,AUDIO,EXAMPLE,MINI_QUESTION,PRACTICE,SPEAKING,WRITING,LISTENING,AI_INTERACTION,MASTERY_TEST,VIDEO}`,
`MediaProcessingStatus{PENDING,READY,FAILED}`, `MediaModerationStatus{UNREVIEWED,APPROVED,BLOCKED}`.

## 2. Logical vs versioned (§3) — PASS (schema), enforcement is service-layer
`Lesson` = stable logical identity; `LessonRevision` = versioned snapshot; schema comments assert revision immutability
(`content.prisma:144,191`). **No code mutates a published revision or its activities in place** (repo-wide search: zero
`lessonRevision.(update|delete)` / `activity.(update|delete)` in `src/`). Immutability holds today only because **no
authoring writer exists**; it is *not* DB-enforced against a future mutator. → the future publish/edit service must
enforce "published revision immutable; edits create a new DRAFT revision."

## 3. Published-revision authority (§5/§6) — PASS
`Lesson.publishedRevisionId` (unique, circular FK, `onDelete: Restrict`) is the current-revision pointer — a
deterministic single authority (NOT "latest by updatedAt"). Lesson-start dereferences it, gated on
`Lesson.status=PUBLISHED` + full hierarchy PUBLISHED (`lesson-execution.repository.ts:39-50`). `LessonRevision.version`
(unique per lesson) is the monotonic revision number. Missing: the **atomic publish transaction** that moves the pointer,
supersedes the prior revision, and stamps `publishedAt/publishedBy` (schema supports it; no service).

## 4. Revision lifecycle (§4/§7) — schema PASS (owner decisions RESOLVED post-recon — §13a)
`RevisionStatus` already models DRAFT → REVIEW → PUBLISHED → ARCHIVED. At recon this raised the question of adding
`SUPERSEDED`/`REJECTED` states. **RESOLVED §13a: no `SUPERSEDED`/`REJECTED` enum for MVP** — a replaced revision moves to
`ARCHIVED`, a rejected review returns to `DRAFT`. Draft mutability, review-freeze, and published/archived immutability
are service rules (no service yet).

## 5. Runtime version selection (§24/§25/§26/§27/§61/§62) — PASS, already ideal
| Layer | References | Verdict |
|--|--|--|
| RoadmapItem | **logical** `lessonId` (+ checkpointId/skillId), no revision | matches §62-D |
| DailyPlanItem | **logical** `lessonId`/`roadmapItemId`/`skillId`, no revision | matches §62-C-as-logical |
| LearningSession | **no** lesson/revision id (time container only) | pin is elsewhere |
| LearnerLessonProgress | `lessonId` + **pinned** `lessonRevisionId` (set once at first start, never re-pinned) | matches §62-B |
| ActivityAttempt | exact `activityId` + `lessonRevisionId` | matches §62-A |
| LearnerLessonCompletion | `lessonId` + `lessonRevisionId` (permanent) | matches §62-A |
| LearnerReviewSession | `skillId` + `lessonId` + **encountered** `lessonRevisionId` (never re-pin) | history-safe |
| SkillMeasurement | **only** `skillId` (+ optional lesson/attempt/review provenance; no revision/activity id) | decoupled |

Freeze mechanism: `LessonExecutionService.startFromDailyPlanItem` resolves `Lesson.publishedRevisionId` and pins it onto
`LearnerLessonProgress` at first start; every attempt copies that pin; submits to activities outside the pinned revision
are rejected (`lesson-execution.service.ts:86-91,189,206`). Review pins the *encountered* revision
(completion > progress fallback, `review-session.repository.ts:42-47`). **Net: completed/active learners are
revision-stable; roadmap/daily-plan hold logical lessons; new/unstarted learners get the current pointer at start; no
retroactive rewrite.** The owner's §62 policy is already the implemented behavior.

## 6. Activities, validation, registry (§13/§14/§15/§16/§17) — **key SERVICE gaps**
- **Content blocks = Activity types** (no separate `ContentBlock` model, by TD-21/22). View-only set
  `{TEXT,EXPLANATION,IMAGE,AUDIO,EXAMPLE}`; objective/scored `{MINI_QUESTION,PRACTICE,MASTERY_TEST}`;
  `{SPEAKING,WRITING,LISTENING,AI_INTERACTION,VIDEO}` are **deferred/unsupported at runtime** (cannot complete a lesson in
  v1 — `lesson-completion-eligibility.ts`). `VIDEO` is not yet a first-class view-only type despite being non-interactive.
- **Ordering:** `Activity.position` with `@@unique([lessonRevisionId,position])`.
- **Validation is runtime-only, hand-rolled, and duplicated.** No Joi/Zod/class-validator. Two separate parsers that can
  drift: `parseObjectiveActivityPayload` (`lesson-activity-objective/v1`, MINI_QUESTION/PRACTICE/MASTERY_TEST) and
  `parseItemPayload` (`placement-item/v1`, assessment, +open_ended). Both run **only** on learner read/score paths;
  **no write-time/authoring validation exists** — seeds insert raw JSONB. The objective-type set is copy-pasted as
  `new Set([...])` in **4 files**. There is **no type→schema→scoring→renderer registry**.
- **answerKey** lives server-only inside payload JSONB and is stripped from learner projections (no leak).
- → **The single most important CMS prerequisite: a shared canonical Activity-type registry + payload validator** (one
  source of truth) used by BOTH publish-time authoring AND learner runtime, replacing the duplicated parsers/sets.
  - **✅ RUNTIME SIDE IMPLEMENTED Phase 2.2A-R** (2026-08-21, code `bd83c99`): canonical
    `src/content/activity/activity-registry.ts` now owns objective/view-only/unsupported classification (the copy-pasted
    `new Set([...])` are gone); a neutral shared choice-question primitive
    (`src/common/payload/choice-question-payload.ts`) backs BOTH `parseObjectiveActivityPayload` and `parseItemPayload`
    without merging the two domain contracts (`lesson-activity-objective/v1` vs `placement-item/v1` stay distinct).
    View-only payload shapes remain undefined (payloadContract = NONE_DEFINED). **Write-time/authoring validation is still
    NOT built — it is the 2.2A authoring backend that will consume this registry.** (TD-246.)

## 7. Skills & prerequisites (§20/§21/§22/§23) 
- **Skill** is subject-scoped (`SkillStatus ACTIVE/ARCHIVED`), unique by (subject,name) and (subject,code); referenced by
  diagnostics, roadmap, review, skill state/measurement — so archive-not-delete for any measured skill. Merge is unmodeled.
- **Skill mapping exists at BOTH levels:** `LessonSkill` and `ActivitySkill`. A new revision can change activity skills;
  because `SkillMeasurement` carries only `skillId` (+ lesson/session provenance, not activity/revision), historical
  evidence stays semantically stable and is not invalidated by a new revision.
- **Prerequisites are Lesson-level** (`LessonPrerequisite`, unique (lesson,prereq)). **Cycle prevention is unimplemented:
  the init migration has only the unique index — no self-reference CHECK and no cycle guard**; the `content.prisma:273`
  "app + CHECK" note is aspirational and, per the owner correction (2026-08-21), **imprecise**. A DB row-level CHECK can
  enforce **only the self-loop** (`lesson_id <> prerequisite_lesson_id`) — it **cannot** detect a multi-node DAG cycle.
  **Full DAG cycle prevention belongs to service/transaction validation** at write/publish time, not a DB CHECK. →
  SCHEMA (self-loop CHECK, DB) + SERVICE (cycle validation).
- Roadmaps are generated snapshots referencing logical lessons; a changed prerequisite graph should affect **future**
  roadmap generation/replanning only — never retroactive mutation (consistent with current snapshot architecture).

## 8. RBAC / subject scope / audit (§8/§9/§10/§38/§70/§71/§72/§73)
- **Permission registry is empty of content codes** — only 4 payment codes registered; no `Permission` DB table
  (`RolePermission.permissionCode` string validated against an app-side registry). Adding content permissions is purely
  additive.
- **Roles** LEARNER/METHODIST/MODERATOR/ADMIN (data). **`PermissionsGuard` is pure permission-code — NO ADMIN bypass**
  (project convention to follow; no hidden admin override).
- **`SubjectAssignment`** (methodist↔subject, unique (userId,subjectId), assignedBy audit) exists but is **unwired** —
  subject-scope enforcement (permission ≠ authorization for a specific subject) must live in a guard/policy/repository
  filter, never client `subjectId`.
- **`StaffAudit`** (actorUserId, actionCode, targetType/targetId, reason, metadata) exists, **unwired** — reuse for
  publish/archive/skill/prereq/import actions (no event-sourcing needed).

## 9. Media (§39/§40) — schema PASS, unwired
`MediaAsset` is provider-neutral (`storageKey`, not URL), with independent processing + moderation axes; activities
reference it **relationally** via `ActivityMedia` (Restrict), never by id inside JSONB. **No** `transcript`/`captions`/
`thumbnail` fields. No upload/storage/processing service. Publish-time media validation (exists, READY, safe MIME, not
BLOCKED) is a future service check; text-first English MVP need not block on transcoding.

## 10. Content format / localization / assessment separation
- **Rich text (§54/§55):** unmodeled at schema level — text lives in `Activity.payload` JSONB (free shape today). Format
  was an OWNER DECISION at recon; **RESOLVED §13a: restricted Markdown** (XSS-safe, mobile-portable, math/code-extensible)
  — raw HTML is NOT authoring authority. (The schema representation is still to be built.)
- **Localization (§41):** no locale field — effectively **one authored language per revision**; Uzbek-instruction /
  English-target coexist inside payload content. Translation variants = OPEN/future.
- **Assessment content (§43):** `AssessmentDefinition/Version/Item` is a **separate domain** (`AssessmentPurposeScope
  DIAGNOSTIC/CHECKPOINT`, its own `placement-item/v1` validator). Keep learning content and diagnostic definitions
  separate — do NOT collapse. **MASTERY_TEST (§44)** is an ordinary lesson `Activity` type → authored through the activity
  registry.
- **Level/CEFR (§59):** `Level.code` is a free String display code (TD-27, enum EMAS) ordered by sortOrder — CEFR-like
  codes are data, not hardcoded. PASS.

## 11. Bulk import / export / AI (§46–§51)
- **No import path exists.** Natural keys: Subject.slug, (subject) Track slug, (track) Level code, (subject) Skill code
  are unique; **Lesson has only a nullable non-unique slug — no stable external import key.** **RESOLVED §13a: a new
  immutable stable Lesson `contentKey` is the import/business identity; title is not identity.** (Lesson **slug** stays a
  separate routing/SEO concern — its uniqueness is NOT part of content identity and remains OPEN.)
- Recommended future import: structured JSON package → validate → dry-run → transactional import → **all DRAFT** →
  human review → publish. Never import into PUBLISHED. Export/backup can reuse the same package format (mark future).
- **AI (§50/§51):** `Activity.source{HUMAN,AI_GENERATED,AI_ASSISTED}` + `aiMetadata` already model provenance (TD-20
  "AI review'siz publish bo'lmaydi" = AI never auto-publishes). AI stays DRAFT/suggestion; human publishes.

---

## 12. Gap classification (§74/§80)
| # | Area | Status |
|--|--|--|
| 1 | hierarchy authoring (Subject…Topic) | PASS (schema); SERVICE GAP (no CRUD) |
| 2 | lesson logical identity | PASS |
| 3 | revision model | PASS |
| 4 | revision status enum | PASS (DRAFT/REVIEW/PUBLISHED/ARCHIVED); RESOLVED §13a (no SUPERSEDED/REJECTED enum — supersede→ARCHIVED, reject→DRAFT) |
| 5 | revision numbering (`version` unique) | PASS |
| 6 | published-revision pointer | PASS |
| 7 | published immutability | PASS (schema/convention); SERVICE GAP (enforce on future writer) |
| 8 | review workflow | SERVICE GAP (states exist; no workflow) |
| 9 | publish authority | RESOLVED §13a (content.publish + subject scope; self-publish OK; no ADMIN bypass) + SERVICE GAP |
| 10 | methodist subject scope | PASS (SubjectAssignment); SERVICE GAP (unwired enforcement) |
| 11 | learner draft isolation | SERVICE GAP (PUBLISHED gate re-implemented per read path, not centralized) |
| 12 | activity payload validation | runtime parsers DE-DUPLICATED via shared primitive (2.2A-R); WRITE-TIME/authoring validation still SERVICE GAP → 2.2A |
| 13 | activity type registry | ✅ IMPLEMENTED 2.2A-R (canonical `activity-registry.ts`; set no longer copy-pasted; exhaustiveness-tested) |
| 14 | ordering (position unique) | PASS; SERVICE GAP (safe reorder endpoint) |
| 15 | skill lifecycle | PASS (schema); SERVICE GAP; RESOLVED §13a (skill merge DEFERRED; measured skills never hard-deleted) |
| 16 | skill mapping (lesson+activity) | PASS |
| 17 | prerequisite DAG | SCHEMA GAP (no self-loop CHECK — DB can only do `lesson_id <> prerequisite_lesson_id`) + SERVICE GAP (full DAG cycle validation is service/transaction-level, not a DB CHECK) |
| 18 | roadmap version behavior | PASS |
| 19 | daily-plan version behavior | PASS |
| 20 | learning-session revision freeze | PASS (pin on progress/attempt) |
| 21 | historical attempt integrity | PASS |
| 22 | preview | SERVICE GAP (no draft preview surface) |
| 23 | urgent takedown | PASS (LessonStatus ARCHIVED + hierarchy gate); SERVICE GAP (no endpoint); RESOLVED §13a (Lesson ARCHIVED for MVP; no HIDDEN state) |
| 24 | archive/delete policy | PASS (Restrict FKs prevent hard-delete of referenced); RESOLVED §13a (referenced never hard-deleted; unreferenced DRAFT deletable) |
| 25 | rich text | RESOLVED §13a (restricted Markdown; raw HTML not authority); schema representation unbuilt |
| 26 | media | PASS (schema); SERVICE GAP (unwired); transcript/captions DEFERRED §13a |
| 27 | bulk import | SERVICE GAP (none) — FUTURE (Phase 2.2D); structured-JSON→validate→dry-run→DRAFT is a RECOMMENDED design (not yet an accepted decision); only import IDENTITY is RESOLVED §13a (Lesson.contentKey) |
| 28 | import idempotency | RESOLVED §13a (immutable Lesson contentKey; title≠identity); SCHEMA GAP (contentKey field built at 2.2A-D) |
| 29 | audit | PASS (StaffAudit); SERVICE GAP (unwired) |
| 30 | edit concurrency | RESOLVED §13a (updatedAt optimistic concurrency; no dedicated version field) |
| 31 | production seed/content process | RESOLVED §13a (initial pilot ≈10–20 English A1 lessons); broader production/import operating process = future |
| 32 | AI authoring boundary | PASS (source/aiMetadata; provenance) |

## 13a. ACCEPTED OWNER DECISIONS (2026-08-21)
The 12 decisions surfaced in §13 (and related items) were resolved by the owner on 2026-08-21. These are **ACCEPTED**
and will be formalized as TDs when Phase 2.2A-D is implemented (no TD assigned in this docs cleanup). Removed from
OPEN_QUESTIONS accordingly.

- **Revision lifecycle:** `DRAFT → REVIEW → PUBLISHED → ARCHIVED`. Review rejection: `REVIEW → DRAFT`. **No `SUPERSEDED`/
  `REJECTED` enum for MVP.** A published revision becomes `ARCHIVED` when replaced.
- **Publish authority:** requires `content.publish` permission **+ subject scope**. **Self-publish allowed for MVP** with
  correct permission/scope. **No hidden ADMIN bypass.**
- **Current published revision authority:** `Lesson.publishedRevisionId` (confirmed).
- **Learner version semantics (ACCEPTED as-is):** Roadmap/DailyPlan hold the **logical lesson**; the exact revision
  **freezes when the lesson starts**; active/completed learner history stays **pinned**; unstarted execution uses the
  **current published revision**.
- **Urgent takedown:** `Lesson` `ARCHIVED` for MVP (revision stays immutable).
- **Delete:** referenced/published content is **never hard-deleted**; unreferenced `DRAFT` may be deletable.
- **Hierarchy move:** published content is **not reparented** in MVP — use clone / new logical content where needed.
- **Rich text:** **restricted Markdown**; raw HTML is **not** authoring authority.
- **Bulk import identity:** requires an **immutable stable Lesson `contentKey`**; title is **not** identity.
- **Edit concurrency:** `updatedAt` **optimistic concurrency** for MVP.
- **Prerequisites:** DB CHECK enforces **only** the self-loop (`lesson_id <> prerequisite_lesson_id`); **full DAG cycle
  prevention is service/transaction validation** (a DB CHECK cannot enforce it).
- **Deferred:** skill merge; media transcript/captions.
- **Production content pilot:** ≈ **10–20 English A1 lessons** first.

## 13. Owner decisions (§75) — RESOLVED 2026-08-21 (see §13a above for the accepted answers)
1. Revision lifecycle: add `SUPERSEDED`/`REJECTED`, or reuse `ARCHIVED` + return-to-DRAFT?
2. Who may publish (own-subject methodist self-publish vs required reviewer vs `content.publish` permission)?
3. Author self-publish allowed for a small team (permission + review state + audit, same person)?
4. `currentPublishedRevision` authority — confirm the existing `Lesson.publishedRevisionId` pointer (recommended PASS).
5. Version selection for Roadmap/DailyPlan/LearningSession — confirm the existing (already-ideal) behavior.
6. Urgent takedown model — `LessonStatus HIDDEN` vs reuse `ARCHIVED`; keep revision immutable.
7. Archive/delete — never hard-delete referenced content; draft-delete policy.
8. Hierarchy move (Lesson→different Topic) — allow for draft; for published, move-for-discovery vs clone.
9. Rich-text format (sanitized structured/Markdown vs HTML).
10. Bulk import identity (stable Lesson external key/slug uniqueness).
11. Optimistic editing concurrency (updatedAt check vs explicit version/lock).
12. Production content pilot size (recommend tiny A1 pilot before bulk).

## 14. Recommended implementation sequence (§76) — subject to owner phase prompt
1. **2.2A-D — Content Lifecycle / Schema Hardening** — ✅ **IMPLEMENTED 2026-08-21** (migration 22; `Lesson.contentKey`
   + `chk_lesson_prerequisite_no_self_loop`; formalized TD-240..245; checkpoints/2.2A-D.md). Scope was minimal, schema-only — exactly two schema changes —
   (a) a **LessonPrerequisite self-loop DB CHECK** (`lesson_id <> prerequisite_lesson_id`), and (b) an **immutable stable
   Lesson `contentKey`** (import/business identity; title is not identity; Lesson slug stays a separate routing/SEO
   concern, not content identity). It **documents** the accepted contracts below but does **not** implement their
   service-layer enforcement: **full-DAG cycle prevention** (a DB CHECK cannot detect a multi-node cycle) → Phase 2.2A;
   **`updatedAt` optimistic-concurrency enforcement** → Phase 2.2A; **revision-lifecycle enforcement**
   (`DRAFT→REVIEW→PUBLISHED→ARCHIVED`, reject→DRAFT) → authoring/publishing phases. **Not built** (accepted §13a):
   SUPERSEDED/REJECTED states, a dedicated concurrency version column, media transcript/captions (deferred). (One
   migration; schema-only.)
2. **2.2A-R — Canonical Activity Registry + Shared Payload Validator** — **✅ DONE on branch** (2026-08-21, code `bd83c99`;
   behavior-preserving refactor, no schema): single runtime source of truth for Activity classification + one Lesson
   objective payload authority + a neutral shared choice-question primitive (AssessmentItem stays a separate contract).
   TD-246; unit 397→417; e2e 436 unchanged; migrations 22 / CHECK 46. Write-time/authoring validation consuming it → 2.2A.
3. **2.2A — Content Authoring Backend**: subject-scoped CRUD for hierarchy + lessons + draft revisions + activities +
   skills + prereqs; SubjectAssignment enforcement; content permission codes; StaffAudit wiring; write-time validation;
   **full-DAG prerequisite cycle prevention** (service/transaction validation, where prerequisite writers exist); and
   **`updatedAt` optimistic-concurrency enforcement** in the update/write services.
4. **2.2B — Publishing / Revision Workflow**: atomic publish transaction (pointer move + supersede + publishedAt/By),
   publish validation (hard blockers vs warnings), preview, idempotent republish, takedown, centralized learner-visibility gate.
5. **2.2C — Methodist CMS** (frontend, out of backend scope here).
6. **2.2D — Bulk Import + Validation** (package → dry-run → DRAFT → review → publish).
7. **2.2E — English A1 Pilot Content** (tiny, human-authored, end-to-end tested before expansion).

## 15. Baseline & boundary
No repository modification this phase (recon only). Baseline: migrations 21, unit 397, e2e 432, total 829, CHECK 45,
drift clean. Payment provider track PAUSED and untouched. (Recon itself added no TD. **Update 2026-08-21:** the owner
accepted the content-lifecycle decisions — see §13a; they will be formalized as TDs when Phase 2.2A-D is implemented.)
