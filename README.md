# Da Reunião ao Documento — Design Docs Gerados por IA

Pacote de documentação técnica produzido a partir da transcrição de uma reunião de 55 minutos e do código de um Order Management System em produção, usando IA como ferramenta principal de produção.

> O enunciado original do desafio está preservado em [`docs/ENUNCIADO.md`](./docs/ENUNCIADO.md).

---

## Sobre o desafio

O ponto de partida é um vácuo documental deliberado: uma empresa decidiu construir um Sistema de Webhooks de Notificação de Pedidos numa call entre tech lead, PM, dois engenheiros e uma engenheira de segurança, e não registrou nada além da gravação. A aplicação existe e funciona — Node.js, TypeScript, Express, Prisma sobre MySQL, com máquina de estados de pedido, controle transacional de estoque e auditoria de mudanças de status — mas não tem nenhum mecanismo de notificação externa. A tarefa é transformar 323 linhas de conversa falada em um pacote de design docs acionável o suficiente para o time começar a codar na segunda-feira.

O que torna o exercício interessante não é gerar texto — é **filtrar**. A transcrição mistura, sem separação visível, decisões fechadas, requisitos funcionais, restrições, detalhes técnicos secundários, ideias explicitamente descartadas e pontos adiados para fases futuras. Identificar o que **não** entra vale tanto quanto identificar o que entra, e a IA, deixada solta, trata tudo o que foi dito como requisito. Meu papel foi o de maestro: definir a fronteira de cada documento, formular prompts dirigidos, e revisar criticamente cada afirmação contra a ata e contra o código. A regra que organizou o trabalho inteiro: **se uma linha não pode apontar para um timestamp da transcrição ou para um caminho de arquivo real, ela sai.**

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **Claude Code (Opus 5, contexto de 1M)** | Ferramenta principal. Leu a transcrição inteira e ~25 arquivos do código-fonte no mesmo contexto, o que permitiu cruzar fala e implementação sem colagem manual de trechos. Produziu todos os documentos e executou a varredura de verificação final. |
| **Skill customizada `resolver-exercicio`** | Skill própria do meu repositório de MBA. Estrutura o trabalho em fases (localizar → extrair requisitos → planejar → executar → verificar) e separa o que a IA faz sozinha do que exige ação manual minha em ferramentas externas. |
| **`gh` CLI + Bash/Grep** | Fork e clone do repositório base; verificação automatizada de todos os caminhos de arquivo citados nos documentos e contagem das métricas do tracker. Aqui a IA serve de executor, não de autor. |

O contexto de 1M foi determinante. Com a transcrição e o código carregados juntos, foi possível escrever coisas como "o `redact` do Pino não cobre `*.secret`" — uma observação que cruza uma fala de segurança ([09:22] Diego) com as linhas 4 a 11 de `src/shared/logger/index.ts`. Em contexto fragmentado, essa conexão não aparece.

---

## Workflow adotado

Segui a ordem sugerida pelo enunciado — decisões primeiro, documentos de alto nível depois — porque ela resolve um problema real: o PRD escrito antes dos ADRs sai genérico, já que não há trade-offs fechados para referenciar.

**1. Contextualização (antes de escrever qualquer coisa).**
Leitura completa de `TRANSCRICAO.md` e mapeamento do código: `prisma/schema.prisma`, os 5 módulos de `src/modules/`, os 4 middlewares, `src/shared/errors/`, `src/shared/logger/`, `src/app.ts`, `src/routes/index.ts`, `src/config/` e `package.json`. Sem isso, a seção de integração do FDD seria ficção plausível.

**2. Inventário rastreável.**
Antes de redigir, montei a lista de decisões, requisitos, restrições, descartes e adiamentos, cada um com seu timestamp. Esse inventário virou o esqueleto do tracker e a fonte de verdade contra a qual cada documento foi conferido depois.

**3. ADRs (8 arquivos).**
As 6 decisões principais do enunciado, mais duas secundárias que mereciam ADR próprio: snapshot do payload na inserção e filtro de status na inserção. Cada um com alternativa real discutida na reunião e trade-off explícito.

