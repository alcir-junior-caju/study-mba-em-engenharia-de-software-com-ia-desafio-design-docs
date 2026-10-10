# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Em revisão |
| **Data** | 2026-10-10 |
| **Autora** | Larissa (Tech Lead) |
| **Revisores técnicos** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

Este documento descreve **como construir** a feature. As decisões arquiteturais estão fechadas nos ADRs; aqui elas viram modelo de dados, fluxos, contratos HTTP, erros e critérios de aceite. Onde a reunião deixou lacunas de implementação, a escolha foi feita seguindo o padrão já existente no código e está listada na seção [15. Decisões de implementação tomadas neste FDD](#15-decisões-de-implementação-tomadas-neste-fdd).

---

## 1. Contexto e motivação técnica

Clientes B2B hoje descobrem mudanças de status fazendo polling em `GET /api/v1/orders`, o que é lento e caro para eles ([09:00] Marcos). A aplicação não possui nenhum mecanismo de eventos, fila ou notificação externa.

A única fonte de verdade de mudança de status é o método `changeStatus` de `src/modules/orders/order.service.ts`, que roda inteiro dentro de `this.prisma.$transaction`: carrega o pedido, valida a transição com `canTransition` (`src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders` e insere em `order_status_history`. Esse é o único ponto onde um evento de status pode nascer com garantia de consistência ([09:40] Bruno).

A solução adotada é o padrão Outbox no MySQL com worker em processo separado ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

1. Registrar o evento de mudança de status **na mesma transação** que muda o status; falha ao registrar provoca rollback ([09:40] Bruno, [09:41] Diego).
2. Entregar o evento ao cliente com latência de pior caso em torno de 2 s após o commit, bem abaixo dos 10 s exigidos ([09:02] Marcos, [09:10] Larissa).
3. Tolerar indisponibilidade do cliente por até ~15 h com retry em backoff, e preservar eventos não entregues em DLQ auditável ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).
4. Permitir ao cliente verificar autenticidade e integridade de cada entrega (HMAC-SHA256) e deduplicar entregas repetidas (`X-Event-Id`) ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
5. Não introduzir dependência nova nem alterar middlewares existentes ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).
6. Não degradar perceptivelmente a latência de `PATCH /orders/:id/status`.

## 3. Escopo e exclusões

### Incluído

