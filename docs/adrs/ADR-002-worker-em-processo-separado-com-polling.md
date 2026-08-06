# ADR-002 — Worker em processo separado consumindo a outbox por polling

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM)
- **Fonte primária:** `TRANSCRICAO.md` [09:08]–[09:13], [09:29]–[09:30]

## Contexto

Com o outbox no MySQL decidido ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), é preciso definir **como** os eventos gravados na `webhook_outbox` são consumidos e entregues via HTTP. O requisito de produto é latência de notificação abaixo de 10 segundos. O MySQL não oferece mecanismo nativo de notificação a processos externos (não há equivalente ao NOTIFY/LISTEN do Postgres).

## Decisão

- O consumo será por **polling em loop: a cada 2 segundos**, o worker busca os eventos pendentes mais antigos, processa e marca como entregues.
- O worker roda como **processo separado da API** — se a API reinicia, o worker não morre junto. Nova entry-point `src/worker.ts` (espelhando o padrão de [`src/server.ts`](../../src/server.ts)) com script `npm run worker`.
- Mesmo banco e mesma stack (Prisma/MySQL), mas com **instância própria de `PrismaClient`**, já que PrismaClient é por processo ( limitação do prisma )
- **Single-worker** nesta fase: processamento em ordem de `created_at`, o que dá ordering implícita por `order_id`. Documentado como **limitação conhecida**: não há garantia de ordering global, e a garantia por pedido só vale enquanto houver um único worker. Os clientes não pediram ordering global.

## Alternativas Consideradas

### 1. Trigger no banco para reagir a novas linhas — descartada
- Trigger no MySQL só executa SQL; não notifica processo externo. Para avisar o worker seria preciso improvisar (escrever em arquivo, bater em endpoint), o que fica ruim.
- Polling de 2s já atende com folga o requisito de <10s.

### 2. Worker dentro do mesmo processo da API — descartada
- Se a API reinicia, perde o worker. Processos separados isolam falhas e ciclos de deploy., facilitando o requisito de <10s

### 3. Múltiplos workers em paralelo — adiada
- Escalar horizontalmente perderia a garantia de ordering por pedido; exigiria particionamento por `order_id` ou lock pessimista. Problema do futuro, não agora.

## Consequências

### Positivas
- Entrega desacoplada da API: cliente lento ou fora do ar não afeta o fluxo de mudança de status.
- Simplicidade operacional: sem broker, sem trigger, apenas mais um processo Node com a mesma stack.
- Latência máxima de detecção do evento ≈ 2s, confortável frente ao SLA de 10s.
- Todo estado fica no MySQL, se o worker morre e volta, ele só faz o próximo poll e retoma de onde parou.

### Negativas / trade-offs
- Latência mínima de ~2s no pior caso — não é push instantâneo. 
- Polling gera consultas constantes ao MySQL mesmo sem eventos pendentes (mitigado por índices em status e `created_at`.
- Single-worker é ponto único de processamento: throughput limitado e necessidade de decisão futura (particionamento/lock) para escalar sem quebrar ordering.
- Um processo a mais para operar, monitorar e manter vivo (deploy, restart, observabilidade).

## Referências

- `TRANSCRICAO.md`: [09:08]–[09:14], [09:29]–[09:30]
- Código: [`src/server.ts`](../../src/server.ts) (modelo de entry-point), [`src/config/database.ts`](../../src/config/database.ts) (instanciação do PrismaClient), [`package.json`](../../package.json) (novo script `npm run worker`)
- Relacionados: [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)
