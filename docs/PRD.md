# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O Order Management System (OMS) hoje não tem nenhum mecanismo de notificação externa, evento ou fila — clientes que precisam saber quando um pedido muda de status são obrigados a fazer polling periódico em `GET /orders` (confirmado por varredura do código, ver `aux_docs/analise-codigo-atual.md`, seção 12). Esta feature introduz um **Sistema de Webhooks de Notificação de Pedidos**: sempre que o status de um pedido muda, o OMS notifica ativamente os endpoints HTTP cadastrados pelos clientes B2B, de forma assíncrona, segura e com garantia de entrega.

A solução foi definida em reunião técnica entre Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma) e Sofia (Eng. Segurança) — ver `TRANSCRICAO.md`. Em alto nível: a mudança de status insere um evento numa tabela outbox dentro da mesma transação já existente; um worker separado processa essa fila com retry e Dead Letter Queue; a entrega é assinada (HMAC-SHA256) e identificada de forma única para deduplicação do lado do cliente. O detalhamento técnico completo está em `docs/RFC.md`, `docs/FDD.md` e nos ADRs em `docs/adrs/`.

## Problema e Motivação

Três clientes B2B integrados à plataforma — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — formalizaram, na semana anterior à reunião, um pedido conjunto de notificação em tempo real de mudança de status de pedido (`TRANSCRICAO.md [09:00] Marcos`).

- **Custo operacional do polling**: os clientes hoje ficam "batendo" em `GET /orders` repetidamente para detectar mudanças, o que Marcos descreve como algo que "tá deixando a integração lenta e cara pra eles" (`TRANSCRICAO.md [09:00]`).
- **Definição de "tempo real" do cliente**: qualquer latência abaixo de **10 segundos** já atende à expectativa — não é um requisito de streaming, é eliminar a espera manual e o polling constante (`TRANSCRICAO.md [09:02] Marcos`).
- **Risco comercial concreto**: a Atlas Comercial sinalizou que pode migrar para um concorrente caso a feature não seja entregue até o fim do trimestre (`TRANSCRICAO.md [09:00] Marcos`).

A oportunidade é resolver isso com um mecanismo outbound de notificação (webhook) que elimina o polling, opera de forma assíncrona para não comprometer a transação crítica de mudança de status já existente em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), e é seguro o suficiente para expor dados de pedido a sistemas de terceiros.

## Público-Alvo e Cenários de Uso

**Cliente B2B integrado** (ex.: Atlas Comercial, MaxDistribuição, Nova Cargo): empresa que consome a API do OMS para acompanhar pedidos que fez ou que estão sendo processados para ela. Não tem login próprio no sistema — quem opera a configuração do webhook em nome dele é um usuário interno do OMS (`TRANSCRICAO.md [09:31]-[09:32] Marcos/Bruno/Larissa`).

**Operador/Administrador interno do OMS**: usuário autenticado (`ADMIN` ou `OPERATOR`) que cadastra e mantém a configuração de webhook de um customer, e que — no caso do papel `ADMIN` — pode reprocessar manualmente eventos que falharam definitivamente.

**Cenários de uso:**
1. Um operador cadastra um webhook para a Atlas Comercial, informando a URL do endpoint dela e os status de pedido que ela quer acompanhar (ex.: `SHIPPED`, `DELIVERED`).
2. Um pedido da Atlas muda de `PROCESSING` para `SHIPPED`; o sistema notifica automaticamente o endpoint cadastrado, sem que a Atlas precise consultar `GET /orders`.
3. O endpoint da Atlas fica temporariamente fora do ar; o sistema tenta novamente com backoff, sem perder o evento.
4. A Atlas suspeita que sua secret vazou e solicita rotação via API, sem perder notificações durante a transição.
5. Um desenvolvedor da Atlas consulta o histórico de entregas de um webhook para depurar por que uma notificação não chegou.
6. Um evento esgota as tentativas e cai em Dead Letter Queue; um `ADMIN` do OMS o reprocessa manualmente após confirmar que o problema do lado do cliente foi resolvido.

## Objetivos e Métricas de Sucesso

