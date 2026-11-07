# ADR-001 - Padrão Outbox em tabela MySQL

## Status

Aceito em 2026-11-07.

## Contexto

A mudança de status de um pedido hoje acontece em uma única transação SQL que atualiza `orders`, insere em `order_status_history` e, quando aplicável, decrementa ou repõe `stock_quantity` de produtos (`src/modules/orders/order.service.ts`, método `changeStatus`). Essa transação já é pesada: no caminho `PENDING -> PAID` ela lê produtos, valida estoque e executa um `UPDATE` por item.

Era necessário notificar clientes B2B quando o status muda, e a primeira pergunta da reunião foi onde disparar a notificação: dentro do service de orders, de forma síncrona, ou através de algum mecanismo de fila.

Bruno levantou dois problemas do disparo síncrono: um cliente lento travaria a mudança de status de outros pedidos, e não existe resposta razoável para "e se o cliente estiver fora do ar, faz rollback do status?". Diego descartou a alternativa de Redis Streams por exigir infraestrutura nova para um time pequeno, e descartou gatilho de banco porque MySQL não notifica processo externo.

## Decisão

Usar o padrão Outbox em uma tabela MySQL (`webhook_outbox`) no mesmo banco da aplicação. A inserção do evento acontece dentro da mesma transação que muda o status do pedido, recebendo o `tx` (Prisma `TransactionClient`) já aberto. Se a transação principal commitar, o evento existe; se der rollback, o evento desaparece junto.

A tabela guarda o evento já renderizado (ver ADR-007) e usa UUID como chave primária, seguindo o padrão de identificadores do projeto.

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro de `changeStatus`.** Descartada por acoplar a latência de um sistema externo ao tempo de resposta da API e por não haver tratamento possível de falha sem violar a consistência do status ([09:04] Bruno, [09:06] Diego). O trade-off aceito foi abrir mão da entrega imediata em troca de desacoplamento e previsibilidade.
2. **Redis Streams (ou fila externa equivalente).** Descartada por exigir subir e operar infraestrutura adicional (cluster Redis) para um time pequeno, o que foi classificado como overengineering ([09:07] Larissa, [09:07] Diego). O trade-off foi abrir mão de entrega quase instantânea e de um broker dedicado em troca de zero infraestrutura nova.
3. **Trigger de banco de dados.** Descartada porque MySQL não possui listener nativo equivalente ao `NOTIFY/LISTEN` do Postgres: a trigger só executa SQL e não notifica processo externo, exigindo gambiarras como escrever em arquivo ([09:09] Diego).

## Consequências

### Positivas

- Atomicidade real entre mudança de status e registro do evento, sem inconsistência possível entre os dois.
- Nenhuma infraestrutura nova: o banco MySQL e o Prisma Client já existentes suportam a solução.
- Um único ponto de falha conhecido (o banco) em vez de uma cadeia de componentes distribuídos.
- O worker pode reler e reenviar o evento quantas vezes precisar, porque o payload está persistido.

### Negativas

- A transação de `changeStatus` fica ainda mais pesada: ganha uma escrita adicional e, na prática, uma leitura de configuração de webhooks (ver ADR-008). Isso aumenta o tempo de lock da linha do pedido.
- A tabela cresce indefinidamente. O arquivamento de linhas entregues após 30 dias foi explicitamente colocado fora do escopo desta feature ([09:08] Diego), então a operação precisa de um plano posterior.
- A garantia de consistência vale para o registro do evento, não para a entrega: o cliente pode receber o evento depois, ou nunca receber (nesse caso vai para DLQ).

## Referências

- Transcrição: [09:03] Larissa, [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:07] Diego, [09:08] Diego, [09:09] Diego, [09:41] Bruno.
- Código: `src/modules/orders/order.service.ts` (método `changeStatus`), `prisma/schema.prisma` (convenções de model e `@db.Char(36)` com UUID).
