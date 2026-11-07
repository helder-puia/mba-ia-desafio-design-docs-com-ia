# Tracker de Rastreabilidade

Este documento mapeia cada item registrado no pacote de design docs para a sua origem na transcrição da reunião ou no código do repositório base. Ele existe para impedir que a documentação derive para conteúdo inventado: se um item não tem linha aqui, ele não tem origem identificável e deve ser removido ou reclassificado como proposta.

Regras aplicadas:

- `Fonte = TRANSCRICAO`: a localização indica timestamp e falante no formato `[hh:mm] Nome`, sempre no instante exato em que o assunto foi tratado.
- `Fonte = CODIGO`: a localização indica o caminho do arquivo no repositório base, relativo à raiz.
- Itens marcados como proposta de implementação, sem origem direta na reunião, estão listados na seção final "Itens derivados" e não aparecem como decisão nos documentos.
- Caminhos de arquivos novos aparecem apenas como destino de implementação no FDD, nunca como origem de rastreabilidade.

## Tabela

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST com URL, customer e lista de status, com secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook por PATCH alterando URL, status e estado ativo | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook por DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem dos webhooks de um customer por GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status escolhido pelo webhook, aplicado na inserção | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Registro do evento na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega HTTP assíncrona ao endpoint do cliente com latência de tempo real | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry com backoff exponencial em cinco tentativas e movimentação para DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de evento da DLQ por endpoint administrativo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico de entregas com resultado, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret por endpoint, com secret anterior válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 de cada entrega com secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Identificador único por evento para deduplicação no cliente | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos, que é a definição de tempo real dos clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Transação de mudança de status não pode fazer chamada HTTP externa | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once, com duplicidade esperada e deduplicação pelo cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Nenhum evento é descartado silenciosamente: entregue ou em DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https, com http recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, com rotação e grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB por payload, com erro e sem truncamento | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Replay de DLQ exige role ADMIN e gera registro de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Logs estruturados com o logger Pino já existente, sem logger novo | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Nenhum endpoint existente muda de contrato ou comportamento | CODIGO | src/routes/index.ts |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado, com entry-point e script próprios | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Módulo segue a estrutura controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | p95 entre commit e primeira entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Os três clientes demandantes ativos em webhook no fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | 100% dos eventos com estado final conhecido, sem descarte | TRANSCRICAO | [09:15] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Derivado: meta proposta de 95% de sucesso na primeira tentativa, a partir do requisito de nenhum evento descartado | TRANSCRICAO | [09:15] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Derivado: queda mensurável do polling dos clientes após 30 dias, sem baseline numérico na reunião | TRANSCRICAO | [09:00] Marcos |
| PRD-OOS-01 | docs/PRD.md | Escopo (fora) | Notificação por e-mail em caso de falha fica para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Escopo (fora) | Dashboard visual do cliente está fora desta fase | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Escopo (fora) | Rate limiting de envio não entra agora, será observado | TRANSCRICAO | [09:38] Diego |
| PRD-OOS-04 | docs/PRD.md | Escopo (fora) | Webhooks inbound, do cliente para a plataforma, estão descartados | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Escopo (fora) | Ordenação global entre pedidos diferentes não é garantida | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Escopo (fora) | Escala para múltiplos workers fica como problema futuro | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-07 | docs/PRD.md | Escopo (fora) | Arquivamento de eventos entregues está fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-08 | docs/PRD.md | Escopo (fora) | Reprocessamento de DLQ é manual, sem automação | TRANSCRICAO | [09:18] Diego |
| PRD-OOS-09 | docs/PRD.md | Escopo (fora) | Redis Streams e broker dedicado foram descartados | TRANSCRICAO | [09:07] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox no MySQL dentro da transação de mudança de status | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker separado em polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão | Retry 1m/5m/30m/2h/12h com DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-04 | docs/PRD.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação em 24 horas | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão | At-least-once com X-Event-Id para deduplicação | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Decisão | Payload renderizado na inserção, refletindo o estado da transição | TRANSCRICAO | [09:52] Larissa |
| PRD-DEC-08 | docs/PRD.md | Decisão | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-DEP-01 | docs/PRD.md | Dependência | Time com capacidade para três sprints de trabalho | TRANSCRICAO | [09:46] Larissa |
| PRD-DEP-02 | docs/PRD.md | Dependência | Dois dias úteis de revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Endpoint HTTPS operacional do lado do cliente | TRANSCRICAO | [09:23] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Documentação de integração no portal do desenvolvedor pelo PM | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Banco MySQL e Prisma existentes, sem infraestrutura nova | CODIGO | prisma/schema.prisma |
| PRD-DEP-06 | docs/PRD.md | Dependência | Autenticação usa JWT de usuário do OMS, não JWT de cliente | TRANSCRICAO | [09:32] Larissa |
| PRD-DEP-07 | docs/PRD.md | Dependência | Prazo esperado pela Atlas no fim de novembro, contra três sprints de estimativa | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Prazo comercial não caber em três sprints, com risco de churn da Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Escrita na outbox elevar a latência da mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RISK-03 | docs/PRD.md | Risco | Worker parado sem detecção e fila crescendo | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Vazamento de secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Cliente processar duplicado por não deduplicar | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-06 | docs/PRD.md | Risco | Crescimento indefinido da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-07 | docs/PRD.md | Risco | Cliente sem capacidade de validar HMAC | TRANSCRICAO | [09:20] Bruno |
| PRD-AC-01 | docs/PRD.md | Critério de Aceite | Cadastro de webhook com URL https e lista de status, com secret na resposta | TRANSCRICAO | [09:33] Bruno |
| PRD-AC-02 | docs/PRD.md | Critério de Aceite | Cadastro com URL http recusado com erro claro e nada persistido | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-03 | docs/PRD.md | Critério de Aceite | Edição de URL, status e estado ativo sem trocar a secret | TRANSCRICAO | [09:33] Bruno |
| PRD-AC-04 | docs/PRD.md | Critério de Aceite | Remoção de webhook faz o cliente deixar de receber eventos | TRANSCRICAO | [09:33] Bruno |
| PRD-AC-05 | docs/PRD.md | Critério de Aceite | Listagem de webhooks de um customer com paginação | TRANSCRICAO | [09:33] Bruno |
| PRD-AC-06 | docs/PRD.md | Critério de Aceite | Mudança de status gera evento para todos os webhooks ativos que assinam aquele status | TRANSCRICAO | [09:33] Marcos |
| PRD-AC-07 | docs/PRD.md | Critério de Aceite | Status sem webhook interessado não gera linha na fila | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-08 | docs/PRD.md | Critério de Aceite | Evento chega ao endpoint do cliente em menos de 10 segundos no cenário nominal | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-09 | docs/PRD.md | Critério de Aceite | Falha de entrega gera cinco tentativas no cronograma 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-AC-10 | docs/PRD.md | Critério de Aceite | Após cinco falhas o evento aparece em DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-AC-11 | docs/PRD.md | Critério de Aceite | Usuário ADMIN reprocessa evento da DLQ e ele volta a ser entregue | TRANSCRICAO | [09:36] Larissa |
| PRD-AC-12 | docs/PRD.md | Critério de Aceite | Usuário sem role ADMIN recebe 403 ao tentar reprocessar | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-13 | docs/PRD.md | Critério de Aceite | Cliente consulta as últimas 100 entregas com resultado, payload, resposta e duração | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-14 | docs/PRD.md | Critério de Aceite | Rotação de secret mantém a anterior válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-15 | docs/PRD.md | Critério de Aceite | Toda entrega traz HMAC-SHA256 verificável com a secret do endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-16 | docs/PRD.md | Critério de Aceite | Duas entregas do mesmo evento carregam o mesmo X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-AC-17 | docs/PRD.md | Critério de Aceite | Nenhuma mudança de status fica sem o evento correspondente quando há webhook interessado | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-18 | docs/PRD.md | Critério de Aceite | Nenhum endpoint existente muda de contrato ou de comportamento | TRANSCRICAO | [09:33] Bruno |
| PRD-TEST-01..02 | docs/PRD.md | Estratégia de Teste | Testes unitários de backoff e de HMAC com relógio e vetor controlados | CODIGO | tests/orders.test.ts |
| PRD-TEST-03..05 | docs/PRD.md | Estratégia de Teste | Testes de integração com supertest no padrão das factories existentes | CODIGO | tests/helpers/factories.ts |
| PRD-TEST-06..10 | docs/PRD.md | Estratégia de Teste | Fake server para retry e DLQ, teste de segurança, contrato, carga e piloto | TRANSCRICAO | [09:46] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Disparo síncrono dentro de changeStatus, descartado por acoplamento | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams ou broker dedicado, descartado por exigir infraestrutura nova | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco, descartado por não notificar processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Retry indefinido, descartado por deixar evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Secret global da plataforma, descartada por ampliar o impacto de vazamento | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Alternativa | Exactly-once, descartado por exigir coordenação entre as duas pontas | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente, a observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Notificação proativa de falha ao cliente por e-mail, adiada | TRANSCRICAO | [09:37] Marcos |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Ordenação global e escala para múltiplos workers, adiadas | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em Aberto | Arquivamento de eventos entregues, fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| RFC-QA-05 | docs/RFC.md | Questão em Aberto | Endurecimento de autorização do CRUD, deixado para depois | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-06 | docs/RFC.md | Questão em Aberto | Prazo de fim de novembro contra estimativa de três sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-QA-07 | docs/RFC.md | Questão em Aberto | Ordenação por pedido não sobrevive a retry pendente, sem decisão sobre head-of-line | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-01 | docs/RFC.md | Risco | Latência de changeStatus degradar pela escrita adicional | TRANSCRICAO | [09:04] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Worker parado passar despercebido, com fila e latência crescendo | TRANSCRICAO | [09:11] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Secret exposta em log da plataforma ou do cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Estimativa de três sprints não caber no prazo comercial | TRANSCRICAO | [09:46] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção do evento na outbox dentro da transação de mudança de status | TRANSCRICAO | [09:06] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Processamento pelo worker com polling, marcação e envio HTTP | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff e recuperação de evento travado em PROCESSING | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Movimentação para DLQ e replay manual administrativo | TRANSCRICAO | [09:18] Diego |
| FDD-OT-01 | docs/FDD.md | Objetivo Técnico | Invariante de que status alterado sempre tem evento registrado | TRANSCRICAO | [09:40] Bruno |
| FDD-OT-02 | docs/FDD.md | Objetivo Técnico | p95 de entrega abaixo de 10 segundos com polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| FDD-OT-03 | docs/FDD.md | Objetivo Técnico | Entrega isolada do ciclo de vida da API, em processo próprio | TRANSCRICAO | [09:11] Diego |
| FDD-OT-04 | docs/FDD.md | Objetivo Técnico | Todo envio carrega HMAC verificável com a secret do endpoint | TRANSCRICAO | [09:20] Sofia |
| FDD-OT-05 | docs/FDD.md | Objetivo Técnico | Todo evento termina entregue ou em DLQ, nunca descartado | TRANSCRICAO | [09:15] Diego |
| FDD-OT-06 | docs/FDD.md | Objetivo Técnico | X-Event-Id constante entre tentativas do mesmo evento | TRANSCRICAO | [09:25] Diego |
| FDD-OT-07 | docs/FDD.md | Objetivo Técnico | Nenhuma linha de outbox para status que ninguém assina | TRANSCRICAO | [09:34] Bruno |
| FDD-OT-08 | docs/FDD.md | Objetivo Técnico | Ordem por order_id preservada com worker único | TRANSCRICAO | [09:12] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks com payload de criação e secret na resposta | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks com listagem paginada por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id para edição de URL, status e estado | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id com remoção do cadastro | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/secret/rotate com grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries com as últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay com role ADMIN | TRANSCRICAO | [09:36] Larissa |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload enxuto com event_id, event_type, timestamp, pedido, status e total | TRANSCRICAO | [09:43] Diego |
| FDD-HEADER-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp e Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-02 | docs/FDD.md | Contrato | Header X-Webhook-Id para identificar o cadastro que originou o envio | TRANSCRICAO | [09:44] Sofia |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND para cadastro inexistente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para URL recusada pela política https | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED para endpoint sem secret utilizável | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_INVALID_STATUS_FILTER para lista de status inválida | TRANSCRICAO | [09:33] Marcos |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64KB, com erro e sem truncar | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_INACTIVE para operação em webhook inativo ou removido | TRANSCRICAO | [09:21] Bruno |
| FDD-ERRO-07 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND no replay de registro inexistente | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-08 | docs/FDD.md | Erro | Derivado: WEBHOOK_ALREADY_REPLAYED barra o segundo replay do mesmo registro, protegendo a auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-ERRO-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT quando o envio passa de 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED para resposta não 2xx ou erro de rede | TRANSCRICAO | [09:15] Diego |
| FDD-TEST-01 | docs/FDD.md | Contrato | Derivado: vetor de teste reproduzível de HMAC-SHA256 sobre o corpo do evento | TRANSCRICAO | [09:20] Sofia |
| FDD-RES-01 | docs/FDD.md | Resiliência | Falha de entrega não derruba o worker: processamento isolado em try/catch | TRANSCRICAO | [09:14] Larissa |
| FDD-RES-02 | docs/FDD.md | Resiliência | Cliente lento não afeta a API, porque a entrega só existe no worker | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-03 | docs/FDD.md | Resiliência | Falha do banco no enqueue derruba a transação de propósito | TRANSCRICAO | [09:40] Bruno |
| FDD-RES-04 | docs/FDD.md | Resiliência | Sem fallback de descarte: evento é entregue, fica em retry ou vai para DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-RES-05 | docs/FDD.md | Resiliência | Worker não guarda retry em memória, o agendamento é persistido | TRANSCRICAO | [09:15] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Recuperação de evento travado em PROCESSING após crash do worker | TRANSCRICAO | [09:11] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Sem circuit breaker nesta fase, por não ter sido discutido | TRANSCRICAO | [09:15] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados em Pino com eventId, webhookId, ordem e resultado | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Redação de secret e do header de assinatura nos logs | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Derivado: métricas de fila, idade do pendente mais antigo, tentativas, duração e DLQ | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Correlação por event_id nos logs, já que não há tracer no projeto | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Alerta de fila pendente, idade de evento e queda de taxa de sucesso | TRANSCRICAO | [09:11] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus passa a chamar publishWebhookEvent com o tx da transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Novos erros estendem as classes existentes com prefixo WEBHOOK_ e são exportados no barrel | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Middleware de erro serializa AppError sem nenhuma alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | redactPaths ganha secret, secretPrevious e header de assinatura | CODIGO | src/shared/logger/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | buildControllers passa a instanciar o controller de webhooks | CODIGO | src/app.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Controllers ganha webhooks e o router monta /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | authenticate e requireRole ADMIN reaproveitados no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | validate passa a receber os schemas Zod do módulo de webhooks | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | env.ts ganha variáveis de polling, timeout, tentativas, limite e lote | CODIGO | src/config/env.ts |
| FDD-INT-10 | docs/FDD.md | Integração | worker usa createPrismaClient em instância própria por processo | CODIGO | src/config/database.ts |
| FDD-INT-11 | docs/FDD.md | Integração | server.ts serve de molde para o novo worker, sem ser alterado | CODIGO | src/server.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Quatro models novos e relação inversa em Customer | CODIGO | prisma/schema.prisma |
| FDD-INT-13 | docs/FDD.md | Integração | Migration aditiva nova no padrão da migration inicial | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-INT-14 | docs/FDD.md | Integração | Factory de teste de webhook no padrão das factories existentes | CODIGO | tests/helpers/factories.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Envelope de paginação reaproveitado nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Script npm run worker segue o padrão dos scripts existentes | CODIGO | package.json |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox em MySQL na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | [09:11] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Retry com backoff exponencial e DLQ em tabela separada | TRANSCRICAO | [09:17] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação em 24 horas | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com deduplicação por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso de AppError, Pino, error middleware e estrutura de módulos | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Snapshot do payload renderizado no momento da inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-008 | docs/adrs/ADR-008-filtro-de-eventos-na-insercao.md | Decisão | Filtro de eventos por status aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-002-CODIGO | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Worker precisa de Prisma Client próprio e mesmo banco da API | CODIGO | src/config/database.ts |
| ADR-006-CODIGO | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Restrição | Middleware só serializa AppError, ZodError e erros conhecidos do Prisma | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RESTRICAO-01 | docs/FDD.md | Restrição | Máquina de estados do pedido limita os status notificáveis | CODIGO | src/modules/orders/order.status.ts |
| FDD-RESTRICAO-02 | docs/FDD.md | Restrição | Payload não envia itens do pedido; detalhe fica em GET /orders/:id | CODIGO | src/modules/orders/order.repository.ts |
| FDD-RESTRICAO-03 | docs/FDD.md | Restrição | Schemas Zod e validação seguem o contrato de changeStatus existente | CODIGO | src/modules/orders/order.schemas.ts |

