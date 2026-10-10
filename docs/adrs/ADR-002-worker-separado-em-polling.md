# ADR-002 — Worker em processo separado, consumindo a outbox por polling de 2 segundos

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM) |
| **Fonte** | `TRANSCRICAO.md` [09:02], [09:08] a [09:14], [09:28] a [09:30] |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Contexto

Com o outbox decidido ([ADR-001](ADR-001-outbox-no-mysql.md)), é preciso definir **quem** lê a tabela `webhook_outbox` e **como** é notificado de novos eventos. O requisito de latência do cliente é "abaixo de 10 segundos" ([09:02] Marcos).

Hoje a aplicação tem um único entry-point, `src/server.ts`, que sobe o Express via `buildApp({ prisma })` e trata `SIGINT`/`SIGTERM` com `prisma.$disconnect()`. O `PrismaClient` é criado em `src/config/database.ts` pela função `createPrismaClient()`. Os scripts de execução estão em `package.json` (`dev`, `start`, etc.).

MySQL não tem mecanismo nativo de notificação a processos externos como o `LISTEN/NOTIFY` do Postgres; triggers só executam SQL ([09:09] Diego).

## Decisão

- O consumo da outbox é feito por um **worker em polling**: a cada **2 segundos**, busca os eventos pendentes mais antigos (ordem de `created_at`), em batch pequeno, processa e marca como entregue ([09:09] Diego, [09:10] Larissa).
- O worker roda como **processo separado** da API, nunca dentro da mesma instância do Express, para que um restart da API não derrube o worker ([09:11] Diego).
- Implementação: novo entry-point `src/worker.ts`, espelhando `src/server.ts`, com script `npm run worker` em `package.json` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
- O worker conecta no **mesmo banco e mesma `DATABASE_URL`**, mas com **instância própria de `PrismaClient`**, pois `PrismaClient` é por processo ([09:11] Bruno, [09:30] Bruno).
- **Single-worker** nesta fase. A ordem de entrega é garantida apenas por `order_id` e apenas enquanto houver um único worker; não há garantia de ordering global ([09:12] a [09:13] Diego e Larissa). Os clientes não pediram ordering global ([09:14] Marcos).

## Alternativas Consideradas

### A. Trigger no banco para acordar o worker

Usar trigger MySQL para ser "mais reativo" ([09:09] Bruno).

- Descartada ([09:09] Diego).
- Trade-off que motivou o descarte: trigger MySQL só executa SQL e não notifica processo externo; seria preciso improvisar (escrever arquivo, bater em endpoint), o que "fica esquisito". Polling de 2 s já atende o requisito de < 10 s com folga.

### B. Worker dentro do mesmo processo da API

Rodar o loop de polling no mesmo processo Node de `src/server.ts`.

- Descartada ([09:11] Diego).
- Trade-off que motivou o descarte: um restart ou crash da API perderia o worker; acopla o ciclo de vida das entregas ao ciclo de vida do HTTP server.

### C. Múltiplos workers em paralelo

Escalar horizontalmente o consumo desde já.

- Adiada ([09:13] Diego: "problema do futuro, não agora").
- Trade-off: perderia a garantia de ordering por `order_id`; exigiria particionamento por `order_id` ou lock pessimista.

## Consequências

### Positivas

- **Latência previsível**: no pior caso ~2 s entre o commit da transação e o início da entrega, bem abaixo dos 10 s pedidos ([09:10] Marcos e Larissa).
- **Isolamento de falhas**: cliente lento ou fora do ar afeta só o worker, nunca a API de pedidos.
- **Stack única**: mesmo código, mesmo Prisma, mesmo logger Pino ([09:11] Diego, [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)).

### Negativas / trade-offs assumidos

- **Ordering apenas por `order_id` e só com single-worker**, documentado como **limitação conhecida** ([09:13] Larissa).
- **Polling gera consultas periódicas ao banco** mesmo sem eventos pendentes; mitigado pelos índices em status e `created_at` ([09:08] Diego).
- **Novo processo para operar**: deploy, monitoramento e restart do worker precisam ser cuidados separadamente da API.
- **Escala horizontal não é trivial**: exigirá rever a decisão (particionamento ou lock) quando um único worker não for suficiente.
