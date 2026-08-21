# Izlan — Backend Foundation (Phase 1.4A)

> Status: Phase 1.4A COMPLETE (2026-08-27). NestJS + Fastify + Prisma 7 infrastructure.
> Kod: `backend/src/`, `backend/test/`. Business domain (auth/content/learning/...) — hali YO'Q.
> Manba: [PRISMA_SCHEMA_V1.md](PRISMA_SCHEMA_V1.md), [TECH_DECISIONS.md](TECH_DECISIONS.md).

## Stack

| Komponent | Versiya |
|---|---|
| Node | 24.17.0 |
| NestJS (@nestjs/common, core, platform-fastify) | 11.2.1 |
| @nestjs/config | 4.0.4 |
| Fastify | 5.11.3 (platform-fastify orqali) |
| Prisma CLI / Client | 7.9.1 |
| @prisma/adapter-pg | 7.9.1 |
| pg | 8.23.0 |
| TypeScript | 5.9.3 |
| Test | Jest 30 + ts-jest + supertest |

**TypeScript 5.9.3 tanlovi:** Phase 1.3'da TS 7.0.2 (native preview) o'rnatilgan edi (faqat Prisma config uchun). Phase 1.4A'da NestJS ekotizimi bilan barqaror **TS 5.9.3**ga tushirildi — Nest decorators + `emitDecoratorMetadata` va `nest build` (tsc) bu versiya bilan sinovdan o'tgan; TS 7 native compiler Nest bilan hali to'liq tasdiqlanmagan. Prisma config TS 5 bilan ham ishlaydi.

## Module system: **CommonJS**

`package.json`da `type` yo'q (CommonJS). Sabab: NestJS default CommonJS'da eng barqaror (decorator metadata emit); generated Prisma Client CommonJS-mos (`main: default.js`); jest/ts-jest CommonJS bilan frictionsiz. Mixed ESM/CJS hack yo'q.

## Structure

```
backend/
  package.json         nest/jest scripts, deps
  tsconfig.json        strict; CommonJS; ES2023; decorators
  tsconfig.build.json  build (test/spec exclude → dist/main.js)
  nest-cli.json
  jest.config.js       unit (src/*.spec.ts)
  prisma.config.ts     Prisma 7 config (Phase 1.3)
  .env / .env.example / .gitignore
  prisma/schema/*      Phase 1.3 (o'zgarmagan)
  prisma/migrations/   Phase 1.3 init (o'zgarmagan)
  src/
    main.ts            Fastify bootstrap
    app.module.ts      Config + Database + Health (business modul yo'q)
    config/
      env.validation.ts    fail-fast startup validation (typed AppEnv)
      env.validation.spec.ts
      config.module.ts     global ConfigModule (validate)
    database/
      prisma.service.ts    Prisma 7 adapter, lifecycle (infrastructure only)
      database.module.ts   @Global, exports PrismaService
    health/
      health.controller.ts liveness/readiness
      health.service.ts     DB ping (SELECT 1, read-only)
      health.module.ts
      health.controller.spec.ts
  test/
    app.e2e-spec.ts    health/ready e2e
    jest-e2e.json
```

Placeholder future modul (auth/content/...) YARATILMADI — noise'ni oldini olish (§7).

## Bootstrap (main.ts)

`NestFastifyApplication` + `FastifyAdapter` (Express EMAS). Global prefix `/api`. `enableShutdownHooks()`. `validateEnv(process.env)` FastifyAdapter'dan oldin (fail-fast + `trustProxy`). `import 'dotenv/config'` — `.env` → `process.env` (bootstrap ConfigModule'dan oldin ishlaydi; prod'da `.env` yo'q bo'lsa jim o'tadi).

## Config

`@nestjs/config` global, `validate: validateEnv`. Application kodi `process.env`ga bevosita murojaat qilmaydi — `ConfigService` (§13). Faqat bootstrap boundary raw env o'qiydi.

