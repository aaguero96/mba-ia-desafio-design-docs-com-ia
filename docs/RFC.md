# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **RFC** | 001 |
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-09-21 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno — time de Pedidos), Diego (Engenheiro Sênior — time de Plataforma), Sofia (Engenheira de Segurança) |
| **Fonte** | Reunião técnica de ~55min registrada em [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| **Documentos relacionados** | [PRD](./PRD.md) · [FDD](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

> **Nota de processo.** A decisão técnica foi tomada na reunião; este RFC a formaliza e a abre para revisão. Larissa registrou a intenção de abrir o doc de design e marcar uma sessão com Bruno e Diego antes de começar a codar ([09:50] Larissa). Sofia condicionou o deploy a pelo menos dois dias úteis de revisão de segurança sobre HMAC e geração de secret ([09:46] Sofia).

---

## 1. TL;DR

Clientes B2B hoje descobrem mudança de status de pedido fazendo polling em `GET /orders`, o que torna a integração deles lenta e cara ([09:00] Marcos). Propomos um sistema de **webhooks outbound** que notifica o endpoint do cliente quando o status de um pedido muda.

A arquitetura é deliberadamente conservadora: **padrão Outbox sobre o MySQL que já temos**, com a linha do evento inserida na mesma transação que muda o status do pedido, e um **worker em processo separado** que consome a outbox por polling de 2 segundos e faz a entrega HTTP. Falhas entram em **retry com backoff exponencial (5 tentativas, até ~15h)** e, esgotadas as tentativas, param numa **DLQ em tabela separada** com replay manual via endpoint admin.

Cada entrega é assinada com **HMAC-SHA256**, usando uma **secret única por endpoint**, rotacionável com grace period de 24h. A garantia é **at-least-once**: o cliente dedupica pelo header `X-Event-Id`.

Nenhuma infraestrutura nova. Nenhuma dependência nova de runtime. O módulo segue exatamente os padrões já estabelecidos na codebase.

**Prazo estimado:** três sprints, incluindo a revisão de segurança ([09:46] Larissa), com alvo no fim de novembro ([09:45] Marcos).

---

## 2. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem notificados em tempo real quando o status dos pedidos deles muda ([09:00] Marcos).

Hoje eles fazem polling em `GET /orders` de tempos em tempos para descobrir se algo mudou. Isso deixa a integração deles lenta e cara ([09:00] Marcos). Há pressão comercial concreta: a Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre ([09:00] Marcos).

"Tempo real", para esses clientes, foi definido com precisão: **qualquer coisa abaixo de 10 segundos** ([09:02] Marcos). O que não pode acontecer é o evento ficar pendurado e eles terem que atualizar manualmente.

O escopo direcional foi fechado logo no início: **apenas outbound**, do OMS para o cliente. Os clientes querem receber, não enviar ([09:02] Sofia; [09:02] Marcos).

Do lado técnico, a aplicação não possui hoje **nenhum** mecanismo de notificação externa, evento, fila ou webhook. O ciclo de vida do pedido é controlado por máquina de estados (`src/modules/orders/order.status.ts`) e a mudança de status é transacional (`src/modules/orders/order.service.ts`, método `changeStatus`), mas termina no banco — nada sai da fronteira do sistema.

---

## 3. Proposta técnica

### 3.1 Visão geral

```
                    ┌─────────────────────────────────────────┐
  PATCH /orders/:id/status                                    │
            │       │   Processo da API (src/server.ts)       │
            ▼       │                                         │
   ┌────────────────────────┐                                 │
   │ OrderService.changeStatus                                │
   │  ── transação única ──                                   │
   │   1. update orders                                       │
   │   2. insert order_status_history                         │
   │   3. debita/repõe estoque                                │
   │   4. insert webhook_outbox  ◄── novo                     │
   └────────────┬───────────┘                                 │
                │         └─────────────────────────────────────┘
                ▼
        ┌───────────────┐
        │ webhook_outbox│  (MySQL, mesmo banco)
        └───────┬───────┘
                │  polling 2s, batch pequeno
                ▼
   ┌─────────────────────────────────────────┐
   │  Processo do worker (src/worker.ts)     │
   │   • monta headers + assina HMAC-SHA256  │
   │   • POST no endpoint do cliente (10s)   │
   │   • 2xx  → DELIVERED                    │
   │   • erro → agenda retry (backoff)       │
   │   • 5ª falha → webhook_dead_letter      │
   └──────────────────┬──────────────────────┘
                      │ HTTPS + X-Signature
                      ▼
              Endpoint do cliente
```

### 3.2 Pilares da proposta

**Captura transacional via Outbox.** Na mesma transação SQL que atualiza `orders` e `order_status_history`, inserimos a linha do evento em `webhook_outbox` ([09:06] Diego). Se a transação comita, o evento existe; se dá rollback, o evento some junto. Bruno colocou a exigência em termos absolutos: não pode existir o caso de o status mudar e o evento não sair ([09:40] Bruno). → [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md)

**Worker separado em polling.** Processo próprio (`src/worker.ts`, script `npm run worker`), que a cada 2 segundos lê os pendentes mais antigos em batch pequeno e dispara ([09:09] Diego; [09:11] Larissa). Fora do processo da API, para que restart da API não derrube a entrega ([09:11] Diego). Mesmo banco, mas `PrismaClient` próprio, porque client é por processo ([09:30] Bruno). Um único worker nesta fase, o que dá ordenação por `order_id` de graça. → [ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md)

**Resiliência por backoff + DLQ.** 5 tentativas com progressão 1m / 5m / 30m / 2h / 12h, timeout de 10s por tentativa, e tabela `webhook_dead_letter` separada ao fim ([09:17] Larissa; [09:18] Diego; [09:42] Diego). Replay manual por endpoint admin, restrito a role `ADMIN` e com log de auditoria ([09:36] Sofia). → [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)

**Autenticidade e integridade por HMAC.** Assinatura HMAC-SHA256 sobre o corpo, transportada em `X-Signature`, com secret única por endpoint e rotação com grace period de 24h ([09:22] Sofia). URL obrigatoriamente `https`, validada no schema Zod ([09:23] Sofia). → [ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)

**Entrega at-least-once.** Duplicatas são possíveis por construção; o cliente dedupica pelo `X-Event-Id`, um UUID estável gerado na inserção na outbox ([09:25] Diego; [09:26] Larissa). → [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)

**Aderência total aos padrões existentes.** Módulo `src/modules/webhooks/` com controller/service/repository/routes/schemas, erros estendendo `AppError` com prefixo `WEBHOOK_`, Pino, `errorMiddleware` sem alteração, `requireRole` reaproveitado ([09:27]–[09:30] Bruno, Larissa). → [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

**Snapshot do payload e filtro na inserção.** O payload é renderizado e persistido no momento da inserção, para que o evento reflita o estado de quando o status mudou, mesmo entregue 15h depois ([09:52] Larissa). E o filtro de status assinados é aplicado na inserção: se nenhum endpoint do customer quer aquele status, a linha nem é criada ([09:34] Bruno). → [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) · [ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md)

### 3.3 Superfície de API proposta

Sete endpoints, todos sob o prefixo `/api/v1` já usado pela aplicação (`src/app.ts`): CRUD de configuração de webhook (criar, listar por customer, editar, remover), rotação de secret, consulta do histórico de entregas, e replay de DLQ restrito a `ADMIN`. Os contratos completos — payloads, headers, status codes e matriz de erros `WEBHOOK_*` — estão no [FDD](./FDD.md#5-contratos-públicos), não aqui.

### 3.4 O que a proposta deliberadamente não inclui

- **Webhooks inbound.** Escopo fechado como outbound-only no início da reunião ([09:02] Marcos).
- **Email de alerta ao cliente quando o webhook falha.** Marcos perguntou; Larissa respondeu que email está fora de escopo desta fase, talvez na próxima, depois de medir o impacto ([09:37] Larissa).
- **Dashboard visual para o cliente.** "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" ([09:40] Larissa).
- **Arquivamento das linhas entregues.** Diego mencionou arquivar após ~30 dias e declarou explicitamente fora do escopo desta feature ([09:08] Diego).
- **Ordering global entre pedidos diferentes.** Registrado como limitação conhecida ([09:13] Larissa); os clientes nunca pediram isso ([09:14] Marcos).

---

## 4. Alternativas consideradas

### 4.1 Disparo HTTP síncrono dentro de `changeStatus`

**O que era.** Chamar o endpoint do cliente diretamente na transação de mudança de status. Foi a primeira opção colocada na mesa ([09:03] Larissa).

**Por que foi descartada.** A transação já é pesada: atualiza `orders`, insere em `order_status_history` e decrementa `stockQuantity` dos produtos. Acrescentar uma chamada HTTP faria qualquer cliente lento travar mudança de status para outros pedidos ([09:04] Bruno). E o caso do cliente offline não tem saída aceitável: dar rollback numa mudança de status válida porque um terceiro caiu é inadmissível ([09:04] Bruno). Diego, ao entrar na call, foi categórico: "síncrono está fora de questão" ([09:06] Diego).

**Trade-off que motivou o descarte.** Latência mínima de entrega em troca de acoplar a disponibilidade de uma operação de negócio central à disponibilidade de sistemas de terceiros. O time recusou o acoplamento.

### 4.2 Redis Streams ou fila dedicada

**O que era.** Publicar o evento em um broker externo e consumir de lá ([09:07] Larissa).

**Por que foi descartada.** Exigiria subir e operar infraestrutura nova. Diego: o time é pequeno, subir Redis Cluster para isso é overengineering, e outbox no MySQL existente resolve ([09:07] Diego).

**Trade-off que motivou o descarte.** Throughput teórico e fan-out nativo de um broker em troca de zero infraestrutura adicional. Vale notar que o broker também **não** daria a garantia transacional de graça: publicar no Redis dentro da transação MySQL reintroduziria o problema de dual-write que o outbox justamente elimina.

### 4.3 Trigger de banco em vez de polling

**O que era.** Usar trigger do MySQL para ser mais reativo, em vez de o worker ficar consultando ([09:09] Bruno).

**Por que foi descartada.** MySQL não tem listener nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres. A trigger executa SQL mas não notifica processo externo; avisar o worker exigiria improvisar algo como escrever em arquivo ou bater num endpoint ([09:09] Diego). E polling de 2 segundos já atende o requisito de 10s com folga.

**Trade-off que motivou o descarte.** Latência sub-segundo e ausência de leituras ociosas, em troca de um mecanismo simples que não depende de recursos que o MySQL não oferece.

### 4.4 DLQ como flag `failed` na própria outbox

**O que era.** Marcar o evento como falho na tabela principal em vez de movê-lo ([09:17] Larissa).

**Por que foi descartada.** Diego preferiu tabela separada: mantém a leitura da outbox principal limpa e deixa a DLQ como evidência para debug e reprocessamento ([09:18] Diego).

**Trade-off que motivou o descarte.** Uma tabela e uma escrita a mais no momento da falha permanente, em troca de a query quente do worker não precisar filtrar lixo acumulado.

### 4.5 Três tentativas de retry em vez de cinco

**O que era.** Política mais agressiva, proposta por Bruno ([09:16] Bruno).

**Por que foi descartada.** Com 3 tentativas a janela total fica em torno de 30 minutos. Diego trouxe um precedente real: já houve cliente com indisponibilidade de duas horas em manutenção planejada, e 3 tentativas matariam os eventos antes de ele voltar ([09:16] Diego).

**Trade-off que motivou o descarte.** Eventos ficam retentáveis por mais tempo, ocupando a outbox durante incidentes, em troca de tolerar indisponibilidade real de cliente.

### 4.6 Secret global da plataforma

**O que era.** Uma única secret compartilhada com todos os clientes ([09:21] Sofia, ao rejeitar).

**Por que foi descartada.** "Se vaza uma, vaza tudo" ([09:21] Sofia). Há precedente: um cliente já vazou secret em log de aplicação dele ([09:22] Diego).

**Trade-off que motivou o descarte.** N secrets para gerar, guardar e rotacionar em vez de uma, em troca de limitar o raio de explosão de um vazamento a um único endpoint.

### 4.7 Exactly-once

**O que era.** Garantir que o cliente receba cada evento exatamente uma vez ([09:24]–[09:25] Diego).

**Por que foi descartada.** Exigiria coordenação dos dois lados e ficaria muito mais complexo; at-least-once com `event_id` resolve 99% dos casos, e é o que Stripe e GitHub fazem ([09:25] Diego).

**Trade-off que motivou o descarte.** O cliente precisa implementar deduplicação — Sofia registrou a ressalva de que isso joga responsabilidade para ele ([09:25] Sofia) — em troca de um protocolo simples e sem estado de coordenação.

---

## 5. Questões em aberto

| # | Questão | O que foi dito | Encaminhamento |
| --- | --- | --- | --- |
| **Q1** | **Rate limiting de saída.** Se um cliente tem 50 pedidos mudando de status em um minuto, bombardeamos ele com 50 chamadas? | Diego levantou o ponto ([09:38] Diego) e ele mesmo respondeu que acha que não faz parte do escopo, mas que vale registrar como ponto em aberto ([09:39] Diego). | Larissa fechou como **"observar e decidir depois"** ([09:39] Larissa). Não decidido. Depende de dado de produção. |
| **Q2** | **Escala para múltiplos workers.** O desenho atual perde ordenação se rodar mais de um worker. | Diego indicou dois caminhos possíveis — particionar por `order_id` ou usar lock pessimista — mas classificou como "problema do futuro, não agora" ([09:13] Diego). | **Adiado.** Larissa registrou como limitação conhecida ([09:13] Larissa). Não há gatilho definido para reabrir. |
| **Q3** | **Endurecimento da autorização do CRUD de configuração.** Hoje qualquer role autenticada pode criar, editar e remover webhook. | Marcos perguntou se o CRUD pode ser qualquer role autenticada e Sofia respondeu: "por enquanto sim. Mais pra frente a gente pode endurecer" ([09:37] Sofia). | **Não decidido.** Apenas o replay de DLQ tem restrição (`ADMIN`, [09:36] Sofia). Precisa de definição antes de a base de clientes crescer. |
| **Q4** | **Política de retenção e arquivamento da outbox.** | Diego mencionou arquivar linhas entregues "depois de 30 dias ou assim" e declarou fora do escopo desta feature ([09:08] Diego). | **Adiado, sem política definida.** O "ou assim" indica que o número não foi decidido. Agravado por [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md), que persiste payload completo por linha. |
| **Q5** | **Alerta ao cliente sobre webhook quebrado.** Marcos pediu email após 3 falhas consecutivas ([09:37] Marcos). | Larissa: email está fora de escopo desta fase, talvez na próxima, **depois que a gente medir o impacto** ([09:37] Larissa). | **Adiado com condição explícita.** Reabre quando houver medição do impacto. |

---

## 6. Impacto e riscos

### 6.1 Impacto no sistema existente

| Área | Impacto |
| --- | --- |
| `OrderService.changeStatus` | **Alto.** A transação passa a ler configuração de webhooks e inserir N linhas na outbox. É o ponto de maior risco da feature, porque toca o caminho crítico de uma operação de negócio central. |
| Modelo de dados | **Médio.** Quatro tabelas novas em `prisma/schema.prisma`. Nenhuma alteração em tabela existente. |
| Middlewares e erros | **Nenhum.** `errorMiddleware` já trata `AppError`; `authenticate` e `requireRole` são reusados como estão ([09:29] Bruno; [09:36] Larissa). |
| Deploy | **Médio.** Um processo novo (`npm run worker`) com supervisão própria. |
| Dependências | **Nenhuma nova em runtime.** HMAC usa `crypto` do Node; o cliente HTTP pode usar `fetch` nativo. |
| Logger | **Baixo, mas obrigatório.** É preciso acrescentar `*.secret` ao `redact.paths` de `src/shared/logger/index.ts`. |

### 6.2 Riscos

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| **R1 — Degradação de `changeStatus`.** A transação já faz update + history + estoque; acrescentar leitura de config e N inserts pode aumentar a latência e a contenção de lock. | Média | Alto — afeta operação central do OMS | Medir latência da transação antes/depois; limitar o número de endpoints por customer; índices adequados na tabela de configuração; testes de carga na transição `PENDING → PAID`, que é a mais pesada (`src/modules/orders/order.service.ts`, `debitStock`). |
| **R2 — Vazamento de secret em log.** `src/shared/logger/index.ts` não redige `*.secret`; há precedente de vazamento de secret em log de cliente ([09:22] Diego). | Alta, se nada for feito | Alto — comprometimento de autenticidade | Adicionar `*.secret` e `*.signature` ao `redact.paths`; nunca logar o corpo assinado; incluir esse ponto na revisão de segurança de Sofia ([09:46] Sofia). |
| **R3 — Worker cai silenciosamente.** Single-worker é ponto único de falha ([09:12] Diego); eventos se acumulam sem erro aparente. | Média | Alto — parada total da feature sem sinal | Alerta sobre **idade do evento pendente mais antigo**, não sobre taxa de erro; heartbeat do worker; supervisão do processo com restart automático. |
| **R4 — Cliente ignora `X-Event-Id` e processa duplicata.** A garantia é at-least-once e a responsabilidade de dedup é do cliente ([09:25] Sofia, ressalva registrada). | Média | Médio — problema no cliente, percepção de culpa nossa | Documentação em destaque no portal do desenvolvedor, assumida por Marcos ([09:26] Marcos); garantir que o `X-Event-Id` seja estável entre retentativas e no replay de DLQ. |
| **R5 — Crescimento sem controle da outbox.** Sem arquivamento (Q4) e com payload completo por linha ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md)), a tabela cresce indefinidamente. | Alta no médio prazo | Médio — degrada o banco de produção | Filtro na inserção reduz volume ([ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md)); monitorar tamanho da tabela desde o dia 1; fechar Q4 antes de a feature escalar. |
| **R6 — Bug no módulo de webhooks derruba mudança de status.** A inserção está dentro da transação; falha ali causa rollback ([09:40] Bruno). | Baixa | Alto | Cobertura de teste na transação; validação de tamanho de payload (64KB) *antes* da inserção; revisão do caminho por Bruno e Diego na sessão que Larissa marcou ([09:50] Larissa). |

---

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md) | Padrão Outbox transacional no MySQL existente |
| [ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado, polling de 2 segundos |
| [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| [ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h |
| [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com idempotência via `X-Event-Id` |
| [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto |
| [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado na inserção |
| [ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md) | Filtro de status assinados aplicado na inserção |

---

## 8. Como revisar este RFC

Os pontos em que uma objeção mudaria o desenho, e não apenas um detalhe:

1. **A inserção na outbox dentro da transação de `changeStatus`** (R1 + [ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md)) — é o maior risco da proposta e o único que toca o caminho crítico do OMS.
2. **Single-worker como ponto único de falha** (R3 + Q2) — aceito nesta fase, mas sem gatilho definido para reabrir.
3. **Rate limiting de saída** (Q1) — em aberto por decisão consciente; se algum revisor tiver dado de volume por cliente, é a hora de trazer.
4. **Ausência de política de retenção** (Q4 + R5) — a única questão em aberto que piora com o tempo se ninguém decidir.

**Próximo passo:** sessão de revisão com Bruno e Diego ([09:50] Larissa), seguida da revisão de segurança de Sofia, com pelo menos dois dias úteis reservados antes do deploy ([09:46] Sofia).
