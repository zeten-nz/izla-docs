# XP Progression Foundation — Implementation (Phase 2.0D)

> Status: **COMPLETE** (2026-08-20). A deterministic, versioned XP level curve plus an **activated, rebuildable
> `XpBalance` projection**. `XpGrant` stays the XP source of truth; `XpBalance` is a mutable cache repaired only in
> the direction `XpGrant → XpBalance`. Learner reads derive canonical progression from `SUM(XpGrant.amount)`.
> **No IZL. No badges. No streaks. No entitlements. No reward on level-up.**
> Owner: **TD-145/146/147/148/149**. Migration `20260820190000_activate_xp_balance_projection`.

Code: `backend/src/xp/progression/xp-progression.engine.ts` (pure), `xp.repository.ts` (`recomputeProjection`),
`xp.service.ts` (`getProgression`/`recomputeProjection`/`reconcile`), `xp.controller.ts`. Bridge in
`daily-mission.service.ts`.

## 1. Scope
A pure integer level curve (`xp-progression-v1`) and a rebuildable `XpBalance` projection. IN: level formula,
progression projection, full-history recompute, canonical read, reconcile-rebuild, concurrency lock, signed/negative
handling, level-down. OUT (deferred): max/display level, `highestLevelEver`, prestige, titles, badges, streaks,
leaderboards, XP history API, admin corrections, level rewards/unlocks, incremental projector.

## 2. XpGrant authority
`XpGrant` (append-only, signed) is the canonical XP total: `totalXp = SUM(XpGrant.amount)`. `XpBalance` is never
authoritative and is never used to modify `XpGrant`.

## 3. XpBalance projection (TD-146)
`XpBalance{userId, totalXp, currentLevel, progressionVersionCode, updatedAt}` is a rebuildable materialized cache.
Activated in this phase (Phase 2.0C-2 wrote it zero times). Written only by the XP projection module.

## 4. Progression version (TD-145 / §16)
`progressionVersionCode = xp-progression-v1` tags which curve produced a cached `currentLevel`, so a future `v2`
deploy can detect stale projections. `NULL` = stale/unversioned; reconcile/recompute upgrades it to v1. This is
projection provenance, not XpGrant provenance.

## 5. Level formula (TD-145 / §3)
Cumulative XP to **be** level L (L ≥ 1): `threshold(L) = 100·(L-1)·L/2 = 50·(L-1)·L`. Level starts at 1 (no Level 0).
Per-level step L→L+1 is `100·L`.

## 6. Threshold examples
| Level | 1 | 2 | 3 | 4 | 5 | 6 | 10 |
|---|---|---|---|---|---|---|---|
| cumulative XP | 0 | 100 | 300 | 600 | 1000 | 1500 | 4500 |

## 7. No maximum level (§5)
`currentLevel` is derived from XP with no cap in v1. Display caps / prestige / seasonal levels are future product
policy.

## 8. Signed total XP (TD-147 / §7)
`XpGrant` supports signed corrections, so `totalXp = SUM(...)` may be negative. History is never censored: the API
returns the actual signed `totalXp`.

## 9. Progression clamp (§7)
Level math uses `progressionXp = max(totalXp, 0)`. A negative total never drops below Level 1. Example: sum `-20` →
`{totalXp: -20, progressionXp: 0, currentLevel: 1}`.

## 10. Level-down semantics (TD-147 / §8)
`currentLevel` is a projection of the *current* total, not `highestLevelEver`. An accepted negative correction that
drops the total below a threshold **lowers** the level (e.g. 310 → L3, then −50 → 260 → L2). Permanent-level /
`highestLevelEver` is out of scope.

## 11. Pure engine (§9)
`computeXpProgression(totalXp)` → `{totalXp, progressionXp, currentLevel, currentLevelStartXp, nextLevelXp,
xpIntoLevel, xpToNextLevel, progressBp, progressionVersion}`. `levelForXp` = max L≥1 with `threshold(L) ≤
progressionXp`. `progressBp = floor(xpIntoLevel·10000 / (nextLevelXp − currentLevelStartXp))`, clamped `0..9999`
(exact threshold enters the next level). No Prisma/HTTP/Clock.