- **Eliminar** o polling como mecanismo primário de sincronização de status para os clientes B2B integrados. *Métrica:* 100% dos 3 clientes iniciais (Atlas, MaxDistribuição, Nova Cargo) migrados para consumo via webhook até o prazo comercial acordado (fim de novembro, `TRANSCRICAO.md [09:45] Marcos`).
- **Entregar** notificações dentro da janela definida pelo cliente como "tempo real". *Métrica:* latência entre a mudança de status e a primeira tentativa de entrega ≤10s em pelo menos 95% das transições, medida em homologação e nas primeiras semanas de produção (`TRANSCRICAO.md [09:02], [09:09]-[09:10]`).
- **Garantir** consistência entre mudança de status e emissão do evento. *Métrica:* 0 casos de status alterado sem o evento correspondente ter sido gerado (quando há webhook interessado), verificado por teste de integração dedicado (ver `docs/FDD.md`, seção 10).
- **Cobrir** cenários reais de indisponibilidade temporária de cliente sem perda de evento. *Métrica:* janela de retry de ~15h (5 tentativas, backoff `1m/5m/30m/2h/12h`) antes de mover para Dead Letter Queue (`TRANSCRICAO.md [09:15]-[09:17]`).
- **Entregar** dentro do prazo comercial combinado. *Métrica:* 3 sprints de desenvolvimento, incluindo pelo menos 2 dias úteis de revisão de segurança dedicada antes do deploy (`TRANSCRICAO.md [09:45]-[09:46]`).

## Escopo

### Incluso
- Cadastro, edição, remoção e listagem de webhooks por customer, com filtro de quais status de pedido cada webhook quer receber.
- Entrega assíncrona via padrão Outbox + worker separado, com retry exponencial e Dead Letter Queue.
- Endpoint administrativo de replay manual de eventos em DLQ (restrito a `ADMIN`).
- Histórico de entregas por webhook (últimas ~100), consultável via API.
- Assinatura HMAC-SHA256 dos payloads, secret exclusiva por endpoint, com rotação via API e grace period de 24h.
- Garantia de entrega at-least-once, com `X-Event-Id` único para deduplicação do lado do cliente.

### Fora de Escopo
- **Notificação por e-mail** em caso de falhas repetidas de entrega ao cliente — descartada nesta fase; possível fase futura, após medir o impacto real (`TRANSCRICAO.md [09:37]-[09:38] Marcos/Larissa`).
- **Rate limiting de envio** por cliente (proteção contra rajadas de mudanças de status gerando rajadas de chamadas HTTP) — reconhecido como risco real, mas decisão adiada para "observar e decidir depois" (`TRANSCRICAO.md [09:38]-[09:39] Diego/Larissa`).
- **Dashboard visual** para o cliente acompanhar seus webhooks — fora de escopo, ficaria a cargo de um projeto separado do time de frontend (`TRANSCRICAO.md [09:39]-[09:40] Marcos/Larissa`).
- **Garantia de ordering global** de eventos entre múltiplos workers — a arquitetura desta fase é single-worker; escalar exigiria decisão futura de particionamento (`TRANSCRICAO.md [09:12]-[09:14] Diego/Bruno/Larissa`).
- **Endurecimento dos papéis (roles)** do CRUD de configuração de webhook — hoje qualquer usuário autenticado pode gerenciar webhooks de qualquer customer; adiado para uma fase futura (`TRANSCRICAO.md [09:36]-[09:37] Sofia/Marcos`).
- **Arquivamento/retenção** de eventos já entregues na tabela outbox — explicitamente fora do escopo desta feature (`TRANSCRICAO.md [09:08] Diego`).
- **Garantia exactly-once** de entrega — descartada por complexidade; a proposta adota at-least-once com deduplicação pelo cliente (`TRANSCRICAO.md [09:25] Diego`).

## Requisitos Funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RF01 | Sistema notifica o cliente B2B quando o status de um pedido dele muda (evento `order.status_changed`) | `TRANSCRICAO.md [09:00] Marcos` |
| RF02 | Notificação deve ser tentada em até ~10s da mudança de status | `TRANSCRICAO.md [09:02] Marcos` |
| RF03 | Webhook é somente outbound (plataforma → cliente); a plataforma não recebe webhook de volta | `TRANSCRICAO.md [09:02] Sofia/Marcos` |
| RF04 | Endpoint para cadastrar webhook: URL, secret gerada pela plataforma e devolvida na criação, lista de status desejados, customer associado | `TRANSCRICAO.md [09:31]-[09:32] Marcos/Bruno` |
| RF05 | Endpoint para editar webhook cadastrado | `TRANSCRICAO.md [09:33] Bruno` |
| RF06 | Endpoint para remover webhook cadastrado | `TRANSCRICAO.md [09:33] Bruno` |
| RF07 | Endpoint para listar webhooks de um customer | `TRANSCRICAO.md [09:33] Bruno` |
| RF08 | Filtro de eventos por webhook — cada endpoint escolhe quais status de pedido quer ouvir | `TRANSCRICAO.md [09:33]-[09:34] Marcos/Bruno/Diego` |
| RF09 | Endpoint de histórico de entregas por webhook (últimas ~100, com sucesso/falha, payload, resposta, tempo de resposta) | `TRANSCRICAO.md [09:34] Marcos` |
| RF10 | Endpoint administrativo de replay manual de evento em Dead Letter Queue, restrito a role `ADMIN`, com log de auditoria de quem executou | `TRANSCRICAO.md [09:18]-[09:19], [09:35]-[09:36]` |
| RF11 | Endpoint para o cliente rotacionar a secret do webhook via API | `TRANSCRICAO.md [09:21] Sofia` |
| RF12 | Payload assinado (HMAC-SHA256) e identificado (`X-Event-Id`) para validação e deduplicação do lado do cliente | `TRANSCRICAO.md [09:20]-[09:25] Sofia/Diego` |

