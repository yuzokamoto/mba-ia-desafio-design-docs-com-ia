# ADR-004: HMAC-SHA256 com secret por endpoint

## Status

Aceito. Data: 2026-10-25. Decidido em `[09:22] Sofia`; confirmado em `[09:48] Larissa`.
Relacionados: [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md).

Responsável: Larissa (Tech Lead). A data acima é de produção deste documento, não da reunião.

## Contexto

Os eventos outbound levam dados de pedidos para fora da infraestrutura. O destinatário precisa verificar a origem e a integridade do corpo recebido; Sofia definiu HMAC-SHA256 com assinatura no header `X-Signature` (`[09:19] Sofia`, `[09:20] Sofia`, `[09:22] Sofia`).

Já houve vazamento de secret em log de cliente. A solução precisa limitar o alcance de um vazamento e permitir troca com tempo de migração para o consumidor; a reunião fechou secret por endpoint e convivência da anterior por 24 horas (`[09:21] Sofia`, `[09:22] Diego`). A revisão de HMAC e geração de secret será feita por Sofia antes do deploy (`[09:46] Sofia`).

## Fatores de decisão

- O consumidor precisa validar origem e integridade do payload (`[09:19] Sofia`).
- HMAC-SHA256 tem suporte nos clientes segundo a avaliação de Sofia (`[09:20] Sofia`).
- A secret exclusiva por endpoint limita o alcance de um vazamento (`[09:21] Sofia`).
- A rotação precisa manter a secret anterior válida por 24 horas para migração (`[09:21] Sofia`).
- O incidente anterior reforça a necessidade de rotação e cuidado com logs (`[09:22] Diego`).
- O mecanismo de segurança precisa passar pela revisão de Sofia antes do deploy (`[09:46] Sofia`).

## Alternativas Consideradas

### Opção 1: HMAC-SHA256 e secret por endpoint (escolhida)

Assinar o corpo com secret compartilhada exclusiva do endpoint, rotacionável com janela de 24 horas (`[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Sofia`).

- Prós: Permite verificar origem e integridade e limita o alcance de uma secret vazada (`[09:19] Sofia`, `[09:21] Sofia`).
- Contras: O consumidor precisa verificar a assinatura e migrar sua secret durante a rotação (`[09:20] Sofia`, `[09:21] Sofia`).

### Opção 2: Secret global da plataforma (ADR-004-ALT-01)

Explicitamente descartada em `[09:21] Sofia`: o comprometimento de todos os clientes por um vazamento supera a conveniência de uma secret única.

- Prós (avaliação do autor, não discutida na reunião): Exige gerir apenas uma credencial.
- Contras: Um vazamento comprometeria todos os endpoints (`[09:21] Sofia`).

## Decisão

ADR-004-DEC-01: Assinar o corpo do request com HMAC-SHA256 e enviar a assinatura em `X-Signature`, usando secret exclusiva por endpoint. A plataforma gera e devolve a secret na criação; o cliente pode solicitar rotação pela API. Na rotação, a anterior permanece válida em paralelo por 24 horas e depois expira (`[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Sofia`, `[09:31] Marcos`).

ADR-004-PROP-01: **Proposta de implementação (não discutida na reunião):** durante as 24 horas contadas da rotação, enviar no mesmo `X-Signature` duas assinaturas separadas por vírgula, no formato `sha256=<hex com a nova>, sha256=<hex com a antiga>`; depois, enviar somente a nova. Isso permite validar com qualquer secret durante a migração outbound; enquanto `previousSecretExpiresAt > agora`, recusar nova rotação com 409 `WEBHOOK_SECRET_ROTATION_IN_PROGRESS`, preservando as 24 h completas com um único `previousSecret`. Revogação emergencial imediata não foi discutida e fica para a revisão de Sofia ([09:46] Sofia). Assinar os bytes exatos do snapshot JSON serializado uma única vez e armazenado como texto, com digest hexadecimal minúsculo, evita divergência entre corpo e assinatura. O formato e a dupla assinatura precisam ser validados na revisão da Sofia, pois a reunião fechou a janela, mas não o protocolo de convivência (`[09:21] Sofia`, `[09:22] Sofia`, `[09:46] Sofia`; [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md), [FDD](../FDD.md#contratos-públicos)).

## Consequências

### Positivas

- ADR-004-CONS-01: O consumidor dispõe de verificação da origem e integridade do corpo (`[09:19] Sofia`, `[09:20] Sofia`).
- ADR-004-CONS-02: O vazamento de uma secret fica limitado ao endpoint correspondente e há janela de migração na rotação (`[09:21] Sofia`, `[09:22] Sofia`).

### Negativas

- ADR-004-CONS-03: A rotação exige coordenação com o consumidor e, na proposta ADR-004-PROP-01, suporte temporário a duas assinaturas, sujeito à revisão de Sofia (`[09:21] Sofia`, `[09:46] Sofia`).
- ADR-004-CONS-04: `X-Timestamp` informa o instante do envio para permitir ao cliente detectar replay attack se quiser (`[09:44] Diego`). A assinatura decidida cobre apenas o corpo (`[09:22] Sofia`), portanto o timestamp não é autenticado e a detecção de replay que permite é limitada. Avaliar essa limitação, por exemplo incluindo o timestamp no conteúdo assinado, cabe à revisão de Sofia (`[09:46] Sofia`); essa inclusão não foi decidida aqui.
- ADR-004-CONS-05: O Pino existente não inclui secrets de webhook nem `X-Signature` na redação; **Proposta de implementação (não discutida na reunião):** ampliar a redação para secrets do módulo e nunca registrar a assinatura, em resposta ao risco já observado (`src/shared/logger/index.ts`; `[09:22] Diego`).

### Trade-off

- ADR-004-CONS-06: Aceitamos gerir secrets por endpoint e sua migração em troca de verificar o corpo e limitar o alcance de vazamentos (`[09:19] Sofia`, `[09:21] Sofia`, `[09:22] Sofia`).

## Referências

- Transcrição: `[09:19] Sofia`; `[09:20] Sofia`; `[09:21] Sofia`; `[09:22] Sofia`; `[09:22] Diego`; `[09:31] Marcos`; `[09:44] Diego`; `[09:46] Sofia`; `[09:48] Larissa`.
- Código: `src/shared/logger/index.ts`, Pino e lista atual de redação sem os novos dados sensíveis.
- Documentos: [RFC, proposta técnica](../RFC.md#proposta-técnica), [RFC, impacto e riscos](../RFC.md#impacto-e-riscos), [FDD, contratos públicos](../FDD.md#contratos-públicos), [FDD, riscos e mitigação](../FDD.md#riscos-e-mitigação).

Redigido com apoio de IA a partir de `TRANSCRICAO.md` e do código; rastreabilidade em `docs/TRACKER.md`.
