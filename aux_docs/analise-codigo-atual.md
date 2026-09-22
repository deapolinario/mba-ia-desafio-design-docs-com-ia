# Análise do Código Atual — Order Management API

> Documento de apoio (não faz parte do pacote de entrega PRD/RFC/FDD/ADRs/Tracker). Objetivo: registrar o estado real do código hoje, como insumo para (1) escrever os design docs da feature de Webhooks com precisão e (2) servir de base para um plano de melhoria técnica posterior.

Data da análise: 2026-09-21 · Branch: `develop` · Commit base: `e7f6311`

## 1. Visão geral

API REST em **Node.js 20 + TypeScript (ESM)** para um Order Management System (OMS). Arquitetura em camadas por módulo de domínio: `routes → controller → service → repository → Prisma → MySQL`. Não há filas, eventos, jobs em background ou qualquer mecanismo de notificação externa — esse é o vácuo que a feature de Webhooks (ver `TRANSCRICAO.md`) pretende preencher.

**Stack:**

| Camada | Tecnologia |
| --- | --- |
| Runtime/linguagem | Node.js ≥20, TypeScript 5.6, ESM nativo (`type: module`) |
| Framework HTTP | Express 4 |
| ORM / banco | Prisma 5.22 sobre MySQL |
| Validação | Zod 3 |
| Auth | JWT (`jsonwebtoken`) + `bcrypt` para hash de senha |
| Logging | Pino (+ `pino-http` de dependência, mas o logger é montado manualmente) |
| Testes | Vitest + Supertest |
| Lint/format | ESLint + Prettier |

Não há Redis, mensageria (SQS/RabbitMQ/Kafka), cache, cron/scheduler ou SDK de HTTP client para chamadas externas nas dependências — qualquer um desses precisará ser introduzido do zero para a feature de webhooks.

## 2. Estrutura de pastas

```
src/
  app.ts                 # monta o Express, DI manual (buildControllers), pipeline de middlewares
  server.ts              # bootstrap: listen + graceful shutdown (SIGINT/SIGTERM)
  config/
    env.ts               # validação de env vars via Zod, falha fast no boot
    database.ts           # singleton PrismaClient
  middlewares/
    auth.middleware.ts        # authenticate (JWT) + requireRole(...roles)
    error.middleware.ts       # error handler central (único ponto de tradução erro→HTTP)
    request-logger.middleware.ts  # requestId + log estruturado por request
    validate.middleware.ts    # valida body/query/params com Zod
  modules/
    auth/      # register, login, me
    users/     # apenas GET /:id (ADMIN only)
    customers/ # CRUD completo
    products/  # CRUD completo (com estoque)
    orders/    # CRUD + máquina de estados + transação de estoque
  routes/index.ts         # agrega os routers de módulo sob /api/v1
  shared/
    errors/    # AppError + subclasses tipadas por domínio de erro
    http/      # helper de paginação
    logger/    # instância Pino com redaction
prisma/
  schema.prisma           # 6 models, 2 enums
  seed.ts
tests/
  auth.test.ts, orders.test.ts, helpers/factories.ts, setup.ts
```

Cada módulo de domínio segue o mesmo padrão de 4 arquivos: `*.routes.ts` (monta o `Router`, aplica middlewares), `*.controller.ts` (thin, só chama o service e mapeia HTTP status), `*.service.ts` (regra de negócio), `*.repository.ts` (acesso a dados via Prisma). `*.schemas.ts` concentra os schemas Zod e os types inferidos (`z.infer`). Não há classe de "domain model" separada do tipo gerado pelo Prisma — os services retornam diretamente os tipos do Prisma Client.

**Injeção de dependência:** manual e centralizada em `buildControllers()` (`src/app.ts`), sem container de DI. Cada `Service` recebe seu `Repository` no construtor; `OrderService` também recebe o `PrismaClient` diretamente (necessário para orquestrar `$transaction`).

## 3. Pipeline de request (`src/app.ts`)

```
express.json({ limit: '1mb' })
  → requestLogger            (gera/propaga X-Request-Id, loga no finish)
  → GET /health               (não autenticado)
  → /api/v1/* (routers de módulo)
      → authenticate          (a maioria dos módulos aplica no router.use)
      → requireRole(...)      (só usado em users/:id)
      → validate({...})       (Zod por rota)
      → controller
  → catch-all 404 → NotFoundError
  → errorMiddleware           (único lugar que serializa erro → JSON)
```

Todo controller segue o padrão `try { ... } catch (err) { next(err) }` — nenhuma lógica de erro fica no controller, tudo converge para `errorMiddleware`.

## 4. Autenticação e autorização

