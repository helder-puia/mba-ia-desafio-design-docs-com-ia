# ADR-006 - Reuso máximo dos padrões existentes do projeto

## Status

Aceito em 2026-11-07.

## Contexto

O projeto tem convenções estáveis: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas; erros são classes que estendem `AppError` com um código próprio; o logger é Pino em toda a aplicação; o middleware de erro centralizado já trata `AppError`, `ZodError` e erros conhecidos do Prisma.

A reunião discutiu onde o módulo de webhooks ficaria, como os erros seriam modelados e se o worker abriria um Prisma Client separado. A conclusão foi de reuso máximo, sem introduzir biblioteca, logger ou hierarquia de erro paralela.

## Decisão

Criar o módulo como `src/modules/webhooks`, com a mesma divisão controller, service, repository, routes e schemas usada por `src/modules/orders` e `src/modules/customers`. A lógica de processamento fica no próprio módulo, em `webhook.processor.ts`, e o entry-point do processo fica em `src/worker.ts`.

Erros do módulo são novas classes que estendem as classes existentes de `src/shared/errors/http-errors.ts`, com códigos prefixados `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, entre outros). O logger continua sendo o Pino exportado por `src/shared/logger/index.ts`, e o middleware de erro existente passa a serializar os novos erros sem nenhuma alteração.

O worker usa `createPrismaClient()` de `src/config/database.ts`, em instância própria por ser outro processo Node, contra o mesmo banco e a mesma `DATABASE_URL`.

O endpoint de replay de DLQ reaproveita `authenticate` e `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`, exatamente como `src/modules/users/user.routes.ts` já faz.

## Alternativas Consideradas

1. **Serviço separado só para webhooks.** Plausível em uma organização com time de plataforma dedicado, mas exigiria novo deploy, novo pipeline e comunicação entre serviços, o que contradiz a decisão de time pequeno e de não subir infraestrutura nova ([09:07] Diego). Descartada.
2. **Hierarquia de erros própria do módulo, independente de `AppError`.** Descartada porque o middleware centralizado só sabe serializar `AppError`, `ZodError` e Prisma; uma hierarquia paralela exigiria alterar o middleware ou tratar erro em cada controller ([09:29] Bruno).
3. **Logger dedicado para o worker.** Descartada: o Pino já está no projeto inteiro e o mesmo logger serve ao processo novo, mantendo formato e redaction consistentes ([09:29] Bruno).
4. **Um único Prisma Client compartilhado entre API e worker.** Inviável por serem processos distintos; cada processo tem seu pool ([09:30] Bruno, [09:30] Diego).

## Consequências

### Positivas

- Curva de leitura baixa: quem conhece os outros módulos reconhece o de webhooks.
- Nenhuma dependência nova, nenhum padrão concorrente de log ou de erro.
- Middleware de erro e logger funcionam sem alteração, reduzindo superfície de regressão.
- Contratos de teste seguem o padrão de `tests/helpers/factories.ts`.

### Negativas

- O módulo herda limitações do desenho atual, como a ausência de camada de eventos e de injeção de dependência formal. `buildControllers` em `src/app.ts` precisa registrar mais um controller manualmente.
- Prefixo `WEBHOOK_` cria códigos que convivem com códigos genéricos (`NOT_FOUND`, `CONFLICT`). Um `WEBHOOK_NOT_FOUND` só é emitido se o service lançar a classe específica; onde o código reutilizar `NotFoundError`, o código genérico aparece.
- Reaproveitar `requireRole` mantém a autorização simples, sem política por cliente, o que é aceitável agora e provavelmente insuficiente depois ([09:36] Sofia, [09:37] Sofia).

## Referências

- Transcrição: [09:27] Bruno, [09:28] Bruno, [09:28] Diego, [09:29] Bruno, [09:29] Larissa, [09:29] Diego, [09:30] Bruno, [09:30] Larissa, [09:36] Larissa.
- Código: `src/modules/orders/order.service.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/config/database.ts`.
