# Da Reunião ao Documento: Design Docs Gerados por IA — Processo de Produção

> Este README documenta **o processo** de produção do pacote de design docs. O enunciado original do desafio foi preservado em [`docs/DESAFIO.md`](docs/DESAFIO.md).
>
> **Status:** em construção — atualizado incrementalmente a cada documento concluído. Concluído até aqui: contextualização + ADRs + RFC + FDD (etapas 1–5).

## Sobre o desafio

Uma empresa que opera um Order Management System (OMS) em produção decidiu, em reunião técnica, construir um Sistema de Webhooks de Notificação de Pedidos para clientes B2B. Nada foi registrado além da transcrição da call ([`TRANSCRICAO.md`](TRANSCRICAO.md)). O desafio é transformar essa transcrição — cruzada com o código real da aplicação — em um pacote completo de design docs (PRD, RFC, FDD, ADRs e Tracker de rastreabilidade), em nível acionável para o time de engenharia começar a implementar.

A regra central: **nenhuma informação sem origem rastreável**. Tudo que entra nos documentos precisa existir na transcrição ou em algum arquivo do código. Identificar o que foi descartado ou adiado na reunião é tão importante quanto o que foi decidido — e a IA é a ferramenta principal de produção, com papel humano de maestro: dirigir, revisar criticamente e corrigir.

## Ferramentas de IA utilizadas

- **Claude Code (Anthropic)** — ferramenta principal: leitura do código-fonte, análise dirigida da transcrição, geração e revisão dos documentos, verificação de consistência (scripts para validar caminhos de código e links citados).
- **Plugin `adrs-management`** (marketplace FullCycle do curso) — usado na fase de contextualização.

## Workflow adotado

Segui a ordem sugerida no enunciado (ADRs → RFC → FDD → PRD), com um ajuste: **Tracker e este README são atualizados ao fim de cada fase**, e não só no final — para não perder o registro das iterações.

1. **Fork e setup** do repositório base.
2. **Contextualização** em duas frentes paralelas:
   - Mapeamento do codebase via plugin do curso (`/adr-map`);
   - Análise dirigida da transcrição (decisões fechadas com timestamp, alternativas descartadas, requisitos, itens fora de escopo, ganchos com o código).
3. **ADRs** (7, em [`docs/adrs/`](docs/adrs/)): gerados um a um a partir das decisões catalogadas na análise, em formato MADR, com revisão humana de cada arquivo (ver iterações abaixo).
4. **Tracker + README** atualizados com o conteúdo da fase.
5. **RFC** ([`docs/RFC.md`](docs/RFC.md)): proposta consolidada em altura de arquitetura — referencia os ADRs em vez de repetir o detalhe deles; alternativas descartadas e questões em aberto da reunião ganham suas seções naturais. Antes de escrever, releitura dos ADRs já revisados, para o RFC herdar a versão corrigida.
6. **FDD** ([`docs/FDD.md`](docs/FDD.md)): o "como construir" — modelo de dados, fluxos (outbox → worker → retry → DLQ), 7 endpoints com payloads, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção de integração com 10 arquivos reais do código. Convenção adotada: o que a reunião não definiu está marcado como *(definido neste FDD)* — separando decisão da reunião de proposta de design, para a revisão humana saber onde olhar.
7. *(Próximas fases: PRD → revisão final contra os critérios de aceite.)*

Interação com a IA: em vez de "gere os ADRs da transcrição", cada documento foi produzido com prompts dirigidos (abaixo), e cada dúvida de fidelidade foi resolvida voltando à transcrição — quando a IA registrava algo que a reunião não sustentava, o trecho era corrigido ou removido.

## Prompts customizados

Prompt de filtragem da transcrição (fase de contextualização) — o objetivo é forçar a separação entre decidido / descartado / adiado, que é onde a IA mais alucina:

```text
Leia TRANSCRICAO.md na íntegra e produza uma análise dirigida com:
1. Decisões FECHADAS na reunião — para cada uma: timestamp + falante da
   formalização, alternativas descartadas com o trade-off que motivou o
   descarte, e detalhes acordados;
2. Requisitos funcionais e não funcionais explícitos (com timestamp);
3. Itens explicitamente DESCARTADOS ou ADIADOS (não podem virar requisito);
4. Questões deixadas em aberto;
5. Ganchos com o código existente (arquivo real de src/ para cada um).
Não inclua NADA sem timestamp ou arquivo de origem identificável.
```

Prompt de geração de cada ADR (fase 3) — um por decisão, ancorado na análise anterior:

```text
Gere o ADR-00N para a decisão <D-N> da análise da transcrição, em formato
MADR com as seções: Status, Contexto, Decisão, Alternativas Consideradas
e Consequências (positivas e negativas, com trade-off explícito).
Regras: citar somente alternativas realmente discutidas na reunião;
referenciar apenas arquivos que existem no repositório (verificar antes
de citar); consequências devem ser DIFERENCIAIS — algo que a alternativa
descartada evitaria; detalhe de implementação fica para o FDD, não para
o ADR.
```

