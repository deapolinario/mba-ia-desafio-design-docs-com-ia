### FDD: Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-09-21
Responsável: Bruno (Eng. Pleno, time de Pedidos) — integração com `order.service`; Diego (Eng. Sênior, time de Plataforma) — worker e resiliência de entrega

---

### 1. Contexto e motivação técnica

O OMS hoje não tem nenhum mecanismo de evento, fila ou notificação externa (confirmado por varredura do código, ver `aux_docs/analise-codigo-atual.md`, seção 12). Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling em `GET /orders` para saber quando um pedido muda de status, o que é lento e caro do lado deles (`TRANSCRICAO.md [09:00] Marcos`). Esta feature adiciona um mecanismo de notificação outbound assíncrona, disparado pela transição de status de um pedido.

O ponto de maior sensibilidade técnica é que a mudança de status já ocorre dentro de uma transação Prisma em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que hoje faz: (1) valida a transição via `canTransition` (`src/modules/orders/order.status.ts`), (2) debita ou repõe estoque conforme a transição (`shouldDebitStock`/`shouldReplenishStock`), (3) atualiza `order.status`, (4) insere em `order_status_history`. Esta feature adiciona um quinto passo — inserção do evento de webhook na tabela `webhook_outbox` — **dentro da mesma transação**, para que a garantia de atomicidade já discutida na reunião valha (`TRANSCRICAO.md [09:40]-[09:41] Bruno/Diego`).

**Atores:**
- **Cliente B2B (customer)**: dono da configuração de webhook; recebe as chamadas HTTP outbound. Não autentica na nossa API com JWT próprio — quem opera a configuração é um usuário interno do OMS agindo em nome do customer (`TRANSCRICAO.md [09:31]-[09:32] Marcos/Bruno/Larissa`).
- **Usuário operador do OMS (`ADMIN`/`OPERATOR`)**: autentica com JWT existente; gerencia CRUD de webhooks. Endpoint de replay de DLQ exige `ADMIN`.
- **Worker (`src/worker.ts`)**: processo Node separado, sem interação humana direta, consome a outbox e realiza as entregas.
- **Endpoint do cliente**: sistema HTTP externo, fora do controle da plataforma, que recebe as chamadas assinadas.

Este documento assume as decisões já fechadas em `docs/RFC.md` e nos ADRs `docs/adrs/ADR-001` a `ADR-007`; aqui o foco é **como implementar**, não repetir o porquê.

---

### 2. Objetivos técnicos

- **Atomicidade evento-status**: 0% de casos em que o status de um pedido muda sem o evento de webhook correspondente existir na outbox (quando há ao menos um webhook ativo interessado naquele status) — invariante garantida pela inserção na mesma transação Prisma (ver ADR-001).
- **Latência de primeira tentativa**: entrega deve ser tentada em até 10s após o commit da transação de status. O polling de 2s contribui com, no pior caso, 2s de espera até o próximo ciclo (`TRANSCRICAO.md [09:10] Larissa: "a latência mínima vai ser 2 segundos no pior caso"`), deixando folga confortável frente ao requisito de "abaixo de 10s" definido pelo cliente (`TRANSCRICAO.md [09:02] Marcos`).
- **Garantia de entrega**: at-least-once, com `X-Event-Id` único e estável em todas as tentativas do mesmo evento (ver ADR-005).
- **Zero mudança em componentes compartilhados críticos**: `error.middleware.ts` deve continuar funcionando sem alteração de código para os novos erros do módulo (invariante de compatibilidade, ver seção 9).
- **Janela de resiliência**: até 5 tentativas de entrega por evento, com backoff determinístico, cobrindo cenários de indisponibilidade de cliente de até ~15h antes de mover para DLQ (ver ADR-003).

---

### 3. Escopo e exclusões