- Módulo `src/modules/webhooks/` com CRUD de endpoints de webhook, histórico de entregas e rotação de secret.
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter` e migration Prisma.
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada por `OrderService.changeStatus`.
- Worker `src/worker.ts` com polling de 2 s, retry em backoff, DLQ.
- Endpoint administrativo de replay de DLQ (role `ADMIN`).
- Assinatura HMAC-SHA256, headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.
- Validação de URL `https` e limite de 64 KB por payload.
- Logs estruturados e eventos de observabilidade no Pino existente.

### Excluído (decidido na reunião)

| Item | Decisão | Fonte |
| --- | --- | --- |
| E-mail de alerta ao cliente quando o webhook falha | Próxima fase | [09:37] Larissa |
| Rate limiting de saída por cliente | Observar e decidir depois | [09:39] Diego e Larissa |
| Dashboard visual para o cliente | Projeto separado do time de frontend | [09:40] Larissa |
| Arquivamento de linhas entregues da outbox | Fora do escopo desta feature | [09:08] Diego |
| Múltiplos workers / ordering global | Problema do futuro | [09:13] Diego |
| Webhooks inbound (cliente → plataforma) | Apenas outbound | [09:02] Marcos |
| Endurecimento de permissões do CRUD | Qualquer role autenticada por enquanto | [09:37] Sofia |
| Envio de `items` no payload | Payload enxuto; cliente consulta `GET /orders/:id` | [09:43] Diego |

## 4. Integração com o sistema existente

Esta seção nomeia cada arquivo real do código base que será alterado ou reaproveitado e descreve como.

| Arquivo existente | Tipo de integração | O que acontece |
| --- | --- | --- |
| `src/modules/orders/order.service.ts` | **Alteração** | `changeStatus` passa a chamar `publishWebhookEvent(tx, order, from, to, { correlationId })` após `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, dentro do mesmo `$transaction`. Qualquer exceção lançada pela função aborta a transação (comportamento padrão do `$transaction` interativo do Prisma), cumprindo "se a outbox falhar de inserir, rollback" ([09:40] Bruno). O `order` carregado no início do método já contém `customerId`, `orderNumber`, `subtotalCents`, `discountCents` e `totalCents`, suficientes para o snapshot. |
| `src/modules/orders/order.status.ts` | **Reuso** | O enum `OrderStatus` e as transições em `transitions` definem o universo de `to_status` possíveis. Como nenhuma transição tem `PENDING` como destino, assinar `PENDING` nunca gera evento: a criação do pedido não é mudança de status. Documentar isso no portal. |
| `src/modules/orders/order.controller.ts` | **Sem alteração** | Erros de `publishWebhookEvent` (ex.: `WEBHOOK_PAYLOAD_TOO_LARGE`) sobem por `next(err)` como qualquer `AppError` lançado pelo service. |
| `src/shared/errors/app-error.ts` | **Reuso** | Todas as novas classes de erro estendem `AppError` (`statusCode`, `errorCode`, `details`). |
| `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | **Alteração (adição)** | Novas classes `WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookPayloadTooLargeError`, `WebhookInactiveError`, `WebhookDeadLetterNotFoundError`, `WebhookAlreadyReplayedError`, `WebhookCustomerNotFoundError`, seguindo o padrão de `InvalidStatusTransitionError` e `InsufficientStockError` (classe por erro de domínio, código em `SCREAMING_SNAKE_CASE`). Como `NotFoundError` fixa o código `NOT_FOUND` no construtor, os erros 404 do módulo estendem `AppError` diretamente para poder usar `WEBHOOK_NOT_FOUND`. |
| `src/middlewares/error.middleware.ts` | **Sem alteração** | Já serializa `AppError` como `{ error: { code, message, details } }`, `ZodError` como `VALIDATION_ERROR` e `PrismaClientKnownRequestError` (`P2002`, `P2025`). Cobre todos os erros do módulo ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | **Reuso** | `authenticate` em todas as rotas do módulo; `requireRole('ADMIN')` na rota de replay ([09:36] Larissa). |
| `src/middlewares/validate.middleware.ts` | **Reuso** | `validate({ body, params, query })` com os schemas de `webhook.schemas.ts`. |
| `src/shared/logger/index.ts` | **Alteração (adição)** | Acrescentar `'*.secret'` e `'*.previousSecret'` em `redactPaths`, para que nenhum log de endpoint exponha secret ([09:22] Diego). O worker usa o mesmo `logger`. |
| `src/middlewares/request-logger.middleware.ts` | **Reuso** | O `req.id` (`X-Request-Id`) do `PATCH /orders/:id/status` é gravado como `correlationId` na linha da outbox e aparece nos logs do worker, ligando entrega à requisição de origem. |
| `src/shared/http/response.ts` | **Reuso** | `paginated()` para `GET /webhooks` e `GET /webhooks/:id/deliveries`, no mesmo formato de `GET /orders`. |
| `src/app.ts` | **Alteração** | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` e `WebhookAdminController`; `Controllers` ganha as chaves `webhooks` e `webhooksAdmin`. |
| `src/routes/index.ts` | **Alteração** | `buildApiRouter` registra `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`, ambos sob o prefixo `/api/v1` definido em `src/app.ts`. |
| `src/server.ts` | **Modelo para o novo `src/worker.ts`** | O worker copia o esqueleto de `bootstrap()`: `logger.info` de start, handlers `SIGINT`/`SIGTERM` que param o loop, aguardam o batch corrente e chamam `prisma.$disconnect()`. |
| `src/config/database.ts` | **Reuso** | O worker importa `prisma` do mesmo módulo. Por ser outro processo Node, obtém sua própria instância de `PrismaClient` com a mesma `DATABASE_URL` ([09:30] Bruno). |
| `src/config/env.ts` | **Alteração (adição)** | Novas variáveis opcionais com default: `WEBHOOK_POLL_INTERVAL_MS=2000`, `WEBHOOK_BATCH_SIZE=50`, `WEBHOOK_HTTP_TIMEOUT_MS=10000`, `WEBHOOK_MAX_PAYLOAD_BYTES=65536`. Os valores da reunião viram defaults; nada é configurável sem default. |
| `prisma/schema.prisma` | **Alteração** | Novos modelos e enum (seção 5). Migration via `npm run db:migrate`. |
| `package.json` | **Alteração** | Script `"worker": "tsx watch --env-file=.env src/worker.ts"` e `"worker:start": "node --env-file=.env dist/worker.js"`, espelhando `dev` e `start`. |
| `tests/setup.ts` | **Alteração** | O `beforeEach` precisa incluir `webhookDelivery`, `webhookOutbox`, `webhookDeadLetter` e `webhookEndpoint` em `deleteMany`, antes de `customer`. |

### Estrutura do novo módulo

```
src/modules/webhooks/
├── webhook.schemas.ts        # Zod: create, update, params, queries
├── webhook.repository.ts     # acesso a webhook_endpoints, deliveries, dead_letter
├── webhook.service.ts        # CRUD, rotação de secret, replay
├── webhook.controller.ts     # handlers HTTP (CRUD + deliveries)
├── webhook-admin.controller.ts
├── webhook.routes.ts         # /webhooks
├── webhook-admin.routes.ts   # /admin/webhooks
├── webhook.publisher.ts      # publishWebhookEvent(tx, ...) — chamado por order.service
├── webhook.signature.ts      # HMAC-SHA256, geração de secret
└── webhook.processor.ts      # loop do worker: poll, deliver, retry, DLQ
src/worker.ts                 # entry-point do processo do worker
```

## 5. Modelo de dados

Novos modelos em `prisma/schema.prisma`, seguindo as convenções do arquivo: UUID `@db.Char(36)` ([09:51] Larissa), `@@map` em snake_case, `Json` para estruturas (como `Customer.address`).

