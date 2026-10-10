# Architectural Decision Records

Este diretório armazena os ADRs (Architecture Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**, extraídos da reunião técnica registrada em `TRANSCRICAO.md` e do código existente em `src/` e `prisma/`.

Convenção de nome: `ADR-NNN-titulo-em-kebab-case.md`. Formato: MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências), com uma tabela de metadados no topo apontando os timestamps da transcrição que originaram a decisão.

## Índice

| ADR | Decisão | Decisão principal do enunciado |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL, inserção na mesma transação de `changeStatus` | Padrão Outbox no MySQL |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado (`src/worker.ts`), polling de 2 s, single-worker | Worker em processo separado em polling |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | 5 tentativas com backoff 1m/5m/30m/2h/12h, DLQ em tabela separada, replay manual por ADMIN | Política de retry com backoff e DLQ |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 em `X-Signature`, secret por endpoint, rotação com grace de 24 h | Autenticação HMAC-SHA256 com secret por endpoint |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | At-least-once com dedup por `X-Event-Id` do lado do cliente | Garantia at-least-once com `X-Event-Id` |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Módulo `src/modules/webhooks/` reaproveitando `AppError`, Pino, error middleware, Zod e `requireRole`; códigos `WEBHOOK_*` | Reuso dos padrões existentes do projeto |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Payload renderizado como snapshot na inserção da outbox | Decisão secundária |
| [ADR-008](ADR-008-filtragem-de-eventos-na-insercao.md) | Filtro por status assinado aplicado na inserção, não no envio | Decisão secundária |

## Ordem de leitura sugerida

ADR-001 → ADR-002 → ADR-003 → ADR-005 → ADR-004 → ADR-006 → ADR-007 → ADR-008.