## 12. Integer arithmetic (§11 / §12)
`levelForXp` uses an integer binary search (exponential upper-bound expansion) — no floating-point quadratic, so no
boundary error. Thresholds stay well within `Number.MAX_SAFE_INTEGER` for any Int-range total. `XpBalance.totalXp`
is `Int`; a sum outside the Int range would fail the cache write (deferred, not silently overflowed) while GET
stays canonical from the SUM. Normal MVP values are far below the limit.

## 13. Projection recompute (§18)
`recomputeProjection(userId)`: per-user advisory lock → `SUM(XpGrant.amount)` → `xp-progression-v1` level →
UPSERT `XpBalance{totalXp, currentLevel, progressionVersionCode}`. **Full recompute**, never `balance += amount`.

## 14. Full-history SUM (§19 / §24)
Recompute sums **all** `XpGrant` rows (mission XP, corrections, any future source), so progression is source-agnostic
and any correction/import naturally enters the current level.

## 15. Concurrency (§47–49)
A per-user transaction-scoped advisory lock (`pg_advisory_xact_lock(hashtext('xp'), hashtext(userId))`) serializes
recomputes; each reads a fresh SUM inside the lock, so concurrent grants/recomputes converge to the canonical total —
no lost updates. (e2e: concurrent reconcile → correct full sum.)

## 16. Failure boundary (TD-148 / §21 / §59)
Order: ActivityAttempt → DailyMissionCompletion → XpGrant → XpBalance. The projection is refreshed via
`tryRecomputeProjection` (non-throwing). A cache failure never rolls back the grant/completion/attempt; reconcile
repairs the cache.

## 17. Canonical GET (TD-149 / §25 / §26)
`GET /api/xp/me` returns the full progression object computed from `SUM(XpGrant.amount)` via the engine — **not** the
cache. It stays correct even when `XpBalance` is stale or absent.

## 18. Cache staleness (§28 / §57)
`XpBalance` is explicitly recoverable and may lag after a downstream failure. The learner-facing read never trusts it
blindly; a stale/corrupt cache is repaired by reconcile or the next grant's projection refresh.

## 19. Reconcile (§23)
`POST /api/xp/me/reconcile`: repair missing mission XpGrants (2.0C-2) **and** rebuild the projection, returning
canonical progression + `grantsCreated`. It is the single repair command for both grant materialization and
projection drift (no separate projection-repair endpoint). A stale/unversioned `XpBalance` is upgraded to
`xp-progression-v1`; a corrupt cache (e.g. 10/L1 while SUM=300) is corrected to 300/L3.

## 20. No level rewards (§38 / §40)
Crossing a threshold changes only the projection/response. It creates no XpGrant, RewardGrant, IZL, badge,
notification, entitlement, subscription, mission, or DailyPlan item — and no `LevelUpEvent`/history table (§39).
Level is a projection, not an event producer.

## 21. No entitlements (§41)
Level unlocks nothing (no PRO/MAX/community role/lesson gating). No access-control dependency on level.

## 22. No streaks/badges (§43 / §44)
No streak counters, freezes, badges, or titles. Future scope.

## 23. Tests
- **Unit** (`progression/xp-progression.engine.spec.ts`, 11): threshold formula; boundaries 99/100/299/300/599/600;
  1000/1500/4500; negative/zero → L1; large-integer determinism; progressBp cases (50→5000, 200→5000, 299→9950,
  100→0); negative total clamp.
- **e2e** (`test/xp.e2e-spec.ts`, +9 progression, §31–77): first-grant projection (10/L1/v1); level boundaries
  100→L2 / 300→L3 / 4500→L10 with cache agreement; negative total (signed preserved, L1); level-down after
  correction; GET canonical vs stale cache + read-only; stale/unversioned → v1 upgrade; zero-state read-only (no
  XpBalance row created); concurrent reconcile convergence; level-up zero side effects.
- Gate: **339 unit + 260 e2e PASS**; `tsc` clean; drift clean.

## 24. Future progression policy
Deferred to later phases (see [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)): `xp-progression-v2` curve, max/display level,
`highestLevelEver`, prestige, level titles, badges, streaks, leaderboards, XP history UI/API, admin XP corrections,
XP expiration, seasonal XP, level-based unlocks/rewards, incremental projector + global rebuild job, IZL economics.
