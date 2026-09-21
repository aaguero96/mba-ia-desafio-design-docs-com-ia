# ADR-001 — Padrão Outbox transacional no MySQL existente

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)
- **Origem:** Reunião técnica "Sistema de Webhooks de Notificação de Pedidos" — `TRANSCRICAO.md` [09:03]–[09:08]
- **Relacionada a:** [ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md), [ADR-006](./ADR-006-reuso-dos-padroes-existentes-do-projeto.md), [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md)

## Contexto

O OMS precisa notificar clientes B2B quando o status de um pedido muda. A primeira pergunta colocada na reunião foi se o disparo aconteceria de forma síncrona dentro do service de pedidos ou através de algum mecanismo assíncrono ([09:03] Larissa).

A transação de mudança de status já é cara. Em `src/modules/orders/order.service.ts`, o método `changeStatus` abre um `this.prisma.$transaction` que atualiza a order, insere em `order_status_history` e, na transição `PENDING → PAID`, decrementa `stockQuantity` de cada produto do pedido. Bruno apontou que acrescentar uma chamada HTTP no meio disso faria um cliente lento travar a mudança de status de outros pedidos ([09:04] Bruno), e que um cliente fora do ar colocaria o sistema na posição absurda de ter que dar rollback numa mudança de status válida ([09:04] Bruno).

Há ainda uma exigência de integridade explícita: não pode existir o caso de o status mudar e o evento não sair ([09:40] Bruno; [09:41] Diego).

## Decisão

Adotar o **padrão Outbox sobre o MySQL já existente**.

Quando o status do pedido muda, **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history`, uma linha é inserida na tabela `webhook_outbox` com o evento ([09:06] Diego). Se a transação principal comita, o evento está registrado; se ela dá rollback, o evento some junto. Não existe janela de inconsistência.

A tabela terá índice no campo de status — com os valores `PENDING`, `PROCESSING`, `FAILED`, `DELIVERED` — e em `created_at`, e o worker lerá apenas os pendentes em batch pequeno ([09:08] Diego). A chave primária é UUID, seguindo o padrão do resto do projeto ([09:51] Larissa; confirmado em `prisma/schema.prisma`, onde todos os modelos usam `@id @default(uuid()) @db.Char(36)`).

## Alternativas Consideradas

### 1. Disparo HTTP síncrono dentro de `changeStatus`

Chamar o endpoint do cliente diretamente na transação de mudança de status.

**Descartada.** A transação já é pesada (order + history + estoque) e um cliente lento passaria a travar mudança de status de outros pedidos ([09:04] Bruno). Pior: uma indisponibilidade do cliente obrigaria a escolher entre dar rollback numa operação de negócio legítima ou perder o evento. Diego foi categórico: "síncrono está fora de questão" ([09:06] Diego).

**Trade-off do descarte:** abrimos mão da latência mínima possível (entrega imediata) em troca de desacoplamento e de não deixar a disponibilidade do nosso core depender da disponibilidade de terceiros.

### 2. Redis Streams ou fila dedicada

Publicar o evento em um broker externo e consumir de lá.

**Descartada.** Exigiria subir e operar infraestrutura nova. Larissa levantou a opção e Diego respondeu que, para um time pequeno, subir um Redis Cluster só para isso é overengineering, e que o outbox no MySQL existente resolve ([09:07] Larissa; [09:07] Diego).

**Trade-off do descarte:** perdemos throughput teórico e a possibilidade de fan-out nativo de um broker, em troca de zero infraestrutura adicional e de uma garantia transacional que um broker externo não daria de graça (publicar no Redis dentro da transação MySQL reintroduziria o problema do dual-write).

## Consequências

### Positivas

- Atomicidade real entre a mudança de status e o registro do evento, sem necessidade de two-phase commit ou de compensação.
- Nenhuma dependência de infraestrutura nova: mesmo MySQL, mesmo Prisma, mesmo `docker-compose.yml`.
- A tabela `webhook_outbox` é auditável por SQL comum — dá para investigar incidente com um `SELECT`.
- O rollback da transação de status descarta o evento automaticamente, o que elimina a classe de bug "evento de um status que nunca existiu".

### Negativas

- Carga de escrita adicional no MySQL de produção, no mesmo banco que atende a API.
- A tabela cresce indefinidamente se não houver arquivamento. Diego mencionou arquivar linhas entregues depois de ~30 dias, mas declarou isso explicitamente fora do escopo desta feature ([09:08] Diego) — fica como dívida conhecida.
- O `changeStatus` passa a ter mais um ponto de falha dentro da transação: se a inserção na outbox falhar, a mudança de status inteira é revertida. Isso é intencional ([09:40] Bruno), mas significa que um bug no módulo de webhooks pode derrubar uma operação de negócio central.
- A latência de entrega passa a depender do intervalo de polling do worker, e não do commit (ver [ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md)).

### Trade-off explícito

Trocamos **latência mínima e isolamento do core** por **garantia de consistência e custo zero de infraestrutura**. O acoplamento transacional entre `changeStatus` e a outbox foi aceito conscientemente: o time preferiu que uma falha de inserção do evento derrube a mudança de status a ter uma mudança de status sem evento correspondente.