**4. RFC.**
Consolidação em nível de arquitetura, com as 7 alternativas descartadas e as 5 questões em aberto — que são o conteúdo mais fácil de perder, porque estão espalhadas pela ata em falas isoladas.

**5. FDD.**
O documento mais longo. Modelo de dados, 4 fluxos, 7 contratos HTTP, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e 13 pontos de integração com arquivos reais.

**6. PRD.**
Por último entre os grandes, como o enunciado sugere. Com ADRs, RFC e FDD prontos, ele vira consolidação em nível de produto.

**7. Tracker e README.**
O tracker varrendo os documentos prontos; o README depois, quando o processo já podia ser descrito com honestidade.

**8. Verificação final.**
Varredura automatizada: todo caminho de arquivo citado existe? Toda linha `TRANSCRICAO` tem timestamp no formato certo? As contagens declaradas batem com a tabela?

**Fronteira entre documentos**, que foi a regra que evitou duplicação:

| Documento | Pergunta que responde | Altura |
| --- | --- | --- |
| PRD | Por que e o quê? | Produto |
| RFC | Como pretendemos resolver, e o que está em aberto? | Arquitetura |
| ADRs | Por que decidimos exatamente assim? | Decisão pontual |
| FDD | Como construir, em detalhe? | Implementação |
| Tracker | De onde veio cada coisa? | Transversal |

---

## Prompts customizados

### Prompt 1 — Filtragem da transcrição, com separação explícita do que não entra

O prompt genérico ("extraia os requisitos dessa transcrição") produz uma lista achatada em que tudo o que foi mencionado vira requisito. A correção foi forçar a IA a **classificar** antes de listar, e a exigir a evidência textual junto:

```
Leia TRANSCRICAO.md inteira antes de responder.

Classifique CADA item discutido em exatamente uma destas categorias:
  (A) DECISÃO FECHADA   — alguém disse explicitamente "decidido/fechado/anotado"
  (B) REQUISITO         — pedido de comportamento que ninguém contestou
  (C) RESTRIÇÃO/NFR     — limite numérico, obrigação técnica ou de segurança
  (D) DESCARTADO        — proposto e rejeitado NESTA reunião
  (E) ADIADO            — empurrado para fase futura ou "decidir depois"
  (F) RUÍDO             — detalhe conversacional sem consequência de projeto

Para cada item devolva: categoria | timestamp [hh:mm] | falante |
citação literal da fala que sustenta a classificação | uma linha de resumo.

Regras:
- Item sem citação literal NÃO entra na lista. Não infira.
- Para (D) e (E), diga QUEM rejeitou/adiou e QUAL foi o motivo declarado.
- Se duas falas se contradizem, liste as duas e marque CONFLITO —
  não escolha uma por conta própria.
- Não resuma a reunião. Não proponha arquitetura. Só classifique.
```

O item `(F) RUÍDO` foi o que mais rendeu: obrigou a IA a decidir ativamente o que descartar, em vez de inflar os documentos com tudo.

### Prompt 2 — Seção de integração ancorada no código, com verificação obrigatória

Esta seção é onde a alucinação é mais provável e mais cara: nomes de arquivo plausíveis mas inexistentes passam despercebidos numa leitura casual. O prompt exige leitura antes de escrita e proíbe genéricos:

```
Escreva a seção "Integração com o sistema existente" do FDD.

Antes de escrever UMA linha: leia de fato os arquivos. Não escreva sobre
arquivo que você não abriu nesta sessão.

Para CADA ponto de integração, entregue nesta ordem:
  1. Caminho exato do arquivo, como ele existe no repositório.
  2. O que o arquivo faz HOJE — com nome de função/classe e número de
     linha reais, não paráfrase.
  3. O que muda, ou a frase "Alteração: nenhuma" quando for reuso puro.
  4. A fala da transcrição que justifica a mudança, com [hh:mm] Nome.

Proibido:
- Citar arquivo, função ou classe que não exista no repositório.
- Escrever "seguindo boas práticas", "de forma adequada" ou equivalente.
- Descrever mudança sem apontar onde exatamente ela entra.

Obrigatório: se ao ler o código você encontrar algo que CONTRADIZ o que
foi decidido na reunião, ou uma lacuna que a decisão não cobre, registre
isso em destaque em vez de suavizar.
```

