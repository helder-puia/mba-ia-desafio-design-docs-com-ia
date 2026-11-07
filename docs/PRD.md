# PRD - Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Produto | Order Management System (OMS) |
| Feature | Sistema de Webhooks de Notificação de Pedidos |
| Responsável pelo PRD | Helder Puia |
| Versão | 1.0 |
| Data | 2026-11-07 |
| Status | Pronto para revisão |
| Documentos relacionados | [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/README.md), [TRACKER](TRACKER.md) |

## 1. Resumo e contexto da feature

O OMS opera hoje sem nenhum mecanismo de notificação externa. Clientes B2B que precisam acompanhar pedidos fazem polling em `GET /api/v1/orders` em intervalos regulares, o que torna a integração deles lenta e cara, e ainda assim sujeita a atraso entre a mudança real de status e a descoberta pelo cliente.

A feature entrega notificação outbound por webhook: sempre que o status de um pedido muda, o cliente cadastrado recebe um POST assinado no endpoint dele, com o evento de mudança. O cadastro é feito por API, com escolha de quais status cada endpoint quer receber. A entrega é assíncrona, resiliente a indisponibilidade do cliente e observável pelo histórico de entregas.

A feature entra em um sistema existente, a API Node.js + TypeScript com Express, Prisma e MySQL que já atende os módulos de autenticação, usuários, clientes, produtos e pedidos. Não é um sistema novo: é uma capacidade nova acoplada ao ciclo de vida do pedido que já existe.

Local de implantação: mesma aplicação e mesmo banco, com um processo adicional para o worker de entrega.

## 2. Problema e motivação

Três clientes B2B, Atlas Comercial, MaxDistribuição e Nova Cargo, fizeram pedido formal de notificação em tempo real de mudança de status ([09:00] Marcos). Hoje eles consultam a API repetidamente para descobrir se algo mudou.

Consequências práticas do modelo atual:

- Integração lenta e cara para o cliente, que paga em volume de requisições e em latência de descoberta ([09:00] Marcos).
- Nenhum sinal proativo: o cliente só sabe da mudança se perguntar.
- Risco comercial concreto: a Atlas sinalizou possibilidade de migrar para o concorrente caso a entrega não aconteça até o fim do trimestre ([09:00] Marcos).

A motivação não é técnica, é de retenção de receita: a ausência da feature é um fator de churn declarado por um cliente nomeado. Ao mesmo tempo, a solução não pode degradar o caminho crítico de mudança de status, que hoje executa em uma transação pesada com atualização de estoque e auditoria ([09:04] Bruno).

### 2.1 Problemas a resolver

- Integração do cliente depende de polling manual, o que é lento e caro do lado dele.
- O cliente não tem sinal proativo: só descobre a mudança de status se perguntar.
- A plataforma não possui nenhum mecanismo de notificação externa, eventos, filas ou webhooks.
- A ausência da capacidade é fator de churn declarado por cliente nomeado.

## 3. Público-alvo e cenários de uso

**Público-alvo primário:** clientes B2B com integração sistêmica ao OMS, que já consomem a API e têm capacidade técnica de expor um endpoint HTTPS e validar assinatura. Os três clientes que originaram a demanda são Atlas Comercial, MaxDistribuição e Nova Cargo.

**Público-alvo secundário:** usuários operadores do cliente, que hoje representam o cliente dentro da plataforma e farão o cadastro do webhook pela API ([09:32] Marcos).

**Público interno:** time de Pedidos, que opera o endpoint de replay de DLQ, e time de Segurança, que revisa a implementação de HMAC e geração de secret antes do deploy ([09:46] Sofia).

Cenários de uso principais:

