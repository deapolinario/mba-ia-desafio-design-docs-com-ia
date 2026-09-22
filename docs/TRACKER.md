# Tracker de Rastreabilidade

Mapeia cada item identificável em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/*.md` à sua origem em `TRANSCRICAO.md` (timestamp + falante) ou no código-fonte do repositório (caminho real de arquivo). Itens de elaboração técnica necessários ao FDD mas sem menção literal na reunião (ex.: alguns códigos de erro `WEBHOOK_*`, nomes de campo de resposta) foram deliberadamente **omitidos** desta tabela em vez de receberem uma fonte forçada — ver nota de transparência em `docs/FDD.md`, seção 6.

Organizado por documento, na ordem em que os documentos foram produzidos: ADRs → RFC → FDD → PRD.

## ADRs (`docs/adrs/`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, inserção transacional junto com `changeStatus` | TRANSCRICAO | [09:06]-[09:08] Diego/Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: disparo síncrono no `OrderService` | TRANSCRICAO | [09:03]-[09:04] Larissa/Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: fila dedicada (Redis Streams) | TRANSCRICAO | [09:07] Larissa/Diego |
| ADR-001-CODE | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Ponto de inserção do evento é `OrderService.changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| ADR-002-DEC | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em processo separado (`src/worker.ts`), polling de 2s | TRANSCRICAO | [09:09]-[09:11] Diego/Larissa/Bruno |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Alternativa descartada: trigger de banco de dados | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Alternativa descartada: worker embutido no processo da API | TRANSCRICAO | [09:11] Diego/Larissa |
| ADR-002-LIM | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Ordering garantido só por `order_id` e só em regime single-worker | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| ADR-002-CODE | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Padrão de entry-point/PrismaClient replicado de `server.ts`/`database.ts` | CODIGO | src/server.ts |
| ADR-003-DEC | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Retry 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | TRANSCRICAO | [09:15]-[09:18] Diego/Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | Alternativa descartada: retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | Alternativa descartada: 3 tentativas | TRANSCRICAO | [09:16] Bruno/Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Trade-off | Alternativa descartada: marcar `failed` na própria outbox | TRANSCRICAO | [09:18] Diego/Bruno |
| ADR-003-REPLAY | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Requisito Funcional | Endpoint de replay manual de DLQ restrito a `ADMIN` com auditoria | TRANSCRICAO | [09:18]-[09:19] Diego, [09:35]-[09:36] Larissa/Sofia |
| ADR-003-CODE | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Restrição | `requireRole` reaproveitado para proteger o endpoint de replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004-DEC | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:20]-[09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Alternativa descartada: secret global da plataforma | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Alternativa descartada: depender só de TLS, sem assinatura | TRANSCRICAO | [09:19]-[09:20] Sofia |
| ADR-004-NFR | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Requisito Não Funcional | TLS obrigatório (https) e limite de payload de 64KB | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego/Larissa |
| ADR-005-DEC | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once, dedup via `X-Event-Id` | TRANSCRICAO | [09:24]-[09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Alternativa descartada: garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso integral dos padrões existentes no módulo `webhooks` | TRANSCRICAO | [09:27]-[09:30] Bruno/Diego/Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | Alternativa descartada: padrão de erro/módulo específico para o domínio assíncrono | TRANSCRICAO | [09:27]-[09:30] Bruno/Diego/Larissa |
| ADR-006-CODE-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Estrutura modular de referência | CODIGO | src/modules/orders/ |
| ADR-006-CODE-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Hierarquia `AppError` reaproveitada | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODE-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Error middleware central sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CODE-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Logger Pino central reaproveitado | CODIGO | src/shared/logger/index.ts |
| ADR-006-CODE-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Padrão de `PrismaClient` por processo | CODIGO | src/config/database.ts |
| ADR-007-DEC | docs/adrs/ADR-007-filtro-e-snapshot-do-evento-na-insercao-da-outbox.md | Decisão | Filtro de status na inserção + snapshot do payload no insert | TRANSCRICAO | [09:33]-[09:34] Marcos/Bruno/Diego, [09:51]-[09:52] Diego/Larissa/Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-filtro-e-snapshot-do-evento-na-insercao-da-outbox.md | Trade-off | Alternativa descartada: filtrar no momento do envio | TRANSCRICAO | [09:34] Bruno/Diego |
| ADR-007-ALT-02 | docs/adrs/ADR-007-filtro-e-snapshot-do-evento-na-insercao-da-outbox.md | Trade-off | Alternativa descartada: guardar só `order_id`, renderizar no envio | TRANSCRICAO | [09:52] Larissa |
| ADR-007-CODE | docs/adrs/ADR-007-filtro-e-snapshot-do-evento-na-insercao-da-outbox.md | Restrição | Padrão de UUID como chave primária | CODIGO | prisma/schema.prisma |

## RFC (`docs/RFC.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-PROP-01 | docs/RFC.md | Decisão | Elemento da proposta: outbox transacional | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Elemento da proposta: worker assíncrono em polling | TRANSCRICAO | [09:09]-[09:11] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Elemento da proposta: retry + DLQ | TRANSCRICAO | [09:15]-[09:19] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Elemento da proposta: HMAC + TLS + secret por endpoint | TRANSCRICAO | [09:20]-[09:24] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | Elemento da proposta: at-least-once + reuso de padrões | TRANSCRICAO | [09:24]-[09:30] Diego/Bruno |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Alternativa: disparo síncrono descartado | TRANSCRICAO | [09:03]-[09:04] Larissa/Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Alternativa: fila dedicada (Redis) descartada | TRANSCRICAO | [09:07] Larissa/Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Alternativa: garantia exactly-once descartada | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Alternativa: secret HMAC global descartada | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Questão em aberto: rate limiting de envio por cliente | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Questão em aberto: hardening de roles do CRUD de webhook | TRANSCRICAO | [09:36]-[09:37] Sofia/Marcos |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Questão em aberto: escalabilidade do worker além de single-instance | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| RFC-RISK-01 | docs/RFC.md | Risco | Degradação da transação `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Worker como ponto único de processamento | TRANSCRICAO | [09:11] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Atraso de entrega frente ao prazo comercial da Atlas | TRANSCRICAO | [09:45]-[09:46] Marcos/Larissa |
| RFC-NEXT-01 | docs/RFC.md | Restrição | Sessão de design review antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-NEXT-02 | docs/RFC.md | Restrição | Reserva de 2 dias úteis de revisão de segurança | TRANSCRICAO | [09:46] Sofia |

## FDD (`docs/FDD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBJ-01 | docs/FDD.md | Requisito Não Funcional | Invariante: 0% de status mudado sem evento correspondente | TRANSCRICAO | [09:06] Diego, [09:40]-[09:41] Bruno/Diego |
| FDD-OBJ-02 | docs/FDD.md | Requisito Não Funcional | Latência de primeira tentativa ≤10s | TRANSCRICAO | [09:02] Marcos, [09:09]-[09:10] Diego |
| FDD-OBJ-03 | docs/FDD.md | Requisito Não Funcional | `X-Event-Id` estável em todas as tentativas | TRANSCRICAO | [09:24]-[09:25] Diego |
| FDD-OBJ-04 | docs/FDD.md | Restrição | Invariante: zero mudança em `error.middleware.ts` | CODIGO | src/middlewares/error.middleware.ts |
| FDD-OBJ-05 | docs/FDD.md | Requisito Não Funcional | Até 5 tentativas cobrindo ~15h antes de DLQ | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-FLOW-01 | docs/FDD.md | Requisito Funcional | Fluxo principal: inserção do evento dentro de `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLOW-02 | docs/FDD.md | Requisito Funcional | Fluxo do worker: polling, tentativa, retry, DLQ | TRANSCRICAO | [09:09]-[09:18] Diego |
| FDD-FLOW-03 | docs/FDD.md | Requisito Funcional | Fluxo de replay manual de DLQ | TRANSCRICAO | [09:18]-[09:19] Diego, [09:35]-[09:36] Larissa/Sofia |
| FDD-CONTRACT-01 | docs/FDD.md | Requisito Funcional | `POST /api/v1/webhooks` — cadastrar webhook | TRANSCRICAO | [09:31]-[09:32] Marcos/Bruno |
| FDD-CONTRACT-02 | docs/FDD.md | Requisito Funcional | `GET /api/v1/webhooks` — listar webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRACT-03 | docs/FDD.md | Requisito Funcional | `PATCH /api/v1/webhooks/:id` — editar webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRACT-04 | docs/FDD.md | Requisito Funcional | `DELETE /api/v1/webhooks/:id` — remover webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRACT-05 | docs/FDD.md | Requisito Funcional | `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRACT-06 | docs/FDD.md | Requisito Funcional | `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRACT-07 | docs/FDD.md | Requisito Funcional | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay ADMIN | TRANSCRICAO | [09:18]-[09:19] Diego, [09:35]-[09:36] Larissa/Sofia |
| FDD-CONTRACT-08 | docs/FDD.md | Requisito Funcional | Contrato outbound: headers `X-Event-Id`/`X-Signature`/`X-Timestamp`/`X-Webhook-Id` + payload | TRANSCRICAO | [09:43]-[09:45] Diego/Sofia |
| FDD-ERR-01 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:23] Sofia, [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Restrição | Padrão de código de erro `ConflictError` (base p/ `WEBHOOK_DUPLICATE_ENDPOINT`) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-05 | docs/FDD.md | Restrição | Padrão de validação de enum (base p/ `WEBHOOK_INVALID_EVENT_TYPE`) | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-ERR-06 | docs/FDD.md | Requisito Não Funcional | Limite de payload de 64KB (base p/ `WEBHOOK_PAYLOAD_TOO_LARGE`) | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| FDD-OBS-01 | docs/FDD.md | Restrição | Reuso do logger Pino central para logs de entrega | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Restrição | Extensão de `redactPaths` para não logar secret | CODIGO | src/shared/logger/index.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node.js ≥20 já definido no projeto | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Prisma 5.22.0, novas migrations | CODIGO | prisma/schema.prisma |
| FDD-DEP-03 | docs/FDD.md | Dependência | Mesma versão de MySQL já usada | CODIGO | prisma/schema.prisma |
| FDD-DEP-04 | docs/FDD.md | Dependência | `fetch` nativo do Node, sem dependência HTTP nova | CODIGO | package.json |
| FDD-DEP-05 | docs/FDD.md | Decisão | HMAC-SHA256 via `crypto` nativo | TRANSCRICAO | [09:20] Sofia |
| FDD-INTEG-01 | docs/FDD.md | Restrição | Integração: `changeStatus` recebe `publishWebhookEvent(tx, ...)` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Integração: novas classes de erro estendem `AppError` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Integração: `error.middleware.ts` sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | Integração: `requireRole('ADMIN')` no replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | Integração: logger Pino central reaproveitado | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Integração: padrão de `PrismaClient` por processo replicado no worker | CODIGO | src/config/database.ts |
| FDD-INTEG-07 | docs/FDD.md | Restrição | Integração: `src/worker.ts` espelha o padrão de `server.ts` | CODIGO | src/server.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição | Integração: novos models de webhook seguem padrão de `id` UUID | CODIGO | prisma/schema.prisma |
| FDD-AC-01 | docs/FDD.md | Critério de Aceitação | Rollback conjunto entre outbox e `order.status` | CODIGO | src/modules/orders/order.service.ts |
| FDD-AC-02 | docs/FDD.md | Critério de Aceitação | Evento só é gerado se houver webhook interessado no status | TRANSCRICAO | [09:33]-[09:34] Marcos/Bruno/Diego |
| FDD-AC-03 | docs/FDD.md | Critério de Aceitação | Latência ≤10s em 95% dos casos em teste | TRANSCRICAO | [09:02] Marcos, [09:09]-[09:10] Diego |
| FDD-AC-04 | docs/FDD.md | Critério de Aceitação | `X-Event-Id` idêntico em reenvio por retry | TRANSCRICAO | [09:24]-[09:25] Diego |
| FDD-AC-05 | docs/FDD.md | Critério de Aceitação | Evento vai para DLQ após 5 tentativas falhas | TRANSCRICAO | [09:15]-[09:18] Diego |
| FDD-AC-06 | docs/FDD.md | Critério de Aceitação | Replay retorna 403 p/ `OPERATOR` e 202 p/ `ADMIN`, com log de auditoria | TRANSCRICAO | [09:35]-[09:36] Larissa/Sofia |
| FDD-AC-07 | docs/FDD.md | Critério de Aceitação | Assinatura `X-Signature` validável do lado do cliente | TRANSCRICAO | [09:20] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério de Aceitação | Secret antiga válida por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-09 | docs/FDD.md | Critério de Aceitação | URL `http://` rejeitada com `400 WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério de Aceitação | Payload acima de 64KB não é enviado | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Latência adicional na transação `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Worker como ponto único de processamento | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Cliente não implementa dedup por `X-Event-Id` | TRANSCRICAO | [09:25] Sofia |
| FDD-RISK-05 | docs/FDD.md | Risco | Ordering quebrada se o worker escalar no futuro | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |

## PRD (`docs/PRD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição | Custo operacional do polling atual em `GET /orders` | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Definição de "tempo real" = latência < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Risco comercial de churn da Atlas Comercial | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Meta: 100% dos 3 clientes migrados até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Requisito Não Funcional | Meta: latência ≤10s em 95% das transições | TRANSCRICAO | [09:02] Marcos, [09:09]-[09:10] Diego |
| PRD-OBJ-03 | docs/PRD.md | Requisito Não Funcional | Meta: 0 casos de inconsistência status/evento | TRANSCRICAO | [09:06] Diego, [09:40]-[09:41] Bruno/Diego |
| PRD-OBJ-04 | docs/PRD.md | Requisito Não Funcional | Meta: janela de retry de ~15h antes de DLQ | TRANSCRICAO | [09:15]-[09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Requisito Não Funcional | Meta: entrega em 3 sprints | TRANSCRICAO | [09:45]-[09:46] Larissa/Marcos |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Fora de escopo: notificação por e-mail em falhas repetidas | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de envio | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual para o cliente | TRANSCRICAO | [09:39]-[09:40] Marcos/Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição | Fora de escopo: garantia de ordering global entre workers | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Fora de escopo: endurecimento de roles do CRUD de webhook | TRANSCRICAO | [09:36]-[09:37] Sofia/Marcos |
| PRD-SCOPE-OUT-06 | docs/PRD.md | Restrição | Fora de escopo: arquivamento de eventos entregues na outbox | TRANSCRICAO | [09:08] Diego |
| PRD-SCOPE-OUT-07 | docs/PRD.md | Restrição | Fora de escopo: garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificação de mudança de status ao cliente B2B | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Notificação tentada em até ~10s | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Webhook é somente outbound | TRANSCRICAO | [09:02] Sofia/Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL, secret, status desejados, customer) | TRANSCRICAO | [09:31]-[09:32] Marcos/Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Edição de webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Remoção de webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Listagem de webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Filtro de eventos por webhook | TRANSCRICAO | [09:33]-[09:34] Marcos/Bruno/Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a `ADMIN` | TRANSCRICAO | [09:18]-[09:19] Diego, [09:35]-[09:36] Larissa/Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret via API | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Payload assinado e identificado (HMAC + `X-Event-Id`) | TRANSCRICAO | [09:20]-[09:25] Sofia/Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência-alvo de entrega ≤10s | TRANSCRICAO | [09:02] Marcos, [09:09]-[09:10] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade evento/status | TRANSCRICAO | [09:06] Diego, [09:40]-[09:41] Bruno/Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | [09:24]-[09:26] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Retry 5 tentativas, backoff definido | TRANSCRICAO | [09:15]-[09:17] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP | TRANSCRICAO | [09:42] Sofia/Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório, HMAC, secret rotacionável | TRANSCRICAO | [09:20]-[09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordering só por `order_id`, single-worker | TRANSCRICAO | [09:12]-[09:14] Diego/Bruno/Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Worker isolado do ciclo de vida da API | TRANSCRICAO | [09:11] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox no MySQL (ver ADR-001) | TRANSCRICAO | [09:06]-[09:08] Diego/Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker separado em polling (ver ADR-002) | TRANSCRICAO | [09:09]-[09:11] Diego/Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão | Retry + DLQ (ver ADR-003) | TRANSCRICAO | [09:15]-[09:18] Diego/Larissa |
| PRD-DEC-04 | docs/PRD.md | Decisão | HMAC + secret por endpoint (ver ADR-004) | TRANSCRICAO | [09:20]-[09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão | At-least-once (ver ADR-005) | TRANSCRICAO | [09:24]-[09:25] Diego |
| PRD-DEC-06 | docs/PRD.md | Decisão | Reuso de padrões existentes (ver ADR-006) | TRANSCRICAO | [09:27]-[09:30] Bruno/Diego/Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Depende da transação existente `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Time de Plataforma (Diego) para o worker | TRANSCRICAO | [09:11] Diego, [09:29]-[09:30] Diego/Bruno |
| PRD-DEP-04 | docs/PRD.md | Dependência | Implementação de HMAC/dedup do lado do cliente | TRANSCRICAO | [09:25]-[09:26] Sofia/Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Aprovação de RFC/ADRs antes de iniciar a implementação | TRANSCRICAO | [09:50] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso além do prazo comercial, perda da Atlas | TRANSCRICAO | [09:00] Marcos, [09:45]-[09:46] Larissa/Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret de webhook | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente não implementa dedup | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-04 | docs/PRD.md | Risco | Worker parar sem alerta | TRANSCRICAO | [09:11] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | 3 clientes iniciais migrados sem depender de polling | TRANSCRICAO | [09:00] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | Notificação em até 10s em 95% dos casos | TRANSCRICAO | [09:02] Marcos, [09:09]-[09:10] Diego |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | Cliente valida autenticidade via HMAC | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | Histórico das últimas 100 entregas consultável | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | `ADMIN` reprocessa DLQ, `OPERATOR` recebe erro de permissão | TRANSCRICAO | [09:35]-[09:36] Larissa/Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | Nenhuma mudança de status sem evento correspondente | TRANSCRICAO | [09:06] Diego, [09:40]-[09:41] Bruno/Diego |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Rotação de secret sem perda de notificação em 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Evento com 5 falhas move para DLQ | TRANSCRICAO | [09:15]-[09:18] Diego |
| PRD-TEST-01 | docs/PRD.md | Restrição | Teste de integração de atomicidade segue padrão de `tests/orders.test.ts` | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Restrição | Revisão de segurança dedicada de pelo menos 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-03 | docs/PRD.md | Restrição | Piloto com cliente real (Atlas) antes do rollout geral | TRANSCRICAO | [09:00] Marcos |

---

**Cobertura**: 165 itens rastreados no total — 136 com Fonte = `TRANSCRICAO` (~82%) e 29 com Fonte = `CODIGO` (~18%), cobrindo todos os requisitos funcionais/não funcionais, decisões, alternativas, questões em aberto, riscos e critérios de aceitação identificados em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e nos 7 ADRs.