A última instrução é a que produziu o achado mais valioso do pacote inteiro — ver abaixo.

---

## Iterações e ajustes

Foram **quatro ciclos** de geração, revisão crítica e correção. Os pontos concretos:

### Iteração 1 — Requisitos que a reunião havia descartado apareceram como escopo

A primeira extração trouxe "notificação por email quando o webhook falha" e "dashboard de webhooks para o cliente" como requisitos funcionais do PRD. Ambos foram **explicitamente negados** na ata: Larissa disse "não. Email tá fora de escopo dessa fase" ([09:37]) e "não, agora não. Só endpoints. Painel é projeto separado do time de frontend" ([09:40]).

A IA tinha captado o pedido de Marcos e ignorado a resposta que veio em seguida. A correção foi o Prompt 1 acima, com as categorias `(D) DESCARTADO` e `(E) ADIADO` obrigando a IA a registrar **quem** rejeitou e **por quê**. Os dois itens migraram para a seção "Fora de escopo" do PRD, onde documentam uma decisão em vez de contradizê-la.

### Iteração 2 — Métricas quantitativas inventadas

O primeiro PRD trazia "99,9% de taxa de entrega bem-sucedida" e "p99 de latência abaixo de 500ms" como objetivos. Números profissionais, bem formatados, e **completamente inventados** — nenhum SLA percentual foi discutido na reunião.

Ao tentar preencher a coluna *Localização* do tracker para essas linhas, não havia timestamp. Foi exatamente o mecanismo que o enunciado descreve: o tracker como detector de alucinação. Substituí por metas ancoradas em falas reais — 10 segundos ([09:02] Marcos), 3 clientes ([09:00] Marcos), fim de novembro ([09:45] Marcos), a janela de 2 horas do incidente concreto ([09:16] Diego). Onde a ata não fixou número, a meta ficou escrita como direção, com nota explícita de que o limiar precisa ser definido com baseline medido.

