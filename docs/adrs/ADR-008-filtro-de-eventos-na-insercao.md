# ADR-008 - Filtro de eventos na inserção da outbox

## Status

Aceito em 2026-11-07.

## Contexto

Cada cadastro de webhook pode escolher quais status quer receber, por exemplo "só quero saber quando vira SHIPPED e DELIVERED" ([09:33] Marcos). Era preciso decidir em que ponto esse filtro é aplicado: ao inserir o evento na outbox ou ao enviar para o cliente.

Diego perguntou explicitamente onde filtrar. Bruno respondeu na inserção, argumentando que se nenhum webhook do customer quer aquele status, a linha nem é criada, economizando espaço na tabela.

## Decisão

Aplicar o filtro na inserção. Dentro da transação de mudança de status, a plataforma verifica quais webhooks ativos do customer assinam o novo status e só insere evento para os que assinam. Se nenhum assinar, nada é inserido.

## Alternativas Consideradas

1. **Filtrar no envio, inserindo sempre.** Descartada por gerar linha para evento que ninguém vai receber, aumentando a tabela e o trabalho do worker ([09:34] Bruno, [09:34] Diego). O trade-off foi aceitar uma leitura adicional dentro da transação em troca de uma outbox menor.
2. **Filtrar nos dois pontos.** Não foi proposta; seria defesa em profundidade, mas duplicaria a regra e criaria dois lugares para manter.

## Consequências

### Positivas

- Outbox menor, com menos linhas para o worker varrer e para arquivar no futuro.
- O worker não precisa conhecer a configuração de assinatura, apenas entregar o que está na fila.
- Reduz envio de eventos que o cliente não pediu.

### Negativas

- A transação de `changeStatus` ganha uma consulta a `webhook_endpoints` (assinaturas ativas do customer). Isso aumenta o tempo de lock da linha do pedido, justamente no caminho que já era pesado ([09:04] Bruno).
- Efeito retroativo indesejado: se o cliente cadastrar um novo filtro depois, eventos que aconteceram antes não existem em lugar nenhum, porque nunca foram inseridos. Não há replay histórico possível.
- A regra de filtro passa a viver no caminho transacional do pedido, acoplando o módulo de orders à configuração de webhooks. A mitigação é encapsular tudo em `publishWebhookEvent(tx, order, fromStatus, toStatus)`, sem injetar o repositório inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Referências

- Transcrição: [09:33] Marcos, [09:33] Bruno, [09:34] Diego, [09:34] Bruno, [09:34] Diego, [09:41] Bruno, [09:41] Diego.
- Código: `src/modules/orders/order.service.ts` (método `changeStatus`, onde o filtro é aplicado dentro do `$transaction`), `src/modules/orders/order.status.ts` (enumeração de status usada no filtro).
