# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Bruno (Eng. Pedidos), Larissa (Tech Lead), Diego (Eng. Plataforma)
- **Origem:** `TRANSCRICAO.md` [09:27]–[09:30], [09:36], [09:41]
- **Relacionada a:** [ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md), [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md)

> Este é o ADR que ancora a feature no código existente. Todos os caminhos citados abaixo foram verificados no repositório.

## Contexto

O OMS já está em produção e tem convenções estabelecidas. Bruno abriu o bloco de estrutura de código afirmando que há um padrão claro na codebase e propondo que webhooks o siga em vez de inventar um novo ([09:27] Bruno).

O que existe hoje, verificado no repositório:

| Padrão | Onde vive | Evidência |
| --- | --- | --- |
| Módulo por domínio com controller, service, repository, routes e schemas | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/`, `src/modules/users/` | Cada pasta tem exatamente os 5 arquivos |
| Hierarquia de erros de aplicação | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | `AppError` com `statusCode`, `errorCode` e `details`; subclasses como `InvalidStatusTransitionError` (código `INVALID_STATUS_TRANSITION`) e `InsufficientStockError` (código `INSUFFICIENT_STOCK`) |
| Tratamento centralizado de erro | `src/middlewares/error.middleware.ts` | Trata `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` e serializa para `{ error: { code, message, details } }` |
| Validação declarativa com Zod | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` de cada módulo | `validate({ body, query, params })` |
| Logging estruturado com Pino | `src/shared/logger/index.ts` | `logger` com `redact`, `base` e timestamp ISO |
| Autenticação e autorização | `src/middlewares/auth.middleware.ts` | `authenticate` e `requireRole('ADMIN')`, este último já usado em `src/modules/users/user.routes.ts` |
| Composição de dependências | `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`) | Instanciação manual repository → service → controller |
| Paginação | `src/shared/http/response.ts` | `paginated(data, page, pageSize, total)` |
| Identificadores | `prisma/schema.prisma` | Todos os modelos usam `@id @default(uuid()) @db.Char(36)` |

## Decisão

**Reuso máximo do que já existe. O módulo de webhooks é mais um módulo igual aos outros, não um subsistema paralelo** ([09:30] Larissa).

Concretamente:

1. **Estrutura de módulo.** Nova pasta `src/modules/webhooks/` com controller, service, repository, routes e schemas, espelhando `src/modules/orders/` ([09:27] Bruno).
2. **Worker.** Entry-point `src/worker.ts` separada, com a lógica de processamento em `src/modules/webhooks/webhook.processor.ts` dentro do módulo ([09:28] Bruno; ver [ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md)).
3. **Erros.** Novas classes estendendo `AppError`, seguindo o formato de `InsufficientStockError` e `InvalidStatusTransitionError`, com **prefixo `WEBHOOK_` em todos os códigos do módulo** ([09:28] Bruno; [09:29] Larissa). Exemplos citados por Bruno: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`.
4. **Middleware de erro.** Nenhuma alteração. Como as novas classes estendem `AppError`, o `errorMiddleware` já as serializa corretamente — Bruno verificou isso: "vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno).
5. **Logger.** Pino, sem nenhuma dependência nova de logging ([09:29] Bruno).
6. **Autorização.** Reaproveitar `requireRole` no endpoint de replay de DLQ ([09:36] Larissa).
7. **Prisma.** Mesmo banco e mesma `DATABASE_URL`; o worker instancia seu próprio `PrismaClient` porque client é por processo ([09:30] Bruno). A factory `createPrismaClient()` de `src/config/database.ts` já cobre esse caso.
8. **Integração com `changeStatus` por função pura.** Em vez de injetar um repository inteiro no `OrderService`, expor uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o tx client da transação corrente ([09:41] Bruno). Diego endossou: "função pura recebendo o tx, não precisa injetar repository inteiro" ([09:41] Diego).
9. **IDs.** UUID, seguindo o padrão do resto do projeto ([09:51] Larissa).

## Alternativas Consideradas

### 1. Subsistema de webhooks separado, com suas próprias convenções

Tratar webhooks como componente de infraestrutura e não como domínio de negócio, com sua própria camada de erros, seu próprio logger e sua própria organização.

**Descartada.** Bruno propôs seguir o padrão existente e Diego e Larissa concordaram sem contraproposta ([09:27]–[09:30]). Um subsistema paralelo significaria que um desenvolvedor do time precisa aprender duas convenções e que o `errorMiddleware` precisaria de um branch novo para erros de webhook.

**Trade-off do descarte:** o módulo fica amarrado às convenções do OMS e herda suas limitações, em troca de custo cognitivo zero para o time e de reaproveitar infraestrutura já testada em produção.

### 2. Injetar `WebhookRepository` no `OrderService`

Alternativa discutida por Bruno antes de propor a função pura: "vai me obrigar a passar um repository do webhook pro OrderService ou uma função de enqueue event" ([09:41] Bruno).

**Descartada.** Acrescentaria uma dependência de construtor ao `OrderService`, que hoje recebe apenas `OrderRepository` e `PrismaClient` (`src/modules/orders/order.service.ts`, linhas 27–30). Todos os testes existentes que constroem `OrderService` passariam a precisar de um novo dublê.

**Trade-off do descarte:** a função `publishWebhookEvent` fica como dependência de módulo importada diretamente, o que é menos "injetável" em teste unitário, em troca de não alterar a assinatura do `OrderService` nem quebrar a construção feita em `buildControllers` (`src/app.ts`, linhas 42–44).

## Consequências

### Positivas

- Nenhuma alteração necessária em `src/middlewares/error.middleware.ts`: erros `WEBHOOK_*` são serializados pelo mesmo caminho que `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`.
- Nenhuma dependência nova no `package.json` para logging, validação ou autenticação. HMAC usa o `crypto` nativo do Node.
- Um desenvolvedor que conhece `src/modules/orders/` consegue navegar `src/modules/webhooks/` sem contexto adicional.
- O registro do router segue o caminho já existente: adicionar `webhooks` ao type `Controllers` e ao `buildApiRouter` em `src/routes/index.ts`, e instanciar em `buildControllers` de `src/app.ts`.
- O prefixo `WEBHOOK_` torna trivial filtrar, alertar e agrupar erros do módulo em log e em métrica.

### Negativas

- O módulo de webhooks **não é um domínio de negócio como os outros**: ele tem um consumidor em background, estado de retry e efeitos colaterais externos. Forçá-lo no formato controller/service/repository pode ficar desconfortável no `webhook.processor.ts`, que não atende requisição HTTP nenhuma.
- A composição de dependências em `src/app.ts` é manual e já tem 5 módulos; adicionar webhooks aumenta essa função, que continua sem container de DI.
- O `OrderService` passa a ter uma dependência implícita (import direto de `publishWebhookEvent`) em vez de explícita por construtor, o que torna o teste de `changeStatus` sem webhooks um pouco mais difícil de isolar. Os testes existentes em `tests/orders.test.ts` que exercitam `changeStatus` precisam ser revistos.
- **Ajuste necessário no logger:** o `redact.paths` em `src/shared/logger/index.ts` cobre `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas **não cobre `*.secret`**. Reusar o logger sem adicionar esse path exporia a secret do webhook nos nossos logs (ver [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md)). Este é o único ponto em que "reuso sem alteração" não se sustenta.

### Trade-off explícito

Trocamos **liberdade de desenhar o módulo na forma que a natureza dele pediria** (um subsistema orientado a eventos) por **consistência com a codebase e velocidade de entrega**. O custo aparece concentrado no worker, que é o componente que menos se parece com os outros módulos; o benefício aparece em todo o resto — CRUD, erros, validação, autorização e logging saem praticamente de graça.
