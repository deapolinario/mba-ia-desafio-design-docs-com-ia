# ADR-007: Filtro de Assinatura e Snapshot do Payload no Momento da Inserção na Outbox

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Marcos (PM), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Larissa (Tech Lead)
- **Tags**: modelagem-de-dados, webhooks

## Contexto e Problema

Cada webhook cadastrado por um cliente pode escolher quais status de pedido quer ouvir (ex.: só `SHIPPED` e `DELIVERED`). Além disso, o conteúdo exato de um evento de mudança de status precisa ser definido: ele reflete o pedido "como estava no momento da mudança" ou é montado a partir do estado atual do pedido no momento do envio? Ambas as escolhas afetam diretamente o volume de dados gravado na `webhook_outbox` (ADR-001) e a corretude histórica do que é entregue ao cliente.

## Decisão

Duas decisões de modelagem foram fechadas juntas, ambas resolvidas **no momento da inserção na outbox** (dentro da mesma transação de `changeStatus`, ver ADR-001):

1. **Filtro por status na inserção, não no envio**: ao mudar o status de um pedido, o sistema verifica quais webhooks do customer têm interesse naquele status; se **nenhum** webhook do customer quiser aquele status, **nenhuma linha é inserida** na outbox para aquela mudança (`TRANSCRICAO.md [09:33]-[09:34] Marcos/Bruno/Diego`).
2. **Snapshot do payload no insert**: o conteúdo do evento (payload JSON completo, ver contrato no FDD) é renderizado e persistido **no momento da inserção**, não apenas uma referência (`order_id`) a ser resolvida depois. Se o pedido mudar novamente antes do envio, o evento já gravado continua refletindo o estado exato do momento da transição que o originou (`TRANSCRICAO.md [09:51]-[09:52] Bruno/Larissa/Diego`).

Complementarmente, o identificador da linha da outbox segue **UUID**, consistente com o padrão de chave primária usado em todo o restante do schema (`prisma/schema.prisma`, todos os models usam `@id @default(uuid())`) (`TRANSCRICAO.md [09:51] Diego/Larissa`).

## Alternativas Consideradas

- **Filtrar apenas no momento do envio** (inserir sempre na outbox e decidir na hora de disparar quais webhooks recebem): rejeitada por gerar linhas na outbox que nunca seriam entregues a ninguém, desperdiçando espaço e exigindo o worker repetir a lógica de filtro a cada leitura (`TRANSCRICAO.md [09:34] Bruno/Diego`).
- **Guardar apenas `order_id` e renderizar o payload em tempo de envio**: rejeitada porque, se o pedido fosse alterado entre a mudança de status e o envio efetivo do webhook (possível dado o modelo assíncrono com retry de até ~15h — ADR-003), o payload entregue não corresponderia mais ao estado do pedido no momento da transição que originou o evento, criando um "caso esquisito" de inconsistência histórica (`TRANSCRICAO.md [09:52] Larissa`).

## Consequências

**Positivas:**
- A outbox só acumula eventos que serão de fato entregues, mantendo a tabela mais enxuta e o trabalho do worker mais simples (sem lógica de filtro a repetir a cada leitura).
- Snapshot garante que o histórico de entregas é fiel ao estado do pedido no momento exato de cada transição — auditável e reprodutível, mesmo que o pedido mude várias vezes depois.
- Consistência com o padrão de UUID já usado em todo o schema do OMS.

**Negativas / trade-offs:**
- A checagem de "quais webhooks querem este status" precisa acontecer dentro da transação de `changeStatus`, adicionando uma consulta (leitura da configuração de webhooks do customer) ao caminho crítico já sensível a performance apontado na reunião (`TRANSCRICAO.md [09:04] Bruno`).
- Payload snapshot significa que, se o formato do payload evoluir no futuro (novo campo, por exemplo), eventos antigos já persistidos na outbox/DLQ mantêm o formato antigo — qualquer consumidor de eventos históricos (ex.: reprocessamento via replay de DLQ) precisa tolerar múltiplas versões de payload ao longo do tempo.

## Referências

- `TRANSCRICAO.md [09:33]-[09:34]` (Marcos, Bruno, Diego) e `[09:51]-[09:52]` (Diego, Larissa, Bruno)
- `prisma/schema.prisma` (padrão `@id @default(uuid())` em todos os models)
- Relacionado: ADR-001 (padrão Outbox), ADR-003 (retry/DLQ, janela em que o snapshot evita divergência)
