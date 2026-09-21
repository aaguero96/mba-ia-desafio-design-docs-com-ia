# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead), com Bruno (Pedidos) e Diego (Plataforma) |
| **Status** | Pronto para implementação |
| **Data** | 2026-09-21 |
| **Documentos relacionados** | [PRD](./PRD.md) · [RFC](./RFC.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |
| **Fonte** | [`TRANSCRICAO.md`](../TRANSCRICAO.md) e o código do repositório |

> Este documento responde **como construir**. O porquê de cada decisão está nos [ADRs](./adrs/); a proposta em nível de arquitetura está no [RFC](./RFC.md). Tudo o que está aqui é rastreável à transcrição ou ao código — ver [TRACKER.md](./TRACKER.md).

---

## 1. Contexto e motivação técnica

O OMS controla o ciclo de vida do pedido com máquina de estados (`src/modules/orders/order.status.ts`), transação de estoque e auditoria de mudanças em `order_status_history`. O que ele não tem é qualquer saída de evento: nenhuma fila, nenhum webhook, nenhuma notificação externa.

Três clientes B2B precisam ser avisados quando o status de um pedido muda, e hoje descobrem isso por polling em `GET /orders` ([09:00] Marcos). A latência aceitável é abaixo de 10 segundos ([09:02] Marcos).

O desafio técnico central não é enviar HTTP — é **garantir que todo commit de mudança de status produza exatamente um evento por assinante, sem acoplar a disponibilidade do OMS à disponibilidade de terceiros**. Bruno colocou a exigência em termos absolutos: não pode existir o caso de o status mudar e o evento não sair ([09:40] Bruno).

A resposta é o padrão Outbox: a linha do evento é inserida na mesma transação SQL que muda o status ([09:06] Diego), e um processo separado faz a entrega ([09:11] Diego).

---

## 2. Objetivos técnicos

| # | Objetivo | Verificação |
| --- | --- | --- |
| OT-1 | Toda transição de status commitada produz uma linha em `webhook_outbox` por endpoint ativo que assina aquele status; toda transação revertida não produz nenhuma. | Teste de integração com rollback forçado. |
| OT-2 | Latência entre o commit e a primeira tentativa de entrega ≤ 2 segundos no pior caso. | Métrica `webhook_pickup_latency_seconds`. |
| OT-3 | Nenhuma tentativa de entrega bloqueia a API: worker roda em processo separado, com `PrismaClient` próprio. | `src/worker.ts` não importa `src/app.ts`. |
| OT-4 | Toda entrega carrega assinatura HMAC-SHA256 verificável e `X-Event-Id` estável entre retentativas. | Teste que compara o header das 5 tentativas do mesmo evento. |
| OT-5 | Falha de entrega nunca resulta em perda silenciosa: o evento termina em `DELIVERED` ou em `webhook_dead_letter`. | Nenhuma linha pode ficar em `PROCESSING` após o timeout de lease. |
| OT-6 | Zero alterações em `src/middlewares/error.middleware.ts`; erros do módulo são serializados pelo caminho existente. | Diff do arquivo vazio ao fim da implementação. |
| OT-7 | Zero dependências novas de runtime no `package.json`. | Diff da seção `dependencies` vazio. |

---

## 3. Escopo e exclusões

### 3.1 Incluído

- Modelagem das tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
- Módulo `src/modules/webhooks/` com controller, service, repository, routes e schemas.
- Entry-point `src/worker.ts` e processador `src/modules/webhooks/webhook.processor.ts`.
- Integração transacional em `OrderService.changeStatus`.
- Assinatura HMAC-SHA256, geração e rotação de secret com grace period.
- Retry com backoff, DLQ e endpoint de replay restrito a `ADMIN`.
- Endpoint de histórico de entregas.

### 3.2 Excluído — com origem na reunião

| Exclusão | Origem |
| --- | --- |
| Webhooks inbound (cliente → OMS) | [09:02] Marcos — "só saindo da gente pra eles" |
| Email de alerta ao cliente após falhas consecutivas | [09:37] Larissa — "email tá fora de escopo dessa fase" |
| Dashboard/painel visual para o cliente | [09:40] Larissa — "só endpoints. Painel é projeto separado do time de frontend" |
| Arquivamento das linhas entregues (~30 dias) | [09:08] Diego — "fora do escopo dessa feature" |
| Rate limiting de saída por cliente | [09:39] Diego / [09:39] Larissa — "observar e decidir depois" |
| Múltiplos workers em paralelo / ordering global | [09:13] Diego — "problema do futuro"; [09:13] Larissa — limitação conhecida |
| Exactly-once | [09:25] Diego — descartado em favor de at-least-once |

### 3.3 Limitações conhecidas assumidas

- **Ordenação garantida apenas por `order_id`**, e apenas enquanto houver um único worker ([09:12] Diego; [09:13] Larissa). Não há garantia de ordem global entre pedidos diferentes.
- **Assinaturas não são retroativas**: um status alterado antes de o cliente assiná-lo não gera evento recuperável ([ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md)).
- **CRUD de configuração aceita qualquer role autenticada** nesta fase ([09:37] Sofia).

---

## 4. Modelo de dados

Quatro modelos novos em `prisma/schema.prisma`. Todos com PK `String @id @default(uuid()) @db.Char(36)`, seguindo o padrão de todos os modelos existentes e a decisão explícita de Larissa ([09:51] Larissa).

### 4.1 `webhook_endpoints`

Configuração do endpoint do cliente. Campos definidos por Bruno e confirmados por Sofia: url + secret + `customer_id` + estado ativo ([09:21] Bruno; [09:21] Sofia), mais a lista de status assinados ([09:33] Marcos).

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | `Char(36)` | PK, UUID |
| `customerId` | `Char(36)` | FK → `customers.id` |
| `url` | `VarChar(2048)` | Obrigatoriamente `https` ([09:23] Sofia) |
| `secret` | `VarChar(255)` | Secret ativa, única por endpoint ([09:21] Sofia) |
| `previousSecret` | `VarChar(255)?` | Secret anterior durante o grace period |
| `previousSecretExpiresAt` | `DateTime?` | `rotatedAt + 24h` ([09:21] Sofia) |
| `subscribedStatuses` | `Json` | Lista de `OrderStatus` assinados ([09:33] Marcos) |
| `active` | `Boolean @default(true)` | Estado ativo ([09:21] Bruno) |
| `createdAt` / `updatedAt` | `DateTime` | Padrão do projeto |

Índices: `@@index([customerId])`, `@@index([active])` — a query de inserção filtra por ambos.

### 4.2 `webhook_outbox`

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | `Char(36)` | PK, UUID. **É o valor do header `X-Event-Id`** ([09:25] Diego; [09:51] Larissa) |
| `webhookEndpointId` | `Char(36)` | FK → `webhook_endpoints.id` |
| `orderId` | `Char(36)` | FK → `orders.id` |
| `eventType` | `VarChar(64)` | `order.status_changed` ([09:43] Diego) |
| `payload` | `Json` | **Snapshot renderizado na inserção** ([09:52] Larissa) — ver [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| `status` | `Enum` | `PENDING` · `PROCESSING` · `FAILED` · `DELIVERED` ([09:08] Diego) |
| `attempts` | `Int @default(0)` | 0 a 5 ([09:15] Diego) |
| `nextAttemptAt` | `DateTime` | Igual a `createdAt` na primeira vez; depois, backoff |
| `lockedAt` | `DateTime?` | Lease do worker, para detectar `PROCESSING` órfão |
| `requestId` | `VarChar(64)?` | Correlação com o request que originou (ver §8.3) |
| `createdAt` | `DateTime @default(now())` | Base da ordenação ([09:12] Diego) |

Índices, conforme Diego ([09:08] Diego): `@@index([status, nextAttemptAt])` para a query quente do worker e `@@index([createdAt])` para a ordenação. Mais `@@index([orderId])`.

### 4.3 `webhook_deliveries`

Histórico de tentativas, para o endpoint de deliveries pedido por Marcos: sucesso/falha, payload, response e tempo de resposta ([09:34] Marcos).

| Campo | Tipo |
| --- | --- |
| `id`, `outboxEventId`, `webhookEndpointId` | `Char(36)` |
| `attemptNumber` | `Int` |
| `success` | `Boolean` |
| `responseStatus` | `Int?` |
| `responseBody` | `Text?` (truncado) |
| `durationMs` | `Int` |
| `errorCode` | `VarChar(64)?` — um dos códigos `WEBHOOK_*` da §6 |
| `attemptedAt` | `DateTime @default(now())` |

Índice: `@@index([webhookEndpointId, attemptedAt])`.

### 4.4 `webhook_dead_letter`

Tabela separada, com payload, motivo da falha e timestamp ([09:18] Diego).

| Campo | Tipo |
| --- | --- |
| `id` | `Char(36)` |
| `originalEventId` | `Char(36)` — **preservado para manter o `X-Event-Id` estável no replay** |
| `webhookEndpointId`, `orderId` | `Char(36)` |
| `payload` | `Json` |
| `failureReason` | `VarChar(500)` |
| `lastErrorCode` | `VarChar(64)` |
| `totalAttempts` | `Int` |
| `replayedAt` | `DateTime?` |
| `replayedById` | `Char(36)?` — FK → `users.id`, para a auditoria exigida por Sofia ([09:36] Sofia) |
| `createdAt` | `DateTime @default(now())` |

---

## 5. Fluxos detalhados

### 5.1 Fluxo 1 — Criação do evento na outbox

Acontece **dentro** da transação de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`, linhas 131–178).

```
PATCH /api/v1/orders/:id/status
   │
   ▼
OrderController.changeStatus                     (order.controller.ts:38)
   │
   ▼
OrderService.changeStatus → this.prisma.$transaction(async (tx) => {
   │
   ├─ 1. tx.order.findUnique                     (já existe, linha 132)
   ├─ 2. valida from === to                      (já existe, linha 140)
   ├─ 3. canTransition(from, to)                 (já existe, linha 147)
   ├─ 4. debitStock / replenishStock             (já existe, linhas 151-156)
   ├─ 5. tx.order.update({ status: to })         (já existe, linha 158)
   ├─ 6. tx.orderStatusHistory.create            (já existe, linha 159)
   │
   ├─ 7. ◄── NOVO ──►
   │      publishWebhookEvent(tx, order, from, to)
   │        a. lê webhook_endpoints do customerId
   │           WHERE active = true
   │             AND JSON_CONTAINS(subscribedStatuses, to)
   │        b. se nenhum assinante → retorna, nada é inserido   ([09:34] Bruno)
   │        c. renderiza o payload uma vez (snapshot)           ([09:52] Larissa)
   │        d. valida tamanho ≤ 64KB → senão WEBHOOK_PAYLOAD_TOO_LARGE
   │        e. para cada assinante: tx.webhookOutbox.create({
   │             id: uuid(), status: PENDING, attempts: 0,
   │             nextAttemptAt: now(), payload, requestId
   │           })
   │
   └─ 8. tx.order.findUnique refreshed           (já existe, linha 169)
})
   │
   ├─ COMMIT  → status mudou E eventos existem
   └─ ROLLBACK → nada mudou E nenhum evento existe
```

**Regras do fluxo:**

- A função é pura e recebe o `tx` da transação corrente — `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Não injeta repository no `OrderService` ([09:41] Bruno; [09:41] Diego).
- Se a inserção falhar, a transação inteira dá rollback. É intencional: não pode haver status mudado sem evento ([09:40] Bruno; [09:41] Diego).
- A validação de 64KB acontece **antes** da inserção, para que um payload gigante falhe cedo e não fique preso na outbox ([09:24] Larissa).
- O filtro é por status **de destino** (`to_status`), que é o que o cliente declara querer ouvir ([09:33] Marcos).

### 5.2 Fluxo 2 — Processamento pelo worker

Loop em `src/modules/webhooks/webhook.processor.ts`, disparado por `src/worker.ts`.

```
a cada 2 segundos:                                          ([09:09] Diego)
   │
   ├─ 1. SELECT ... FROM webhook_outbox
   │       WHERE status = 'PENDING' AND nextAttemptAt <= NOW()
   │       ORDER BY createdAt ASC                           ([09:12] Diego)
   │       LIMIT <batch pequeno>                            ([09:08] Diego)
   │
   ├─ 2. marca o batch como PROCESSING, lockedAt = NOW()
   │
   └─ para cada evento, sequencialmente:
        │
        ├─ 3. carrega o endpoint; se inactive → WEBHOOK_ENDPOINT_INACTIVE, DLQ direto
        │
        ├─ 4. body = JSON.stringify(payload)   ← serializa UMA vez
        │      signature = HMAC-SHA256(body, endpoint.secret)
        │      headers = {
        │        'Content-Type': 'application/json',
        │        'X-Event-Id':   event.id,        ([09:25] Diego)
        │        'X-Signature':  signature,       ([09:20] Sofia)
        │        'X-Timestamp':  ISO-8601 do envio, ([09:44] Diego)
        │        'X-Webhook-Id': endpoint.id      ([09:44] Sofia)
        │      }
        │
        ├─ 5. POST endpoint.url, timeout 10s               ([09:42] Diego)
        │
        ├─ 6. grava linha em webhook_deliveries            ([09:34] Marcos)
        │
        └─ 7. resultado:
              ├─ 2xx        → status = DELIVERED
              ├─ 4xx/5xx    → §5.3 (retry)
              ├─ timeout    → §5.3, errorCode WEBHOOK_DELIVERY_TIMEOUT
              └─ DNS/TLS/conn → §5.3, errorCode WEBHOOK_CONNECTION_FAILED
```

**Pontos críticos de implementação:**

- O corpo é serializado **uma única vez** e o HMAC é calculado sobre **exatamente os mesmos bytes** que vão no `POST`. Qualquer reserialização entre assinar e enviar quebra a verificação do cliente — é a fonte clássica de bug nesse tipo de integração.
- O processamento é sequencial dentro do batch, o que preserva a ordenação por `created_at` ([09:12] Diego).
- Eventos presos em `PROCESSING` com `lockedAt` mais antigo que um limite (sugestão: 2 × timeout, isto é, 20s) voltam para `PENDING` no início do ciclo. Sem isso, um crash do worker no meio do batch deixa eventos órfãos para sempre — e é exatamente o cenário que o single-worker torna plausível (R3 do [RFC](./RFC.md#62-riscos)).

### 5.3 Fluxo 3 — Retry com backoff

```
falha na tentativa N:
   │
   ├─ attempts = N
   ├─ grava webhook_deliveries { success: false, errorCode, durationMs }
   │
   ├─ se N < 5:                                            ([09:15] Diego)
   │     status        = PENDING
   │     nextAttemptAt = NOW() + BACKOFF[N]
   │     lockedAt      = null
   │
   └─ se N = 5:  → §5.4 (DLQ)

BACKOFF = [ 1min, 5min, 30min, 2h, 12h ]                   ([09:17] Diego)
```

| Tentativa | Espera antes dela | Tempo acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 | — | 0 |
| 2 | 1 min | 1 min |
| 3 | 5 min | 6 min |
| 4 | 30 min | 36 min |
| 5 | 2 h | 2h 36min |
| (fim) | 12 h | **14h 36min** |

Diego descreveu o total como "quase 15 horas" ([09:17] Diego) e Marcos aceitou a janela ([09:17] Marcos).

**Tratamento por classe de resposta:**

| Resposta do cliente | Ação |
| --- | --- |
| `2xx` | Sucesso. `DELIVERED`. |
| `408`, `429`, `5xx` | Retry. Falha transitória. |
| `4xx` exceto `408`/`429` | Retry mesmo assim. Não temos como distinguir bug momentâneo do cliente de rejeição definitiva, e a decisão do time foi nunca perder evento silenciosamente ([ADR-005](./adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). O `errorCode` registrado é `WEBHOOK_DELIVERY_REJECTED`. |
| Timeout de 10s | Retry. `WEBHOOK_DELIVERY_TIMEOUT` ([09:42] Diego). |
| Falha de conexão/DNS/TLS | Retry. `WEBHOOK_CONNECTION_FAILED`. |

### 5.4 Fluxo 4 — Dead Letter Queue e replay

```
5ª falha:
   │
   ├─ INSERT webhook_dead_letter {
   │     originalEventId: event.id,   ← preserva o X-Event-Id
   │     payload, failureReason, lastErrorCode,
   │     totalAttempts: 5
   │   }
   ├─ DELETE (ou status FAILED) da linha em webhook_outbox   ([09:18] Diego)
   └─ log.error { event: 'webhook_dead_lettered', ... }

replay manual:
   POST /api/v1/admin/webhooks/dead-letter/:id/replay        ([09:35] Diego)
   │
   ├─ authenticate + requireRole('ADMIN')                    ([09:36] Sofia/Larissa)
   ├─ log.warn { event: 'webhook_dlq_replayed',
   │             replayedById: req.user.id }  ← auditoria    ([09:36] Sofia)
   │
   ├─ INSERT webhook_outbox {
   │     id: deadLetter.originalEventId,  ← MESMO id, não gera novo
   │     status: PENDING, attempts: 0, nextAttemptAt: NOW()
   │   }
   └─ UPDATE webhook_dead_letter SET replayedAt, replayedById
```

> **Atenção na implementação.** Reutilizar `originalEventId` como `id` da nova linha é obrigatório. Gerar um UUID novo faria o cliente receber o mesmo fato com dois `X-Event-Id` diferentes, e a deduplicação prometida em [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) falharia justamente no cenário em que ela mais importa.

---

## 6. Contratos públicos

Todos os endpoints ficam sob o prefixo `/api/v1`, montado em `src/app.ts` (linha 67). A transcrição cita os caminhos sem o prefixo — `GET /webhooks/:id/deliveries` ([09:34] Marcos) e `POST /admin/webhooks/dead-letter/:id/replay` ([09:35] Diego) —; o prefixo é uma imposição do código existente.

Todos exigem `Authorization: Bearer <jwt>` via `authenticate` (`src/middlewares/auth.middleware.ts`). Apenas o replay de DLQ exige `ADMIN` ([09:36] Sofia); o restante aceita qualquer role autenticada nesta fase ([09:37] Sofia).

### 6.1 `POST /api/v1/webhooks` — Cadastrar endpoint

O `customerId` vem **no body**, não do JWT: Bruno apontou que o JWT atual é do usuário operador e Larissa fechou que o customer é passado no body ou no path ([09:32] Bruno; [09:32] Larissa).

**Request**

```json
{
  "customerId": "6f1c2e9a-3b7d-4c55-9f10-2a8e4d6b1c03",
  "url": "https://webhooks.atlascomercial.com.br/oms/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`** — a secret é gerada por nós e devolvida **apenas aqui** ([09:31] Marcos):

```json
{
  "id": "b2d9f4a1-77c8-41e6-95aa-0d3f8c1b6e24",
  "customerId": "6f1c2e9a-3b7d-4c55-9f10-2a8e4d6b1c03",
  "url": "https://webhooks.atlascomercial.com.br/oms/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f2c7a1e5b834d06a8f1c3e7d92b45a0",
  "createdAt": "2026-09-21T13:02:44.118Z"
}
```

| Status | Quando |
| --- | --- |
| `201` | Criado |
| `400` | `WEBHOOK_INVALID_URL` (não-`https` ou malformada), `WEBHOOK_INVALID_STATUS_FILTER`, `VALIDATION_ERROR` |
| `401` | Sem token válido |
| `404` | `NOT_FOUND` — customer inexistente (reusa `NotFoundError`) |

### 6.2 `GET /api/v1/webhooks?customerId=...` — Listar endpoints do customer

Pedido por Bruno ([09:33] Bruno). Usa o helper `paginated()` de `src/shared/http/response.ts`, no mesmo formato de `OrderService.list`.

**Response `200 OK`** — a `secret` **nunca** é devolvida em listagem:

```json
{
  "data": [
    {
      "id": "b2d9f4a1-77c8-41e6-95aa-0d3f8c1b6e24",
      "customerId": "6f1c2e9a-3b7d-4c55-9f10-2a8e4d6b1c03",
      "url": "https://webhooks.atlascomercial.com.br/oms/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotatedAt": null,
      "createdAt": "2026-09-21T13:02:44.118Z",
      "updatedAt": "2026-09-21T13:02:44.118Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
| --- | --- |
| `200` | Sucesso (lista vazia também é `200`) |
| `400` | `VALIDATION_ERROR` — `customerId` ausente ou não-UUID |
| `401` | Sem token válido |

### 6.3 `PATCH /api/v1/webhooks/:id` — Editar endpoint

Pedido por Bruno ([09:33] Bruno). Campos parciais, seguindo o padrão `createSchema.partial()` já usado em `src/modules/customers/customer.schemas.ts`.

**Request**

```json
{
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response `200 OK`**

```json
{
  "id": "b2d9f4a1-77c8-41e6-95aa-0d3f8c1b6e24",
  "customerId": "6f1c2e9a-3b7d-4c55-9f10-2a8e4d6b1c03",
  "url": "https://webhooks.atlascomercial.com.br/oms/orders",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "updatedAt": "2026-09-21T15:40:09.771Z"
}
```

| Status | Quando |
| --- | --- |
| `200` | Atualizado |
| `400` | `WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_STATUS_FILTER` |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |

> **Semântica:** alterar `subscribedStatuses` **não é retroativo**. Eventos já capturados na outbox continuam com o destino definido no momento da inserção ([ADR-008](./adrs/ADR-008-filtro-de-eventos-aplicado-na-insercao.md)); eventos não capturados não são recuperáveis.

### 6.4 `DELETE /api/v1/webhooks/:id` — Remover endpoint

Pedido por Bruno ([09:33] Bruno).

**Response `204 No Content`**, sem corpo — mesmo padrão de `OrderController.delete` (`src/modules/orders/order.controller.ts`, linha 51).

| Status | Quando |
| --- | --- |
| `204` | Removido |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |
| `409` | `WEBHOOK_HAS_PENDING_EVENTS` — há eventos pendentes para este endpoint |

### 6.5 `POST /api/v1/webhooks/:id/secret/rotate` — Rotacionar secret

Sofia: a secret tem que ser rotacionável, com endpoint para o cliente pedir nova; a antiga fica válida por 24h em paralelo e depois morre ([09:21] Sofia).

**Request:** sem corpo.

**Response `200 OK`**

```json
{
  "id": "b2d9f4a1-77c8-41e6-95aa-0d3f8c1b6e24",
  "secret": "whsec_4e81b27d0c9a4f63b5d8e1a70f2c93b6",
  "rotatedAt": "2026-09-21T16:11:02.004Z",
  "previousSecretExpiresAt": "2026-09-22T16:11:02.004Z"
}
```

| Status | Quando |
| --- | --- |
| `200` | Rotacionada |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |
| `409` | `WEBHOOK_ROTATION_IN_GRACE_PERIOD` — já há rotação em grace period ativo |

> **Semântica do grace period:** durante as 24h, o worker assina com a secret **nova**. A antiga permanece armazenada apenas para que o cliente que ainda não migrou consiga validar entregas que tenham sido assinadas antes da rotação — inclusive retentativas de eventos antigos. Passado o prazo, `previousSecret` é apagada.

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — Histórico de entregas

Marcos: "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" ([09:34] Marcos).

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "1a7c4e90-2f63-4b88-b0d5-9e3a7c214f68",
      "outboxEventId": "e47ac10b-58cc-4372-a567-0e02b2c3d479",
      "attemptNumber": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 312,
      "errorCode": null,
      "attemptedAt": "2026-09-21T16:20:11.900Z",
      "payload": {
        "event_id": "e47ac10b-58cc-4372-a567-0e02b2c3d479",
        "event_type": "order.status_changed",
        "timestamp": "2026-09-21T16:19:10.412Z",
        "order_id": "3c9e1f77-4a2b-4d10-8e55-71b6c0d4a9f2",
        "order_number": "ORD-000482",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "6f1c2e9a-3b7d-4c55-9f10-2a8e4d6b1c03",
        "total_cents": 149900
      }
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
| --- | --- |
| `200` | Sucesso |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — Replay de DLQ

Caminho proposto por Diego ([09:18] Diego; [09:35] Diego). **Exige role `ADMIN`** e registra quem fez, para auditoria ([09:36] Sofia; [09:36] Larissa). Usa o `requireRole` existente, da mesma forma que `src/modules/users/user.routes.ts` já faz.

**Request:** sem corpo.

**Response `202 Accepted`** — o replay reenfileira, não entrega:

```json
{
  "deadLetterId": "9b3d7f21-6c58-4a09-83e4-1f0b5d2a7c66",
  "eventId": "e47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "REQUEUED",
  "replayedAt": "2026-09-21T17:04:33.512Z",
  "replayedById": "d41a8f07-2b9c-4e63-a1f5-806c3d9e2b47"
}
```

| Status | Quando |
| --- | --- |
| `202` | Reenfileirado |
| `401` | `UNAUTHORIZED` — sem token |
| `403` | `FORBIDDEN` — role diferente de `ADMIN`. **Código herdado de `ForbiddenError`, não `WEBHOOK_*`**, porque o `requireRole` existente é reusado sem alteração ([09:36] Larissa) |
| `404` | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| `409` | `WEBHOOK_ALREADY_REPLAYED` — `replayedAt` já preenchido |

### 6.8 Contrato de saída — o que o cliente recebe

**Headers** ([09:44] Diego; [09:44] Sofia):

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento, estável entre retentativas ([09:25] Diego) |
| `X-Signature` | `sha256=<hex>` — HMAC-SHA256 do corpo, com a secret do endpoint ([09:20] Sofia) |
| `X-Timestamp` | ISO 8601 do envio, para o cliente detectar replay attack se quiser ([09:44] Diego) |
| `X-Webhook-Id` | Id do cadastro, para clientes com vários endpoints ([09:44] Sofia) |

**Body** — formato definido por Diego ([09:43] Diego). Sem `items`, deliberadamente, para não inflar o payload; quem quiser detalhes chama `GET /orders/:id`:

```json
{
  "event_id": "e47ac10b-58cc-4372-a567-0e02b2c3d479",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-21T16:19:10.412Z",
  "order_id": "3c9e1f77-4a2b-4d10-8e55-71b6c0d4a9f2",
  "order_number": "ORD-000482",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2e9a-3b7d-4c55-9f10-2a8e4d6b1c03",
  "total_cents": 149900
}
```

**Resposta esperada do cliente:** qualquer `2xx`, em até 10 segundos ([09:42] Diego). Corpo irrelevante — é registrado em `webhook_deliveries` truncado, para debug.

---

## 7. Matriz de erros

Todos os códigos do módulo usam o prefixo `WEBHOOK_`, conforme fechado por Larissa ([09:29] Larissa). Os três primeiros foram nomeados literalmente por Bruno ([09:28] Bruno); os demais seguem a mesma regra de nomenclatura.

As classes estendem `AppError` (`src/shared/errors/app-error.ts`), no mesmo formato de `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), e são serializadas pelo `errorMiddleware` existente **sem nenhuma alteração** ([09:29] Bruno).

### 7.1 Erros de API

| Código | HTTP | Classe sugerida | Quando ocorre |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends NotFoundError` | Endpoint de webhook inexistente ([09:28] Bruno) |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError extends BadRequestError` | URL malformada ou com esquema diferente de `https` ([09:28] Bruno; [09:23] Sofia) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `WebhookSecretRequiredError extends BadRequestError` | Operação que exige secret sem secret disponível ([09:28] Bruno) |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `WebhookInvalidStatusFilterError extends BadRequestError` | `subscribedStatuses` vazio ou com valor fora do enum `OrderStatus` ([09:33] Marcos) |
| `WEBHOOK_ROTATION_IN_GRACE_PERIOD` | 409 | `extends ConflictError` | Nova rotação pedida com grace period de 24h ainda ativo ([09:21] Sofia) |
| `WEBHOOK_HAS_PENDING_EVENTS` | 409 | `extends ConflictError` | `DELETE` em endpoint com eventos pendentes na outbox |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `extends NotFoundError` | Replay de DLQ inexistente ([09:35] Diego) |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `extends ConflictError` | Replay de item já reprocessado ([09:18] Diego) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `extends UnprocessableEntityError` | Payload > 64KB. **Erra, não trunca** ([09:23] Sofia; [09:24] Diego; [09:24] Larissa) |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | `extends ConflictError` | Operação de entrega sobre endpoint com `active = false` ([09:21] Bruno) |

**Formato da resposta** — produzido pelo `errorMiddleware` existente (`src/middlewares/error.middleware.ts`, linhas 15–23):

```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must use https",
    "details": { "url": "http://cliente.com/hook" }
  }
}
```

> **Nota deliberada:** `401` e `403` no módulo de webhooks retornam `UNAUTHORIZED` e `FORBIDDEN`, **sem** prefixo `WEBHOOK_`. Isso é consequência direta de reusar `authenticate` e `requireRole` sem alteração ([09:36] Larissa) — o prefixo vale para os erros do domínio do módulo, não para os do middleware compartilhado.

### 7.2 Erros internos do worker

Não viram resposta HTTP; são gravados em `webhook_deliveries.errorCode` e emitidos em log estruturado.

| Código | Quando ocorre | Ação |
| --- | --- | --- |
| `WEBHOOK_DELIVERY_TIMEOUT` | Cliente não respondeu em 10s ([09:42] Diego) | Retry |
| `WEBHOOK_DELIVERY_REJECTED` | Resposta não-2xx | Retry |
| `WEBHOOK_CONNECTION_FAILED` | DNS, TLS ou conexão recusada | Retry |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | 5ª falha ([09:15] Diego) | Move para DLQ |
| `WEBHOOK_SIGNATURE_GENERATION_FAILED` | Secret ausente ou corrompida no momento de assinar | Move para DLQ — não adianta retentar |

---

## 8. Estratégias de resiliência

### 8.1 Timeouts

| Operação | Timeout | Origem |
| --- | --- | --- |
| `POST` no endpoint do cliente | **10 s** | [09:42] Diego |
| Lease de evento em `PROCESSING` | 20 s (2× o timeout) | Derivado — necessário para recuperar evento órfão após crash do worker |
| Intervalo de polling | 2 s | [09:09] Diego |

### 8.2 Retry e backoff

Já detalhado em §5.3: 5 tentativas, progressão 1m/5m/30m/2h/12h, ~14h36 de janela total ([09:17] Diego).

**Sem jitter nesta fase.** Com um único worker e volume baixo, o risco de thundering herd é desprezível. Se Q1 (rate limiting de saída, [09:39] Larissa) for reaberta, jitter entra junto.

### 8.3 Isolamento de falha

- **Worker fora do processo da API** ([09:11] Diego): deploy, crash ou restart de um não afeta o outro.
- **`PrismaClient` próprio no worker** ([09:30] Bruno): o pool de conexões do worker não compete com o da API.
- **Processamento sequencial dentro do batch:** um cliente lento atrasa o batch, mas com timeout de 10s o atraso é limitado. Batch pequeno ([09:08] Diego) mantém esse limite baixo.

### 8.4 Fallback

**Não há fallback automático de canal.** Email como alternativa foi explicitamente adiado para uma fase futura ([09:37] Larissa). O único caminho de recuperação é a DLQ com replay manual ([09:18] Diego), o que significa que **a detecção depende inteiramente da observabilidade da §9** — se ninguém olhar a DLQ, o evento morre lá.

### 8.5 Degradação do caminho crítico

O risco mais sério não é a entrega falhar — é a feature derrubar `changeStatus`. Mitigações no desenho:

- Validação de 64KB **antes** de inserir (§5.1), para falhar cedo.
- Filtro por assinante na inserção reduz o número de inserts por transação ([09:34] Bruno).
- Índices em `webhook_endpoints` por `customerId` e `active`, para que a leitura dentro da transação seja barata.
- Monitoração da latência de `changeStatus` com baseline anterior à feature (§9.1).

---

## 9. Observabilidade

O projeto usa **Pino** (`src/shared/logger/index.ts`), e a decisão foi não introduzir nada novo ([09:29] Bruno).

### 9.1 Métricas

| Métrica | Tipo | Por que importa |
| --- | --- | --- |
| `webhook_outbox_pending_count` | Gauge | Tamanho da fila. |
| `webhook_outbox_oldest_pending_age_seconds` | Gauge | **A métrica mais importante do sistema.** Com single-worker ([09:12] Diego), um worker morto não gera erro — gera silêncio. Só a idade do pendente mais antigo revela isso. Alerta se > 60s. |
| `webhook_pickup_latency_seconds` | Histogram | Do commit à primeira tentativa. Valida OT-2 e o requisito de 10s ([09:02] Marcos). |
| `webhook_delivery_duration_ms` | Histogram | Tempo de resposta do cliente, com label por `webhook_endpoint_id`. Antecipa timeouts. |
| `webhook_delivery_total{result}` | Counter | `success` / `retry` / `dead_letter`. |
| `webhook_dead_letter_count` | Gauge | Como o replay é manual ([09:18] Diego), qualquer valor > 0 exige ação humana. |
| `webhook_attempts_histogram` | Histogram | Distribuição de tentativas até sucesso. Se a maioria sucede na 2ª, o backoff inicial de 1min está mal calibrado. |
| `order_change_status_duration_ms` | Histogram | **Métrica de guarda.** Mede o impacto da feature no caminho crítico (R1 do [RFC](./RFC.md#62-riscos)). Exige baseline coletado antes do deploy. |

### 9.2 Logs

Pino, com `logger.info`/`warn`/`error` no padrão `{ contexto }, 'evento_snake_case'` já usado em `src/server.ts` (`'server_started'`) e em `src/middlewares/request-logger.middleware.ts` (`'http_request'`).

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook_event_enqueued` | debug | `eventId`, `orderId`, `webhookEndpointId`, `toStatus`, `requestId` |
| `webhook_delivery_attempt` | info | `eventId`, `webhookEndpointId`, `attemptNumber`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | warn | + `errorCode`, `nextAttemptAt` |
| `webhook_dead_lettered` | error | `eventId`, `webhookEndpointId`, `totalAttempts`, `failureReason` |
| `webhook_dlq_replayed` | warn | `deadLetterId`, `eventId`, **`replayedById`** — auditoria exigida por Sofia ([09:36] Sofia) |
| `webhook_secret_rotated` | warn | `webhookEndpointId`, `rotatedById`, `previousSecretExpiresAt`. **Nunca o valor da secret.** |
| `webhook_worker_tick` | trace | `batchSize`, `durationMs` |

> **Alteração obrigatória em `src/shared/logger/index.ts`.** O array `redactPaths` cobre hoje `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` — **mas não `*.secret`**. Sem acrescentar `*.secret` e `*.signature`, qualquer log que inclua o objeto do endpoint vaza a secret do cliente nos nossos próprios logs. É exatamente o incidente que Diego relatou do lado do cliente ([09:22] Diego), e é o único ponto em que "reusar sem mudar nada" não se sustenta ([ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).

### 9.3 Tracing

O projeto **não tem stack de tracing distribuído** — não há OpenTelemetry nem equivalente no `package.json`. O que existe é correlação por request id: `src/middlewares/request-logger.middleware.ts` gera (ou aceita via `x-request-id`) um UUID, grava em `req.id` e devolve no header `X-Request-Id`.

A estratégia, portanto, é **estender essa correlação existente através da fronteira assíncrona**, sem introduzir dependência nova:

1. O `requestId` da requisição `PATCH /orders/:id/status` é persistido na coluna `webhook_outbox.requestId` no momento da inserção.
2. Todo log do worker referente àquele evento inclui `requestId` e `eventId`.
3. Isso permite reconstruir a cadeia completa — request HTTP → transação → evento → N tentativas de entrega → DLQ → replay — com um `grep` por `requestId`, ou por `eventId` a partir de uma reclamação do cliente que cita o `X-Event-Id`.

Se no futuro o projeto adotar OpenTelemetry, `requestId` e `eventId` são os candidatos naturais a atributos de span; a decisão fica fora do escopo desta feature.

---

## 10. Dependências e compatibilidade

### 10.1 Dependências de runtime

**Nenhuma nova** ([09:29] Bruno; [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).

| Necessidade | Já disponível |
| --- | --- |
| HMAC-SHA256 | `node:crypto` (nativo) |
| Cliente HTTP com timeout | `fetch` + `AbortSignal.timeout` (nativo no Node ≥ 20; `package.json` exige `>=20`) |
| Geração de UUID | `uuid@11.0.3`, já em `dependencies` |
| Validação | `zod@3.23.8`, já em `dependencies` |
| Logging | `pino@9.5.0`, já em `dependencies` |
| ORM | `@prisma/client@5.22.0`, já em `dependencies` |

### 10.2 Dependências de infraestrutura

- **MySQL 8.0** — o mesmo de `docker-compose.yml`, sem instância nova ([09:07] Diego).
- **Processo supervisionado para o worker** — `npm run worker` precisa de restart automático (systemd, container ou equivalente), porque é single-worker e ponto único de falha ([09:11] Diego).

### 10.3 Alterações no `package.json`

Um script novo, ao lado de `dev` e `start` ([09:11] Larissa):

```json
"worker": "tsx watch --env-file=.env src/worker.ts",
"worker:start": "node --env-file=.env dist/worker.js"
```

### 10.4 Variáveis de ambiente

Acrescentar ao schema de `src/config/env.ts`, que hoje valida `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET` e `JWT_EXPIRES_IN`:

| Variável | Default | Origem |
| --- | --- | --- |
| `WEBHOOK_POLL_INTERVAL_MS` | `2000` | [09:09] Diego |
| `WEBHOOK_BATCH_SIZE` | `50` | [09:08] Diego — "batch pequeno" (valor exato não foi decidido) |
| `WEBHOOK_HTTP_TIMEOUT_MS` | `10000` | [09:42] Diego |
| `WEBHOOK_MAX_ATTEMPTS` | `5` | [09:15] Diego |
| `WEBHOOK_MAX_PAYLOAD_BYTES` | `65536` | [09:24] Diego / [09:24] Larissa |
| `WEBHOOK_SECRET_GRACE_PERIOD_HOURS` | `24` | [09:21] Sofia |

O worker reusa `DATABASE_URL`, sem variável própria ([09:30] Bruno).

### 10.5 Compatibilidade

- **Retrocompatível.** Nenhum endpoint existente muda de contrato. Nenhuma tabela existente muda de schema.
- **Migration aditiva:** quatro tabelas novas, nenhuma alteração nas existentes. A migration inicial em `prisma/migrations/20260519182739_init/` permanece intocada.
- **`OrderService.changeStatus` mantém a assinatura pública** `(id, input, userId)`, então `OrderController` e as rotas não mudam.
- **Efeito colateral novo em `changeStatus`**, porém: os testes em `tests/orders.test.ts` que exercitam mudança de status passam a disparar a inserção na outbox e precisam ser revistos.

---

## 11. Integração com o sistema existente

Esta seção mapeia cada ponto de contato com o código atual.

> **Convenção desta seção.** Todo caminho citado em um cabeçalho `###` ou sob "Verificado:" **existe hoje no repositório** e foi conferido antes de ser citado. Os únicos caminhos que **ainda não existem** são os arquivos a serem criados pela feature, sempre introduzidos por "criar" ou "novo": `src/worker.ts` e os arquivos de `src/modules/webhooks/` (`webhook.publisher.ts`, `webhook.processor.ts`, `webhook.errors.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais controller, service e repository).

### 11.1 `src/modules/orders/order.service.ts` — o ponto crítico

Bruno: "a alteração crítica é dentro do service de orders, no método `changeStatus`" ([09:40] Bruno).

**Estado atual:** `changeStatus` (linhas 126–179) abre `this.prisma.$transaction` e executa, em ordem: `findUnique` da order (132), validação de `from === to` (140), `canTransition` (147), `debitStock`/`replenishStock` (151–156), `tx.order.update` (158), `tx.orderStatusHistory.create` (159) e o `findUnique` de retorno (169).

**Alteração:** inserir a chamada **entre a linha 167 (fim do `orderStatusHistory.create`) e a linha 169 (`refreshed`)**, ainda dentro do `$transaction`:

```ts
// src/modules/orders/order.service.ts, dentro de changeStatus
await publishWebhookEvent(tx, order, from, to);
```

**Forma da integração:** função pura recebendo o `tx`, **não** injeção de repository ([09:41] Bruno; [09:41] Diego). Assinatura:

```ts
// src/modules/webhooks/webhook.publisher.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
  requestId?: string,
): Promise<void>
```

O type `TxClient = Prisma.TransactionClient` já está declarado na linha 24 do arquivo e é reaproveitado.

**Por que não injetar repository:** o construtor de `OrderService` (linhas 27–30) recebe hoje apenas `OrderRepository` e `PrismaClient`. Mudá-lo exigiria alterar `buildControllers` em `src/app.ts` (linhas 42–44) e todos os testes que instanciam o service.

**Consequência aceita:** se a inserção lançar, a transação inteira reverte — e isso é o comportamento desejado ([09:40] Bruno; [09:41] Diego).

### 11.2 `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` — reuso da hierarquia de erros

Bruno: "a gente já tem um padrão. Tem classe `AppError`, classes específicas tipo `InsufficientStockError`, `InvalidStatusTransitionError`. (...) Quero seguir igual pra webhook" ([09:28] Bruno).

**Modelo a seguir** — `InvalidStatusTransitionError` (linhas 45–53) estende `ConflictError`, que estende `AppError`, e fixa o `errorCode` no construtor:

```ts
export class InvalidStatusTransitionError extends ConflictError {
  constructor(from: string, to: string) {
    super(`Invalid status transition from ${from} to ${to}`, 'INVALID_STATUS_TRANSITION', { from, to });
  }
}
```

**Alteração:** criar `src/modules/webhooks/webhook.errors.ts` com as classes da §7.1, seguindo exatamente essa forma, e reexportá-las. As classes base reusadas de `src/shared/errors/index.ts` são `BadRequestError`, `NotFoundError`, `ConflictError` e `UnprocessableEntityError` — nenhuma precisa ser modificada.

### 11.3 `src/middlewares/error.middleware.ts` — reuso sem alteração

Bruno: "o middleware de erro centralizado já trata `AppError`, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno).

**Verificado:** a linha 15 faz `if (err instanceof AppError)` e serializa `statusCode`, `errorCode` e `details`. Como toda classe da §7.1 estende `AppError`, os erros `WEBHOOK_*` são tratados sem nenhum branch novo.

**Alteração: nenhuma.** É um dos critérios de aceite técnicos (OT-6).

### 11.4 `src/middlewares/auth.middleware.ts` — reuso de `authenticate` e `requireRole`

Larissa: "role `ADMIN` obrigatório no replay e a gente reaproveita o `requireRole` que já existe" ([09:36] Larissa).

**Verificado:** `authenticate` (linha 27) valida o Bearer e popula `req.user` com `{ id, email, role }`; `requireRole(...roles)` (linha 49) lança `ForbiddenError` quando a role não bate. O padrão de uso está em `src/modules/users/user.routes.ts` (linhas 12–18).

**Alteração:** nenhuma no middleware. Em `src/modules/webhooks/webhook.routes.ts`:

```ts
router.use(authenticate);                                    // todo o módulo
// ...
router.post('/admin/webhooks/dead-letter/:id/replay',
  requireRole('ADMIN'),                                      // só o replay
  validate({ params: deadLetterIdParamSchema }),
  controller.replayDeadLetter);
```

O `replayedById` da auditoria vem de `req.user.id`, já populado pelo `authenticate`.

### 11.5 `src/shared/logger/index.ts` — reuso **com uma alteração obrigatória**

Bruno: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" ([09:29] Bruno).

**Verificado:** `redactPaths` (linhas 4–11) contém `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`.

**Alteração obrigatória:** acrescentar `'*.secret'` e `'*.signature'` ao array. Sem isso, logar o objeto de endpoint expõe a secret do cliente — o mesmo tipo de incidente que Diego relatou ([09:22] Diego). Este é o **único** ponto em que a diretriz de reuso sem alteração ([09:30] Larissa) precisa ser flexibilizada, e deve entrar na revisão de segurança de Sofia ([09:46] Sofia).

### 11.6 `prisma/schema.prisma` — quatro modelos novos

**Verificado:** o arquivo define `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence`, todos com `@id @default(uuid()) @db.Char(36)` e `@@map` para snake_case. O enum `OrderStatus` (linhas 16–23) tem os seis valores usados no filtro de assinatura.

**Alteração:** acrescentar `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery` e `WebhookDeadLetter` (§4), mais o enum `WebhookEventStatus` com `PENDING`/`PROCESSING`/`FAILED`/`DELIVERED` ([09:08] Diego). `WebhookEndpoint` ganha relação com `Customer`; `WebhookDeadLetter.replayedById` referencia `User`. Migration aditiva, sem alteração de tabela existente.

### 11.7 `src/server.ts` e `src/config/database.ts` — molde para `src/worker.ts`

Larissa: "tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em `src/server.ts`, criar um `src/worker.ts`" ([09:11] Larissa).

**Verificado:** `src/server.ts` importa `prisma` de `src/config/database.ts`, sobe o servidor e registra `SIGINT`/`SIGTERM` com `prisma.$disconnect()` antes do `process.exit(0)` (linhas 13–21). `src/config/database.ts` expõe a factory `createPrismaClient()` além da instância singleton.

**Alteração:** criar `src/worker.ts` espelhando essa estrutura, mas chamando `createPrismaClient()` para ter **client próprio** — Bruno: "separado. `PrismaClient` é por processo. Mesmo banco, mesma `DATABASE_URL`, mas instância nova porque é outro processo Node" ([09:30] Bruno). O shutdown gracioso deve terminar o batch em andamento antes de desconectar, para não deixar eventos em `PROCESSING` órfãos.

`src/worker.ts` **não importa** `src/app.ts` nem sobe Express (OT-3).

### 11.8 `src/routes/index.ts` e `src/app.ts` — registro do módulo

**Verificado:** o type `Controllers` (`src/routes/index.ts`, linhas 13–19) lista os cinco controllers, e `buildApiRouter` monta cada router sob seu prefixo. `buildControllers` (`src/app.ts`, linhas 26–53) instancia manualmente repository → service → controller.

**Alteração:** acrescentar `webhooks: WebhookController` ao type `Controllers`, `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` em `buildApiRouter`, e o trio `WebhookRepository → WebhookService → WebhookController` em `buildControllers`, seguindo o padrão das linhas 42–44.

### 11.9 `src/middlewares/validate.middleware.ts` — validação de `https` via Zod

Sofia: "TLS obrigatório. URL do webhook tem que ser https. Se o cliente cadastrar http, recusamos com erro de validação. Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod" ([09:23] Sofia).

**Verificado:** `validate({ body, query, params })` (linha 11) parseia com Zod e converte `ZodError` em `ValidationError` com `details` estruturado.

**Alteração:** nenhuma no middleware. Em `src/modules/webhooks/webhook.schemas.ts`, seguindo o estilo de `src/modules/customers/customer.schemas.ts`:

```ts
const httpsUrlSchema = z.string().url().max(2048)
  .refine((u) => u.startsWith('https://'), { message: 'Webhook URL must use https' });
```

### 11.10 `src/modules/orders/order.status.ts` — fonte da verdade do filtro

**Verificado:** define o mapa de transições válidas e as funções `canTransition`, `shouldDebitStock` e `shouldReplenishStock`.

**Alteração:** nenhuma. O módulo de webhooks apenas **consome** o enum `OrderStatus` para validar `subscribedStatuses`, usando `z.nativeEnum(OrderStatus)` como `src/modules/orders/order.schemas.ts` já faz na linha 19. O evento é emitido depois de `canTransition` já ter aprovado a transição, o que garante que nenhum webhook reporta transição inválida.

### 11.11 `src/shared/http/response.ts` — paginação dos endpoints de listagem

**Verificado:** `paginated(data, page, pageSize, total)` retorna `{ data, pagination }`, usado por `OrderService.list` (linha 41).

**Alteração:** nenhuma. Os endpoints §6.2 e §6.6 usam o mesmo helper e devolvem o mesmo envelope.

### 11.12 `tests/orders.test.ts` — testes existentes impactados

**Verificado:** o arquivo existe e exercita o fluxo de pedidos, incluindo mudança de status.

**Impacto:** como `changeStatus` passa a inserir na outbox, os testes que mudam status passarão a tocar as tabelas novas. Precisam ser revistos para (a) não quebrar quando não há endpoint cadastrado — caminho que deve ser no-op ([09:34] Bruno) — e (b) cobrir o caso com assinante. `tests/helpers/factories.ts` ganha factory de `WebhookEndpoint`.

---

## 12. Critérios de aceite técnicos

| # | Critério | Como verificar |
| --- | --- | --- |
| CA-01 | Transição de status commitada com 1 assinante gera exatamente 1 linha em `webhook_outbox`. | Teste de integração. |
| CA-02 | Transição revertida (ex.: `InsufficientStockError`) não deixa nenhuma linha na outbox. | Teste que força estoque insuficiente e checa `COUNT(*) = 0`. |
| CA-03 | Transição sem nenhum endpoint assinando o `to_status` não gera linha. | Teste com `subscribedStatuses` desalinhado ([09:34] Bruno). |
| CA-04 | Falha na inserção do evento reverte a mudança de status. | Teste com mock que lança na inserção; `order.status` permanece o anterior ([09:40] Bruno). |
| CA-05 | `X-Signature` é HMAC-SHA256 verificável do corpo exato enviado. | Teste que recalcula o HMAC a partir do body capturado. |
| CA-06 | `X-Event-Id` é idêntico nas 5 tentativas do mesmo evento **e** no replay de DLQ. | Teste que força 5 falhas e compara os headers. |
| CA-07 | Backoff respeita 1m/5m/30m/2h/12h. | Teste com clock controlado sobre `nextAttemptAt`. |
| CA-08 | 5ª falha move para `webhook_dead_letter` e remove da outbox. | Teste de integração ([09:18] Diego). |
| CA-09 | Replay sem role `ADMIN` retorna `403` com código `FORBIDDEN`. | Teste com JWT de `OPERATOR` ([09:36] Sofia). |
| CA-10 | Replay bem-sucedido registra `replayedById` em log e no banco. | Teste que inspeciona a linha e o log ([09:36] Sofia). |
| CA-11 | Cadastro com URL `http` retorna `400` / `WEBHOOK_INVALID_URL`. | Teste de schema ([09:23] Sofia). |
| CA-12 | Payload acima de 64KB gera `WEBHOOK_PAYLOAD_TOO_LARGE` e **não** é truncado. | Teste com payload inflado ([09:24] Larissa). |
| CA-13 | Timeout de 10s marca falha e agenda retry. | Teste com servidor que atrasa 11s ([09:42] Diego). |
| CA-14 | Após rotação, a secret antiga expira em exatamente 24h. | Teste com clock controlado ([09:21] Sofia). |
| CA-15 | `git diff` de `src/middlewares/error.middleware.ts` vazio ao fim da implementação. | Revisão de código (OT-6; [09:29] Bruno). |
| CA-16 | `git diff` da seção `dependencies` do `package.json` vazio. | Revisão de código (OT-7; [09:29] Bruno). |
| CA-17 | `redactPaths` em `src/shared/logger/index.ts` inclui `*.secret`; nenhum log contém valor de secret. | Teste que loga o objeto de endpoint e inspeciona a saída. |
| CA-18 | Latência de captação ≤ 2s no caminho feliz. | Teste de integração medindo commit → primeira tentativa ([09:10] Larissa). |
| CA-19 | Evento em `PROCESSING` com lease expirado volta para `PENDING`. | Teste que simula crash do worker no meio do batch. |
| CA-20 | `src/worker.ts` não importa `src/app.ts` nem sobe Express. | Inspeção de imports (OT-3; [09:11] Diego). |

---

## 13. Riscos e mitigação

| # | Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| RT-1 | **Degradação de `changeStatus`.** A transação ganha leitura de config + N inserts, no caminho crítico do OMS. | Média | Alto | Baseline de `order_change_status_duration_ms` antes do deploy (§9.1); índices em `webhook_endpoints(customerId, active)`; teste de carga na transição `PENDING → PAID`, a mais pesada. |
| RT-2 | **Bug no HMAC por reserialização do corpo.** Assinar um JSON e enviar outro (chaves reordenadas, espaçamento diferente) quebra a verificação do cliente de forma intermitente. | Média | Alto | Serializar uma vez e usar a mesma string para assinar e enviar (§5.2); CA-05; revisão de Sofia ([09:46] Sofia). |
| RT-3 | **Vazamento de secret em log.** `redactPaths` não cobre `*.secret`. | Alta, se nada for feito | Alto | CA-17; §11.5; nunca logar o objeto de endpoint inteiro. |
| RT-4 | **Worker morre silenciosamente.** Single-worker; eventos acumulam sem erro aparente ([09:12] Diego). | Média | Alto | Alerta sobre `webhook_outbox_oldest_pending_age_seconds > 60s` (§9.1); supervisão do processo com restart; CA-19 para eventos órfãos. |
| RT-5 | **`X-Event-Id` regenerado no replay**, quebrando a dedup do cliente justamente na recuperação. | Média | Médio | Reusar `originalEventId` (§5.4); CA-06. |
| RT-6 | **Crescimento sem controle da outbox.** Sem arquivamento ([09:08] Diego) e com snapshot de payload por linha ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md)). | Alta no médio prazo | Médio | Filtro na inserção reduz volume ([09:34] Bruno); monitorar tamanho da tabela desde o dia 1; decidir a política de retenção (Q4 do [RFC](./RFC.md#5-questões-em-aberto)). |
| RT-7 | **Cliente com muitos endpoints multiplica inserts na transação** (fan-out de escrita). | Baixa | Médio | Limite prático de endpoints por customer; monitorar inserts por transação. |
| RT-8 | **Testes existentes quebram** porque `changeStatus` ganha efeito colateral. | Alta | Baixo | Revisar `tests/orders.test.ts`; garantir que ausência de assinante é no-op (CA-03). |
| RT-9 | **Cliente ignora `X-Event-Id`** e processa duplicata. | Média | Médio | Documentação em destaque no portal do desenvolvedor ([09:26] Marcos); CA-06 garante que o header é confiável. |
