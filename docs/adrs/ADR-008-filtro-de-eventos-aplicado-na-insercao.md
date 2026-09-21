# ADR-008 — Filtro de status assinados aplicado na inserção da outbox, não no envio

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Marcos (PM)
- **Origem:** `TRANSCRICAO.md` [09:33]–[09:34]
- **Relacionada a:** [ADR-001](./ADR-001-outbox-transacional-no-mysql.md), [ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md)

## Contexto

Cada endpoint de webhook cadastrado declara quais status quer ouvir. Marcos descreveu o caso de uso: "só quero saber quando vira SHIPPED e DELIVERED" ([09:33] Marcos). O filtro é, portanto, uma lista de valores do enum `OrderStatus` — definido em `prisma/schema.prisma` como `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`.

Isso abre uma escolha de posicionamento que Diego formulou diretamente: filtra na inserção do outbox ou na hora de mandar? ([09:34] Diego).

A pergunta importa porque a inserção acontece **dentro da transação de `changeStatus`** ([ADR-001](./ADR-001-outbox-transacional-no-mysql.md)), enquanto o envio acontece no worker, fora dela.

## Decisão

**O filtro é aplicado na inserção.** Se nenhum webhook do customer quer aquele status, a linha nem é inserida na outbox ([09:34] Bruno). Diego concordou ([09:34] Diego).

Consequência direta do desenho: a inserção deixa de ser "uma linha por mudança de status" e passa a ser **uma linha por (mudança de status × endpoint assinante)**. Um customer com três endpoints, todos assinando `SHIPPED`, gera três linhas na mesma transação; um customer sem nenhum endpoint assinando `SHIPPED` gera zero.

Isso implica que a transação de `changeStatus` em `src/modules/orders/order.service.ts` passa a ler a configuração de webhooks do customer antes de decidir o que inserir.

## Alternativas Consideradas

### 1. Filtrar no momento do envio

Inserir um evento por mudança de status, sem consultar assinaturas, e deixar o worker decidir para quem enviar.

**Descartada.** Bruno argumentou pela economia: se nenhum webhook do customer quer aquele status, nem insere — economiza linha na tabela ([09:34] Bruno). Dado que não há política de arquivamento nesta fase ([09:08] Diego), linha que não precisa existir é linha que nunca é limpa.

**Trade-off do descarte:** a transação de mudança de status passa a depender do estado da configuração de webhooks, em troca de uma outbox que só contém trabalho real.

### 2. Filtrar nos dois pontos (inserção e envio)

Não levantada na reunião; registrada como alternativa plausível, comum em sistemas que toleram configuração mudando durante a janela de retry.

**Descartada.** Contradiz o snapshot decidido em [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md): o evento é um registro do que era verdade no instante da transição. Revalidar a assinatura no envio significaria que desativar um endpoint apaga retroativamente eventos já capturados, comportamento que ninguém pediu.

**Trade-off do descarte:** eventos capturados antes de uma alteração de assinatura serão entregues segundo a assinatura antiga, em troca de um único ponto de decisão e de semântica consistente com o snapshot.

## Consequências

### Positivas

- A outbox contém apenas trabalho que realmente precisa ser executado; o worker não gasta ciclo lendo evento que vai descartar.
- O volume da tabela passa a ser função da demanda real dos clientes, e não do volume total de mudanças de status do OMS — o que importa dado que o arquivamento ficou fora de escopo ([09:08] Diego).
- O ponto de decisão é único e fica junto do snapshot do payload, o que torna o comportamento fácil de explicar: o que foi capturado, foi capturado como era naquele instante.

### Negativas

- **A transação de `changeStatus` fica mais pesada.** Ela já faz update em `orders`, insere em `order_status_history` e, na transição `PENDING → PAID`, atualiza `stockQuantity` de cada item (`src/modules/orders/order.service.ts`, linhas 151–167). Agora acrescenta uma leitura da configuração de webhooks e N inserções. Isso é exatamente a preocupação que Bruno levantou contra o envio síncrono ([09:04] Bruno) — em escala muito menor, já que são operações no mesmo banco e não chamadas de rede, mas na mesma direção.
- **Mudança de assinatura não é retroativa.** Se o cliente adicionar `SHIPPED` à lista logo depois de um pedido ter virado `SHIPPED`, o evento não existe e não há como recuperá-lo. Precisa estar documentado no portal do desenvolvedor.
- Cadastrar um webhook novo não gera histórico: o cliente só recebe eventos a partir do cadastro.
- Fan-out na escrita: um customer com muitos endpoints assinando o mesmo status multiplica as inserções dentro da transação de negócio.

### Trade-off explícito

Trocamos **leveza da transação de `changeStatus`** por **uma outbox enxuta e um único ponto de decisão**. O risco é conhecido e mensurável — é o mesmo eixo do argumento contra o síncrono ([09:04] Bruno) —, e por isso a latência da transação de mudança de status entra como métrica de observabilidade obrigatória, com o baseline de antes da feature como referência.
