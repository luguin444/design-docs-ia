# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Fonte primária:** `TRANSCRICAO.md` [09:14]–[09:18]

## Contexto

O endpoint do cliente pode estar indisponível no momento da entrega do webhook. Já houve caso real de cliente com indisponibilidade de duas horas em manutenção planejada. É preciso uma política que reentregue eventos sem deixá-los pendurados para sempre, e um destino claro para falhas permanentes. Falhas transitórias também incluem timeout: chamadas que não respondem em 10 segundos são tratadas como falha e marcadas para retry.

## Decisão

- **Retry com backoff exponencial, 5 tentativas**, com progressão **1m / 5m / 30m / 2h / 12h** — aproximadamente 15 horas entre a primeira falha e a última tentativa.
- Após esgotar as 5 tentativas, a falha é considerada permanente e o evento vai para uma **DLQ persistida em tabela separada `webhook_dead_letter`**, guardando payload, motivo da falha e timestamp.
- Reprocessamento é **manual**, via endpoint ADMIN `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. O endpoint exige role `ADMIN` (via `requireRole` existente em [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts)) e registra quem executou o replay, para auditoria.

## Alternativas Consideradas

### 1. Apenas 3 tentativas (mais agressivo) — descartada
- descartada porque 3 tentativas cobririam ~30 minutos — insuficiente para indisponibilidades reais já observadas de 2 horas.

### 2. Retry indefinido com backoff — descartada
- Traz o problema de evento pendurado para sempre se o cliente sumiu. Cinco tentativas cobrem uma janela satisfatória de 12–24h.

### 3. Marcar como "failed" na própria outbox, sem tabela DLQ separada — descartada
- A tabela separada deixa a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento.

## Consequências

### Positivas
- Tolerância a indisponibilidades longas do cliente (janela total ~15h) sem intervenção humana.
- Falha permanente tem destino explícito e auditável (`webhook_dead_letter` com payload, motivo e timestamp), facilitando debug e reprocessamento.
- Replay manual controlado e auditado (role `ADMIN` + log de quem executou).

### Negativas / trade-offs
- Evento pode levar até ~15h para ser desistido — durante esse período consome tentativas e monitoramento; não há notificação proativa ao cliente sobre falhas (email ficou para fase futura, [09:37] Larissa).
- Reprocessamento exige ação humana; DLQ pode acumular se ninguém observar (rate limiting/alertas ficaram como ponto em aberto, [09:38]–[09:39]).
- Mais uma tabela e mais estados para gerenciar no worker (pendente, processando, falhou, entregue + dead letter).

## Referências

- `TRANSCRICAO.md`: [09:14]–[09:18], [09:35]–[09:36], [09:42]
- Código: [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts) (`requireRole`), [`prisma/schema.prisma`](../../prisma/schema.prisma) (novas tabelas)
- Relacionados: [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)
