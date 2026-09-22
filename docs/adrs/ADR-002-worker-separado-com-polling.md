# ADR-002: Worker em Processo Separado com Polling de 2 Segundos

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)
- **Tags**: arquitetura, infraestrutura, webhooks

## Contexto e Problema

Com o padrão Outbox decidido (ADR-001), era preciso definir quem lê a tabela `webhook_outbox` e dispara as chamadas HTTP, e como esse processo é acionado. O cliente definiu "tempo real" como qualquer latência abaixo de 10 segundos (`TRANSCRICAO.md [09:02] Marcos`). Além disso, o processamento não pode ficar acoplado ao ciclo de vida da API: se a API reinicia (deploy, crash, escala), o processamento de eventos não pode parar junto (`TRANSCRICAO.md [09:11] Diego`).

## Decisão

O processamento da outbox roda em um **processo Node separado da API**, com **polling em loop a cada 2 segundos** buscando os eventos pendentes mais antigos, processando-os e marcando como entregues. O projeto ganha um novo entry-point `src/worker.ts` (espelhando o padrão já existente em `src/server.ts`) e um script `npm run worker`. Esse processo conecta-se ao mesmo banco (`DATABASE_URL`), mas instancia seu próprio `PrismaClient` — cada processo Node tem sua própria instância, seguindo o padrão de singleton por processo já usado em `src/config/database.ts` (`TRANSCRICAO.md [09:09]-[09:11], [09:29]-[09:30]`).

## Alternativas Consideradas

- **Trigger de banco de dados**: descartada porque o MySQL não tem um mecanismo nativo equivalente ao `LISTEN/NOTIFY` do PostgreSQL — um trigger só executa SQL, não consegue notificar um processo externo. Fazer o trigger "escrever em arquivo" ou "bater em um endpoint" foi considerado e rejeitado por introduzir complexidade e acoplamento desnecessários (`TRANSCRICAO.md [09:09] Diego`).
- **Worker embutido na mesma instância/processo da API**: implicitamente rejeitada — o requisito explícito era que o worker sobrevivesse a um restart da API, o que exige processo isolado (`TRANSCRICAO.md [09:11] Diego/Larissa`).

## Consequências

**Positivas:**
- Isolamento de falhas: um crash ou deploy da API não interrompe o processamento de eventos pendentes, e vice-versa.
- Latência previsível: polling de 2s garante folga confortável frente ao requisito de "abaixo de 10s" (`TRANSCRICAO.md [09:09]-[09:10]`).
- Implementação simples, sem dependência de infraestrutura de mensageria adicional.

**Negativas / trade-offs:**
- Latência mínima de até 2 segundos é inerente ao modelo (pior caso), diferente de um modelo reativo por push.
- **Ordering só é garantido por `order_id` e apenas em regime single-worker** — não há garantia de ordenação global entre pedidos diferentes. Escalar para múltiplos workers em paralelo quebraria essa garantia; soluções futuras (particionamento por `order_id`, lock pessimista) foram cogitadas mas não decididas, ficando registradas como limitação conhecida, não como requisito resolvido (`TRANSCRICAO.md [09:12]-[09:14] Diego/Bruno/Larissa`). Os clientes não pediram ordering global, apenas saber que cada pedido individual mudou (`TRANSCRICAO.md [09:14] Marcos`).
- Escalar o throughput de entrega exige evoluir o desenho do worker (hoje single-instance) antes de simplesmente subir mais réplicas.

## Referências

- `TRANSCRICAO.md [09:09]-[09:14]` (Diego, Bruno, Larissa, Marcos)
- `src/server.ts` (padrão de entry-point a ser espelhado por `src/worker.ts`)
- `src/config/database.ts` (padrão de instância única de `PrismaClient` por processo)
- Relacionado: ADR-001 (padrão Outbox), ADR-003 (retry e DLQ)