## Requisitos Não Funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| NFR01 | Latência-alvo de entrega: até 10s da mudança de status, com folga operacional via polling de 2s do worker | `TRANSCRICAO.md [09:02], [09:09]-[09:10]` |
| NFR02 | Atomicidade: inserção do evento e mudança de status do pedido ocorrem na mesma transação — sem inconsistência possível | `TRANSCRICAO.md [09:06], [09:40]-[09:41]` |
| NFR03 | Garantia de entrega at-least-once (não exactly-once) | `TRANSCRICAO.md [09:24]-[09:26]` |
| NFR04 | Retry: 5 tentativas com backoff `1m/5m/30m/2h/12h` antes de mover para DLQ | `TRANSCRICAO.md [09:15]-[09:17]` |
| NFR05 | Timeout de 10s por chamada HTTP de saída ao endpoint do cliente | `TRANSCRICAO.md [09:42]` |
| NFR06 | Segurança de transporte e autenticidade: TLS obrigatório (URL `https`), payload assinado HMAC-SHA256, secret por endpoint rotacionável com grace period de 24h | `TRANSCRICAO.md [09:20]-[09:23]` |
| NFR07 | Limite de payload de 64KB por evento, com erro explícito caso ultrapasse (sem truncar) | `TRANSCRICAO.md [09:23]-[09:24]` |
| NFR08 | Ordering garantida apenas por `order_id`, apenas em regime single-worker — não é garantia global | `TRANSCRICAO.md [09:12]-[09:14]` |
| NFR09 | Isolamento operacional: worker roda em processo separado da API, sobrevivendo a restart dela | `TRANSCRICAO.md [09:11]` |

## Decisões e Trade-offs Principais

