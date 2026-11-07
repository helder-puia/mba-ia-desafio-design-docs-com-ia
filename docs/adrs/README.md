# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature de Webhooks de Notificação de Pedidos.
Cada decisão arquitetural relevante é registrada em um arquivo individual, nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox em tabela MySQL | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint | Aceito |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Garantia at-least-once com deduplicação por X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado na inserção da outbox | Aceito |
| [ADR-008](ADR-008-filtro-de-eventos-na-insercao.md) | Filtro de eventos na inserção da outbox | Aceito |

Formato adotado: MADR, com as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências.
Cada ADR referencia a origem da decisão na transcrição da reunião e, quando aplicável, os arquivos do código base envolvidos.