| Env | Default | Izoh |
|---|---|---|
| NODE_ENV | development | development/test/production |
| HOST | 0.0.0.0 | |
| PORT | 3000 | 1..65535 |
| DATABASE_URL | — | **majburiy**, fake default YO'Q; postgres:// shakl sanity |
| CORS_ORIGINS | (bo'sh) | vergul bilan; bo'sh → CORS yoqilmaydi |
| TRUST_PROXY | false | §26 — deployment fazasida hal qilinadi |

Fail-fast: noto'g'ri env → startup xato (traffic'gacha). Xato xabarida qiymatlar (DATABASE_URL/parol) **yozilmaydi**.

## Database (Prisma infrastructure)

Prisma 7 **driver adapter** (§4): `DATABASE_URL → new PrismaPg({ connectionString }) → new PrismaClient({ adapter }) → PrismaService`. PrismaPg pool'ni ichida boshqaradi; `$disconnect()` uni yopadi. Accelerate/Data Proxy YO'Q — direct PostgreSQL.

- **PrismaService** — infrastructure only (§15): business query (findUser/createLesson) YO'Q; `PrismaClient`'ni extend qiladi.
- **DatabaseModule** — `@Global` (§14): PrismaService cross-cutting; global registratsiya takroriy import noise'ini yo'qotadi va "bitta managed Prisma boundary per process" (§33) ni kuchaytiradi.
- Repository/BaseRepository/UnitOfWork — YO'Q (§32/31): domain modullar bilan keladi.
- Pool tuning (DB_POOL_MAX...) — hozir yo'q; deployment fazasida (§34).

## Lifecycle

- Startup: `onModuleInit` → `$connect()` → "Database connection established". DB ulanmasa startup fail (§17), full connection string oshkor etilmaydi.
- Shutdown: `enableShutdownHooks()` SIGTERM/SIGINT → `onModuleDestroy` → `$disconnect()` → "Database connection closed". Verified: `app.close()` lifecycle testi (DB resurslar toza yopiladi).
- **Windows signal eslatmasi:** Windows haqiqiy POSIX signal bermaydi; graceful shutdown `app.close()` lifecycle bilan tasdiqlangan (enableShutdownHooks SIGTERM'da aynan shuni chaqiradi). Linux deployment'da SIGTERM to'g'ridan-to'g'ri ishlaydi.

## Health vs Readiness (§18)

- `GET /api/health` (liveness) → `200 { status: "ok" }`; DB query talab qilmaydi.
- `GET /api/ready` (readiness) → DB reachable bo'lsa `200 { status: "ready", database: "up" }`; ishlamasa `503`. Internal detallar (host/user/URL) oshkor etilmaydi. DB ping — parametrsiz `SELECT 1` (read-only, domain jadvalisiz, §20/52).
- Terminus ishlatilmadi (§19) — ikkita sodda endpoint uchun minimal custom HealthService yetarli.

## Logging

Nest built-in `Logger` (Pino/Winston YO'Q, §24). Startup log: env, host/port, DB connection, ready. Log qilinMAYDI: DATABASE_URL, OTP, token, credential.

## Security

- Secret storage: schema'da faqat hash (Phase 1.3); kod/config'da hardcoded parol/URL/secret YO'Q (grep-verified).
- `.env` gitignored; `.env.example` — faqat placeholder (real parol yo'q).
- CORS (§25): `CORS_ORIGINS` env; `origin:'*'` + credentials YO'Q; bo'sh bo'lsa CORS yoqilmaydi.
- Trust proxy (§26): default `false`; configurable; arbitrary forwarded header ishonilmaydi — deployment fazasida hal qilinadi.
- Raw body / webhook (§27): global raw-body YO'Q; kelajakdagi Click/Payme signature verification uchun arxitektura to'siq qo'ymaydi.
- Global ValidationPipe / Swagger / ExceptionFilter framework — YO'Q (§23/28/29): Nest default xato ishlovi bu faza uchun yetarli; prod'da stack/internal detallar leak qilmaydi.

## Testing

| Turi | Fayl | Qamrov |
|---|---|---|
| Unit | `src/config/env.validation.spec.ts` | 7 test — valid/fail-fast/secret-leak-yo'q |
| Unit | `src/health/health.controller.spec.ts` | 5 test — liveness, readiness up/down (503 mapping mock, §39) |
| E2e | `test/app.e2e-spec.ts` | 2 test — /api/health 200, /api/ready 200 (real DB, read-only), leak-yo'q |

Natija: **unit 12/12 PASS, e2e 2/2 PASS.** E2e DB: mavjud `izlan_dev` read-only (`SELECT 1`, mutation yo'q, §37) — alohida TEST_DATABASE_URL kelajakda planlash tavsiya etiladi. Destructive test YO'Q; readiness failure mock bilan (PostgreSQL o'ldirilmaydi).

## Deferred (Phase 1.4A ATAYLAB implement qilmaydi)

Auth/OTP/SMS/session/JWT/refresh · users/roles/permissions API · content/assessment/learning/rewards/subscription/payment/community/notification · AI · frontend · seed (Role seed keyingi controlled init fazada) · Redis/BullMQ/Passport/bcrypt · Swagger · Pino/Winston · global ValidationPipe · repository/UnitOfWork abstraksiyalari · pool tuning · TEST_DATABASE_URL · lint (Nest lint tooling hozir sozlanmagan — kod TS-strict va toza).
