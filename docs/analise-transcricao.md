# Análise da Transcrição — Sistema de Webhooks de Notificação de Pedidos

> Documento de trabalho (etapa de contextualização). Extrai da `TRANSCRICAO.md` as decisões fechadas, requisitos, itens descartados/adiados e ganchos com o código existente, sempre com timestamp e falante. Serve de insumo para ADRs, RFC, FDD, PRD e Tracker.

## 1. Contexto de negócio

| Item | Fonte |
| --- | --- |
| Pedido formal de 3 clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo, que querem notificação em tempo real de mudança de status de pedidos | [09:00] Marcos |
| Hoje os clientes fazem polling no `GET /orders`, integração lenta e cara | [09:00] Marcos |
| Risco de churn: Atlas ameaça migrar para concorrente se não entregar até o fim do trimestre | [09:00] Marcos |
| "Tempo real" para os clientes = latência abaixo de 10 segundos | [09:02] Marcos |
| Escopo é somente outbound (nós → cliente); inbound não entra | [09:02] Sofia / Marcos |
| Prazo: Atlas quer para fim de novembro; estimativa de 3 sprints incluindo revisão de segurança | [09:45] Marcos / [09:46]–[09:47] Larissa |
| Reservar pelo menos 2 dias úteis para revisão de segurança (HMAC e geração de secret) antes do deploy | [09:46] Sofia |

## 2. Decisões fechadas (candidatas a ADR)

### D1 — Padrão Outbox no MySQL
- **Decisão:** inserir o evento numa tabela `webhook_outbox` **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history`. Worker separado lê e dispara HTTP. — [09:06] Diego; formalizada em [09:08] Larissa ("Tá decidido então: outbox em MySQL").
- **Alternativas descartadas:**
  - Disparo síncrono no service de orders — travaria a transação (que já atualiza order, history e estoque) com HTTP call; cliente fora do ar não pode causar rollback de status. [09:03] Larissa (pergunta), [09:04] Bruno (contra), [09:06] Diego ("fora de questão").
  - Redis Streams / fila externa — exigiria subir mais infra; time pequeno, overengineering. [09:07] Larissa / Diego.
- **Detalhes:** índices em status (pendente, processando, falhou, entregue) e `created_at`; worker lê pendentes em batch pequeno. [09:08] Diego.

### D2 — Worker em processo separado, polling de 2s
- **Decisão:** worker roda como processo separado da API, em polling a cada 2 segundos. — [09:09] Diego; [09:10] Larissa ("Vamos registrar isso como uma decisão"); [09:11] Diego (processo separado).
- **Alternativas descartadas:** trigger do banco — MySQL não tem NOTIFY/LISTEN nativo como Postgres; trigger só executa SQL, não notifica processo externo. [09:09] Bruno (pergunta) / Diego (descarte).
- **Detalhes:** latência mínima de 2s no pior caso, aceita [09:10]; entry-point nova `src/worker.ts` + script `npm run worker`, espelhando `src/server.ts` [09:11] Larissa; mesmo banco/mesma stack, PrismaClient **separado** por processo [09:29]–[09:30] Diego/Bruno.
- **Ordering (limitação conhecida):** single-worker processa em ordem de `created_at`; ordering garantida só por `order_id` e enquanto for single-worker; escala futura via particionamento por `order_id` ou lock pessimista ("problema do futuro"). [09:12]–[09:13] Diego, [09:13] Larissa, [09:14] Marcos.

### D3 — Retry com backoff exponencial + DLQ
- **Decisão:** 5 tentativas com backoff 1m / 5m / 30m / 2h / 12h (~15h entre primeira falha e última tentativa); após esgotar, evento vai para DLQ em tabela separada `webhook_dead_letter` (payload, motivo da falha, timestamp). — [09:15]–[09:17] Diego/Larissa ("Decidido"), [09:18] Diego (tabela separada).
- **Alternativas descartadas:**
  - 3 tentativas — pouco; cliente já teve indisponibilidade de 2h em manutenção planejada. [09:16] Bruno (proposta) / Diego (descarte).
  - Retry indefinido com backoff — evento pendurado para sempre se o cliente sumiu. [09:15] Diego.
  - Marcar "failed" na própria outbox (em vez de tabela DLQ separada) — leitura da outbox fica menos limpa; tabela separada serve de evidência para debug/reprocessamento. [09:17] Larissa (pergunta) / [09:18] Diego.
- **Reprocessamento:** manual, via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, recoloca na outbox como pendente. [09:18] Diego.

### D4 — HMAC-SHA256 com secret por endpoint e rotação
- **Decisão:** assinar o corpo do request com HMAC-SHA256, assinatura no header `X-Signature`; secret única **por endpoint** de webhook; rotação de secret via API com grace period de 24h (antiga válida em paralelo). — [09:20]–[09:22] Sofia ("Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h").
- **Alternativas descartadas:** secret global da plataforma — se vaza uma, vaza tudo. [09:21] Sofia. (Contexto: já houve cliente que vazou secret em log [09:22] Diego.)

### D5 — Entrega at-least-once com `X-Event-Id`
- **Decisão:** garantia at-least-once; cliente pode receber evento duplicado e deduplica pelo header `X-Event-Id` (UUID gerado quando o evento entra na outbox). — [09:24]–[09:25] Diego; [09:26] Larissa ("Decisão").
- **Alternativas descartadas:** exactly-once — exigiria coordenação dos dois lados, muito mais complexo; at-least-once + event_id é padrão de mercado (Stripe, GitHub). [09:25] Diego.
- **Nota:** joga responsabilidade de dedup para o cliente [09:25] Sofia; Marcos documenta destacado no portal do desenvolvedor [09:26].

### D6 — Reuso dos padrões existentes do projeto
- **Decisão:** "reuso máximo do que já existe: AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros." — [09:30] Larissa.
- **Detalhes:**
  - Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas. [09:27] Bruno.
  - Worker: entry `src/worker.ts` + lógica em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`. [09:28] Bruno.
  - Erros: seguir `AppError` e classes específicas; códigos com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). [09:28] Bruno, [09:29] Larissa.
  - Logger Pino já presente; error middleware centralizado já trata AppError, Zod e Prisma — sem mudanças. [09:29] Bruno.
  - Replay admin reaproveita o `requireRole` existente. [09:36] Larissa.
  - IDs em UUID, seguindo o padrão do projeto. [09:51] Diego/Larissa.

