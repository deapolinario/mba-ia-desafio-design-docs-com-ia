# Mapeamento Preliminar — TRANSCRICAO.md

> Documento de apoio (não faz parte do pacote de entrega). Objetivo: extrair de `TRANSCRICAO.md` toda decisão fechada, alternativa descartada, questão em aberto, requisito funcional/não funcional e ponto de integração com o código, com timestamp + falante, para servir de insumo direto aos ADRs, RFC, FDD, PRD e Tracker. Cada linha abaixo já nasce no formato que o Tracker exige (`Fonte = TRANSCRICAO`, `Localização = [hh:mm] Nome`), então parte deste conteúdo pode ser copiado quase 1:1 para `docs/TRACKER.md`.

Participantes (candidatos a revisores do RFC): Larissa (Tech Lead, facilitadora), Marcos (PM), Bruno (Eng. Pleno, time Pedidos), Diego (Eng. Sênior, time Plataforma), Sofia (Eng. Segurança).

---

## 1. As 6 decisões arquiteturais principais (mínimo exigido para os ADRs)

### D1 — Padrão Outbox no MySQL (não fila/Redis, não síncrono)
- **Decisão:** inserir o evento em uma tabela `webhook_outbox` dentro da mesma transação SQL que atualiza `orders`/`order_status_history`; um worker separado lê e dispara. `[09:06] Diego`, confirmado `[09:08] Larissa`.
- **Motivação/contexto:**
  - Síncrono descartado: transação de `changeStatus` já é pesada (update order + insert history + decrement stock); HTTP call no meio trava outras mudanças de status e não permite rollback se o cliente estiver fora do ar. `[09:04] Bruno`.
  - Time pequeno, sem capacidade operacional extra. `[09:07] Diego`.
- **Alternativa descartada 1:** disparo síncrono dentro do `OrderService`. Trade-off: bloqueia a transação de status por latência de rede de terceiro; sem opção de rollback do lado do cliente externo. `[09:03]-[09:04] Larissa/Bruno`.
- **Alternativa descartada 2:** Redis Streams (ou fila dedicada). Trade-off: exigiria subir infraestrutura nova (ex.: Redis Cluster) para um time pequeno — overengineering frente ao MySQL já existente. `[09:07] Larissa/Diego`.
- **Garantia obtida:** consistência atômica — se a transação principal comita, o evento existe; se dá rollback, o evento some junto. `[09:06] Diego`.
- **Detalhe de manutenção (fora do escopo desta feature):** arquivar linhas entregues após ~30 dias. `[09:08] Diego`.

### D2 — Worker em processo separado, em polling
- **Decisão:** processo Node separado (`src/worker.ts`, script `npm run worker`), não a mesma instância da API; conecta no mesmo banco com um `PrismaClient` próprio (PrismaClient é por processo). Polling a cada 2 segundos. `[09:09]-[09:11] Diego/Larissa/Bruno`.
- **Motivação:** se a API reinicia, não pode derrubar o worker junto. `[09:11] Diego`.
- **Alternativa descartada:** trigger de banco (MySQL não tem `LISTEN/NOTIFY` como Postgres; trigger só executa SQL, não notifica processo externo — "improvisar" escrevendo em arquivo ou batendo endpoint foi considerado e rejeitado por ser "esquisito"). `[09:09] Diego`.
- **Parâmetro decidido:** intervalo de 2s — atende a exigência de "abaixo de 10s" do cliente com folga. `[09:09]-[09:10] Diego/Marcos/Larissa`.
- **Limitação de ordering assumida (documentar, não é decisão nova):** ordenação garantida só por `order_id` e apenas enquanto for single-worker; não há garantia de ordering global. Se escalar para múltiplos workers no futuro, perde a garantia (soluções futuras cogitadas: particionar por `order_id` ou lock pessimista — não decidido agora). `[09:12]-[09:14] Diego/Bruno/Larissa`. Confirmado como aceitável porque clientes não pediram ordering global, só saber que cada pedido mudou. `[09:14] Marcos`.

