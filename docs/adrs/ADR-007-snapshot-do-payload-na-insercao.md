# ADR-007 — Snapshot do payload renderizado na inserção da outbox

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior), Bruno (Eng. Pleno, questionador)
- **Fonte primária:** `TRANSCRICAO.md` [09:51]–[09:52]

## Contexto

O evento gravado na `webhook_outbox` ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)) pode ser entregue segundos ou — em cenários de retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) — até ~15 horas depois da mudança de status. Nesse intervalo, o pedido pode ser alterado. Bruno levantou a questão: a outbox guarda o payload já renderizado, ou guarda só o `order_id` e renderiza na hora do envio?

O payload definido para o evento contém `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents` — sem items, para não inflar.

## Decisão

Gravar na outbox o **payload já renderizado no momento da inserção (snapshot)**. Se o pedido mudar depois, o evento continua refletindo o estado de quando o status mudou.

Como a inserção acontece dentro da transação de `changeStatus` em [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts), o snapshot captura exatamente o estado consistente da transação que gerou o evento.

## Alternativas Consideradas

### 1. Guardar só `order_id` e renderizar o payload na hora do envio — descartada
- Se o pedido mudar entre a inserção e o envio (ou entre retries), o evento entregue não corresponderia ao estado do momento da transição de status. Um evento `PAID → PROCESSING` entregue horas depois poderia carregar valores de um pedido já alterado.

## Consequências

### Positivas
- Fidelidade temporal: o evento é um registro imutável do estado no instante da transição, independente de quando for entregue (essencial com retries de até ~15h).
- Envio mais simples e barato: o worker não faz join/consulta na order para montar payload, apenas lê a linha da outbox e envia.
- O payload persistido serve de evidência para o histórico de entregas e para a DLQ.

### Negativas / trade-offs
- Linhas maiores na `webhook_outbox` (payload completo em vez de uma FK) — mitigado pelo payload enxuto sem items e pelo limite de 64KB.
- Se o formato do payload evoluir, eventos antigos ainda pendentes na outbox carregam o formato antigo — o consumidor da tabela precisa tolerar versões.

## Referências

- `TRANSCRICAO.md`: [09:43]–[09:44], [09:51]–[09:52]
- Código: [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts) (transação de `changeStatus`, onde o snapshot é gerado)
- Relacionados: [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)