### D7 — Snapshot do payload na inserção
- **Decisão:** a outbox guarda o payload **já renderizado** no momento da inserção (snapshot), não apenas o `order_id`. Se o pedido mudar depois, o evento reflete o estado de quando o status mudou. — [09:51] Bruno (pergunta), [09:52] Larissa / Diego / Bruno ("Decidido").
- **Alternativa descartada:** guardar só `order_id` e renderizar na hora do envio — "caso esquisito" se o pedido mudar entre inserção e envio. [09:52] Larissa.

### Decisões secundárias (nível FDD)
| Decisão | Fonte |
| --- | --- |
| Filtro de eventos aplicado **na inserção** da outbox (se nenhum webhook do customer quer aquele status, nem insere) | [09:33]–[09:34] Marcos/Bruno/Diego |
| Timeout do HTTP call do worker: 10 segundos → falha e marca para retry | [09:42] Sofia/Diego |
| Payload JSON: `event_id`, `event_type` ("order.status_changed"), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, campos básicos da order (ex.: `total_cents`); **sem items** (payload enxuto; detalhes via `GET /orders/:id`) | [09:43] Diego, [09:44] Bruno |
| Headers: `X-Event-Id`, `X-Signature`, `X-Timestamp` (detecção de replay attack), `X-Webhook-Id` (sugestão da Sofia), `Content-Type: application/json` | [09:44] Diego/Sofia |
| URL do webhook deve ser HTTPS; http recusado com erro de validação (schema Zod) | [09:23] Sofia |
| Limite de payload 64KB; acima disso, erro (não trunca) | [09:23] Sofia / [09:24] Diego/Larissa |
| Secret gerada pela plataforma e devolvida na criação do webhook | [09:31] Marcos |
| `customer_id` vem no body ou path (não do JWT — JWT é do usuário operador) | [09:32] Bruno/Marcos/Larissa |
| CRUD de configuração: qualquer role autenticada; replay de DLQ: role ADMIN + log de auditoria de quem fez | [09:35]–[09:37] Sofia/Larissa/Marcos |

## 3. Requisitos funcionais (dump do PM + discussão)

