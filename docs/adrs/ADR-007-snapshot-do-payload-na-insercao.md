# ADR-007 — Persistir o payload renderizado (snapshot) no momento da inserção na outbox

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pedidos), Diego (Eng. Plataforma)
- **Origem:** `TRANSCRICAO.md` [09:43], [09:51]–[09:52]
- **Relacionada a:** [ADR-001](./ADR-001-outbox-transacional-no-mysql.md), [ADR-003](./ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Já no fim da call, depois da saída de Marcos e Sofia, Bruno levantou a última questão em aberto do modelo de dados: o evento da outbox guarda o payload já renderizado, ou guarda só o `order_id` e renderiza na hora do envio? ([09:51] Bruno).

A pergunta não é cosmética. Com retry de até ~15 horas ([ADR-003](./ADR-003-retry-com-backoff-exponencial-e-dlq.md)), existe uma janela larga entre o fato e a entrega efetiva. Nessa janela o pedido pode mudar de novo: em `src/modules/orders/order.status.ts` a máquina de estados permite `PAID → PROCESSING → SHIPPED → DELIVERED`, e várias transições podem ocorrer em sequência rápida — cenário que a própria Larissa levantou mais cedo ([09:12] Larissa).

## Decisão

**O payload é renderizado e persistido na outbox no momento da inserção — um snapshot do estado no instante em que o status mudou** ([09:52] Larissa; [09:52] Diego; confirmado por Bruno em [09:52]).

Larissa justificou: se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou; a alternativa produz casos esquisitos ([09:52] Larissa).

O conteúdo do snapshot é o payload JSON definido por Diego ([09:43] Diego):

- `event_id`
- `event_type`, no formato `order.status_changed`
- `timestamp` em ISO 8601
- `order_id`
- `order_number`
- `from_status` e `to_status`
- `customer_id`
- campos básicos da order, como `total_cents`

**Os `items` do pedido não vão no payload**, deliberadamente, para não inflar o evento. Se o cliente quiser detalhes, ele chama `GET /orders/:id` depois ([09:43] Diego). Bruno endossou: mantém payload enxuto ([09:44] Bruno).

A chave primária da linha é o UUID que também viaja como `X-Event-Id` ([09:51] Larissa; ver [ADR-005](./ADR-005-entrega-at-least-once-com-x-event-id.md)).

## Alternativas Consideradas

### 1. Guardar apenas `order_id` e renderizar o payload no momento do envio

A alternativa explicitamente colocada por Bruno ([09:51] Bruno).

**Descartada.** Produziria eventos mentirosos. Um evento de transição `PAID → PROCESSING` que só é entregue 12 horas depois, após retry, renderizaria um pedido que já está `SHIPPED` — o cliente receberia `to_status: PROCESSING` junto de um snapshot dizendo outra coisa. Larissa classificou como "caso esquisito" ([09:52] Larissa).

**Trade-off do descarte:** pagamos armazenamento duplicado (o JSON na outbox além da própria order) e perdemos a capacidade de corrigir retroativamente o formato do payload de eventos já enfileirados, em troca de o evento ser um registro fiel e imutável do que aconteceu.

### 2. Incluir os `items` do pedido no payload

Não foi proposta como alternativa formal, mas está implícita na decisão de Diego de deixá-los de fora ([09:43] Diego).

**Descartada.** Um pedido com muitos itens inflaria o evento, e o time definiu teto de 64KB com erro — não truncamento — para payload ([09:23] Sofia; [09:24] Diego; [09:24] Larissa). Incluir itens aproximaria eventos legítimos desse teto sem necessidade.

**Trade-off do descarte:** o cliente precisa de uma segunda chamada (`GET /orders/:id`) quando quer detalhe de itens, em troca de payload previsível, pequeno e longe do limite de tamanho.

## Consequências

### Positivas

- O evento é um registro histórico fiel: o que o cliente recebe corresponde ao estado do pedido no instante da transição, mesmo entregue 15 horas depois.
- A DLQ ganha valor de auditoria real, porque guarda o payload exato que tentamos entregar ([09:18] Diego), e não uma reconstrução aproximada.
- O worker não precisa ler `orders`, `order_items` nem `customers` no momento do envio — ele lê uma linha da outbox e dispara. Isso reduz a carga do worker e o desacopla do schema de pedidos.
- O replay de DLQ reenvia exatamente o mesmo conteúdo da tentativa original, o que é coerente com a estabilidade exigida do `X-Event-Id` ([ADR-005](./ADR-005-entrega-at-least-once-com-x-event-id.md)).

### Negativas

- Duplicação de dados: o mesmo conteúdo existe na `orders` e replicado em cada linha da outbox. Com volume alto, a tabela cresce mais rápido do que cresceria guardando só o `order_id` — o que agrava a ausência de política de arquivamento, declarada fora de escopo ([09:08] Diego).
- Mudanças no formato do payload não se aplicam retroativamente: eventos já na outbox serão entregues no formato antigo. Versionamento de payload vira uma preocupação futura.
- O JSON persistido contém dados de pedido e de cliente; a tabela passa a ter relevância para políticas de retenção e privacidade, mais do que uma tabela de fila puramente técnica teria.

### Trade-off explícito

Trocamos **economia de armazenamento e flexibilidade de formato** por **fidelidade histórica do evento**. Foi o mesmo raciocínio que já sustenta `order_status_history` em `prisma/schema.prisma`, que também guarda `fromStatus` e `toStatus` materializados em vez de derivá-los: em registro de fato consumado, o custo de armazenamento é menor que o custo de ambiguidade.