A seção [§4 do tracker](./docs/TRACKER.md#4-o-que-o-tracker-impediu-de-entrar) registra sete itens descartados assim.

### Iteração 3 — RFC e FDD dizendo a mesma coisa

A primeira versão do RFC tinha payloads JSON completos, lista de colunas de tabela e matriz de erros — conteúdo de FDD. O RFC deve ser conciso, de 2 a 4 páginas, e falar em decisão, não em implementação.

Reescrevi o RFC para operar só em nível de arquitetura, com uma linha explícita apontando o leitor para o FDD quando o assunto desce a contrato. Em troca, aprofundei no RFC o que é genuinamente dele e estava raso: as **alternativas descartadas** (7, cada uma com o trade-off que motivou o descarte) e as **questões em aberto** (5, cada uma com o encaminhamento literal que recebeu na reunião). Essas estavam espalhadas em falas isoladas e eram as mais fáceis de perder.

### Iteração 4 — Do código veio uma contradição com a reunião

A última rodada foi a mais produtiva, e não veio de corrigir a IA — veio de mandar ela ler o código a sério, com a instrução final do Prompt 2.

Bruno afirmou na reunião: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" ([09:29]), e Larissa fechou a diretriz de reuso máximo sem alteração ([09:30]). Ao abrir `src/shared/logger/index.ts`, o `redactPaths` cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` — **e não cobre `*.secret`**.

Como a feature passa a armazenar a secret HMAC de cada cliente, logar o objeto de endpoint vazaria essa secret nos nossos próprios logs. É precisamente o incidente que Diego relatou ter acontecido do lado de um cliente ([09:22]). A diretriz de "reuso sem alteração" não se sustenta nesse ponto, e ninguém percebeu isso durante a call.

O achado entrou como risco no [RFC](./docs/RFC.md#62-riscos) (R2) e no [PRD](./docs/PRD.md#10-riscos-e-mitigação) (R2), como consequência negativa em [ADR-004](./docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) e [ADR-006](./docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md), e como a **única alteração obrigatória** na seção de integração do [FDD](./docs/FDD.md#115-srcsharedloggerindexts--reuso-com-uma-alteração-obrigatória), com critério de aceite próprio (CA-17).

O mesmo ciclo produziu outros dois pontos que a reunião não cobriu e o código revelou: o `X-Event-Id` precisa ser preservado no replay de DLQ, sob pena de quebrar a deduplicação prometida em [ADR-005](./docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md); e eventos presos em `PROCESSING` após crash do worker precisam de um lease, porque com single-worker ninguém os recupera.

### O que ficou de aprendizado

A IA é excelente produtora e péssima auditora de si mesma. Ela não distingue, sozinha, "isso foi decidido" de "isso foi mencionado" — e o texto sai igualmente convincente nos dois casos. O tracker não é burocracia do desafio: é o instrumento que força cada afirmação a apontar para uma origem, e é onde a invenção aparece. Três dos quatro ciclos acima começaram com a pergunta "de onde veio essa linha?" ficando sem resposta.

---

## Como navegar a entrega

### Estrutura

```
.
├── README.md                ← este arquivo (processo de produção)
├── TRANSCRICAO.md           ← fonte primária, não alterada
├── docs/
│   ├── ENUNCIADO.md         ← enunciado original do desafio
│   ├── PRD.md               ← problema, escopo, requisitos, métricas
│   ├── RFC.md               ← proposta técnica para revisão
│   ├── FDD.md               ← especificação de implementação
│   ├── TRACKER.md           ← rastreabilidade (186 linhas)
│   └── adrs/
│       ├── ADR-001-outbox-transacional-no-mysql.md
│       ├── ADR-002-worker-em-processo-separado-com-polling.md
│       ├── ADR-003-retry-com-backoff-exponencial-e-dlq.md
│       ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
│       ├── ADR-005-entrega-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
│       ├── ADR-007-snapshot-do-payload-na-insercao.md
│       └── ADR-008-filtro-de-eventos-aplicado-na-insercao.md
├── src/  prisma/  tests/    ← código do OMS, não alterado
└── ...
```

### Ordem sugerida de leitura

1. **[`docs/PRD.md`](./docs/PRD.md)** — o problema, quem pediu, o que entra e, principalmente, o que **não** entra.
2. **[`docs/RFC.md`](./docs/RFC.md)** — a proposta em nível de arquitetura, as alternativas descartadas e as 5 questões que continuam abertas.
3. **[`docs/adrs/`](./docs/adrs/)** — na ordem numérica. ADR-001 e ADR-002 sustentam o desenho inteiro; **[ADR-006](./docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)** é o que mais conversa com o código existente.
4. **[`docs/FDD.md`](./docs/FDD.md)** — o "como construir". Se for ler uma seção só, leia a **[§11, Integração com o sistema existente](./docs/FDD.md#11-integração-com-o-sistema-existente)**.
5. **[`docs/TRACKER.md`](./docs/TRACKER.md)** — a rastreabilidade. A **[§4](./docs/TRACKER.md#4-o-que-o-tracker-impediu-de-entrar)** mostra o que foi barrado por não ter origem.

### Atalhos por interesse

| Se você quer... | Vá direto para |
| --- | --- |
| Entender o desenho em 5 minutos | [RFC §1 (TL;DR)](./docs/RFC.md#1-tldr) e o diagrama da [§3.1](./docs/RFC.md#31-visão-geral) |
| Saber o que **não** foi incluído e por quê | [PRD §5.2](./docs/PRD.md#52-fora-de-escopo) |
| Começar a implementar | [FDD §5 (fluxos)](./docs/FDD.md#5-fluxos-detalhados) e [§11 (integração)](./docs/FDD.md#11-integração-com-o-sistema-existente) |
| Revisar os contratos de API | [FDD §6](./docs/FDD.md#6-contratos-públicos) |
| Conferir a origem de qualquer afirmação | [TRACKER](./docs/TRACKER.md) |
| Ver o que ainda precisa ser decidido | [RFC §5](./docs/RFC.md#5-questões-em-aberto) |

### Nota sobre o escopo da entrega

A entrega é **puramente documental**. Nenhum arquivo de `src/`, `prisma/`, `tests/` ou de configuração foi alterado; `TRANSCRICAO.md` está intacto. O código serviu de contexto e de referência verificável — todos os caminhos de arquivo citados nos documentos foram conferidos contra o repositório antes do commit.
