# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-10-10 |
| **Revisores** | Bruno (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma), Sofia (Eng. de Segurança), Marcos (Product Manager) |
| **Origem** | Reunião técnica registrada em `TRANSCRICAO.md` |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo executivo (TL;DR)

Propomos adicionar ao OMS um mecanismo de **webhooks outbound** que notifica clientes B2B sempre que o status de um pedido muda. A solução usa o **padrão Outbox no MySQL existente**: a mudança de status já é uma transação única em `OrderService.changeStatus`, e passaremos a inserir o evento nessa mesma transação. Um **worker em processo separado** lê a outbox por **polling a cada 2 s** e entrega via HTTPS, com **retry em backoff exponencial (5 tentativas)** e **DLQ em tabela separada** com replay manual por ADMIN. Cada entrega é assinada com **HMAC-SHA256** usando **secret única por endpoint**, rotacionável com grace period de 24 h, e carrega um **`X-Event-Id`** para deduplicação, já que a garantia é **at-least-once**. Nenhuma infraestrutura nova é introduzida; o módulo segue os padrões já existentes do projeto. Estimativa: **três sprints**, incluindo revisão de segurança.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados quando o status dos seus pedidos muda. Hoje eles fazem polling no `GET /orders`, o que torna a integração lenta e cara para eles; a Atlas sinalizou risco de migrar para concorrente se não houver entrega até o fim do trimestre ([09:00] Marcos). Para esses clientes, qualquer latência abaixo de 10 s é "tempo real" ([09:02] Marcos). O fluxo é apenas de saída: a plataforma envia, o cliente recebe ([09:02] Marcos).

A aplicação atual (Node.js, TypeScript, Express, Prisma sobre MySQL) não tem nenhum mecanismo de eventos, fila ou notificação externa. O ponto natural de integração é a transação de `changeStatus` em `src/modules/orders/order.service.ts`, que hoje atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos. Qualquer solução precisa preservar a atomicidade dessa transação e não pode ser bloqueada pela latência ou disponibilidade de um endpoint externo ([09:04] Bruno).

## 3. Proposta técnica

### 3.1 Visão geral

```
PATCH /orders/:id/status
        │
        ▼
OrderService.changeStatus ── $transaction ──┐
   update orders                            │
   insert order_status_history              │  commit/rollback atômico
   debit/replenish stock                    │
   publishWebhookEvent(tx, ...) ──► webhook_outbox (snapshot JSON, 1 linha por webhook assinante)
                                            ┘
                                                     ▲ polling 2 s
                                                     │
                                   src/worker.ts (processo separado)
                                     lê pendentes → assina HMAC → POST https://cliente
                                        ├── 2xx  → entregue (+ registro em deliveries)
                                        └── erro/timeout 10 s → reagenda (1m/5m/30m/2h/12h)
                                                               └── 5ª falha → webhook_dead_letter
                                                                               ▲
                                   POST /admin/webhooks/dead-letter/:id/replay (ADMIN)
```

### 3.2 Componentes

- **Módulo `src/modules/webhooks/`**, no mesmo layout dos módulos atuais (`controller`, `service`, `repository`, `routes`, `schemas`), mais o processador do worker. Reaproveita `AppError`, o error middleware, o logger Pino, os schemas Zod e `requireRole` sem alterações nesses componentes. Códigos de erro com prefixo `WEBHOOK_` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).
- **Configuração de webhook** (CRUD autenticado): `url` (somente HTTPS), `secret` gerada pela plataforma, `customer_id`, lista de status assinados e flag de ativo. Endpoints `POST`, `PATCH`, `DELETE`, `GET` de listagem, histórico `GET /webhooks/:id/deliveries` e rotação de secret ([09:31] a [09:34]).
- **Outbox transacional** `webhook_outbox`, alimentada dentro da transação de `changeStatus` por `publishWebhookEvent(tx, order, fromStatus, toStatus)`. O payload é gravado como snapshot no momento da inserção e o filtro por status assinado é aplicado na inserção, gerando uma linha por webhook de destino ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md), [ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao.md)).
- **Worker** em `src/worker.ts` (`npm run worker`), processo separado da API, mesma `DATABASE_URL` e `PrismaClient` próprio, polling de 2 s em batch pequeno, single-worker nesta fase ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).
- **Entrega**: `POST` com `Content-Type: application/json` e headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`; timeout de 10 s; payload enxuto (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`), sem itens ([09:43] a [09:45]).
- **Resiliência**: 5 tentativas com backoff 1m/5m/30m/2h/12h; após a última falha, o evento vai para `webhook_dead_letter` com payload, motivo e timestamp; replay manual via endpoint ADMIN com log de quem acionou ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).
- **Segurança**: HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h de grace; limite de 64 KB por payload; revisão de segurança antes do deploy ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- **Semântica de entrega**: at-least-once, deduplicação pelo cliente via `X-Event-Id`; ordering garantido apenas por `order_id` e enquanto houver um único worker ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).

### 3.3 Mudanças no código existente

