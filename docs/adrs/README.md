# Registros de decisões arquiteturais

Convenção de nomes: `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Título | Status | Relacionados |
|---|---|---|---|
| [ADR-001](ADR-001-outbox-transacional-no-mysql.md) | Outbox transacional no MySQL | Aceito | [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) |
| [ADR-002](ADR-002-worker-separado-com-polling.md) | Worker separado com polling | Aceito | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md) | Retentativas com backoff e DLQ | Aceito | [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint | Aceito | [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com X-Event-Id | Aceito | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md) |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload na inserção | Aceito | [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) |

O [RFC](../RFC.md) é a proposta que originou estes registros de decisão.

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
