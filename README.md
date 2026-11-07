# Do Transcript à Documentação Técnica: Design Docs do Sistema de Webhooks

## Sobre o desafio

A tarefa foi transformar uma reunião técnica de 55 minutos em um pacote de design docs acionável para o time de engenharia. A reunião já tinha decidido tudo sobre um Sistema de Webhooks de Notificação de Pedidos, mas a única evidência da decisão era a transcrição literal da call. Meu trabalho foi reconstruir a decisão a partir dessa transcrição, cruzar com o código real de um Order Management System em Node.js, TypeScript, Prisma e MySQL, e produzir PRD, RFC, FDD, ADRs, tracker de rastreabilidade e este README.

O desafio tem uma regra que muda a natureza do trabalho: nada pode ser inventado. Todo requisito, decisão ou restrição precisa apontar para um timestamp da transcrição ou para um caminho de arquivo do repositório. Isso desloca o esforço de "escrever bem" para "filtrar bem": a parte difícil não foi gerar texto, foi decidir o que **não** entra, identificar o que foi descartado, o que foi adiado e o que foi apenas mencionado de passagem sem virar requisito. A entrega é puramente documental, nenhum arquivo de `src/`, `prisma/` ou `tests/` foi alterado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| Hermes Agent (Nous Research), modelo DeepSeek Reasoner | Ferramenta principal. Leu a transcrição e o código, extraiu decisões e descartes, redigiu os documentos e executou as revisões críticas entre ciclos. |
| Terminal integrado (bash) do agente | Inspeção do repositório: leitura de todos os módulos, `package.json`, schema Prisma, migrations e factories de teste, para citar apenas caminhos e símbolos que existem de fato. |
| Script Python de auditoria (escrito pelo agente) | Validação mecânica da entrega: parse da tabela do tracker, verificação de que cada timestamp `[hh:mm] Nome` existe na transcrição, cálculo do percentual de rastreabilidade e verificação de existência de cada caminho de arquivo citado. |

Não usei Claude Code, Cursor ou Copilot nesta execução. Foi uma sessão de agente único com ferramentas de terminal, e o script de auditoria foi o contrapeso mecânico à geração de texto: em vez de confiar na leitura do próprio modelo, a entrega foi checada por código.

## Workflow adotado

A ordem de produção seguiu a lógica de que decisão vem antes de proposta, e proposta vem antes de especificação:

1. **Reconhecimento do repositório antes de qualquer escrita.** Li a transcrição inteira, depois todos os arquivos de `src/`, o `schema.prisma`, a migration inicial, as factories de teste e o `package.json`. Isso criou um inventário de símbolos reais (`changeStatus`, `AppError`, `InvalidStatusTransitionError`, `redactPaths`, `paginated`, `requireRole`, `createPrismaClient`) que só depois virou texto.
2. **Extração de decisões em duas colunas.** Antes de escrever, produzi internamente três listas separadas: o que foi decidido, o que foi descartado ou adiado, e o que foi mencionado sem virar requisito. As duas últimas listas são as que impedem alucinação.
3. **ADRs primeiro.** Oito decisões, das seis principais exigidas mais duas secundárias (snapshot do payload e filtro na inserção). Cada ADR com contexto, alternativas reais da reunião e consequências negativas explícitas.
4. **RFC em cima das decisões.** Conciso e em nível de arquitetura, com alternativas e questões em aberto, apontando para os ADRs em vez de repetir o detalhe.
5. **FDD em cima do RFC.** O documento mais profundo, com fluxos em Mermaid, sete contratos HTTP com exemplos, matriz de erros, resiliência, observabilidade e a seção de integração com caminhos reais do código.
6. **PRD por último entre os grandes documentos.** Com RFC, FDD e ADRs prontos, o PRD virou consolidação em altura de produto, sem repetir detalhe técnico.
7. **Tracker varrendo os documentos prontos.** Montado depois, lendo cada documento e extraindo os IDs.
8. **README e auditoria.** O README por último, e então um script de verificação item por item da checklist de aceite.

O padrão de interação foi sempre o mesmo: instrução com escopo fechado, geração, revisão contra a transcrição e o código, correção pontual. Nenhum documento foi aceito na primeira saída.