**Incluído**
- Tabela `webhook_outbox` e inserção transacional a partir de `OrderService.changeStatus`.
- Worker (`src/worker.ts` + módulo de processamento) com polling de 2s, retry com backoff e escrita em Dead Letter Queue.
- CRUD de configuração de webhook (`POST`/`GET`/`PATCH`/`DELETE`), endpoint de rotação de secret, endpoint de histórico de entregas.
- Endpoint administrativo de replay manual de DLQ (`ADMIN`).
- Assinatura HMAC-SHA256, validação de URL `https`, limite de payload de 64KB.
- Módulo `src/modules/webhooks/` seguindo o padrão arquitetural existente.

**Excluído (fora de escopo desta fase — ver `docs/RFC.md` e `docs/PRD.md` para tratamento completo)**
- Notificação por e-mail em caso de falhas repetidas (`TRANSCRICAO.md [09:37]-[09:38]`).
- Rate limiting de envio por cliente (`TRANSCRICAO.md [09:38]-[09:39]`).
- Dashboard visual para o cliente (`TRANSCRICAO.md [09:39]-[09:40]`).
- Garantia de ordering entre múltiplos workers / particionamento (`TRANSCRICAO.md [09:12]-[09:14]`).
- Arquivamento/retenção de linhas entregues na outbox após ~30 dias (`TRANSCRICAO.md [09:08] Diego`).

---

### 4. Fluxos detalhados e diagramas

**Fluxo principal — mudança de status gera evento**

1. Requisição autenticada chega em `PATCH /api/v1/orders/:id/status` (endpoint já existente, sem mudança de contrato).
2. `OrderService.changeStatus` abre a transação Prisma já existente e valida a transição (`canTransition`).
3. Executa débito/reposição de estoque conforme já implementado hoje (sem alteração).
4. **Novo passo**: consulta as configurações de webhook ativas do `customerId` do pedido cujo `events` inclui o `toStatus` da transição.
5. Se houver ao menos um webhook interessado: monta o payload (snapshot dos dados do pedido no estado pós-transição) e insere uma linha por webhook interessado em `webhook_outbox` com `status = PENDING`, `event_id` (UUID novo), `attempt_count = 0` — tudo dentro da mesma transação (ver ADR-007). Se nenhum webhook estiver interessado, nenhuma linha é inserida.
6. `order.status` é atualizado e a linha de `order_status_history` é inserida (fluxo já existente).
7. Transação comita. Se qualquer etapa (incluindo a inserção na outbox) falhar, toda a transação é revertida — status não muda e nenhum evento é criado.
8. Resposta HTTP do `PATCH` retorna normalmente, sem esperar qualquer entrega de webhook (a entrega é assíncrona).

**Fluxo do worker — processamento da outbox**

1. A cada 2 segundos, o worker consulta `webhook_outbox` filtrando `status IN (PENDING) AND (next_retry_at IS NULL OR next_retry_at <= now())`, ordenado por `created_at ASC`, em lote limitado (hipótese: `BATCH_SIZE = 20`, configurável).
2. Para cada evento do lote: monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) e envia `POST` para a `url` cadastrada, com timeout de 10s (`TRANSCRICAO.md [09:42] Sofia/Diego`).
3. **Sucesso (2xx)**: marca o evento como `DELIVERED`, grava uma linha em `webhook_delivery` (histórico) com `statusCode`, corpo de resposta (truncado), tempo de resposta.
4. **Falha (timeout, erro de rede, status não-2xx)**: incrementa `attempt_count`; se `attempt_count < 5`, calcula `next_retry_at = now() + backoff[attempt_count]` (`1m/5m/30m/2h/12h`) e mantém `status = PENDING`; grava a tentativa falha em `webhook_delivery` para o histórico.
5. **Esgotadas as 5 tentativas**: move o evento para `webhook_dead_letter` (payload, motivo da última falha, timestamp) e marca a linha da outbox como `FAILED` (ver ADR-003).

**Fluxo de replay manual de DLQ**