- **Outbox no MySQL, não fila dedicada nem disparo síncrono**: evita infraestrutura nova (Redis/fila) para um time pequeno, ao custo de throughput/escalabilidade limitados ao MySQL existente. Detalhe em [ADR-001](./adrs/ADR-001-outbox-no-mysql.md).
- **Worker em processo separado com polling de 2s, não trigger de banco**: garante isolamento de falha da API, ao custo de uma latência mínima inerente de até 2s e de uma limitação de ordering em regime single-worker. Detalhe em [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md).
- **Retry de 5 tentativas com DLQ em tabela separada, não retry indefinido nem 3 tentativas**: cobre indisponibilidades reais já observadas (até ~2h) sem deixar eventos pendurados para sempre. Detalhe em [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).
- **Secret HMAC por endpoint, não secret global**: contém o raio de impacto de um vazamento (já houve precedente), ao custo de mais superfície de gestão de segredo. Detalhe em [ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md).
- **At-least-once com dedup por `X-Event-Id`, não exactly-once**: evita a complexidade de coordenação transacional entre plataforma e cliente, ao custo de transferir a responsabilidade de deduplicação para o cliente. Detalhe em [ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md).
- **Reuso total dos padrões existentes do projeto** (estrutura modular, `AppError`, error middleware, `requireRole`, logger Pino): reduz curva de aprendizado e risco de regressão em código compartilhado, ao custo de adaptar um padrão pensado para fluxos síncronos a um processamento assíncrono (worker). Detalhe em [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

## Dependências

- **Transação existente `OrderService.changeStatus`** (`src/modules/orders/order.service.ts`) — a feature estende esse método diretamente; qualquer mudança futura nesse método precisa considerar o webhook.
- **Revisão de segurança da Sofia**: pelo menos 2 dias úteis reservados antes do deploy, especificamente sobre HMAC e geração/rotação de secret (`TRANSCRICAO.md [09:46] Sofia`).
- **Time de Plataforma (Diego)**: responsável pelo desenho do worker e da infraestrutura de processo separado.
- **Implementação do lado do cliente**: os clientes B2B precisam implementar verificação de assinatura HMAC e deduplicação por `X-Event-Id` — dependência externa, fora do controle direto da equipe, mitigada por documentação no portal do desenvolvedor (`TRANSCRICAO.md [09:26] Marcos`).
- **Aprovação de `docs/RFC.md` e dos ADRs em `docs/adrs/`** antes do início da implementação — decisões técnicas já fechadas na reunião, mas sujeitas à sessão de design review entre Larissa, Bruno e Diego (`TRANSCRICAO.md [09:50] Larissa`).

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Atraso além do prazo comercial (fim de novembro) e perda da Atlas Comercial para concorrente | Média | Alto — perda de cliente estratégico já sinalizada explicitamente (`TRANSCRICAO.md [09:00]`) | Escopo desta fase deliberadamente contido (sem e-mail de fallback, sem dashboard, sem rate limiting); estimativa de 3 sprints acordada com o time, incluindo folga para revisão de segurança |
| Vazamento de secret de webhook expõe dados de pedido a terceiros não autorizados | Baixa (mas com precedente já ocorrido com outro cliente, `TRANSCRICAO.md [09:22] Diego`) | Alto — compromete autenticidade das notificações e confiança do cliente | Secret exclusiva por endpoint (não global); rotação via API com grace period de 24h; revisão de segurança dedicada da Sofia antes do deploy |
| Cliente não implementa deduplicação por `X-Event-Id` e processa o mesmo evento de negócio mais de uma vez | Média | Médio, mas fora do controle direto da plataforma (`TRANSCRICAO.md [09:25] Sofia`) | Documentação destacada no portal do desenvolvedor; `event_id` mantido estável em todas as tentativas de retry |
| Worker parar de processar sem alerta, acumulando atraso de notificação para todos os clientes | Baixa a média | Médio — cliente percebe atraso, risco reputacional | Processo isolado do ciclo de vida da API (ver ADR-002); observabilidade com métrica de fila pendente e alerta de estagnação (ver `docs/FDD.md`, seção 7) |

## Critérios de Aceitação

- [ ] Os 3 clientes B2B iniciais (Atlas, MaxDistribuição, Nova Cargo) conseguem cadastrar webhook e passam a receber notificação de mudança de status sem depender de polling em `GET /orders`.
- [ ] A notificação é tentada em até 10s da mudança de status em pelo menos 95% dos casos observados em homologação.
- [ ] O cliente consegue validar a autenticidade de uma notificação recebida recalculando a assinatura HMAC-SHA256 com a secret fornecida.
- [ ] O cliente consegue consultar o histórico das últimas 100 entregas de um webhook, incluindo sucesso/falha, payload e tempo de resposta.
- [ ] Um usuário `ADMIN` consegue reprocessar manualmente um evento que caiu em Dead Letter Queue; um `OPERATOR` recebe erro de permissão ao tentar.
- [ ] Nenhuma mudança de status de pedido ocorre sem o evento de webhook correspondente ser gerado, quando existe ao menos um webhook interessado naquele status.
- [ ] O cliente consegue rotacionar sua secret sem perder notificações durante a janela de 24h de transição.
- [ ] Um evento que falha 5 vezes é movido para Dead Letter Queue e deixa de ser reprocessado automaticamente.

## Estratégia de Testes e Validação

- **Teste de integração de atomicidade**: garantir que uma falha na inserção do evento de webhook reverte também a mudança de status do pedido (e vice-versa), seguindo o padrão de testes já usado em `tests/orders.test.ts` (Vitest + Supertest).
- **Teste de contrato dos endpoints**: cobertura automatizada de cada endpoint HTTP novo (payload, headers, status codes) descrito em `docs/FDD.md`, seção 5.
- **Teste de carga na transação `changeStatus`**: medir o impacto de latência da nova consulta de filtro de webhooks sobre a transação já apontada como sensível (`TRANSCRICAO.md [09:04] Bruno`), comparando antes/depois da feature.
- **Teste dirigido de assinatura HMAC e retry/backoff**: ambiente de homologação com um endpoint mock simulando sucesso, falha e timeout, validando a progressão exata do backoff (`1m/5m/30m/2h/12h`) e a movimentação para DLQ após a 5ª falha.
- **Piloto com cliente real antes do rollout geral**: validar ponta a ponta (cadastro, entrega, assinatura, deduplicação) com a Atlas Comercial antes de estender aos demais clientes.
- **Revisão de segurança dedicada**: pelo menos 2 dias úteis de revisão da Sofia sobre geração, transporte e rotação de secret antes do deploy em produção (`TRANSCRICAO.md [09:46] Sofia`).