| # | Cenário | Ator | Resultado esperado |
| --- | --- | --- | --- |
| CU-01 | Cliente cadastra um endpoint para receber apenas `SHIPPED` e `DELIVERED` | Operador do cliente | Recebe secret gerada e passa a receber só esses dois status ([09:33] Marcos) |
| CU-02 | Pedido muda de `PAID` para `SHIPPED` | Sistema | Cliente recebe o evento em menos de 10 segundos ([09:02] Marcos) |
| CU-03 | Endpoint do cliente está fora do ar | Sistema | Retry com backoff por até ~15 horas, depois DLQ ([09:17] Diego) |
| CU-04 | Cliente precisa auditar entregas | Operador do cliente | Consulta o histórico das últimas 100 entregas com resultado, payload e tempo de resposta ([09:34] Marcos) |
| CU-05 | Cliente suspeita de vazamento de secret | Operador do cliente | Rotaciona a secret e tem 24 horas de convivência com a antiga ([09:21] Sofia) |
| CU-06 | Evento falhou permanentemente e o cliente corrigiu o endpoint | Time interno com role ADMIN | Reprocessa manualmente o evento da DLQ ([09:18] Diego) |

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| PRD-OBJ-01 | Entregar mudança de status em tempo real para o cliente | p95 do intervalo entre commit da mudança de status e primeira tentativa de entrega | abaixo de 10 segundos ([09:02] Marcos) |
| PRD-OBJ-02 | Eliminar a necessidade de polling pelos clientes B2B | Quantidade dos três clientes demandantes com webhook ativo em produção | 3 de 3 no fim do trimestre ([09:00] Marcos) |
| PRD-OBJ-03 | Não perder evento de forma silenciosa | Percentual de eventos com estado final conhecido (`DELIVERED` ou em DLQ) | 100% dos eventos, sem descarte ([09:15] Diego) |
| PRD-OBJ-04 | Ser confiável o bastante para substituir o polling na prática | Taxa de entrega bem-sucedida na primeira tentativa | 95% ou mais (meta proposta, sem baseline na reunião) |
| PRD-OBJ-05 | Reduzir o custo de integração do cliente | Volume de chamadas de `GET /api/v1/orders` feitas pelos três clientes após 30 dias do lançamento | queda mensurável em relação ao baseline medido antes do lançamento (meta a calibrar) |

Observação de honestidade sobre metas: OBJ-01, OBJ-02 e OBJ-03 derivam diretamente de falas da reunião. OBJ-04 e OBJ-05 não têm número acordado, e estão marcadas como propostas a validar com o PM antes de virarem compromisso.

## 5. Escopo

### 5.1 Incluso

- Cadastro, edição, remoção e listagem de webhooks por customer, via API autenticada.
- Seleção de quais status cada webhook quer receber.
- Geração de evento transacional na mudança de status, sem inconsistência entre status e evento.
- Entrega assíncrona por worker em processo separado, com latência abaixo de 10 segundos.
- Retry com backoff exponencial em cinco tentativas ao longo de ~15 horas.
- Dead Letter Queue com reprocessamento manual por endpoint administrativo.
- Assinatura HMAC-SHA256 por endpoint, com secret única e rotação com grace period de 24 horas.
- Histórico de entregas consultável pelo cliente, com resultado, payload, resposta e tempo de resposta.
- Garantia at-least-once com identificador de evento para deduplicação no cliente.
- Documentação de integração no portal do desenvolvedor, mantida pelo PM ([09:26] Marcos).

### 5.2 Fora de escopo

Itens explicitamente descartados ou adiados na reunião:

| # | Item | Situação |
| --- | --- | --- |
| PRD-OOS-01 | Notificação por e-mail ao cliente quando o webhook falha | Adiado para próxima fase, depois de medir impacto ([09:37] Larissa) |
| PRD-OOS-02 | Dashboard visual para o cliente acompanhar seus webhooks | Descartado nesta fase; painel é projeto separado do time de frontend ([09:40] Larissa) |
| PRD-OOS-03 | Rate limiting de envio por cliente | Não entra agora; decisão foi observar e implementar se virar problema ([09:38] Diego, [09:39] Larissa) |
| PRD-OOS-04 | Webhooks inbound, com cliente enviando evento para a plataforma | Descartado no início da reunião; o fluxo é somente outbound ([09:02] Sofia, [09:02] Marcos) |
| PRD-OOS-05 | Garantia de ordenação global entre pedidos diferentes | Descartado; a garantia é apenas por `order_id` e com worker único ([09:13] Larissa) |
| PRD-OOS-06 | Escala para múltiplos workers com particionamento por `order_id` | Adiado como problema futuro ([09:13] Diego) |
| PRD-OOS-07 | Arquivamento de eventos entregues após 30 dias | Fora do escopo desta feature ([09:08] Diego) |
| PRD-OOS-08 | Reprocessamento automático de DLQ | Descartado; o replay é manual e administrativo ([09:18] Diego) |
| PRD-OOS-09 | Fila externa ou broker dedicado, como Redis Streams | Descartado por exigir infraestrutura nova ([09:07] Diego) |

## 6. Requisitos funcionais

### PRD-FR-01 - Cadastro de webhook

Prioridade: Alta.

O cliente cadastra um endpoint de webhook informando customer, URL e lista de status que quer receber. A secret é gerada pela plataforma e devolvida na resposta da criação.

Fluxo principal:
1. Operador autenticado envia `POST /api/v1/webhooks` com `customerId`, `url` e `statuses`.
2. Sistema valida o schema, incluindo URL `https` e lista de status não vazia.
3. Sistema valida que o customer existe.
4. Sistema gera secret única do endpoint e persiste o cadastro como ativo.
5. Sistema devolve `201` com o cadastro e a secret em texto claro, apenas nessa resposta.

Fluxos alternativos e exceções:
- URL `http`: recusada na validação, nada é persistido ([09:23] Sofia).
- Customer inexistente: `404` reaproveitando `NotFoundError`.
- Lista de status vazia: recusada com `WEBHOOK_INVALID_STATUS_FILTER`.

Erros previstos: `VALIDATION_ERROR`, `NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_STATUS_FILTER`, `UNAUTHORIZED`.

### PRD-FR-02 - Edição de webhook

Prioridade: Alta.

O cliente altera URL, lista de status ou estado ativo de um webhook existente, sem trocar a secret.

Fluxo principal:
1. Operador envia `PATCH /api/v1/webhooks/:id` com os campos a alterar.
2. Sistema valida existência e schema.
3. Sistema persiste a alteração e devolve `200` com o cadastro atualizado, sem secret.

Fluxos alternativos e exceções:
- Webhook inexistente: `404 WEBHOOK_NOT_FOUND`.
- URL nova em `http`: `422 WEBHOOK_INVALID_URL`.
- Desativar webhook: permitido, e o cadastro deixa de gerar eventos a partir daí ([09:21] Bruno, [09:33] Bruno).

Erros previstos: `VALIDATION_ERROR`, `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`.

### PRD-FR-03 - Remoção de webhook

Prioridade: Alta.

O cliente remove um cadastro que não quer mais usar. Entregas já registradas permanecem no histórico.

Fluxo principal:
1. Operador envia `DELETE /api/v1/webhooks/:id`.
2. Sistema valida existência e remove o cadastro.
3. Sistema devolve `204`.

Fluxos alternativos e exceções:
- Webhook inexistente: `404 WEBHOOK_NOT_FOUND`.
- Eventos ainda pendentes do webhook removido não são entregues, e o worker os encerra com motivo registrado, sem descarte silencioso, coerente com PRD-NFR-05.

Erros previstos: `WEBHOOK_NOT_FOUND`.

### PRD-FR-04 - Listagem de webhooks por customer

Prioridade: Alta.

O cliente consulta quais webhooks tem cadastrados, com paginação.

Fluxo principal:
1. Operador envia `GET /api/v1/webhooks?customerId=...`.
2. Sistema devolve `200` com a lista paginada, sem secret.

Fluxos alternativos e exceções:
- `customerId` ausente ou inválido: `400 VALIDATION_ERROR`.

Erros previstos: `VALIDATION_ERROR`.

### PRD-FR-05 - Filtro de eventos por status assinado