1. Usuário `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
2. `requireRole('ADMIN')` valida a permissão (reuso, ver seção 9).
3. Serviço busca o registro em `webhook_dead_letter`, recria uma linha em `webhook_outbox` com `status = PENDING` e `attempt_count = 0`.
4. Log estruturado de auditoria é emitido: `userId` de quem executou, `eventId` reprocessado, timestamp (`TRANSCRICAO.md [09:36] Sofia`).
5. Evento reprocessado volta a ser candidato normal do polling do worker (fluxo acima).

**Diagrama de sequência (fluxo principal, texto)**

```
Cliente API -> OrderService.changeStatus (tx Prisma)
  OrderService -> order.status.ts: canTransition(from, to)
  OrderService -> Product: debita/repõe estoque (já existente)
  OrderService -> WebhookConfigRepository: busca webhooks ativos do customer p/ status "to"
  OrderService -> WebhookOutboxRepository: insere evento(s) (snapshot + event_id)
  OrderService -> Order: update status
  OrderService -> OrderStatusHistory: insert
  OrderService -> DB: COMMIT
OrderService --> Cliente API: 200 OK (resposta não espera entrega do webhook)

[assíncrono, a cada 2s]
Worker -> webhook_outbox: SELECT pendentes/retry devidos
Worker -> Cliente externo (HTTPS): POST payload assinado
Cliente externo --> Worker: 2xx | erro/timeout
Worker -> webhook_delivery: registra tentativa
Worker -> webhook_outbox | webhook_dead_letter: atualiza estado
```

---

### 5. Contratos públicos (endpoints, headers, exemplos)

Todos os endpoints do módulo residem sob `/api/v1/webhooks` (exceto o replay administrativo) e exigem `Authorization: Bearer <jwt>` como os demais módulos do OMS (`authenticate`, ver seção 9).

**1) `POST /api/v1/webhooks` — cadastrar webhook**
- Autorização: qualquer role autenticada (`TRANSCRICAO.md [09:36]-[09:37] Sofia`).
- Status: `201 Created` | `400 WEBHOOK_INVALID_URL` | `422 WEBHOOK_INVALID_EVENT_TYPE`

Requisição:
```json
{
  "customerId": "b3f1a2c4-1111-4a5b-9c1d-000000000001",
  "url": "https://api.atlascomercial.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Resposta (`201`):
```json
{
  "id": "8f4e2a10-2222-4a5b-9c1d-000000000002",
  "customerId": "b3f1a2c4-1111-4a5b-9c1d-000000000001",
  "url": "https://api.atlascomercial.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_5f3a...c9",
  "createdAt": "2026-09-21T14:00:00.000Z"
}
```
A `secret` só é retornada em texto claro nesta resposta de criação (e na de rotação); nunca é exposta em `GET`/`PATCH` subsequentes (hipótese de segurança consistente com o requisito da Sofia de proteção de secret).

**2) `GET /api/v1/webhooks?customerId=...&page=1&pageSize=20` — listar webhooks**
- Status: `200 OK`. Segue o padrão de paginação já existente (`src/shared/http/response.ts`, `PaginatedResponse<T>`).

Resposta (`200`):
```json
{
  "data": [
    { "id": "8f4e2a10-2222-4a5b-9c1d-000000000002", "url": "https://api.atlascomercial.com/webhooks/oms", "events": ["SHIPPED", "DELIVERED"], "active": true }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**3) `PATCH /api/v1/webhooks/:id` — editar webhook**
- Status: `200 OK` | `404 WEBHOOK_NOT_FOUND` | `400 WEBHOOK_INVALID_URL`

Requisição:
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

Resposta (`200`):
```json
{
  "id": "8f4e2a10-2222-4a5b-9c1d-000000000002",
  "customerId": "b3f1a2c4-1111-4a5b-9c1d-000000000001",
  "url": "https://api.atlascomercial.com/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-09-21T15:00:00.000Z"
}
```
A `secret` não é retornada nesta resposta (apenas em criação e rotação, ver seção 5).

**4) `DELETE /api/v1/webhooks/:id` — remover webhook**
- Status: `204 No Content` | `404 WEBHOOK_NOT_FOUND`

**5) `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret**
- Status: `200 OK` | `404 WEBHOOK_NOT_FOUND`

