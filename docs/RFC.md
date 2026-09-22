# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão — decisões técnicas fechadas em reunião ([09:00]–[09:53], ver `TRANSCRICAO.md`); revisão de segurança da Sofia e sessão de design review com Bruno/Diego pendentes antes do início da implementação (`TRANSCRICAO.md [09:50]`) |
| **Data** | 2026-09-21 (registro deste documento) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Sofia (Eng. Segurança) |

## Resumo Executivo (TL;DR)

Propomos um **Sistema de Webhooks de Notificação de Pedidos** que substitui o polling atual em `GET /orders` por notificações outbound assíncronas. A entrega usa **padrão Outbox sobre o MySQL existente** — o evento é gravado na mesma transação SQL de `OrderService.changeStatus` — processado por um **worker Node em processo separado** via polling de 2s, com **retry exponencial (5 tentativas) e Dead Letter Queue** para falhas persistentes, **assinatura HMAC-SHA256** por endpoint para autenticidade/integridade, e **garantia at-least-once** deduplicada pelo cliente via `X-Event-Id`. O módulo novo (`src/modules/webhooks`) reaproveita integralmente os padrões já existentes no projeto (estrutura modular, `AppError`, error middleware, `requireRole`, logger Pino). Estimativa: 3 sprints, incluindo revisão de segurança dedicada.

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) formalizaram pedido de notificação em tempo real quando o status de seus pedidos muda na plataforma. Hoje eles fazem polling periódico em `GET /orders`, o que é lento e caro do lado deles; a Atlas sinalizou risco de migração para um concorrente caso a feature não seja entregue até o fim do trimestre (`TRANSCRICAO.md [09:00] Marcos`). "Tempo real", nesse contexto, foi definido pelo cliente como qualquer latência abaixo de 10 segundos (`TRANSCRICAO.md [09:02] Marcos`) — não é um requisito de streaming, mas de eliminar a espera manual.

O OMS hoje não tem nenhum mecanismo de evento, fila ou notificação externa (ver `aux_docs/analise-codigo-atual.md`, seção 12) — esse vácuo é o que esta proposta preenche. O ponto de maior atenção arquitetural é que a mudança de status de pedido (`OrderService.changeStatus`, `src/modules/orders/order.service.ts:126-179`) já executa uma transação sensível (atualiza `orders`, insere em `order_status_history`, decrementa estoque); qualquer solução não pode degradar essa transação nem criar uma janela de inconsistência entre "status mudou" e "cliente foi notificado".

## Proposta Técnica

A solução é composta por cinco elementos, cada um com uma decisão registrada em ADR próprio (links acima); aqui apresentamos a visão de conjunto — o detalhamento de contratos, fluxos e matriz de erros fica no FDD.

1. **Outbox transacional** ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)): ao mudar o status de um pedido, uma linha é inserida em `webhook_outbox` na mesma transação SQL do `changeStatus`. Commitou a transação, o evento existe; deu rollback, o evento nunca existiu. A inserção já aplica o filtro de quais webhooks do customer têm interesse naquele status e grava um snapshot do payload no momento da transição, não uma referência a ser resolvida depois ([ADR-007](./adrs/ADR-007-filtro-e-snapshot-do-evento-na-insercao-da-outbox.md)).
2. **Worker assíncrono** ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)): processo Node independente (`src/worker.ts`, novo entry-point espelhando `src/server.ts`), com `PrismaClient` próprio, fazendo polling a cada 2 segundos sobre eventos pendentes e disparando as chamadas HTTP.
3. **Resiliência de entrega** ([ADR-003](./adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)): timeout de 10s por chamada; falhas entram em retry com backoff `1m/5m/30m/2h/12h` (5 tentativas); esgotadas, o evento vai para uma tabela `webhook_dead_letter` própria, reprocessável manualmente via endpoint administrativo restrito a `ADMIN`.
4. **Segurança de transporte e autenticidade** ([ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)): URL cadastrada obrigatoriamente `https`; payload assinado com HMAC-SHA256 usando secret exclusiva por endpoint (não global), com rotação via API e grace period de 24h para a secret anterior.
5. **Contrato de entrega e reuso de padrões** ([ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md), [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)): garantia at-least-once, deduplicação por `X-Event-Id` do lado do cliente; módulo novo `src/modules/webhooks` segue a mesma estrutura `controller/service/repository/routes/schemas` dos módulos existentes, reaproveitando `AppError` (códigos prefixados `WEBHOOK_*`), o error middleware central (sem alterações), `requireRole` e o logger Pino.

