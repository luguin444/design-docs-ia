# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Metadados

- **Status:** Aceito
- **Data:** 2026-08-05 (decisão tomada em reunião técnica; ver `TRANSCRICAO.md`)
- **Decisores:** Sofia (Eng. Segurança, proponente), Larissa (Tech Lead), Diego (Eng. Sênior), Bruno (Eng. Pleno)
- **Fonte primária:** `TRANSCRICAO.md` [09:19]–[09:22]

## Contexto

Os webhooks expõem eventos com dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir validar duas coisas: que a requisição veio realmente da nossa plataforma e que o payload não foi adulterado no caminho. Há histórico real de cliente que vazou uma secret em log de aplicação, o que torna o raio de dano de um vazamento uma preocupação concreta.

## Decisão

- Assinar o **corpo do request com HMAC-SHA256**, enviando a assinatura no header `X-Signature`; o cliente verifica do lado dele com a secret compartilhada.
- **Secret única por endpoint de webhook** (não global da plataforma), armazenada na configuração do webhook junto de url, customer_id e estado ativo. Caso uma vaza secret vaze, não serão todos os webhooks afetados.
- **Secret rotacionável via API**: o cliente pode pedir nova secret; durante um **grace period de 24 horas** a antiga permanece válida em paralelo, para dar tempo de migração; depois disso, morre.
- A secret é **gerada pela plataforma** e devolvida na criação do webhook.
- Complementos de segurança decididos na mesma discussão (nível de requisito, não de ADR próprio): URL obrigatoriamente HTTPS, validada no schema Zod; header `X-Timestamp` para o cliente poder detectar replay attack.

Formalizada por Sofia em [09:22]: *"Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."*

## Alternativas Consideradas

### 1. Secret global da plataforma — descartada
- Desconsiderada porque se vazar uma, vazaria todas. Secret por endpoint limita o raio de dano de um vazamento a um único webhook — cenário que já ocorreu na prática.

### 2. Outros algoritmos de assinatura — descartados implicitamente
- HMAC-SHA256 é o padrão de mercado, logo todo cliente tem biblioteca pra isso. Adotar algo diferente aumentaria o custo de integração dos clientes sem ganho de segurança relevante.

## Consequências

### Positivas
- Autenticidade e integridade verificáveis pelo cliente com bibliotecas padrão (mesmo modelo de Stripe/GitHub).
- Vazamento de uma secret compromete apenas um endpoint, e a rotação com grace period permite troca sem janela de indisponibilidade.
- Sem estado de sessão ou handshake: assinatura por request, simples de implementar no worker.

### Negativas / trade-offs
- Gestão de ciclo de vida de secrets (geração, armazenamento, rotação, expiração da antiga após 24h) adiciona complexidade e superfície de auditoria.
- Durante o grace period de 24h existem duas secrets válidas por endpoint — a verificação e o armazenamento precisam contemplar isso.
- A segurança efetiva depende do cliente guardar bem a secret e validar a assinatura do lado dele (responsabilidade compartilhada).

## Referências

- `TRANSCRICAO.md`: [09:19]–[09:24], [09:31], [09:44], [09:46]
- Código: validação de URL https seguirá o padrão de schemas Zod dos módulos (ex.: [`src/modules/orders/order.schemas.ts`](../../src/modules/orders/order.schemas.ts)) aplicados via [`src/middlewares/validate.middleware.ts`](../../src/middlewares/validate.middleware.ts)
- Relacionados: [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