Resposta (`200`):
```json
{
  "id": "8f4e2a10-2222-4a5b-9c1d-000000000002",
  "secret": "whsec_9a7b...e2",
  "previousSecretValidUntil": "2026-09-22T14:00:00.000Z"
}
```
`previousSecretValidUntil` reflete o grace period de 24h decidido na reunião (`TRANSCRICAO.md [09:21] Sofia`).

**6) `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20` — histórico de entregas**
- Status: `200 OK` | `404 WEBHOOK_NOT_FOUND`. Retorna até as últimas 100 entregas conforme requisito (`TRANSCRICAO.md [09:34] Marcos`).

Resposta (`200`):
```json
{
  "data": [
    {
      "eventId": "b1c2d3e4-3333-4a5b-9c1d-000000000003",
      "orderId": "c2d3e4f5-4444-4a5b-9c1d-000000000004",
      "eventType": "order.status_changed",
      "outcome": "success",
      "statusCode": 200,
      "attempt": 1,
      "durationMs": 340,
      "deliveredAt": "2026-09-21T14:05:02.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**7) `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay manual (ADMIN)**
- Autorização: `requireRole('ADMIN')` (`TRANSCRICAO.md [09:35]-[09:36] Larissa/Sofia`).
- Status: `202 Accepted` | `403 FORBIDDEN` | `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`

Resposta (`202`):
```json
{ "eventId": "b1c2d3e4-3333-4a5b-9c1d-000000000003", "status": "requeued" }
```

**8) Chamada de saída do worker → endpoint do cliente (contrato outbound)**

Headers enviados (`TRANSCRICAO.md [09:44]-[09:45] Diego/Sofia`):

| Header | Semântica |
| --- | --- |
| `X-Event-Id` | UUID único do evento, idêntico em todas as tentativas de retry, usado pelo cliente para deduplicação |
| `X-Signature` | HMAC-SHA256 do corpo da requisição, usando a secret do endpoint |
| `X-Timestamp` | Timestamp ISO 8601 do momento do envio (permite ao cliente detectar replay attack) |
| `X-Webhook-Id` | Id do cadastro de webhook, para clientes com múltiplos endpoints identificarem qual foi acionado |
| `Content-Type` | `application/json` |

Corpo (payload), formato definido em `TRANSCRICAO.md [09:43] Diego` — não inclui itens do pedido, propositalmente enxuto:
```json
{
  "event_id": "b1c2d3e4-3333-4a5b-9c1d-000000000003",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-21T14:05:00.000Z",
  "order_id": "c2d3e4f5-4444-4a5b-9c1d-000000000004",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "b3f1a2c4-1111-4a5b-9c1d-000000000001",
  "total_cents": 15000
}
```

Limites: payload máximo de 64KB (erro se ultrapassar, não trunca — `TRANSCRICAO.md [09:23]-[09:24]`); timeout de 10s por tentativa; resposta esperada do cliente é qualquer `2xx` para considerar sucesso.

---

### 6. Erros, exceções e fallback

**Matriz de erros (`AppError` com prefixo `WEBHOOK_`, seguindo `src/shared/errors/http-errors.ts`)**