```prisma
enum WebhookOutboxStatus {
  PENDING      // aguardando primeiro envio
  PROCESSING   // reservado pelo worker no ciclo corrente
  FAILED       // último envio falhou; aguarda nextAttemptAt
  DELIVERED    // cliente respondeu 2xx
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[] assinados
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36) // = X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)  // sem FK: snapshot é autocontido
  eventType     String              @db.VarChar(64) // "order.status_changed"
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(1000)
  correlationId String?             @db.Char(36)
  createdAt     DateTime            @default(now())
  deliveredAt   DateTime?

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([webhookId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.VarChar(2048) // truncado
  error          String?  @db.VarChar(1000)
  durationMs     Int
  attemptedAt    DateTime @default(now())

  @@index([webhookId, attemptedAt])
  @@index([eventId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @db.Char(36) // mesmo id da linha da outbox (preserva X-Event-Id)
  webhookId    String    @db.Char(36)
  orderId      String    @db.Char(36)
  eventType    String    @db.VarChar(64)
  payload      Json
  reason       String    @db.VarChar(1000)
  attempts     Int
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  @@index([webhookId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:

- Uma linha de outbox por **(evento, webhook assinante)** ([ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao.md)); o `id` da linha é o `X-Event-Id` enviado ao cliente.
- `orderId` **não tem FK** porque `OrderService.delete` permite apagar pedidos em `PENDING` ou `CANCELLED`, e o evento de cancelamento ainda precisa ser entregue após a exclusão. O snapshot já contém tudo que o envio precisa ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)).
- Índices por `status` e `created_at` seguem a orientação de [09:08] Diego; o índice composto `(status, nextAttemptAt)` é a consulta exata do worker.

## 6. Fluxos detalhados

### 6.1 Criação do evento na outbox

Chamador: `OrderService.changeStatus`, dentro de `$transaction`.

```ts
// src/modules/webhooks/webhook.publisher.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Pick<Order, 'id' | 'orderNumber' | 'customerId' | 'subtotalCents' | 'discountCents' | 'totalCents'>,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
  opts: { correlationId?: string; occurredAt?: Date } = {},
): Promise<string[]> /* ids inseridos */
```

Passos:

1. `endpoints = tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`.
2. `targets = endpoints.filter(e => (e.events as OrderStatus[]).includes(toStatus))`. Se vazio, **retorna `[]` sem inserir nada** ([09:34] Bruno).
3. Para cada target: gera `eventId = randomUUID()`, monta o payload (seção 7.6) com `event_id = eventId` e `timestamp = occurredAt ?? now` (instante da mudança de status).
4. Verifica `Buffer.byteLength(JSON.stringify(payload), 'utf8') <= WEBHOOK_MAX_PAYLOAD_BYTES` (64 KB). Se exceder, lança `WebhookPayloadTooLargeError` (`WEBHOOK_PAYLOAD_TOO_LARGE`), o que aborta a transação de status ([09:24] Larissa, [09:40] Bruno).
5. `tx.webhookOutbox.createMany({ data: rows })` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`, `correlationId`.
6. `logger.info({ orderId, toStatus, count, correlationId }, 'webhook_outbox_enqueued')`.

Sequência:

```
Controller ─► OrderService.changeStatus ─► $transaction(tx)
                 │  findUnique(order)
                 │  canTransition / debitStock / replenishStock
                 │  tx.order.update
                 │  tx.orderStatusHistory.create
                 │  publishWebhookEvent(tx, order, from, to) ──► findMany(endpoints) → filter → createMany(outbox)
                 │        └─ throw → rollback de tudo
                 │  findUnique(refreshed)
                 └─ commit
```

### 6.2 Processamento pelo worker

Entry-point `src/worker.ts` chama `runWebhookProcessor({ prisma, logger, signal })`. Loop:

```
loop a cada WEBHOOK_POLL_INTERVAL_MS (2000):
  1. recuperar linhas presas:
       updateMany({ where: { status: PROCESSING, updatedAt < now - 60s }, data: { status: PENDING } })
  2. batch = findMany({
       where: { status: { in: [PENDING, FAILED] }, nextAttemptAt: { lte: now } },
       orderBy: { createdAt: 'asc' },
       take: WEBHOOK_BATCH_SIZE (50),
     })
  3. para cada linha, em sequência (preserva ordem por created_at):
       reservado = updateMany({ where: { id, status: linha.status }, data: { status: PROCESSING } })
       se reservado.count === 0 → pular (já tratado)
       deliver(linha)
  4. logger.debug({ batch: n, pendingLag }, 'webhook_poll_cycle')
  5. aguardar intervalo (ou sair se signal abortado)
```

`deliver(linha)`:

1. Carrega o `WebhookEndpoint`. Se inativo ou inexistente, marca a linha como `FAILED` com `lastError = WEBHOOK_INACTIVE` e `nextAttemptAt` conforme backoff (um endpoint reativado volta a receber).
2. `body = JSON.stringify(linha.payload)`; `timestamp = new Date().toISOString()`.
3. Assina: `X-Signature` = HMAC-SHA256(secret, body) em hex, prefixado `sha256=`. Se `previousSecret` existe e `previousSecretExpiresAt > now`, o header carrega **duas** assinaturas separadas por vírgula (`sha256=<nova>,sha256=<antiga>`); o cliente aceita se qualquer uma bater.
4. `fetch(url, { method: 'POST', headers, body, signal: AbortSignal.timeout(WEBHOOK_HTTP_TIMEOUT_MS) })`. `fetch` nativo do Node 20 (`engines.node >= 20` em `package.json`), sem dependência nova.
5. Registra `WebhookDelivery` com `attempt = attempts + 1`, `success = res.ok`, `responseStatus`, `responseBody` truncado em 2 048 bytes, `durationMs`.
6. Sucesso (2xx): `status = DELIVERED`, `deliveredAt = now`, `logger.info(..., 'webhook_delivery_succeeded')`.
7. Falha (não-2xx, timeout, erro de rede): segue para 6.3.

### 6.3 Retry

Progressão decidida: 1 min, 5 min, 30 min, 2 h, 12 h ([09:17] Diego e Larissa).

```ts
const BACKOFF_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000];
```

Após uma falha: `attempts += 1`.

