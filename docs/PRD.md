# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Marcos (Product Manager) |
| **Revisores** | Larissa (Tech Lead), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança) |
| **Status** | Aprovado (decisões confirmadas em reunião, [09:47]–[09:49]) |
| **Data** | 2026-08-05 |
| **Documentos relacionados** | [RFC](RFC.md) (proposta técnica) · [FDD](FDD.md) (especificação de implementação) · [ADRs](adrs/README.md) (decisões) · [TRACKER](TRACKER.md) (rastreabilidade) |

## 1. Resumo e contexto

O OMS (Order Management System) opera em produção com clientes B2B que acompanham seus pedidos pela API. Esta feature adiciona **webhooks outbound de notificação de mudança de status de pedido**: quando um pedido muda de status (PENDING → PAID → PROCESSING → SHIPPED → DELIVERED, ou CANCELLED), a plataforma envia um HTTP POST assinado para as URLs cadastradas pelo cliente, em até 10 segundos.

A demanda é um **pedido formal de três clientes B2B** — Atlas Comercial, MaxDistribuição e Nova Cargo — recebido na semana anterior à reunião de decisão ([09:00] Marcos). O escopo é exclusivamente outbound: a plataforma notifica; não recebe webhooks ([09:02]).

## 2. Problema e motivação

**Problema:** hoje os clientes descobrem mudanças de status "batendo no `GET /orders` de tempos em tempos", o que torna a integração deles **lenta e cara** ([09:00] Marcos). Não há nenhum mecanismo de notificação, evento ou fila na plataforma — o cliente que precisa reagir a um pedido enviado ou entregue fica refém do próprio polling.

**Motivação de negócio:**
- **Retenção:** a Atlas sugeriu que pode migrar para o concorrente se a funcionalidade não for entregue até o fim do trimestre ([09:00] Marcos). Prazo combinado: fim de novembro ([09:45]).
- **Qualidade de integração:** para os clientes, "tempo real" significa latência abaixo de 10 segundos — o importante é não ficar pendurado atualizando manualmente ([09:02] Marcos).
- **Custo de infraestrutura:** polling constante dos clientes gera carga desnecessária na API para ambos os lados.

## 3. Público-alvo e cenários de uso

**Público primário:** times de engenharia/integração dos clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo — e futuros clientes com o mesmo perfil), que consomem a API autenticados via JWT ([09:32]).

**Público secundário:** administradores internos da plataforma (role `ADMIN`), responsáveis por reprocessar entregas com falha permanente ([09:36]); e o time de produto, que documentará a integração no portal do desenvolvedor ([09:26], [09:40]).

**Cenários de uso:**

1. **Configurar notificações** — o integrador cadastra uma URL HTTPS e escolhe quais status quer ouvir (ex.: "só quero saber quando vira SHIPPED e DELIVERED"); recebe a secret para validar as assinaturas ([09:31]–[09:33]).
2. **Reagir a mudança de status** — um pedido do cliente muda de status; o sistema do cliente recebe o evento assinado em menos de 10s e dispara seu próprio fluxo (ex.: avisar o comprador, atualizar o ERP) ([09:02]).
3. **Sobreviver a indisponibilidade** — o sistema do cliente fica fora do ar por 2h em manutenção; ao voltar, recebe os eventos pendentes via retries automáticos ([09:16]).
4. **Auditar entregas** — o integrador consulta os últimos 100 envios (sucesso/falha, payload, resposta, tempo) para diagnosticar problemas de integração ([09:34]).
5. **Rotacionar credencial** — o integrador pede nova secret via API; tem 24h para migrar seus sistemas antes de a antiga expirar ([09:21]).
6. **Recuperar falha permanente** — um admin da plataforma reprocessa manualmente um evento que esgotou os retries, com registro de quem executou ([09:18], [09:36]).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| O1 | Notificar mudanças de status em "tempo real" na percepção do cliente | Tempo entre a transição de status e a entrega do webhook (medível via `webhook_deliveries`) | **< 10 segundos** no caminho feliz ([09:02]); latência mínima de polling aceita: 2s ([09:10]) |
| O2 | Entregar dentro da janela de retenção da Atlas | Data de disponibilização em produção | **Fim de novembro** ([09:45]), estimadas 3 sprints incluindo revisão de segurança ([09:46]) |
| O3 | Nenhum evento perdido silenciosamente | % de eventos de status assinados que terminam como entregues ou registrados na DLQ com evidência (payload + motivo) | **100%** — garantido por construção pela transação atômica ([09:40]) e pela DLQ ([09:18]) |
| O4 | Eliminar a dependência de polling dos clientes integrados | Volume de `GET /orders` de origem dos clientes com webhook ativo | Redução expressiva após adoção (baseline a medir; a reunião não fixou número) |

