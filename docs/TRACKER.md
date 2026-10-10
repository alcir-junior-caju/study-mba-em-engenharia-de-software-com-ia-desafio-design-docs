# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este tracker mapeia cada item registrado em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/*.md` à sua origem: uma fala da transcrição (`TRANSCRICAO`, com `[hh:mm] Nome`) ou um arquivo do código base (`CODIGO`, com caminho real). Itens sem origem identificável foram removidos dos documentos ou marcados neles como decisão de implementação justificada pelo código.

Convenção de IDs: `<DOC>-<TIPO>-<NN>`. Tipos: `FR` requisito funcional, `NFR` requisito não funcional, `OBJ` objetivo, `FE` fora de escopo, `ALT` alternativa descartada, `QA` questão em aberto, `DET` detalhe da decisão, `COD` ponto de integração com o código, `INT` integração, `CONTRATO` contrato HTTP, `ERRO` código de erro, `FLUXO` fluxo, `RES` resiliência, `OBS` observabilidade, `DI` decisão de implementação, `RISK` risco, `DEP` dependência.

**Resumo:** 247 linhas · 188 com Fonte = `TRANSCRICAO` (76%) · 59 com Fonte = `CODIGO`.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) por notificação de status | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-01 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-02 | docs/PRD.md | Motivação | Atlas pode migrar para concorrente se não houver entrega até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-03 | docs/PRD.md | Requisito Não Funcional | "Tempo real" para os clientes é qualquer latência abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-PRB-04 | docs/PRD.md | Restrição | Webhooks apenas outbound (plataforma → cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Cadastro feito via API com JWT do OMS por usuários que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | Administradores (role ADMIN) reprocessam a DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência de notificação < 10 s em 100% dos eventos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Nenhum evento perdido: termina em DELIVERED ou DLQ | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Eliminar o polling dos clientes em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Entrega até fim de novembro, em três sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Não degradar a latência de PATCH /orders/:id/status | TRANSCRICAO | [09:04] Bruno |
| PRD-FE-01 | docs/PRD.md | Fora de escopo | E-mail de alerta ao cliente adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-FE-02 | docs/PRD.md | Fora de escopo | Rate limiting de envio: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-FE-03 | docs/PRD.md | Fora de escopo | Dashboard visual é projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-FE-04 | docs/PRD.md | Fora de escopo | Webhooks inbound descartados | TRANSCRICAO | [09:02] Marcos |
| PRD-FE-05 | docs/PRD.md | Fora de escopo | Arquivamento de linhas entregues fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| PRD-FE-06 | docs/PRD.md | Fora de escopo | Ordering global e múltiplos workers: limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| PRD-FE-07 | docs/PRD.md | Fora de escopo | Itens do pedido não vão no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FE-08 | docs/PRD.md | Fora de escopo | Permissões diferenciadas do CRUD ficam para depois | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (POST) com customerId, URL e status; secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Requisito Funcional | customerId informado no body/path, não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Lista de status assinados por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Payload JSON: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-FR-07b | docs/PRD.md | Requisito Funcional | Header X-Webhook-Id com id do cadastro | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Secret única por endpoint, rotacionável via API, antiga válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com sucesso/falha, payload, response, tempo (GET /webhooks/:id/deliveries) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry 5 tentativas com backoff 1m/5m/30m/2h/12h, depois DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-10b | docs/PRD.md | Requisito Funcional | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11b | docs/PRD.md | Requisito Funcional | Replay exige role ADMIN e loga quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | URL deve ser https; http é recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Garantia at-least-once; cliente dedupica por X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Evento reflete o estado do pedido no momento da mudança (snapshot) | TRANSCRICAO | [09:52] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s; polling de 2 s atende | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Evento na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento/offline não afeta a API de pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload máximo 64 KB, erro acima disso | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 sobre o corpo, secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Janela de resiliência de quase 15 h | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Auditoria do replay; secret nunca em logs | TRANSCRICAO | [09:22] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova (sem Redis) | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Reuso dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Ordering por order_id apenas com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Três sprints com revisão de segurança incluída | TRANSCRICAO | [09:47] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Transação changeStatus é o ponto único de geração de eventos | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança de Sofia (≥ 2 dias úteis) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Portal de desenvolvedor documenta at-least-once e integração | TRANSCRICAO | [09:26] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Perder a Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não dedupica (responsabilidade jogada para o cliente) | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente (já ocorreu) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Eventos na DLQ sem alerta (e-mail adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-RISK-05 | docs/PRD.md | Risco | Transação de status já é pesada e ficará mais lenta | TRANSCRICAO | [09:04] Bruno |
| PRD-RISK-06 | docs/PRD.md | Risco | Worker único processa em sequência; cliente lento atrasa os demais | TRANSCRICAO | [09:12] Diego |
| PRD-RISK-07 | docs/PRD.md | Risco | Outbox cresce sem arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-TEST-01 | docs/PRD.md | Estratégia de testes | Testes de integração seguem o padrão existente (vitest + supertest) | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia de testes | Revisão manual de segurança de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-03 | docs/PRD.md | Estratégia de testes | Marcos atualiza os clientes sobre prazo e contrato | TRANSCRICAO | [09:49] Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo HTTP síncrono dentro de changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila externa (mais infra) | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger no banco (MySQL não notifica processo externo) | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Três tentativas de retry | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Alerta por e-mail ao cliente | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Arquivamento da outbox após ~30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers (particionamento ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | customer_id no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-QA-06 | docs/RFC.md | Questão em aberto | Endurecimento de permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-COD-01 | docs/RFC.md | Mudança no código | changeStatus chama publishWebhookEvent(tx, ...) | CODIGO | src/modules/orders/order.service.ts |
| RFC-COD-02 | docs/RFC.md | Mudança no código | Novos modelos Prisma com ids UUID | CODIGO | prisma/schema.prisma |
| RFC-COD-03 | docs/RFC.md | Mudança no código | Registro do módulo em buildControllers/buildApiRouter | CODIGO | src/app.ts |
| RFC-COD-04 | docs/RFC.md | Mudança no código | Novo entry-point src/worker.ts espelhando server.ts | CODIGO | src/server.ts |
| RFC-COD-05 | docs/RFC.md | Mudança no código | Secret de webhook na redaction do Pino | CODIGO | src/shared/logger/index.ts |
| RFC-IMP-01 | docs/RFC.md | Impacto | Estimativa de três sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-IMP-02 | docs/RFC.md | Impacto | Clientes implementam HMAC e dedup; portal documenta | TRANSCRICAO | [09:26] Marcos |
| RFC-IMP-03 | docs/RFC.md | Impacto | Segundo processo para operar (worker) | TRANSCRICAO | [09:11] Diego |
| RFC-IMP-04 | docs/RFC.md | Impacto | Atlas espera entrega até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | HTTP síncrono no service de orders | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams: overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-DET-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Detalhe | Inserção na mesma transação SQL de orders e order_status_history | TRANSCRICAO | [09:06] Diego |
| ADR-001-DET-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Detalhe | Função publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| ADR-001-DET-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Detalhe | Índices no status do evento e em created_at | TRANSCRICAO | [09:08] Diego |
| ADR-001-DET-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Detalhe | Chave primária UUID como no resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-DET-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Falha na outbox provoca rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| ADR-001-COD-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Integração | Transação atual de changeStatus (update, history, estoque) | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-COD-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Integração | Máquina de estados que valida transições | CODIGO | src/modules/orders/order.status.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Trigger no banco | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Worker dentro da instância da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa adiada | Múltiplos workers em paralelo | TRANSCRICAO | [09:13] Diego |
| ADR-002-DET-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Detalhe | Entry-point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-DET-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Detalhe | PrismaClient separado por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-DET-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Ordering por order_id apenas com single-worker | TRANSCRICAO | [09:12] Diego |
| ADR-002-DET-04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Detalhe | Lógica do worker em src/modules/webhooks/webhook.worker.ts | TRANSCRICAO | [09:28] Bruno |
| ADR-002-COD-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Integração | Esqueleto de bootstrap e shutdown copiado de server.ts | CODIGO | src/server.ts |
| ADR-002-COD-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Integração | Fábrica createPrismaClient | CODIGO | src/config/database.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Três tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Marcar failed na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DET-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Detalhe | DLQ em tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| ADR-003-DET-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Detalhe | Replay manual via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| ADR-003-DET-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Restrição | Replay exige ADMIN e loga o executor | TRANSCRICAO | [09:36] Sofia |
| ADR-003-DET-04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Detalhe | Timeout de 10 s conta como falha | TRANSCRICAO | [09:42] Diego |
| ADR-003-DET-05 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Detalhe | Cliente já teve indisponibilidade de 2 h em manutenção | TRANSCRICAO | [09:16] Diego |
| ADR-003-COD-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Integração | requireRole reaproveitado no replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-DET-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Detalhe | Assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-DET-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Detalhe | Algoritmo SHA-256 (padrão de mercado) | TRANSCRICAO | [09:20] Sofia |
| ADR-004-DET-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Detalhe | Tabela com url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-DET-04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Detalhe | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-DET-05 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Detalhe | X-Timestamp para detecção de replay attack | TRANSCRICAO | [09:44] Diego |
| ADR-004-DET-06 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | URL https validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-DET-07 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Limite de 64 KB | TRANSCRICAO | [09:24] Diego |
| ADR-004-DET-08 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Dependência | Revisão de segurança de Sofia | TRANSCRICAO | [09:46] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once (coordenação dos dois lados) | TRANSCRICAO | [09:25] Diego |
| ADR-005-DET-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Detalhe | UUID gerado ao entrar na outbox, único por evento | TRANSCRICAO | [09:25] Diego |
| ADR-005-DET-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Detalhe | X-Webhook-Id para clientes com vários endpoints | TRANSCRICAO | [09:44] Sofia |
| ADR-005-DET-03 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Dependência | Documentação destacada no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-005-CON-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-DET-04 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Contexto | Cliente deve estar preparado para receber duplicado | TRANSCRICAO | [09:24] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa descartada | Infra própria de erro/log ("não vamos botar nada novo") | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DET-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Detalhe | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-DET-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Detalhe | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-DET-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Detalhe | Seguir AppError/InsufficientStockError/InvalidStatusTransitionError | TRANSCRICAO | [09:28] Bruno |
| ADR-006-DET-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Detalhe | Error middleware já trata AppError, Zod e Prisma | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DET-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Detalhe | CRUD com qualquer role; replay com requireRole ADMIN | TRANSCRICAO | [09:36] Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Layout de módulo por domínio | CODIGO | src/modules/orders/order.routes.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Classe base AppError (statusCode, errorCode, details) | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Erros de domínio com código SCREAMING_SNAKE_CASE | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Error middleware centralizado (AppError, ZodError, P2002/P2025) | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Logger Pino com redaction | CODIGO | src/shared/logger/index.ts |
| ADR-006-COD-06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | authenticate e requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD-07 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Middleware validate com Zod | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-COD-08 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | buildControllers e buildApiRouter | CODIGO | src/routes/index.ts |
| ADR-006-COD-09 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Variáveis de ambiente validadas com Zod | CODIGO | src/config/env.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-DET-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Detalhe | Formato do payload e exclusão de items | TRANSCRICAO | [09:43] Diego |
| ADR-007-DET-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Detalhe | Limite de 64 KB validado na inserção | TRANSCRICAO | [09:24] Larissa |
| ADR-008 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao.md | Decisão | Filtro por status aplicado na inserção; sem assinante, não insere | TRANSCRICAO | [09:34] Bruno |
| ADR-008-ALT-01 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao.md | Alternativa descartada | Filtrar na hora de enviar | TRANSCRICAO | [09:34] Diego |
| ADR-008-DET-01 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao.md | Detalhe | Filtro é a lista de status que o webhook quer ouvir | TRANSCRICAO | [09:33] Marcos |
| ADR-008-DET-02 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao.md | Detalhe | Cliente pode ter vários webhooks | TRANSCRICAO | [09:44] Sofia |
| ADR-008-COD-01 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao.md | Integração | Enum OrderStatus define os status assináveis | CODIGO | prisma/schema.prisma |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent dentro do $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Nenhuma transição tem PENDING como destino; criar pedido não gera evento | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Controller repassa erros do service via next(err), sem mudança | CODIGO | src/modules/orders/order.controller.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Novas classes estendem AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Novas classes de erro ao lado de InvalidStatusTransitionError; NotFoundError fixa código NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Barrel de exportação dos erros | CODIGO | src/shared/errors/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Error middleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | authenticate em todas as rotas; requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | validate({ body, params, query }) | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-10 | docs/FDD.md | Integração | redactPaths ganha *.secret e *.previousSecret | CODIGO | src/shared/logger/index.ts |
| FDD-INT-11 | docs/FDD.md | Integração | req.id / X-Request-Id vira correlationId da outbox | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-12 | docs/FDD.md | Integração | paginated() para listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-13 | docs/FDD.md | Integração | buildControllers instancia o módulo; prefixo /api/v1 | CODIGO | src/app.ts |
| FDD-INT-14 | docs/FDD.md | Integração | buildApiRouter registra /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-15 | docs/FDD.md | Integração | worker.ts copia bootstrap, SIGINT/SIGTERM e $disconnect | CODIGO | src/server.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Worker importa prisma; instância própria por processo | CODIGO | src/config/database.ts |
| FDD-INT-17 | docs/FDD.md | Integração | Variáveis WEBHOOK_* com default | CODIGO | src/config/env.ts |
| FDD-INT-18 | docs/FDD.md | Integração | Novos modelos e enum; migration aditiva | CODIGO | prisma/schema.prisma |
| FDD-INT-19 | docs/FDD.md | Integração | Scripts worker e worker:start espelhando dev/start | CODIGO | package.json |
| FDD-INT-20 | docs/FDD.md | Integração | beforeEach limpa as novas tabelas | CODIGO | tests/setup.ts |
| FDD-MD-01 | docs/FDD.md | Modelo de dados | Status da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-MD-02 | docs/FDD.md | Modelo de dados | Endpoint armazena url, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-MD-03 | docs/FDD.md | Modelo de dados | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-MD-04 | docs/FDD.md | Modelo de dados | UUID em todas as tabelas | TRANSCRICAO | [09:51] Larissa |
| FDD-MD-05 | docs/FDD.md | Modelo de dados | Secret rotacionável com antiga válida por 24 h (previousSecret) | TRANSCRICAO | [09:21] Sofia |
| FDD-MD-06 | docs/FDD.md | Modelo de dados | orderId sem FK: pedidos CANCELLED podem ser apagados | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação; falha = rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Sem webhook assinante, nada é inserido | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Worker busca pendentes mais antigos a cada 2 s, processa, marca | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Processamento em ordem de created_at | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Retry em backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Mover para DLQ e recolocar na outbox como pendente no replay | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Rotação gera nova secret; antiga válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Mesmo X-Event-Id em todas as tentativas | TRANSCRICAO | [09:25] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks com url, status, customerId; secret devolvida | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks lista por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Headers da entrega: X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Body do evento (event_type order.status_changed, campos, sem items) | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries com sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Replay exige ADMIN; CRUD aceita qualquer role | TRANSCRICAO | [09:36] Sofia |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Cliente responde 2xx em até 10 s; senão retentativa | TRANSCRICAO | [09:42] Diego |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Cliente com detalhes consulta GET /orders/:id | TRANSCRICAO | [09:43] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | Prefixo WEBHOOK_ em todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para URL não https | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT após 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED como motivo na DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-07 | docs/FDD.md | Erro | Erros de shape continuam VALIDATION_ERROR via validate | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERRO-08 | docs/FDD.md | Erro | 401/403 vêm de authenticate/requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-ERRO-09 | docs/FDD.md | Erro | Formato { error: { code, message, details } } | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s via AbortSignal | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Restart da API não afeta o worker | TRANSCRICAO | [09:11] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Batch pequeno e índices para picos | TRANSCRICAO | [09:08] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Sem fallback de e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Cliente fora do ar coberto por backoff até ~15 h | TRANSCRICAO | [09:17] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Shutdown gracioso no padrão de server.ts | CODIGO | src/server.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Pino já no projeto; nenhuma lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | correlationId propagado do X-Request-Id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Latência do PATCH já emitida em http_request.durationMs | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Replay logado com quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Nunca logar secret (vazamento em log já ocorreu) | TRANSCRICAO | [09:22] Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >= 20 (fetch, crypto nativos) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8 via docker-compose | CODIGO | docker-compose.yml |
| FDD-DEP-03 | docs/FDD.md | Dependência | Testes vitest + supertest no padrão existente | CODIGO | tests/orders.test.ts |
| FDD-ACEITE-01 | docs/FDD.md | Critério de aceite | Entrega em até 2 s após o commit | TRANSCRICAO | [09:10] Larissa |
| FDD-ACEITE-02 | docs/FDD.md | Critério de aceite | Falha na outbox mantém status, histórico e estoque inalterados | TRANSCRICAO | [09:40] Bruno |
| FDD-ACEITE-03 | docs/FDD.md | Critério de aceite | OPERATOR recebe 403 no replay; ADMIN recebe 202 | TRANSCRICAO | [09:36] Larissa |
| FDD-ACEITE-04 | docs/FDD.md | Critério de aceite | Revisão de segurança concluída | TRANSCRICAO | [09:46] Sofia |
| FDD-ACEITE-05 | docs/FDD.md | Critério de aceite | Secret aparece apenas na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-PLANO-01 | docs/FDD.md | Plano | Três sprints: outbox/DLQ, worker/retry, CRUD/deliveries/HMAC | TRANSCRICAO | [09:46] Larissa |
| FDD-DI-01 | docs/FDD.md | Decisão de implementação | customerId no body do POST e na query do GET | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-DI-02 | docs/FDD.md | Decisão de implementação | Rotas sob /api/v1 | CODIGO | src/app.ts |
| FDD-DI-03 | docs/FDD.md | Decisão de implementação | Paginação page/pageSize | CODIGO | src/shared/http/response.ts |
| FDD-DI-04 | docs/FDD.md | Decisão de implementação | 1 envio + 5 retentativas (soma ≈ 15 h) | TRANSCRICAO | [09:17] Diego |
| FDD-DI-05 | docs/FDD.md | Decisão de implementação | Duas assinaturas em X-Signature durante o grace | TRANSCRICAO | [09:21] Sofia |
| FDD-DI-06 | docs/FDD.md | Decisão de implementação | Secret via node:crypto, sem dependência nova | TRANSCRICAO | [09:29] Bruno |
| FDD-DI-07 | docs/FDD.md | Decisão de implementação | fetch nativo do Node 20 | CODIGO | package.json |
| FDD-DI-08 | docs/FDD.md | Decisão de implementação | Batch de 50 e recuperação de PROCESSING após 60 s | TRANSCRICAO | [09:08] Diego |
| FDD-DI-09 | docs/FDD.md | Decisão de implementação | Payload > 64 KB aborta a mudança de status | TRANSCRICAO | [09:24] Larissa |
| FDD-DI-10 | docs/FDD.md | Decisão de implementação | Replay preserva o X-Event-Id original | TRANSCRICAO | [09:25] Diego |
| FDD-DI-11 | docs/FDD.md | Decisão de implementação | subtotal_cents e discount_cents no snapshot (campos de Order) | CODIGO | prisma/schema.prisma |
| FDD-DI-12 | docs/FDD.md | Decisão de implementação | Classes de erro em http-errors.ts | CODIGO | src/shared/errors/http-errors.ts |
| FDD-DI-13 | docs/FDD.md | Decisão de implementação | Defaults WEBHOOK_* em env.ts | CODIGO | src/config/env.ts |
| FDD-DI-14 | docs/FDD.md | Decisão de implementação | responseBody truncado para caber em deliveries | TRANSCRICAO | [09:34] Marcos |