| `attempts` após a falha | Ação | `nextAttemptAt` |
| --- | --- | --- |
| 1 | `status = FAILED` | now + 1 min |
| 2 | `status = FAILED` | now + 5 min |
| 3 | `status = FAILED` | now + 30 min |
| 4 | `status = FAILED` | now + 2 h |
| 5 | `status = FAILED` | now + 12 h |
| 6 | mover para DLQ (6.4) | — |

Ou seja: **1 envio imediato + 5 retentativas**, com ~14,6 h entre a primeira falha e o último envio, o que bate com "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). `lastError` recebe o código interno (`WEBHOOK_DELIVERY_TIMEOUT`, `WEBHOOK_DELIVERY_REJECTED`, `WEBHOOK_DELIVERY_NETWORK_ERROR`) e um resumo. Toda tentativa reenvia **o mesmo `X-Event-Id` e o mesmo body** ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)).

### 6.4 DLQ e replay

Mover para DLQ, em uma transação:

1. `tx.webhookDeadLetter.create({ id: linha.id, webhookId, orderId, eventType, payload, reason: 'WEBHOOK_MAX_ATTEMPTS_EXCEEDED: ' + lastError, attempts: 6 })`.
2. `tx.webhookOutbox.delete({ where: { id } })` — a outbox fica limpa ([09:18] Diego).
3. `logger.warn({ eventId, webhookId, attempts }, 'webhook_dead_lettered')`.

Replay (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, role `ADMIN`):

1. Carrega a DLQ; 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` se não existe; 409 `WEBHOOK_ALREADY_REPLAYED` se `replayedAt` preenchido.
2. Verifica que o endpoint está ativo; 409 `WEBHOOK_INACTIVE` caso contrário.
3. Em transação: `webhookOutbox.create` com **o mesmo `id`**, `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`; `webhookDeadLetter.update({ replayedAt: now, replayedById: req.user.id })`.
4. `logger.info({ eventId, webhookId, replayedById: req.user.id }, 'webhook_replayed')` — trilha de auditoria exigida em [09:36] Sofia.
5. Responde `202 Accepted` com a linha da outbox recriada.

### 6.5 Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret`:

1. Gera `newSecret = randomBytes(32).toString('hex')` (`node:crypto`, sem dependência nova).
2. `previousSecret = secret`, `previousSecretExpiresAt = now + 24 h`, `secret = newSecret` ([09:21] Sofia).
3. Responde com a nova secret **uma única vez**; nenhum `GET` devolve secret.
4. O worker assina com as duas enquanto o grace vigora (6.2 passo 3). Após expirar, só a nova é usada; `previousSecret` é limpo na próxima rotação ou lazily pelo worker.

## 7. Contratos públicos

