# ADR-002: Worker separado com polling

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:10] Larissa` e `[09:11] Diego`; confirmado em `[09:48] Larissa`.
Relacionados: [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião.

## Contexto

A outbox precisa de um consumidor que execute as chamadas HTTP. A expectativa de negócio é a chegada da notificação ao sistema do cliente em menos de 10 segundos após a mudança de status confirmada ([09:00] Marcos, [09:02] Marcos); o polling de 2 segundos foi aceito pelo PM (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Marcos`). Esse intervalo contribui para a latência, sujeita a backlog (`[09:07] Bruno`, `[09:08] Diego`).

**Proposta de implementação (não discutida na reunião):** o OMS mede commit da mudança (linha na outbox) até início do envio da primeira tentativa, a parte que controla; não observa o instante de chegada ao cliente. Esse indicador operacional não comprova sozinho a meta de produto. O tempo de resposta, registrado no histórico, ocorre após a chegada e pode atingir o timeout de 10 s ([09:34] Marcos, [09:42] Diego; [FDD](../FDD.md#observabilidade)).

A execução precisa sobreviver independentemente dos reinícios da API e usar o mesmo banco e stack. `src/server.ts` é a referência de entrada existente; `package.json` ainda não tem script de worker (`[09:11] Diego`, `[09:11] Larissa`, `[09:30] Bruno`).

## Fatores de decisão

- Separar o processo evita vincular a execução do worker aos reinícios da API (`[09:11] Diego`).
- O intervalo de 2 segundos foi aceito para a expectativa de notificação (`[09:10] Marcos`, `[09:10] Larissa`).
- O MySQL existente não oferece a notificação externa pretendida por trigger (`[09:09] Diego`).
- A execução inicial com um worker limita as garantias de ordenação e escala (`[09:12] Diego`, `[09:13] Larissa`).
- Cada processo Node terá sua própria instância de Prisma, usando a mesma `DATABASE_URL` (`[09:30] Bruno`).

## Alternativas Consideradas

### Opção 1: Processo separado com polling (escolhida)

Consultar os pendentes mais antigos a cada 2 segundos em processo próprio (`[09:09] Diego`, `[09:11] Diego`).

- Prós: Tem execução independente da API e intervalo aceito pelo PM (`[09:10] Marcos`, `[09:11] Diego`).
- Contras: Introduz espera de polling e uma entrada adicional de execução (`[09:10] Larissa`, `[09:11] Larissa`).

### Opção 2: Worker no processo da API (ADR-002-ALT-01)

Descartada por `[09:11] Diego`: o reinício da API interromperia também o worker, anulando a independência pretendida.

- Prós (avaliação do autor, não discutida na reunião): Reduz em um o número de processos a implantar e monitorar.
- Contras: Acopla o ciclo de vida das entregas ao da API (`[09:11] Diego`).

### Opção 3: Trigger para notificar o worker (ADR-002-ALT-02)

Proposta em `[09:09] Bruno` para maior reatividade e descartada em `[09:09] Diego`: a trigger executa SQL e não notifica diretamente processo externo.

- Prós: Pretendia reagir à alteração no banco sem aguardar polling (`[09:09] Bruno`).
- Contras: Exigiria contornos como arquivo ou chamada a endpoint, considerados inadequados (`[09:09] Diego`).

## Decisão

ADR-002-DEC-01: Executar um único worker Node em processo separado, com `src/worker.ts` e script `npm run worker`, ambos a criar, consultando pendentes mais antigos a cada 2 segundos. Reutilizar a stack e o banco da API, com instância própria de Prisma por processo (`[09:09] Diego`, `[09:10] Larissa`, `[09:11] Larissa`, `[09:11] Diego`, `[09:12] Diego`, `[09:30] Bruno`; `src/server.ts`, `src/config/database.ts`, `package.json`). **Proposta de implementação (não discutida na reunião):** cada ciclo começa 2 s após o início do anterior; se durar mais, o próximo começa ao fim dele, sem sobreposição. Enviar o lote em paralelo, até `WEBHOOK_BATCH_SIZE = 10`, com no máximo uma linha por par (`webhookId`, `orderId`), mantém um único processo e limita o ciclo a cerca de um timeout de 10 s mais tempo de banco ([09:08] Diego, [09:09] Diego, [09:10] Larissa, [09:42] Diego).

ADR-002-DEC-02: Ordenação por `order_id` enquanto houver um único worker; sem ordenação global ([09:12] Diego, [09:13] Larissa, [09:14] Marcos). **Proposta de implementação (não discutida na reunião):** fila FIFO por par (`webhookId`, `orderId`): uma linha só é elegível se nenhuma anterior por (`createdAt`, `id`) do mesmo par estiver `PENDING` ou `PROCESSING`; índice composto apoia a consulta. Retry bloqueia os eventos seguintes desse pedido para esse endpoint até entrega ou DLQ. Depois da DLQ, os seguintes avançam e um replay posterior pode entregar o antigo fora de ordem; o payload preserva `from_status`, `to_status` e `timestamp` ([09:17] Diego, [09:18] Diego, [09:43] Diego; nomes conforme `prisma/schema.prisma`). Consulta e índice no [FDD](../FDD.md#fdd-flow-02-processamento-pelo-worker).

## Consequências

### Positivas

- ADR-002-CONS-01: Reinícios da API deixam de determinar o ciclo de vida do worker (`[09:11] Diego`).
- ADR-002-CONS-02: O polling usa o banco existente com intervalo explicitamente aceito pelo PM (`[09:09] Diego`, `[09:10] Marcos`).

### Negativas

- ADR-002-CONS-03: Polling e backlog podem comprometer a chegada ao cliente em menos de 10 s ([09:02] Marcos, [09:07] Bruno, [09:10] Larissa). **Proposta de implementação (não discutida na reunião):** destinatários lentos atrasam novos envios em até um ciclo, cerca de um timeout de 10 s mais banco; o indicador interno mede apenas commit até início, e o histórico registra separadamente o tempo de resposta ([09:34] Marcos, [09:42] Diego; ADR-002-DEC-01).
- ADR-002-CONS-04: **Proposta de implementação (não discutida na reunião):** falha persistente de um endpoint segura os eventos seguintes daquele pedido para aquele endpoint até a DLQ, preservando a ordem durante retries; o mesmo endpoint provavelmente também falharia. Após DLQ, o replay pode chegar fora de ordem, conforme ADR-002-DEC-02 ([09:12] Diego, [09:13] Larissa, [09:17] Diego, [09:18] Diego, [09:43] Diego).
- ADR-002-CONS-05: Múltiplos workers, particionamento por `order_id` e lock pessimista ficam adiados, como questão de escala no [RFC](../RFC.md#questões-em-aberto), sem garantia atual para esse cenário (`[09:12] Diego`, `[09:13] Diego`).

### Trade-off

- ADR-002-CONS-06: Aceitamos a espera do polling e a limitação inicial de escala em troca de execução independente sobre a infraestrutura existente (`[09:09] Diego`, `[09:10] Larissa`, `[09:11] Diego`, `[09:13] Diego`).

## Referências

- Transcrição: `[09:02] Marcos`; `[09:07] Bruno`; `[09:08] Diego`; `[09:09] Diego`; `[09:09] Bruno`; `[09:10] Marcos`; `[09:10] Larissa`; `[09:11] Diego`; `[09:11] Larissa`; `[09:12] Diego`; `[09:13] Larissa`; `[09:13] Diego`; `[09:17] Diego`; `[09:30] Bruno`; `[09:42] Diego`; `[09:48] Larissa`.
- Código: `src/server.ts`, entrada da API; `src/config/database.ts`, `createPrismaClient`; `package.json`, stack Node e ausência de script `worker`; `prisma/schema.prisma`, campos `createdAt` em camelCase.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [RFC, questões em aberto](../RFC.md#questões-em-aberto), [FDD, fluxos detalhados](../FDD.md#fluxos-detalhados), [FDD, estratégias de resiliência](../FDD.md#estratégias-de-resiliência).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