Prompt do mapeamento (adaptação do `/adr-map` do plugin, com restrições do desafio):

```text
Execute Phase 1: Create codebase mapping with --project-dir=<repo>
--output-dir=docs
Write the mapping file to docs/mapping.md (NOT docs/adrs/mapping.md — the
docs/adrs/ folder is reserved for ADR files only). Do not modify any file
under src/, prisma/, tests/ or the root README.md / TRANSCRICAO.md.
Write the mapping document in Brazilian Portuguese (pt-BR).
```

Prompt de double check (revisão de fidelidade) — criado depois que a IA, revisando um documento, "corrigiu" tom e estilo em vez de fatos (ver iteração 3):

```text
Faça um double check em <documento> contra a transcrição real e o código do
repo. Aponte SOMENTE erros concretos: decisão registrada errada (ex.:
"escolheram síncrono" quando escolheram outbox), afirmação tecnicamente
falsa (ex.: "arranjo de pastas melhora performance do banco"), timestamp ou
falante trocado, caminho de arquivo inexistente. NÃO sugira ajustes de tom,
estilo, parafraseio ou pontuação. Se não houver nada nesse nível, a resposta
é "está tudo certo, nada concreto encontrado".
```

## Iterações e ajustes

Registro dos principais momentos em que a saída da IA precisou de correção humana:

1. **O plugin do curso não servia para gerar estes ADRs.** O plano inicial era usar o fluxo completo `adr-map → adr-identify → adr-generate`. Ao inspecionar o plugin, ficou claro que as fases 2 e 3 extraem decisões da arquitetura **já implementada** no código — e as 6 decisões deste desafio existem apenas na transcrição (a feature ainda não existe). Ajuste de workflow: plugin mantido só no mapeamento; ADRs gerados de forma dirigida a partir da análise da transcrição. Sem esse ajuste, os ADRs sairiam sobre Prisma/JWT/Express em vez de outbox/retry/HMAC.
2. **Consequências sem lastro no ADR-006 (reuso de padrões).** A IA registrou dois trade-offs que a transcrição não sustentava: (a) "problema de performance no MySQL afeta API e worker juntos" — verdadeiro, mas consequência do outbox no MySQL (ADR-001), não do reuso de padrões: a alternativa descartada no ADR-006 não evitaria isso, então a consequência não era diferencial; (b) "polling sem framework de jobs" implicava que o time decidiu "fazer tudo à mão" — decisão que a reunião nunca tomou (o que existe é o descarte de Redis em [09:07] e o polling em [09:09]). Ambos corrigidos após questionamento, ficando só o que a transcrição sustenta ([09:29]: logging permanece Pino, "nada novo").
3. **A IA "corrigiu" o que não precisava de correção.** Num double check da análise da transcrição, a IA propôs e aplicou 3 ajustes puramente cosméticos (suavizar "ameaça migrar", ampliar um range de timestamp, pontuação dentro de uma citação) — zero erro concreto. Os 3 foram revertidos e a regra virou prompt fixo de revisão (acima): só fato errado conta; "nada encontrado" é resposta válida. No double check seguinte (TRACKER.md, 45 linhas), a regra funcionou: verificação de timestamps, falantes, caminhos e contagens, zero achado inventado.
4. **Fronteira "questão em aberto" × "adiado" no RFC.** Questionei a separação entre as 4 questões em aberto e os 2 itens de rodapé (email, dashboard) — por que só os últimos seriam "explicitamente adiados"? A revisão explicitou o critério: pergunta que ficou **sem resposta** na reunião é questão em aberto (ex.: rate limiting, que o próprio Diego pede para "registrar como ponto em aberto" em [09:39]); pedido que recebeu **"não" explícito** da tech lead é decisão de escopo fechada ([09:37] email, [09:40] dashboard) e pertence ao "Fora de escopo" do PRD. O RFC foi levemente editado após essa discussão.

## Como navegar a entrega

Ordem de leitura sugerida (estado atual):

1. [`docs/DESAFIO.md`](docs/DESAFIO.md) — enunciado original do desafio.
2. [`docs/analise-transcricao.md`](docs/analise-transcricao.md) — documento de trabalho: o que a reunião decidiu, descartou e deixou em aberto (com timestamps).
3. [`docs/RFC.md`](docs/RFC.md) — a proposta técnica: o que estamos propondo, o que foi descartado e o que segue em aberto.
4. [`docs/adrs/README.md`](docs/adrs/README.md) — índice dos ADRs, e os 7 ADRs (`ADR-001` a `ADR-007`).
5. [`docs/FDD.md`](docs/FDD.md) — como construir, em detalhe acionável (fluxos, contratos, erros, integração com o código).
6. [`docs/TRACKER.md`](docs/TRACKER.md) — rastreabilidade de cada item à transcrição ou ao código.

Pendente (próxima fase): `docs/PRD.md`.
