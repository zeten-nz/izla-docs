# Izlan — Open Questions

> **Only genuinely UNRESOLVED owner / product / architecture decisions.** Resolved decisions live in
> [TECH_DECISIONS.md](TECH_DECISIONS.md) or the relevant architecture/implementation doc; historical phase status lives
> in [PHASE_HISTORY.md](PHASE_HISTORY.md) + [checkpoints/](checkpoints/). Cleaned 2026-08-21 (the former per-phase
> "HOLATI" blocks were consolidated into PHASE_HISTORY; accepted content-lifecycle decisions moved to
> [CONTENT_AUTHORING_RECON.md](CONTENT_AUTHORING_RECON.md)).

## 1. Technology choices (still open)
1. **SMS provider** (OTP) — production uses `UnavailableSmsAdapter` (not configured); no provider chosen.
2. **Object storage provider** (audio / image / content media) — `MediaAsset.storageKey` is provider-neutral; provider TBD.
3. **Queue / background-job technology** (also gates reservation/order expiry cleanup, scheduler-based reconciliation).
4. **AI provider / model strategy** — provider(s), model(s), fallback, cost model.
5. **Deployment infrastructure.**
6. **Speech pipeline** — audio capture / STT / speaking evaluation (tied to AI provider choice).

## 2. Product / pedagogy (before MVP; not blocking architecture)

### Learning & assessment
- Full **adaptive scoring** rules / question count / threshold (foundation `TD-96` accepted; final psychometric tuning open).
- **Estimated duration** estimation system.
- **Progress calculation** formulas (skill % growth).
- **Level nomenclature / CEFR** display (`displayLevel` currently null; English CEFR-like codes are data).
- **Mastery threshold** final value (90% mission exists; final not fixed).
- **Completed-lesson display**: which revision on re-view (`TD-37` pinning accepted; display policy open).
- **Mistake taxonomy** registry ownership (Methodist/AI) + initial list.
- **Signal lifecycle** tuning (when REVIEW_DUE is created; when a signal EXPIRES).
- **Reassessment policy** (model has purpose=REASSESSMENT; policy open).
- **Daily Plan** regeneration limits/day; unfinished MUST_DO carry-over.
- **AI tutor** conversation retention.

### Rewards & economics
- **1 IZL = X UZS** value; IZL rate change policy (`IzlRateVersion` supports either).
- Daily/weekly **reward-eligible target**; **exact per-category IZL values** (`RewardPolicyVersion.config`).
- **IZL expiry**.
- **MVP redemption list** + cash-out policy.
- **Anti-fraud engine** design (principles `D-27` accepted; engine later).
- **Entitlement change timing** — Admin plan-matrix change applies now vs next cycle (default proposal: next cycle).
- **Refund policy** — payment refund's effect on subscription, earned IZL, spent IZL (also a Payme launch dependency; payment track paused).
- **Negative balance / clawback** — adjustment exceeding balance (current: signed balance; reserved uses `max(available,0)`).

### Subscription & payment (product policy — provider track PAUSED)
- **Tariff prices** (UZS); **feature matrix** (subjects / AI / speaking / writing usage per tier).
- **Trial / free preview** level (beyond guest demo).
- **Subscription-end** learning state (grace period / read-only).
- **Upgrade / downgrade** rules + IZL/progress impact.
- **Recurring / auto-renew** model (manual vs auto) — pending real provider.

### Legal & safety
- **Minors** legal/product review (privacy, parental consent, payment/reward restrictions) — launch blocker `D-32`.
- **Moderation policy** / guidelines + penalties; **block capability** in MVP?
- **Privacy erasure / anonymization** policy (learning-history impact).
- **Media moderation flow** (`TD-74` processing≠moderation accepted; pre/post default + process policy open).
- **Security event retention** + full-phone storage policy.

### Community
- **Media limits** (image count/size, audio duration/size); **reaction list**; **reputation** mechanics.
- **Post/reply editing** policy; **question-post title** requirement.

### Notifications & platform
- **Notification channels** + minimal MVP set (reaction default off?).
- **Learning reminder** design.
- **Platform UI language(s)** — uz only vs uz/ru/en (unresolved anywhere).
- **Analytics** provider / approach.

### Auth & account
- **Account recovery** final policy (phone-access loss; phone recycling risk).
- **Phone change** security steps; **DEACTIVATED** account semantics; **CAPTCHA/escalation** on abuse.

### Routing / infra-adjacent
- **Module/Topic slug/URL** policy (routing/SEO depth).
- **Reservation/order expiry cleanup** mechanism (queue tech).

## 3. Telegram (architecture CANDIDATE — implementation NOT approved)
Telegram is a candidate architecture track. **Implementation is not approved.** Recon done in Phase 2.2T-P
([TELEGRAM_INTEGRATION_RECON.md](TELEGRAM_INTEGRATION_RECON.md) — analysis + recommendations; nothing accepted). These
are the genuinely unresolved **owner decisions** to settle before any Telegram implementation phase (recommendations in
the recon doc §16):
1. **Identity model** — generic `UserIdentity` + nullable phone (rec) vs a Telegram-only side table.
2. **Telegram-only signup** — allow phone-less accounts (rec) vs require a verified phone.
3. **Existing phone-account auto-link** — only on verified-phone match; auto vs confirm-step.
4. **`phone` scope** — request by default vs optional (it is consent-gated, never guaranteed).
5. **Recovery policy** when Telegram and/or phone is lost.
6. **Last-auth-method invariant** — forbid removing the only auth method without recovery (rec: adopt).
7. **Mini App session model** — per-request `initData` vs one-time exchange for an Izlan session (rec; needs a
   non-cookie refresh transport since the current `SameSite=Lax` cookie can't cross Telegram's origin).
8. **Bot notification opt-in** model (which consent gates bot messaging).
9. **Channel mission/reward** policy — none / one-time cosmetic-XP / IZL (rec: never IZL for a repeatable join).
10. **Telegram Stars boundary** — Stars (`XTR`) for digital goods inside Telegram; Stars as a future `PaymentProvider`
    behind the existing finalization boundary.
11. **Website vs Telegram payment** — izlan.uz → CLICK/Payme; Telegram → Stars; both allowed per surface.
12. **Manual P2P / admin payment** — future auditable `MANUAL_ADMIN_PAYMENT` only, or reject entirely (no scraping).

## 4. Later (deferred by decision — `D-43`)
- Mobile stack (Android/iOS full app); game-currency vendor integrations.
- Private DM; community/lesson video; advanced follow/group systems.
- Granular methodist scope (English → Grammar level); automatic post classification.
- Sophisticated notification segmentation; advanced AI features.
- Full lesson-builder UX (MVP content authoring can start simpler).
