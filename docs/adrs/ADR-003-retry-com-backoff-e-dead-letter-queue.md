# ADR-003: Retry com Backoff Exponencial e Dead Letter Queue em Tabela Separada

- **Status**: Aceito
- **Data**: 2026-09-21 (registro); decisão tomada na reunião técnica descrita em `TRANSCRICAO.md`
- **Decisores**: Diego (Eng. Sênior — Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno — Pedidos), Sofia (Eng. Segurança)
- **Tags**: confiabilidade, resiliência, webhooks

## Contexto e Problema

Endpoints de clientes B2B podem ficar temporariamente indisponíveis (já houve caso de indisponibilidade de ~2h por manutenção planejada de um cliente — `TRANSCRICAO.md [09:16] Diego`). Era preciso decidir por quanto tempo e com que estratégia o sistema insiste em entregar um evento antes de desistir, e o que fazer com um evento que esgotou as tentativas.

## Decisão

Adotamos **retry com backoff exponencial de 5 tentativas**, com intervalos `1min → 5min → 30min → 2h → 12h` (total de ~15h entre a primeira falha e a última tentativa). Esgotadas as tentativas, o evento é movido para uma **tabela separada `webhook_dead_letter`** (Dead Letter Queue), guardando payload, motivo da falha e timestamp. O reprocessamento é **manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente, exige role `ADMIN` (reaproveitando `requireRole`, ver ADR-006) e deve registrar em log quem executou o replay, para auditoria (`TRANSCRICAO.md [09:15]-[09:19], [09:35]-[09:36]`).

## Alternativas Consideradas

- **Retry indefinido**: rejeitada porque deixaria eventos "pendurados" para sempre caso o cliente tenha saído de operação ou trocado de endpoint permanentemente (`TRANSCRICAO.md [09:15] Diego`).
- **3 tentativas** (mais agressivo): rejeitada por matar o evento cedo demais em cenários reais de indisponibilidade prolongada de cliente já observados (`TRANSCRICAO.md [09:16] Bruno/Diego`).
- **Marcar como `failed` na própria tabela outbox**, em vez de tabela separada: rejeitada por misturar estados operacionais (pendente/processando) com estado terminal de falha na mesma tabela, dificultando a leitura da fila ativa e o processo de debug/reprocessamento (`TRANSCRICAO.md [09:18] Diego/Bruno`).

## Consequências

**Positivas:**
- Janela de ~15h de tentativas cobre a maioria dos cenários reais de indisponibilidade temporária de cliente sem exigir intervenção manual.
- DLQ em tabela própria mantém a outbox operacional "limpa" (só eventos ativos) e preserva evidência completa (payload + motivo) para investigação.
- Replay manual restrito a `ADMIN` com auditoria evita reprocessamento acidental ou não rastreável.

**Negativas / trade-offs:**
- Um evento pode demorar até ~15h para ser definitivamente considerado falho, período em que o cliente pode operar com dado desatualizado sem qualquer alerta automático (notificação por e-mail em caso de falha repetida foi explicitamente descartada nesta fase — ver `docs/RFC.md`, seção de fora de escopo).
- Reprocessamento manual via endpoint admin é um processo operacional adicional — não há reprocessamento automático de itens em DLQ.

## Referências

- `TRANSCRICAO.md [09:14]-[09:19]` (Diego, Bruno, Larissa) e `[09:35]-[09:36]` (Larissa, Sofia)
- `src/middlewares/auth.middleware.ts` (`requireRole`, reaproveitado no endpoint de replay)
- Relacionado: ADR-001 (padrão Outbox), ADR-006 (reuso de padrões existentes)