Prioridade: Alta.

Cada webhook escolhe quais status quer ouvir, e o filtro é aplicado no momento em que o evento é registrado, não no envio.

Fluxo principal:
1. Pedido muda para um status.
2. Sistema verifica quais webhooks ativos do customer têm aquele status na lista assinada.
3. Sistema cria evento apenas para os webhooks interessados.
4. Se nenhum webhook assinar o status, nenhum evento é criado ([09:34] Bruno).

Fluxos alternativos e exceções:
- Mais de um webhook do mesmo customer assinando o mesmo status: um evento por webhook.
- Webhook cadastrado depois de um evento: não recebe retroativamente, porque o evento nunca existiu.

Erros previstos: nenhum erro exposto ao cliente; falha aqui reverte a transação de status.

### PRD-FR-06 - Notificação transacional de mudança de status

Prioridade: Alta.

Toda mudança de status de pedido gera evento registrado na mesma transação da mudança, para qualquer webhook interessado.

Fluxo principal:
1. Operador muda o status via `PATCH /api/v1/orders/:id/status`.
2. Transação atualiza pedido, insere histórico e ajusta estoque quando aplicável.
3. Na mesma transação, sistema registra o evento com payload completo do estado do pedido naquele instante.
4. Transação commita; pedido e evento passam a existir juntos.

Fluxos alternativos e exceções:
- Falha ao registrar o evento: rollback completo, incluindo a mudança de status ([09:40] Bruno, [09:41] Diego).
- Transição inválida: comportamento atual do OMS é preservado, sem evento.

Erros previstos: erros atuais de `changeStatus` permanecem (`INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`); falha de persistência do evento retorna `500` e nada muda.

### PRD-FR-07 - Entrega assíncrona com baixa latência

Prioridade: Alta.

O cliente recebe o evento por HTTP POST no endpoint cadastrado, com latência compatível com "tempo real" na definição dele.

Fluxo principal:
1. Worker separado verifica eventos pendentes a cada 2 segundos ([09:09] Diego).
2. Worker envia POST assinado para o endpoint do cliente.
3. Cliente responde `2xx`; sistema marca como entregue e registra a entrega no histórico.

Fluxos alternativos e exceções:
- Cliente demora mais de 10 segundos: tratado como falha e entra no cronograma de retry ([09:42] Diego).
- Cliente responde erro: tratado como falha e entra no cronograma de retry.

Erros previstos: `WEBHOOK_DELIVERY_TIMEOUT`, `WEBHOOK_DELIVERY_FAILED`, ambos internos ao worker.

### PRD-FR-08 - Retry com backoff e Dead Letter Queue

Prioridade: Alta.

Falha de entrega não descarta o evento. Ele é retentado em cinco oportunidades e, se todas falharem, vai para DLQ.

Fluxo principal:
1. Falha na entrega agenda nova tentativa em 1 minuto.
2. Sequência de espera: 5 minutos, 30 minutos, 2 horas, 12 horas ([09:17] Diego).
3. Quinta falha move o evento para DLQ com payload, motivo e timestamp ([09:18] Diego).

Fluxos alternativos e exceções:
- Sucesso em qualquer tentativa encerra o ciclo e marca como entregue.
- Reinício do worker não reseta o cronograma, porque o agendamento é persistido.

Erros previstos: nenhum exposto ao cliente; o cliente vê o resultado no histórico de entregas.

### PRD-FR-09 - Reprocessamento manual de DLQ

Prioridade: Média.

Usuário com role ADMIN recoloca um evento da DLQ na fila de entrega.

Fluxo principal:
1. ADMIN envia `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
2. Sistema valida o registro e o estado do webhook.
3. Sistema recria o evento como pendente, com o mesmo identificador de evento.
4. Sistema registra em log e no registro da DLQ quem fez o replay.

Fluxos alternativos e exceções:
- Role diferente de ADMIN: `403 FORBIDDEN` ([09:36] Sofia).
- Registro inexistente: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.
- Webhook inativo: `409 WEBHOOK_INACTIVE`.

Erros previstos: `FORBIDDEN`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_INACTIVE`, `WEBHOOK_ALREADY_REPLAYED`.