| Código | HTTP | Condição | Tratamento |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook referenciado não existe | Citado literalmente na reunião (`TRANSCRICAO.md [09:28] Bruno`) |
| `WEBHOOK_INVALID_URL` | 400 | URL não é `https` ou é malformada | Citado literalmente na reunião; validação via Zod (`TRANSCRICAO.md [09:23], [09:28]`) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação exige secret e ela está ausente/inválida | Citado literalmente na reunião (`TRANSCRICAO.md [09:28] Bruno`) |
| `WEBHOOK_DUPLICATE_ENDPOINT`* | 409 | Já existe webhook ativo com mesma `url` para o `customerId` | Elaboração técnica seguindo o padrão de `ConflictError` já usado em `EMAIL_ALREADY_USED`/`SKU_ALREADY_USED` |
| `WEBHOOK_INVALID_EVENT_TYPE`* | 400 | Um valor em `events` não corresponde a um `OrderStatus` válido | Elaboração técnica, validação Zod análoga a `updateOrderStatusSchema` |
| `WEBHOOK_PAYLOAD_TOO_LARGE`* | 422 | Payload do evento excede 64KB | Decisão da reunião (`TRANSCRICAO.md [09:23]-[09:24] Sofia/Diego`); código específico é elaboração técnica |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND`* | 404 | Id de DLQ inexistente no replay | Elaboração técnica, análoga a `WEBHOOK_NOT_FOUND` |

`*` Códigos marcados com asterisco não foram citados literalmente na transcrição — são elaboração técnica necessária para cobrir os fluxos decididos, seguindo o padrão `WEBHOOK_*` explicitamente acordado (`TRANSCRICAO.md [09:28]-[09:29] Bruno/Larissa`) e o estilo de código de erro já usado no restante do projeto (`src/shared/errors/http-errors.ts`). Recomenda-se marcá-los distintamente no Tracker.

**Estratégias de resiliência**
- **Timeout**: 10s por chamada HTTP de saída (`TRANSCRICAO.md [09:42]`).
- **Retry**: backoff exponencial `1m/5m/30m/2h/12h`, 5 tentativas (ver ADR-003).
- **Backoff**: aplicado via `next_retry_at` calculado na própria linha da outbox — não há fila de atraso externa.
- **Fallback**: esgotadas as tentativas, evento vai para `webhook_dead_letter`; não há fallback automático de canal (e-mail está fora de escopo).

**Invariantes críticos**
- Nenhum evento é inserido na outbox fora de uma transação que também altera `order.status`.
- `event_id` de um evento nunca muda entre tentativas de retry do mesmo evento.
- Um evento em `webhook_dead_letter` só volta a ser processado por ação explícita de replay (nunca automaticamente).

---

### 7. Observabilidade

**Métricas**
- `webhook_outbox_pending_count` (gauge): eventos aguardando envio ou retry.
- `webhook_delivery_attempts_total{outcome="success|failure"}` (counter).
- `webhook_delivery_duration_ms` (histograma): tempo de resposta do endpoint do cliente.
- `webhook_dead_letter_total` (counter/gauge): eventos movidos para DLQ.
- `webhook_worker_poll_duration_ms` (histograma): tempo de cada ciclo de polling.

**Logs**
- Reaproveita o logger Pino central (`src/shared/logger/index.ts`) e seu formato estruturado já em uso (`http_request`, etc.).
- Cada tentativa de entrega gera um log estruturado com: `eventId`, `webhookId`, `customerId`, `orderId`, `attempt`, `outcome`, `statusCode`, `durationMs`.
- Replay de DLQ gera log de auditoria dedicado: `userId`, `eventId`, `action: "dead_letter_replay"`.
- **Extensão necessária, não decidida explicitamente na reunião**: adicionar `secret`, `X-Signature` e qualquer campo de assinatura à lista `redactPaths` já existente em `src/shared/logger/index.ts`, para nunca vazar segredo em log — consistente com o que já é feito para `password`/`token` (elaboração técnica, marcada como hipótese de implementação).

**Tracing**
- O projeto não tem infraestrutura de tracing distribuído hoje (confirmado por varredura do código). Nesta fase, propomos apenas spans internos documentados via log estruturado (correlação por `eventId`), sem introduzir uma dependência de tracing nova — decisão de manter simples, análoga à decisão de reuso máximo (ADR-006). Adoção de OpenTelemetry fica registrada como possível evolução futura, fora do escopo desta entrega.

**Dashboards e alertas mínimos**
- Alerta: `webhook_outbox_pending_count` não decresce por mais de 5 minutos (indício de worker parado).
- Alerta: taxa de `webhook_dead_letter_total` acima de um limiar por hora (indício de cliente com integração quebrada ou problema sistêmico).
- Painel: latência de entrega (p50/p95) e taxa de sucesso por `webhookId`.

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| Node.js | ≥20 | Já definido em `package.json` (`engines.node`); mesmo runtime do worker e da API |
| Prisma / `@prisma/client` | 5.22.0 | Já em uso; requer nova migration para `webhook_outbox`, `webhook_dead_letter`, `webhook_config`, `webhook_delivery` |
| MySQL | Mesma versão já usada em produção | Nenhuma extensão de banco necessária, tabelas relacionais padrão |
| Cliente HTTP para chamadas de saída | `fetch` nativo do Node ≥18 (disponível no Node 20 já usado) | Não deve exigir nova dependência de `package.json` |
| `crypto` (Node builtin) | Nativo | Usado para HMAC-SHA256, sem dependência externa nova |

**Garantias de compatibilidade**
- Nenhum contrato de endpoint existente (`orders`, `customers`, `products`, `users`, `auth`) é alterado.
- `error.middleware.ts`, `validate.middleware.ts` e `auth.middleware.ts` não sofrem alteração de assinatura ou comportamento para os módulos já existentes.
- A resposta do `PATCH /orders/:id/status` mantém o mesmo formato hoje existente; a inserção do evento de webhook é um efeito colateral interno, não observável na resposta.

---

### 9. Integração com o sistema existente

| Caminho real no código | Como o módulo de webhooks se integra |
| --- | --- |
| `src/modules/orders/order.service.ts` (método `changeStatus`, linhas 126-179) | Recebe uma nova etapa dentro da mesma transação Prisma: após validar a transição e antes/junto do `tx.order.update`, consulta configurações de webhook ativas do customer e insere evento(s) em `webhook_outbox` via uma função proposta na reunião, `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` client atual em vez de exigir a injeção de um repository inteiro no `OrderService` (`TRANSCRICAO.md [09:41] Bruno/Diego`) |
| `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts` | Novo arquivo `webhook-errors.ts` (proposto) define classes que estendem `AppError`, seguindo exatamente o padrão de `InsufficientStockError`/`InvalidStatusTransitionError`, com `errorCode` prefixado `WEBHOOK_` (ver matriz na seção 6); exportadas via `src/shared/errors/index.ts` como as demais |
| `src/middlewares/error.middleware.ts` | **Não sofre nenhuma alteração** — já trata `instanceof AppError` de forma genérica, cobrindo automaticamente qualquer erro novo do módulo webhook (`TRANSCRICAO.md [09:29] Bruno`) |
| `src/middlewares/auth.middleware.ts` (`requireRole`) | Reaproveitado sem alteração para proteger `POST /api/v1/admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')`, mesmo padrão já usado em `src/modules/users/user.routes.ts` |
| `src/shared/logger/index.ts` | Logger Pino central reaproveitado para logs de tentativa de entrega e replay de DLQ; lista `redactPaths` precisa ser estendida para incluir `secret`/assinatura (extensão de configuração, não de comportamento) |
| `src/config/database.ts` | Padrão de `PrismaClient` singleton por processo é replicado no novo `src/worker.ts`, que instancia sua própria conexão com a mesma `DATABASE_URL` |
| `src/server.ts` | Serve de modelo estrutural para o novo entry-point `src/worker.ts` (bootstrap, graceful shutdown em `SIGINT`/`SIGTERM`, log de inicialização) |
| `prisma/schema.prisma` | Recebe os novos models `WebhookConfig`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`, seguindo o padrão já usado de `id String @id @default(uuid()) @db.Char(36)` em todos os models existentes |

---

### 10. Critérios de aceite técnicos

- [ ] Teste de integração comprova que, se a inserção em `webhook_outbox` falhar dentro da transação, a mudança de `order.status` também é revertida (nenhum status muda sem o evento correspondente).
- [ ] Evento é gerado somente quando existe ao menos um webhook ativo do customer com o `toStatus` na lista `events`; nenhuma linha é inserida quando não há interessados (ADR-007).
- [ ] Em ambiente de teste, o intervalo entre commit da transação de status e a primeira tentativa de entrega é ≤ 10s em pelo menos 95% das execuções.
- [ ] Um evento reenviado por retry chega ao endpoint do cliente com o mesmo `X-Event-Id` da primeira tentativa.
- [ ] Após 5 tentativas falhas, o evento aparece em `webhook_dead_letter` e deixa de ser processado automaticamente pelo worker.
- [ ] `POST /api/v1/admin/webhooks/dead-letter/:id/replay` retorna `403` para usuário com role `OPERATOR` e `202` para `ADMIN`, com log de auditoria gravado.
- [ ] Assinatura enviada em `X-Signature` é validável do lado do cliente recalculando HMAC-SHA256 do corpo com a `secret` retornada na criação.
- [ ] Rotação de secret mantém a secret anterior válida por 24h e a nova operante imediatamente após a rotação.
- [ ] Cadastro de webhook com URL `http://` é rejeitado com `400 WEBHOOK_INVALID_URL`.
- [ ] Payload de evento acima de 64KB não é enviado e gera erro tratado (não é truncado silenciosamente).