- **Autenticação:** `authenticate` middleware valida `Authorization: Bearer <jwt>`, decodifica com `jsonwebtoken`, popula `req.user = { id, email, role }`. Token é assinado em `AuthService.signToken` com `JWT_SECRET`/`JWT_EXPIRES_IN` (env). Não há refresh token, blacklist/revogação ou rotação de secret — logout é responsabilidade do cliente (descartar o token).
- **Autorização por papel:** existe `requireRole(...roles)` mas **é usado em um único lugar**: `GET /api/v1/users/:id` (`ADMIN` apenas). **Todas as rotas de `orders`, `customers` e `products` exigem apenas `authenticate`** — qualquer usuário autenticado, seja `ADMIN` ou `OPERATOR`, pode criar/editar/excluir clientes, produtos e pedidos, e pode mudar status de pedido (`PATCH /orders/:id/status`) sem nenhuma restrição de papel. Isso é uma inconsistência relevante do estado atual (ver seção 8).
- Roles existentes: `ADMIN`, `OPERATOR` (enum `UserRole` no Prisma).

## 5. Modelo de dados (Prisma / MySQL)

6 tabelas, todas com PK `id CHAR(36)` (UUID gerado em app, `@default(uuid())`):

| Model | Papel | Relacionamentos | Índices |
| --- | --- | --- | --- |
| `User` | conta de operador/admin | 1—N `Order` (criador), 1—N `OrderStatusHistory` (quem mudou status) | único em `email` |
| `Customer` | cliente do pedido | 1—N `Order` | índice em `document` |
| `Product` | catálogo | 1—N `OrderItem` | únicos `sku`; índices `active`, `name` |
| `Order` | pedido | N—1 `Customer`, N—1 `User` (criador), 1—N `OrderItem`, 1—N `OrderStatusHistory` | únicos `orderNumber`; índices `customerId`, `status`, `createdAt`, `createdById` |
| `OrderItem` | item de pedido (snapshot de preço) | N—1 `Order` (cascade delete), N—1 `Product` (**sem `onDelete`, default `RESTRICT`**) | `orderId`, `productId` |
| `OrderStatusHistory` | auditoria de transição de status | N—1 `Order` (cascade delete), N—1 `User` (quem mudou) | `orderId`, `changedAt` |
| `OrderNumberSequence` | contador global para gerar `orderNumber` | — | linha única `id=1` |

Observações relevantes:
- **Preço é "congelado" no item** (`unitPriceCents`, `totalCents` em `OrderItem`) no momento da criação do pedido — mudanças posteriores no preço do produto não afetam pedidos já criados.
- **Sem soft delete** em nenhuma entidade — `delete` é hard delete físico em todos os repositories.
- **`OrderItem.product` não tem `onDelete: Cascade`** — excluir um `Product` referenciado por algum `OrderItem` vai falhar com FK constraint no MySQL (Prisma `P2003`), que **não é tratado especificamente** no `error.middleware.ts` (só `P2002` e `P2025` são tratados) — cai no branch genérico e retorna `500 INTERNAL_SERVER_ERROR` em vez de um `409`/`422` claro. Candidato a bug/melhoria.
- `Order.orderNumber` é gerado via upsert sequencial em `OrderNumberSequence` dentro da mesma transação (`ORD-000001`, `ORD-000002`, ...) — serializa a criação de pedidos nessa linha, mas é seguro contra race condition porque roda dentro do `$transaction`.

## 6. Máquina de estados de pedido (`order.status.ts`)

```
PENDING ──▶ PAID ──▶ PROCESSING ──▶ SHIPPED ──▶ DELIVERED
   │           │            │
   └─▶ CANCELLED ◀──────────┘
```

- Transições permitidas ficam num mapa estático `transitions: Record<OrderStatus, OrderStatus[]>`. `DELIVERED` e `CANCELLED` são terminais (`isTerminal`).
- `PENDING → CANCELLED`, `PAID → CANCELLED`, `PROCESSING → CANCELLED` são permitidas; **não é possível cancelar depois de `SHIPPED`**.
- **Débito de estoque** ocorre só na transição `PENDING → PAID` (`shouldDebitStock`), dentro da transação de `changeStatus`. Se algum item não tiver estoque suficiente, lança `InsufficientStockError` (422) e a transação inteira é revertida (nenhum pedido fica "meio pago").
- **Reposição de estoque** ocorre ao cancelar a partir de `PAID` ou `PROCESSING` (`shouldReplenishStock`) — ou seja, o estoque só foi debitado nesses casos, então só é reposto nesses casos. Cancelar a partir de `PENDING` não mexe em estoque (nunca foi debitado).
- Toda mudança de status grava uma linha em `OrderStatusHistory` (auditoria com `fromStatus`, `toStatus`, `changedById`, `reason` opcional, `changedAt`) **na mesma transação** do `Order.update` — não existe hoje nenhum hook, evento de domínio ou callback disparado após a mudança de status. **É exatamente este ponto (`OrderService.changeStatus`, `src/modules/orders/order.service.ts:126-179`) que será o gatilho natural para disparar webhooks**, pois é o único lugar em que uma transição de status é confirmada.
- `changeStatus` também rejeita transição para o mesmo status (`ConflictError` 409) antes mesmo de checar `canTransition`.

