# Architectural Decision Records

Este diretório armazena os ADRs (Architecture Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**, extraídos das decisões tomadas na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md).

Formato: MADR (variante), em arquivos `ADR-NNN-titulo-em-kebab-case.md`. Cada ADR contém Status, Contexto, Decisão, Alternativas Consideradas e Consequências, com referências de timestamp da transcrição e arquivos reais do código.

## Índice

| ADR | Decisão | Fonte principal |
| --- | --- | --- |
| [ADR-001](ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox no MySQL, na mesma transação do `changeStatus` | [09:03]–[09:08] |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado, polling de 2s, single-worker | [09:08]–[09:13] |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada | [09:14]–[09:18] |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace de 24h | [09:19]–[09:22] |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com dedup via `X-Event-Id` | [09:24]–[09:26] |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto (módulos, AppError, Pino, `WEBHOOK_*`) | [09:27]–[09:30] |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado na inserção da outbox | [09:51]–[09:52] |
