# ADR-007 - Snapshot do payload renderizado na inserção da outbox

## Status

Aceito em 2026-11-07.

## Contexto

A linha da outbox poderia guardar apenas a referência do pedido (`order_id`, `from_status`, `to_status`) e renderizar o payload no momento do envio, ou guardar o JSON já pronto no instante da inserção.

Bruno levantou a dúvida no fim da reunião, depois que os demais participantes já haviam saído, e Larissa decidiu pelo payload renderizado na inserção: se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou.

## Decisão

Renderizar o payload completo no momento da inserção na outbox e persistir o JSON como snapshot. O worker envia exatamente o que está armazenado, sem consultar `orders` para montar o corpo.

## Alternativas Consideradas

1. **Renderizar no envio, a partir do `order_id`.** Descartada por Larissa: se o pedido mudasse depois, o evento entregue refletiria um estado diferente do momento da mudança de status, gerando "caso esquisito" ([09:52] Larissa). O trade-off foi pagar armazenamento duplicado em troca de fidelidade temporal.
2. **Guardar apenas a diferença (`order_id`, `from_status`, `to_status`) e completar no envio.** Mesmo problema do item anterior, com payload ainda menor. Descartada pelo mesmo motivo.

## Consequências

### Positivas

- O evento entregue reflete fielmente o estado do pedido no instante da transição.
- O worker fica desacoplado do schema de `orders`: não precisa de join nem de leitura adicional, o que reduz carga no banco durante o processamento.
- Retries reenviam exatamente o mesmo corpo, o que mantém a assinatura HMAC estável e a deduplicação previsível.
- O snapshot é a evidência do que foi entregue, útil para auditoria e para o histórico de entregas.

### Negativas

- Duplicação de dados: o payload é armazenado na outbox, na DLQ (quando falha) e no histórico de entregas.
- Campos novos adicionados ao payload no futuro não aparecem em eventos já enfileirados, criando convivência de formatos por um período.
- Consumidores que precisam de dados completos do pedido ainda dependem de `GET /api/v1/orders/:id`, como a reunião previu ([09:43] Diego).

## Referências

- Transcrição: [09:51] Bruno, [09:52] Larissa, [09:52] Diego, [09:52] Bruno, [09:43] Diego.
- Código: `src/modules/orders/order.service.ts` (dados disponíveis no escopo da transação: `order.id`, `order.orderNumber`, `order.customerId`, `order.totalCents`).
