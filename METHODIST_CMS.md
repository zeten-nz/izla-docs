# Methodist CMS — Living UI/Contract Reference (`izlan/web`)

> **Mutable living doc** for the Methodist content CMS web app (introduced Phase 2.2C, TD-251). Records the
> frontend↔backend contract mapping, auth/OCC invariants, and route map. Phase *results* live in
> [checkpoints/2.2C.md](checkpoints/2.2C.md); the *decision* is [TD-251](TECH_DECISIONS.md). Keep this current as the CMS
> evolves. The **backend is always the authorization + validation authority** — everything here is UX plumbing on top.

## Location & stack
- App: `izlan/web` (inside the runtime repo; backend at repo root). Independently runnable from `web/`.
- Next.js App Router + TypeScript + React + Tailwind (class-based dark mode via CSS-variable tokens), React Icons.
- Scripts: `npm run dev` (port 4000), `build`, `start`, `lint`, `typecheck`, `test` (Vitest + RTL + jsdom).
- Env: `NEXT_PUBLIC_API_BASE_URL` (backend origin, no trailing slash). The backend `CORS_ORIGINS` must include the web
  origin (credentialed CORS + the refresh flow's `credentials: include`). Dev: backend `:3000`, web `:4000`.

## Route map
| Route | Purpose |
|---|---|
| `/staff/login` | Phone → OTP → verify. Access token kept in memory only. |
| `/staff/content` | Assigned-subject list (create when `subjectManage`). |
| `/staff/content/subjects/[subjectId]` | Subject workspace: hierarchy drill-down (Track→Level→Module→Topic→Lesson), Skills, Assignments. |
| `/staff/content/lessons/[lessonId]` | Lesson: header + edit/move/takedown, revisions, LessonSkills, prerequisites. |
| `/staff/content/revisions/[revisionId]` | Revision editor: metadata, activities (markdown/objective, reorder, delete, ActivitySkills), readiness, learner preview, workflow. |

## Auth invariants (TD-251)
- **Access token = memory only** (`lib/auth/token-store.ts`). Never persisted anywhere. Reload re-bootstraps via refresh.
- **Refresh** = HttpOnly rotating cookie `izlan_refresh` (browser-managed) + `X-Izlan-CSRF: 1` + `credentials: include`.
- **Bootstrap** (`AuthProvider`): one `POST /auth/refresh` → `GET /auth/me`; a missing in-memory token before bootstrap is
  NOT logout.
- **Single-flight refresh** (`lib/api/client.ts`): concurrent 401s share ONE refresh; retry each request at most once; a
  failed refresh does not loop. No 409/OCC auto-retry.

## Capability model
- `GET /api/staff/content/session` → `{ userId, capabilities: { author, publish, subjectManage } }` (requires
  `content.author`; 403 → "Content Studio access unavailable"). Derived from effective permission codes. **No role-name
  hard-coding anywhere.** Capabilities gate UX only; the backend re-enforces permission + SubjectAssignment scope per call.

## OCC aggregate-token map (what each mutation sends)
| Mutation | Token field | Aggregate authority |
|---|---|---|
| Subject/Track/Level/Module/Topic/Skill PATCH & hierarchy publish | `expectedUpdatedAt` | that entity's `updatedAt` |
| LessonSkill add/remove, Prerequisite add/remove, Lesson PATCH/move, takedown | `expectedLessonUpdatedAt` | `Lesson.updatedAt` |
| Activity create/patch/delete, reorder, ActivitySkill add/remove, revision metadata, submit-review, return-to-draft | `expectedRevisionUpdatedAt` / `expectedUpdatedAt` | `Revision.updatedAt` (ONE authority in `RevisionEditorProvider`) |
| Publish revision | `expectedRevisionUpdatedAt` + `expectedLessonUpdatedAt` | both (freshly re-read Lesson token) |
- `CONTENT_EDIT_CONFLICT` → visible conflict banner (reload latest / cancel). Never a silent retry.

## Content authoring invariants
- Markdown payload `lesson-activity-markdown/v1` = exactly `{ schemaVersion, markdown }` (trimmed, non-empty, ≤50000). No
  `rawHtml`. Rendered with react-markdown, **raw HTML disabled** (no `rehype-raw`, no `dangerouslySetInnerHTML`).
- Objective payload `lesson-activity-objective/v1` = `{ schemaVersion, format, prompt, options[], answerKey:{correctOptionIds} }`.
  `answerKey` is authoring-only — editable in the objective editor, **never** shown in the learner preview.
- **Learner preview** renders an explicit allowlist view model (`lib/activity/preview-view-model.ts`): only
  id/type/position + (objective) format/prompt/options{id,text} or (markdown) markdown. Never `answerKey`/`correctOptionIds`/
  `storageKey`; never stringifies the raw payload.
- Creatable ActivityTypes: TEXT/EXPLANATION/EXAMPLE + MINI_QUESTION/PRACTICE/MASTERY_TEST. IMAGE/AUDIO = display-only
  (media authoring deferred). SPEAKING/WRITING/LISTENING/AI_INTERACTION/VIDEO = not creatable.
- Lifecycle freeze mirrors the backend: containers/lessons editable only in DRAFT; revisions/activities/mappings only when
  the owning revision/lesson is DRAFT; skills only when ACTIVE. Impossible actions are hidden, not shown-then-erroring.

## Design & motion
- **Tokens** (`app/globals.css`): CSS variables for bg/surface/surface-2/border/text/muted/primary/danger/warning/success/
  info + ring/shadow; intentional light AND dark palettes (Tailwind reads them via `rgb(var(--token) / <alpha>)`); no
  hard-coded hex in components. **Typography:** `next/font` Inter, `latin` + `cyrillic` subsets, exposed as `--font-sans`.
- **Motion** (`lib/motion/motion.ts`, Framer Motion): one preset set (spring/ease, 120–280ms) for dialog/palette, overlay,
  toast, tab indicator (shared layout), list in/out, drawer, sidebar collapse, page fade, button press. Global
  `MotionConfig reducedMotion="user"` respects the OS reduce-motion setting.
- **Shell:** collapsible animated sidebar (icons + tooltips collapsed; pref in `izl-sidebar`), mobile drawer, ⌘K command
  palette (`components/shell/CommandPalette.tsx` — navigate/switch-subject/theme, current data only). Escape closes overlays.

## i18n (UI chrome only)
- Client `I18nProvider` + `useT()` (`lib/i18n/`), flattened dictionaries with `{var}` interpolation. Locales **uz
  (default) / ru / en**; `ru`/`en` are typed `Messages` (the `uz` shape) so missing/extra keys fail typecheck. Locale
  persisted in `localStorage` + a non-auth cookie (`izl-locale`); `<html lang>` updates; decoupled from the auth token.
- **UI language ≠ content language.** Only application chrome is localized. Authored Lesson content and the content model
  are unchanged (no `titleUz/Ru/En`, no translation tables, no schema/migration). Backend enum values are never localized —
  only their display labels (e.g. `PUBLISHED` → Nashr qilingan / Опубликовано / Published).

## Authentication (TD-252)
- **Login = phone + password** (`POST /auth/login`). `/staff/login` has an accessible show/hide password toggle
  (`autocomplete=current-password`), a **forgot-password OTP recovery** flow (phone → reset code → new password → sign in),
  and a dev-only demo helper (`NEXT_PUBLIC_ENABLE_DEMO_ACCOUNTS`, never production) that fills the **phone only**.
- OTP is recovery/registration only — not login. Access token stays **memory-only**; the HttpOnly rotating refresh cookie +
  single-flight refresh are unchanged. Staff use the same `/auth/login` as everyone; RBAC + the capability endpoint decide
  access. Backend: Argon2id `PasswordCredential` (separate from identity), enumeration-safe login, reset revokes sessions.

## Interaction accessibility
- **Activity reorder** uses the canonical dnd-kit pattern: sortable element `setNodeRef`; drag handle `setActivatorNodeRef`
  + attributes/listeners (only the handle is a drag surface). Keyboard drag + accessible up/down alternatives retained.
- **Modals** (Dialog + command palette) share `lib/hooks/use-focus-trap.ts`: initial focus inside, `Tab`/`Shift+Tab`
  containment with wrap, Escape close, and focus restoration to the opener on close. Covered by UI-A11Y-01..05.

## Deferred / out of scope (as of 2.2C)
ActivityMedia upload/delivery (→ later), bulk import (→ 2.2D), learner web app (→ 3.x), AI authoring. See
[PROJECT_STATE.md](PROJECT_STATE.md).