## Prompts customizados

**Prompt 1: extração de decisões com filtro de descartes.**

```
Você vai ler TRANSCRICAO.md e o código em src/ e prisma/.
Não escreva documento nenhum ainda. Produza três listas:

1. DECIDIDO: toda decisão fechada, com timestamp, falante e a frase que fecha a decisão.
2. DESCARTADO OU ADIADO: toda alternativa que foi recusada ou empurrada para depois,
   com o motivo textual do descarte. Inclua quem descartou.
3. MENCIONADO SEM VIRAR REQUISITO: assuntos que apareceram mas não geraram decisão.

Regras:
- Não deduza. Se a decisão não tem frase de fechamento, vai para a lista 3.
- Não promova sugestão de um participante a decisão se outro participante não confirmou.
- Para cada item da lista 2, diga explicitamente qual trade-off motivou o descarte.
- Saída em tabela, sem prosa.
```

Esse prompt foi o que separou o desafio de um resumo: as listas 2 e 3 viraram a seção "Fora de escopo" do PRD, as alternativas do RFC e as questões em aberto.

**Prompt 2: FDD restrito a caminhos reais de código.**

```
Escreva a seção "Integração com o sistema existente" do FDD.

Regras absolutas:
- Só cite arquivos que existem no repositório. Antes de citar, confirme com ls/cat.
- Para cada arquivo, diga: o que muda, o que NÃO muda, e qual símbolo concreto é afetado
  (método, classe, constante, middleware).
- Se a feature exigir um arquivo novo, marque explicitamente como "a criar" e não como origem.
- Mostre o trecho de código mínimo da alteração, não pseudocódigo genérico.
- Se você não conseguir apontar o símbolo exato, não escreva a linha.
```

**Prompt 3: geração do tracker a partir dos documentos prontos.**

```
Leia docs/PRD.md, docs/RFC.md, docs/FDD.md e docs/adrs/*.md e gere docs/TRACKER.md.

Formato: | ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |

Regras:
- Fonte só pode ser TRANSCRICAO ou CODIGO.
- TRANSCRICAO exige "[hh:mm] Nome" exatamente como aparece na transcrição.
- CODIGO exige caminho relativo que exista no repositório.
- Um item sem origem identificável não entra na tabela: ele vai para uma seção separada
  chamada "Itens derivados", explicando por que não tem origem.
- Cobertura: toda linha numerada dos documentos precisa de linha correspondente no tracker.
```

**Prompt 4: revisão adversarial do documento gerado.**

```
Releia o FDD como revisor hostil e responda:
1. Que afirmações aqui não estão na transcrição nem no código?
2. Que trecho contradiz o comportamento real do código? Confira assinatura de método,
   formato de resposta e nomes de campo no arquivo fonte.
3. Que detalhe de implementação eu apresentei como decisão da reunião sem ser?
4. O que está no nível errado, ou seja, detalhe de FDD vazando para o RFC ou repetição
   entre documentos?
Liste por seção, com a correção sugerida. Não reescreva o documento inteiro.
```

## Iterações e ajustes

Foram cinco ciclos principais, e nenhum deles foi cosmético.

**Ciclo 1, exploração e inventário.** A primeira passada tentou tratar a transcrição como fonte suficiente. O resultado seria raso, porque o FDD precisa de símbolos reais. A correção foi inverter a ordem: ler o código primeiro e só depois escrever, com uma lista de símbolos do projeto em mãos.

**Ciclo 2, ADRs e o risco de generalidade.** A primeira versão dos ADRs descrevia as decisões de forma correta e inútil, sem as consequências negativas. Corrigi exigindo, por ADR, pelo menos uma consequência negativa com trade-off explícito: o custo de arquivar a outbox saiu do escopo, a janela de 15 horas sem entrega, as duas secrets válidas por 24 horas, a perda de eventos históricos por filtrar na inserção.

**Ciclo 3, FDD e três erros concretos de IA.** A revisão adversarial encontrou problemas que passariam batido em leitura rápida:

- *Envelope de resposta inventado.* O FDD usava `meta` na paginação, mas `paginated()` em `src/shared/http/response.ts` devolve `{ data, pagination }`. Corrigido nos três endpoints paginados. É exatamente o tipo de erro que só aparece quando se lê o arquivo fonte.
- *Mecanismo de assinatura impossível.* Eu havia escrito que "o worker assina com a secret nova e aceita validação com qualquer uma das duas". O worker envia, não valida. A versão final envia `X-Signature` com a secret nova e `X-Signature-Previous` durante o grace period.
- *Contradição de status code.* A reunião tratou `https` como "só uma validação Zod" (que produz 400), mas citou `WEBHOOK_INVALID_URL` (que naturalmente seria 422). Em vez de escolher um e esconder o outro, documentei a fronteira: 400 para URL malformada, 422 para URL bem formada que viola política.

**Ciclo 4, consistência interna e descarte silencioso.** Ao revisar o PRD contra os requisitos não funcionais, encontrei uma contradição criada por mim: o DELETE dizia que eventos pendentes do webhook removido "são descartados na leitura do worker", o que viola a invariante de nunca perder evento. A correção foi remoção lógica com `deletedAt` e encerramento do evento com motivo `WEBHOOK_INACTIVE`. Também corrigi a contagem de endpoints no critério de aceite, de oito para sete, e um caso de rotação repetida que eu havia inventado como decidido, reclassificado como ponto em aberto.

**Ciclo 5, tracker e auditoria mecânica.** O tracker foi gerado, e então um script verificou três coisas: se cada `[hh:mm] Nome` existe na transcrição, se cada caminho `CODIGO` existe no repositório e qual é o percentual real de rastreabilidade. O script apontou 158 linhas, 130 com origem na transcrição (82,3%), 28 com origem no código e três caminhos citados que ainda não existem, todos arquivos novos da feature. Os números da tabela de cobertura estavam errados na primeira versão e foram substituídos pelos medidos. Também ajustei a redação para deixar explícito que `src/modules/webhooks` e `src/worker.ts` são destino de implementação, não origem de rastreabilidade.

## Como navegar a entrega

```
.
├── README.md                          este documento
├── TRANSCRICAO.md                     fonte primária, não alterada
├── docs/
│   ├── PRD.md                         por que e o quê
│   ├── RFC.md                         como pretendemos resolver, e o que está aberto
│   ├── FDD.md                         como construir em detalhe
│   ├── TRACKER.md                     de onde veio cada item
│   └── adrs/
│       ├── README.md                  índice dos ADRs
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-retry-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes.md
│       ├── ADR-007-snapshot-do-payload-na-insercao.md
│       └── ADR-008-filtro-de-eventos-na-insercao.md
├── src/                               código da aplicação, não alterado
├── prisma/                            schema e migrations, não alterados
└── tests/                             testes, não alterados
```

**Ordem de leitura sugerida:**

1. `docs/PRD.md`, seções 1 a 5, para entender o problema e o que está fora de escopo.
2. `docs/RFC.md`, para a proposta em altura de arquitetura e as questões em aberto.
3. `docs/adrs/README.md` e os oito ADRs, para o porquê de cada decisão isolada.
4. `docs/FDD.md`, seções 4 a 7, para fluxos e contratos, e a seção 12 para a integração com o código existente.
5. `docs/TRACKER.md`, como referência cruzada durante a revisão, item por item.

**Para quem vai implementar:** comece pelo FDD seção 12 (integração) e seção 4 (fluxos), depois volte ao ADR-001 e ao ADR-008, que são as decisões que tocam a transação de mudança de status. A seção 13 do FDD lista, de forma explícita, o que é proposta de implementação e não decisão da reunião, e precisa ser validada antes de virar código.

**Para quem vai revisar:** o `docs/TRACKER.md` é o caminho mais curto. Cada linha tem documento, tipo, resumo, fonte e localização. Linha sem localização válida indica item sem origem, e a seção "Itens derivados" lista os casos assumidos de propósito.

## Referência ao enunciado original

O enunciado do desafio está preservado no histórico do Git deste repositório, no commit anterior à substituição deste README. O repositório base original é [helder-puia/mba-ia-desafio-design-docs-com-ia](https://github.com/helder-puia/mba-ia-desafio-design-docs-com-ia).
