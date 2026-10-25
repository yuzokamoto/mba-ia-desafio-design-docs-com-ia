# ADR-001: Outbox transacional no MySQL

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:08] Larissa`; confirmado em `[09:48] Larissa`.
Relacionados: [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião.

## Contexto

As mudanças de status precisam produzir notificações sem depender da disponibilidade do destinatário. O time é pequeno e pretende aproveitar o MySQL existente (`[09:00] Marcos`, `[09:07] Diego`; `prisma/schema.prisma`, `docker-compose.yml`).

`OrderService.changeStatus` já executa `this.prisma.$transaction`: valida a transição, trata estoque quando aplicável, atualiza o pedido, cria o histórico e consulta o pedido atualizado. O débito ocorre somente em `PENDING` para `PAID`; a reposição, em `PAID` ou `PROCESSING` para `CANCELLED` (`src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`). A publicação precisa participar dessa unidade atômica (`[09:40] Bruno`, `[09:41] Diego`).

## Fatores de decisão

- O commit da mudança elegível deve implicar a persistência do evento (`[09:06] Diego`).
- Falha ao inserir na outbox deve desfazer a mudança de status (`[09:40] Bruno`).
- A entrega HTTP deve ocorrer em worker separado (`[09:06] Diego`).
- A infraestrutura disponível e o tamanho do time favorecem o MySQL existente (`[09:07] Diego`).
- A leitura precisa considerar o risco de acúmulo de eventos (`[09:07] Bruno`, `[09:08] Diego`).

## Alternativas Consideradas

### Opção 1: Outbox na transação MySQL (escolhida)

Persistir o evento junto da mudança de status e entregá-lo posteriormente (`[09:06] Diego`).

- Prós: Une commit e rollback do pedido e do evento sem acrescentar infraestrutura (`[09:06] Diego`, `[09:07] Diego`).
- Contras: Acrescenta uma tabela que exige leitura seletiva e cuidado com acúmulo (`[09:07] Bruno`, `[09:08] Diego`).

### Opção 2: HTTP síncrono no serviço de pedidos (ADR-001-ALT-01)

Levantada por `[09:03] Larissa` e descartada em `[09:04] Bruno` e `[09:06] Diego`: o custo de bloquear a transação e vinculá-la à disponibilidade externa foi determinante.

- Prós (avaliação do autor, não discutida na reunião): Dispensa tabela e processo adicionais e permite tentativa imediata.
- Contras: Cliente lento bloquearia mudanças; cliente indisponível criaria o dilema de rollback por falha HTTP (`[09:04] Bruno`).

### Opção 3: Redis Streams (ADR-001-ALT-02)

Levantada em `[09:07] Larissa` e descartada em `[09:07] Diego`: acrescentar Redis Cluster foi considerado excessivo para o time.

- Prós (avaliação do autor, não discutida na reunião): Oferece recursos de fila prontos para uso.
- Contras: Exige infraestrutura adicional, em troca de uma necessidade atendida pelo MySQL existente (`[09:07] Larissa`, `[09:07] Diego`).

## Decisão

ADR-001-DEC-01: Criar a tabela `webhook_outbox` no MySQL existente e inserir os eventos elegíveis dentro do `tx` de `OrderService.changeStatus`, na mesma transação de pedido, histórico e operações condicionais de estoque. Falha de inserção provoca rollback; a entrega fica a cargo do worker ([ADR-002](ADR-002-worker-separado-com-polling.md)). O filtro é aplicado na inserção; sem webhook interessado no status, não há evento a inserir (`[09:06] Diego`, `[09:34] Bruno`, `[09:40] Bruno`, `[09:41] Diego`; `src/modules/orders/order.service.ts`).

ADR-001-DEC-02: O gancho é a mudança de status, não `OrderService.create`; a nova publicação usará a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, a criar, conforme `[09:40] Bruno` e `[09:41] Bruno`, aceita por `[09:41] Diego`. O reuso dos erros, autenticação, logger e cliente de banco está registrado no [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md); o fluxo de integração fica no [FDD](../FDD.md#integração-com-o-sistema-existente).

## Consequências

### Positivas

- ADR-001-CONS-01: A transação confirma ou desfaz conjuntamente a mudança elegível e seu evento (`[09:06] Diego`, `[09:40] Bruno`).
- ADR-001-CONS-02: A entrega assíncrona desacopla a disponibilidade do cliente da transação de pedidos (`[09:04] Bruno`, `[09:06] Diego`).

### Negativas

- ADR-001-CONS-03: Falha da persistência de notificação também impede a mudança de status, por escolha de atomicidade (`[09:40] Bruno`, `[09:41] Diego`).
- ADR-001-CONS-04: O acúmulo exige índices e leitura de pendentes; arquivamento de entregues permanece fora desta feature, sem retenção exata definida (`[09:07] Bruno`, `[09:08] Diego`).

### Trade-off

- ADR-001-CONS-05: Aceitamos uma gravação adicional acoplada à transação em troca de atomicidade entre a mudança de status e o registro do evento (`[09:06] Diego`, `[09:40] Bruno`).

## Referências

- Transcrição: `[09:00] Marcos`; `[09:03] Larissa`; `[09:04] Bruno`; `[09:06] Diego`; `[09:07] Larissa`; `[09:07] Diego`; `[09:07] Bruno`; `[09:08] Diego`; `[09:08] Larissa`; `[09:34] Bruno`; `[09:40] Bruno`; `[09:41] Bruno`; `[09:41] Diego`; `[09:48] Larissa`.
- Código: `src/modules/orders/order.service.ts`, `OrderService.changeStatus` e `this.prisma.$transaction`; `src/modules/orders/order.status.ts`, condições de débito e reposição; `prisma/schema.prisma`, provider MySQL e modelos atuais sem outbox; `docker-compose.yml`, infraestrutura MySQL existente.
- Código de apoio ao reuso do [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md): `src/shared/errors/app-error.ts`, contrato `AppError`; `src/shared/errors/http-errors.ts`, erros HTTP e de domínio; `src/middlewares/error.middleware.ts`, tratamento HTTP central; `src/middlewares/auth.middleware.ts`, autenticação e `requireRole`; `src/shared/logger/index.ts`, Pino; `src/config/database.ts`, fábrica `createPrismaClient` e instância por processo.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [RFC, alternativas consideradas](../RFC.md#alternativas-consideradas), [FDD, fluxos detalhados](../FDD.md#fluxos-detalhados), [FDD, integração com o sistema existente](../FDD.md#integração-com-o-sistema-existente).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
