# Da Reunião ao Documento: Design Docs Gerados por IA

> O enunciado original do desafio foi preservado em [`README_ENUNCIADO.md`](./README_ENUNCIADO.md). Este README documenta o processo de produção do pacote de documentação.

## Sobre o Desafio

O desafio consiste em transformar a transcrição literal de uma reunião técnica (`TRANSCRICAO.md`) e o código-fonte de um Order Management System já em produção em um pacote completo de design docs para uma feature nova — um Sistema de Webhooks de Notificação de Pedidos. A decisão técnica já havia sido fechada entre tech lead, PM, dois engenheiros e uma engenheira de segurança durante a call; nada disso estava registrado além da gravação em texto.

O trabalho não era "resumir a transcrição": era decidir o que de fato virava requisito, decisão ou restrição documentada, o que ficava explicitamente fora de escopo, e em qual dos cinco documentos (PRD, RFC, ADRs, FDD, Tracker) cada informação deveria morar — sem duplicar conteúdo entre eles e sem inventar nada que não fosse rastreável à reunião ou ao código.

## Ferramentas de IA Utilizadas

- **Claude Code (Sonnet 5)** — ferramenta principal de produção. Usado interativamente durante toda a sessão: leitura e análise do código-fonte, leitura dirigida da transcrição, extração estruturada de decisões/requisitos, e geração de todos os documentos do pacote (ADRs, RFC, FDD, PRD, Tracker).
- **Skills do marketplace `@tech-leads-club/agent-skills`** (`create-adr`, `create-rfc`, `fdd-creator`, `prd-writer`), instaladas via `npx @tech-leads-club/agent-skills install --skill <nome>` — usadas como ponto de partida estrutural para cada documento, mas **adaptadas manualmente** em cada caso para satisfazer os requisitos específicos deste desafio (formato de arquivo, seções obrigatórias, idioma), já que os templates genéricos dessas skills não foram desenhados para este contexto (ver "Iterações e Ajustes" abaixo).

## Workflow Adotado

Segui a ordem de execução sugerida no enunciado original (seção "Ordem de execução sugerida" de `README_ENUNCIADO.md`), com duas etapas de preparação antes dos documentos formais:

1. **Contextualização com IA**: pedi uma análise do estado atual do código, registrada em [`aux_docs/analise-codigo-atual.md`](./aux_docs/analise-codigo-atual.md) — arquitetura, módulos, máquina de estados de pedido, padrões de erro, autenticação, e uma lista de inconsistências reais encontradas no código (não relacionadas à feature nova).
2. **Mapeamento da transcrição**: antes de escrever qualquer documento formal, pedi um mapeamento estruturado de `TRANSCRICAO.md` em [`aux_docs/mapeamento-transcricao.md`](./aux_docs/mapeamento-transcricao.md) — as 6 decisões arquiteturais obrigatórias, decisões secundárias, requisitos funcionais e não funcionais numerados, itens fora de escopo, questões em aberto e pontos de integração com o código, cada um já com timestamp e falante. Esse documento se tornou a fonte de verdade reaproveitada em todos os documentos seguintes, evitando redescobrir (ou reinterpretar de forma inconsistente) a transcrição a cada etapa.
3. **ADRs primeiro** (`docs/adrs/ADR-001` a `ADR-007`), usando a skill `/create-adr` como estrutura base (formato MADR: Status/Contexto/Decisão/Alternativas/Consequências), mas gerando um arquivo por decisão do mapeamento em vez de aguardar instrução manual item a item.
4. **RFC** (`docs/RFC.md`), consolidando os 7 ADRs em uma proposta única, com alternativas descartadas e questões em aberto extraídas do mapeamento, linkando cada elemento da proposta ao ADR correspondente.
5. **FDD** (`docs/FDD.md`), detalhando fluxos, contratos HTTP, matriz de erros `WEBHOOK_*` e a seção obrigatória "Integração com o sistema existente" — usando a skill `/fdd-creator` como esqueleto, mas gerado diretamente (sem entrevista passo a passo) porque o mapeamento e a análise de código já continham todas as respostas necessárias.
6. **PRD** (`docs/PRD.md`), por último entre os documentos grandes — na prática, uma consolidação em linguagem de produto do que já estava decidido em RFC/FDD/ADRs.
7. **Tracker** (`docs/TRACKER.md`), construído por varredura sistemática dos quatro documentos anteriores, linha a linha, e depois **verificado programaticamente** (script Python rodado via terminal) para confirmar as taxas de cobertura exigidas pelo enunciado.
8. **Este README**, escrito logo após o Tracker, documentando o processo até aquele ponto.
9. **Revisão final**: antes de commitar, pedi duas passadas de auditoria — (a) uma checagem de consistência cruzada entre `TRANSCRICAO.md`, o código-fonte e os 11 documentos gerados (existência de todo caminho de arquivo citado, consistência numérica de parâmetros como retry/timeout/limites entre documentos, resolução de todos os links internos) e (b) a checklist completa de critérios de aceite do `README_ENUNCIADO.md`, item por item. A auditoria encontrou 3 inconsistências reais, corrigidas antes do commit (ver "Iterações e Ajustes", item 5). Este README foi atualizado por último, após a revisão, para registrar essa etapa.

## Prompts Customizados

Dois exemplos representativos de prompts que escrevi/adaptei durante o processo — o segundo ilustra o padrão que repeti para RFC, FDD e PRD (usar a skill nomeada, mas adaptá-la ao requisito exato do desafio):

```text
Eu preciso inicialmente de fazer uma análise do código entendendo o estado do
código atualmente pensando que vou montar um plano de melhoria desse código
posteriormente. Me ajude a fazer essa análise de funcionamento atual do
código. escreva a analise dentro da pasta aux_docs
```

