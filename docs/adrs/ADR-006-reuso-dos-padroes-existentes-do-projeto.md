# ADR-006 — Reuso dos padrões existentes do projeto para o módulo de webhooks

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Bruno (Eng. Pleno, proponente), Larissa (Tech Lead), Diego (Eng. Sênior)
- **Fonte primária:** `TRANSCRICAO.md` [09:27]–[09:30]

## Contexto

A codebase do OMS tem padrões consolidados e consistentes entre os módulos. A feature de webhooks poderia introduzir estruturas, bibliotecas ou convenções próprias — ou aderir integralmente ao que já existe. Os padrões relevantes no código:

- **Estrutura modular:** cada domínio é um módulo em `src/modules/` com `controller`, `service`, `repository`, `routes` e `schemas` (ex.: [`src/modules/orders/`](../../src/modules/orders), [`src/modules/customers/`](../../src/modules/customers)).
- **Erros:** classe base [`AppError`](../../src/shared/errors/app-error.ts) com `statusCode`, `errorCode` e `details`; classes específicas como `InsufficientStockError` e `InvalidStatusTransitionError` em [`src/shared/errors/http-errors.ts`](../../src/shared/errors/http-errors.ts), com códigos no padrão `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`.
- **Error handling centralizado:** [`src/middlewares/error.middleware.ts`](../../src/middlewares/error.middleware.ts) já trata `AppError`, `ZodError` e erros do Prisma.
- **Logging:** logger Pino compartilhado em [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts) (com redaction de campos sensíveis).
- **Autorização:** `requireRole` em [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts).
- **Validação:** schemas Zod por módulo, aplicados via [`src/middlewares/validate.middleware.ts`](../../src/middlewares/validate.middleware.ts).
- **IDs:** UUID em todas as tabelas ([`prisma/schema.prisma`](../../prisma/schema.prisma)).

## Decisão

**Reuso máximo do que já existe**: o webhook entra como um módulo igual aos outros, sem introduzir nenhuma biblioteca ou convenção nova.

- Novo módulo `src/modules/webhooks/` com controller, service, repository, routes e schemas.
- Worker: entry-point `src/worker.ts` separada, com a lógica de processamento dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`).
- Erros do módulo seguem `AppError`/classes específicas, com **códigos prefixados `WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
- Logger Pino existente; nada novo de logging.
- Error middleware centralizado permanece intocado — já captura os novos erros por herdarem de `AppError`.
- Endpoint admin de replay reaproveita o `requireRole` existente.
- Novas tabelas seguem o padrão UUID do projeto.

## Alternativas Consideradas

### 1. Estrutura/stack própria para o módulo de webhooks — descartada
Introduzir bibliotecas novas (logger próprio, framework de jobs, camada de erros distinta) ou organizar o código fora do padrão modular. Descartada porque quebraria a consistência da codebase, aumentaria a curva de aprendizado e criaria manutenção duplicada — enquanto a infraestrutura existente (AppError + error middleware + Pino + Zod) já cobre todas as necessidades do módulo sem alteração.

## Consequências

### Positivas
- Consistência: qualquer pessoa que conhece os módulos existentes navega o módulo de webhooks sem fricção.
- Zero mudança em infraestrutura compartilhada: o error middleware trata os erros `WEBHOOK_*` automaticamente por herança de `AppError`.
- Menos código novo para revisar na janela apertada de 3 sprints.

### Negativas / trade-offs
- O módulo herda também as limitações dos padrões atuais (ex.: logs Pino etc ).

## Referências

- `TRANSCRICAO.md`: [09:27]–[09:30], [09:36], [09:51]
- Código: [`src/modules/orders/`](../../src/modules/orders) (módulo de referência), [`src/shared/errors/app-error.ts`](../../src/shared/errors/app-error.ts), [`src/shared/errors/http-errors.ts`](../../src/shared/errors/http-errors.ts), [`src/middlewares/error.middleware.ts`](../../src/middlewares/error.middleware.ts), [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts), [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts), [`src/middlewares/validate.middleware.ts`](../../src/middlewares/validate.middleware.ts), [`prisma/schema.prisma`](../../prisma/schema.prisma)
- Relacionados: [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)
