# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Fonte primária:** `TRANSCRICAO.md` [09:03]–[09:08]

## Contexto

Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser notificados quando o status de seus pedidos muda, com latência percebida abaixo de 10 segundos. O OMS não possui hoje nenhum mecanismo de eventos, filas ou webhooks.

A mudança de status de pedido acontece dentro de uma transação já pesada no método `changeStatus` de [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts): atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos. Qualquer mecanismo de notificação precisa garantir consistência com essa transação: **se o status mudou, o evento tem que existir; se houve rollback, o evento não pode existir**.

## Decisão

Adotar o **padrão Outbox transacional sobre o MySQL já existente**:

- Na mesma transação SQL que atualiza `orders` e `order_status_history`, inserir uma linha na nova tabela `webhook_outbox` com o evento.
- Um worker separado lê a tabela e dispara as chamadas HTTP (detalhado no [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- A tabela terá índices no campo de status do evento (pendente, processando, falhou, entregue) e em `created_at`; o worker lê apenas pendentes, em batches pequenos.
- O evento é gravado com payload já renderizado (snapshot) — ver [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md).

## Alternativas Consideradas

### 1. Disparo síncrono dentro do service de orders — descartada
Fazer a chamada HTTP ao cliente dentro do próprio `changeStatus`.
- Acrescentaria um HTTP call no meio de uma transação que já é pesada; um cliente lento travaria mudanças de status de outros pedidos.
- Se o cliente estiver fora do ar, não há resposta razoável, não faz sentido dar rollback na mudança de status por falha de notificação.

### 2. Fila externa (Redis Streams ou similar) — descartada
- Exigiria subir e operar infraestrutura nova; o time é pequeno e um Redis Cluster para isso seria overengineering.
- O outbox no MySQL existente resolve o problema com a infraestrutura atual.

## Consequências

### Positivas
- **Consistência garantida por construção:** commit da transação principal ⇒ evento registrado; rollback ⇒ evento some junto. "Não tem inconsistência possível" ([09:06] Diego).
- Nenhuma infraestrutura nova para operar; reutiliza MySQL + Prisma já presentes ([`prisma/schema.prisma`](../../prisma/schema.prisma)).
- Tabela outbox serve de registro natural do que foi/será enviado, apoiando o histórico de entregas.

### Negativas / trade-offs
- A entrega passa a depender de um worker de polling — latência mínima da notificação fica atrelada ao intervalo de polling (2s, ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)), em vez de push imediato.
- A tabela `webhook_outbox` cresce continuamente; arquivamento de linhas entregues (~30 dias) ficou explicitamente **fora do escopo** desta feature e precisará ser tratado depois.
- Acrescenta um insert à transação de `changeStatus`, que já é pesada, custo aceito por ser muito menor que um HTTP call.

## Referências

- `TRANSCRICAO.md`: [09:03]–[09:08], [09:40]–[09:41]
- Código: [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts) (método `changeStatus`), [`prisma/schema.prisma`](../../prisma/schema.prisma)
- Relacionados: [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)
