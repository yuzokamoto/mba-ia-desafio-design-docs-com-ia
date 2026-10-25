# ADR-006: Reuso dos padrões existentes do projeto

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:30] Larissa`; confirmado em `[09:48] Larissa`.
Relacionados: [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião. A escolha de UUID da outbox foi decidida depois do resumo, em `[09:51] Larissa`, e confirmada em `[09:51] Diego`.

## Contexto

O projeto organiza pedidos em controller, service, repository, routes e schemas e já oferece erros, logging, validação e autenticação compartilhados. A reunião fechou o reuso dessa organização para webhooks (`[09:27] Bruno`, `[09:28] Bruno`, `[09:29] Bruno`, `[09:30] Larissa`; arquivos do módulo de pedidos listados nas referências).

O banco e os componentes compartilhados serão reutilizados com instância própria de Prisma por processo ([09:30] Bruno; `src/config/database.ts`). A integração do publisher é decidida no [ADR-001](ADR-001-outbox-transacional-no-mysql.md); a autorização do replay, no [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md).

## Fatores de decisão

- O módulo novo deve seguir a organização dos domínios existentes (`[09:27] Bruno`, `[09:30] Larissa`).
- Erros próprios devem seguir `AppError` e usar prefixo `WEBHOOK_` (`[09:28] Bruno`, `[09:29] Larissa`).
- Pino e o middleware HTTP central devem ser reutilizados (`[09:29] Bruno`).
- Cada processo precisa de instância própria de Prisma com a mesma `DATABASE_URL` (`[09:30] Bruno`).
- As integrações reutilizam os componentes existentes, conforme [ADR-001](ADR-001-outbox-transacional-no-mysql.md) e [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md).
- IDs da outbox seguem o padrão UUID do projeto ([09:51] Larissa; `prisma/schema.prisma`).

## Alternativas Consideradas

### Opção 1: Reutilizar os padrões existentes (escolhida)

Criar webhooks como módulo do projeto, com suas dependências compartilhadas e IDs UUID ([09:30] Larissa, [09:51] Larissa).

- Prós: Mantém a estrutura e os contratos de erros e logging já conhecidos pelo time (`[09:27] Bruno`, `[09:28] Bruno`, `[09:29] Bruno`).
- Contras: O reuso exige respeitar os limites dos componentes, pois `errorMiddleware` trata requisições Express, não o worker (`src/middlewares/error.middleware.ts`; `[09:11] Diego`).

### Opção 2: Criar padrões próprios para o módulo (ADR-006-ALT-01)

Alternativa analítica, não discutida na reunião: nova organização de módulo e componentes próprios em vez de reutilizar os existentes. Preterida pela decisão de reuso máximo ([09:27] Bruno, [09:29] Bruno, [09:30] Larissa).

- Prós (avaliação do autor, não discutida na reunião): Permitiria adaptar estrutura, validação, erros e logging exclusivamente ao domínio novo.
- Contras: Duplicaria componentes que o time decidiu reutilizar ([09:30] Larissa; `src/shared/errors/app-error.ts`, `src/shared/logger/index.ts`).

### Opção 3: ID auto incremental na outbox (ADR-006-ALT-02)

Levantada em `[09:51] Diego` e preterida em `[09:51] Larissa` pela consistência com UUID nas entidades de domínio.

- Prós (avaliação do autor, não discutida na reunião): Oferece chave monotônica e compacta.
- Contras: Diverge do padrão predominante escolhido para a nova entidade; `OrderNumberSequence` é uma exceção com ID inteiro, sem `autoincrement()`, no código atual (`[09:51] Larissa`; `prisma/schema.prisma`).

## Decisão

ADR-006-DEC-01: Criar `src/modules/webhooks` (a criar) com controller, service, repository, routes e schemas Zod, reutilizando `AppError`, códigos próprios `WEBHOOK_`, Pino e `errorMiddleware` para HTTP. Preservar os códigos genéricos de autenticação e o comportamento atual dos erros compartilhados; o worker trata e registra suas próprias falhas, pois não passa pelo Express (`[09:27] Bruno`, `[09:28] Bruno`, `[09:29] Larissa`, `[09:29] Bruno`, `[09:30] Larissa`; `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`).

ADR-006-DEC-02: Reutilizar `createPrismaClient` com instância por processo e mesma `DATABASE_URL` ([09:30] Bruno; `src/config/database.ts`). Adotar UUID na outbox como reuso explícito: “UUID, segue o padrão do resto do projeto. Tudo é uuid.” ([09:51] Larissa; `prisma/schema.prisma`). A composição de autenticação do replay apenas aplica o [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md), usando `authenticate` e `requireRole` existentes ([09:36] Larissa; `src/middlewares/auth.middleware.ts`, `src/modules/users/user.routes.ts`). A integração do publisher apenas aplica o [ADR-001](ADR-001-outbox-transacional-no-mysql.md), sem nova decisão neste registro ([09:41] Bruno, [09:41] Diego; `src/modules/orders/order.service.ts`).

## Consequências

### Positivas

- ADR-006-CONS-01: A API mantém o envelope central de erros e a organização dos módulos, com Zod e Pino existentes (`[09:30] Larissa`; `src/middlewares/error.middleware.ts`).
- ADR-006-CONS-02: O publisher reutiliza a transação corrente conforme [ADR-001](ADR-001-outbox-transacional-no-mysql.md), sem decidir sua integração novamente ([09:41] Bruno, [09:41] Diego).
- ADR-006-CONS-03: Replay reutiliza proteção por role conforme [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md), sem redefinir autorização ([09:36] Larissa; `src/middlewares/auth.middleware.ts`, `src/modules/users/user.routes.ts`).

### Negativas

- ADR-006-CONS-04: O middleware HTTP não cobre falhas do worker; a mesma instância de Prisma também não é compartilhada entre processos (`src/middlewares/error.middleware.ts`, `src/config/database.ts`; `[09:30] Bruno`).
- ADR-006-CONS-05: `NotFoundError` e `ValidationError` têm códigos fixos; erros próprios com `WEBHOOK_` devem usar `AppError` ou subclasses de código parametrizável, sem presumir conversão automática pelo middleware (`src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`; `[09:29] Larissa`).
- ADR-006-CONS-06: Reutilizar Pino não protege automaticamente novos campos sensíveis, pois a redação atual não inclui secrets de webhook; o ajuste proposto está no [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) (`src/shared/logger/index.ts`; `[09:22] Diego`).

### Trade-off

- ADR-006-CONS-07: Aceitamos adaptar o módulo aos contratos e limites dos componentes existentes em troca de consistência com a base e reuso da infraestrutura compartilhada (`[09:30] Larissa`; `src/middlewares/error.middleware.ts`, `src/config/database.ts`).

## Referências

- Transcrição: `[09:11] Diego`; `[09:22] Diego`; `[09:27] Bruno`; `[09:28] Bruno`; `[09:29] Larissa`; `[09:29] Bruno`; `[09:30] Bruno`; `[09:30] Larissa`; `[09:36] Larissa`; `[09:40] Bruno`; `[09:41] Bruno`; `[09:41] Diego`; `[09:48] Larissa`; `[09:51] Diego`; `[09:51] Larissa`; `[09:52] Larissa`.
- Código: `src/modules/orders/order.controller.ts`, encaminhamento de falhas; `src/modules/orders/order.service.ts`, `OrderService.changeStatus`, `this.prisma.$transaction` e `refreshed`; `src/modules/orders/order.repository.ts`, persistência; `src/modules/orders/order.routes.ts`, composição de rotas; `src/modules/orders/order.schemas.ts`, schemas Zod; `src/modules/orders/order.status.ts`, regras condicionais de estoque.
- Código: `src/shared/errors/app-error.ts`, contrato com código e detalhes; `src/shared/errors/http-errors.ts`, subclasses com códigos fixos ou parametrizáveis; `src/middlewares/error.middleware.ts`, tratamento Express de `AppError`, Zod e erros Prisma selecionados; `src/middlewares/auth.middleware.ts`, `authenticate` e `requireRole`; `src/modules/users/user.routes.ts`, composição com `ADMIN`; `src/shared/logger/index.ts`, Pino e redação atual; `src/config/database.ts`, fábrica `createPrismaClient` e instância exportada; `prisma/schema.prisma`, provider MySQL, UUID predominante, campos camelCase e tabelas com `@@map`.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [FDD, integração com o sistema existente](../FDD.md#integração-com-o-sistema-existente), [FDD, dependências e compatibilidade](../FDD.md#dependências-e-compatibilidade).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
