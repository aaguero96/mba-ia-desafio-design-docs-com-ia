# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado para desenvolvimento |
| **Data** | 2026-09-21 |
| **Prazo alvo** | Fim de novembro — três sprints ([09:45] Marcos; [09:46] Larissa) |
| **Documentos relacionados** | [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

---

## 1. Resumo e contexto da feature

O OMS passa a notificar ativamente os clientes B2B quando o status de um pedido muda, entregando um evento HTTP assinado no endpoint que o próprio cliente cadastra pela nossa API.

Três clientes fizeram o pedido formalmente na semana anterior à reunião: **Atlas Comercial, MaxDistribuição e Nova Cargo** ([09:00] Marcos). Todos querem saber em tempo real quando os pedidos deles mudam de status na plataforma.

A feature é **outbound apenas** — eventos saem do OMS para o cliente. Sofia levantou a questão de escopo logo no início e Marcos fechou: "só saindo da gente pra eles. Eles querem receber, não mandar" ([09:02] Sofia; [09:02] Marcos).

Do lado técnico, a aplicação não tem hoje nenhum mecanismo de notificação externa, fila ou evento. A entrega envolve, portanto, criar essa capacidade do zero, ancorada na transação de mudança de status que já existe.

---

## 2. Problema e motivação

### 2.1 O problema do cliente

Hoje os clientes descobrem mudanças de status **fazendo polling** em `GET /orders` de tempos em tempos. Marcos descreveu o efeito: isso deixa a integração deles lenta e cara ([09:00] Marcos).

O custo é duplo. Do lado do cliente, requisições repetidas em intervalo fixo que na maior parte das vezes não trazem novidade. Do nosso lado, tráfego de leitura que existe apenas porque não há como avisar.

### 2.2 Pressão comercial

A Atlas Comercial sinalizou que, se a entrega não sair até o fim do trimestre, pode migrar para um concorrente ([09:00] Marcos). A feature é, portanto, retenção de conta, não apenas melhoria de integração.

### 2.3 O que "tempo real" significa aqui

Marcos perguntou especificamente aos clientes. A resposta: **qualquer coisa abaixo de 10 segundos já é tempo real** para eles. O que importa é que o evento não fique pendurado e eles não precisem atualizar manualmente ([09:02] Marcos).

Esse número define o requisito de latência de toda a feature.

---

## 3. Público-alvo e cenários de uso

### 3.1 Público

| Público | Quem é | O que precisa |
| --- | --- | --- |
| **Clientes B2B integrados** | Atlas Comercial, MaxDistribuição, Nova Cargo ([09:00] Marcos) | Receber evento quando o status de um pedido deles muda, sem polling |
| **Usuários da plataforma que representam o cliente** | Contas com JWT do nosso sistema ([09:32] Marcos) | Cadastrar, editar, listar e remover endpoints de webhook; rotacionar secret; consultar histórico de entregas |
| **Time de operação / administradores** | Usuários com role `ADMIN` | Reprocessar eventos que falharam definitivamente ([09:36] Sofia) |

> **Detalhe de identidade.** Bruno apontou que o JWT atual é do usuário operador, não do cliente ([09:32] Bruno). Marcos confirmou que o cadastro é feito pela nossa API, autenticado com JWT do nosso sistema, e que há usuários que representam o cliente ([09:32] Marcos). Larissa fechou: o `customer_id` é passado no body ou no path, **não vem do JWT** ([09:32] Larissa).

### 3.2 Cenários de uso

**C1 — Onboarding da integração.** O cliente cadastra a URL do endpoint dele e escolhe quais status quer ouvir — por exemplo, "só quero saber quando vira `SHIPPED` e `DELIVERED`" ([09:33] Marcos). Recebe a secret na resposta da criação ([09:31] Marcos) e configura a verificação de assinatura no sistema dele.

**C2 — Recebimento no dia a dia.** Um pedido muda de `PROCESSING` para `SHIPPED`. Em até 2 segundos o evento é capturado e entregue, assinado, no endpoint do cliente. O cliente responde `2xx` e atualiza o sistema dele sem ter feito nenhuma requisição.

**C3 — Indisponibilidade do cliente.** O endpoint do cliente está fora por manutenção. O OMS retenta em 1min, 5min, 30min, 2h e 12h. Se o cliente voltar dentro dessa janela — o caso real de 2 horas que o time já viveu ([09:16] Diego) —, o evento é entregue sem intervenção.

**C4 — Falha definitiva e recuperação.** O cliente não volta em ~15 horas. O evento vai para a DLQ. Depois de resolvido, um administrador reprocessa via endpoint de replay ([09:18] Diego), e a ação fica registrada para auditoria ([09:36] Sofia).

**C5 — Rotação de secret.** O cliente suspeita de vazamento — cenário com precedente real ([09:22] Diego) — e pede nova secret pela API. Durante 24 horas a antiga continua válida, o que dá tempo de migrar os sistemas dele sem perder entrega ([09:21] Sofia).

**C6 — Investigação de divergência.** O cliente afirma não ter recebido um evento. Ele consulta o histórico das últimas entregas — sucesso/falha, payload, response e tempo de resposta ([09:34] Marcos) — e resolve sozinho, sem abrir chamado.

---

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| **OBJ-1** | Entregar notificação dentro da janela que os clientes chamam de tempo real | Tempo entre o commit da mudança de status e a primeira tentativa de entrega | **≤ 2 segundos**, com teto absoluto de **10 segundos** | [09:02] Marcos (teto de 10s); [09:10] Larissa (polling de 2s aceito) |
| **OBJ-2** | Eliminar o polling em `GET /orders` dos clientes que pediram a feature | Número de clientes B2B migrados de polling para webhook | **3 de 3** — Atlas Comercial, MaxDistribuição e Nova Cargo | [09:00] Marcos |
| **OBJ-3** | Não perder nenhum evento de mudança de status assinado | Divergência entre transições registradas em `order_status_history` e eventos criados na outbox para status assinados | **0** | [09:40] Bruno — "não pode ter caso de status mudar e evento não sair" |
| **OBJ-4** | Reter a conta em risco | Confirmação de prazo aceita pela Atlas | Entrega até **o fim de novembro** | [09:00] Marcos (risco de migração); [09:45] Marcos (prazo); [09:47] Marcos |
| **OBJ-5** | Absorver indisponibilidade real de cliente sem intervenção manual | Percentual de eventos entregues com sucesso após retry, sem chegar à DLQ, para clientes com indisponibilidade ≤ 2h | Cobrir integralmente a janela de **2 horas** do incidente conhecido | [09:16] Diego |
| **OBJ-6** | Não degradar o caminho crítico do OMS | Latência da transação de `changeStatus`, comparada ao baseline anterior à feature | Sem regressão significativa | [09:04] Bruno (motivação contra o síncrono) |

> **Nota de honestidade sobre as metas.** Os números acima vêm todos da reunião. Onde a transcrição não fixou um alvo — OBJ-6, por exemplo, não teve percentual definido —, a meta está escrita como direção, não como número inventado. Definir esses limiares é tarefa da implementação, com baseline medido.

---

## 5. Escopo

### 5.1 Incluído nesta fase

1. Cadastro, listagem, edição e remoção de endpoints de webhook via API ([09:31]–[09:33] Marcos, Bruno).
2. Geração da secret pelo OMS na criação, devolvida ao cliente naquele momento ([09:31] Marcos).
3. Rotação de secret via API, com grace period de 24 horas ([09:21] Sofia).
4. Filtro por status: cada endpoint declara quais status quer ouvir ([09:33] Marcos).
5. Captura transacional do evento junto à mudança de status ([09:06] Diego; [09:40] Bruno).
6. Entrega HTTP assinada com HMAC-SHA256 ([09:22] Sofia).
7. Retry automático com backoff exponencial e DLQ ([09:17] Larissa; [09:18] Diego).
8. Endpoint de histórico de entregas ([09:34] Marcos).
9. Endpoint administrativo de replay de DLQ, restrito a `ADMIN` ([09:35] Diego; [09:36] Sofia).

### 5.2 Fora de escopo

| Item | O que foi dito | Classificação |
| --- | --- | --- |
| **Email de alerta ao cliente quando o webhook dele falha** | Marcos perguntou se dá para avisar o cliente por email depois de 3 falhas seguidas ([09:37] Marcos). Larissa: "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" ([09:37] Larissa). Marcos anotou como "futuro" ([09:38] Marcos). | **Adiado**, com condição explícita: medir o impacto primeiro |
| **Dashboard / painel visual para o cliente** | Marcos perguntou sobre painel para o cliente ver os webhooks dele ([09:39] Marcos). Larissa: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" ([09:40] Larissa). | **Descartado desta feature** — pertence a outro time |
| **Rate limiting de envio para o cliente** | Diego levantou o cenário de 50 pedidos mudando em um minuto e bombardear o cliente ([09:38] Diego); concluiu que não faz parte do escopo, mas vale registrar como ponto em aberto ([09:39] Diego). Larissa: "observar e decidir depois" ([09:39] Larissa). | **Adiado**, condicionado a observação em produção |
| **Webhooks inbound (cliente → OMS)** | Sofia perguntou se os clientes também enviariam para nós; Marcos: "só saindo da gente pra eles" ([09:02] Sofia; [09:02] Marcos). | **Descartado** — escopo direcional fechado |
| **Arquivamento das linhas entregues da outbox** | Diego: linhas entregues serão arquivadas "depois de 30 dias ou assim, fora do escopo dessa feature" ([09:08] Diego). | **Adiado**, sem política definida |
| **Ordenação global entre pedidos diferentes** | Diego: com múltiplos workers a garantia se perde; particionamento ou lock pessimista é "problema do futuro" ([09:13] Diego). Larissa documentou como limitação conhecida ([09:13] Larissa). Marcos: os clientes nunca pediram ordering global ([09:14] Marcos). | **Limitação assumida** |
| **Entrega exactly-once** | Diego descartou em favor de at-least-once com `event_id`, alinhado a Stripe e GitHub ([09:25] Diego). | **Descartado** |
| **Autorização restrita no CRUD de configuração** | Marcos perguntou se o CRUD pode ser qualquer role autenticada; Sofia: "por enquanto sim. Mais pra frente a gente pode endurecer" ([09:37] Sofia). | **Adiado** — só o replay exige `ADMIN` nesta fase |

---

## 6. Requisitos funcionais

| # | Requisito | Origem |
| --- | --- | --- |
| **RF-01** | O cliente cadastra um endpoint de webhook informando a URL, a lista de status que quer receber e o `customer_id`. A secret é gerada pelo OMS e devolvida na resposta da criação. | [09:31] Marcos; [09:32] Larissa |
| **RF-02** | O cliente lista os endpoints de webhook de um customer. | [09:33] Bruno |
| **RF-03** | O cliente edita um endpoint cadastrado. | [09:33] Bruno |
| **RF-04** | O cliente remove um endpoint cadastrado. | [09:33] Bruno |
| **RF-05** | Cada endpoint define quais status quer ouvir. Se nenhum endpoint do customer assina aquele status, o evento não é sequer criado. | [09:33] Marcos; [09:34] Bruno |
| **RF-06** | O cliente rotaciona a secret pela API. A secret antiga permanece válida por 24 horas em paralelo e depois é invalidada. | [09:21] Sofia |
| **RF-07** | O cliente consulta o histórico de entregas de um endpoint, com sucesso/falha, payload, response e tempo de resposta. | [09:34] Marcos |
| **RF-08** | Quando o status de um pedido muda, o evento é registrado na mesma transação que atualiza o pedido e o histórico de status. Falha no registro reverte a mudança de status. | [09:06] Diego; [09:40] Bruno |
| **RF-09** | O evento é entregue por `POST` HTTP no endpoint do cliente, com os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. | [09:44] Diego; [09:44] Sofia |
| **RF-10** | O corpo do evento contém `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. Os itens do pedido **não** são enviados. | [09:43] Diego |
| **RF-11** | Entregas que falham são retentadas automaticamente com backoff exponencial, até 5 tentativas, na progressão 1min / 5min / 30min / 2h / 12h. | [09:15] Diego; [09:17] Diego; [09:17] Larissa |
| **RF-12** | Esgotadas as 5 tentativas, o evento é movido para uma dead letter queue persistida em tabela separada, com payload, motivo da falha e timestamp. | [09:18] Diego |
| **RF-13** | Um administrador reprocessa um item da DLQ por endpoint dedicado, que recoloca o evento na fila como pendente. A operação exige role `ADMIN` e registra quem a executou, para auditoria. | [09:18] Diego; [09:35] Diego; [09:36] Sofia; [09:36] Larissa |
| **RF-14** | O endpoint cadastrado tem estado ativo/inativo. | [09:21] Bruno; [09:21] Sofia |
| **RF-15** | O payload entregue é o snapshot do estado do pedido no momento em que o status mudou, e não uma renderização no momento do envio. | [09:52] Larissa; [09:52] Diego |

---

## 7. Requisitos não funcionais

| # | Requisito | Origem |
| --- | --- | --- |
| **RNF-01** | **Latência.** A captação do evento ocorre em até 2 segundos após o commit; o teto aceitável de ponta a ponta é 10 segundos. | [09:02] Marcos; [09:09] Diego; [09:10] Larissa |
| **RNF-02** | **TLS obrigatório.** A URL do webhook precisa usar `https`. Cadastro com `http` é recusado com erro de validação. | [09:23] Sofia |
| **RNF-03** | **Limite de payload.** Eventos acima de 64KB geram erro. O payload **não** é truncado. | [09:23] Sofia; [09:24] Diego; [09:24] Larissa |
| **RNF-04** | **Timeout de entrega.** Cada tentativa tem timeout de 10 segundos; cliente que não responde nesse prazo é tratado como falha. | [09:42] Diego |
| **RNF-05** | **Isolamento de secret.** Cada endpoint tem secret única. Não existe secret global da plataforma. | [09:21] Sofia |
| **RNF-06** | **Isolamento de processo.** O worker de entrega roda fora do processo da API, para que restart da API não o derrube. | [09:11] Diego |
| **RNF-07** | **Atomicidade.** O registro do evento é atômico com a mudança de status: ou ambos acontecem, ou nenhum. | [09:06] Diego; [09:41] Diego |
| **RNF-08** | **Ordenação.** Garantida por `order_id` enquanto houver um único worker. Não há garantia de ordem global entre pedidos diferentes. | [09:12] Diego; [09:13] Larissa |
| **RNF-09** | **Garantia de entrega.** At-least-once. O cliente pode receber o mesmo evento mais de uma vez e deduplica pelo `X-Event-Id`. | [09:24] Diego; [09:26] Larissa |
| **RNF-10** | **Sem infraestrutura nova.** A feature usa o MySQL existente; nada de broker ou serviço adicional. | [09:07] Diego |
| **RNF-11** | **Aderência aos padrões do projeto.** Módulo em `src/modules/webhooks`, erros estendendo `AppError` com prefixo `WEBHOOK_`, logging com Pino, middleware de erro existente sem alteração. | [09:28] Bruno; [09:29] Larissa; [09:30] Larissa |
| **RNF-12** | **Identificadores.** UUID, seguindo o padrão de todos os modelos do projeto. | [09:51] Larissa |
| **RNF-13** | **Auditoria.** O replay de DLQ registra qual usuário o executou. | [09:36] Sofia |

---

## 8. Decisões e trade-offs principais

Cada decisão abaixo tem um ADR dedicado, com contexto, alternativas e consequências.

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| **Outbox transacional no MySQL**, em vez de disparo síncrono | Abre mão de latência mínima e acopla o módulo à transação de negócio, em troca de consistência garantida e zero infraestrutura nova | [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md) |
| **Worker separado em polling de 2s**, em vez de trigger ou worker embutido | Abre mão de reatividade sub-segundo e de escala horizontal, em troca de simplicidade e de ordenação por pedido sem coordenação | [ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| **5 tentativas com backoff até ~15h e DLQ em tabela separada** | Eventos ocupam a fila por mais tempo durante incidentes, em troca de tolerar indisponibilidade real de cliente — calibrado pelo caso concreto de 2h ([09:16] Diego) | [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| **HMAC-SHA256 com secret por endpoint e rotação com grace de 24h** | Passamos a guardar N segredos de terceiros e a conviver com duas secrets válidas por 24h, em troca de raio de explosão limitado e rotação sem downtime | [ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| **At-least-once com `X-Event-Id`**, em vez de exactly-once | Transfere a deduplicação para o cliente — ressalva registrada por Sofia ([09:25] Sofia) — em troca de nunca perder evento silenciosamente | [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| **Reuso máximo dos padrões existentes** | O worker fica desconfortável no formato controller/service/repository, em troca de custo cognitivo zero e reaproveitamento de infraestrutura testada | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| **Snapshot do payload na inserção** | Duplica dados e congela o formato dos eventos já enfileirados, em troca de fidelidade histórica do evento | [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| **Filtro de status aplicado na inserção** | Deixa a transação de `changeStatus` mais pesada e torna assinaturas não-retroativas, em troca de uma fila que só contém trabalho real | [ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md) |

---

## 9. Dependências

### 9.1 Dependências de produto

| Dependência | Responsável | Status |
| --- | --- | --- |
| Documentação no portal do desenvolvedor explicando como integrar via API | Marcos ([09:40] Marcos) | Assumido |
| Documentação em destaque sobre a garantia at-least-once e a necessidade de deduplicar pelo `X-Event-Id` | Marcos ([09:26] Marcos) | Assumido — **bloqueante para a qualidade da integração** |
| Confirmação do prazo com os clientes | Marcos ([09:47] Marcos) | Assumido |

### 9.2 Dependências técnicas

| Dependência | Detalhe |
| --- | --- |
| MySQL existente | Mesmo banco, mesma `DATABASE_URL`, sem instância nova ([09:07] Diego; [09:30] Bruno) |
| Transação de `changeStatus` | A feature se insere dentro da transação já existente ([09:40] Bruno) |
| Padrões do projeto | `AppError`, Pino, middleware de erro, estrutura de módulos, schemas Zod ([09:30] Larissa) |
| `requireRole` existente | Reaproveitado no endpoint de replay ([09:36] Larissa) |
| Processo supervisionado para o worker | Novo artefato de deploy ([09:11] Diego) |

### 9.3 Dependência de processo — bloqueante

**Revisão de segurança por Sofia antes do deploy**, com pelo menos **dois dias úteis** reservados. Ela quer olhar HMAC e geração de secret com calma ([09:46] Sofia). Larissa incluiu a revisão dentro das três sprints ([09:47] Larissa) e Sofia reforçou no encerramento: "só não esqueçam de me agendar pra revisão de segurança antes de subir" ([09:49] Sofia).

---

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **R1** | **Degradação da mudança de status.** A transação de `changeStatus` já atualiza pedido, histórico e estoque; a feature acrescenta leitura de configuração e inserção de eventos no mesmo caminho crítico. | **Média** | **Alto** — afeta a operação central do OMS, não só a feature | Medir a latência da transação antes e depois; filtro na inserção reduz o número de linhas ([09:34] Bruno); índices adequados na tabela de configuração; teste de carga na transição `PENDING → PAID`, a mais pesada. |
| **R2** | **Vazamento de secret.** Guardamos segredos de terceiros, e há precedente de vazamento de secret em log de aplicação ([09:22] Diego). O redact do logger atual não cobre campos de secret. | **Alta**, se nada for feito | **Alto** — comprometimento da autenticidade das entregas | Rotação auto-serviço com grace de 24h ([09:21] Sofia); secret nunca devolvida em listagem; ajuste do redact do logger; revisão de segurança de Sofia ([09:46] Sofia). |
| **R3** | **Worker cai silenciosamente.** É um processo único ([09:12] Diego); se morrer, não há erro — há silêncio, e os eventos se acumulam sem ninguém notar. | **Média** | **Alto** — parada total da feature sem sinal | Alerta baseado na idade do evento pendente mais antigo, não na taxa de erro; supervisão do processo com restart automático; heartbeat do worker. |
| **R4** | **Cliente ignora o `X-Event-Id` e processa pedidos duplicados.** A garantia é at-least-once e a dedup é responsabilidade dele — ressalva que Sofia registrou em ata ([09:25] Sofia). | **Média** | **Médio** — problema no cliente, percepção de culpa nossa | Documentação em destaque no portal ([09:26] Marcos); garantir que o `X-Event-Id` é estável entre retentativas e no replay. |
| **R5** | **Crescimento sem controle da fila de eventos.** Não há política de arquivamento nesta fase ([09:08] Diego) e cada linha guarda o payload completo ([09:52] Larissa). | **Alta** no médio prazo | **Médio** — degrada o banco de produção | Filtro na inserção reduz volume; monitorar o tamanho da tabela desde o primeiro dia; definir a política de retenção antes da feature escalar. |
| **R6** | **Prazo apertado.** Três sprints com revisão de segurança incluída ([09:46] Larissa), contra um cliente que ameaça migrar ([09:00] Marcos). | **Média** | **Alto** — risco comercial direto | Escopo já enxugado na reunião (email, dashboard e rate limiting fora); sessão de revisão do design com Bruno e Diego antes de codar ([09:50] Larissa); dois dias de Sofia reservados desde já ([09:46] Sofia). |

---

## 11. Critérios de aceitação

| # | Critério | Origem |
| --- | --- | --- |
| **CA-01** | Um cliente consegue cadastrar um endpoint informando URL `https`, lista de status e `customer_id`, e recebe a secret na resposta. | RF-01 |
| **CA-02** | Cadastro com URL `http` é recusado com erro de validação. | RNF-02 |
| **CA-03** | Um cliente consegue listar, editar e remover os endpoints de um customer. | RF-02, RF-03, RF-04 |
| **CA-04** | Um endpoint que assina apenas `SHIPPED` e `DELIVERED` não recebe eventos de `PAID` nem de `PROCESSING`. | RF-05 |
| **CA-05** | Mudança de status de um pedido com assinante gera entrega no endpoint do cliente em até 10 segundos. | OBJ-1, RNF-01 |
| **CA-06** | A entrega chega com os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`, e o corpo contém exatamente os campos de RF-10, sem os itens do pedido. | RF-09, RF-10 |
| **CA-07** | A assinatura em `X-Signature` é verificável pelo cliente com HMAC-SHA256 e a secret do endpoint. | RNF-05 |
| **CA-08** | Se a mudança de status é revertida, nenhum evento é criado. | RF-08, RNF-07 |
| **CA-09** | Endpoint indisponível gera retentativas em 1min, 5min, 30min, 2h e 12h. | RF-11 |
| **CA-10** | Após a quinta falha o evento aparece na DLQ, com payload, motivo e timestamp. | RF-12 |
| **CA-11** | Um usuário com role `ADMIN` consegue reprocessar um item da DLQ; um usuário sem essa role recebe erro de permissão. | RF-13 |
| **CA-12** | O replay registra qual usuário o executou. | RNF-13 |
| **CA-13** | Após rotacionar a secret, entregas assinadas com a secret anterior continuam verificáveis por 24 horas; depois disso, não mais. | RF-06 |
| **CA-14** | O histórico de entregas mostra, por tentativa, sucesso/falha, payload, response e tempo de resposta. | RF-07 |
| **CA-15** | Um evento acima de 64KB gera erro e não é truncado nem entregue. | RNF-03 |
| **CA-16** | Um endpoint que não responde em 10 segundos é tratado como falha e entra em retry. | RNF-04 |
| **CA-17** | O mesmo evento entregue mais de uma vez carrega sempre o mesmo `X-Event-Id`. | RNF-09 |
| **CA-18** | Reiniciar a API não interrompe a entrega de eventos pendentes. | RNF-06 |

Os critérios de aceite **técnicos** — cobertura de código, contratos internos e verificações de implementação — estão no [FDD, seção 12](./FDD.md#12-critérios-de-aceite-técnicos).

---

## 12. Estratégia de testes e validação

### 12.1 Níveis de teste

| Nível | O que cobre | Ferramenta |
| --- | --- | --- |
| **Unitário** | Cálculo de HMAC, progressão do backoff, filtro de status assinados, validação de URL `https` e do limite de 64KB | Vitest, já configurado em `vitest.config.ts` |
| **Integração** | Atomicidade da inserção junto ao `changeStatus` — inclusive o caso de rollback —, ciclo completo de retry, movimentação para DLQ e replay | Vitest + banco de teste, no padrão de `tests/orders.test.ts` |
| **Contrato** | Payload e headers entregues ao cliente, conforme RF-09 e RF-10 | Servidor HTTP de teste que captura a requisição |
| **Carga** | Impacto na latência de `changeStatus`, especialmente na transição `PENDING → PAID`, que também mexe em estoque | Teste de carga com baseline anterior à feature (R1) |

### 12.2 Cenários de validação obrigatórios

1. **Caminho feliz.** Mudança de status com assinante → entrega em ≤ 2s, resposta `2xx`, evento marcado como entregue.
2. **Sem assinante.** Mudança de status sem nenhum endpoint que assine aquele status → nenhum evento criado ([09:34] Bruno).
3. **Rollback.** Transação revertida por estoque insuficiente → nenhum evento criado (CA-08).
4. **Falha na criação do evento.** Erro na inserção → mudança de status revertida ([09:40] Bruno).
5. **Indisponibilidade de 2 horas.** Cliente fora por 2h → evento entregue na retentativa, sem chegar à DLQ. Reproduz o incidente real citado por Diego ([09:16] Diego) e valida OBJ-5.
6. **Falha definitiva.** Cliente fora por mais de 15h → evento na DLQ, replay bem-sucedido depois.
7. **Rotação de secret.** Entrega com a secret nova verificável; entrega assinada com a anterior ainda verificável dentro das 24h e não mais depois (CA-13).
8. **Payload acima do limite.** Evento > 64KB → erro, sem truncamento (CA-15).
9. **Timeout.** Endpoint que demora 11 segundos → tratado como falha (CA-16).
10. **Autorização.** Replay com role `OPERATOR` → negado; com `ADMIN` → aceito e auditado (CA-11, CA-12).

### 12.3 Validação em produção

- **Rollout gradual:** começar por um dos três clientes, validar o comportamento real, e só então liberar para os demais.
- **Métrica de guarda:** latência de `changeStatus` comparada ao baseline (R1). Regressão significativa é motivo de rollback.
- **Alerta de silêncio:** idade do evento pendente mais antigo, que é o único sinal confiável de worker morto (R3).
- **Validação pelo cliente:** o endpoint de histórico de entregas (RF-07) é o que permite ao cliente confirmar o recebimento sem abrir chamado.

### 12.4 Gate de segurança

Nenhum deploy antes da revisão de Sofia sobre HMAC e geração de secret, com **pelo menos dois dias úteis** reservados ([09:46] Sofia; reforçado em [09:49] Sofia).
