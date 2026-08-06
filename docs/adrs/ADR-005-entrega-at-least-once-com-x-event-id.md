# ADR-005 — Garantia de entrega at-least-once com deduplicação via `X-Event-Id`

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Diego (Eng. Sênior, proponente), Larissa (Tech Lead), Bruno (Eng. Pleno), Sofia (Eng. Segurança), Marcos (PM)
- **Fonte primária:** `TRANSCRICAO.md` [09:24]–[09:26]

## Contexto

Com outbox + worker + retry ([ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), há cenários em que o mesmo evento pode ser entregue mais de uma vez — por exemplo, o cliente recebe e responde, mas a confirmação se perde antes de o worker marcar o evento como entregue, e a entrega é repetida. É preciso definir qual semântica de entrega a plataforma garante e como o cliente lida com duplicatas.

## Decisão

- A plataforma garante **at-least-once**: o cliente pode receber o mesmo evento duas vezes e deve estar preparado.
- Cada evento carrega um **`event_id` (UUID) gerado quando entra na outbox**, enviado no header **`X-Event-Id`**, único por evento. O cliente **deduplica pelo `event_id`** do lado dele.
- A responsabilidade de dedup do lado do cliente será documentada com destaque no portal do desenvolvedor.

Formalizada por Larissa em [09:26]: *"At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."*

## Alternativas Consideradas

### 1. Garantia exactly-once — descartada
- Exigiria coordenação dos dois lados (plataforma e cliente) e ficaria muito mais complexo.
- At-least-once com `event_id` "resolve 99% dos casos" e é o padrão de mercado — Stripe e GitHub fazem assim.

## Consequências

### Positivas
- Modelo simples e robusto: o worker pode reentregar com segurança em qualquer cenário de dúvida, sem risco de perder evento.
- Alinhado ao padrão de mercado, o que reduz atrito de integração para os clientes B2B.
- O UUID por evento também serve de chave de rastreabilidade ponta a ponta (outbox → entrega → histórico de deliveries).

### Negativas / trade-offs
- Transfere responsabilidade para o cliente: quem não implementar dedup por `X-Event-Id` pode processar o mesmo evento duas vezes.
- Exige documentação externa clara (portal do desenvolvedor) como parte da entrega — a garantia só funciona bem se o cliente souber dela.

## Referências

- `TRANSCRICAO.md`: [09:24]–[09:26], [09:44] (header `X-Event-Id` no request)
- Relacionados: [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)
