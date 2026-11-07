# ADR-003 - Retry com backoff exponencial e DLQ em tabela separada

## Status

Aceito em 2026-11-07.

## Contexto

O cliente pode estar indisponível quando o worker tentar entregar o evento. Era preciso definir quantas tentativas fazer, com qual intervalo, e o que fazer quando todas falharem. O histórico conhecido da empresa inclui cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego), o que estabeleceu o piso mínimo tolerável de janela de retry.

A discussão contrapôs três tentativas (proposta de Bruno, mais agressiva) e retry indefinido (defendido por "algumas pessoas", segundo Diego, mas que deixaria evento pendurado para sempre se o cliente desaparecesse). Diego também propôs uma tabela separada para as falhas permanentes.

## Decisão

Cinco tentativas, com backoff exponencial de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando aproximadamente 15 horas entre a primeira falha e a última tentativa. Depois da quinta falha, o evento é considerado falha permanente e movido para `webhook_dead_letter`, tabela separada que guarda payload, motivo da falha e timestamp.

O reprocessamento é manual, por endpoint administrativo que recoloca o evento na outbox como pendente (ver FDD).

## Alternativas Consideradas

1. **Três tentativas.** Descartada porque três retentativas em cerca de 30 minutos matariam o evento em uma indisponibilidade matinal do cliente, cenário que já ocorreu na base ([09:16] Bruno, [09:16] Diego). O trade-off de menos carga de retry foi recusado em favor de cobrir a janela real de manutenção dos clientes.
2. **Retry indefinido com backoff.** Descartada porque evento de cliente que sumiu ficaria pendurado para sempre, sem critério de encerramento e sem sinal claro de falha ([09:15] Diego). O trade-off foi aceitar uma janela finita de ~15 horas em troca de um estado final determinístico.
3. **Marcar falha com status `failed` na própria outbox, sem tabela separada.** Descartada por Diego: uma tabela dedicada mantém a leitura da outbox principal limpa e serve como evidência para debug e reprocessamento ([09:18] Diego). O custo aceito é duplicar o payload em outra tabela.

## Consequências

### Positivas

- Janela de ~15 horas cobre indisponibilidades reais já observadas em clientes.
- Estado final determinístico: todo evento termina como entregue ou em DLQ, nunca "pendurado".
- A DLQ preserva payload, motivo e timestamp, permitindo diagnóstico e replay pontual.
- Backoff crescente reduz pressão sobre um cliente que já está com problema.

### Negativas

- Um cliente pode ficar até ~15 horas sem receber o evento antes de a falha ser considerada permanente. Marcos considerou aceitável, mas é uma janela longa para "tempo real" ([09:17] Marcos).
- A DLQ exige operação humana: ninguém é avisado automaticamente (notificação por e-mail ficou fora de escopo), então o monitoramento depende de observabilidade e de processo interno.
- O replay manual pode gerar entrega duplicada se o cliente já tiver recebido o evento por outra via, o que reforça a dependência de deduplicação por `X-Event-Id` (ADR-005).
- O worker precisa persistir `attempt_count` e `next_attempt_at` para respeitar o cronograma, aumentando o estado a manter na tabela.

## Referências

- Transcrição: [09:14] Larissa, [09:15] Diego, [09:15] Bruno, [09:16] Bruno, [09:16] Diego, [09:17] Diego, [09:17] Marcos, [09:17] Larissa, [09:18] Diego, [09:18] Bruno.
- Código: `package.json` (orquestração de scripts do worker), `prisma/schema.prisma` (padrão de nomes de tabela com `@@map`).
