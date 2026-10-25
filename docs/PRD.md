# PRD: Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0

Data: 2026-10-25 (produção do documento; a reunião não informa data civil)

Responsável: Larissa (Tech Lead), conforme [09:50] Larissa

---

## Resumo e contexto da feature

**PRD-CTX-01:** Webhooks outbound no OMS existente reduzem consultas periódicas, lentidão, custo e risco de perda da Atlas ([09:00] Marcos, [09:02] Marcos). Integração em `src/modules/orders/order.service.ts` ([09:40] Bruno). Marcos responde pelo produto e comunicação ([09:47] Marcos, [09:49] Marcos); Larissa conduz o desenho ([09:50] Larissa). Abordagem no [RFC](RFC.md), decisões nos [ADRs](adrs/README.md), construção no [FDD](FDD.md).

---

## Problema e motivação

Prioridades avaliadas pela autora conforme impacto relatado.

| ID | Problema | Impacto | Prioridade | Fonte |
|---|---|---|---|---|
| PRD-PROB-01 | Dependência de consultas periódicas para descobrir mudanças de status. | Integração lenta e cara; Atlas considera migrar sem entrega até fim do trimestre. | Alta | [09:00] Marcos |
| PRD-PROB-02 | Espera sem notificação obriga atualização manual. | Clientes aceitam notificação em menos de 10 segundos. | Alta | [09:02] Marcos |

---

## Público-alvo e cenários de uso

Público-alvo:

- **PRD-PUB-01:** Atlas Comercial, MaxDistribuição e Nova Cargo; seus softwares não foram nomeados ([09:00] Marcos, [09:02] Marcos).
- **PRD-PUB-02:** Representantes autenticados na API; cliente informado explicitamente, sem inferência do JWT ([09:32] Marcos, [09:32] Larissa; `src/middlewares/auth.middleware.ts`).
- **PRD-PUB-03:** Administradores `ADMIN` recuperam falhas por replay ([09:18] Diego, [09:36] Sofia).
- **PRD-PUB-04:** Desenvolvedores consumidores do portal de integração e deduplicação de Marcos ([09:26] Marcos, [09:40] Marcos).

Cenários de uso chave:

- **PRD-CEN-01:** O representante cadastra um destino HTTPS e escolhe os status que deseja acompanhar ([09:31] Marcos, [09:23] Sofia).
- **PRD-CEN-02:** Receber mudança de status selecionado, por exemplo `SHIPPED` e `DELIVERED` ([09:00] Marcos, [09:33] Marcos, [09:34] Bruno).
- **PRD-CEN-03:** Diagnosticar sucesso/falha, conteúdo, resposta e duração ([09:34] Marcos).
- **PRD-CEN-04:** Recuperar indisponibilidades com retentativas e replay administrativo ([09:15] Diego, [09:17] Diego, [09:18] Diego, [09:36] Sofia).
- **PRD-CEN-05:** O representante solicita nova secret e migra seu sistema durante a convivência de 24 h com a anterior ([09:21] Sofia).

---

## Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Fonte |
|---|---|---|---|---|
| PRD-OBJ-01 | Disponibilizar rapidamente a mudança. | Tempo entre a mudança de status confirmada e a chegada da notificação ao sistema do cliente. | Menor que 10 s. | [09:00] Marcos, [09:02] Marcos |
| PRD-OBJ-02 | Atender solicitantes no prazo comercial | Feature disponível em produção | 3 de 3 clientes, até fim de novembro, pedido da Atlas a confirmar; risco comercial até fim do trimestre em PRD-PROB-01 | [09:00] Marcos, [09:45] Marcos, [09:47] Marcos |
| PRD-OBJ-03 | Encerrar a recuperação automática. | Entrega confirmada ou falha preservada para recuperação. | Todo evento publicado entregue ou recuperável ao esgotar retentativas; **Proposta de implementação (não discutida na reunião):** janela nominal de 14h36, a confirmar em RFC-OQ-07. | [09:16] Diego, [09:17] Diego, [09:17] Larissa, [09:48] Larissa, [09:18] Diego |

