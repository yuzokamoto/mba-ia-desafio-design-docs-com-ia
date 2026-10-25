# ADR-007: Snapshot do payload na inserção

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:52] Larissa`; confirmado em `[09:52] Bruno`.
Relacionados: [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião. Esta decisão foi tomada após o resumo de `[09:48] Larissa`.

## Contexto

O processamento assíncrono permite que o pedido mude entre a publicação e a entrega. Bruno perguntou se a outbox guardaria o payload renderizado ou apenas `order_id`; Larissa escolheu o snapshot para preservar o estado do instante da mudança (`[09:51] Bruno`, `[09:52] Larissa`).

Retentativas e replay ampliam essa distância temporal. O payload JSON da transição já foi delimitado e a DLQ precisa preservá-lo; `OrderService.changeStatus` já consulta o pedido atualizado dentro da transação, oferecendo a base para capturar o evento (`[09:17] Diego`, `[09:18] Diego`, `[09:43] Diego`; `src/modules/orders/order.service.ts`).

## Fatores de decisão

- O evento deve continuar refletindo o estado de quando o status mudou (`[09:52] Larissa`).
- O pedido pode sofrer alterações posteriores à inserção na outbox (`[09:52] Larissa`).
- O payload da transição é JSON e tem conteúdo delimitado (`[09:43] Diego`).
- A DLQ deve preservar o payload para diagnóstico e reprocessamento (`[09:18] Diego`).
- A publicação ocorre no cliente transacional já disponível (`[09:41] Bruno`, `[09:41] Diego`; `src/modules/orders/order.service.ts`).

## Alternativas Consideradas

### Opção 1: Renderizar e persistir na inserção (escolhida)

Guardar o snapshot renderizado quando o evento entra na outbox (`[09:52] Larissa`, `[09:52] Diego`, `[09:52] Bruno`).

- Prós: Preserva o significado temporal do evento mesmo que o pedido mude depois (`[09:52] Larissa`).
- Contras: Guarda o payload junto do evento, além da identificação do pedido, consequência direta da escolha discutida por `[09:51] Bruno` e `[09:52] Larissa`.

### Opção 2: Guardar order_id e renderizar no envio (ADR-007-ALT-01)

Levantada em `[09:51] Bruno` e preterida em `[09:52] Larissa`: ler o estado posterior alteraria o significado do evento da transição.

- Prós (avaliação do autor, não discutida na reunião): Reduz o tamanho das linhas e usa os dados atuais no envio.
- Contras: Mudanças posteriores poderiam produzir payload diferente do estado que originou a notificação (`[09:52] Larissa`).

## Decisão

ADR-007-DEC-01: Renderizar e armazenar o payload como snapshot no momento da inserção na outbox, dentro da transação da mudança de status. Usar o pedido atualizado `refreshed`, já consultado por `OrderService.changeStatus`, e os status anterior e novo explícitos na publicação, para capturar a transição correta (`[09:40] Bruno`, `[09:41] Bruno`, `[09:52] Larissa`, `[09:52] Diego`, `[09:52] Bruno`; `src/modules/orders/order.service.ts`). A decisão cobre o payload, sem determinar snapshot de URL ou secret.

ADR-007-PROP-01: **Proposta de implementação (não discutida na reunião):** serializar o JSON uma única vez e armazená-lo como texto para reutilizar os mesmos bytes nas retentativas e na assinatura; preservar o mesmo snapshot no replay, evitando reconstruí-lo a partir do pedido atual. Isso mantém a fidelidade temporal e a coerência entre corpo e HMAC (fundamentos: `[09:18] Diego`, `[09:22] Sofia`, `[09:52] Larissa`; [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)). O contrato completo do payload fica no [FDD](../FDD.md#contratos-públicos).

## Consequências

### Positivas

- ADR-007-CONS-01: Alterações posteriores do pedido não modificam o conteúdo do evento já registrado (`[09:52] Larissa`).
- ADR-007-CONS-02: Na proposta ADR-007-PROP-01, retentativa e replay reutilizam o conteúdo original, coerente com a evidência preservada na DLQ (`[09:18] Diego`, `[09:52] Larissa`).

### Negativas

- ADR-007-CONS-03: A persistência carrega o corpo renderizado em vez de apenas uma referência; o custo de armazenamento decorre dessa escolha, sem estimativa quantitativa na reunião (`[09:51] Bruno`, `[09:52] Larissa`).
- ADR-007-CONS-04: O payload entregue pode representar um estado anterior ao estado atual do pedido, consequência intencional do snapshot; detalhes atuais continuam consultáveis na API de pedidos (`[09:52] Larissa`, `[09:43] Diego`).

### Trade-off

- ADR-007-CONS-05: Aceitamos armazenar o payload renderizado em troca de preservar fielmente o estado da transição independentemente de mudanças posteriores (`[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`).

## Referências

- Transcrição: `[09:17] Diego`; `[09:18] Diego`; `[09:22] Sofia`; `[09:40] Bruno`; `[09:41] Bruno`; `[09:41] Diego`; `[09:43] Diego`; `[09:48] Larissa`; `[09:51] Bruno`; `[09:52] Larissa`; `[09:52] Diego`; `[09:52] Bruno`.
- Código: `src/modules/orders/order.service.ts`, transação de `changeStatus` e consulta `refreshed` antes do retorno.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [RFC, alternativas consideradas](../RFC.md#alternativas-consideradas), [FDD, fluxos detalhados](../FDD.md#fluxos-detalhados), [FDD, contratos públicos](../FDD.md#contratos-públicos), [FDD, integração com o sistema existente](../FDD.md#integração-com-o-sistema-existente).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
