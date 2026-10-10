# ADR-006 — Webhooks como módulo padrão do projeto, reaproveitando AppError, Pino, error middleware, Zod e `requireRole`

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Bruno (Eng. Pleno, Pedidos), Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Fonte** | `TRANSCRICAO.md` [09:23], [09:27] a [09:30], [09:36], [09:48] |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) |

## Contexto

A codebase tem um padrão claro e consistente, que a feature de webhooks pode seguir sem criar nada novo ([09:27] Bruno). Mapeamento do que já existe:

| Padrão existente | Onde está no código |
| --- | --- |
| Módulo por domínio com `controller`, `service`, `repository`, `routes` e `schemas` | `src/modules/orders/` (`order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`); mesmo layout em `customers`, `products`, `users` |
| Montagem de dependências e roteador | `src/app.ts` (`buildControllers`, `buildApp`) e `src/routes/index.ts` (`buildApiRouter`, prefixo `/api/v1`) |
| Classe base de erro com `statusCode`, `errorCode` e `details` | `src/shared/errors/app-error.ts` (`AppError`) |
| Erros HTTP e de domínio com código em `SCREAMING_SNAKE_CASE` | `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError`, `InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) |
| Error middleware centralizado que trata `AppError`, `ZodError` e `PrismaClientKnownRequestError` (`P2002`, `P2025`) | `src/middlewares/error.middleware.ts` |
| Logger Pino com redaction de campos sensíveis (`*.password`, `*.token`, `authorization`) | `src/shared/logger/index.ts` |
| Validação de entrada com Zod via middleware | `src/middlewares/validate.middleware.ts` e `*.schemas.ts` de cada módulo |
| Autenticação JWT e autorização por role (`ADMIN` / `OPERATOR`) | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| `PrismaClient` criado por fábrica | `src/config/database.ts` (`createPrismaClient`) |
| Variáveis de ambiente validadas com Zod | `src/config/env.ts` |

## Decisão

"Reuso máximo do que já existe" ([09:30] Larissa). Concretamente:

- **Novo módulo `src/modules/webhooks/`** com a mesma estrutura dos demais (`controller`, `service`, `repository`, `routes`, `schemas`), mais o processador do worker (`webhook.worker.ts` ou `webhook.processor.ts`) ([09:27] a [09:28] Bruno). O entry-point do worker fica em `src/worker.ts` ([ADR-002](ADR-002-worker-separado-em-polling.md)).
- **Erros**: novas classes estendem `AppError` (ou as subclasses de `http-errors.ts`) e usam códigos com **prefixo `WEBHOOK_`** para tudo do módulo, por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa).
- **Error middleware**: nenhuma alteração em `src/middlewares/error.middleware.ts`; como os novos erros são `AppError`, já são serializados no formato `{ error: { code, message, details } }` ([09:29] Bruno).
- **Logger**: Pino de `src/shared/logger/index.ts`, sem adicionar nova biblioteca de log ([09:29] Bruno). A secret de webhook deve entrar na lista de redaction.
- **Validação**: schemas Zod no padrão `webhook.schemas.ts`, incluindo a validação de URL `https` obrigatória ([09:23] Sofia).
- **Autorização**: endpoints de CRUD de configuração usam `authenticate` com qualquer role por enquanto ([09:37] Sofia); o replay de DLQ usa `requireRole('ADMIN')` já existente ([09:36] Larissa).
- **Banco**: o worker cria sua própria instância de `PrismaClient` via a mesma fábrica, com a mesma `DATABASE_URL` ([09:30] Bruno).
- **Identificadores**: UUID em todas as novas tabelas, como no restante de `prisma/schema.prisma` ([09:51] Larissa).

## Alternativas Consideradas

### A. Módulo isolado com infraestrutura própria de erro e log

Criar hierarquia de erros e logger dedicados ao módulo de webhooks, ou adotar uma biblioteca nova.

- Descartada ([09:29] Bruno: "não vamos botar nada novo").
- Trade-off que motivou o descarte: duplicaria conceitos já resolvidos (`AppError`, error middleware, Pino com redaction), quebraria a uniformidade do formato de erro da API e aumentaria a superfície de revisão de segurança.

### B. Pacote/serviço separado para webhooks

Extrair webhooks para um repositório ou serviço próprio.

- Não defendida na reunião; conflita com a decisão do outbox na mesma transação ([ADR-001](ADR-001-outbox-no-mysql.md)), que exige o `Prisma.TransactionClient` do `OrderService`.
- Trade-off: isolaria o deploy, mas inviabilizaria a atomicidade e obrigaria um time pequeno a operar mais um serviço.

## Consequências

### Positivas

- **Velocidade de implementação**: o time já conhece o layout; a estimativa de três sprints assume reuso ([09:46] Larissa).
- **Consistência de API**: mesmos formatos de erro, mesma autenticação, mesmos logs estruturados.
- **Zero mudança no error middleware** e nos demais middlewares.
- **Menos superfície para a revisão de segurança**, que pode focar em HMAC e secret ([09:46] Sofia).

### Negativas / trade-offs assumidos

- **`OrderService.changeStatus` precisa ser alterado** para chamar `publishWebhookEvent(tx, ...)`, criando dependência do módulo de pedidos para o de webhooks ([09:41] Bruno).
- **`src/app.ts` e `src/routes/index.ts` precisam ser editados** para registrar o novo módulo em `buildControllers` e `buildApiRouter`.
- **A lista de redaction do Pino precisa ser estendida** para cobrir a secret do webhook; omitir isso vazaria secret em log, exatamente o incidente relatado em [09:22].
- **Limites dos padrões atuais são herdados**: por exemplo, `requireRole` só conhece `ADMIN` e `OPERATOR`; qualquer endurecimento futuro de permissões do CRUD ([09:37] Sofia) terá que evoluir esse mecanismo.