---

### 11. Riscos e mitigação

### Latência adicional na transação de `changeStatus`

- **Probabilidade:** média
- **Impacto:** a transação já foi apontada como "pesada" na reunião (`TRANSCRICAO.md [09:04] Bruno`); uma consulta extra de configuração de webhooks poderia agravar contenção em picos de mudança de status
- **Mitigação:**
    - Indexar `webhook_config` por `customerId` e considerar índice composto incluindo o filtro de eventos
    - Medir impacto com teste de carga antes do rollout, comparando `changeStatus` com e sem a etapa nova
- **Plano de contingência:** se o impacto for inaceitável, avaliar mover a decisão de "quais webhooks notificar" para o worker (trade-off: outbox passaria a ter uma linha por transição, não por webhook, exigindo revisão do ADR-007)

### Worker como ponto único de processamento

- **Probabilidade:** baixa a média
- **Impacto:** se o processo do worker cair e não houver alerta, eventos se acumulam na outbox sem serem entregues, sem violar a garantia de atomicidade mas atrasando a notificação ao cliente
- **Mitigação:**
    - Métrica `webhook_outbox_pending_count` com alerta de estagnação (seção 7)
    - Supervisão de processo (ex.: `pm2`, `systemd`, orquestrador de containers) reiniciando o worker automaticamente em caso de crash