Todas as rotas abaixo ficam sob `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <JWT>` via `authenticate`, e devolvem erros no formato do error middleware:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found", "details": { "id": "…" } } }
```

Autorização: CRUD, deliveries e rotação aceitam qualquer role autenticada ([09:37] Sofia); replay exige `ADMIN` ([09:36] Larissa).

### 7.1 `POST /api/v1/webhooks` — cadastrar endpoint

Request:

```json
{
  "customerId": "6f1c2c1e-3a7b-4a1d-9c0e-2d3f4a5b6c7d",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```

Validação (`webhook.schemas.ts`): `customerId` uuid; `url` string URL válida com até 2 048 caracteres; `events` array não vazio de `z.nativeEnum(OrderStatus)`, sem duplicatas. Protocolo diferente de `https:` é rejeitado no service com `WEBHOOK_INVALID_URL` ([09:23] Sofia).

Response `201 Created` (única resposta que expõe a secret):

```json
{
  "id": "9b2e7d44-5c1a-4f3e-8a6b-1c2d3e4f5a6b",
  "customerId": "6f1c2c1e-3a7b-4a1d-9c0e-2d3f4a5b6c7d",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "secret": "4f0c9a1e7b3d5e2f8a6c4b1d9e7f3a5c2b8d6e4f1a9c7b5d3e2f8a6c4b1d9e7f",
  "createdAt": "2026-10-10T12:00:00.000Z",
  "updatedAt": "2026-10-10T12:00:00.000Z"
}
```

Status codes: `201`; `400 VALIDATION_ERROR` (shape); `400 WEBHOOK_INVALID_URL`; `401 UNAUTHORIZED`; `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### 7.2 `GET /api/v1/webhooks?customerId=&page=&pageSize=` — listar

Query: `customerId` uuid obrigatório; `page` ≥ 1 (default 1); `pageSize` 1..100 (default 20), como em `listOrdersQuerySchema`.

Response `200 OK` (formato de `paginated()` em `src/shared/http/response.ts`; **sem `secret`**):

```json
{
  "data": [
    {
      "id": "9b2e7d44-5c1a-4f3e-8a6b-1c2d3e4f5a6b",
      "customerId": "6f1c2c1e-3a7b-4a1d-9c0e-2d3f4a5b6c7d",
      "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
      "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-10-10T12:00:00.000Z",
      "updatedAt": "2026-10-10T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status codes: `200`; `400 VALIDATION_ERROR`; `401`.

### 7.3 `PATCH /api/v1/webhooks/:id` — editar

Request (todos opcionais, ao menos um):

```json
{ "url": "https://integracao.atlascomercial.com.br/oms/webhooks/v2", "events": ["SHIPPED", "DELIVERED"], "active": false }
```

Response `200 OK`: mesmo corpo de 7.2 (sem secret). Status codes: `200`; `400 VALIDATION_ERROR`; `400 WEBHOOK_INVALID_URL`; `401`; `404 WEBHOOK_NOT_FOUND`.

Semântica: alterar `events` **não** afeta linhas já enfileiradas na outbox ([ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao.md)). `active = false` interrompe novas inserções; linhas pendentes ficam em `FAILED` até reativação ou DLQ.

### 7.4 `DELETE /api/v1/webhooks/:id` — remover

Response `204 No Content`. Status codes: `204`; `401`; `404 WEBHOOK_NOT_FOUND`. Linhas de outbox do endpoint ainda não entregues são removidas na mesma transação; `webhook_deliveries` e `webhook_dead_letter` são mantidas para auditoria.

### 7.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Sem body. Response `200 OK`:

```json
{
  "id": "9b2e7d44-5c1a-4f3e-8a6b-1c2d3e4f5a6b",
  "secret": "a7c3e9f1b5d2a8c4e6f0b3d7a1c5e9f2b4d8a6c0e2f4b6d8a0c2e4f6b8d0a2c4",
  "previousSecretExpiresAt": "2026-10-11T12:00:00.000Z"
}
```

Status codes: `200`; `401`; `404 WEBHOOK_NOT_FOUND`.

### 7.6 Contrato de entrega (plataforma → cliente)

`POST {url}` do endpoint cadastrado. Headers ([09:44] Diego, [09:44] Sofia):

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID da linha da outbox; idêntico em todas as tentativas |
| `X-Webhook-Id` | UUID do endpoint cadastrado |
| `X-Timestamp` | ISO 8601 do instante do envio (muda a cada tentativa) |
| `X-Signature` | `sha256=<hex HMAC-SHA256(secret, body)>`; durante o grace, `sha256=<nova>,sha256=<antiga>` |

Body (snapshot gerado na inserção, [09:43] Diego):

```json
{
  "event_id": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-10T12:34:56.000Z",
  "order_id": "0e1f2a3b-4c5d-4e6f-8a9b-0c1d2e3f4a5b",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2c1e-3a7b-4a1d-9c0e-2d3f4a5b6c7d",
  "subtotal_cents": 18000,
  "discount_cents": 1000,
  "total_cents": 17000
}
```

Semântica para o cliente: responder **qualquer 2xx em até 10 s** significa recebido; qualquer outra resposta, timeout ou erro de conexão gera retentativa com o mesmo `X-Event-Id`. O cliente deve deduplicar por `event_id` (at-least-once) e verificar `X-Signature` antes de processar.

Verificação de referência (Node):

```ts
const expected = 'sha256=' + createHmac('sha256', secret).update(rawBody).digest('hex');
const ok = req.headers['x-signature'].split(',').some((s) => timingSafeEqual(Buffer.from(s), Buffer.from(expected)));
```

### 7.7 `GET /api/v1/webhooks/:id/deliveries?page=&pageSize=` — histórico de entregas

Query: `page` ≥ 1 (default 1); `pageSize` 1..100 (default 100, "os últimos 100" de [09:34] Marcos). Ordenado por `attemptedAt desc`.

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "7a8b9c0d-1e2f-4a3b-8c4d-5e6f7a8b9c0d",
      "eventId": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "error": null,
      "durationMs": 184,
      "attemptedAt": "2026-10-10T12:36:01.000Z",
      "payload": { "event_id": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f", "event_type": "order.status_changed", "to_status": "SHIPPED", "order_number": "ORD-000123" }
    },
    {
      "id": "6f7a8b9c-0d1e-4f2a-9b3c-4d5e6f7a8b9c",
      "eventId": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
      "attempt": 1,
      "success": false,
      "responseStatus": 503,
      "responseBody": "Service Unavailable",
      "error": "WEBHOOK_DELIVERY_REJECTED: HTTP 503",
      "durationMs": 95,
      "attemptedAt": "2026-10-10T12:35:00.000Z",
      "payload": { "event_id": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f", "event_type": "order.status_changed", "to_status": "SHIPPED", "order_number": "ORD-000123" }
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

`payload` é lido da outbox ou da DLQ pelo `eventId` (join no repository). Status codes: `200`; `400 VALIDATION_ERROR`; `401`; `404 WEBHOOK_NOT_FOUND`.

### 7.8 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ (ADMIN)

Sem body. Middlewares: `authenticate`, `requireRole('ADMIN')`.

Response `202 Accepted`:

```json
{
  "eventId": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "webhookId": "9b2e7d44-5c1a-4f3e-8a6b-1c2d3e4f5a6b",
  "status": "PENDING",
  "attempts": 0,
  "nextAttemptAt": "2026-10-10T15:00:00.000Z",
  "replayedById": "2c3d4e5f-6a7b-4c8d-9e0f-1a2b3c4d5e6f",
  "replayedAt": "2026-10-10T15:00:00.000Z"
}
```

Status codes: `202`; `401 UNAUTHORIZED`; `403 FORBIDDEN` (role insuficiente, via `requireRole`); `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_ALREADY_REPLAYED`; `409 WEBHOOK_INACTIVE`.

## 8. Matriz de erros

Prefixo `WEBHOOK_` em tudo do módulo ([09:29] Larissa). Erros HTTP estendem `AppError`; erros de entrega são internos ao worker e aparecem em `lastError`, `webhook_deliveries.error` e `webhook_dead_letter.reason`.

| Código | HTTP | Classe / origem | Quando | `details` |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends AppError` | `:id` de endpoint inexistente | `{ id }` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError extends AppError` | `customerId` inexistente no `POST` | `{ customerId }` |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError extends BadRequestError` | URL com protocolo diferente de `https:` | `{ url }` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `WebhookPayloadTooLargeError extends UnprocessableEntityError` | Snapshot > 64 KB na inserção; aborta `PATCH /orders/:id/status` | `{ bytes, limit }` |
| `WEBHOOK_INACTIVE` | 409 | `WebhookInactiveError extends ConflictError` | Replay para endpoint `active = false` | `{ id }` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError extends AppError` | `:id` de DLQ inexistente | `{ id }` |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `WebhookAlreadyReplayedError extends ConflictError` | DLQ já reprocessada | `{ id, replayedAt }` |
| `WEBHOOK_SECRET_REQUIRED` | — (interno) | worker | Endpoint sem secret utilizável ao assinar; linha vai para `FAILED` | — |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | worker | `fetch` abortado por `AbortSignal.timeout(10000)` | — |
| `WEBHOOK_DELIVERY_REJECTED` | — (interno) | worker | Resposta não-2xx | — |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — (interno) | worker | DNS, TLS, conexão recusada | — |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — (interno) | worker | 6º envio falhou; motivo registrado na DLQ | — |

Erros de shape (campo faltando, uuid inválido, status fora do enum) continuam como `VALIDATION_ERROR` (400) pelo `validate` middleware e `ZodError` no error middleware; `401 UNAUTHORIZED` e `403 FORBIDDEN` vêm de `auth.middleware.ts` sem alteração.

## 9. Estratégias de resiliência

| Preocupação | Estratégia | Fonte |
| --- | --- | --- |
| Cliente lento | Timeout de 10 s por chamada via `AbortSignal.timeout`; estouro conta como falha | [09:42] Diego |
| Cliente fora do ar | Backoff 1m/5m/30m/2h/12h, 5 retentativas, depois DLQ | [09:17] Diego e Larissa |
| Cliente sumiu de vez | DLQ com payload e motivo; replay manual por ADMIN | [09:18] Diego |
| Worker cai no meio de um batch | Linhas ficam `PROCESSING`; na volta, qualquer `PROCESSING` com mais de 60 s é devolvida a `PENDING` (passo 1 do loop) | Decisão de implementação (seção 15) |
| API reinicia | Worker é processo separado; não é afetado | [09:11] Diego |
| Banco indisponível para o worker | Loop captura a exceção, loga `webhook_poll_failed` e tenta no próximo ciclo; não derruba o processo | Decisão de implementação |
| Pico de eventos | Batch de 50 por ciclo; índice `(status, nextAttemptAt)`; eventos excedentes entram no ciclo seguinte | [09:08] Diego |
| Duplicidade por retry | `X-Event-Id` estável; cliente dedupica | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Payload gigante | Limite de 64 KB validado na inserção; erro e rollback | [09:24] Larissa |
| Vazamento de secret | Secret por endpoint, rotação com grace 24 h, redaction no Pino, secret nunca retornada em `GET` | [09:21] Sofia, [09:22] Diego |
| Shutdown gracioso | `SIGTERM` para o loop, aguarda a entrega corrente terminar (máx. 10 s), `prisma.$disconnect()` | Padrão de `src/server.ts` |

Fallback: **não há** canal alternativo de notificação nesta fase (e-mail adiado, [09:37] Larissa). O fallback operacional é o histórico de `deliveries` para o cliente e a DLQ com replay para a equipe.

## 10. Observabilidade

O projeto não tem exportador de métricas nem tracing distribuído; `package.json` só traz `pino` e `pino-http`. Por isso a observabilidade desta fase é **baseada em logs estruturados** com nomes fixos, de onde métricas são derivadas pela plataforma de logs. Nenhuma biblioteca nova ([09:29] Bruno).

### Logs (Pino, `src/shared/logger/index.ts`)

| Evento (`msg`) | Nível | Campos | Emitido por |
| --- | --- | --- | --- |
| `webhook_outbox_enqueued` | info | `orderId`, `toStatus`, `count`, `correlationId` | `publishWebhookEvent` |
| `webhook_poll_cycle` | debug | `batchSize`, `oldestPendingAgeMs` | worker |
| `webhook_poll_failed` | error | `err` | worker |
| `webhook_delivery_attempt` | info | `eventId`, `webhookId`, `attempt`, `correlationId` | worker |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `attempt`, `durationMs`, `responseStatus` | worker |
| `webhook_delivery_failed` | warn | `eventId`, `webhookId`, `attempt`, `durationMs`, `errorCode`, `nextAttemptAt` | worker |
| `webhook_dead_lettered` | warn | `eventId`, `webhookId`, `attempts`, `reason` | worker |
| `webhook_replayed` | info | `eventId`, `webhookId`, `replayedById` | admin controller |
| `webhook_secret_rotated` | info | `webhookId`, `userId`, `previousSecretExpiresAt` | service |
| `worker_started` / `worker_stopped` | info | `pollIntervalMs`, `batchSize` | `src/worker.ts` |

Nunca logar `secret`, `previousSecret`, `X-Signature` ou o body completo; `redactPaths` cobre os dois primeiros.

### Métricas (derivadas dos logs)

| Métrica | Derivação | Uso |
| --- | --- | --- |
| `webhook_delivery_total{result}` | contagem de `webhook_delivery_succeeded` / `webhook_delivery_failed` | taxa de sucesso por cliente |
| `webhook_delivery_duration_ms` | percentis de `durationMs` | latência do cliente; p99 próximo de 10 s indica cliente lento |
| `webhook_outbox_pending_age_ms` | `oldestPendingAgeMs` de `webhook_poll_cycle` | lag da fila; > 10 s viola o requisito de [09:02] |
| `webhook_dead_letter_total` | contagem de `webhook_dead_lettered` | alerta operacional |
| `webhook_status_change_duration_ms` | `durationMs` do `http_request` em `PATCH /orders/:id/status` (já emitido por `request-logger.middleware.ts`) | detectar degradação da transação de status |

### Tracing

- `correlationId` = `req.id` do `PATCH /orders/:id/status` (gerado ou propagado por `X-Request-Id` em `request-logger.middleware.ts`), gravado na outbox e presente em todos os logs do worker daquele evento. Permite seguir requisição → outbox → tentativas → DLQ com um único identificador.
- `X-Event-Id` é o identificador de correlação com o cliente: o portal orienta o cliente a logá-lo.
- Tracing distribuído (OpenTelemetry) **não entra** nesta fase; o `correlationId` é o gancho para adotá-lo depois.

## 11. Dependências e compatibilidade

| Dependência | Situação |
| --- | --- |
| Node `>= 20` (`package.json` → `engines`) | `fetch`, `AbortSignal.timeout`, `crypto.randomUUID`, `crypto.createHmac` nativos; nenhuma lib HTTP nova |
| `@prisma/client` 5.22 / `prisma` 5.22 | Migration adicional; transações interativas já usadas em `OrderService` |
| MySQL 8 (`docker-compose.yml`) | Coluna `JSON` para `events`/`payload`; sem extensão nova |
| `zod` 3.23 | Schemas do módulo |
| `pino` 9.5 | Logs; redaction estendida |
| `jsonwebtoken` / `auth.middleware.ts` | Sem mudança; roles existentes `ADMIN` e `OPERATOR` |
| `vitest` + `supertest` (`tests/`) | Testes de integração do CRUD e do publisher; `tests/setup.ts` precisa limpar as novas tabelas |

Compatibilidade:

- `PATCH /api/v1/orders/:id/status` mantém contrato e resposta; ganha um novo erro possível (`422 WEBHOOK_PAYLOAD_TOO_LARGE`), na prática inalcançável com o payload enxuto.
- Nenhuma rota, tabela ou coluna existente é removida ou renomeada. A migration é aditiva.
- Deploy: a migration deve rodar **antes** de subir a API nova; o worker pode subir depois da API sem perda, pois eventos ficam na outbox.
- Testes existentes em `tests/orders.test.ts` continuam válidos: sem endpoint cadastrado, `publishWebhookEvent` não insere nada.

## 12. Critérios de aceite técnicos

Outbox e integração:

1. `PATCH /orders/:id/status` com um endpoint ativo assinando `to_status` cria exatamente uma linha `PENDING` em `webhook_outbox` por endpoint assinante, com `payload.event_id === id` da linha e `correlationId` igual ao `X-Request-Id` da resposta.
2. Com nenhum endpoint assinando `to_status`, nenhuma linha é criada e a resposta é idêntica à atual.
3. Se a inserção na outbox falhar (simulada por payload > 64 KB), a resposta é `422 WEBHOOK_PAYLOAD_TOO_LARGE`, e `orders.status`, `order_status_history` e `products.stock_quantity` permanecem inalterados.
4. Transição inválida (`INVALID_STATUS_TRANSITION`) não cria linha na outbox.

CRUD:

5. `POST /webhooks` com URL `http://` retorna `400 WEBHOOK_INVALID_URL`; com `https://` retorna `201` e a secret aparece **apenas** nessa resposta.
6. `GET /webhooks`, `PATCH`, `DELETE`, `GET /webhooks/:id/deliveries` seguem os contratos da seção 7 e nunca expõem `secret`.
7. `POST /webhooks/:id/rotate-secret` devolve nova secret e define `previousSecretExpiresAt = now + 24 h`.

Worker:

8. Com endpoint de teste respondendo `200`, a entrega ocorre em até 2 s após o commit (medido em teste de integração com servidor HTTP local), com headers `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature`, `Content-Type: application/json`.
9. `X-Signature` verifica contra `HMAC-SHA256(secret, body)`; durante o grace, a verificação com a secret antiga também passa.
10. Endpoint respondendo `503` leva a linha para `FAILED` com `attempts = 1` e `nextAttemptAt ≈ now + 1 min`; uma linha de `webhook_deliveries` é gravada com `success = false`.
11. Endpoint que demora mais de 10 s gera `WEBHOOK_DELIVERY_TIMEOUT`.
12. Na 6ª falha a linha some da outbox e aparece em `webhook_dead_letter` com `reason` iniciando em `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`.
13. Dois pedidos distintos com eventos pendentes são entregues em ordem de `createdAt`.
14. Linha `PROCESSING` há mais de 60 s é retomada no ciclo seguinte.

Replay:

15. `POST /admin/webhooks/dead-letter/:id/replay` com role `OPERATOR` retorna `403 FORBIDDEN`; com `ADMIN` retorna `202`, recria a linha na outbox com o mesmo `id`, marca `replayedAt`/`replayedById` e emite log `webhook_replayed` com `replayedById`.
16. Segundo replay do mesmo id retorna `409 WEBHOOK_ALREADY_REPLAYED`.

Não funcionais:

17. Nenhum log contém secret (teste com `pino` em modo de captura procurando o valor).
18. `npm run lint` e `npm test` passam; `npm run build` gera `dist/worker.js`.
19. Revisão de segurança de Sofia concluída sobre `webhook.signature.ts` e a rotação ([09:46] Sofia).

## 13. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Transação de `changeStatus` fica mais lenta (consulta de endpoints + inserção) | Média | Médio | Índice em `customerId`/`active`; payload enxuto; medir `http_request.durationMs` do `PATCH` antes e depois; limite claro de 1 consulta + 1 `createMany` |
| Head-of-line blocking: cliente lento (até 10 s) atrasa os demais do mesmo batch, pois o processamento é sequencial | Média | Médio | Batch pequeno e ciclo de 2 s limitam o atraso; registrar como limitação; concorrência por endpoint é evolução natural junto com múltiplos workers |
| Cliente não dedupica por `X-Event-Id` | Média | Médio | Contrato at-least-once destacado no portal ([09:26] Marcos); `event_id` também dentro do body |
| Secret armazenada em claro no banco | Baixa | Alto | Necessária para assinar; redaction em logs, nunca exposta em `GET`, rotação com grace; ponto explícito para a revisão de Sofia |
| Outbox cresce sem arquivamento | Alta (longo prazo) | Baixo (curto prazo) | Linhas entregues saem do índice de trabalho (`status = DELIVERED`); arquivamento é questão em aberto no RFC |
| Eventos na DLQ sem ninguém perceber | Média | Médio | Log `warn` `webhook_dead_lettered` como gatilho de alerta; e-mail ao cliente fica para a próxima fase |
| Interpretação de "5 tentativas" (5 envios vs. 1 + 5 retentativas) | Baixa | Baixo | Este FDD adota 1 + 5 por bater com "quase 15 horas"; confirmar com Diego na revisão |
| `tests/setup.ts` não limpar as novas tabelas gera testes flaky | Média | Baixo | Item explícito na seção 4 |

## 14. Plano de implementação (ordem sugerida)

Segue a estimativa de [09:46] Larissa: três sprints.

| Sprint | Entrega |
| --- | --- |
| 1 | Modelos Prisma + migration; `webhook.publisher.ts`; alteração em `order.service.ts`; classes de erro; testes dos critérios 1 a 4 |
| 2 | `webhook.processor.ts`, `src/worker.ts`, script `npm run worker`; retry, DLQ; `webhook.signature.ts`; testes dos critérios 8 a 14 |
| 3 | CRUD, deliveries, rotação, replay ADMIN; redaction no logger; testes 5 a 7, 15 a 18; dois dias úteis de revisão de segurança (critério 19) |

## 15. Decisões de implementação tomadas neste FDD

Pontos que a reunião não fechou, resolvidos aqui seguindo o código existente. Todos estão abertos a objeção na revisão.

| # | Decisão | Justificativa |
| --- | --- | --- |
| 1 | `customerId` vai no **body** do `POST` e na **query** do `GET` | A reunião deixou "body ou path" ([09:32] Larissa). `createOrderSchema` e `listOrdersQuerySchema` em `order.schemas.ts` usam exatamente esse padrão |
| 2 | Rotas sob `/api/v1/webhooks` e `/api/v1/admin/webhooks` | Prefixo em `src/app.ts`; caminho de replay literal de [09:35] Diego |
| 3 | Paginação `page`/`pageSize` com `paginated()` | `src/shared/http/response.ts` e `listOrdersQuerySchema` |
| 4 | Interpretação "1 envio + 5 retentativas" (6 envios) | Soma 1m+5m+30m+2h+12h = 14,6 h bate com "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego) |
| 5 | `X-Signature` com formato `sha256=<hex>` e duas assinaturas durante o grace | Única forma de a secret antiga "ficar válida em paralelo" ([09:21] Sofia) em webhook outbound sem o cliente conhecer a nova |
| 6 | Secret gerada com `randomBytes(32).toString('hex')` | `node:crypto`, sem dependência nova ([09:29] Bruno) |
| 7 | `fetch` nativo + `AbortSignal.timeout` | `engines.node >= 20`; sem lib HTTP nova |
| 8 | Batch de 50, recuperação de `PROCESSING` após 60 s, `responseBody` truncado em 2 KB | "Batch pequeno" ([09:08] Diego); worker pode reiniciar ([09:11] Diego); resposta precisa aparecer em deliveries ([09:34] Marcos) sem inflar a tabela |
| 9 | Erro de payload > 64 KB **aborta** a mudança de status | "Erra" ([09:23] Sofia, [09:24] Larissa) combinado com "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno) |
| 10 | `orderId` da outbox sem FK | `OrderService.delete` permite apagar pedidos `CANCELLED`; o evento de cancelamento deve sobreviver |
| 11 | Replay preserva o `id` (`X-Event-Id`) original | Mantém a deduplicação do cliente coerente ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) |
| 12 | Snapshot inclui `subtotal_cents` e `discount_cents` além de `total_cents` | "Campos básicos da order tipo total_cents" ([09:43] Diego); os três estão em `Order` no `schema.prisma`; `items` continua fora |
| 13 | Classes de erro em `src/shared/errors/http-errors.ts` | Onde `InvalidStatusTransitionError` e `InsufficientStockError` já vivem |
| 14 | Variáveis `WEBHOOK_*` em `env.ts` com defaults iguais aos valores da reunião | Padrão de `env.ts` (Zod com `.default`); evita números mágicos no código |