## 7. Tratamento de erros

Hierarquia única, tudo herda de `AppError` (`message`, `statusCode`, `errorCode`, `details?`):

```
AppError
 ├─ BadRequestError        400 BAD_REQUEST
 ├─ ValidationError        400 VALIDATION_ERROR
 ├─ UnauthorizedError       401 UNAUTHORIZED
 ├─ ForbiddenError          403 FORBIDDEN
 ├─ NotFoundError           404 NOT_FOUND
 ├─ ConflictError           409 CONFLICT
 │   └─ InvalidStatusTransitionError   409 INVALID_STATUS_TRANSITION (from/to nos details)
 └─ UnprocessableEntityError 422 UNPROCESSABLE_ENTITY
     └─ InsufficientStockError         422 INSUFFICIENT_STOCK (lista de itens indisponíveis)
```

Padrão de `errorCode`: string constante em `SCREAMING_SNAKE_CASE`, específica por caso de negócio (ex.: `EMAIL_ALREADY_USED`, `SKU_ALREADY_USED`, `INACTIVE_PRODUCT`, `INVALID_ORDER_STATE_FOR_DELETE`), não apenas o nome genérico da classe. Isso é o "padrão de código de erro" mencionado no README — qualquer novo erro de domínio (ex.: erros do sistema de webhooks) deveria seguir essa mesma convenção.

`error.middleware.ts` é o único serializador de erro → resposta HTTP. Ordem de checagem: `AppError` → `ZodError` (400 `VALIDATION_ERROR`) → `Prisma.PrismaClientKnownRequestError` (só trata `P2002` unique e `P2025` not-found) → fallback genérico `500 INTERNAL_SERVER_ERROR` com log via Pino (`logger.error`) incluindo `requestId`. Formato de resposta sempre `{ error: { code, message, details? } }`.

## 8. Validação

Todo input externo (body/query/params) passa por `validate({ body?, query?, params? })`, que roda `schema.parse()` e repassa erros do Zod para `ValidationError` com `details` no formato `{ path, message }[]`. Schemas Zod residem em `*.schemas.ts` por módulo e também são a fonte dos tipos TS (`z.infer`) usados pelos services — não há duplicação de tipos entre schema e domínio.

## 9. Logging e observabilidade

- Logger único (Pino) em `shared/logger/index.ts`, com `redact` de `authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken` — já pensado para não vazar segredo em log, útil como referência para logs de payload/assinatura de webhook.
- `request-logger.middleware.ts` gera/propaga `X-Request-Id` (aceita `x-request-id` de entrada ou gera um `uuidv4`), mede duração via `process.hrtime.bigint()`, loga `http_request` com `method`, `path`, `statusCode`, `durationMs`, `userId`.
- Não há métricas (Prometheus/StatsD), tracing distribuído, nem correlação de log entre serviços — é só log estruturado local via stdout.

## 10. Testes

- Vitest + Supertest, banco real (não mockado) via `tests/setup.ts` + `helpers/factories.ts` (helpers para criar usuário autenticado, customer, product de teste e obter uma instância do app).
- **Cobertura por módulo é desigual**: existem `auth.test.ts` (84 linhas) e `orders.test.ts` (219 linhas, cobre criação, validação de estoque insuficiente, produto inexistente/inativo, transições de status). **Não há testes para `customers`, `products` nem `users`** — é uma lacuna, especialmente porque `products`/`customers` têm regras de unicidade (SKU, email) e paginação com busca (`contains`) que hoje não são cobertas.
- Não há testes de contrato de erro genérico (ex.: garantir que erro `P2003` de FK não vaze `500` cru), nem testes de autorização (ninguém testa hoje que `OPERATOR` consegue fazer tudo em orders/customers/products, que é o comportamento atual mas talvez não o desejado).

## 11. Configuração / ambiente

`src/config/env.ts` valida `process.env` com Zod no boot (`NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET` ≥16 chars, `JWT_EXPIRES_IN`) e derruba o processo (`process.exit(1)`) se algo estiver inválido/faltando — comportamento fail-fast, sem defaults perigosos em produção (exceto secrets, que não têm default). Não há hoje nenhuma env var relacionada a serviços externos (fila, HTTP client de webhook, chave de assinatura HMAC, etc.) — tudo isso precisará ser adicionado.

