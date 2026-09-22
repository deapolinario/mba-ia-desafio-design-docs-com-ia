# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Order Management System (OMS) REST API — Node.js 20 + TypeScript (ESM, `type: module`) on Express 4, Prisma 5 over MySQL. Auth via JWT, validation via Zod, logging via Pino.

## Commands

```bash
# Local database (MySQL via Docker)
docker compose up -d
npm run db:migrate      # applies Prisma migrations
npm run db:seed         # seeds initial data

# Development
npm run dev              # tsx watch, loads .env

# Build / run production build
npm run build             # tsc -p tsconfig.build.json -> dist/
npm start                 # node dist/server.js (requires .env)

# Tests (Vitest + Supertest, run against a real MySQL DB — no mocking)
npm test                  # single run
npm run test:watch        # watch mode
npx vitest run tests/orders.test.ts              # single file
npx vitest run -t "creates an order in PENDING"  # single test by name

# Lint / format
npm run lint
npm run format
```

Tests require a running database with migrations applied (`docker compose up -d && npm run db:migrate`) — there is no mocked Prisma layer. `vitest.config.ts` forces `fileParallelism: false` and `singleFork: true`, so test files run sequentially against shared DB state; do not assume test isolation between files.

## Architecture

### Module pattern

Each domain lives in `src/modules/<domain>/` as five files: `*.routes.ts` (builds an Express `Router`, wires middlewares), `*.controller.ts` (thin — calls the service, maps HTTP status, always `try { } catch (err) { next(err) }`), `*.service.ts` (business rules), `*.repository.ts` (Prisma access only), `*.schemas.ts` (Zod schemas, also the source of inferred TS types via `z.infer` — there is no separate domain-model type). New domains should follow this exact shape (`auth`, `users`, `customers`, `products`, `orders` all do).

Dependency injection is manual, centralized in `buildControllers()` (`src/app.ts`): each `Service` receives its `Repository` in the constructor. `OrderService` additionally receives the raw `PrismaClient` because it orchestrates `$transaction`.

### Request pipeline

`express.json` → `requestLogger` (assigns/propagates `X-Request-Id`) → per-module router (`authenticate` then optionally `requireRole(...)` then `validate({ body?, query?, params? })`) → controller → catch-all 404 → `errorMiddleware`. `errorMiddleware` (`src/middlewares/error.middleware.ts`) is the single place that turns an error into an HTTP response: `AppError` instances (own `statusCode`/`errorCode`), then `ZodError`, then known Prisma error codes (currently only `P2002` unique and `P2025` not-found are special-cased — other Prisma error codes fall through to a generic `500`), then generic `500` with a Pino log. New error types should extend `AppError` (`src/shared/errors/`) with a `SCREAMING_SNAKE_CASE` domain-specific code (e.g. `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) rather than reusing a generic one — `errorMiddleware` needs no changes to support a new `AppError` subclass.

`requireRole` exists in `src/middlewares/auth.middleware.ts` but today is applied to exactly one route (`GET /users/:id`, `ADMIN` only). Every other route on `orders`, `customers`, `products` only calls `authenticate` — any authenticated user regardless of role (`ADMIN` or `OPERATOR`) can mutate them, including changing order status. Don't assume role-based restriction exists on a route just because `requireRole` is imported elsewhere in the codebase — check the specific route file.

### Order state machine

`src/modules/orders/order.status.ts` defines the allowed transitions (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, with `CANCELLED` reachable from `PENDING`/`PAID`/`PROCESSING` but not after `SHIPPED`). `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) runs the whole transition inside one Prisma `$transaction`: validates the transition, debits stock only on `PENDING → PAID`, replenishes stock only when cancelling from `PAID`/`PROCESSING` (stock was never debited from `PENDING`), updates `order.status`, and appends a row to `OrderStatusHistory` — all atomically. This transaction is the single point where a status transition is confirmed; anything that needs to react to an order status change (e.g. the webhook notification feature, see below) belongs here, inside the same transaction, not bolted on afterward.

### Data model notes worth knowing before touching Prisma

- Every model uses `id String @id @default(uuid()) @db.Char(36)` — follow this convention for new tables.
- `OrderItem` snapshots `unitPriceCents`/`totalCents` at creation time; later product price changes don't retroactively affect existing orders.
- No soft deletes anywhere — `delete` is a hard delete in every repository.
- `OrderItem.product` has no `onDelete` clause (defaults to `RESTRICT`): deleting a `Product` referenced by an order item throws a Prisma `P2003`, which `error.middleware.ts` does not special-case, so it currently surfaces as a generic `500` instead of a `409`/`422`.

### Config, logging, testing conventions

- `src/config/env.ts` validates `process.env` with Zod at boot and calls `process.exit(1)` on failure — fail fast, no silent defaults for secrets.
- Single Pino logger instance (`src/shared/logger/index.ts`) with a `redact` list for `authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken` — extend this list rather than adding ad-hoc redaction when logging new sensitive fields.
- Imports are relative with explicit `.js` extensions (ESM requirement); the `@/*` path alias defined in `tsconfig.json` is configured but not actually used anywhere in the codebase — don't introduce it inconsistently.
- Tests hit a real database via `tests/helpers/factories.ts` (creates authenticated users, customers, products) — there is no repository mocking layer to hook into.

### Planned feature: order webhook notifications

`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` and `docs/adrs/ADR-001` through `ADR-007` document an already-designed (not yet implemented) feature: outbound webhook notifications on order status change, using a transactional outbox table + separate polling worker process, HMAC-signed payloads, and retry/DLQ. If asked to implement webhooks, read `docs/FDD.md` first — it names the exact integration points in `OrderService.changeStatus` and the existing error/auth/logging patterns to reuse. `TRANSCRICAO.md` is the source meeting transcript those docs were derived from; `aux_docs/analise-codigo-atual.md` and `aux_docs/mapeamento-transcricao.md` are supporting analysis, not part of the design-doc package itself.
