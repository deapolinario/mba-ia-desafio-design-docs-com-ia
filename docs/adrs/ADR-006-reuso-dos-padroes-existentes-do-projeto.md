# ADR-006: Reuso dos Padrões Arquiteturais Existentes do Projeto no Módulo de Webhooks

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Larissa (Tech Lead)
- **Tags**: arquitetura, consistência-de-codebase, webhooks

## Contexto e Problema

O OMS já tem convenções estabelecidas de estrutura de módulo, tratamento de erro, autorização e logging, usadas de forma consistente em `auth`, `users`, `customers`, `products` e `orders`. Ao introduzir um domínio novo (webhooks), era preciso decidir entre seguir essas convenções à risca ou desenhar um padrão próprio para o novo módulo, dado que ele tem características diferentes (processamento assíncrono, worker externo).

## Decisão

O módulo de webhooks **reaproveita integralmente os padrões já existentes no código**, sem introduzir convenções paralelas:

- **Estrutura modular**: novo diretório `src/modules/webhooks/` seguindo exatamente o mesmo padrão de 5 arquivos usado pelos demais domínios — `webhook.routes.ts`, `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.schemas.ts` (mesmo padrão de `src/modules/orders/`) (`TRANSCRICAO.md [09:27]-[09:28] Bruno/Diego`).
- **Hierarquia de erros**: novas classes de erro estendem `AppError` (`src/shared/errors/app-error.ts`) seguindo o mesmo padrão de `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), com códigos prefixados **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo estilo de `INSUFFICIENT_STOCK`/`INVALID_STATUS_TRANSITION` (`TRANSCRICAO.md [09:28]-[09:29] Bruno`).
- **Error middleware**: `src/middlewares/error.middleware.ts` **não é alterado** — ele já trata qualquer `instanceof AppError` de forma genérica, então cobre os novos erros do módulo automaticamente (`TRANSCRICAO.md [09:29] Bruno`).
- **Autorização**: `requireRole` (`src/middlewares/auth.middleware.ts`) é reaproveitado sem alteração para restringir `POST /admin/webhooks/dead-letter/:id/replay` a `ADMIN` (ver ADR-003).
- **Logging**: o logger Pino centralizado (`src/shared/logger/index.ts`), incluindo sua lista de `redactPaths`, é reaproveitado para logar tentativas de entrega e falhas do worker — nenhum logger novo é introduzido.
- **Validação**: schemas seguem o mesmo padrão Zod usado em `*.schemas.ts` dos demais módulos, incluindo o middleware `validate()` já existente (`src/middlewares/validate.middleware.ts`).
- **Processo/infraestrutura**: o worker abre sua própria instância de `PrismaClient`, mesma `DATABASE_URL`, seguindo o padrão já estabelecido em `src/config/database.ts` de uma instância por processo Node.

## Alternativas Consideradas

- **Desenhar um padrão de erro e de estrutura de módulo específico para webhooks** (por exemplo, por conta da natureza assíncrona do domínio), separado das convenções de `AppError`/estrutura modular usadas pelos módulos síncronos existentes: rejeitada — o time concluiu que a natureza assíncrona do processamento (worker) não justifica divergir das convenções de código já validadas no restante do projeto; a inconsistência resultante custaria mais em manutenção do que economizaria em "encaixe perfeito" para o caso assíncrono (`TRANSCRICAO.md [09:27]-[09:30] Bruno/Diego/Larissa`).

## Consequências

**Positivas:**
- Qualquer desenvolvedor já familiarizado com `orders`, `products` ou `customers` consegue navegar o módulo `webhooks` sem curva de aprendizado adicional de convenção.
- Zero mudança necessária em `error.middleware.ts` — reduz superfície de risco de regressão em código compartilhado por todos os módulos existentes.
- Auditoria e observabilidade (logs, formato de erro) ficam uniformes entre o domínio síncrono (API) e o assíncrono (worker).

**Negativas / trade-offs:**
- O padrão de módulo (`routes/controller/service/repository/schemas`) foi desenhado originalmente para fluxos request/response síncronos; aplicá-lo ao `webhook.worker.ts`/`webhook.processor.ts` (que roda em loop de polling, não em resposta a uma requisição HTTP) exige adaptar como esse arquivo se encaixa na estrutura do módulo, já que não é nem controller nem repository no sentido estrito.
- Reuso da hierarquia `AppError` amarra os erros do worker ao mesmo `statusCode` HTTP mesmo quando o "erro" não está sendo retornado como resposta HTTP a ninguém (é apenas registrado em log/DLQ) — um leve desalinhamento semântico aceito em favor da consistência.

## Referências

- `TRANSCRICAO.md [09:27]-[09:30]` (Bruno, Diego, Larissa)
- `src/modules/orders/` (estrutura de referência a ser replicada por `src/modules/webhooks/`)
- `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` (hierarquia de erro reaproveitada)
- `src/middlewares/error.middleware.ts` (não alterado)
- `src/middlewares/auth.middleware.ts` (`requireRole`, reaproveitado)
- `src/shared/logger/index.ts` (logger Pino reaproveitado)
- `src/config/database.ts` (padrão de `PrismaClient` por processo)
- Relacionado: todos os demais ADRs deste pacote (esta decisão é transversal)
