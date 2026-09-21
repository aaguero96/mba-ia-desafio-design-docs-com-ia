# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado no pacote de documentação e sua origem — a transcrição da reunião (`TRANSCRICAO.md`) ou o código-fonte da aplicação.

**Regra aplicada:** todo item que não pôde ter a coluna *Localização* preenchida foi removido dos documentos. Nada aqui é invenção da IA.

**Convenção da coluna Fonte:**

- `TRANSCRICAO` → localização no formato `[hh:mm] Nome`, conforme o formato da ata.
- `CODIGO` → caminho real de arquivo do repositório, verificado antes de ser citado.

---

## 1. Resumo da cobertura

| Métrica | Valor |
| --- | --- |
| Total de linhas | **186** |
| Fonte = `TRANSCRICAO` | **147** (79,0%) |
| Fonte = `CODIGO` | **39** (21,0%) |
| Linhas `TRANSCRICAO` com timestamp `[hh:mm] Nome` válido | **147** (100%) |
| Linhas por documento | PRD 79 · RFC 31 · FDD 61 · ADRs 15 |
| Documentos cobertos | `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, os 8 ADRs |

---

## 2. Tabela de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| **PRD-CTX-01** | docs/PRD.md | Contexto | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| **PRD-CTX-02** | docs/PRD.md | Problema | Clientes hoje fazem polling em `GET /orders`, o que torna a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| **PRD-CTX-03** | docs/PRD.md | Restrição | Atlas pode migrar para concorrente se não for entregue até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| **PRD-CTX-04** | docs/PRD.md | Restrição | "Tempo real" definido pelos clientes como abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| **PRD-CTX-05** | docs/PRD.md | Escopo | Feature é outbound-only: eventos saem do OMS para o cliente | TRANSCRICAO | [09:02] Marcos |
| **PRD-CTX-06** | docs/PRD.md | Escopo | Pergunta de escopo sobre direção do webhook levantada antes da arquitetura | TRANSCRICAO | [09:02] Sofia |
| **PRD-CTX-07** | docs/PRD.md | Contexto | A aplicação não possui hoje nenhum mecanismo de notificação, fila ou evento | CODIGO | `src/routes/index.ts` (só auth, users, customers, products, orders) |
| **PRD-PUB-01** | docs/PRD.md | Restrição | JWT atual é do usuário operador, não do cliente | TRANSCRICAO | [09:32] Bruno |
| **PRD-PUB-02** | docs/PRD.md | Decisão | Cadastro feito pela nossa API, com JWT do nosso sistema; há usuários que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| **PRD-PUB-03** | docs/PRD.md | Decisão | `customer_id` vem no body ou no path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| **PRD-PUB-04** | docs/PRD.md | Restrição | Roles disponíveis no sistema são `ADMIN` e `OPERATOR` | CODIGO | `prisma/schema.prisma` (enum `UserRole`) |
| **PRD-OBJ-01** | docs/PRD.md | Métrica | Latência de captação ≤ 2s, teto absoluto de 10s | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| **PRD-OBJ-02** | docs/PRD.md | Métrica | Migrar 3 de 3 clientes B2B de polling para webhook | TRANSCRICAO | [09:00] Marcos |
| **PRD-OBJ-03** | docs/PRD.md | Métrica | Zero divergência entre transições de status e eventos criados | TRANSCRICAO | [09:40] Bruno |
| **PRD-OBJ-04** | docs/PRD.md | Métrica | Entrega até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| **PRD-OBJ-05** | docs/PRD.md | Métrica | Cobrir integralmente indisponibilidade de cliente de até 2 horas | TRANSCRICAO | [09:16] Diego |
| **PRD-OBJ-06** | docs/PRD.md | Métrica | Não degradar a latência da transação de mudança de status | TRANSCRICAO | [09:04] Bruno |
| **PRD-RF-01** | docs/PRD.md | Requisito Funcional | Cadastro de webhook por POST, com secret gerada por nós e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| **PRD-RF-02** | docs/PRD.md | Requisito Funcional | GET para listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| **PRD-RF-03** | docs/PRD.md | Requisito Funcional | PATCH para editar webhook | TRANSCRICAO | [09:33] Bruno |
| **PRD-RF-04** | docs/PRD.md | Requisito Funcional | DELETE para remover webhook | TRANSCRICAO | [09:33] Bruno |
| **PRD-RF-05** | docs/PRD.md | Requisito Funcional | Filtro de eventos: lista de status que cada endpoint quer ouvir | TRANSCRICAO | [09:33] Marcos |
| **PRD-RF-05b** | docs/PRD.md | Decisão | Se nenhum webhook do customer quer aquele status, nem insere | TRANSCRICAO | [09:34] Bruno |
| **PRD-RF-06** | docs/PRD.md | Requisito Funcional | Rotação de secret pela API, com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| **PRD-RF-07** | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| **PRD-RF-08** | docs/PRD.md | Requisito Funcional | Evento inserido na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| **PRD-RF-08b** | docs/PRD.md | Restrição | Falha na inserção do evento causa rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| **PRD-RF-09** | docs/PRD.md | Requisito Funcional | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| **PRD-RF-09b** | docs/PRD.md | Requisito Funcional | Header adicional `X-Webhook-Id` com o id do cadastro | TRANSCRICAO | [09:44] Sofia |
| **PRD-RF-10** | docs/PRD.md | Requisito Funcional | Payload com event_id, event_type, timestamp ISO 8601, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| **PRD-RF-11** | docs/PRD.md | Requisito Funcional | Retry automático com backoff exponencial, 5 tentativas | TRANSCRICAO | [09:15] Diego |
| **PRD-RF-11b** | docs/PRD.md | Decisão | Progressão do backoff: 1m / 5m / 30m / 2h / 12h | TRANSCRICAO | [09:17] Diego |
| **PRD-RF-12** | docs/PRD.md | Requisito Funcional | DLQ em tabela separada, com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| **PRD-RF-13** | docs/PRD.md | Requisito Funcional | Endpoint admin `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Diego |
| **PRD-RF-13b** | docs/PRD.md | Restrição | Replay exige role `ADMIN` e loga quem executou, para auditoria | TRANSCRICAO | [09:36] Sofia |
| **PRD-RF-14** | docs/PRD.md | Requisito Funcional | Cadastro guarda url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| **PRD-RF-15** | docs/PRD.md | Decisão | Payload é snapshot renderizado na inserção, não na hora do envio | TRANSCRICAO | [09:52] Larissa |
| **PRD-RNF-01** | docs/PRD.md | Requisito Não Funcional | Polling de 2 segundos define a latência de captação | TRANSCRICAO | [09:09] Diego |
| **PRD-RNF-02** | docs/PRD.md | Requisito Não Funcional | TLS obrigatório: URL tem que ser https; http é recusado na validação | TRANSCRICAO | [09:23] Sofia |
| **PRD-RNF-03** | docs/PRD.md | Requisito Não Funcional | Limite de 64KB de payload, com erro — não truncamento | TRANSCRICAO | [09:24] Diego; [09:24] Larissa |
| **PRD-RNF-03b** | docs/PRD.md | Trade-off | Posição de "erra, não trunca" quando o payload é grande demais | TRANSCRICAO | [09:23] Sofia |
| **PRD-RNF-04** | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| **PRD-RNF-05** | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, não global da plataforma | TRANSCRICAO | [09:21] Sofia |
| **PRD-RNF-06** | docs/PRD.md | Requisito Não Funcional | Worker roda como processo separado da API | TRANSCRICAO | [09:11] Diego |
| **PRD-RNF-07** | docs/PRD.md | Requisito Não Funcional | Atomicidade: fora da transação perde a garantia toda | TRANSCRICAO | [09:41] Diego |
| **PRD-RNF-08** | docs/PRD.md | Restrição | Ordenação garantida por order_id apenas enquanto single-worker | TRANSCRICAO | [09:12] Diego |
| **PRD-RNF-08b** | docs/PRD.md | Restrição | Ausência de ordering global documentada como limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| **PRD-RNF-09** | docs/PRD.md | Decisão | Garantia at-least-once; cliente pode receber o mesmo evento duas vezes | TRANSCRICAO | [09:24] Diego |
| **PRD-RNF-10** | docs/PRD.md | Restrição | Sem infraestrutura nova; Redis Cluster seria overengineering para o time | TRANSCRICAO | [09:07] Diego |
| **PRD-RNF-11** | docs/PRD.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, schemas Zod, códigos de erro | TRANSCRICAO | [09:30] Larissa |
| **PRD-RNF-12** | docs/PRD.md | Decisão | Identificadores em UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| **PRD-RNF-12b** | docs/PRD.md | Restrição | Todos os modelos existentes usam `@id @default(uuid()) @db.Char(36)` | CODIGO | `prisma/schema.prisma` |
| **PRD-RNF-13** | docs/PRD.md | Requisito Não Funcional | Auditoria do replay de DLQ registrando o usuário executor | TRANSCRICAO | [09:36] Sofia |
| **PRD-FORA-01** | docs/PRD.md | Restrição | Email de alerta ao cliente fora de escopo desta fase; talvez a próxima, após medir impacto | TRANSCRICAO | [09:37] Larissa |
| **PRD-FORA-01b** | docs/PRD.md | Restrição | Pedido original de email após 3 falhas seguidas, anotado como "futuro" | TRANSCRICAO | [09:37] Marcos; [09:38] Marcos |
| **PRD-FORA-02** | docs/PRD.md | Restrição | Dashboard visual fora de escopo; painel é projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| **PRD-FORA-03** | docs/PRD.md | Restrição | Rate limiting de saída fora de escopo: "observar e decidir depois" | TRANSCRICAO | [09:39] Larissa |
| **PRD-FORA-04** | docs/PRD.md | Restrição | Webhooks inbound fora de escopo | TRANSCRICAO | [09:02] Marcos |
| **PRD-FORA-05** | docs/PRD.md | Restrição | Arquivamento de linhas entregues (~30 dias) fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| **PRD-FORA-06** | docs/PRD.md | Restrição | Múltiplos workers e ordering global são "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| **PRD-FORA-07** | docs/PRD.md | Restrição | Exactly-once descartado em favor de at-least-once | TRANSCRICAO | [09:25] Diego |
| **PRD-FORA-08** | docs/PRD.md | Restrição | CRUD de configuração aceita qualquer role autenticada por enquanto | TRANSCRICAO | [09:37] Sofia |
| **PRD-DEP-01** | docs/PRD.md | Dependência | Marcos documenta no portal como integrar via API | TRANSCRICAO | [09:40] Marcos |
| **PRD-DEP-02** | docs/PRD.md | Dependência | Marcos documenta a garantia at-least-once em destaque no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| **PRD-DEP-03** | docs/PRD.md | Dependência | Marcos confirma prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| **PRD-DEP-04** | docs/PRD.md | Dependência | Revisão de segurança de Sofia, com dois dias úteis reservados antes do deploy | TRANSCRICAO | [09:46] Sofia |
| **PRD-DEP-04b** | docs/PRD.md | Dependência | Sofia reforça no encerramento o agendamento da revisão antes de subir | TRANSCRICAO | [09:49] Sofia |
| **PRD-DEP-05** | docs/PRD.md | Dependência | Estimativa de três sprints com a revisão de segurança incluída | TRANSCRICAO | [09:46] Larissa; [09:47] Larissa |
| **PRD-R1** | docs/PRD.md | Risco | Degradação de `changeStatus`: transação já atualiza order, history e estoque | TRANSCRICAO | [09:04] Bruno |
| **PRD-R1b** | docs/PRD.md | Risco | Transação atual faz update de order, insert em history e debita/repõe estoque | CODIGO | `src/modules/orders/order.service.ts` (`changeStatus`, linhas 131–178) |
| **PRD-R2** | docs/PRD.md | Risco | Vazamento de secret: já houve cliente que vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| **PRD-R2b** | docs/PRD.md | Risco | `redactPaths` do logger não cobre campos de secret | CODIGO | `src/shared/logger/index.ts` (linhas 4–11) |
| **PRD-R3** | docs/PRD.md | Risco | Single-worker como ponto único de falha | TRANSCRICAO | [09:12] Diego |
| **PRD-R4** | docs/PRD.md | Risco | At-least-once joga a responsabilidade de dedup para o cliente | TRANSCRICAO | [09:25] Sofia |
| **PRD-R5** | docs/PRD.md | Risco | Crescimento da outbox sem política de arquivamento definida | TRANSCRICAO | [09:08] Diego |
| **PRD-R6** | docs/PRD.md | Risco | Prazo de três sprints contra cliente que ameaça migrar | TRANSCRICAO | [09:00] Marcos; [09:46] Larissa |
| **PRD-TEST-01** | docs/PRD.md | Estratégia de Teste | Testes seguem o setup Vitest já existente no projeto | CODIGO | `vitest.config.ts`, `tests/setup.ts` |
| **PRD-TEST-02** | docs/PRD.md | Estratégia de Teste | Testes de integração no padrão dos existentes de pedidos | CODIGO | `tests/orders.test.ts` |
| **PRD-TEST-03** | docs/PRD.md | Estratégia de Teste | Cenário obrigatório: indisponibilidade de 2h do cliente coberta pelo retry | TRANSCRICAO | [09:16] Diego |
| **RFC-META-01** | docs/RFC.md | Contexto | Revisores do RFC são os participantes da reunião | TRANSCRICAO | [09:00] Larissa (lista de participantes) |
| **RFC-META-02** | docs/RFC.md | Contexto | Larissa abre o doc de design e marca sessão de revisão com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| **RFC-PROP-01** | docs/RFC.md | Decisão | Padrão outbox: linha inserida na mesma transação que atualiza orders e history | TRANSCRICAO | [09:06] Diego |
| **RFC-PROP-02** | docs/RFC.md | Decisão | Worker em polling a cada 2 segundos, lendo os pendentes mais antigos | TRANSCRICAO | [09:09] Diego |
| **RFC-PROP-03** | docs/RFC.md | Decisão | Nova entry-point `src/worker.ts` e script `npm run worker`, espelhando `src/server.ts` | TRANSCRICAO | [09:11] Larissa |
| **RFC-PROP-03b** | docs/RFC.md | Restrição | `src/server.ts` existe como modelo de entry-point, com shutdown gracioso | CODIGO | `src/server.ts` |
| **RFC-PROP-04** | docs/RFC.md | Decisão | Worker usa `PrismaClient` próprio: client é por processo | TRANSCRICAO | [09:30] Bruno |
| **RFC-PROP-05** | docs/RFC.md | Decisão | Índices na outbox por status e `created_at`; leitura em batch pequeno | TRANSCRICAO | [09:08] Diego |
| **RFC-PROP-06** | docs/RFC.md | Decisão | Estados da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| **RFC-PROP-07** | docs/RFC.md | Decisão | Endpoints sob o prefixo `/api/v1`, imposto pelo código existente | CODIGO | `src/app.ts` (linha 67) |
| **RFC-ALT-01** | docs/RFC.md | Trade-off | Alternativa descartada: disparo síncrono no service de orders | TRANSCRICAO | [09:03] Larissa; [09:06] Diego |
| **RFC-ALT-01b** | docs/RFC.md | Trade-off | Motivo do descarte: HTTP na transação faria cliente lento travar outros pedidos | TRANSCRICAO | [09:04] Bruno |
| **RFC-ALT-01c** | docs/RFC.md | Trade-off | Motivo do descarte: cliente fora do ar exigiria rollback da mudança de status | TRANSCRICAO | [09:04] Bruno |
| **RFC-ALT-02** | docs/RFC.md | Trade-off | Alternativa descartada: Redis Streams ou fila dedicada | TRANSCRICAO | [09:07] Larissa |
| **RFC-ALT-02b** | docs/RFC.md | Trade-off | Motivo do descarte: overengineering para time pequeno; exigiria infra nova | TRANSCRICAO | [09:07] Diego |
| **RFC-ALT-03** | docs/RFC.md | Trade-off | Alternativa descartada: trigger de banco em vez de polling | TRANSCRICAO | [09:09] Bruno |
| **RFC-ALT-03b** | docs/RFC.md | Trade-off | Motivo do descarte: MySQL não tem NOTIFY/LISTEN; trigger não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| **RFC-ALT-04** | docs/RFC.md | Trade-off | Alternativa descartada: DLQ como flag "failed" na própria outbox | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| **RFC-ALT-05** | docs/RFC.md | Trade-off | Alternativa descartada: 3 tentativas de retry em vez de 5 | TRANSCRICAO | [09:16] Bruno |
| **RFC-ALT-05b** | docs/RFC.md | Trade-off | Motivo do descarte: 3 tentativas matariam eventos de cliente com 2h de manutenção | TRANSCRICAO | [09:16] Diego |
| **RFC-ALT-06** | docs/RFC.md | Trade-off | Alternativa descartada: secret global da plataforma — "se vaza uma, vaza tudo" | TRANSCRICAO | [09:21] Sofia |
| **RFC-ALT-07** | docs/RFC.md | Trade-off | Alternativa descartada: exactly-once, por exigir coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| **RFC-Q1** | docs/RFC.md | Questão em Aberto | Rate limiting de saída: risco de 50 chamadas em um minuto para o mesmo cliente | TRANSCRICAO | [09:38] Diego |
| **RFC-Q1b** | docs/RFC.md | Questão em Aberto | Encaminhamento: "observar e decidir depois" | TRANSCRICAO | [09:39] Larissa |
| **RFC-Q2** | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers: particionar por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| **RFC-Q3** | docs/RFC.md | Questão em Aberto | Endurecimento da autorização do CRUD de configuração no futuro | TRANSCRICAO | [09:37] Sofia |
| **RFC-Q4** | docs/RFC.md | Questão em Aberto | Política de retenção/arquivamento da outbox não definida ("30 dias ou assim") | TRANSCRICAO | [09:08] Diego |
| **RFC-Q5** | docs/RFC.md | Questão em Aberto | Email de alerta reabre somente após medição de impacto | TRANSCRICAO | [09:37] Larissa |
| **RFC-IMP-01** | docs/RFC.md | Restrição | Middlewares e erros existentes não precisam de alteração | TRANSCRICAO | [09:29] Bruno |
| **RFC-IMP-02** | docs/RFC.md | Restrição | `errorMiddleware` já trata AppError, Zod e Prisma | CODIGO | `src/middlewares/error.middleware.ts` (linhas 14–54) |
| **RFC-IMP-03** | docs/RFC.md | Restrição | Nenhuma dependência nova de runtime é necessária | CODIGO | `package.json` (uuid, zod, pino, @prisma/client já presentes; Node >= 20) |
| **FDD-DADOS-01** | docs/FDD.md | Decisão | Tabela de configuração guarda url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| **FDD-DADOS-02** | docs/FDD.md | Decisão | `webhook_outbox` guarda o payload renderizado (snapshot) | TRANSCRICAO | [09:52] Larissa; [09:52] Diego |
| **FDD-DADOS-03** | docs/FDD.md | Decisão | PK da outbox é o UUID usado como `X-Event-Id` | TRANSCRICAO | [09:25] Diego; [09:51] Larissa |
| **FDD-DADOS-04** | docs/FDD.md | Decisão | `webhook_dead_letter` guarda payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| **FDD-DADOS-05** | docs/FDD.md | Requisito Funcional | `webhook_deliveries` registra sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| **FDD-DADOS-06** | docs/FDD.md | Restrição | Enum `OrderStatus` com PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED | CODIGO | `prisma/schema.prisma` (linhas 16–23) |
| **FDD-DADOS-07** | docs/FDD.md | Restrição | Modelos existentes usam `@@map` para snake_case | CODIGO | `prisma/schema.prisma` |
| **FDD-FLUXO-01** | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação de `changeStatus`, após o insert em history | CODIGO | `src/modules/orders/order.service.ts` (linhas 158–167) |
| **FDD-FLUXO-02** | docs/FDD.md | Decisão | Integração via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | [09:41] Bruno |
| **FDD-FLUXO-02b** | docs/FDD.md | Decisão | Função pura recebendo o tx, sem injetar repository inteiro | TRANSCRICAO | [09:41] Diego |
| **FDD-FLUXO-02c** | docs/FDD.md | Restrição | Type `TxClient = Prisma.TransactionClient` já declarado no service | CODIGO | `src/modules/orders/order.service.ts` (linha 24) |
| **FDD-FLUXO-03** | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em batch, processa e marca | TRANSCRICAO | [09:09] Diego; [09:08] Diego |
| **FDD-FLUXO-04** | docs/FDD.md | Fluxo | Ordenação do processamento por `created_at` da outbox | TRANSCRICAO | [09:12] Diego |
| **FDD-FLUXO-05** | docs/FDD.md | Fluxo | Retry: 5 tentativas com backoff 1m/5m/30m/2h/12h, ~15h de janela total | TRANSCRICAO | [09:17] Diego |
| **FDD-FLUXO-06** | docs/FDD.md | Fluxo | DLQ após a quinta falha, em tabela separada | TRANSCRICAO | [09:18] Diego |
| **FDD-FLUXO-07** | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| **FDD-CONTRATO-01** | docs/FDD.md | Contrato | `POST /api/v1/webhooks` — cadastro, com secret devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| **FDD-CONTRATO-02** | docs/FDD.md | Contrato | `GET /api/v1/webhooks?customerId=` — listagem por customer | TRANSCRICAO | [09:33] Bruno |
| **FDD-CONTRATO-03** | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` — edição | TRANSCRICAO | [09:33] Bruno |
| **FDD-CONTRATO-04** | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` — remoção | TRANSCRICAO | [09:33] Bruno |
| **FDD-CONTRATO-05** | docs/FDD.md | Contrato | `POST /api/v1/webhooks/:id/secret/rotate` — rotação com grace de 24h | TRANSCRICAO | [09:21] Sofia |
| **FDD-CONTRATO-06** | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| **FDD-CONTRATO-07** | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay restrito a ADMIN | TRANSCRICAO | [09:35] Diego; [09:36] Larissa |
| **FDD-CONTRATO-08** | docs/FDD.md | Contrato | Envelope de listagem `{ data, pagination }` reaproveitado do projeto | CODIGO | `src/shared/http/response.ts` (`paginated`) |
| **FDD-CONTRATO-09** | docs/FDD.md | Contrato | `DELETE` responde 204 sem corpo, seguindo o padrão existente | CODIGO | `src/modules/orders/order.controller.ts` (linha 51) |
| **FDD-CONTRATO-10** | docs/FDD.md | Contrato | Headers de saída: X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| **FDD-CONTRATO-11** | docs/FDD.md | Contrato | Header X-Webhook-Id acrescentado | TRANSCRICAO | [09:44] Sofia |
| **FDD-CONTRATO-12** | docs/FDD.md | Contrato | Body do evento sem os items, para manter o payload enxuto | TRANSCRICAO | [09:43] Diego; [09:44] Bruno |
| **FDD-ERRO-01** | docs/FDD.md | Decisão | Prefixo `WEBHOOK_` em todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| **FDD-ERRO-02** | docs/FDD.md | Decisão | Códigos nomeados na reunião: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| **FDD-ERRO-03** | docs/FDD.md | Restrição | Formato a seguir: AppError com statusCode, errorCode e details | CODIGO | `src/shared/errors/app-error.ts` |
| **FDD-ERRO-04** | docs/FDD.md | Restrição | Modelos de classe: `InvalidStatusTransitionError`, `InsufficientStockError` | CODIGO | `src/shared/errors/http-errors.ts` (linhas 45–63) |
| **FDD-ERRO-05** | docs/FDD.md | Restrição | 401/403 mantêm UNAUTHORIZED/FORBIDDEN por reuso do middleware | CODIGO | `src/middlewares/auth.middleware.ts` (linhas 49–61) |
| **FDD-ERRO-06** | docs/FDD.md | Decisão | `WEBHOOK_PAYLOAD_TOO_LARGE` para payload acima de 64KB | TRANSCRICAO | [09:24] Larissa |
| **FDD-ERRO-07** | docs/FDD.md | Decisão | `WEBHOOK_DELIVERY_TIMEOUT` para resposta acima de 10s | TRANSCRICAO | [09:42] Diego |
| **FDD-RESIL-01** | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| **FDD-RESIL-02** | docs/FDD.md | Requisito Não Funcional | Intervalo de polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| **FDD-RESIL-03** | docs/FDD.md | Restrição | Sem fallback por outro canal: email adiado para fase futura | TRANSCRICAO | [09:37] Larissa |
| **FDD-OBS-01** | docs/FDD.md | Decisão | Logging com Pino, sem nenhuma dependência nova | TRANSCRICAO | [09:29] Bruno |
| **FDD-OBS-02** | docs/FDD.md | Restrição | Padrão de log `{ contexto }, 'evento_snake_case'` já usado no projeto | CODIGO | `src/server.ts` (`'server_started'`); `src/middlewares/request-logger.middleware.ts` (`'http_request'`) |
| **FDD-OBS-03** | docs/FDD.md | Restrição | Correlação por request id existente, sem stack de tracing distribuído | CODIGO | `src/middlewares/request-logger.middleware.ts` (linhas 5–9) |
| **FDD-OBS-04** | docs/FDD.md | Risco | `redactPaths` não cobre `*.secret`; precisa ser estendido | CODIGO | `src/shared/logger/index.ts` (linhas 4–11) |
| **FDD-OBS-05** | docs/FDD.md | Requisito Não Funcional | Log de auditoria do replay com o usuário executor | TRANSCRICAO | [09:36] Sofia |
| **FDD-DEP-01** | docs/FDD.md | Dependência | Script `npm run worker` no package.json | TRANSCRICAO | [09:11] Larissa |
| **FDD-DEP-02** | docs/FDD.md | Restrição | Node >= 20 permite `fetch` e `AbortSignal.timeout` nativos | CODIGO | `package.json` (`engines.node`) |
| **FDD-DEP-03** | docs/FDD.md | Restrição | Novas variáveis de ambiente entram no schema Zod existente | CODIGO | `src/config/env.ts` |
| **FDD-DEP-04** | docs/FDD.md | Restrição | MySQL 8.0 do compose existente, sem instância nova | CODIGO | `docker-compose.yml` |
| **FDD-INT-01** | docs/FDD.md | Integração | Alteração crítica no método `changeStatus` do service de orders | TRANSCRICAO | [09:40] Bruno |
| **FDD-INT-02** | docs/FDD.md | Integração | `OrderService.changeStatus` e sua transação | CODIGO | `src/modules/orders/order.service.ts` (linhas 126–179) |
| **FDD-INT-03** | docs/FDD.md | Integração | Hierarquia de erros a estender | CODIGO | `src/shared/errors/http-errors.ts` |
| **FDD-INT-04** | docs/FDD.md | Integração | Middleware de erro reusado sem alteração | CODIGO | `src/middlewares/error.middleware.ts` (linha 15) |
| **FDD-INT-05** | docs/FDD.md | Integração | `authenticate` e `requireRole` reusados; padrão de uso já existente | CODIGO | `src/middlewares/auth.middleware.ts`; `src/modules/users/user.routes.ts` (linhas 12–18) |
| **FDD-INT-06** | docs/FDD.md | Integração | Modelos novos em schema Prisma, migration aditiva | CODIGO | `prisma/schema.prisma`; `prisma/migrations/20260519182739_init/` |
| **FDD-INT-07** | docs/FDD.md | Integração | `createPrismaClient()` usado pelo worker para client próprio | CODIGO | `src/config/database.ts` |
| **FDD-INT-08** | docs/FDD.md | Integração | Registro do módulo em `Controllers` e `buildApiRouter` | CODIGO | `src/routes/index.ts` (linhas 13–30); `src/app.ts` (linhas 26–53) |
| **FDD-INT-09** | docs/FDD.md | Integração | Validação https via Zod no middleware existente | CODIGO | `src/middlewares/validate.middleware.ts` |
| **FDD-INT-09b** | docs/FDD.md | Restrição | Exigência de https como validação de schema, não decisão arquitetural | TRANSCRICAO | [09:23] Sofia |
| **FDD-INT-10** | docs/FDD.md | Integração | Enum `OrderStatus` consumido via `z.nativeEnum`, como já se faz em orders | CODIGO | `src/modules/orders/order.schemas.ts` (linha 19); `src/modules/orders/order.status.ts` |
| **FDD-INT-11** | docs/FDD.md | Integração | Estrutura de módulo `src/modules/webhooks` com controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| **FDD-INT-12** | docs/FDD.md | Integração | Processador em `src/modules/webhooks/webhook.processor.ts` | TRANSCRICAO | [09:28] Bruno |
| **FDD-INT-13** | docs/FDD.md | Integração | Testes existentes de pedidos impactados pelo novo efeito colateral | CODIGO | `tests/orders.test.ts`; `tests/helpers/factories.ts` |
| **ADR-001** | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Padrão Outbox transacional no MySQL existente | TRANSCRICAO | [09:06] Diego; [09:08] Larissa |
| **ADR-002** | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado, polling de 2 segundos | TRANSCRICAO | [09:09] Diego; [09:10] Larissa; [09:11] Diego |
| **ADR-003** | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h, 5 tentativas, DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| **ADR-004** | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | [09:22] Sofia |
| **ADR-004b** | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Algoritmo SHA-256 como padrão de mercado | TRANSCRICAO | [09:20] Sofia |
| **ADR-005** | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com dedup pelo `X-Event-Id` do lado do cliente | TRANSCRICAO | [09:26] Larissa |
| **ADR-005b** | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Justificativa por padrão de mercado: Stripe e GitHub fazem assim | TRANSCRICAO | [09:25] Diego |
| **ADR-006** | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes; webhook como módulo igual aos outros | TRANSCRICAO | [09:30] Larissa |
| **ADR-006b** | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | Inventário verificado dos padrões do projeto (módulos, erros, Zod, Pino, auth, DI, paginação) | CODIGO | `src/modules/*/`, `src/shared/`, `src/middlewares/`, `src/app.ts` |
| **ADR-007** | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Snapshot do payload renderizado na inserção | TRANSCRICAO | [09:52] Larissa; [09:52] Diego; [09:52] Bruno |
| **ADR-007b** | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Contexto | Pergunta original: payload renderizado ou só order_id? | TRANSCRICAO | [09:51] Bruno |
| **ADR-007c** | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Restrição | Precedente do projeto: `order_status_history` materializa fromStatus e toStatus | CODIGO | `prisma/schema.prisma` (model `OrderStatusHistory`) |
| **ADR-008** | docs/adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md | Decisão | Filtro de status assinados aplicado na inserção, não no envio | TRANSCRICAO | [09:34] Bruno; [09:34] Diego |
| **ADR-008b** | docs/adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md | Contexto | Pergunta original: filtra na inserção do outbox ou na hora de mandar? | TRANSCRICAO | [09:34] Diego |
| **ADR-008c** | docs/adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md | Restrição | Transação atual de estoque na transição PENDING → PAID | CODIGO | `src/modules/orders/order.service.ts` (`debitStock`, linhas 204–231) |

