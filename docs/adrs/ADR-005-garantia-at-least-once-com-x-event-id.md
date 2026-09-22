# ADR-005: Garantia de Entrega At-Least-Once com Deduplicação via `X-Event-Id`

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Diego (Eng. Sênior — Plataforma), Sofia (Eng. Segurança), Marcos (PM)
- **Tags**: confiabilidade, contrato-de-api, webhooks

## Contexto e Problema

Combinado o modelo de retry (ADR-003), é possível que uma entrega seja considerada "falha" pelo worker (ex.: timeout) mesmo que o cliente tenha efetivamente recebido e processado a requisição — nesse caso, uma nova tentativa gera entrega duplicada do mesmo evento. Era preciso decidir qual garantia de entrega o sistema oferece e como o cliente lida com duplicidade.

## Decisão

O sistema garante **at-least-once delivery**: o cliente pode receber o mesmo evento mais de uma vez, e é responsabilidade dele deduplicar. Para viabilizar isso, cada evento carrega um **`event_id` (UUID) único**, gerado no momento em que o evento entra na outbox, enviado no header `X-Event-Id` em toda tentativa (inclusive nas retentativas do mesmo evento). O cliente usa esse identificador para detectar e descartar entregas repetidas do seu lado (`TRANSCRICAO.md [09:24]-[09:25] Diego`). Essa responsabilidade será documentada de forma destacada no portal do desenvolvedor (`TRANSCRICAO.md [09:26] Marcos`).

## Alternativas Consideradas

- **Garantia exactly-once**: rejeitada por exigir coordenação transacional entre os dois lados (plataforma e cliente), o que aumenta significativamente a complexidade de implementação e operação para um ganho marginal, já que at-least-once com deduplicação por ID resolve a esmagadora maioria dos casos práticos — mesmo padrão adotado por provedores de referência como Stripe e GitHub (`TRANSCRICAO.md [09:25] Diego`).

## Consequências

**Positivas:**
- Modelo simples de implementar e operar no worker: não é necessário rastrear se uma entrega "realmente" chegou antes de decidir reenviar em caso de timeout/erro de rede.
- Alinhado a um padrão de mercado amplamente conhecido e documentável, reduzindo a curva de integração para os clientes B2B.

**Negativas / trade-offs:**
- Desloca parte da responsabilidade de corretude para o cliente: se ele não implementar a deduplicação por `X-Event-Id`, pode processar o mesmo evento de negócio mais de uma vez (`TRANSCRICAO.md [09:25] Sofia — "isso joga responsabilidade pro cliente"`).
- Não há, nesta fase, nenhum mecanismo do lado da plataforma para verificar se os clientes de fato implementaram a deduplicação corretamente — a garantia depende de comunicação e documentação, não de enforcement técnico.

## Referências

- `TRANSCRICAO.md [09:24]-[09:26]` (Diego, Sofia, Marcos)
- Relacionado: ADR-003 (retry gera as duplicatas que esta decisão endereça), ADR-004 (o `event_id` acompanha os demais headers de segurança/rastreio da entrega)