### PRD-FR-10 - Histórico de entregas

Prioridade: Média.

O cliente consulta as últimas entregas do seu webhook, com resultado, payload enviado, resposta recebida e tempo de resposta ([09:34] Marcos).

Fluxo principal:
1. Operador envia `GET /api/v1/webhooks/:id/deliveries`.
2. Sistema devolve `200` com no máximo 100 registros por página, ordenados do mais recente para o mais antigo.

Fluxos alternativos e exceções:
- Webhook inexistente: `404 WEBHOOK_NOT_FOUND`.
- Webhook sem entregas: `200` com lista vazia.

Erros previstos: `WEBHOOK_NOT_FOUND`, `VALIDATION_ERROR`.

### PRD-FR-11 - Rotação de secret com grace period

Prioridade: Média.

O cliente troca a secret do endpoint sem parar a integração.

Fluxo principal:
1. Operador envia `POST /api/v1/webhooks/:id/secret/rotate`.
2. Sistema gera nova secret e devolve na resposta.
3. Secret anterior permanece aceita por 24 horas ([09:21] Sofia).
4. Após 24 horas, a anterior é invalidada.

Fluxos alternativos e exceções:
- Webhook inativo: `409 WEBHOOK_INACTIVE`.
- Rotação repetida dentro do grace period: caso não discutido na reunião. O desenho proposto substitui a secret anterior e reinicia a contagem de 24 horas, e o ponto fica aberto para confirmação na revisão do RFC.