## 5. Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por customer, com filtro de status por endpoint ([09:31]–[09:33]).
- Entrega assinada (HMAC-SHA256) com headers de verificação e payload enxuto ([09:20], [09:43]–[09:44]).
- Retry automático com backoff (5 tentativas / ~15h) e DLQ com replay manual por admin ([09:15]–[09:18]).
- Histórico de entregas consultável (últimos 100 envios) ([09:34]).
- Rotação de secret via API com grace period de 24h ([09:21]).
- Garantia at-least-once com deduplicação via `X-Event-Id` ([09:24]–[09:26]).

### Fora de escopo (descartado ou adiado na reunião)

| Item | Decisão | Fonte |
| --- | --- | --- |
| Notificação por **email** quando o webhook falha repetidamente | Adiado — "próxima fase, depois que a gente medir o impacto" | [09:37] |
| **Dashboard/painel visual** para o cliente | Descartado nesta entrega — "só endpoints"; painel é projeto separado do frontend | [09:40] |
| **Rate limiting** de envio | Adiado — "observar e decidir depois" | [09:39] |
| **Webhooks inbound** (cliente → plataforma) | Fora de escopo — só outbound | [09:02] |
| Arquivamento de eventos entregues (~30 dias) | Fora do escopo da feature | [09:08] |
| Múltiplos workers / escala horizontal | Adiado — "problema do futuro" | [09:13] |
| Garantia de ordering global | Não requisitada pelos clientes; garantia apenas por pedido | [09:13]–[09:14] |

## 6. Requisitos funcionais

| # | Requisito | Fonte |
| --- | --- | --- |
| RF1 | Cadastrar webhook via `POST`: URL (HTTPS) + lista de status desejados; a secret é gerada pela plataforma e devolvida na criação; `customer_id` informado no body/path | [09:31] Marcos, [09:32] Larissa |
| RF2 | Editar (`PATCH`), remover (`DELETE`) e listar (`GET`) os webhooks de um customer | [09:33] Bruno |
| RF3 | Filtrar eventos por webhook: cada endpoint escolhe quais status quer receber; o filtro é aplicado na inserção da outbox | [09:33]–[09:34] |
| RF4 | Notificar mudança de status via HTTP POST assinado, em até 10s | [09:02], [09:20] |
| RF5 | Retentar entregas com falha automaticamente (backoff 1m/5m/30m/2h/12h, 5 tentativas) e mover falhas permanentes para a DLQ | [09:15]–[09:18] |
| RF6 | Consultar histórico de entregas: últimos 100 envios com sucesso/falha, payload, resposta e tempo de resposta | [09:34] Marcos |
| RF7 | Reprocessar evento da DLQ manualmente via endpoint admin (role `ADMIN`, com auditoria de quem executou) | [09:18], [09:35]–[09:36] |
| RF8 | Rotacionar secret via API, mantendo a antiga válida por 24h | [09:21] Sofia |
| RF9 | Identificar cada evento com UUID único (`X-Event-Id`) para deduplicação pelo cliente | [09:25] Diego |

## 7. Requisitos não funcionais

| # | Requisito | Fonte |
| --- | --- | --- |
| RNF1 | Latência de notificação < 10s (SLA de percepção de tempo real) | [09:02] |
| RNF2 | Consistência absoluta: status mudou ⇔ evento registrado (mesma transação; rollback conjunto) | [09:06], [09:40] |
| RNF3 | Autenticidade e integridade verificáveis: HMAC-SHA256 sobre o corpo, secret única por endpoint | [09:20]–[09:22] |
| RNF4 | Transporte exclusivamente HTTPS | [09:23] |
| RNF5 | Payload limitado a 64KB — acima disso, erro (não trunca) | [09:23]–[09:24] |
| RNF6 | Timeout de entrega de 10s por tentativa | [09:42] |
| RNF7 | Semântica at-least-once (duplicatas possíveis; dedup é responsabilidade do cliente, documentada no portal) | [09:24]–[09:26] |
| RNF8 | Ordering garantida por pedido (não global), enquanto single-worker — limitação documentada | [09:12]–[09:14] |

## 8. Decisões e trade-offs principais