| # | Requisito | Fonte |
| --- | --- | --- |
| RF1 | Cadastrar webhook: `POST` com url e lista de status desejados; secret gerada pela plataforma e devolvida na criação; `customer_id` no body/path | [09:31] Marcos, [09:32] Larissa |
| RF2 | Editar webhook (`PATCH`), remover (`DELETE`), listar webhooks de um customer (`GET`) | [09:33] Bruno |
| RF3 | Filtro de eventos por webhook: lista de status que quer ouvir; filtragem na inserção da outbox | [09:33]–[09:34] Marcos/Bruno |
| RF4 | Histórico de entregas: `GET /webhooks/:id/deliveries` — últimos 100 envios com sucesso/falha, payload, response e tempo de resposta | [09:34] Marcos |
| RF5 | Replay manual de DLQ: `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN, auditado) | [09:18] Diego, [09:35]–[09:36] |
| RF6 | Rotação de secret via API, com grace period de 24h para a secret antiga | [09:21] Sofia |
| RF7 | Notificar mudança de status de pedido em até 10s (polling 2s) | [09:02] Marcos, [09:09] Diego |
| RF8 | Assinatura HMAC-SHA256 no header `X-Signature` + headers `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` | [09:20] Sofia, [09:44] Diego/Sofia |
| RF9 | Retry automático com backoff e movimentação para DLQ após 5 falhas | [09:15]–[09:17] Diego |

## 4. Requisitos não funcionais

| # | Requisito | Fonte |
| --- | --- | --- |
| RNF1 | Latência de notificação < 10s (percepção de "tempo real") | [09:02] Marcos |
| RNF2 | Atomicidade: status mudou ⇔ evento registrado (mesma transação; rollback conjunto) | [09:06] Diego, [09:40]–[09:41] Bruno/Diego |
| RNF3 | TLS obrigatório (somente URLs https) | [09:23] Sofia |
| RNF4 | Limite de payload: 64KB, erro se ultrapassar | [09:23]–[09:24] |
| RNF5 | Timeout de entrega: 10s | [09:42] Diego |
| RNF6 | Entrega at-least-once; dedup pelo cliente via `X-Event-Id` | [09:24]–[09:26] |
| RNF7 | Ordering garantida apenas por `order_id` e enquanto single-worker (limitação documentada) | [09:12]–[09:13] |
| RNF8 | Secrets únicas por endpoint, rotacionáveis (grace 24h) | [09:21] Sofia |
| RNF9 | Auditoria de replay admin (logar quem executou) | [09:36] Sofia |

## 5. Fora de escopo / adiado (NÃO documentar como requisito)

| Item | Status | Fonte |
| --- | --- | --- |
| Notificação por email quando webhook falha repetidamente | Adiado — "próxima fase, depois que a gente medir o impacto" | [09:37] Marcos (pedido) / Larissa (descarte) |
| Rate limiting de envio para o cliente | Em aberto — "observar e decidir depois" | [09:38]–[09:39] Diego/Larissa |
| Dashboard/painel visual para o cliente | Fora de escopo — projeto separado do time de frontend | [09:39]–[09:40] Marcos/Larissa |
| Webhooks inbound (cliente → plataforma) | Fora de escopo — só outbound | [09:02] Sofia/Marcos |
| Arquivamento de linhas entregues da outbox (~30 dias) | Fora do escopo da feature | [09:08] Diego |
| Escala para múltiplos workers (particionamento/lock) | Adiado — "problema do futuro" | [09:13] Diego |
| Endurecer roles do CRUD de webhooks | Adiado — "mais pra frente" | [09:36]–[09:37] Marcos/Sofia |
| Garantia de ordering global | Não requisitada pelos clientes | [09:14] Marcos |

## 6. Questões em aberto (candidatas à seção do RFC)

1. **Rate limiting de saída** — cliente com 50 mudanças de status/minuto seria bombardeado; decidiu-se observar em produção e decidir depois. [09:38]–[09:39]
2. **Escala do worker** — como escalar para múltiplos workers mantendo ordering (particionamento por `order_id` ou lock pessimista); adiado. [09:13]
3. **Arquivamento/retenção da outbox** — arquivar linhas entregues após ~30 dias; fora do escopo desta feature. [09:08]
4. **Endurecimento de permissões do CRUD** — hoje qualquer role autenticada; pode ser restringido no futuro. [09:37]

## 7. Ganchos com o código existente

| Ponto de integração | Arquivo real | Relação com a feature |
| --- | --- | --- |
| Transação de `changeStatus` (update order + insert history + estoque) — ponto onde entra o insert na `webhook_outbox` via `publishWebhookEvent(tx, order, fromStatus, toStatus)` | `src/modules/orders/order.service.ts` (método `changeStatus`) | [09:40]–[09:41] Bruno/Diego |
| Máquina de estados do pedido (status PENDING→…→DELIVERED/CANCELLED que geram eventos) | `src/modules/orders/order.status.ts` | [09:12] (sequência PAID/PROCESSING/SHIPPED) |
| Classe base de erro + classes específicas (`InsufficientStockError`, `InvalidStatusTransitionError`) — modelo para erros `WEBHOOK_*` | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | [09:28] Bruno |
| Error middleware centralizado (trata AppError, Zod e Prisma) — pega os erros do módulo sem mudança | `src/middlewares/error.middleware.ts` | [09:29] Bruno |
| `requireRole` para o endpoint admin de replay | `src/middlewares/auth.middleware.ts` | [09:36] Larissa |
| Logger Pino compartilhado | `src/shared/logger/index.ts` | [09:29] Bruno |
| Entry-point da API, modelo para `src/worker.ts` | `src/server.ts` | [09:11] Larissa |
| Modelos Prisma / MySQL (padrão UUID, tabelas novas `webhook_outbox`, `webhook_dead_letter`, configuração de webhooks) | `prisma/schema.prisma` | [09:51] (UUID), [09:06] (outbox) |
| Padrão de módulos (controller, service, repository, routes, schemas) a replicar em `src/modules/webhooks/` | `src/modules/orders/`, `src/modules/customers/`, … | [09:27] Bruno |
| Validação Zod (ex.: recusar URL http) | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` dos módulos | [09:23] Sofia |
