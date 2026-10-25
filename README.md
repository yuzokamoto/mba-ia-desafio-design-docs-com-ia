# Da Reunião ao Documento: design docs do Sistema de Webhooks

[Enunciado original do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

O desafio foi transformar a reunião técnica registrada em `TRANSCRICAO.md` e o código existente do Order Management System em um pacote de design docs para o Sistema de Webhooks de Notificação de Pedidos. A entrega reúne PRD, RFC, FDD, registros de decisões arquiteturais e um tracker para apoiar o início da implementação.

A restrição central foi a rastreabilidade: cada requisito, decisão e restrição precisa ter origem identificável na transcrição ou no código. O que foi descartado ou adiado não pode virar requisito, e os detalhes necessários à implementação que não foram discutidos precisam aparecer como propostas. O trabalho foi exclusivamente documental, sem alteração do código da aplicação. Este relato usa como fonte factual o registro do revisor, `process-facts.md`, mantido na pasta local de trabalho.

## Ferramentas de IA utilizadas

- **Hermes Agent v0.22.5 (Nous Research)**, rodando o modelo **Claude Opus 5.5 (Anthropic, `claude-opus-5-5`)**. Foi o orquestrador e revisor: leu o enunciado e os materiais do curso, extraídos das páginas Notion com seu navegador, escreveu os prompts, conferiu citações e conteúdo, registrou resoluções vinculantes e escreveu o verificador determinístico. O commit ficou sob sua responsabilidade. Não escreveu o texto dos documentos.
- **Codex CLI 0.163.0 (OpenAI)**, modelo **`gpt-6.1-sol`**, reasoning effort **`high`**. Gerou todo o conteúdo entregue, do inventário ao README, e fez a revisão crítica independente. As execuções usaram `codex exec`, sem interação e com busca na web desabilitada; um watchdog limitou tokens e tempo por execução.

Os prompts usaram como referência o “Prompt de Entrevista para Gerar PRD” e seu esqueleto de feature, o “Prompt para geração de um FDD” e seu esqueleto, os exemplos de PRD e FDD de Rate Limiter e as regras MADR do agente `adr-generator`, do plugin `adrs-management`, no repositório `devfullcycle/claude-mkt-place`.

## Workflow adotado

A ordem de produção foi:

```text
Inventário da transcrição e do código (somente leitura)
  -> resoluções vinculantes do revisor
  -> ADRs -> RFC -> FDD -> PRD -> TRACKER
  -> revisão independente -> correções -> README
```

Cada execução recebeu as regras compartilhadas de `_rules.md` e o prompt específico da tarefa, `NN-*.task.md`. Cada etapa começou em uma conversa nova: a continuidade veio dos arquivos, sem reaproveitamento do contexto de conversa. Os prompts, saídas, inventário e verificador ficaram em `.handoff/`, fora do git e da entrega.

O inventário veio antes da redação para separar decisões, alternativas descartadas, itens adiados e ambiguidades. O revisor resolveu as ambiguidades e anexou as resoluções ao inventário, usado como checklist nas etapas seguintes. A prioridade das fontes foi transcrição, código, enunciado, materiais do curso e inventário revisado. A transcrição prevaleceu quando uma resolução precisou ser revista.

Os ADRs vieram primeiro porque as seis decisões principais já estavam fechadas na reunião. O RFC reuniu a abordagem, as alternativas e as questões em aberto; o FDD detalhou contratos e integração com arquivos reais. O PRD veio depois para consolidar o produto e apontar para os contratos já escritos. O TRACKER ligou os IDs dos documentos às fontes.

O Hermes escreveu um verificador determinístico, somente leitura, que conferiu seções obrigatórias, contagens mínimas, existência dos prefixos de citação e caminhos de código, links, âncoras, IDs duplicados, cobertura e proporções do TRACKER e limites dos arquivos alterados. Cada execução precisava rodá-lo e corrigir as falhas pertinentes à etapa. A revisão manual verificou se as fontes sustentavam o conteúdo; a revisão independente confrontou os documentos com transcrição, código e enunciado. O Hermes ficou responsável pela revisão final e pelo commit.

## Prompts customizados

Os prompts foram escritos em inglês para o modelo; os documentos foram produzidos em pt-BR. Os blocos abaixo são trechos literais dos arquivos locais de prompts. `[...]` marca cortes de linhas inteiras.

### Regras compartilhadas: rastreabilidade e separação dos documentos

**Etapa:** todas as execuções. **Arquivo:** `_rules.md`.

As regras fixaram a origem de cada afirmação e separaram conteúdo decidido de proposta de implementação. Também delimitaram a pergunta de cada documento para evitar repetição.

```text
[...]
Traceability (the most important rule):
- Every requirement, decision, constraint, number, endpoint, field, header, error code, risk and out-of-scope item
  must come from the transcript (cite `[hh:mm] Nome` exactly as it appears at the start of the transcript line) or
  from the code (cite the real file path). Never invent requirements, decisions, limits or numbers.
- Things the meeting explicitly discarded or postponed must appear only as out-of-scope / alternatives / open
  questions, never as requirements.
- When an implementation detail is unavoidable for the FDD to be actionable but was not discussed (for example a
  route name, a column name, an encoding), derive it from an existing code convention and cite that file, or label it
  inline as `Proposta de implementação (não discutida na reunião)` with a one-line rationale. Keep these minimal.
- Before citing `[hh:mm] Nome`, check that a line starting with exactly that prefix exists in `TRANSCRICAO.md` and that
  the cited line really says it. Several lines share the same minute; pick the right speaker.
- Before citing a code path, check that the file exists. Files that do not exist yet (for example
  `src/modules/webhooks/...` or `src/worker.ts`) may only be described as "a criar" and never cited as an existing
  source.
- The API is mounted under `/api/v1` (`src/app.ts`, `src/routes/index.ts`); transcript paths like `/webhooks/...`
  or `/admin/webhooks/...` become `/api/v1/webhooks/...` and `/api/v1/admin/webhooks/...`.

Language and style of artifacts (overrides any other language preference):
- All document prose is Brazilian Portuguese (pt-BR) with correct accents. Code, identifiers, file paths, table and
  column names, JSON keys, HTTP headers, enum values and error codes stay exactly as in English/code.
- Do not use the em dash character (U+2014). Use commas, colons, parentheses or a plain hyphen instead.
- Each document stays at its own altitude and does not repeat the others: PRD = why and what (product); RFC = the
  proposed approach, alternatives and open questions (architecture, concise); ADR = one closed decision with context
  and consequences; FDD = how to build it (flows, contracts, errors, integration with the code); TRACKER = where each
  item came from. Link to the other document instead of repeating its content.
- Give every identifiable item a stable ID inside its document (for example `PRD-FR-01`, `PRD-NFR-03`, `RFC-ALT-02`,
  `RFC-OQ-01`, `FDD-API-04`, `FDD-ERR-07`, `FDD-INT-02`, `ADR-003`). The tracker maps these IDs to their sources.
```

### Inventário dirigido antes da redação

**Etapa:** inventário, execução 01. **Arquivo:** `01-inventory.task.md`.

O pedido exigiu classificação das falas e análise das ambiguidades, em vez de um resumo genérico. A checagem aritmética procurou incompatibilidades entre quantidades, intervalos e totais antes de redigir os documentos.

```text
[...]

Goal: before any document is written, build the evidence base. Your final message IS the deliverable (it is saved
to a file by the runner). Do not write files. Read `TRANSCRICAO.md` in full, line by line, and the code listed in the
shared rules. Do not summarise the meeting generically: classify every statement that carries information.
[...]
Points where the transcript is ambiguous, internally inconsistent, or silent about something the implementation must
decide. For each: what the transcript says (with sources), why it is ambiguous, 2 or 3 possible readings, and the
reading you recommend with the argument from the transcript or the code. Check numbers arithmetically (counts,
intervals, totals) against each other. Check every decision against the code it touches (transactions, roles, routes,
error handling, ids, logging).
[...]
```

### PRD: adaptar a entrevista à transcrição

**Etapa:** PRD, execução 05. **Arquivo:** `05-prd.task.md`. **Base do curso:** “Prompt de Entrevista para Gerar PRD” e esqueleto de PRD de feature.

A entrevista virou leitura da transcrição, com “Checagens de Consistência” obrigatórias e “Defaults Inteligentes” proibidos. Isso procurou impedir números sem fonte, mas o próprio prompt ainda introduziu uma meta indevida, corrigida no ciclo descrito adiante.

```text
[...]

Read first: `.handoff/analysis/inventory.md` (section 7 holds binding resolutions), `docs/RFC.md`, `docs/FDD.md`
and `docs/adrs/` (already reviewed; do not edit them), `TRANSCRICAO.md`, the course PRD interview prompt and its
output skeleton `.handoff/refs/course-prd-prompt.md` (its "Checagens de Consistência" are mandatory for you; its
"Defaults Inteligentes" must NOT be used: every number must come from the transcript), the course example
`.handoff/refs/course-prd-example.md`, and the PRD requirements and acceptance criteria in
`.handoff/refs/challenge-statement.md` ("1. PRD da feature").

Write only `docs/PRD.md`, in pt-BR. The PRD is the product/business altitude: why and what, for whom, how we know it
worked. It is written last, so it consolidates: link the RFC, ADRs and FDD instead of repeating their technical
detail (no schemas, no payload JSON, no header lists, no worker algorithm, no error matrix).
[...]
Before finishing, run the course checklist ("Checagens de Consistência antes de finalizar") against your text and fix
any failure. When done, reply with the list of IDs and any uncertainty.
[...]
```

### FDD: tornar a implementação acionável

**Etapa:** FDD, execução 04. **Arquivo:** `04-fdd.task.md`. **Base do curso:** “Prompt para geração de um FDD”, seu esqueleto e suas “Regras para Coleta”.

O prompt pediu premissas explícitas, exemplos mínimos e semântica dos contratos, além da leitura dos arquivos citados. A separação entre documentos manteve a narrativa de produto no PRD e o debate de alternativas no RFC e nos ADRs.

```text
[...]

Read first: `.handoff/analysis/inventory.md` (section 7 holds binding resolutions), the 7 ADRs in `docs/adrs/` and
`docs/RFC.md` (already reviewed; do not edit them), `TRANSCRICAO.md`, every code file you are going to cite, the
course FDD prompt and skeleton `.handoff/refs/course-fdd-prompt.md` (follow its structure and its "Regras para Coleta":
explicit assumptions, configurable parameters with defaults, minimal examples per contract, semantics of fields and
headers, metrics/logs/tracing) and the FDD requirements and acceptance criteria in
`.handoff/refs/challenge-statement.md` ("3. FDD da feature").

Write only `docs/FDD.md`, in pt-BR. It is the deepest document: a developer must be able to start coding from it.
It does not repeat the business narrative (PRD) or the alternatives debate (RFC/ADRs): link them.
[...]
```

### Correção dos ADRs após revisão

**Etapa:** correção dos ADRs, primeira parte da execução 03. **Arquivo:** `03a-adr-fixes.task.md`.

A correção delimitou os pontos a alterar e indicou a fala que sustentava cada ajuste. No exemplo de segurança, preservou tanto a intenção do header quanto a limitação da assinatura, sem transformar uma possível melhoria em decisão da reunião.

```text
[...]

The 7 ADRs in `docs/adrs/` were reviewed. Apply exactly these corrections, keeping everything else as is:

1. Readability: remove the `ADR-00N-CTX-NN:` and `ADR-00N-FAC-NN:` prefixes everywhere (keep the sentences). Keep
   the IDs `ADR-00N-DEC-NN`, `ADR-00N-PROP-NN`, `ADR-00N-ALT-NN` and `ADR-00N-CONS-NN`; they are the traceable items.
   Make sure no remaining text refers to a removed ID.
2. ADR-004: `[09:44] Diego` says `X-Timestamp` carries the send time "pra cliente conseguir detectar replay attack se
   quiser". Rewrite ADR-004-CONS-04 so it first states that intent with that citation, then the limitation: the
   signature decided in `[09:22] Sofia` covers only the body, so the timestamp is not authenticated and the replay
   detection it allows is limited; evaluating it (for example, including the timestamp in the signed content) belongs
   to Sofia's review (`[09:46] Sofia`), not decided here. Do not present inclusion in the signature as decided.
[...]
```

## Iterações e ajustes

### Inventário e resoluções do revisor (execução 01)

O Codex produziu 137 itens da transcrição, 34 ambiguidades e 45 fatos do código. O Hermes conferiu por script as 516 citações e a existência dos caminhos citados. O inventário expôs conflitos, mas ainda não os resolveu para orientar a redação: “5 tentativas”, “retentaria três vezes em 30 minutos” ([09:16] Diego) e “quase 15 horas” ([09:17] Diego) exigiam interpretação; o `customer_id` sugerido a partir do JWT ([09:31] Marcos) foi corrigido na própria reunião ([09:32] Larissa).

O Hermes anexou 34 resoluções vinculantes ao inventário. Inicialmente, interpretou a contagem como envio inicial mais cinco retentativas, pelo vetor de intervalos; detalhes não discutidos deveriam receber o rótulo de proposta. As resoluções orientaram as etapas seguintes, mas algumas precisaram ser corrigidas na revisão independente.

### ADRs e correção dirigida (execuções 02 e 03)

A execução 02 entregou 7 ADRs e índice, sem falhas próprias no verificador. A revisão manual encontrou IDs de contexto em todo parágrafo, prejudicando a leitura; omissão da finalidade do `X-Timestamp`, indicada em [09:44] Diego, e da limitação de assinar apenas o corpo; ausência de [09:10] Larissa no status do ADR-002; prós vazios como “conforme a alternativa descrita por ...”; e referências longas no ADR-006.

O prompt `03a-adr-fixes.task.md`, aplicado antes do RFC na execução 03, removeu os prefixos de contexto, corrigiu as referências, explicitou a limitação de segurança, substituiu os prós vazios por avaliações rotuladas e condensou as referências. O revisor conferiu as 5 correções.

### Ajuste do processo para preservar links

Os links entre documentos apontavam para seções, como `FDD.md#integração-com-o-sistema-existente`. Títulos numerados mudariam essas âncoras. Os prompts do RFC e do FDD passaram a exigir títulos sem numeração, e o verificador passou a validar as âncoras no padrão do GitHub. O ajuste incorporou a verificação dos destinos ao processo.

### RFC e ajustes de forma (execuções 03 e 06)

O RFC inicial teve 1.703 palavras, 6 alternativas descartadas, 6 questões em aberto, 5 riscos e links para os 7 ADRs. O conteúdo foi considerado fiel, mas o título precisava de ajuste e o diagrama Mermaid não mostrava histórico, DLQ e replay. Esses pontos foram corrigidos na execução 06.

### FDD e conferência do código (execução 04)

O FDD teve cerca de 13,6 mil palavras. O Hermes conferiu cerca de 35 afirmações contra o código, incluindo mensagens e classes de erro, envelope paginado, resposta 204 na exclusão, `reserveOrderNumber`, tratamento de P2002/P2025, shutdown de `src/server.ts`, scripts e versões de `package.json`, ordem do setup dos testes e `redactPaths` do logger. Todas estavam corretas; o documento foi aceito sem correção nessa etapa. A revisão independente posterior encontrou problemas de comportamento no pacote, descritos abaixo.

### PRD e retirada de uma meta sem sustentação (execuções 05 e 06)

O PRD inicial reuniu 12 requisitos funcionais, 3 objetivos com meta, 10 itens fora de escopo e 7 riscos. O PRD-OBJ-02 afirmava “os 3 clientes integrados até o fim de novembro”, embora ninguém tivesse fixado essa meta. Parte do erro veio da redação do próprio prompt.

Na primeira parte da execução 06, o revisor ditou a substituição com base em [09:00] Marcos, [09:45] Marcos e [09:47] Marcos: os pedidos dos clientes, o prazo solicitado pela Atlas e a confirmação ainda a fazer não deveriam ser fundidos em uma meta de integração de todos. A linha foi corrigida.

### TRACKER e amostragem das fontes (execução 06)

O prompt exigiu levantar os IDs por script e escrever o resumo a partir do significado de cada item. A entrega inicial teve 376 linhas, cobrindo 100% dos IDs: 336 com fonte TRANSCRICAO (89,4%) e 40 com fonte CODIGO. O Hermes conferiu uma amostra de 28 linhas contra a transcrição, todas sustentadas pelas fontes.

O verificador registrou 129 checagens, com só as 6 do README pendentes, conforme o registro dessa etapa. A revisão independente ainda encontraria localizações incompletas no TRACKER, corrigidas depois.

### Revisão independente e novas resoluções (execução 08)

Uma execução nova do Codex, somente leitura, recebeu a instrução de ser cética e não pôde ler inventário nem prompts. Confrontou apenas documentos, transcrição, código e enunciado. Encontrou 9 problemas: 5 de severidade alta, incluindo o README ainda não escrito, 3 média e 1 baixa. Marcou como falho o critério de ausência de contradições com a transcrição.

Os principais achados, conferidos pelo Hermes, foram:

- Uma segunda rotação de secret encurtava a janela de 24 h prometida em [09:21] Sofia. A regra vinha de uma resolução do próprio revisor.
- A regra de que retry pendente não bloquearia eventos posteriores, também proposta pelo revisor, contradizia a ordenação por `order_id` com worker único de [09:12] Diego e [09:13] Larissa.
- “Envio inicial + 5 retentativas” aparecia como decisão, apesar de [09:48] Larissa dizer “total 5 tentativas”. A interpretação fechava com o vetor e com “quase 15 horas”, mas precisava de rótulo e confirmação.
- A meta de 10 s fora redefinida como tempo do commit até o início do envio, sem rótulo de proposta.
- Esperar 2 s após um lote sequencial de 10 envios podia resultar em cerca de 100 s entre consultas.
- O PRD tinha detalhes técnicos em excesso e o TRACKER tinha localizações incompletas.

O Hermes escreveu 9 resoluções vinculantes, R1 a R9. Aceitou parcialmente duas sugestões: preservou os fluxos por requisito do esqueleto de PRD do curso e a seção de decisões exigida pelo enunciado; manteve UUID no ADR-006, porque [09:51] Larissa o tratou como reuso do padrão existente, sem criar um ADR-008.

### Correções do pacote após a revisão (execução 09)

A execução 09 aplicou as resoluções aos documentos e rotulou as novas regras como propostas de implementação:

- Recusar nova rotação com 409 e `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` enquanto a secret anterior estiver na janela de 24 h.
- Usar fila FIFO por par (`webhookId`, `orderId`): um evento em retry segura os seguintes do mesmo pedido para o mesmo endpoint até entrega ou DLQ.
- Contar o ciclo do worker a cada 2 s desde o início do ciclo anterior, com envios do lote em paralelo e no máximo um por par.
- Manter “inicial + 5 retentativas” como interpretação a confirmar, registrada na nova questão RFC-OQ-07.
- Restabelecer a meta de 10 s como chegada da notificação ao cliente e rotular o indicador interno.

O PRD passou de cerca de 5.630 para 4.154 palavras sem perder IDs, e o TRACKER foi atualizado para 378 itens, com cobertura de 100%. O Hermes conferiu as regras em ADRs, RFC, FDD e PRD e confirmou que nenhum arquivo de código mudou. O registro aponta 123 checagens dos documentos OK, com só as 6 do README pendentes.

### README por último (execução 07)

O README foi gerado por último a partir do registro factual do revisor. O prompt recebeu número anterior porque havia sido escrito antes, mas a execução ocorreu após as correções do pacote.

Foram **9 execuções do Codex no total**: 6 de geração (01 inventário, 02 ADRs, 03 RFC, 04 FDD, 05 PRD, 06 TRACKER), 1 de revisão independente (08), 1 de correção dedicada (09) e 1 do README (07).

Foram **3 iterações principais de correção sobre documentos já gerados**: ADRs (primeira parte da execução 03), PRD e RFC (primeira parte da execução 06) e o pacote inteiro depois da revisão independente (execução 09). Antes delas, as 34 resoluções do ciclo 1 corrigiram o rumo antes de qualquer documento ser escrito.

## Como navegar a entrega

```text
.
├── README.md                  # processo de produção
├── TRANSCRICAO.md             # fonte primária, preservada
└── docs/
    ├── PRD.md                 # produto e negócio
    ├── RFC.md                 # proposta de arquitetura
    ├── FDD.md                 # especificação de implementação
    ├── TRACKER.md             # rastreabilidade
    └── adrs/
        ├── README.md         # índice das decisões
        ├── ADR-001-outbox-transacional-no-mysql.md
        ├── ADR-002-worker-separado-com-polling.md
        ├── ADR-003-retry-com-backoff-e-dead-letter-queue.md
        ├── ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-entrega-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
        └── ADR-007-snapshot-do-payload-na-insercao.md
```

A ordem sugerida de leitura é:

- [README](README.md): como o pacote foi produzido, revisado e corrigido?
- [PRD](docs/PRD.md): por que construir a feature, para quem, com qual escopo e quais critérios de sucesso?
- [RFC](docs/RFC.md): qual abordagem arquitetural está em revisão, quais alternativas foram descartadas e o que segue em aberto?
- [ADRs](docs/adrs/README.md): por que cada decisão arquitetural foi tomada e quais consequências traz?
- [FDD](docs/FDD.md): como implementar os fluxos, contratos, erros e integração com o sistema existente?
- [TRACKER](docs/TRACKER.md): de onde veio cada item dos documentos?
- [TRANSCRICAO](TRANSCRICAO.md): o que os participantes efetivamente disseram? Consulte-a ao conferir as fontes do TRACKER.

O verificador determinístico foi usado durante a produção. Ele permaneceu em `.handoff/`, fora do git e do pacote entregue; por isso não há instrução para executá-lo a partir desta entrega.
