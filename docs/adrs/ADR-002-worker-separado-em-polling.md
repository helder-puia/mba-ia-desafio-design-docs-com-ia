# ADR-002 - Worker em processo separado com polling de 2 segundos

## Status

Aceito em 2026-11-07.

## Contexto

Com o outbox definido (ADR-001), faltava decidir quem lê a tabela e como. A leitura poderia acontecer dentro da própria API (em memória, no mesmo processo Express) ou em um processo dedicado. Também era necessário definir o mecanismo de acionamento: polling, trigger ou algum listener de banco.

A restrição de latência veio do cliente: "qualquer coisa abaixo de 10 segundos já é tempo real" ([09:02] Marcos). Diego descartou trigger como mecanismo de aviso, porque MySQL não notifica processo externo ([09:09] Diego). Larissa apontou que já existe um entry-point em `src/server.ts`, então criar um `src/worker.ts` com script `npm run worker` segue o padrão do projeto ([09:11] Larissa).

## Decisão

O worker roda como processo Node separado, com entry-point próprio em `src/worker.ts` e script `npm run worker`. Ele abre seu próprio `PrismaClient` (instância por processo, mesmo `DATABASE_URL`, mesmo banco) e faz polling a cada 2 segundos, buscando os eventos pendentes mais antigos em lotes pequenos, processando e marcando o resultado.

O worker roda em instância única nesta fase. O ordenamento é implícito por `order_id` enquanto houver um único worker lendo em ordem de `created_at` e nenhuma tentativa pendente para o pedido. A ordem não é preservada quando um evento do pedido falha e entra em backoff: o evento seguinte do mesmo pedido, já elegível, é entregue antes da retentativa.

## Alternativas Consideradas

1. **Worker dentro do processo da API.** Descartada explicitamente: se a API reinicia ou é redeployada, o worker morre junto e as notificações param sem que ninguém perceba ([09:11] Diego). O trade-off foi aceitar operar um segundo processo em troca de isolamento de ciclo de vida.
2. **Trigger de banco para reagir na hora.** Descartada por não haver canal de notificação para processo externo em MySQL; a saída seria improvisar arquivo ou endpoint, o que foi considerado "esquisito" ([09:09] Diego).
3. **Listener nativo de banco (estilo `NOTIFY/LISTEN` do Postgres).** Não disponível na stack atual, que é MySQL ([09:09] Diego). Trocar de banco para ganhar reatividade foi considerado custo desproporcional.
4. **Webhook inbound da aplicação para acordar o worker.** Não discutida na reunião e não adotada por adicionar superfície de rede e autenticação para resolver um problema que 2 segundos de polling já resolvem.

## Consequências

### Positivas

- Ciclo de vida independente: deploy ou restart da API não interrompe o processamento da fila.
- Latência mínima previsível de 2 segundos e pior caso bem abaixo do teto de 10 segundos aceito pelos clientes ([09:10] Larissa).
- Simplicidade operacional: mesmo runtime, mesmo banco, mesmo Prisma, sem broker.
- Permite escalar horizontalmente no futuro sem mudança de contrato.

### Negativas

- Passa a existir um segundo processo para provisionar, monitorar e reiniciar (deploy, healthcheck, logs separados).
- Polling gera consultas periódicas mesmo quando não há evento, com custo constante de banco.
- O ordenamento global não é garantido: é implícito por `order_id`, depende de worker único e quebra quando há retry pendente para o mesmo pedido, porque o evento seguinte não espera a retentativa do anterior. A correção exigiria head-of-line blocking por pedido, particionamento ou lock pessimista, todos fora desta fase. Escalar para múltiplos workers exige particionamento por `order_id` ou lock pessimista, explicitamente adiado como problema futuro ([09:12] Diego, [09:13] Larissa).
- Reinício do worker no meio de um envio pode gerar reenvio do mesmo evento, o que é absorvido pela garantia at-least-once (ADR-005).

## Referências

- Transcrição: [09:09] Diego, [09:09] Bruno, [09:10] Marcos, [09:10] Larissa, [09:11] Diego, [09:11] Larissa, [09:11] Bruno, [09:12] Diego, [09:13] Diego, [09:13] Larissa.
- Código: `src/server.ts` (padrão de bootstrap e shutdown), `src/config/database.ts` (criação do `PrismaClient`), `package.json` (scripts `dev`, `start`, `test`).