### D3 — Retry com backoff exponencial + DLQ
- **Decisão:** 5 tentativas, backoff `1m / 5m / 30m / 2h / 12h` (~15h totais entre 1ª falha e última tentativa); após esgotar, evento vai para Dead Letter Queue em **tabela separada** `webhook_dead_letter` (payload, motivo da falha, timestamp). `[09:15]-[09:18] Diego/Larissa`.
- **Alternativa descartada 1:** retry indefinido. Trade-off: evento fica pendurado para sempre se o cliente sumiu. `[09:15] Diego`.
- **Alternativa descartada 2:** 3 tentativas (mais agressivo). Trade-off: cliente já teve indisponibilidade de 2h em manutenção planejada; 3 tentativas em 30 min mataria o evento cedo demais. `[09:16] Bruno/Diego`.
- **Alternativa descartada 3 (para a DLQ):** marcar como `failed` na própria tabela outbox em vez de tabela separada. Trade-off: "suja" a leitura da outbox principal; tabela separada é mais limpa para debug/reprocessamento. `[09:18] Diego/Bruno`.
- **Reprocessamento:** manual, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Exige role `ADMIN` (reaproveitando `requireRole`) e deve logar quem fez o replay (auditoria). `[09:18]-[09:19] Diego/Larissa`, `[09:35]-[09:36] Larissa/Sofia`.

### D4 — Autenticação HMAC-SHA256 com secret por endpoint
- **Decisão:** payload assinado com HMAC-SHA256, assinatura enviada no header `X-Signature`. `[09:20] Sofia`.
- **Motivação:** cliente precisa validar que a requisição veio da plataforma e que o payload não foi adulterado. `[09:19] Sofia`.
- **Secret por endpoint (não global):** se uma vazar, não compromete os demais. Precedente citado: cliente já vazou secret em log de aplicação dele. `[09:21]-[09:22] Sofia/Diego`.
- **Rotação de secret:** endpoint para o cliente pedir nova secret via API; secret antiga permanece válida por **24h em paralelo** (grace period) para permitir migração; depois disso, morre. `[09:21] Sofia`.
- **Requisito de dado derivado:** tabela de configuração de webhook armazena `url + secret + customer_id + estado ativo`. `[09:21] Bruno/Sofia`.
- **Requisitos de segurança complementares (não são ADR, viram NFR/validação de schema):**
  - TLS obrigatório — URL deve ser `https`; `http` é recusado com erro de validação Zod. `[09:23] Sofia`.
  - Limite de payload: **64KB**; se ultrapassar, erra (não trunca). `[09:23]-[09:24] Sofia/Diego/Larissa`.

### D5 — Garantia at-least-once com `X-Event-Id`
- **Decisão:** entrega garantida é at-least-once (cliente pode receber o mesmo evento 2x); dedupe é responsabilidade do cliente via `X-Event-Id` (UUID único gerado quando o evento entra na outbox). `[09:24]-[09:25] Diego`.
- **Alternativa descartada:** exactly-once. Trade-off: exigiria coordenação dos dois lados, complexidade muito maior. `[09:25] Diego`.
- **Justificativa de mercado:** citado Stripe e GitHub como precedente do mesmo padrão. `[09:25] Diego`.
- **Mitigação de "responsabilidade jogada pro cliente":** Marcos documenta isso de forma destacada no portal do desenvolvedor. `[09:25]-[09:26] Sofia/Marcos`.

### D6 — Reuso dos padrões existentes do projeto
- **Decisão:** módulo novo segue exatamente o padrão já usado (`controller/service/repository/routes/schemas` em `src/modules/webhooks`); reaproveita `AppError` e subclasses, Pino, error middleware central, padrão de schemas Zod, padrão de código de erro. `[09:27]-[09:30] Bruno/Diego/Larissa`.
- **Convenção de erro:** prefixo **`WEBHOOK_`** para todos os códigos do módulo (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), seguindo o padrão já existente de `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`. `[09:28]-[09:29] Bruno/Larissa`.
- **Confirmação explícita:** error middleware central já trata `AppError`, `ZodError` e erros do Prisma — não precisa mudar nada nele para o módulo novo funcionar. `[09:29] Bruno`.
- **Worker x Prisma:** cada processo (API e worker) abre sua própria instância de `PrismaClient`, mesma `DATABASE_URL`. `[09:29]-[09:30] Diego/Bruno`.

---

## 2. Decisões secundárias (viram ADR extra ou ficam só no FDD — a decidir na escrita)