Visão de produto das decisões técnicas (detalhe e alternativas nos ADRs):

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Outbox no MySQL existente, na transação da mudança de status | Latência atrelada a polling em vez de push; crescimento de tabela a gerenciar depois | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) |
| Worker separado em polling de 2s, single-worker | Até 2s de latência adicional; throughput limitado a um processo | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| 5 retries com backoff (~15h) e DLQ com replay manual | Sem notificação proativa ao cliente sobre falhas nesta fase | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint e rotação com grace de 24h | Gestão de ciclo de vida de secrets; janela com duas secrets válidas | [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com dedup pelo cliente | Transfere responsabilidade de dedup ao integrador | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso integral dos padrões do projeto | Herda limitações da stack atual (sem métricas/tracing prontos) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Payload snapshot no momento da transição | Linhas maiores na outbox; versões antigas de payload podem coexistir | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

## 9. Dependências

**Técnicas (internas):**
- Módulo de pedidos existente — a publicação do evento vive dentro da transação de `changeStatus` (`src/modules/orders/order.service.ts`).
- MySQL/Prisma e autenticação JWT/roles já existentes; nenhuma infraestrutura ou dependência nova ([09:07], [09:30]).

**Organizacionais:**
- **Portal do desenvolvedor:** documentação de integração para os clientes, incluindo o destaque sobre deduplicação — compromisso do PM ([09:26], [09:40]).
- **Revisão de segurança:** 2 dias úteis da Sofia antes do deploy, focada em HMAC e geração de secret ([09:46]).
- **Do lado do cliente:** implementar endpoint HTTPS, validação da assinatura e dedup por `X-Event-Id` — sem isso, a integração não se completa ([09:25]).
- **Comunicação de prazo:** confirmação com a Atlas e demais clientes ([09:47]).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Perder a Atlas se a entrega passar do fim de novembro ([09:00], [09:45]) | Média | Alto | Estimativa de 3 sprints já inclui a revisão de segurança ([09:46]–[09:47]); escopo enxuto com adiamentos explícitos (email, dashboard, rate limiting); PM confirma prazo com o cliente ([09:47]) |
| Cliente não implementa dedup e processa eventos duplicados (at-least-once) | Média | Médio | Documentação destacada no portal do desenvolvedor ([09:26]); `X-Event-Id` em todo evento; padrão de mercado conhecido (Stripe/GitHub, [09:25]) |
| Cliente indisponível por mais de ~15h perde a entrega automática | Baixa | Médio | DLQ preserva o evento com evidência; replay manual por admin ([09:18]); histórico de deliveries para diagnóstico ([09:34]) |
| Vazamento de secret por parte de um cliente (há precedente, [09:22]) | Baixa | Alto | Secret única por endpoint limita o raio de dano ([09:21]); rotação com grace de 24h; revisão de segurança dedicada ([09:46]) |
| Acúmulo silencioso na DLQ (replay é manual e não há alerta nesta fase, [09:37]) | Média | Médio | Logs estruturados e métrica de tamanho da DLQ (FDD §9); alerta por email avaliado na próxima fase |

## 11. Critérios de aceitação

- Integrador cadastra, edita, remove e lista webhooks via API autenticada; a secret é exibida apenas na criação e na rotação.
- Mudança de status de pedido com webhook assinante gera notificação entregue em < 10s (cliente saudável), com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
- O integrador consegue validar a assinatura HMAC-SHA256 com a secret compartilhada.
- Webhook recebe somente os status que assinou.
- URL `http://` é recusada no cadastro com erro de validação.
- Indisponibilidade temporária do cliente não perde eventos: retries automáticos na progressão 1m/5m/30m/2h/12h.
- Após 5 falhas, o evento está na DLQ com payload, motivo e timestamp; admin (e somente admin) consegue reprocessá-lo, com registro de autoria.
- `GET /webhooks/:id/deliveries` mostra os últimos 100 envios com sucesso/falha, payload, resposta e tempo.
- Rotação de secret devolve a nova e mantém a antiga válida por exatamente 24h.
- Rollback da transação de status não deixa evento órfão (e vice-versa: commit garante o evento).

## 12. Estratégia de testes e validação

**Testes (padrão existente do projeto — Vitest + Supertest, integração ponta a ponta contra MySQL real, como em `tests/orders.test.ts`):**
- **API de configuração:** CRUD completo, validações (URL http, status inválido), controle de acesso (CRUD autenticado, replay só ADMIN).
- **Transação:** falha forçada na inserção da outbox deve reverter a mudança de status; commit deve criar exatamente um evento por webhook assinante (critérios técnicos detalhados no [FDD §12](FDD.md)).
- **Worker:** entrega com sucesso, retry na progressão correta, movimentação para DLQ na 5ª falha, replay recolocando como pendente.
- **Ponta a ponta:** a estimativa da Larissa reservou explicitamente meio sprint para "integração no order.service e testes ponta a ponta" ([09:46]).

**Validação:**
- **Revisão de segurança** da Sofia (2 dias úteis) sobre HMAC e geração/rotação de secret, antes do deploy ([09:46], [09:49]).
- **Sessão de revisão do design** com Bruno e Diego antes de começar a codar ([09:50] Larissa).
- **Validação com clientes:** documentação no portal do desenvolvedor e confirmação de prazo/expectativas com Atlas, MaxDistribuição e Nova Cargo ([09:40], [09:47]).
- **Pós-lançamento:** acompanhar as métricas dos objetivos (latência de entrega, DLQ, volume de polling remanescente) para decidir os itens adiados — email de alerta e rate limiting ([09:37]–[09:39]).
