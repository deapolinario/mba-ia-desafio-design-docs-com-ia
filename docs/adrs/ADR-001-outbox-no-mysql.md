# ADR-001: Padrão Outbox no MySQL para Eventos de Webhook

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Larissa (Tech Lead), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma)
- **Tags**: arquitetura, confiabilidade, webhooks

## Contexto e Problema

O Order Management System precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido muda, para eliminar o polling atual em `GET /orders` (`TRANSCRICAO.md [09:00] Marcos`). A mudança de status já ocorre dentro de uma transação relativamente pesada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que atualiza `orders`, insere em `order_status_history` e decrementa `stockQuantity` dos produtos. Era preciso decidir como emitir o evento de notificação sem comprometer essa transação nem introduzir inconsistência entre "status mudou" e "evento foi gerado".

## Decisão

Adotamos o **padrão Outbox sobre o MySQL já existente**: ao mudar o status de um pedido, uma linha é inserida em uma nova tabela `webhook_outbox` **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history`. Um worker separado (ver ADR-002) lê essa tabela de forma assíncrona e realiza as chamadas HTTP. Se a transação principal falhar, o evento nunca existiu; se ela comitar, o evento está garantidamente registrado — não há caminho para status mudar sem o evento correspondente, nem para o evento existir sem o status ter mudado de fato (`TRANSCRICAO.md [09:06] Diego`).

## Alternativas Consideradas

- **Disparo síncrono dentro do `OrderService`** (chamar o HTTP do cliente diretamente na transação de `changeStatus`): rejeitada porque um cliente lento ou fora do ar travaria a mudança de status de outros pedidos, e não haveria como fazer rollback de uma chamada HTTP já enviada (`TRANSCRICAO.md [09:03]-[09:04] Larissa/Bruno`).
- **Fila dedicada (Redis Streams ou equivalente)**: rejeitada por exigir subir infraestrutura nova (ex.: Redis Cluster) para um time pequeno sem essa capacidade operacional hoje — considerada overengineering frente ao MySQL já disponível (`TRANSCRICAO.md [09:07] Larissa/Diego`).

## Consequências

**Positivas:**
- Garantia de atomicidade entre mudança de status e emissão do evento, sem componente de infraestrutura novo.
- Reaproveita o `PrismaClient`/MySQL já em produção — sem custo operacional adicional de setup.
- Modelo simples de auditar: a linha da outbox é a fonte da verdade do que foi (ou será) notificado.

**Negativas / trade-offs:**
- Acopla o schema do OMS a uma tabela de infraestrutura de mensageria (a tabela `webhook_outbox` cresce com o volume de eventos e precisa de estratégia de retenção/arquivamento, explicitamente fora do escopo desta fase — `TRANSCRICAO.md [09:08] Diego`).
- Sem um broker dedicado, throughput e escalabilidade de entrega ficam limitados pela capacidade de leitura do MySQL e pelo modelo de worker único (ver ADR-002).

## Referências

- `TRANSCRICAO.md [09:03]-[09:08]` (Larissa, Bruno, Diego)
- `src/modules/orders/order.service.ts` (método `changeStatus`, ponto de inserção do evento)
- Relacionado: ADR-002 (worker em processo separado), ADR-007 (modelagem do evento na outbox)