| Decisão | Timestamp / quem | Detalhe |
| --- | --- | --- |
| Filtro de eventos por webhook | `[09:33]-[09:34] Marcos/Bruno/Diego` | Cada endpoint escolhe a lista de status que quer ouvir; filtragem acontece **na inserção do outbox** (se nenhum webhook do customer quer aquele status, nem insere) — não na hora de enviar. Economiza linha na tabela. |
| ID da outbox: UUID (não auto-incremento) | `[09:51] Diego/Larissa` | Segue padrão do resto do projeto (tudo é UUID). |
| Payload snapshot no insert (não lazy) | `[09:51]-[09:52] Bruno/Larissa/Diego` | O evento guarda o payload já renderizado no momento da inserção na outbox, não apenas `order_id` — evita que uma mudança posterior no pedido altere o conteúdo do evento histórico. |
| Ponto de integração no `changeStatus` | `[09:40]-[09:41] Bruno/Diego` | Inserção na `webhook_outbox` ocorre **dentro da mesma transação** do `changeStatus`; se a inserção falhar, dá rollback na mudança de status inteira — "não pode ter caso de status mudar e evento não sair". Proposta de função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` client atual, em vez de injetar um repository inteiro no `OrderService`. |
| Timeout do HTTP call do worker | `[09:42] Sofia/Diego` | 10 segundos; cliente que não responde nesse prazo é tratado como falha e entra no fluxo de retry. |
| Formato do payload | `[09:43] Diego` | JSON: `event_id`, `event_type` (ex. `"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, campos básicos do pedido (ex. `total_cents`). **Não inclui `items`** — cliente busca detalhe via `GET /orders/:id` se precisar. |
| Headers do request de saída | `[09:44]-[09:45] Diego/Sofia` | `X-Event-Id` (UUID), `X-Signature` (HMAC), `X-Timestamp` (momento do envio, permite ao cliente detectar replay attack), `Content-Type: application/json`, `X-Webhook-Id` (id do cadastro/endpoint, útil se o customer tiver múltiplos webhooks). |
| Roles dos endpoints CRUD de configuração | `[09:36]-[09:37] Marcos/Sofia` | CRUD normal (criar/editar/remover/listar webhook) aceita **qualquer role autenticada** por enquanto; só o replay de DLQ exige `ADMIN`. Sofia sinaliza que isso pode "endurecer" mais pra frente (não decidido agora — é observação, não requisito). |
| Autenticação do CRUD de webhook | `[09:31]-[09:32] Marcos/Bruno/Larissa` | Endpoint autenticado normal com JWT do usuário operador do sistema (não JWT do cliente B2B — cliente não tem login próprio). `customer_id` vai explícito no body/path, **não** é derivado do JWT. |
| Estimativa de entrega | `[09:45]-[09:46] Marcos/Larissa/Sofia` | 3 sprints ao todo (outbox+DLQ: 1 sprint; worker+retry: 1 sprint; CRUD config+deliveries: 0.5 sprint; integração no `order.service`+testes e2e: 0.5 sprint; HMAC/schemas/validações: resto), incluindo pelo menos 2 dias úteis de revisão de segurança da Sofia antes do deploy. Prazo alvo: fim de novembro (pedido do cliente Atlas). |

---

## 3. Requisitos funcionais explícitos (candidatos a PRD-FR / FDD)