**Fora desta proposta** (tratado em detalhe no PRD, seção "Fora de escopo"): notificação por e-mail em falhas repetidas, rate limiting de saída, dashboard visual para o cliente e garantia de ordering global entre múltiplos workers — todos explicitamente descartados ou adiados na reunião.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo síncrono** dentro de `OrderService.changeStatus` (chamar o webhook do cliente na própria transação) | Um cliente lento ou indisponível travaria a mudança de status de outros pedidos; não há como fazer rollback de uma chamada HTTP já enviada (`TRANSCRICAO.md [09:03]-[09:04] Larissa/Bruno`) |
| **Fila dedicada (Redis Streams ou equivalente)** em vez de outbox no MySQL | Exigiria subir infraestrutura nova para um time pequeno sem capacidade operacional para isso hoje — avaliado como overengineering frente ao MySQL já disponível (`TRANSCRICAO.md [09:07] Larissa/Diego`) |
| **Garantia exactly-once** de entrega, em vez de at-least-once | Exigiria coordenação transacional entre plataforma e cliente, complexidade muito maior para resolver um problema que at-least-once + `X-Event-Id` já resolve na prática — mesmo padrão usado por Stripe e GitHub (`TRANSCRICAO.md [09:25] Diego`) |
| **Secret HMAC global da plataforma**, em vez de secret por endpoint | Uma única secret vazada comprometeria a autenticidade de todos os clientes; já houve precedente de vazamento de secret em log de aplicação de um cliente (`TRANSCRICAO.md [09:21]-[09:22] Sofia/Diego`) |

## Questões em Aberto

1. **Rate limiting de envio por cliente**: se um customer tiver muitas mudanças de status em curto intervalo (ex.: 50 em 1 minuto), o worker hoje dispararia todas as chamadas sem controle de taxa. Reconhecido como risco real na reunião, mas a decisão foi "observar e decidir depois" — não há solução definida (`TRANSCRICAO.md [09:38]-[09:39] Diego/Larissa`).
2. **Autorização do CRUD de configuração de webhook**: hoje qualquer usuário autenticado (`ADMIN` ou `OPERATOR`) pode criar/editar/remover webhooks de qualquer customer — só o replay de DLQ exige `ADMIN`. Sofia sinalizou que isso deve "endurecer" no futuro, sem definir quando ou como (`TRANSCRICAO.md [09:36]-[09:37] Sofia/Marcos`).
3. **Escalabilidade do worker além de single-instance**: a garantia de ordering por `order_id` depende de operar com um único worker; se for necessário escalar para múltiplos workers, seria preciso particionamento por `order_id` ou lock pessimista — ambos cogitados verbalmente, nenhum decidido (`TRANSCRICAO.md [09:12]-[09:14] Diego/Bruno/Larissa`).

## Impacto e Riscos

**Impacto: ALTO.** A proposta introduz um novo processo em produção (worker), uma nova superfície de segurança (endpoints externos com secret e assinatura), e uma modificação no caminho crítico existente mais sensível do sistema (`changeStatus`).

| Risco | Descrição | Mitigação proposta |
| --- | --- | --- |
| Degradação da transação de `changeStatus` | A inserção na outbox (com checagem de filtro por status) adiciona trabalho à transação já apontada como "pesada" pelo Bruno (`TRANSCRICAO.md [09:04]`) | Inserção é uma operação local ao MySQL (sem I/O externo); medir impacto em teste de carga antes do rollout |
| Vazamento de secret de cliente | Precedente já ocorrido com um cliente (`TRANSCRICAO.md [09:22] Diego`) | Secret por endpoint (não global) + rotação com grace period de 24h ([ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) limitam o raio de impacto de um vazamento |
| Worker como ponto único de processamento | Se o worker cair, eventos se acumulam na outbox sem alerta automático definido nesta fase | Processo isolado do ciclo de vida da API ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)); observabilidade (métricas/alertas) detalhada no FDD |
| Atraso de entrega para o cliente Atlas | Prazo comercial de fim de novembro; estimativa de 3 sprints incluindo 2 dias de revisão de segurança da Sofia (`TRANSCRICAO.md [09:45]-[09:46]`) | Escopo desta fase deliberadamente contido (sem e-mail de fallback, sem dashboard, sem rate limiting) para caber no prazo |

## Decisões Relacionadas

Cada elemento da proposta técnica (seção acima) corresponde a uma decisão registrada em ADR próprio:

- [ADR-001 — Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em Processo Separado com Polling](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com Backoff Exponencial e Dead Letter Queue](./adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)
- [ADR-004 — Autenticação HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Garantia de Entrega At-Least-Once com X-Event-Id](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos Padrões Arquiteturais Existentes do Projeto](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Filtro e Snapshot do Evento na Inserção da Outbox](./adrs/ADR-007-filtro-e-snapshot-do-evento-na-insercao-da-outbox.md)

## Próximos Passos

- Sessão de design review entre Larissa, Bruno e Diego antes do início da implementação (`TRANSCRICAO.md [09:50] Larissa`).
- Reserva de pelo menos 2 dias úteis de revisão de segurança da Sofia sobre HMAC e geração/rotação de secret antes do deploy (`TRANSCRICAO.md [09:46] Sofia`).
- Detalhamento de contratos, fluxos e matriz de erros no `docs/FDD.md`.
