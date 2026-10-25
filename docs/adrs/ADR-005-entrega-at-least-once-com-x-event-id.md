# ADR-005: Entrega at-least-once com X-Event-Id

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:26] Larissa`; confirmado em `[09:48] Larissa`.
Relacionados: [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião.

## Contexto

A entrega pode fazer o cliente receber o mesmo evento mais de uma vez. A reunião adotou at-least-once e atribuiu ao consumidor a deduplicação por um UUID único do evento, enviado em `X-Event-Id` (`[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa`).

A política tem tentativas limitadas e DLQ, portanto a semântica precisa ser lida junto da persistência atômica e do replay. Há identidade do evento e identidade do cadastro destinatário, este distinguido por `X-Webhook-Id`; a relação exata entre evento e linhas da outbox não foi definida na reunião (`[09:06] Diego`, `[09:17] Larissa`, `[09:18] Diego`, `[09:44] Sofia`, `[09:51] Larissa`).

## Fatores de decisão

- Duplicatas são admitidas pela política de entrega (`[09:24] Diego`).
- O UUID deve identificar unicamente o evento desde sua entrada na outbox (`[09:25] Diego`).
- A deduplicação cabe ao consumidor, responsabilidade reconhecida explicitamente (`[09:25] Sofia`, `[09:26] Larissa`).
- A integração será documentada para os clientes no portal de desenvolvedor (`[09:26] Marcos`).
- A recuperação de eventos esgotados ocorre por replay manual da DLQ (`[09:18] Diego`).

## Alternativas Consideradas

### Opção 1: At-least-once com UUID estável (escolhida)

Permitir duplicatas e expor `X-Event-Id` para deduplicação pelo consumidor (`[09:25] Diego`, `[09:26] Larissa`).

- Prós: Oferece ao cliente a identificação necessária para reconhecer o mesmo evento (`[09:25] Diego`).
- Contras: Transfere ao cliente a responsabilidade de deduplicar (`[09:25] Sofia`).

### Opção 2: Exactly-once (ADR-005-ALT-01)

Descartada em `[09:25] Diego`: a coordenação bilateral e a complexidade exigidas superam o benefício pretendido.

- Prós (avaliação do autor, não discutida na reunião): Evita duplicatas para o consumidor.
- Contras: Exigiria coordenação dos dois lados e solução mais complexa; a expressão “99% dos casos” foi argumento, não meta aprovada (`[09:25] Diego`).

## Decisão

ADR-005-DEC-01: Adotar at-least-once e enviar em `X-Event-Id` o UUID gerado quando o evento entra na outbox, mantendo a identidade nas retentativas; o consumidor deduplica pelo `event_id` (`[09:25] Diego`, `[09:26] Larissa`). A garantia compreende registro atômico e envio pelo menos uma vez enquanto houver tentativas, com duplicatas possíveis e preservação na DLQ ao esgotá-las. Não promete recebimento eventual por cliente permanentemente indisponível (`[09:06] Diego`, `[09:15] Diego`, `[09:18] Diego`; [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md)).

ADR-005-PROP-01: **Proposta de implementação (não discutida na reunião):** gerar um `event_id` UUID v4 por evento lógico de mudança de status e uma linha de outbox por endpoint ativo cujo filtro contenha `to_status`; as linhas compartilham `event_id` e cada uma tem `id` UUID próprio. No replay, criar nova linha `PENDING` para o mesmo `webhookId`, preservando `event_id` e payload snapshot, com contador zerado e próxima tentativa imediata. Separar identidade lógica e persistência permite retentativas independentes por destino sem perder a deduplicação no replay (fundamentos: `[09:25] Diego`, `[09:34] Bruno`, `[09:44] Sofia`, `[09:51] Larissa`, `[09:18] Diego`; convenção UUID em `prisma/schema.prisma`). Os controles de repetição e auditoria do replay ficam no [FDD](../FDD.md#fluxos-detalhados).

## Consequências

### Positivas

- ADR-005-CONS-01: O consumidor pode reconhecer duplicatas usando a mesma identidade de evento (`[09:25] Diego`).
- ADR-005-CONS-02: Na proposta ADR-005-PROP-01, retentativa e replay preservam a identidade lógica mesmo com novas linhas de outbox, coerentemente com a deduplicação e o reprocessamento definidos (`[09:25] Diego`, `[09:18] Diego`).

### Negativas

- ADR-005-CONS-03: Cada consumidor precisa implementar a deduplicação, e Marcos precisa destacá-la na documentação da integração (`[09:25] Sofia`, `[09:26] Marcos`, `[09:26] Larissa`).
- ADR-005-CONS-04: O esgotamento pode deixar o evento na DLQ sem recebimento pelo cliente; a recuperação depende de replay manual, não de entrega eventual incondicional (`[09:15] Diego`, `[09:18] Diego`).

### Trade-off

- ADR-005-CONS-05: Aceitamos recebimentos duplicados e deduplicação no consumidor em troca de evitar a complexidade de coordenação bilateral discutida na reunião (`[09:24] Diego`, `[09:25] Diego`, `[09:25] Sofia`).

## Referências

- Transcrição: `[09:06] Diego`; `[09:15] Diego`; `[09:17] Larissa`; `[09:18] Diego`; `[09:24] Diego`; `[09:25] Diego`; `[09:25] Sofia`; `[09:26] Marcos`; `[09:26] Larissa`; `[09:34] Bruno`; `[09:44] Sofia`; `[09:48] Larissa`; `[09:51] Larissa`.
- Código: `prisma/schema.prisma`, UUID predominante nas entidades de domínio, sem modelos atuais de webhook ou outbox.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [RFC, alternativas consideradas](../RFC.md#alternativas-consideradas), [FDD, fluxos detalhados](../FDD.md#fluxos-detalhados), [FDD, contratos públicos](../FDD.md#contratos-públicos).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