---

## 3. Itens deliberadamente não rastreados

Três classes de conteúdo aparecem nos documentos sem linha própria acima, por não serem itens de requisito, decisão ou restrição:

1. **Valores derivados de regra, e não de fala.** Por exemplo, o lease de 20s para evento em `PROCESSING` ([FDD §8.1](./FDD.md#81-timeouts)) e o default de `WEBHOOK_BATCH_SIZE`. A transcrição fixou "batch pequeno" ([09:08] Diego) sem número. Esses casos estão marcados como derivados no próprio FDD.

2. **Consequências analíticas dos ADRs.** As seções "Consequências" desenvolvem implicações das decisões já rastreadas; a decisão está no tracker, o desdobramento é análise.

3. **Exemplos ilustrativos.** UUIDs, nomes de host e valores monetários nos payloads de exemplo da [FDD §6](./FDD.md#6-contratos-públicos) são fictícios por natureza. Os **campos** são rastreados (FDD-CONTRATO-10 a 12); os valores, não.

---

## 4. O que o tracker impediu de entrar

Registro de itens que apareceram em rascunhos e foram removidos por não terem origem verificável — a função prática deste documento:

| Item descartado | Por que saiu |
| --- | --- |
| Meta de "99,9% de entregas com sucesso" | Nenhum SLA percentual foi discutido na reunião. Substituído por metas com número de fala real (OBJ-1 a OBJ-6). |
| Uso de `pino-http` no worker | A dependência existe no `package.json`, mas é para requisições HTTP de entrada; o worker não atende HTTP. |
| `X-Signature-Timestamp` como header | A transcrição diz `X-Timestamp` ([09:44] Diego). Nome corrigido para o literal da ata. |
| Endpoint `GET /webhooks/:id` (detalhe individual) | Bruno listou apenas POST, PATCH, DELETE e GET de listagem ([09:33] Bruno). Não foi pedido. |
| Retry com jitter | Não mencionado. Registrado no FDD como ausência consciente, vinculada a Q1, em vez de virar requisito. |
| Alerta automático de DLQ por email | Explicitamente adiado ([09:37] Larissa). Manter seria contrariar a ata. |
| Paginação configurável no histórico de entregas | Marcos falou em "últimos 100" ([09:34] Marcos); o `pageSize` padrão reflete isso sem inventar limites. |