- **Plano de contingência:** replay manual em massa via o mesmo endpoint de DLQ, após o worker voltar, para eventos que tenham sido movidos incorretamente para falha por indisponibilidade prolongada

### Vazamento de secret de cliente

- **Probabilidade:** baixa
- **Impacto:** alto — já houve precedente de cliente vazando secret em log de aplicação dele (`TRANSCRICAO.md [09:22] Diego`)
- **Mitigação:**
    - Secret por endpoint, nunca global (ADR-004)
    - `redactPaths` do logger estendido para nunca logar secret ou assinatura do lado da plataforma
    - Rotação disponível via API a qualquer momento, com grace period de 24h
- **Plano de contingência:** revogação imediata da secret comprometida (força rotação, secret antiga invalidada antes do prazo padrão de 24h em caso de incidente confirmado)

### Cliente não implementa deduplicação por `X-Event-Id`

- **Probabilidade:** média
- **Impacto:** baixo a médio, e fora do controle direto da plataforma — cliente pode processar o mesmo evento de negócio mais de uma vez
- **Mitigação:**
    - Documentação destacada no portal do desenvolvedor (`TRANSCRICAO.md [09:26] Marcos`)
    - Manter `event_id` estável e idêntico em todas as tentativas de retry do mesmo evento
- **Plano de contingência:** nenhuma ação corretiva do lado da plataforma; risco aceito conscientemente na decisão de at-least-once (ADR-005)

### Ordering quebrada se o worker escalar para múltiplas instâncias no futuro

- **Probabilidade:** baixa nesta fase (arquitetura atual é single-worker por decisão)
- **Impacto:** médio, caso a decisão de escalar seja tomada sem revisão da estratégia de ordering
- **Mitigação:**
    - Documentar single-worker como constraint explícita desta versão (ver ADR-002, questão em aberto no RFC)
- **Plano de contingência:** implementar particionamento por `order_id` ou lock pessimista antes de qualquer escala horizontal do worker — não implementado nesta fase
