# ADR-003 — Retry com backoff exponencial de 5 tentativas e DLQ em tabela separada

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Origem:** `TRANSCRICAO.md` [09:14]–[09:19], [09:36], [09:42]
- **Relacionada a:** [ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md), [ADR-005](./ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

Entrega de webhook depende de um endpoint que não controlamos. Larissa colocou a pergunta direta: se o cliente está offline, o que fazemos? ([09:14] Larissa).

O histórico do time informa a resposta: já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). Uma política agressiva demais mataria eventos legítimos nesse cenário.

Ao mesmo tempo, retry infinito cria o problema oposto: evento pendurado para sempre quando o cliente simplesmente sumiu ([09:15] Diego).

## Decisão

**Backoff exponencial, 5 tentativas, progressão 1m / 5m / 30m / 2h / 12h. Esgotadas as tentativas, o evento vai para uma tabela `webhook_dead_letter` separada.**

A progressão soma quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego), janela que Marcos considerou aceitável: "se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos). Larissa fechou a decisão em [09:17].

Cada tentativa usa **timeout HTTP de 10 segundos**; cliente que não responde nesse prazo é tratado como falha e entra no ciclo de retry ([09:42] Diego).

A DLQ é uma **tabela separada**, não uma flag na outbox, e guarda o payload, o motivo da falha e o timestamp ([09:18] Diego). O reprocessamento é **manual, via endpoint admin** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego; [09:35] Diego).

Esse endpoint **exige role `ADMIN`** e **registra em log quem fez o replay, para auditoria** ([09:36] Sofia; [09:36] Larissa). A verificação reaproveita o `requireRole` já existente em `src/middlewares/auth.middleware.ts` ([09:36] Larissa).

## Alternativas Consideradas

### 1. Três tentativas em vez de cinco

Bruno sugeriu 3, argumentando ser mais agressivo ([09:16] Bruno).

**Descartada.** Diego mostrou o cenário concreto: com 3 tentativas a janela total ficaria em torno de 30 minutos, e um cliente com indisponibilidade matinal teria os eventos mortos antes de voltar — algo que já aconteceu, com duas horas de manutenção planejada ([09:16] Diego).

**Trade-off do descarte:** eventos permanecem retentáveis por mais tempo, ocupando linhas na outbox e adiando a sinalização de falha permanente, em troca de tolerar indisponibilidades reais de clientes.

### 2. Retry indefinido com backoff

Mencionada por Diego como posição defendida por parte do mercado ([09:15] Diego).

**Descartada.** Cria eventos pendurados para sempre quando o cliente desaparece de vez, e transforma a outbox num cemitério que cresce sem limite.

**Trade-off do descarte:** aceitamos perder entregas de clientes com indisponibilidade acima de ~15h, em troca de um estado terminal explícito e de uma fila que não cresce indefinidamente.

### 3. DLQ como flag "failed" na própria outbox

Larissa colocou a opção na mesa ([09:17] Larissa).

**Descartada.** Diego preferiu tabela separada: mantém a leitura da outbox principal mais limpa e deixa a DLQ como evidência para debug e reprocessamento ([09:18] Diego).

**Trade-off do descarte:** pagamos uma tabela e uma escrita a mais no momento da falha permanente, em troca de a query quente do worker não precisar filtrar lixo acumulado.

## Consequências

### Positivas

- Indisponibilidades reais de cliente, inclusive manutenções planejadas de algumas horas, são absorvidas sem perda de evento.
- Estado terminal explícito: um evento ou foi entregue, ou está em retry, ou está na DLQ. Não existe limbo.
- A DLQ é uma tabela consultável, o que dá visibilidade imediata de quais clientes estão quebrados.
- A query quente do worker permanece enxuta, porque eventos mortos saíram da outbox.
- O replay manual dá caminho de recuperação sem intervenção direta em banco.

### Negativas

- Latência de recuperação alta no pior caso: um evento que falha na primeira tentativa e só é aceito na quinta chega ao cliente quase 15 horas depois do fato.
- Com 5 tentativas e backoff longo, um cliente quebrado mantém eventos ocupando a outbox por até ~15h, o que infla a tabela durante incidentes.
- Reprocessamento é manual: alguém precisa perceber a DLQ, decidir e chamar o endpoint. Não há alerta automático — o disparo de email para o cliente foi explicitamente adiado para uma fase futura ([09:37] Larissa).
- O endpoint de replay é uma porta privilegiada para reinjetar tráfego de saída; daí a exigência de `ADMIN` e do log de auditoria ([09:36] Sofia).

### Trade-off explícito

Trocamos **rapidez em declarar falha** por **tolerância a indisponibilidade de terceiros**. A janela de 15 horas foi calibrada não por teoria, mas por um incidente real do time (cliente com 2h de manutenção planejada, [09:16] Diego): a política precisa cobrir o caso conhecido com folga, e 3 tentativas não cobriam.
