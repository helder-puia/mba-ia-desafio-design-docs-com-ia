# ADR-005 - Garantia at-least-once com deduplicação por X-Event-Id

## Status

Aceito em 2026-11-07.

## Contexto

O worker pode reenviar um evento por falha de rede, timeout ambíguo ou reinício no meio do processamento. Era preciso decidir qual garantia de entrega assumir e como o cliente se protege de duplicidade.

Sofia observou que a solução joga responsabilidade para o cliente. Diego defendeu que é o padrão de mercado e que garantir exactly-once exigiria coordenação entre as duas partes, com complexidade muito maior.

## Decisão

Adotar garantia at-least-once. O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para deduplicar.

Cada evento recebe um UUID no momento em que é inserido na outbox, persistido na própria linha e enviado no header `X-Event-Id`, que é único por evento e estável entre tentativas. O cliente deduplica pelo valor desse header.

## Alternativas Consideradas

1. **Exactly-once.** Descartada por exigir coordenação entre plataforma e cliente (confirmação transacional dos dois lados), com complexidade desproporcional ao problema ([09:25] Diego). O trade-off aceito foi transferir a deduplicação para o cliente em troca de um worker simples e resiliente.
2. **At-most-once (não repetir nunca).** Alternativa plausível, mas levaria a perda de evento em qualquer falha transitória de rede, o que contraria a expectativa de que todo cliente receba toda mudança de status ([09:24] Diego). Descartada por perda silenciosa.
3. **Deduplicação no servidor com controle de confirmação do cliente.** Não foi proposta na reunião, e exigiria um endpoint de ack e estado adicional por evento, reaproximando o desenho de exactly-once.

## Consequências

### Positivas

- Entrega resiliente: falha transitória é retentada, não descartada.
- `X-Event-Id` dá ao cliente uma chave estável de deduplicação, inclusive entre retries e replay manual de DLQ.
- O worker não precisa de estado de confirmação do cliente para avançar.
- Alinhado ao comportamento de provedores conhecidos pelo mercado, o que facilita a documentação no portal do desenvolvedor ([09:25] Diego, [09:26] Marcos).

### Negativas

- A plataforma transfere ao cliente a obrigação de deduplicar. Cliente que ignore o header vai processar o mesmo pedido duas vezes.
- O `event_id` precisa ser gerado na inserção, não no envio, senão cada retry teria um id diferente e a deduplicação ficaria impossível (ver ADR-007).
- Eventos reenviados por replay de DLQ carregam o mesmo `event_id`, o que é correto para deduplicação, mas exige que o cliente mantenha a janela de ids por tempo suficiente.

## Referências

- Transcrição: [09:24] Diego, [09:25] Bruno, [09:25] Diego, [09:25] Sofia, [09:26] Marcos, [09:26] Larissa.
- Código: `prisma/schema.prisma` (uso de UUID como padrão de identificador).