## 12. O que NÃO existe hoje (gaps confirmados por grep, relevantes para a feature de Webhooks)

Busca por `webhook|notification|queue|event` em `src/` e `prisma/`: **zero ocorrências**. Confirma o que o README already diz: não há mecanismo de notificação externa, fila, evento de domínio ou emissor de webhook. Especificamente, não existem:
- Nenhum event bus / emissor de domain event.
- Nenhuma tabela para armazenar destino de webhook, assinatura, tentativas de entrega ou log de disparo.
- Nenhum cliente HTTP de saída (`fetch`/`axios`/`got`) nas dependências.
- Nenhum mecanismo de fila/retry/backoff (BullMQ, SQS, etc.) nem dependência de Redis.
- Nenhuma assinatura de payload (HMAC) ou qualquer padrão de segurança para chamadas de saída.

## 13. Pontos de reuso já existentes no código (referenciar nos design docs)

- **Gatilho de disparo:** `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`) — ponto único e transacional onde toda transição de status é confirmada; é o local natural para publicar um evento/job de webhook após o commit.
- **Padrão de erro de domínio:** estender `AppError`/`ConflictError`/`UnprocessableEntityError` com `errorCode` específico, como já feito para `InvalidStatusTransitionError` e `InsufficientStockError`.
- **Middleware `requireRole`:** já existe e está pronto para restringir endpoints administrativos de configuração de webhook (ex.: só `ADMIN` cadastra endpoints de destino) — só precisa ser aplicado.
- **`errorMiddleware` central:** qualquer novo erro (ex.: falha ao registrar um destino de webhook duplicado) só precisa de uma nova subclasse de `AppError`; não precisa tocar no middleware.
- **Logger Pino com redaction:** reaproveitável para logar tentativas de entrega de webhook sem vazar segredo de assinatura (bastaria adicionar o path do secret à lista de `redactPaths`).
- **Padrão modular `routes/controller/service/repository`:** um módulo `webhooks/` novo seguiria exatamente essa mesma estrutura de 4 arquivos + `schemas.ts`.

## 14. Inconsistências e riscos técnicos atuais (candidatos a plano de melhoria, independentes da feature de webhooks)

| # | Observação | Onde | Risco |
| --- | --- | --- | --- |
| 1 | `requireRole` só é usado em `GET /users/:id`; `orders`, `customers` e `products` não têm nenhuma restrição de papel — qualquer `OPERATOR` autenticado pode excluir clientes/produtos e mudar status de qualquer pedido | `*.routes.ts` de orders/customers/products | Autorização inconsistente; provavelmente não intencional dado que `requireRole` existe e foi usado só uma vez |
| 2 | Excluir um `Product` referenciado por `OrderItem` (sem `onDelete` definido → `RESTRICT`) gera `Prisma P2003`, não tratado no `error.middleware.ts` (só trata `P2002`/`P2025`) → cai em `500` genérico | `product.repository.ts` delete + `error.middleware.ts` | Erro operacional vira 500 em vez de 409/422 previsível; possível vazamento de detalhe de erro interno |
| 3 | Sem testes para `customers`, `products`, `users` (só `auth` e `orders` têm suíte) | `tests/` | Regressões nesses módulos não seriam pegas por CI |
| 4 | Hard delete em todas as entidades, sem soft delete/auditoria de exclusão (diferente de `OrderStatusHistory`, que audita status) | `*.repository.ts` | Perda de histórico ao excluir customer/product/order |
| 5 | JWT sem refresh token, revogação ou rotação — logout é só client-side | `auth.service.ts` | Token comprometido fica válido até expirar |
| 6 | Sem rate limiting em `/auth/login` nem em geral | `app.ts` | Exposição a força bruta de credenciais |
| 7 | `OrderNumberSequence` é uma única linha global incrementada por transação — ponto de serialização (baixo risco no volume atual, mas escala linearmente com contenção) | `order.service.ts:reserveOrderNumber` | Possível gargalo sob alta concorrência de criação de pedidos |

## 15. Resumo para uso posterior

- Este documento cobre **o que existe hoje**; não contém nenhuma decisão sobre a feature de Webhooks nem sobre o plano de melhoria — isso fica para os próximos documentos (RFC/ADRs para a feature nova; um plano à parte para os itens da seção 14).
- Toda afirmação acima é rastreável a um arquivo/linha específico do repositório nesta branch (`develop`, commit `e7f6311`) — pode ser citada diretamente no Tracker de rastreabilidade exigido pelo desafio.
