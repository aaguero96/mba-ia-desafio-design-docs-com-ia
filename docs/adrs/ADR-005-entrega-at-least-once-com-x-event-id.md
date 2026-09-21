# ADR-005 — Garantia de entrega at-least-once com idempotência delegada via `X-Event-Id`

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Diego (Eng. Plataforma), Larissa (Tech Lead), Sofia (Eng. Segurança), Marcos (PM)
- **Origem:** `TRANSCRICAO.md` [09:24]–[09:26], [09:44]
- **Relacionada a:** [ADR-001](./ADR-001-outbox-transacional-no-mysql.md), [ADR-003](./ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

A combinação de outbox ([ADR-001](./ADR-001-outbox-transacional-no-mysql.md)) com retry ([ADR-003](./ADR-003-retry-com-backoff-exponencial-e-dlq.md)) produz duplicatas por construção. O caso clássico: o cliente processa a requisição com sucesso mas a resposta se perde, o worker interpreta como falha e reenvia. Do nosso lado é impossível distinguir "o cliente não recebeu" de "o cliente recebeu e a resposta não voltou".

Diego trouxe isso explicitamente para a mesa antes de alguém tropeçar no problema em produção: vamos garantir at-least-once, pode acontecer de o cliente receber o mesmo evento duas vezes, e ele tem que estar preparado ([09:24] Diego).

Bruno então fez a pergunta prática: como o cliente diferencia? ([09:25] Bruno).

## Decisão

**Garantia de entrega at-least-once, com deduplicação delegada ao cliente através do header `X-Event-Id`.**

Um UUID é gerado no momento em que o evento entra na outbox e é único por evento ([09:25] Diego). Esse UUID:

- é a chave primária da linha na `webhook_outbox` ([09:51] Larissa — UUID, seguindo o padrão do projeto);
- viaja no header `X-Event-Id` de toda tentativa de entrega ([09:25] Diego; [09:44] Diego);
- **permanece o mesmo em todas as retentativas do mesmo evento**, que é o que torna a deduplicação do lado do cliente possível.

Se o cliente recebeu duas vezes, ele dedupica pelo `event_id` do lado dele ([09:25] Diego).

Sofia registrou a objeção de que isso joga responsabilidade para o cliente ([09:25] Sofia). Diego respondeu que é o padrão de mercado — Stripe faz assim, GitHub faz assim — e que garantir exactly-once exigiria coordenação dos dois lados, ficando muito mais complexo; at-least-once com event_id resolve 99% dos casos ([09:25] Diego). Marcos se comprometeu a documentar isso em destaque no portal do desenvolvedor ([09:26] Marcos). Larissa fechou a decisão ([09:26] Larissa).

## Alternativas Consideradas

### 1. Exactly-once

Garantir que o cliente receba cada evento exatamente uma vez.

**Descartada.** Exigiria coordenação entre os dois lados — algum protocolo de confirmação com estado compartilhado — e ficaria muito mais complexo ([09:25] Diego). Na prática, exactly-once entre dois sistemas que só trocam HTTP é inatingível sem que o receptor também implemente idempotência; ou seja, a complexidade adicional não elimina o requisito do lado do cliente, apenas o esconde.

**Trade-off do descarte:** o cliente precisa implementar deduplicação, em troca de um protocolo simples, sem estado de coordenação e sem acoplamento entre as duas pontas.

### 2. Deduplicação do nosso lado, por `(webhook_id, order_id, to_status)`

Não levantada na reunião; registrada como alternativa plausível. Consistiria em suprimir reenvios que já tiveram alguma resposta.

**Descartada.** Não resolve o caso que origina o problema: quando a resposta se perde, não sabemos se o cliente processou. Suprimir o reenvio nessa situação transformaria at-least-once em at-most-once, trocando duplicata (que o cliente sabe tratar) por perda de evento (que ele não tem como detectar).

**Trade-off do descarte:** aceitamos duplicatas visíveis em troca de nunca perder evento silenciosamente.

## Consequências

### Positivas

- Protocolo simples e sem estado de coordenação: cada tentativa é autocontida.
- Alinhado ao padrão que os clientes já conhecem de Stripe e GitHub ([09:25] Diego), o que reduz atrito de integração.
- Combina naturalmente com o retry: reenviar é sempre seguro, porque o `X-Event-Id` é estável entre tentativas.
- Nenhum evento é perdido silenciosamente — na dúvida, reenviamos.

### Negativas

- **Transfere trabalho para o cliente.** É uma dívida de integração real, e Sofia registrou a ressalva em ata ([09:25] Sofia). Um cliente que ignorar o `X-Event-Id` vai processar pedidos duplicados.
- Cria uma obrigação de documentação: se o portal do desenvolvedor não deixar isso explícito, o bug aparece do lado do cliente e a percepção de culpa é nossa. Marcos assumiu essa tarefa ([09:26] Marcos).
- O `X-Event-Id` precisa ser rigorosamente estável entre retentativas. Regerar o UUID no replay de DLQ, por exemplo, quebraria a deduplicação — e o endpoint de replay recoloca o evento na outbox ([09:18] Diego), então esse é um ponto de atenção concreto na implementação.
- Métricas de entrega passam a contar tentativas, não eventos distintos recebidos pelo cliente; relatórios precisam ser lidos com esse cuidado.

### Trade-off explícito

Trocamos **garantia de unicidade** por **garantia de não-perda**. Entre os dois modos de falha possíveis, o time escolheu conscientemente o que é detectável e tratável pelo receptor (duplicata identificada por `X-Event-Id`) em detrimento do que é silencioso e irrecuperável (evento perdido).
