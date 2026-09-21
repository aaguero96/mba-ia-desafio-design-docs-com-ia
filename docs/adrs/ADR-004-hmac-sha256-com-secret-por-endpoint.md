# ADR-004 — Autenticação das entregas via HMAC-SHA256 com secret por endpoint e rotação com grace period

- **Status:** Aceita
- **Data:** 2026-09-21
- **Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)
- **Origem:** `TRANSCRICAO.md` [09:19]–[09:23], [09:44]
- **Relacionada a:** [ADR-005](./ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](./ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

## Contexto

A feature expõe dados de pedidos para endpoints fora da nossa infraestrutura. Sofia colocou o problema em duas partes ([09:19] Sofia):

1. O cliente precisa conseguir **validar que a requisição veio realmente de nós** — caso contrário qualquer um que descubra a URL do webhook pode injetar eventos falsos.
2. O cliente precisa conseguir verificar que **ninguém adulterou o payload no meio do caminho**.

Há também um risco operacional com precedente no time: já houve cliente que vazou a secret em log de aplicação dele ([09:22] Diego). Ou seja, a secret vai vazar algum dia, e o desenho precisa sobreviver a isso.

## Decisão

**HMAC-SHA256 calculado sobre o corpo do request, com secret única por endpoint de webhook e suporte a rotação com grace period de 24 horas** ([09:22] Sofia).

Detalhamento:

- **Algoritmo:** HMAC-SHA256. Sofia justificou como padrão de mercado, com biblioteca disponível em qualquer cliente sério ([09:20] Sofia).
- **Transporte:** a assinatura vai no header `X-Signature` ([09:20] Sofia). Acompanham `X-Timestamp` com o timestamp do envio, para que o cliente possa detectar replay attack se quiser ([09:44] Diego), e `X-Webhook-Id` com o id do cadastro, para que um cliente com vários endpoints saiba qual caiu naquele envio ([09:44] Sofia).
- **Escopo da secret:** uma secret por endpoint cadastrado, nunca uma secret global da plataforma ([09:21] Sofia). A tabela de configuração guarda url, secret, `customer_id` e estado ativo ([09:21] Bruno; [09:21] Sofia).
- **Geração:** a secret é gerada por nós e devolvida ao cliente no momento da criação do webhook ([09:31] Marcos).
- **Rotação:** o cliente pede nova secret pela API. Durante 24 horas a antiga continua válida em paralelo, para ele ter tempo de migrar os sistemas dele. Passado o grace period, a antiga morre ([09:21] Sofia).
- **Transporte obrigatoriamente cifrado:** a URL do webhook tem que ser `https`. Cadastro com `http` é recusado com erro de validação. Sofia classificou isso não como decisão arquitetural, mas como validação de schema Zod ([09:23] Sofia) — e de fato cabe no `validate.middleware.ts` que já existe.

## Alternativas Consideradas

### 1. Secret global da plataforma

Uma única secret compartilhada com todos os clientes.

**Descartada.** Sofia foi direta: "se vaza uma, vaza tudo" ([09:21] Sofia). Dado o precedente de cliente vazando secret em log ([09:22] Diego), o raio de explosão de uma secret global seria a base inteira de clientes.

**Trade-off do descarte:** passamos a ter N secrets para gerar, armazenar, rotacionar e proteger, em vez de uma, em troca de isolar o comprometimento a um único endpoint.

### 2. Rotação imediata, sem grace period

Trocar a secret e invalidar a anterior no mesmo instante.

**Descartada implicitamente pela decisão de Sofia** ([09:21] Sofia). Rotação instantânea significa janela de entregas rejeitadas entre o momento em que geramos a nova secret e o momento em que o cliente terminou o deploy dele. Como não controlamos o processo de deploy do cliente, essa janela é imprevisível.

**Trade-off do descarte:** durante 24 horas existem duas secrets válidas para o mesmo endpoint, o que enfraquece marginalmente a postura de segurança, em troca de rotação sem downtime de entrega — o que, na prática, é o que faz o cliente realmente rotacionar em vez de adiar para sempre.

### 3. mTLS ou assinatura assimétrica

Não foi levantada na reunião. Registrada aqui como alternativa plausível: garantiria não-repúdio e dispensaria segredo compartilhado.

**Descartada.** Exigiria gestão de certificados ou de pares de chaves dos dois lados, muito acima do custo de integração que clientes B2B aceitam para receber notificação. HMAC compartilhado é o que Stripe e GitHub usam e o que o cliente já sabe implementar.

**Trade-off do descarte:** abrimos mão de não-repúdio criptográfico em troca de uma barreira de integração baixa o suficiente para os clientes realmente adotarem.

## Consequências

### Positivas

- O cliente consegue autenticar origem e verificar integridade do payload com ~5 linhas de código e biblioteca padrão.
- Comprometimento de uma secret afeta um único endpoint, não a plataforma.
- A rotação é auto-serviço via API, sem ticket e sem intervenção nossa.
- O grace period de 24h remove a desculpa operacional para não rotacionar.
- `X-Timestamp` dá ao cliente a opção de implementar proteção contra replay, sem nos obrigar a manter estado para isso.

### Negativas

- Passamos a armazenar segredos de terceiros em nossa base, o que torna a tabela de configuração de webhooks um alvo de alto valor.
- Durante 24 horas coexistem duas secrets válidas por endpoint; o modelo de dados e a verificação precisam suportar isso, e a secret antiga precisa ser efetivamente expurgada depois.
- O cálculo do HMAC precisa ser feito sobre **exatamente** os bytes enviados no corpo. Qualquer reserialização entre assinar e enviar quebra a verificação do cliente — é a fonte clássica de bug nesse tipo de implementação.
- **Risco concreto identificado no código:** o redact do Pino em `src/shared/logger/index.ts` cobre `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas **não cobre `*.secret`**. Sem adicionar esse path, a secret do cliente pode vazar nos nossos próprios logs — exatamente o incidente que Diego relatou do lado do cliente ([09:22] Diego).
- Sofia pediu ao menos dois dias úteis de revisão de segurança sobre HMAC e geração de secret antes do deploy ([09:46] Sofia), o que é uma dependência de cronograma real.

### Trade-off explícito

Trocamos **não-repúdio e ausência de segredo compartilhado** (que mTLS daria) por **facilidade de adoção pelo cliente**. E trocamos **invalidação imediata na rotação** por **uma janela de 24h com duas secrets válidas**, porque uma rotação que causa downtime de entrega é uma rotação que o cliente nunca faz.