Erros previstos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INACTIVE`, `WEBHOOK_SECRET_REQUIRED`.

### PRD-FR-12 - Assinatura HMAC-SHA256 de cada entrega

Prioridade: Alta.

O cliente consegue provar que a requisição veio da plataforma e que o payload não foi adulterado.

Fluxo principal:
1. Sistema calcula HMAC-SHA256 do corpo do request com a secret do endpoint.
2. Sistema envia `X-Signature`, `X-Timestamp` e `X-Webhook-Id` junto do corpo.
3. Cliente recalcula o HMAC e compara.

Fluxos alternativos e exceções:
- Secret rotacionada: durante 24 horas o cliente pode validar com a secret antiga ou a nova.
- Endpoint sem secret utilizável: envio bloqueado com `WEBHOOK_SECRET_REQUIRED` no registro do evento.

Erros previstos: `WEBHOOK_SECRET_REQUIRED`.

### PRD-FR-13 - Idempotência com identificador de evento

Prioridade: Alta.

O cliente consegue descartar entrega duplicada sem risco de processar o mesmo pedido duas vezes.

Fluxo principal:
1. Sistema gera identificador único no registro do evento.
2. Todo envio, inclusive retry e replay, leva o mesmo `X-Event-Id` ([09:25] Diego).
3. Cliente deduplica pelo identificador.

Fluxos alternativos e exceções:
- Cliente não deduplica: processa duplicado; a responsabilidade é dele e deve estar documentada no portal ([09:26] Marcos).

Erros previstos: nenhum.

## 7. Requisitos não funcionais

| # | Categoria | Especificação |
| --- | --- | --- |
| PRD-NFR-01 | Performance | p95 do intervalo entre commit da mudança de status e primeira tentativa de entrega abaixo de 10 segundos ([09:02] Marcos, [09:10] Larissa) |
| PRD-NFR-02 | Performance | Timeout de 10 segundos por chamada HTTP de entrega ([09:42] Diego) |
| PRD-NFR-03 | Performance | Latência da mudança de status não pode crescer de forma relevante; a transação não faz I/O externo ([09:04] Bruno) |
| PRD-NFR-04 | Confiabilidade | Garantia at-least-once; duplicidade possível e esperada, com deduplicação pelo cliente ([09:24] Diego) |
| PRD-NFR-05 | Confiabilidade | Nenhum evento é descartado silenciosamente: todo evento termina entregue ou em DLQ ([09:15] Diego) |
| PRD-NFR-06 | Segurança | URL obrigatoriamente `https`; `http` recusado na validação ([09:23] Sofia) |
| PRD-NFR-07 | Segurança | HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com grace period de 24 horas ([09:20] Sofia, [09:21] Sofia) |
| PRD-NFR-08 | Segurança | Limite de 64KB por payload, com erro acima disso, sem truncamento ([09:24] Diego, [09:24] Larissa) |
| PRD-NFR-09 | Segurança | Role ADMIN obrigatória para replay de DLQ, com registro de auditoria de quem executou ([09:36] Sofia) |
| PRD-NFR-10 | Observabilidade | Logs estruturados via Pino, com identificação de evento, webhook, pedido e resultado da tentativa ([09:29] Bruno) |
| PRD-NFR-11 | Compatibilidade | Nenhum endpoint existente muda de contrato; polling em `GET /orders` continua funcionando |
| PRD-NFR-12 | Operabilidade | Worker roda em processo separado, com entry-point e script próprios ([09:11] Diego, [09:11] Larissa) |
| PRD-NFR-13 | Manutenibilidade | Módulo segue a estrutura controller, service, repository, routes e schemas dos demais módulos ([09:27] Bruno) |

## 8. Arquitetura e abordagem

A feature é entregue dentro do OMS existente, sem microsserviço novo e sem broker de mensagens. A abordagem tem quatro componentes: registro transacional do evento em uma tabela outbox no MySQL, entrega por um worker em processo separado que consulta a outbox em intervalos de 2 segundos, assinatura HMAC-SHA256 por endpoint em cada envio, e persistência de configuração, fila e histórico no mesmo banco que a aplicação já usa.

- **Componentes novos:** tabela de configuração de webhooks, tabela outbox, tabela de dead letter, histórico de entregas e um processo worker.
- **Integrações externas:** endpoint HTTPS do cliente, único ponto de saída de rede da feature.
- **Pontos de acoplamento internos:** a transação de mudança de status, o middleware de erro, o logger e o módulo de autenticação/autorização já existentes.
- **Padrão de comunicação:** persistência assíncrona no banco e HTTP POST assinado na saída, sem chamada síncrona no caminho crítico.

O detalhe de arquitetura e as alternativas descartadas estão no [RFC](RFC.md); o desenho de implementação, no [FDD](FDD.md).

## 9. Decisões e trade-offs principais

| # | Decisão | Justificativa | Trade-off |
| --- | --- | --- | --- |
| PRD-DEC-01 | Outbox em tabela MySQL, escrita na mesma transação da mudança de status | Atomicidade entre status e evento, sem infraestrutura nova ([09:06] Diego, [09:07] Diego) | Transação mais pesada e tabela que cresce sem arquivamento nesta fase |
| PRD-DEC-02 | Worker em processo separado com polling de 2 segundos | Isola ciclo de vida e atende o teto de 10 segundos ([09:09] Diego, [09:11] Diego) | Segundo processo para operar; ordenação global só com worker único |
| PRD-DEC-03 | Retry com backoff de 1m/5m/30m/2h/12h e DLQ em tabela separada | Cobre indisponibilidades reais de até 2 horas e dá estado final determinístico ([09:16] Diego, [09:18] Diego) | Cliente pode ficar até ~15 horas sem receber antes da falha permanente |
| PRD-DEC-04 | HMAC-SHA256 com secret por endpoint e rotação em 24 horas | Isola vazamento por cliente e permite troca sem downtime ([09:21] Sofia) | Cliente precisa implementar verificação e guardar segredo com cuidado |
| PRD-DEC-05 | At-least-once com `X-Event-Id` | Resolve duplicidade sem coordenação entre as partes ([09:25] Diego) | Responsabilidade de deduplicação transferida ao cliente |
| PRD-DEC-06 | Reuso dos padrões existentes do projeto | Consistência, velocidade e menos superfície de regressão ([09:29] Bruno, [09:30] Larissa) | Módulo herda limitações do desenho atual |
| PRD-DEC-07 | Snapshot do payload na inserção | Evento reflete o estado no momento da transição ([09:52] Larissa) | Duplicação de dados e eventos antigos com formato antigo |
| PRD-DEC-08 | Filtro por status na inserção do evento | Outbox menor e menos trabalho para o worker ([09:34] Bruno) | Consulta extra na transação e nenhum replay histórico de eventos não assinados |

## 10. Dependências

| # | Tipo | Dependência | Detalhe |
| --- | --- | --- | --- |
| PRD-DEP-01 | Organizacional | Time de Pedidos com capacidade para três sprints | Estimativa de Larissa, incluindo revisão de segurança ([09:46] Larissa) |
| PRD-DEP-02 | Organizacional | Revisão de segurança da Sofia antes do deploy | Pelo menos dois dias úteis reservados para HMAC e geração de secret ([09:46] Sofia) |
| PRD-DEP-03 | Externa | Endpoint HTTPS operacional no cliente | Sem endpoint válido, o cadastro é recusado ou o evento vai para retry e DLQ |
| PRD-DEP-04 | Externa | Documentação de integração no portal do desenvolvedor | Marcos se comprometeu a documentar, incluindo a necessidade de deduplicar ([09:26] Marcos, [09:40] Marcos) |
| PRD-DEP-05 | Técnica | Banco MySQL e Prisma já em produção | Todas as tabelas novas vivem no banco existente ([09:07] Diego) |
| PRD-DEP-06 | Técnica | Usuários do OMS que representam o cliente | Autenticação do CRUD usa o JWT atual, não um JWT de cliente ([09:32] Marcos) |
| PRD-DEP-07 | Comercial | Prazo esperado pela Atlas | Cliente pediu fim de novembro; estimativa técnica é de três sprints ([09:45] Marcos, [09:46] Larissa) |

## 11. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação | Contingência |
| --- | --- | --- | --- | --- | --- |
| PRD-RISK-01 | Prazo comercial de fim de novembro não ser atingido com três sprints de escopo | Alta | Alto | Renegociar prazo com a Atlas usando a estimativa técnica; priorizar CRUD e entrega com retry no primeiro incremento ([09:46] Larissa) | Entregar escopo mínimo (cadastro e entrega) e adiar rotação de secret e histórico para incremento seguinte, com acordo do PM |
| PRD-RISK-02 | Aumento da latência de mudança de status por causa da escrita na outbox | Média | Alto | Inserção enxuta, índice por status e data, medição antes e depois, payload montado em memória ([09:04] Bruno) | Desligar a geração de eventos por feature flag enquanto o problema é investigado |
| PRD-RISK-03 | Worker parado e fila crescendo sem ninguém perceber | Média | Alto | Monitoramento de tamanho da fila e idade do evento mais antigo; alerta; healthcheck ([09:11] Diego) | Reprocessamento em massa após religar o worker, já que o evento está persistido |
| PRD-RISK-04 | Secret vazada em log da plataforma ou em log do cliente | Baixa | Alto | Redação do campo no logger, devolução apenas na criação e rotação, revisão de segurança da Sofia ([09:22] Diego, [09:46] Sofia) | Rotação imediata da secret afetada e comunicação ao cliente |
| PRD-RISK-05 | Cliente processa evento duplicado por não deduplicar | Média | Médio | `X-Event-Id` estável e documentação destacada no portal ([09:26] Marcos) | Suporte conjunto para reprocessar o pedido do lado do cliente |
| PRD-RISK-06 | Crescimento indefinido da outbox por falta de arquivamento | Alta | Médio | Monitorar tamanho e tempo de leitura; arquivamento registrado como próxima fase ([09:08] Diego) | Arquivamento emergencial por script, fora do escopo da feature |
| PRD-RISK-07 | Cliente sem capacidade técnica de validar HMAC | Média | Médio | Documentação com exemplo de verificação no portal; suporte na integração | Oferecer período de validação assistida durante o onboarding |

## 12. Critérios de aceitação

- [ ] PRD-AC-01: É possível cadastrar um webhook com URL `https` e lista de status, recebendo a secret na resposta.
- [ ] PRD-AC-02: Cadastro com URL `http` é recusado com erro claro e nada é persistido.
- [ ] PRD-AC-03: É possível editar URL, status e estado ativo sem trocar a secret.
- [ ] PRD-AC-04: É possível remover um webhook e deixar de receber eventos dele.
- [ ] PRD-AC-05: É possível listar os webhooks de um customer com paginação.
- [ ] PRD-AC-06: Mudança de status de pedido gera evento para todos os webhooks ativos que assinam aquele status.
- [ ] PRD-AC-07: Status sem nenhum webhook interessado não gera linha na fila.
- [ ] PRD-AC-08: Evento chega ao endpoint do cliente em menos de 10 segundos no cenário nominal.
- [ ] PRD-AC-09: Falha de entrega gera cinco tentativas no cronograma 1m/5m/30m/2h/12h.
- [ ] PRD-AC-10: Após cinco falhas o evento aparece em DLQ com payload, motivo e timestamp.
- [ ] PRD-AC-11: Usuário ADMIN reprocessa evento da DLQ e o evento volta a ser entregue.
- [ ] PRD-AC-12: Usuário sem role ADMIN recebe `403` ao tentar reprocessar.
- [ ] PRD-AC-13: Cliente consulta as últimas 100 entregas com resultado, payload, resposta e tempo de resposta.
- [ ] PRD-AC-14: Rotação de secret mantém a anterior válida por 24 horas.
- [ ] PRD-AC-15: Toda entrega traz HMAC-SHA256 verificável com a secret do endpoint.
- [ ] PRD-AC-16: Duas entregas do mesmo evento carregam o mesmo `X-Event-Id`.
- [ ] PRD-AC-17: Nenhuma mudança de status fica registrada sem o evento correspondente quando há webhook interessado.
- [ ] PRD-AC-18: Nenhum endpoint existente da API muda de contrato ou de comportamento.

## 13. Estratégia de testes e validação

| # | Tipo | O que valida | Como |
| --- | --- | --- | --- |
| TEST-01 | Unitário | Cálculo do cronograma de backoff e decisão de mover para DLQ | Testes de função pura com relógio controlado |
| TEST-02 | Unitário | Cálculo e verificação do HMAC-SHA256 | Comparação com vetor conhecido e com assinatura recalculada |
| TEST-03 | Integração | Inserção do evento dentro da transação de `changeStatus` | `supertest` no padrão de `tests/orders.test.ts`, com verificação de rollback forçado |
| TEST-04 | Integração | CRUD de webhooks, filtros e validação de URL | `supertest` com factories de `tests/helpers/factories.ts` |
| TEST-05 | Integração | Worker entregando para um servidor HTTP local de teste | Servidor fake que simula 2xx, 5xx, timeout e indisponibilidade |
| TEST-06 | Integração | Retry, DLQ e replay | Fake server forçando falha e relógio controlado para não esperar 15 horas |
| TEST-07 | Segurança | Secret nunca aparece em log; rotação invalida a antiga após o prazo | Inspeção de log e teste de verificação de assinatura com secret antiga e nova |
| TEST-08 | Contrato | Payload e headers exatamente como especificados no FDD | Teste de contrato comparando JSON enviado com fixture |
| TEST-09 | Carga | Latência de `changeStatus` e p95 de entrega sob volume | Teste de carga no caminho de status, comparando com baseline antes da feature |
| TEST-10 | Manual | Integração ponta a ponta com um cliente real | Piloto com a Atlas em ambiente de homologação antes do deploy |

Validação de aceite: checklist da seção 12 executado em homologação, com revisão de segurança da Sofia antes do deploy ([09:46] Sofia) e confirmação do PM de que o comportamento atende ao pedido dos três clientes.
