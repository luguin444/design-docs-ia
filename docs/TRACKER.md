# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos do pacote à sua origem: `TRANSCRICAO` (timestamp + falante) ou `CODIGO` (caminho de arquivo real).

> **Status:** em construção — cobre as fases concluídas (ADRs e RFC). Linhas de FDD e PRD serão adicionadas conforme cada documento for produzido.

## Convenções de ID

- `ADR-NNN` — decisão principal do ADR
- `ADR-NNN-ALT-XX` — alternativa considerada/descartada registrada no ADR
- `ADR-NNN-DET-XX` — detalhe, restrição ou requisito registrado no ADR
- `ADR-NNN-COD-XX` — referência/integração com o código existente registrada no ADR
- `RFC-PROP-XX` — proposta técnica do RFC
- `RFC-ALT-XX` — alternativa considerada/descartada no RFC
- `RFC-QA-XX` — questão em aberto registrada no RFC
- `RFC-IMP-XX` — impacto/risco/restrição registrado no RFC
- `RFC-COD-XX` — referência ao código existente registrada no RFC

## Tabela

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Outbox no MySQL: evento inserido na `webhook_outbox` na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Alternativa descartada | Disparo síncrono do webhook dentro do service de orders (travaria a transação; rollback impossível) | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Alternativa descartada | Redis Streams / fila externa (infra nova; overengineering para time pequeno) | TRANSCRICAO | [09:07] Diego |
| ADR-001-DET-01 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Detalhe técnico | Índices da outbox em status e created_at; worker lê pendentes em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-DET-02 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Restrição (fora de escopo) | Arquivamento de linhas entregues (~30 dias) fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| ADR-001-COD-01 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Integração com código | Transação de `changeStatus` (update order + history + estoque) recebe o insert da outbox | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em polling a cada 2s sobre a outbox | TRANSCRICAO | [09:10] Larissa |
| ADR-002-DET-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado da API (`src/worker.ts` + `npm run worker`) | TRANSCRICAO | [09:11] Diego |
| ADR-002-DET-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição técnica | PrismaClient é por processo: worker abre instância própria, mesmo banco | TRANSCRICAO | [09:30] Bruno |
| ADR-002-DET-03 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Limitação conhecida | Ordering garantida só por order_id e enquanto single-worker; sem garantia global | TRANSCRICAO | [09:13] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa descartada | Trigger no banco (MySQL não notifica processo externo; sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa adiada | Múltiplos workers (particionamento por order_id ou lock pessimista) — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| ADR-002-COD-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Integração com código | `src/worker.ts` espelha o padrão da entry-point existente da API | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff exponencial: 5 tentativas, 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DET-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela separada `webhook_dead_letter` (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| ADR-003-DET-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Requisito funcional | Replay manual via `POST /admin/webhooks/dead-letter/:id/replay`, recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| ADR-003-DET-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Requisito não funcional | Replay exige role ADMIN e log de auditoria de quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas (janela de ~30min insuficiente; houve indisponibilidade real de 2h) | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido com backoff (evento pendurado para sempre se cliente sumiu) | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | Marcar "failed" na própria outbox em vez de tabela DLQ separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-COD-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Integração com código | `requireRole` existente reaproveitado no endpoint admin de replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo do request, assinatura no header `X-Signature` | TRANSCRICAO | [09:22] Sofia |
| ADR-004-DET-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint; rotação via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-DET-02 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Requisito funcional | Secret gerada pela plataforma e devolvida na criação do webhook | TRANSCRICAO | [09:31] Marcos |
| ADR-004-DET-03 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Requisito não funcional | URL de webhook obrigatoriamente HTTPS, recusada com erro de validação (schema Zod) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-DET-04 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Requisito não funcional | Header `X-Timestamp` para detecção de replay attack pelo cliente | TRANSCRICAO | [09:44] Diego |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global da plataforma ("se vaza uma, vaza tudo") | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once; dedup pelo cliente via header `X-Event-Id` (UUID por evento) | TRANSCRICAO | [09:26] Larissa |
| ADR-005-DET-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Detalhe técnico | `event_id` UUID gerado quando o evento entra na outbox, único por evento | TRANSCRICAO | [09:25] Diego |
| ADR-005-DET-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Dependência externa | Dedup documentado com destaque no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once (coordenação dos dois lados, complexidade alta; at-least-once é padrão de mercado) | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes: módulo, AppError, Pino, error middleware, Zod, códigos de erro | TRANSCRICAO | [09:30] Larissa |
| ADR-006-DET-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Novo módulo `src/modules/webhooks` seguindo o padrão controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-DET-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Códigos de erro com prefixo `WEBHOOK_` (ex.: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL) | TRANSCRICAO | [09:29] Larissa |
| ADR-006-DET-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | IDs das novas tabelas em UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Classe base `AppError` (statusCode, errorCode, details) como modelo dos erros do módulo | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Classes de erro específicas (`InsufficientStockError`, `InvalidStatusTransitionError`) como padrão a seguir | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Error middleware centralizado já trata AppError/Zod/Prisma — captura os erros `WEBHOOK_*` sem mudança | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Logger Pino compartilhado (com redaction) reutilizado; nada novo de logging | CODIGO | src/shared/logger/index.ts |
| ADR-006-COD-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Módulo de orders como referência do padrão modular a replicar | CODIGO | src/modules/orders/ |
| ADR-006-COD-06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Validação por schemas Zod aplicados via middleware `validate` | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Outbox guarda o payload já renderizado no momento da inserção (snapshot) | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa descartada | Guardar só order_id e renderizar no envio (evento refletiria estado alterado do pedido) | TRANSCRICAO | [09:51] Bruno |
| ADR-007-DET-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Detalhe técnico | Payload enxuto: event_id, event_type, timestamp ISO 8601, order_id, order_number, from/to_status, customer_id, total_cents — sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-COD-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Integração com código | Snapshot gerado dentro da transação de `changeStatus`, capturando estado consistente | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Proposta técnica | Proposta consolidada: outbox + worker polling 2s + retry/DLQ + HMAC + at-least-once + reuso de padrões (resumo confirmado por todos) | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de orders (travaria a transação; rollback impossível) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Fila externa Redis Streams (infra nova; overengineering para time pequeno) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger no banco para acionar o worker (MySQL não notifica processo externo) | TRANSCRICAO | [09:09] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio: observar em produção e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers mantendo ordering (particionamento ou lock) — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Arquivamento de linhas entregues da outbox (~30 dias) fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Endurecimento futuro das permissões do CRUD de webhooks (hoje qualquer role autenticada) | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Única alteração em código existente: `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` na transação | TRANSCRICAO | [09:41] Bruno |
| RFC-IMP-02 | docs/RFC.md | Restrição (prazo) | Estimativa de 3 sprints incluindo revisão de segurança (2 dias úteis reservados) | TRANSCRICAO | [09:46] Larissa |
| RFC-COD-01 | docs/RFC.md | Integração com código | Transação de `changeStatus` como ponto de publicação do evento | CODIGO | src/modules/orders/order.service.ts |

## Cobertura atual

- **56 linhas** cobrindo os 7 ADRs e o RFC (decisões, alternativas, questões em aberto, impactos e referências de código)
- Fonte `TRANSCRICAO`: 45 linhas (~80%) — todas com timestamp no formato `[hh:mm] Nome`
- Fonte `CODIGO`: 11 linhas — todas com caminho de arquivo existente no repositório