**Proposta de implementação (não discutida na reunião):** o OMS mede mudança confirmada até início do primeiro envio, pois não observa a chegada ao cliente. Polling de 2 s contribui para esse indicador; backlog pode prejudicar a meta ([09:06] Diego, [09:09] Diego, [09:10] Larissa, [09:07] Bruno). O tempo de resposta ocorre após a chegada e não integra a meta ([09:42] Diego). Instrumentação no [FDD](FDD.md#observabilidade).

**Proposta de implementação (não discutida na reunião):** PRD-OBJ-03 adota envio inicial + 5 retentativas, a confirmar com Bruno e Diego ([09:50] Larissa; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)). A janela não inclui chamadas, polling ou backlog. Conciliar remoções separadamente ([09:33] Bruno; [FDD-API-04](FDD.md#fdd-api-04-delete-apiv1webhooksid)). Sem baseline informado.

---

## Escopo

### Incluso

- **PRD-SCOPE-01:** Gestão por cliente via API: cadastro, listagem, edição, remoção e seleção de status ([09:31] Marcos, [09:32] Larissa, [09:33] Bruno, [09:33] Marcos). **Proposta de implementação (não discutida na reunião):** ativar/desativar operacionaliza estado ativo ([09:21] Bruno; PRD-FR-05).
- **PRD-SCOPE-02:** Outbound filtrado por status, fiel à transição ([09:00] Marcos, [09:02] Marcos, [09:33] Marcos, [09:34] Bruno, [09:52] Larissa).
- **PRD-SCOPE-03:** Secret gerada pelo OMS, origem/integridade, HTTPS, rotação com 24 h de convivência ([09:19] Sofia, [09:21] Sofia, [09:23] Sofia, [09:31] Marcos).
- **PRD-SCOPE-04:** Diagnóstico de entregas, retentativas, falhas recuperáveis e replay administrativo auditado ([09:34] Marcos, [09:17] Diego, [09:18] Diego, [09:36] Sofia).
- **PRD-SCOPE-05:** Identidade para deduplicação e documentação por Marcos no portal ([09:25] Diego, [09:26] Marcos, [09:40] Marcos).

### Fora de escopo

| ID | Item | Status | Fonte |
|---|---|---|---|
| PRD-OUT-01 | Aviso de falhas por email, avaliar fase futura após medir impacto. | Adiado | [09:37] Larissa, [09:38] Marcos |
| PRD-OUT-02 | Dashboard visual para clientes, projeto separado do time de frontend. | Adiado | [09:40] Larissa |
| PRD-OUT-03 | Rate limiting outbound, observar e decidir depois, sem limite de taxa aprovado. | Em aberto | [09:38] Diego, [09:39] Diego, [09:39] Larissa |
| PRD-OUT-04 | Receber webhooks inbound no OMS. | Excluído | [09:02] Marcos |
| PRD-OUT-05 | Arquivar eventos entregues; não há política exata de retenção decidida. | Adiado | [09:08] Diego |
| PRD-OUT-06 | Múltiplos workers e mecanismos futuros de escala. | Adiado | [09:13] Diego |
| PRD-OUT-07 | Garantia de ordenação global. | Sem garantia | [09:13] Larissa, [09:14] Marcos |
| PRD-OUT-08 | Garantia exactly-once. | Descartado | [09:25] Diego |
| PRD-OUT-09 | Itens na notificação; detalhes pela API de pedidos. | Excluído | [09:43] Diego |
| PRD-OUT-10 | Endurecimento adicional das permissões de configuração. | Adiado | [09:37] Sofia |

---

## Requisitos funcionais

Prioridades avaliadas pela autora conforme [09:48] Larissa. Autenticação: [09:32] Marcos, [09:37] Sofia, `src/middlewares/auth.middleware.ts`; sem isolamento usuário-cliente atual (`prisma/schema.prisma`).

#### PRD-FR-01 Cadastrar webhook e receber secret

Cadastrar destino e receber secret exclusiva.

**Fluxo principal**

- Usuário autenticado informa cliente, URL e status; recebe cadastro confirmado e secret gerada.

**Fluxos alternativos e exceções**

- Cliente não vem do JWT. **Proposta de implementação (não discutida na reunião):** secret visível somente na criação/rotação, reduzindo exposição.

**Erros previstos**

- Autenticação, cliente ou configuração inválidos.

**Prioridade:** alta.

**Fonte:** [09:31] Marcos, [09:32] Larissa, [09:21] Sofia, [09:23] Sofia; `src/middlewares/auth.middleware.ts`, `src/modules/orders/order.schemas.ts` (convenção de identificação do cliente).

**Contrato:** [FDD-API-01](FDD.md#fdd-api-01-post-apiv1webhooks).

---

#### PRD-FR-02 Listar webhooks de um cliente

Consultar os webhooks de um cliente.

**Fluxo principal**

- Usuário autenticado informa o cliente; o sistema apresenta seus cadastros.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** lista vazia sem cadastros; consultas omitem secrets para reduzir exposição.

**Erros previstos**

- Autenticação ou cliente inválidos; cliente inexistente.

**Prioridade:** alta.

**Fonte:** [09:33] Bruno, [09:37] Sofia, [09:31] Marcos; `src/middlewares/auth.middleware.ts`, `src/modules/orders/order.schemas.ts` (convenções de consulta).

**Contrato:** [FDD-API-02](FDD.md#fdd-api-02-get-apiv1customerscustomeridwebhooks).

---

#### PRD-FR-03 Editar configuração

Alterar configuração existente.

**Fluxo principal**

- Usuário identifica cadastro, informa alterações e recebe confirmação.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** URL e seleção de status editáveis; novo filtro vale para futuras notificações. Secret segue PRD-FR-08.

**Erros previstos**

- Autenticação, cadastro ou configuração inválidos.

**Prioridade:** alta.

**Fonte:** [09:33] Bruno, [09:33] Marcos, [09:21] Sofia, [09:23] Sofia, [09:28] Bruno, [09:37] Sofia; `src/middlewares/auth.middleware.ts`.

**Contrato:** [FDD-API-03](FDD.md#fdd-api-03-patch-apiv1webhooksid).

---

#### PRD-FR-04 Remover webhook

Remover configuração pela API.

**Fluxo principal**

- Usuário autenticado identifica o cadastro e confirma sua remoção.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** remoção encerra notificações pendentes, sem cancelar envio iniciado. Efeitos sobre informações consultáveis no FDD.

**Erros previstos**

- Autenticação inválida ou cadastro inexistente, inclusive remoção repetida.

**Prioridade:** alta.

**Fonte:** [09:33] Bruno, [09:28] Bruno, [09:37] Sofia; `src/middlewares/auth.middleware.ts`, `src/modules/orders/order.service.ts`, `src/modules/customers/customer.repository.ts` (convenção de exclusão).

**Contrato:** [FDD-API-04](FDD.md#fdd-api-04-delete-apiv1webhooksid).

---

#### PRD-FR-05 Ativar e desativar webhook

**Proposta de implementação (não discutida na reunião):** permitir pausar notificações, operacionalizando o estado ativo.

**Fluxo principal**

- Usuário autenticado altera atividade; somente cadastros ativos recebem novas notificações.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** desativar preserva possibilidade de recuperação administrativa após reativar; não equivale a excluir.

**Erros previstos**

- Autenticação inválida, cadastro inexistente ou estado inválido.

**Prioridade:** alta.

**Fonte:** [09:21] Bruno, [09:21] Sofia, [09:33] Bruno, [09:34] Bruno, [09:18] Diego; `src/middlewares/auth.middleware.ts`.

**Contrato:** [FDD-API-03](FDD.md#fdd-api-03-patch-apiv1webhooksid), com efeitos em [FDD-FLOW-02](FDD.md#fdd-flow-02-processamento-pelo-worker).

---

#### PRD-FR-06 Selecionar notificações pelo status de destino

Notificar somente os status selecionados.

**Fluxo principal**

- Pedido muda para status válido; somente configurações interessadas recebem a notificação.

**Fluxos alternativos e exceções**

- Sem interessados não há notificação. **Proposta de implementação (não discutida na reunião):** seleção restrita aos destinos da máquina de estados atual.

**Erros previstos**

- Seleção ou transição inválidas; transição recusada não notifica.

**Prioridade:** alta.

**Fonte:** [09:33] Marcos, [09:34] Bruno; `src/modules/orders/order.status.ts`, `src/modules/orders/order.service.ts`.

**Contrato:** [FDD-API-01](FDD.md#fdd-api-01-post-apiv1webhooks) e [FDD-API-09](FDD.md#fdd-api-09-publishwebhookeventtx-order-fromstatus-tostatus).

---

#### PRD-FR-07 Notificar automaticamente mudanças válidas

Notificar automaticamente mudanças válidas, com conteúdo fiel à transição.

**Fluxo principal**

- OMS confirma mudança e notificação juntas; inicia entrega verificável pelo destinatário.

**Fluxos alternativos e exceções**

- Indisponibilidade segue PRD-FR-10, sem desfazer mudança confirmada. Criação do pedido não notifica.

**Erros previstos**

- Falha no registro impede confirmar mudança; transição inválida ou estoque insuficiente mantêm os erros atuais.

**Prioridade:** alta.

**Fonte:** [09:00] Marcos, [09:02] Marcos, [09:06] Diego, [09:19] Sofia, [09:40] Bruno, [09:52] Larissa; `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`.

**Contrato:** [FDD-API-08](FDD.md#fdd-api-08-post-para-a-url-https-do-cliente) e [FDD-API-09](FDD.md#fdd-api-09-publishwebhookeventtx-order-fromstatus-tostatus).

---

#### PRD-FR-08 Rotacionar secret com transição de 24 h

Solicitar nova secret, mantendo a anterior válida por 24 h.

**Fluxo principal**

- Usuário solicita rotação, recebe nova secret e migra durante a convivência; depois a anterior expira.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** acesso por qualquer role autenticada; nova secret devolvida somente nessa resposta. Protocolo sujeito à revisão de Sofia.

**Erros previstos**

- Autenticação inválida, cadastro inexistente ou falha ao confirmar. **Proposta de implementação (não discutida na reunião):** recusar nova rotação durante a janela ativa para preservar as 24 h completas; erro no FDD.

**Prioridade:** alta.

**Fonte:** [09:21] Sofia, [09:22] Sofia, [09:31] Marcos, [09:37] Sofia, [09:46] Sofia; `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`.

**Contrato:** [FDD-API-05](FDD.md#fdd-api-05-post-apiv1webhooksidrotate-secret).

---

#### PRD-FR-09 Consultar histórico de entregas

Diagnosticar sucesso/falha, conteúdo, resposta e tempo de resposta.

**Fluxo principal**

- Usuário identifica webhook e consulta os resultados de entrega.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** qualquer role autenticada consulta; apresentar ausência de resposta. “Últimos 100” não fixa retenção; detalhamento no FDD.

**Erros previstos**

- Autenticação ou consulta inválidas; cadastro inexistente. Falha de entrega é resultado consultável.

**Prioridade:** alta.

**Fonte:** [09:34] Marcos, [09:37] Sofia, [09:28] Bruno; `src/middlewares/auth.middleware.ts`, `src/modules/orders/order.schemas.ts`.

**Contrato:** [FDD-API-06](FDD.md#fdd-api-06-get-apiv1webhooksiddeliveries).

---

#### PRD-FR-10 Recuperar falhas com retentativas e DLQ

Retentar falhas e preservar esgotamento para diagnóstico e recuperação.

**Fluxo principal**

- Aplicar política limitada; confirmar entrega ou preservar falha recuperável para PRD-FR-11.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** inicial + 5 retentativas, a confirmar: “5 tentativas” conflita com cinco intervalos ([09:17] Larissa, [09:48] Larissa; [RFC-OQ-07](RFC.md#rfc-oq-07-contagem-de-tentativas-a-confirmar)). Duplicatas exigem deduplicação; indisponibilidade permanente não garante recebimento.

**Erros previstos**

- Sem resposta em 10 s, falha de comunicação ou resposta de falha.

**Prioridade:** alta.

**Fonte:** [09:15] Diego, [09:16] Diego, [09:17] Diego, [09:18] Diego, [09:24] Diego, [09:25] Diego, [09:42] Diego.

**Contrato:** [FDD-API-08](FDD.md#fdd-api-08-post-para-a-url-https-do-cliente); política em [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) e [FDD, resiliência](FDD.md#estratégias-de-resiliência).

---

#### PRD-FR-11 Executar replay administrativo auditado

Permitir recuperação manual por administrador, registrando executor.

**Fluxo principal**

- `ADMIN` solicita replay da falha; sistema registra quem executou e aceita reprocessamento.

**Fluxos alternativos e exceções**

- Aceitação não confirma recebimento. **Proposta de implementação (não discutida na reunião):** recusar repetição já confirmada e preservar identidade/conteúdo para deduplicação.

**Erros previstos**

- Autenticação inválida, ausência de `ADMIN`, falha inexistente, já reprocessada ou destino excluído.

**Prioridade:** alta.

**Fonte:** [09:18] Diego, [09:36] Sofia, [09:36] Larissa, [09:25] Diego; `src/middlewares/auth.middleware.ts`.

**Contrato:** [FDD-API-07](FDD.md#fdd-api-07-post-apiv1adminwebhooksdead-letteridreplay).

---

#### PRD-FR-12 Aceitar somente destinos HTTPS

Aceitar somente destinos HTTPS.

**Fluxo principal**

- Usuário informa URL; sistema valida e aceita destino HTTPS.

**Fluxos alternativos e exceções**

- **Proposta de implementação (não discutida na reunião):** aplicar também na edição; proteções adicionais seguem revisão de segurança no RFC.

**Erros previstos**

- URL HTTP ou inválida bloqueia configuração com erro de validação.

**Prioridade:** alta.

**Fonte:** [09:23] Sofia, [09:33] Bruno; `src/modules/orders/order.schemas.ts` (padrão Zod).

**Contrato:** [FDD-API-01](FDD.md#fdd-api-01-post-apiv1webhooks), [FDD-API-03](FDD.md#fdd-api-03-patch-apiv1webhooksid); questões em [RFC-OQ-06](RFC.md#rfc-oq-06-revisão-de-segurança).

---

## Requisitos não funcionais

### Performance

- **PRD-NFR-01:** Meta: chegada ao sistema do cliente em menos de 10 s após mudança confirmada, conforme PRD-OBJ-01 ([09:00] Marcos, [09:02] Marcos). **Proposta de implementação (não discutida na reunião):** indicador interno até início do envio, sujeito a backlog e polling; resposta é posterior à chegada e medida separadamente ([09:07] Bruno, [09:09] Diego, [09:42] Diego; [FDD](FDD.md#observabilidade)).

### Disponibilidade e confiabilidade

- **PRD-NFR-02:** Falha HTTP do destinatário não desfaz mudança confirmada; entregas têm tentativas limitadas e preservação na DLQ ao esgotar, com duplicatas possíveis ([09:04] Bruno, [09:06] Diego, [09:15] Diego, [09:18] Diego, [09:24] Diego). Sem SLO de disponibilidade definido.

### Segurança e autorização

- **PRD-NFR-03:** Configuração exige autenticação com JWT do OMS; qualquer role autenticada pode usar o CRUD nesta fase. Não há associação automática entre usuário e cliente ([09:32] Marcos, [09:32] Larissa, [09:37] Sofia; `src/middlewares/auth.middleware.ts`, `prisma/schema.prisma`). Extensão à rotação e ao histórico é **Proposta de implementação (não discutida na reunião)**, coerente com o acesso de gestão proposto no FDD.
- **PRD-NFR-04:** Verificação de origem e integridade por assinatura HMAC-SHA256 do corpo com secret por endpoint e rotação com 24 h de convivência, além de HTTPS ([09:19] Sofia, [09:22] Sofia, [09:23] Sofia). Protocolo no [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md), sujeito à revisão de Sofia ([09:46] Sofia); isso não implica proteção completa contra replay.
- **PRD-NFR-05:** Payload limitado a 64 KB; excesso produz erro, sem truncar o conteúdo enviado ([09:23] Sofia, [09:24] Diego, [09:24] Larissa). Aplicação e unidade estão na proposta do [FDD](FDD.md#estratégias-de-resiliência).

### Observabilidade

- **PRD-NFR-06:** Pino e histórico para diagnóstico ([09:29] Bruno, [09:34] Marcos; `src/shared/logger/index.ts`). **Proposta de implementação (não discutida na reunião):** evidências correlacionadas de latência/falhas, sem plataforma nova; [FDD](FDD.md#observabilidade).

### Integridade de dados

- **PRD-NFR-07:** Mudança elegível e notificação confirmadas juntas; falha desfaz transição e efeitos aplicáveis ([09:06] Diego, [09:40] Bruno; `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`).
- **PRD-NFR-08:** Conteúdo fiel à transição ([09:52] Larissa); ordem por pedido com um único worker, sem ordem global ([09:12] Diego, [09:13] Larissa). **Proposta de implementação (não discutida na reunião):** retries fazem notificações seguintes do mesmo pedido/endpoint esperar até entrega ou falha esgotada; replay posterior pode chegar fora de ordem (PRD-RISK-07; [FDD](FDD.md#fdd-flow-02-processamento-pelo-worker)).

### Compatibilidade

- **PRD-NFR-09:** Reutilizar stack, MySQL e padrões do OMS ([09:11] Diego, [09:30] Larissa; `package.json`, `prisma/schema.prisma`); gestão em `/api/v1` (`src/app.ts`, `src/routes/index.ts`), outbound JSON ([09:43] Diego). Compatibilidade no [FDD](FDD.md#dependências-e-compatibilidade).

### Compliance/auditoria

- **PRD-NFR-10:** Replay exige `ADMIN` e registro de quem o executou para auditoria ([09:36] Sofia, [09:36] Larissa). Sem norma legal, certificação ou retenção definidas ([09:34] Marcos).

---

## Arquitetura e abordagem

**PRD-ARCH-01:** Gestão no OMS existente e entrega assíncrona aos clientes ([09:27] Bruno, [09:40] Bruno, [09:06] Diego, [09:11] Diego). Polling de 2 s aceito, sujeito a backlog ([09:09] Diego, [09:10] Marcos, [09:07] Bruno); HTTPS, assinatura e deduplicação protegem a integração ([09:23] Sofia, [09:20] Sofia, [09:25] Diego). Abordagem no [RFC](RFC.md#proposta-técnica); construção no [FDD](FDD.md).

---

## Decisões e trade-offs principais

#### PRD-DEC-01 Registrar a mudança e a notificação juntas

**Justificativa:** evitar que uma mudança elegível seja confirmada sem registro de notificação, aproveitando o MySQL existente ([09:06] Diego, [09:07] Diego).  
**Trade-off:** uma falha de persistência da notificação também impede a mudança de status ([09:40] Bruno). [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md).

#### PRD-DEC-02 Entregar por processo separado

**Justificativa:** independência da API, com espera aceita ([09:11] Diego, [09:10] Marcos).  
**Trade-off:** espera de polling e escala inicial limitada ([09:10] Larissa, [09:12] Diego, [09:13] Diego). [ADR-002](adrs/ADR-002-worker-separado-com-polling.md).

#### PRD-DEC-03 Limitar retentativas e preservar falhas

**Justificativa:** absorver indisponibilidade sem retry indefinido ([09:15] Diego, [09:16] Diego).  
**Trade-off:** esgotamento exige recuperação administrativa ([09:18] Diego). [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md).

#### PRD-DEC-04 Proteger origem e integridade por endpoint

**Justificativa:** verificar origem e limitar vazamentos ([09:19] Sofia, [09:21] Sofia).  
**Trade-off:** o integrador precisa verificar a assinatura e migrar secrets na rotação ([09:20] Sofia, [09:21] Sofia). [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md).

#### PRD-DEC-05 Admitir duplicatas identificáveis

**Justificativa:** viabilizar recuperação sem a complexidade de coordenação bilateral da alternativa exactly-once ([09:24] Diego, [09:25] Diego).  
**Trade-off:** consumidores deduplicam; portal deve orientar ([09:25] Sofia, [09:26] Marcos). [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).

#### PRD-DEC-06 Reutilizar padrões do OMS

**Justificativa:** manter a solução na organização e nas ferramentas que o time já usa ([09:27] Bruno, [09:29] Bruno, [09:30] Larissa).  
**Trade-off:** adaptar entrega aos limites existentes (`src/middlewares/error.middleware.ts`, `src/config/database.ts`; [09:30] Bruno). [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

#### PRD-DEC-07 Preservar o conteúdo do instante da transição

**Justificativa:** preservar significado da transição ([09:52] Larissa).  
**Trade-off:** retratar estado histórico, diferente do atual ([09:51] Bruno, [09:52] Larissa). [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md).

---

## Dependências

- **PRD-DEP-01 (organizacional):** Sofia revisará HMAC e geração de secret antes do deploy; reservar pelo menos dois dias úteis, dentro da estimativa consolidada de três sprints com revisão incluída ([09:46] Sofia, [09:46] Larissa, [09:47] Larissa).
- **PRD-DEP-02 (organizacional):** Marcos publicará integração via API e deduplicação no portal ([09:26] Marcos, [09:40] Marcos).
- **PRD-DEP-03 (organizacional):** Marcos confirmará novembro com Atlas e atualizará clientes, sem data civil fechada ([09:45] Marcos, [09:47] Marcos, [09:49] Marcos).
- **PRD-DEP-04 (técnica):** MySQL e autenticação JWT existentes, com regras de pedidos preservadas, sustentam a integração ([09:07] Diego, [09:32] Marcos, [09:36] Larissa; `prisma/schema.prisma`, `src/middlewares/auth.middleware.ts`, `src/modules/orders/order.service.ts`).
- **PRD-DEP-05 (externa):** Sistemas dos clientes precisam disponibilizar destinos HTTPS, verificar assinatura e deduplicar para integrar ([09:20] Sofia, [09:23] Sofia, [09:25] Diego). Software não informado.
- **PRD-DEP-06 (organizacional, futura):** Painel futuro depende do frontend, sem bloquear API ([09:40] Larissa).
- **PRD-DEP-07 (organizacional):** Larissa organizará a revisão do desenho com Bruno e Diego antes de codificar ([09:50] Larissa). Três sprints estimadas não asseguram data comercial ([09:46] Larissa, [09:47] Larissa).

---

## Riscos e mitigação

Probabilidades avaliadas pela autora, sem frequência medida. Contingências propostas não criam compromissos comerciais.

### PRD-RISK-01 Atraso e perda da Atlas

**Probabilidade:** média (avaliação da autora): novembro ainda depende da confirmação e revisão ([09:45] Marcos, [09:47] Larissa, [09:47] Marcos).

**Impacto:** atraso até fim do trimestre ameaça retenção da Atlas ([09:00] Marcos).

**Mitigação:**

- Marcos confirmar o prazo com a Atlas e manter os clientes informados ([09:47] Marcos, [09:49] Marcos).
- Reservar a revisão de Sofia antes do deploy dentro da estimativa ([09:46] Sofia, [09:47] Larissa).

**Plano de contingência:** **Proposta de implementação (não discutida na reunião):** reavaliar e comunicar previsão (PRD-DEP-03).

### PRD-RISK-02 Destinatário indisponível por longo período

**Probabilidade:** alta (avaliação da autora): manutenção já durou duas horas ([09:16] Diego).

**Impacto:** atraso ou esgotamento sem recebimento, exigindo recuperação posterior ([09:15] Diego, [09:18] Diego).

**Mitigação:**

- **Proposta de implementação (não discutida na reunião):** janela nominal de 14h36, a confirmar ([09:17] Diego, [09:17] Larissa, [09:48] Larissa; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)).
- Preservação de conteúdo e motivo na DLQ para diagnóstico ([09:18] Diego).

**Plano de contingência:** **Proposta de implementação (não discutida na reunião):** replay por `ADMIN`, auditado, após recuperação ([09:18] Diego, [09:36] Sofia).

### PRD-RISK-03 Consumidor não deduplica eventos

**Probabilidade:** alta exposição (avaliação da autora): duplicatas admitidas ([09:24] Diego, [09:25] Diego).

**Impacto:** efeitos duplicados, inferidos da responsabilidade do consumidor ([09:25] Sofia).

**Mitigação:**

- Fornecer identidade única e estável para reconhecer o mesmo evento ([09:25] Diego).
- Destacar a deduplicação no portal de desenvolvedor ([09:26] Marcos).

**Plano de contingência:** **Proposta de implementação (não discutida na reunião):** integrador diagnostica e corrige deduplicação antes do replay ([09:34] Marcos, [09:25] Diego).

### PRD-RISK-04 Vazamento de secret no cliente

**Probabilidade:** média (avaliação da autora): já ocorreu vazamento em log de aplicação de cliente ([09:22] Diego).

**Impacto:** origem comprometida, inferência de [09:19] Sofia e [09:21] Sofia.

**Mitigação:**

- Secret individual e possibilidade de rotação ([09:21] Sofia).
- Revisão de HMAC e geração de secret por Sofia antes do deploy ([09:46] Sofia).

**Plano de contingência:** **Proposta de implementação (não discutida na reunião):** avaliar incidente com Sofia; rotação mantém anterior por 24 h, sem revogação emergencial decidida ([09:21] Sofia).

### PRD-RISK-05 Rajada sobrecarrega o sistema integrado

**Probabilidade:** média (avaliação da autora): rajadas levantadas; rate limiting adiado ([09:38] Diego, [09:39] Diego, [09:39] Larissa).

**Impacto:** sobrecarga e falhas, inferência de [09:38] Diego.

**Mitigação:**

- Observar antes de decidir controle de taxa ([09:39] Diego, [09:39] Larissa).
- **Proposta de implementação (não discutida na reunião):** usar diagnóstico de sobrecarga pelo histórico de [09:34] Marcos.

**Plano de contingência:** revisar [RFC-OQ-01](RFC.md#rfc-oq-01-rate-limiting-de-saída), conforme [09:39] Larissa.

### PRD-RISK-06 Acúmulo compromete a rapidez das notificações

**Probabilidade:** média (avaliação da autora): acúmulo e lentidão possíveis; arquivamento adiado ([09:07] Bruno, [09:08] Diego, [09:42] Diego).

**Impacto:** chegada ao cliente acima da meta de 10 s e integração lenta ([09:02] Marcos).

**Mitigação:**

- Leitura seletiva e índices no FDD ([09:08] Diego; [FDD](FDD.md#fluxos-detalhados)).
- **Proposta de implementação (não discutida na reunião):** acompanhar latência e pendências no [FDD, observabilidade](FDD.md#observabilidade), sem novo SLO.

**Plano de contingência:** **Proposta de implementação (não discutida na reunião):** reavaliar escala e arquivamento: [RFC-OQ-03](RFC.md#rfc-oq-03-escala-e-ordenação-com-retry) e [RFC-OQ-04](RFC.md#rfc-oq-04-arquivamento-e-retenção), adiados ([09:13] Diego, [09:08] Diego).

### PRD-RISK-07 Espera por recuperação e replay fora de sequência

**Probabilidade:** média (avaliação da autora): retentativas longas e recuperação manual ([09:17] Diego, [09:18] Diego).

**Impacto:** **Proposta de implementação (não discutida na reunião):** falha persistente segura notificações seguintes do mesmo pedido/endpoint até esgotar; após isso avançam e replay posterior pode entregar a antiga fora de ordem ([ADR-002](adrs/ADR-002-worker-separado-com-polling.md)).

**Mitigação:**

- **Proposta de implementação (não discutida na reunião):** preservar ordem durante retries, limitada a um worker ([09:12] Diego, [09:13] Larissa).
- Preservar contexto da transição para diagnóstico ([09:43] Diego, [09:52] Larissa).

**Plano de contingência:** revisar ordenação dos replays com a evolução de escala ([09:13] Diego; [RFC-OQ-03](RFC.md#rfc-oq-03-escala-e-ordenação-com-retry)).

---

## Critérios de aceitação

Checklist futuro; execução no [FDD](FDD.md#critérios-de-aceite-técnicos).

- [ ] **PRD-AC-01:** Usuário autenticado cadastra destino com cliente, URL HTTPS e status selecionados; recebe secret gerada pela plataforma. **Proposta de implementação (não discutida na reunião):** consultas posteriores não reapresentam a secret, conforme PRD-FR-01 ([09:31] Marcos, [09:32] Larissa, [09:23] Sofia).
- [ ] **PRD-AC-02:** Pela API, listar cadastros de um cliente, editar e remover são verificáveis. Ativar/desativar segue a **Proposta de implementação (não discutida na reunião)** de PRD-FR-05; URL HTTP é recusada no cadastro e na edição ([09:33] Bruno, [09:21] Bruno, [09:23] Sofia).
- [ ] **PRD-AC-03:** Mudança válida notifica somente interessados; falha ao registrar notificação impede confirmar a mudança, preservando regras de pedidos ([09:34] Bruno, [09:40] Bruno; `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`).
- [ ] **PRD-AC-04:** Conferir chegada ao cliente em menos de 10 s após mudança confirmada ([09:00] Marcos, [09:02] Marcos). **Proposta de implementação (não discutida na reunião):** cenário controlado usa destinatário simulado e compara chegada com indicador interno; OMS não observa chegada em produção. Avaliar backlog e resposta separadamente ([09:07] Bruno, [09:09] Diego; [FDD](FDD.md#critérios-de-aceite-técnicos)).
- [ ] **PRD-AC-05:** Conteúdo recebido reflete a transição original mesmo depois de outra alteração do pedido; o consumidor consegue verificar a assinatura e reconhecer duplicatas pela identidade do evento ([09:52] Larissa, [09:20] Sofia, [09:25] Diego).
- [ ] **PRD-AC-06:** Rotação entrega nova secret e mantém a anterior válida durante 24 h; após a janela, a anterior não valida novos envios. O protocolo proposto deve passar pela revisão de Sofia ([09:21] Sofia, [09:46] Sofia; [FDD-API-05](FDD.md#fdd-api-05-post-apiv1webhooksidrotate-secret)).
- [ ] **PRD-AC-07:** Consulta de histórico mostra sucesso/falha, payload, resposta quando existente e tempo de resposta. Ausência de resposta e granularidade por tentativa seguem a proposta PRD-FR-09 ([09:34] Marcos).
- [ ] **PRD-AC-08:** Falhas esgotadas ficam recuperáveis com conteúdo, motivo e instante ([09:18] Diego); ausência de resposta em 10 s é falha ([09:42] Diego). **Proposta de implementação (não discutida na reunião):** inicial + 5 retentativas, janela nominal de 14h36, a confirmar diante da inconsistência literal ([09:16] Diego, [09:17] Diego, [09:17] Larissa, [09:48] Larissa; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)).
- [ ] **PRD-AC-09:** Usuário sem autenticação não acessa a gestão; autenticado sem `ADMIN` não confirma replay; `ADMIN` recoloca o evento pendente e há registro de quem executou ([09:32] Marcos, [09:37] Sofia, [09:36] Sofia, [09:36] Larissa; `src/middlewares/auth.middleware.ts`).
- [ ] **PRD-AC-10:** Payload que excede 64 KB não é enviado truncado e produz erro; unidade e preservação da falha seguem a proposta do FDD ([09:23] Sofia, [09:24] Larissa).
- [ ] **PRD-AC-11:** Resultados da regressão de pedidos e validação ponta a ponta estão disponíveis; revisão de segurança de Sofia ocorre antes do deploy, com pelo menos dois dias úteis reservados ([09:46] Larissa, [09:46] Sofia; `tests/orders.test.ts`).
- [ ] **PRD-AC-12:** Documentação do portal orienta integração via API e deduplicação por identidade do evento; Marcos confirma o prazo com a Atlas e atualiza os clientes ([09:26] Marcos, [09:40] Marcos, [09:47] Marcos, [09:49] Marcos).

---

## Estratégia de testes e validação

**Proposta de implementação (não discutida na reunião):** os testes futuros abaixo concretizam ponta a ponta ([09:46] Larissa) com Vitest/Supertest (`tests/orders.test.ts`, `package.json`), sem meta numérica adicional.

- **PRD-TEST-01 (unitários):** Validar seleção, URLs, tamanho, identidade, assinatura e rotação com relógio controlado ([09:33] Marcos, [09:23] Sofia, [09:24] Larissa, [09:25] Diego, [09:21] Sofia, [09:17] Diego); proposta detalhada no [FDD](FDD.md#critérios-de-aceite-técnicos).
- **PRD-TEST-02 (integração e ponta a ponta):** Validar gestão, filtragem, confirmação indivisível, diagnóstico e recuperação com destinatário simulado ([09:34] Bruno, [09:40] Bruno, [09:34] Marcos, [09:18] Diego, [09:46] Larissa; `tests/orders.test.ts`). Simulação é proposta interna, sem piloto comercial. Procedimentos no [FDD](FDD.md#critérios-de-aceite-técnicos).
- **PRD-TEST-03 (segurança):** Conferir autenticação, `ADMIN`, auditoria e integridade; revisão de HMAC e geração de secret por Sofia ([09:32] Marcos, [09:36] Sofia, [09:20] Sofia, [09:46] Sofia; `src/middlewares/auth.middleware.ts`). **Proposta de implementação (não discutida na reunião):** conferir exposição de secrets em consultas/logs pelo incidente de [09:22] Diego; procedimentos no [FDD](FDD.md#critérios-de-aceite-técnicos).
- **PRD-TEST-04 (regressão):** Preservar criação, transições, estoque, totais e erros de `tests/orders.test.ts`, incluindo confirmação indivisível ([09:40] Bruno, [09:46] Larissa); proposta no [FDD](FDD.md#critérios-de-aceite-técnicos).
- **PRD-TEST-05 (latência e resiliência):** Comparar chegada ao cliente e indicador interno; avaliar lentidão, indisponibilidade, duplicatas e esgotamento, sem volume de carga obrigatório ([09:02] Marcos, [09:07] Bruno, [09:16] Diego, [09:24] Diego, [09:42] Diego); procedimento proposto no [FDD-AC-17](FDD.md#critérios-de-aceite-técnicos).
- **PRD-TEST-06 (validação da entrega):** Consolidar PRD-AC-01 a PRD-AC-12, revisão do desenho e segurança ([09:50] Larissa, [09:46] Sofia). **Proposta de implementação (não discutida na reunião):** conciliar PRD-OBJ-03 com evidências do [FDD](FDD.md#observabilidade) e acompanhar disponibilidade para os três solicitantes, sem exigir piloto comercial.

---

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