| # | Requisito | Timestamp |
| --- | --- | --- |
| RF1 | Cliente B2B recebe notificação quando o status de um pedido dele muda (evento `order.status_changed`) | `[09:00] Marcos` |
| RF2 | Notificação deve chegar em até ~10s da mudança de status (definição de "tempo real" dada pelo cliente) | `[09:02] Marcos` |
| RF3 | Webhook é só outbound (plataforma → cliente); plataforma não recebe webhook de volta | `[09:02] Sofia/Marcos` |
| RF4 | `POST` para cadastrar webhook: `url`, `secret` (gerada pela plataforma e devolvida na criação), lista de status desejados, `customer_id` | `[09:31]-[09:32] Marcos/Bruno` |
| RF5 | `PATCH` para editar webhook cadastrado | `[09:33] Bruno` |
| RF6 | `DELETE` para remover webhook cadastrado | `[09:33] Bruno` |
| RF7 | `GET` para listar webhooks de um customer | `[09:33] Bruno` |
| RF8 | Filtro de eventos por webhook (lista de status que aquele endpoint quer ouvir), aplicado na inserção do outbox | `[09:33]-[09:34] Marcos/Bruno/Diego` |
| RF9 | `GET /webhooks/:id/deliveries` — histórico das últimas ~100 entregas (sucesso/falha, payload, response, tempo de resposta) | `[09:34] Marcos` |
| RF10 | `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar manualmente um evento em DLQ, role `ADMIN`, com log de auditoria de quem executou | `[09:18]-[09:19] Diego/Larissa`, `[09:35]-[09:36] Larissa/Sofia` |
| RF11 | Endpoint para o cliente rotacionar a secret do webhook via API | `[09:21] Sofia` |
| RF12 | Payload assinado (HMAC-SHA256) e identificado (`X-Event-Id`) para validação e deduplicação do lado do cliente | `[09:20]-[09:25] Sofia/Diego` |

## 4. Requisitos não funcionais (candidatos a PRD-NFR / FDD)

| # | Requisito | Timestamp |
| --- | --- | --- |
| NFR1 | Latência-alvo: entrega em até 10s (limite dado pelo cliente); polling de 2s do worker garante folga | `[09:02]`, `[09:09]-[09:10]` |
| NFR2 | Atomicidade: inserção do evento na outbox e mudança de status do pedido são a mesma transação — sem inconsistência possível | `[09:06]`, `[09:40]-[09:41]` |
| NFR3 | Garantia de entrega: at-least-once, não exactly-once | `[09:24]-[09:26]` |
| NFR4 | Retry: 5 tentativas, backoff `1m/5m/30m/2h/12h` | `[09:15]-[09:17]` |
| NFR5 | Timeout de chamada HTTP de saída: 10s | `[09:42]` |
| NFR6 | Segurança de transporte: TLS obrigatório (URL https), payload assinado HMAC-SHA256, secret por endpoint rotacionável (grace period 24h) | `[09:20]-[09:23]` |
| NFR7 | Limite de tamanho de payload: 64KB, erro se ultrapassar (não trunca) | `[09:23]-[09:24]` |
| NFR8 | Ordering: garantida só por `order_id`, apenas em regime single-worker — não é garantia global | `[09:12]-[09:14]` |
| NFR9 | Isolamento operacional: worker roda em processo separado da API (não cai junto em restart da API) | `[09:11]` |

## 5. Fora de escopo / explicitamente descartado ou adiado (obrigatório citar ≥2 no PRD)

| Item | Timestamp | Situação |
| --- | --- | --- |
| Notificação por e-mail em caso de falhas repetidas do webhook do cliente | `[09:37]-[09:38] Marcos/Larissa` | Descartado nesta fase; possível fase futura, após medir impacto |
| Rate limiting de envio ao cliente (ex.: 50 mudanças de status em 1 min) | `[09:38]-[09:39] Diego/Larissa` | Não entra no escopo; "observar e decidir depois" — fica como ponto em aberto, não requisito |
| Dashboard visual para o cliente acompanhar webhooks | `[09:39]-[09:40] Marcos/Larissa` | Fora de escopo; seria projeto separado do time de frontend |
| Garantia de ordering global entre workers (escalar para múltiplos workers) | `[09:12]-[09:14] Diego/Bruno/Larissa` | Adiado — solução (particionamento por `order_id` ou lock pessimista) só "é problema do futuro" |
| Endurecer roles do CRUD de configuração de webhook (hoje qualquer autenticado) | `[09:36]-[09:37] Marcos/Sofia` | Adiado — "por enquanto sim [qualquer role]. Mais pra frente a gente pode endurecer" |
| Arquivamento de eventos entregues na outbox (após ~30 dias) | `[09:08] Diego` | Explicitamente "fora do escopo dessa feature" |
| Exactly-once delivery | `[09:25] Diego` | Descartado por complexidade; at-least-once escolhido no lugar |

## 6. Questões em aberto (obrigatório citar ≥2 no RFC)

1. **Rate limiting de saída por cliente** — reconhecido como risco real (rajada de mudanças de status → rajada de chamadas HTTP), mas decisão adiada para "observar e decidir depois". `[09:38]-[09:39] Diego/Larissa`.
2. **Hardening de roles no CRUD de configuração de webhook** — hoje qualquer usuário autenticado pode gerenciar webhooks de qualquer customer; Sofia já sinaliza que isso deve mudar no futuro, sem definir quando/como. `[09:36]-[09:37] Sofia/Marcos`.
3. **Escalabilidade do worker além de single-instance** — solução de particionamento por `order_id` ou lock pessimista foi cogitada verbalmente mas não definida; fica só registrada como limitação conhecida, não resolvida. `[09:13] Diego`.
4. *(secundária)* Estratégia de arquivamento/retenção da outbox após entrega (~30 dias "ou assim") não tem detalhe definido, só a intenção. `[09:08] Diego`.

## 7. Pontos de integração com o código existente (obrigatório ≥4 caminhos reais no FDD)

| Ponto no código | Como o webhook se integra | Timestamp da decisão |
| --- | --- | --- |
| `src/modules/orders/order.service.ts` — método `changeStatus` (linha ~126, ver `aux_docs/analise-codigo-atual.md` §6) | Passa a chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação Prisma, após validar a transição e antes/depois do update de status — inserção na outbox faz parte do mesmo commit/rollback | `[09:40]-[09:41] Bruno/Diego` |
| `src/shared/errors/` (`AppError`, `http-errors.ts`, padrão de subclasses como `InsufficientStockError`/`InvalidStatusTransitionError`) | Novas classes de erro do módulo webhook estendem `AppError` com `errorCode` prefixado `WEBHOOK_` (ex.: `WebhookNotFoundError` → `WEBHOOK_NOT_FOUND`) | `[09:28]-[09:29] Bruno` |
| `src/middlewares/error.middleware.ts` | Nenhuma alteração necessária — já trata `instanceof AppError` genericamente, cobre qualquer erro novo do módulo webhook automaticamente | `[09:29] Bruno` |
| `src/middlewares/auth.middleware.ts` (`requireRole`) | Reaproveitado sem alteração para proteger `POST /admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')` | `[09:35]-[09:36] Larissa/Sofia` |
| `src/shared/logger/index.ts` (Pino + `redactPaths`) | Reaproveitado para logar tentativas de entrega/erros do worker; secret/assinatura devem ser adicionados a `redactPaths` (extensão pontual, não decidida explicitamente na call — atenção no FDD) | `[09:29] Bruno` (reuso geral do Pino) |
| Estrutura de módulos (`src/modules/<dominio>/{controller,service,repository,routes,schemas}.ts`) | Novo módulo `src/modules/webhooks/` segue exatamente o mesmo padrão de 5 arquivos | `[09:27]-[09:28] Bruno/Diego` |
| `src/config/database.ts` (padrão de `PrismaClient` singleton por processo) | Worker (`src/worker.ts`) instancia seu próprio `PrismaClient`, mesma `DATABASE_URL`, processo Node separado do `src/server.ts` | `[09:29]-[09:30] Diego/Bruno`, `[09:11] Larissa` |
| `src/server.ts` (padrão de entry-point) | Novo entry-point `src/worker.ts` + script `npm run worker`, espelhando o padrão já usado por `server.ts` | `[09:11] Larissa` |

---

## 8. Observações para a etapa de ADRs

- As 6 decisões da seção 1 (D1–D6) mapeiam diretamente para os 6 tópicos obrigatórios do README — cobertura mínima de 5/6 é trivial de atingir; dá para fazer as 6 ADRs "core" e mais 1–2 ADRs secundários (candidatos fortes: **retry+DLQ como tabela separada com replay manual** já está dentro de D3, e **filtro de eventos na inserção do outbox + payload snapshot** poderiam virar um ADR combinado de "modelagem do evento na outbox", já que ambos afetam o schema da tabela).
- Todo item de "questão em aberto" (seção 6) e "fora de escopo" (seção 5) deve aparecer no RFC — não deve virar requisito no PRD nem ser assumido como resolvido em nenhum ADR.
- Nenhuma decisão acima contradiz o código atual mapeado em `aux_docs/analise-codigo-atual.md`; os pontos de integração (seção 7) já foram cruzados manualmente com os arquivos reais do repositório.
