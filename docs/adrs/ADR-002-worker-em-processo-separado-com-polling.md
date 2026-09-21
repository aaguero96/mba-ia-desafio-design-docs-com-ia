# ADR-002 — Worker em processo separado, consumindo a outbox por polling de 2 segundos

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Marcos (PM)
- **Origem:** `TRANSCRICAO.md` [09:08]–[09:13], [09:28]–[09:30]
- **Relacionada a:** [ADR-001](./ADR-001-outbox-transacional-no-mysql.md), [ADR-003](./ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-006](./ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

## Contexto

Com a outbox decidida ([ADR-001](./ADR-001-outbox-transacional-no-mysql.md)), restava definir **como** os eventos pendentes saem da tabela e viram chamadas HTTP ([09:08] Larissa).

Duas restrições delimitam o espaço de solução:

1. **Requisito de latência.** Marcos perguntou aos clientes o que eles entendem por tempo real e a resposta foi: qualquer coisa abaixo de 10 segundos ([09:02] Marcos). O importante para eles é não ficar pendurado.
2. **Restrição de plataforma.** O banco é MySQL. Diferente do Postgres, o MySQL não tem `NOTIFY`/`LISTEN`; uma trigger de banco executa SQL mas não consegue avisar um processo externo ([09:09] Diego).

Havia ainda a questão de onde o worker roda. Diego alertou que, se ele rodar dentro da mesma instância da API, um restart da API derruba o worker junto ([09:11] Diego).

## Decisão

**Worker em polling, a cada 2 segundos, rodando como processo separado da API.**

O loop busca os eventos pendentes mais antigos, processa e marca ([09:09] Diego). Com polling de 2s, a latência de captação no pior caso é de 2 segundos, bem dentro do teto de 10s — o time aceitou isso explicitamente ([09:10] Larissa; [09:10] Marcos).

O worker ganha uma entry-point própria, `src/worker.ts`, espelhando o que já existe em `src/server.ts`, e um script `npm run worker` no `package.json` ([09:11] Larissa). A lógica de processamento vive dentro do módulo, em `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno).

O worker conecta no **mesmo banco, com a mesma `DATABASE_URL`**, mas instancia seu **próprio `PrismaClient`**, porque o client é por processo ([09:11] Bruno; [09:30] Bruno). Em `src/config/database.ts` já existe a factory `createPrismaClient()` que serve exatamente para isso.

Por ora roda **um único worker**. A ordenação é implícita por `created_at` da outbox, o que garante ordem por `order_id` enquanto houver apenas um consumidor ([09:12] Diego).

## Alternativas Consideradas

### 1. Trigger de banco notificando o worker

Bruno perguntou se não daria para usar trigger do MySQL e ser mais reativo ([09:09] Bruno).

**Descartada.** A trigger só executa SQL; para avisar um processo externo seria preciso improvisar algo como escrever em arquivo ou bater num endpoint, o que Diego classificou como esquisito ([09:09] Diego).

**Trade-off do descarte:** abrimos mão de latência sub-segundo e de evitar leituras ociosas ao banco, em troca de um mecanismo simples que não depende de recursos que o MySQL não oferece nativamente.

### 2. Worker dentro do processo da API

Rodar o loop de processamento em background na mesma instância Express.

**Descartada.** Um restart ou crash da API levaria o worker junto ([09:11] Diego), e o processamento de webhooks passaria a competir por event loop com o atendimento de requisições HTTP.

**Trade-off do descarte:** assumimos um processo a mais para operar, deployar e monitorar, em troca de isolamento de falha entre API e entrega de eventos.

### 3. Múltiplos workers em paralelo desde o início

**Descartada para esta fase.** Com mais de um consumidor, a ordenação por `order_id` se perde ([09:12] Diego). Diego indicou os caminhos para o futuro — particionar por `order_id` ou usar lock pessimista — mas classificou como "problema do futuro, não agora" ([09:13] Diego), e Larissa fechou registrando como limitação conhecida ([09:13] Larissa). Marcos confirmou que os clientes nunca pediram ordering global ([09:14] Marcos).

**Trade-off do descarte:** limitamos o throughput ao de um único processo, em troca de manter a garantia de ordem por pedido sem nenhuma coordenação.

## Consequências

### Positivas

- Latência de captação previsível e limitada a 2 segundos, com folga de 5x sobre o requisito de 10s.
- Falha ou deploy da API não interrompe a entrega de webhooks, e vice-versa.
- Ordenação por `order_id` sai de graça, sem lock nem particionamento.
- Implementação simples: um loop com intervalo e uma query indexada. Sem broker, sem coordenação distribuída.

### Negativas

- O worker consulta o banco a cada 2 segundos mesmo quando não há nada a processar — carga constante, ainda que barata por conta do índice em status e `created_at` ([09:08] Diego).
- Single-worker é ponto único de falha: se o processo cair e ninguém perceber, os eventos se acumulam silenciosamente na outbox. Isso torna a métrica de idade do evento pendente mais importante que a de taxa de erro.
- Não há garantia de ordering global entre pedidos diferentes, e essa limitação precisa ser documentada para o cliente.
- Escalar horizontalmente no futuro exige trabalho adicional (particionamento ou lock), não é só subir mais réplicas.
- Mais um artefato de deploy: `npm run worker` precisa de supervisão própria.

### Trade-off explícito

Trocamos **reatividade e escalabilidade horizontal** por **simplicidade operacional e garantia de ordem**. A decisão é justificada pelo requisito: o cliente pediu "abaixo de 10 segundos" ([09:02] Marcos), não instantâneo, e 2 segundos de polling entregam isso com margem sobrando — não havia motivo para pagar o custo de uma solução reativa.