## Cobertura

| Métrica | Valor |
| --- | --- |
| Total de linhas | 174 |
| Linhas com Fonte = TRANSCRICAO | 148 |
| Linhas com Fonte = CODIGO | 26 |
| Percentual de linhas com origem na transcrição | 85.1% |
| Arquivos de código citados | 21, todos existentes no repositório base |
| ADRs cobertos | 8 de 8 |
| Endpoints cobertos | 7 de 7 |
| Códigos de erro WEBHOOK_* cobertos | 10 de 10 |

Cada item numerado dos documentos tem linha correspondente, com exceção dos itens declarados como proposta de implementação na seção 13 do FDD, listados abaixo.

## Itens derivados

Itens que aparecem nos documentos como proposta ou decisão de implementação, sem origem direta na transcrição nem no código. As linhas do tracker marcadas com "Derivado:" pertencem a este grupo: o timestamp aponta para o requisito que as motivou, não para uma fala que as tenha definido. Estão listados para transparência e ficam sujeitos a validação na revisão do RFC.

| Item | Onde aparece | Por que não tem origem |
| --- | --- | --- |
| Nomes das tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries` | FDD seção 12.12 | A reunião definiu o conceito de outbox e DLQ, não os nomes |
| Formato do valor de X-Signature com prefixo `sha256=` | FDD seção 5.1 | A reunião definiu header e algoritmo, não o formato do valor |
| Header X-Signature-Previous durante o grace period | FDD seções 5.6 e 13 | A reunião definiu a validade paralela de 24 horas, não o mecanismo |
| Tamanho de lote de 20 eventos por ciclo | FDD seção 13 | "Batch pequeno" não foi quantificado na reunião |
| Recuperação de eventos em PROCESSING por updatedAt antigo | FDD seções 4.2 e 13 | Mecanismo necessário para evento órfão, não discutido |
| Truncamento de responseBody em 2KB no histórico | FDD seção 5.7 | Proteção de armazenamento, não discutida |
| Remoção lógica de webhook com deletedAt | FDD seções 5.5 e 13 | Preserva integridade dos eventos em fila, não discutido |
| Métricas e tracing propostos na seção de observabilidade | FDD seção 8 | A reunião só definiu reuso do Pino, sem instrumentação |
| Metas OBJ-04 e OBJ-05 | PRD seção 4 | Não houve baseline nem número acordado na reunião |
