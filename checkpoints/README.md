# Accepted Phase Checkpoints (immutable)

One file per accepted phase: `<phase-id>.md` (e.g. `2.2A-D.md`). **Immutable once committed** — never rewrite an
accepted historical checkpoint. Corrections are recorded in a **later** checkpoint or an explicit `CORRECTION-<phase-id>.md`
note, never by editing the original.

Recording begins **2026-08-21**. Phases before that date have no per-phase SHA and are indexed in
[../PHASE_HISTORY.md](../PHASE_HISTORY.md) (all contained in code `19461eb`, docs `92cadce`).

## Required fields (every checkpoint)
```
# Phase <phase-id> — <title>  (ACCEPTED <date>)

- Result: PASS / PASS WITH GAPS / PASS WITH BLOCKER / ...
- Code commit SHA (izlan):   <40-hex>
- Docs commit SHA (izla-docs): <40-hex>
- Branch: phase/<phase-id>
- Migration count: <n>   (last migration: <name>)
- Unit / E2E totals: <u> / <e>
- Critical invariants / regressions covered: <list — NOT just counts>
- Changed file paths: <list>
- TDs added: <TD-xxx..yyy or none>
- Remaining blockers: <list or none>

## Summary
<the accepted checkpoint body>
```

## Rules
- The recorded **code commit SHA is the exact implementation inspected** for the checkpoint. Documentation must not be
  claimed to represent code whose SHA differs from this. If a later inspection finds divergence, record **DOCUMENTATION
  DRIFT** in a new checkpoint/correction note.
- A checkpoint is written only for an **accepted** phase (owner-approved). Reconnaissance-only phases may be recorded
  here when accepted, with code SHA unchanged from the prior phase and the change confined to izla-docs.