| Ponto de mudança | O que muda |
| --- | --- |
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, ...)` dentro da transação; falha na inserção provoca rollback |
| `prisma/schema.prisma` | Novos modelos: configuração de webhook, `webhook_outbox`, `webhook_dead_letter` (e registro de entregas); ids UUID |
| `src/app.ts` e `src/routes/index.ts` | Registro do novo módulo em `buildControllers` e `buildApiRouter` |
| `src/server.ts` → novo `src/worker.ts` | Segundo entry-point, com script em `package.json` |
| `src/shared/logger/index.ts` | Inclusão da secret de webhook nos caminhos de redaction |

O detalhamento de modelos, contratos HTTP, matriz de erros e fluxos está no [FDD](FDD.md).

## 4. Alternativas consideradas

| Alternativa | Trade-off que levou ao descarte | Detalhe |
| --- | --- | --- |
| **Disparo HTTP síncrono dentro de `changeStatus`** | A transação de status ficaria refém da latência e disponibilidade do cliente; cliente lento travaria mudança de status de outros pedidos, e não existe rollback possível se o cliente está fora do ar ([09:04] Bruno, [09:06] Diego) | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **Fila externa (Redis Streams)** | Exige subir e operar infraestrutura nova para um time pequeno; considerado overengineering. Publicar fora da transação SQL perderia a atomicidade ([09:07] Larissa e Diego) | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **Trigger no banco para acordar o worker** | MySQL não notifica processos externos (sem `LISTEN/NOTIFY`); seria preciso improvisar. Polling de 2 s já atende o requisito de < 10 s ([09:09] Diego) | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| **Três tentativas de retry** | Esgotaria em ~30 min; manutenção planejada de 2 h do cliente mandaria eventos à DLQ sem necessidade ([09:16] Diego) | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| **Entrega exactly-once** | Exigiria coordenação dos dois lados; at-least-once com `event_id` é o padrão de mercado (Stripe, GitHub) e resolve a grande maioria dos casos ([09:25] Diego) | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| **Secret global da plataforma** | Um vazamento comprometeria todos os clientes; secret por endpoint contém o raio de impacto ([09:21] Sofia) | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |

## 5. Questões em aberto

1. **Rate limiting de saída.** Se um cliente tem 50 pedidos mudando de status em um minuto, enviamos 50 chamadas. Decidido "observar e implementar se virar problema" ([09:38] Diego, [09:39] Larissa). Fica fora desta fase, sem métrica de gatilho definida.
2. **Alerta ao cliente por e-mail** quando o webhook dele falha repetidamente. Explicitamente adiado para "próxima fase, depois de medir o impacto" ([09:37] Larissa). Nesta fase, o cliente só descobre falhas consultando `GET /webhooks/:id/deliveries`.
3. **Arquivamento da outbox.** Linhas entregues devem ser arquivadas "depois de 30 dias ou assim", mas isso foi declarado fora do escopo da feature ([09:08] Diego). Sem política definida, a tabela cresce indefinidamente.
4. **Escala para múltiplos workers.** Single-worker é limitação conhecida; particionamento por `order_id` ou lock pessimista ficou como "problema do futuro" ([09:13] Diego). Não há critério de quando revisitar.
5. **Onde vai o `customer_id` no CRUD.** O JWT representa o usuário operador, não o cliente; ficou definido que o `customer_id` vem "no body ou no path" ([09:32] Larissa), sem escolha final. Precisa ser fechado no FDD.
6. **Endurecimento de permissões do CRUD.** Hoje qualquer role autenticada pode gerenciar webhooks; "mais pra frente a gente pode endurecer" ([09:37] Sofia).

Fora de escopo por decisão explícita, sem necessidade de reabrir: dashboard visual para o cliente, que é projeto separado do time de frontend ([09:40] Larissa); webhooks inbound ([09:02] Marcos).

## 6. Impacto e riscos

### Impacto

- **Código**: alteração no caminho crítico de `changeStatus`, novo módulo, novas tabelas e migration, novo entry-point de processo. Nenhuma mudança nos middlewares ou nos demais módulos.
- **Operação**: passa a existir um segundo processo (worker) para deployar, monitorar e reiniciar de forma independente da API.
- **Clientes**: precisam implementar verificação HMAC, deduplicação por `X-Event-Id` e tolerar eventos fora de ordem em retries; o portal de desenvolvedor documentará isso ([09:26] Marcos, [09:40] Marcos).
- **Prazo**: três sprints, com pelo menos dois dias úteis de revisão de segurança de Sofia antes do deploy ([09:46] Larissa e Sofia). Atlas espera entrega até o fim de novembro ([09:45] Marcos).

### Riscos

| Risco | Mitigação |
| --- | --- |
| Transação de status fica mais pesada (consulta de webhooks + inserção de snapshot) | Inserção única por webhook assinante, payload enxuto, índices na outbox; medir latência de `changeStatus` antes e depois |
| Cliente não deduplica e processa evento duas vezes | Contrato at-least-once documentado com destaque no portal; `X-Event-Id` estável em todas as tentativas |
| Vazamento de secret pelo cliente (já ocorreu, [09:22] Diego) | Secret por endpoint, rotação com grace de 24 h, redaction no Pino |
| Outbox cresce sem arquivamento | Registrado como questão em aberto; índices por status e `created_at` mantêm o polling barato enquanto isso |
| Single-worker vira gargalo com aumento de volume | Limitação documentada; caminho de evolução (particionamento ou lock) já identificado |
| Eventos presos na DLQ sem ninguém perceber | Endpoint de replay com auditoria; alerta automático fica para fase futura |
| Prazo apertado para fim de novembro | Escopo cortado deliberadamente (sem e-mail, rate limit, dashboard); revisão de segurança já embutida na estimativa |

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado, polling de 2 s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint e rotação](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com `X-Event-Id`](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload na inserção](adrs/ADR-007-snapshot-do-payload-na-insercao.md)
- [ADR-008 — Filtragem de eventos na inserção](adrs/ADR-008-filtragem-de-eventos-na-insercao.md)
