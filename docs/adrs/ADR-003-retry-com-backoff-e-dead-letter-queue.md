# ADR-003: Retentativas com backoff e DLQ

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:17] Larissa`; confirmado em `[09:48] Larissa`.
Relacionados: [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião.

## Contexto

Clientes podem ficar indisponíveis por períodos relevantes; já houve manutenção planejada de duas horas. O processamento precisa absorver falhas temporárias e ter um término operacional definido (`[09:14] Larissa`, `[09:15] Diego`, `[09:16] Diego`).

Depois do esgotamento, o evento precisa continuar disponível para diagnóstico e reprocessamento manual. A reunião escolheu uma DLQ em tabela separada, preservando payload, motivo e timestamp, com replay administrativo e auditoria (`[09:18] Diego`, `[09:19] Larissa`, `[09:36] Sofia`).

## Fatores de decisão

- Indisponibilidade de duas horas já ocorreu e precisa informar a janela de recuperação (`[09:16] Diego`).
- Eventos de clientes que desapareceram precisam chegar a um estado de falha permanente (`[09:15] Diego`).
- A progressão fechada foi `1m/5m/30m/2h/12h` (`[09:17] Diego`, `[09:17] Larissa`).
- A tabela separada deve manter limpa a leitura principal e preservar evidências (`[09:18] Diego`).
- Replay exige role `ADMIN` e registro de quem o executou (`[09:36] Sofia`, `[09:36] Larissa`).

## Alternativas Consideradas

### Opção 1: Retentativas limitadas e DLQ separada (escolhida)

Usar a progressão fechada e preservar falhas esgotadas em `webhook_dead_letter`, a criar (`[09:17] Larissa`, `[09:18] Diego`).

- Prós: Cobre uma janela maior de indisponibilidade e mantém evidência para replay (`[09:16] Diego`, `[09:17] Diego`, `[09:18] Diego`).
- Contras: Após esgotar a política, a recuperação depende de intervenção administrativa (`[09:18] Diego`, `[09:36] Sofia`).

### Opção 2: Retry indefinido (ADR-003-ALT-01)

Descartado em `[09:15] Diego`: evitar a falha permanente manteria eventos pendurados quando o cliente desaparecesse.

- Prós (avaliação do autor, não discutida na reunião): Nenhum evento é abandonado enquanto o cliente está indisponível.
- Contras: Não encerra eventos de clientes que não voltam (`[09:15] Diego`).

### Opção 3: Apenas três retentativas (ADR-003-ALT-02)

Levantada em `[09:16] Bruno` e rejeitada em `[09:16] Diego` como janela insuficiente para a manutenção já observada.

- Prós (avaliação do autor, não discutida na reunião): Permite detectar a falha permanente mais cedo.
- Contras: Esgotaria cedo diante de indisponibilidade de duas horas (`[09:16] Diego`).

## Decisão

ADR-003-DEC-01: Registrar literalmente a decisão: “5 tentativas, backoff 1m/5m/30m/2h/12h” ([09:17] Larissa), confirmada como “total 5 tentativas” ([09:48] Larissa). Preservar falhas esgotadas na tabela separada `webhook_dead_letter`, a criar, com payload, motivo e timestamp ([09:18] Diego). “Backoff exponencial” denomina o vetor fechado, sem fórmula adicional ([09:15] Diego, [09:17] Diego).

ADR-003-DEC-02: Há inconsistência: cinco intervalos implicam seis envios, enquanto Larissa fecha cinco tentativas. Diego fala em “três vezes em 30 minutos” ([09:16] Diego) e “quase 15 horas entre primeira falha e última tentativa” ([09:17] Diego), sustentando cinco retentativas após o envio inicial. **Proposta de implementação (não discutida na reunião):** adotar `1 inicial + 5 retentativas`, até seis envios HTTP por linha e endpoint, a confirmar na revisão com Bruno e Diego ([09:50] Larissa; [RFC-OQ-07](../RFC.md#rfc-oq-07-contagem-de-tentativas-a-confirmar)). Cinco envios no total descartariam o intervalo de 12 h e contradiriam o vetor decidido. A soma das esperas é `1 + 5 + 30 + 120 + 720 = 876 minutos = 14h36`; os três primeiros intervalos somam 36 minutos, aproximados verbalmente como 30. Chamadas, polling e backlog ampliam o tempo observado. Replay manual por `POST /api/v1/admin/webhooks/dead-letter/:id/replay` recoloca o evento pendente e exige `ADMIN` com auditoria ([09:18] Diego, [09:36] Sofia, [09:36] Larissa; prefixo em `src/app.ts`, `src/routes/index.ts`); identidade no [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md).

## Consequências

### Positivas

- ADR-003-CONS-01: A janela de backoff acomoda indisponibilidades maiores que a alternativa curta, sem retry indefinido (`[09:15] Diego`, `[09:16] Diego`, `[09:17] Diego`).
- ADR-003-CONS-02: A DLQ preserva evidências e permite reprocessamento manual auditado (`[09:18] Diego`, `[09:36] Sofia`).

### Negativas

- ADR-003-CONS-03: A falha pode permanecer sem entrega após esgotar tentativas, exigindo replay administrativo (`[09:15] Diego`, `[09:18] Diego`).
- ADR-003-CONS-04: Marcar apenas `failed` na outbox foi preterido pela tabela separada, que acrescenta persistência em troca de leitura principal mais limpa (`[09:17] Larissa`, `[09:18] Diego`).
- ADR-003-CONS-05: A janela inicial de “12 ou 24 horas” era uma estimativa superada pela sequência fechada, não um prazo adicional garantido (`[09:15] Diego`, `[09:17] Diego`, `[09:17] Larissa`).

### Trade-off

- ADR-003-CONS-06: Aceitamos interromper automaticamente as retentativas e depender de replay manual após o teto em troca de limitar eventos pendurados e preservar a possibilidade de recuperação (`[09:15] Diego`, `[09:18] Diego`).

## Referências

- Transcrição: `[09:14] Larissa`; `[09:15] Diego`; `[09:16] Bruno`; `[09:16] Diego`; `[09:17] Diego`; `[09:17] Larissa`; `[09:18] Diego`; `[09:19] Larissa`; `[09:36] Sofia`; `[09:36] Larissa`; `[09:48] Larissa`.
- Código: `src/app.ts`, montagem em `/api/v1`; `src/routes/index.ts`, composição das rotas; `src/middlewares/auth.middleware.ts`, `requireRole` reutilizável para o replay.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [FDD, fluxos detalhados](../FDD.md#fluxos-detalhados), [FDD, estratégias de resiliência](../FDD.md#estratégias-de-resiliência), [FDD, contratos públicos](../FDD.md#contratos-públicos).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
