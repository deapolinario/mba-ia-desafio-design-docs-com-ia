# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Sofia (Eng. Segurança), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)
- **Tags**: segurança, webhooks

## Contexto e Problema

O sistema passa a expor dados de pedidos para endpoints HTTP fora da infraestrutura da empresa, controlados pelos clientes B2B. Era necessário garantir que (a) o cliente consiga verificar que a requisição realmente veio da plataforma e (b) o payload não foi adulterado em trânsito, sem depender apenas de TLS, que autentica o canal mas não a origem da aplicação (`TRANSCRICAO.md [09:19] Sofia`).

## Decisão

Cada requisição de webhook é **assinada com HMAC-SHA256** sobre o corpo (body) da requisição, e a assinatura vai no header `X-Signature`. Cada endpoint de webhook cadastrado tem uma **secret própria, não uma secret global da plataforma** — a tabela de configuração armazena `url + secret + customer_id + estado ativo`. A secret é **rotacionável via API**: ao rotacionar, a secret antiga permanece válida em paralelo por **24 horas (grace period)**, para o cliente migrar sem downtime, e depois disso é invalidada (`TRANSCRICAO.md [09:20]-[09:22] Sofia`). Complementarmente (tratadas como requisitos não funcionais, não decisões arquiteturais à parte): TLS obrigatório na URL cadastrada (recusa `http` na validação Zod) e limite de 64KB de payload, com erro explícito se ultrapassado (`TRANSCRICAO.md [09:23]-[09:24] Sofia/Diego/Larissa`).

## Alternativas Consideradas

- **Secret única/global da plataforma para todos os endpoints**: rejeitada porque o vazamento de uma única secret comprometeria a autenticidade de todos os webhooks de todos os clientes; há precedente concreto de cliente que já vazou uma secret em log de aplicação (`TRANSCRICAO.md [09:21]-[09:22] Sofia/Diego`).
- **Depender apenas de TLS, sem assinatura de payload**: implicitamente descartada — TLS garante confidencialidade/integridade em trânsito, mas não prova ao cliente que o remetente é de fato a plataforma nem protege contra adulteração antes do envio; por isso a assinatura HMAC foi tratada como requisito adicional, não substituível pelo TLS (`TRANSCRICAO.md [09:19]-[09:20] Sofia`).

## Consequências

**Positivas:**
- Cliente pode verificar autenticidade e integridade do payload de forma independente do transporte, usando bibliotecas padrão de mercado (HMAC-SHA256 é amplamente suportado).
- Isolamento de blast radius: vazamento de uma secret afeta apenas um endpoint/cliente, não a base inteira.
- Rotação com grace period de 24h permite operação segura sem downtime de integração para o cliente.

**Negativas / trade-offs:**
- Introduz gestão de segredo por endpoint (geração, armazenamento seguro, rotação, expiração da versão antiga) — superfície de configuração e de possível erro operacional maior do que uma secret única.
- A responsabilidade de verificar a assinatura corretamente do lado do cliente não pode ser garantida pela plataforma; falha de implementação no lado do cliente permanece um risco fora do controle direto da equipe.

## Referências

- `TRANSCRICAO.md [09:19]-[09:24]` (Sofia, Diego, Bruno, Larissa)
- Relacionado: ADR-005 (at-least-once e `X-Event-Id`), ADR-006 (reuso do padrão de erro `WEBHOOK_*` para falhas de validação de URL/secret)
