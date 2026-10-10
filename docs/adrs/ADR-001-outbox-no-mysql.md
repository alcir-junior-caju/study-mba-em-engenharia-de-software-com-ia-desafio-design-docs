# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de pedido

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos) |
| **Fonte** | `TRANSCRICAO.md` [09:03] a [09:08], [09:40] a [09:41], [09:51] |
| **Relacionados** | [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md), [ADR-008](ADR-008-filtragem-de-eventos-na-insercao.md) |

## Contexto

Três clientes B2B pediram notificação em tempo real de mudança de status de pedidos; hoje eles fazem polling no `GET /orders`, o que é lento e caro para eles ([09:00] Marcos). A aplicação não possui nenhum mecanismo de eventos, fila ou webhook.

A mudança de status já existe como uma transação única em `src/modules/orders/order.service.ts` (método `changeStatus`, dentro de `this.prisma.$transaction`): ela valida a transição pela máquina de estados de `src/modules/orders/order.status.ts`, debita ou repõe estoque em `products`, atualiza `orders` e insere em `order_status_history`. Bruno descreveu essa transação como "pesada" e argumentou que um HTTP call no meio dela faria qualquer cliente lento travar mudança de status para outros pedidos ([09:04]). Além disso, não há como dar rollback na mudança de status se o cliente estiver fora do ar ([09:04] Bruno).

O time é pequeno e quer evitar subir infraestrutura nova ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox transacional no MySQL já existente**:

- Criar uma tabela `webhook_outbox` no mesmo banco (via Prisma). Quando o status de um pedido muda, a linha do evento é inserida **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history` ([09:06] Diego). Se a transação principal commita, o evento está registrado; se dá rollback, o evento some junto.
- A inserção na outbox é feita por uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` da transação corrente e é chamada por `OrderService.changeStatus` ([09:41] Bruno e Diego). Falha ao inserir na outbox provoca rollback da mudança de status: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).
- A tabela tem índice no campo de status do evento (pendente, processando, falhou, entregue) e em `created_at`, para o worker ler apenas os pendentes em lotes pequenos ([09:08] Diego).
- A chave primária da outbox é UUID, seguindo o padrão de todas as tabelas em `prisma/schema.prisma` ([09:51] Larissa).
- O consumo da tabela é feito por um worker separado, decidido no [ADR-002](ADR-002-worker-separado-em-polling.md).

## Alternativas Consideradas

### A. Disparo HTTP síncrono dentro de `changeStatus`

Chamar o endpoint do cliente diretamente no service de orders, no momento da mudança de status.

- Descartada por unanimidade ([09:04] Bruno, [09:04] Larissa, [09:06] Diego: "síncrono está fora de questão").
- Trade-off que motivou o descarte: a transação de status ficaria refém da latência e disponibilidade do cliente externo, e não há semântica de rollback possível quando o cliente está fora do ar.

### B. Fila/stream externo (Redis Streams ou similar)

Publicar o evento em Redis Streams e consumir de lá.

- Descartada ([09:07] Larissa e Diego).
- Trade-off que motivou o descarte: exigiria subir e operar infraestrutura nova (Redis Cluster) para um time pequeno; considerado overengineering. Além disso, publicar em um broker externo fora da transação SQL não daria a atomicidade que o outbox dá.

## Consequências

### Positivas

- **Atomicidade garantida**: evento e mudança de status commitam ou falham juntos; não existe estado em que o pedido mudou e o evento não foi registrado ([09:06] Diego, [09:41] Diego).
- **Zero infraestrutura nova**: reaproveita o MySQL e o Prisma já em uso ([09:07] Diego).
- **Evidência auditável**: a outbox é consultável para debug, assim como `order_status_history` já é hoje.

### Negativas / trade-offs assumidos

- **Latência mínima** passa a depender do intervalo de polling do worker (2 s no pior caso, ver [ADR-002](ADR-002-worker-separado-em-polling.md)), não é entrega instantânea ([09:10] Larissa).
- **Crescimento da tabela**: linhas entregues precisam ser arquivadas (sugestão: após 30 dias), mas o arquivamento foi declarado **fora do escopo** desta feature ([09:08] Diego).
- **Acoplamento do módulo de pedidos ao de webhooks**: `OrderService.changeStatus` passa a chamar `publishWebhookEvent`. O acoplamento foi minimizado ao escolher uma função que recebe o `tx` em vez de injetar um repository inteiro ([09:41] Diego).
- **Carga extra na transação de status**: uma inserção adicional (e a consulta de quais webhooks assinam o status, ver [ADR-008](ADR-008-filtragem-de-eventos-na-insercao.md)) dentro de uma transação que já é considerada pesada.
