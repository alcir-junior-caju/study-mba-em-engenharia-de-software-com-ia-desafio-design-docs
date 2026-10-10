# ADR-008 — Filtragem por status assinado feita na inserção da outbox, não no envio

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Marcos (PM) |
| **Fonte** | `TRANSCRICAO.md` [09:31] a [09:34] |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) |

## Contexto

Cada webhook cadastrado tem uma **lista de status** que o cliente quer receber, por exemplo "só quero saber quando vira `SHIPPED` e `DELIVERED`" ([09:31] Marcos, [09:33] Bruno e Marcos). Um cliente pode ter vários webhooks com filtros diferentes ([09:44] Sofia). Os status possíveis são os do enum `OrderStatus` em `prisma/schema.prisma` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), e as transições válidas estão em `src/modules/orders/order.status.ts`.

Diego perguntou onde aplicar o filtro: na inserção da outbox ou na hora de mandar ([09:34]).

## Decisão

- O filtro é aplicado **na inserção**: ao mudar o status de um pedido, `publishWebhookEvent` consulta os webhooks ativos do `customer_id` do pedido e insere uma linha na outbox **apenas para os webhooks que assinam o `to_status`**. Se nenhum webhook do customer quer aquele status, **nada é inserido** ([09:34] Bruno, [09:34] Diego).
- Consequentemente, a linha da outbox é por **(evento, webhook)**: cada entrega pendente já nasce associada ao endpoint de destino, o que casa com o header `X-Webhook-Id` ([09:44] Sofia) e com a secret por endpoint ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)).

## Alternativas Consideradas

### A. Inserir sempre e filtrar no envio

Gravar o evento para todo cliente e deixar o worker decidir, no momento do envio, para quais webhooks mandar ([09:34] Diego levantou a opção).

- Descartada ([09:34] Bruno e Diego).
- Trade-off que motivou o descarte: gasta linhas na tabela com eventos que ninguém assina, e o worker faria a consulta de configuração no caminho quente a cada 2 s; a economia de linhas foi o argumento decisivo.

## Consequências

### Positivas

- **Outbox menor**: só existem linhas que de fato serão entregues ([09:34] Bruno).
- **Worker sem lógica de roteamento**: cada linha já sabe seu destino, URL e secret; o worker apenas assina e envia.
- **Semântica clara de "entrega"**: o histórico `GET /webhooks/:id/deliveries` ([09:34] Marcos) mapeia 1:1 com linhas da outbox/DLQ daquele webhook.

### Negativas / trade-offs assumidos

- **Alterar o filtro de um webhook não é retroativo**: eventos já enfileirados continuam seguindo o filtro vigente na inserção; eventos que ocorreram enquanto o status não era assinado nunca serão entregues (nem via replay).
- **Consulta adicional dentro da transação de status** para carregar os webhooks ativos do customer, somando-se ao custo já discutido em [ADR-001](ADR-001-outbox-no-mysql.md) e [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md).
- **Mais linhas quando um customer tem muitos webhooks** para o mesmo status: o snapshot é duplicado por destino. Aceitável para os três clientes atuais; a pergunta sobre rate limiting de saída permanece como ponto em aberto ([09:38] a [09:39] Diego e Larissa).