```text
utilize a skill /create-adr e adapte a geração das ADRs para satisfazer o
item 4 do arquivo @README.md. Utilize as decisões conversadas na reunião
entre Larissa (Tech Lead, facilitadora), Marcos (PM), Bruno (Eng. Pleno,
time Pedidos), Diego (Eng. Sênior, time Plataforma), Sofia (Eng. Segurança).
```

O segundo padrão de prompt ("use a skill X, mas adapte para satisfazer o item Y do enunciado") foi reaplicado nas três chamadas seguintes (`/create-rfc`, `/fdd-creator`, `/prd-writer`), porque nenhuma das quatro skills genéricas foi desenhada para os requisitos exatos deste desafio — cada uma tinha um formato de saída próprio que precisava ser confrontado com a lista de seções obrigatórias do `README_ENUNCIADO.md`.

## Iterações e Ajustes

O processo não foi "gerar uma vez e aceitar" em nenhum dos documentos maiores. Os ajustes mais relevantes:

1. **PRD**: a skill `/prd-writer` é desenhada para produtos greenfield multi-feature — gera em inglês, com sistema de IDs `F01`-`F99`, grafo de dependência entre features, "execution waves" e um arquivo JSON de progresso para automação de implementação. Nada disso se aplicava a uma feature única de um sistema já em produção documentada em português. Descartei inteiramente esse aparato e reconstruí o PRD em torno das 12 seções fixas exigidas pelo enunciado (Resumo, Problema, Público-alvo, Objetivos, Escopo, RFs, NFRs, Decisões, Dependências, Riscos, Critérios de aceite, Estratégia de testes).
2. **RFC**: o template padrão da skill `/create-rfc` assume uma decisão *ainda não tomada* — critérios de decisão com peso, RACI, comparação de opções, seção "Outcome" em branco para preencher depois. Neste desafio as decisões já estavam fechadas na reunião; o RFC precisava simular o documento "submetido para revisão" descrito no enunciado, não um processo de decisão em aberto. Troquei a estrutura inteira pela lista de seções exigida no `README_ENUNCIADO.md` (Alternativas consideradas, Questões em aberto, Decisões relacionadas linkando os ADRs).
3. **Tracker — falha de formato detectada por verificação programática**: na primeira versão do `docs/TRACKER.md`, cerca de 47 das ~135 linhas com `Fonte = TRANSCRICAO` traziam apenas o timestamp (ex.: `[09:12]-[09:14]`) sem o nome do falante, por eu ter comprimido intervalos de tempo que cobriam falas de várias pessoas. Rodei um script Python para checar programaticamente o formato `[hh:mm] Nome` exigido pelo critério de aceite (mínimo de 70% das linhas) e descobri que a taxa real de conformidade estava em ~54%, abaixo do exigido. Corrigi sistematicamente as 47 linhas (adicionando o(s) falante(s) principal(is) de cada trecho) e reverifiquei programaticamente até confirmar 82,3% de conformidade.
4. **FDD — separação entre decisão citada e elaboração técnica**: ao montar a matriz de erros `WEBHOOK_*`, apenas 3 códigos foram citados literalmente na reunião (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). Em vez de apresentar os demais códigos necessários (ex.: `WEBHOOK_PAYLOAD_TOO_LARGE`) como se também tivessem sido decididos na call, marquei-os explicitamente como elaboração técnica no próprio corpo do FDD, para não violar a regra do desafio de não registrar informação sem origem identificável.
5. **Revisão final — 3 inconsistências reais encontradas e corrigidas**: a auditoria de consistência (passo 9 do workflow) pegou erros que a geração inicial não tinha detectado sozinha. (a) O FDD afirmava uma meta de latência de "~12s", contradizendo o PRD/RFC ("≤10s") e, mais grave, contradizendo a própria transcrição (`[09:10] Larissa: "a latência mínima vai ser 2 segundos no pior caso"`) — o número veio de um raciocínio meu que somou erroneamente o timeout de 10s ao polling de 2s; corrigido para ≤10s em FDD e Tracker. (b) O contrato `PATCH /api/v1/webhooks/:id` no FDD tinha exemplo de requisição mas não de resposta, faltando ao critério de aceite que exige ambos; adicionei o exemplo de resposta. (c) "Decisões Relacionadas", uma das 8 seções obrigatórias do RFC, estava apenas como uma linha da tabela de metadados em vez de seção própria; promovi para uma seção dedicada linkando os 7 ADRs.

## Como Navegar a Entrega

Ordem sugerida de leitura, da fonte primária até a documentação final:

1. [`TRANSCRICAO.md`](./TRANSCRICAO.md) — fonte primária, gravação da reunião técnica.
2. [`aux_docs/analise-codigo-atual.md`](./aux_docs/analise-codigo-atual.md) — mapeamento do código existente (arquitetura, módulos, padrões).
3. [`aux_docs/mapeamento-transcricao.md`](./aux_docs/mapeamento-transcricao.md) — extração estruturada da reunião (decisões, requisitos, escopo, questões em aberto).
4. [`docs/adrs/`](./docs/adrs/) — 7 ADRs, na ordem numérica (`ADR-001` a `ADR-007`), cada decisão isolada com contexto e consequências.
5. [`docs/RFC.md`](./docs/RFC.md) — proposta técnica consolidada, linkando os ADRs.
6. [`docs/FDD.md`](./docs/FDD.md) — detalhamento de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código existente.
7. [`docs/PRD.md`](./docs/PRD.md) — consolidação em nível de produto/negócio.
8. [`docs/TRACKER.md`](./docs/TRACKER.md) — rastreabilidade cruzada de cada item aos documentos acima.
9. [`README_ENUNCIADO.md`](./README_ENUNCIADO.md) — enunciado original do desafio, para referência.
