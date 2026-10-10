# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Em revisão |
| **Data** | 2026-10-10 |
| **Autor** | Marcos (Product Manager) |
| **Revisores** | Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto da feature

O Order Management System (OMS) passará a **notificar clientes B2B em tempo real**, via webhooks HTTP, sempre que o status de um pedido deles mudar. O cliente cadastra uma URL HTTPS, escolhe quais status quer receber e passa a receber um `POST` assinado a cada mudança, em vez de consultar a API repetidamente.

A feature nasce de um pedido formal de três clientes (Atlas Comercial, MaxDistribuição e Nova Cargo) recebido na semana anterior à reunião técnica ([09:00] Marcos). O fluxo é exclusivamente **de saída**: a plataforma envia, o cliente recebe ([09:02] Marcos). A solução técnica foi fechada na reunião registrada em `TRANSCRICAO.md` e está detalhada no [RFC](RFC.md), no [FDD](FDD.md) e nos [ADRs](adrs/README.md).

## 2. Problema e motivação

**Problema.** Hoje o cliente só descobre que um pedido mudou de status fazendo polling em `GET /orders` "de tempos em tempos". Isso torna a integração **lenta e cara** para ele e gera carga desnecessária na nossa API ([09:00] Marcos). O OMS não tem nenhum mecanismo de notificação externa, evento ou fila; esse vazio é o que a feature preenche.

**Motivação de negócio.** A Atlas Comercial sinalizou que, sem essa entrega até o fim do trimestre, pode migrar para o concorrente ([09:00] Marcos). Os três clientes definem "tempo real" como **qualquer latência abaixo de 10 segundos**; o que importa para eles é não precisar atualizar manualmente ([09:02] Marcos). A Atlas espera a entrega para **fim de novembro** ([09:45] Marcos).

**Motivação técnica.** A mudança de status já é uma transação consistente no `OrderService` (atualiza pedido, histórico e estoque). A feature precisa se encaixar nessa transação sem torná-la refém da disponibilidade de sistemas externos ([09:04] Bruno). Detalhes no [FDD, seção 1](FDD.md#1-contexto-e-motivação-técnica).

## 3. Público-alvo e cenários de uso

### Público-alvo

| Perfil | Quem é | O que precisa |
| --- | --- | --- |
| **Cliente B2B integrador** | Times técnicos de Atlas Comercial, MaxDistribuição, Nova Cargo (e futuros clientes) | Receber mudanças de status em até 10 s, validar que o envio é legítimo, consultar o que foi entregue |
| **Usuário da plataforma que representa o cliente** | Usuários autenticados com JWT do OMS que atuam em nome do cliente ([09:32] Marcos) | Cadastrar, editar, listar e remover webhooks por cliente via API |
| **Administrador da plataforma** | Usuários com role `ADMIN` | Reprocessar eventos que esgotaram as tentativas (DLQ) com auditoria |
| **Equipe de engenharia/operação** | Times de Pedidos e Plataforma | Operar o worker, acompanhar entregas e falhas pelos logs |

### Cenários de uso

1. **Integração inicial.** O time da Atlas cadastra `https://integracao.atlascomercial.com.br/oms/webhooks` assinando `PAID`, `SHIPPED`, `DELIVERED` e `CANCELLED`, recebe a secret na resposta e configura a verificação HMAC do lado deles.
2. **Pedido avança.** Um operador muda o pedido `ORD-000123` de `PROCESSING` para `SHIPPED`. Em até ~2 s a Atlas recebe o evento `order.status_changed` com `from_status`, `to_status`, `order_number` e `total_cents`; se quiser os itens, consulta `GET /orders/:id` ([09:43] Diego).
3. **Cliente em manutenção.** O endpoint da MaxDistribuição fica fora do ar por duas horas. A plataforma retenta com intervalos crescentes (1 min, 5 min, 30 min, 2 h, 12 h) e entrega quando o cliente volta; o cliente dedupica eventuais repetições pelo `X-Event-Id` ([09:16] Diego, [09:25] Diego).
4. **Cliente sumiu.** Após esgotar as tentativas, o evento vai para a DLQ. Um `ADMIN` da plataforma aciona o replay quando o cliente confirma que voltou; a ação fica registrada com quem a fez ([09:18] Diego, [09:36] Sofia).
5. **Auditoria pelo cliente.** A Nova Cargo consulta `GET /webhooks/:id/deliveries` para ver as últimas 100 entregas, com sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos).
6. **Rotação de secret.** Depois de um incidente de vazamento em log, a Atlas pede uma nova secret; a antiga continua válida por 24 h enquanto eles migram ([09:21] Sofia, [09:22] Diego).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| O1 | Notificar em "tempo real" | Tempo entre o commit da mudança de status e a primeira tentativa de entrega (`webhook_outbox_pending_age_ms`) | **< 10 s para 100% dos eventos**; esperado ~2 s pelo polling | [09:02] Marcos, [09:10] Larissa |
| O2 | Não perder eventos | Eventos entregues (2xx) dentro da janela de retry ÷ eventos gerados | **Nenhum evento perdido**: todo evento termina em `DELIVERED` ou na DLQ, nunca desaparece | [09:06] Diego, [09:15] Diego |
| O3 | Eliminar o polling dos clientes | Volume de `GET /orders` originado pelos três clientes | Queda após a adoção; valor-alvo a combinar com cada cliente (não definido na reunião) | [09:00] Marcos |
| O4 | Reter a Atlas | Entrega em produção | **Até o fim de novembro**, em três sprints incluindo revisão de segurança | [09:45] Marcos, [09:46] Larissa |
| O5 | Não degradar o OMS | Latência de `PATCH /orders/:id/status` (`http_request.durationMs`) | Sem aumento perceptível após a integração; medir antes/depois | [09:04] Bruno |

## 5. Escopo

### Incluído

- Cadastro, edição, remoção e listagem de endpoints de webhook por cliente, com filtro de status por endpoint.
- Geração de secret pela plataforma e rotação com período de graça de 24 h.
- Envio de evento `order.status_changed` a cada mudança de status assinada, com assinatura HMAC-SHA256 e identificador único por evento.
- Retentativas automáticas com backoff e fila de eventos mortos (DLQ) com replay manual por administrador.
- Histórico de entregas consultável pelo cliente via API.
- Validação de URL HTTPS e limite de tamanho de payload.
- Documentação de integração no portal de desenvolvedor ([09:26] Marcos, [09:40] Marcos).

### Fora de escopo

| Item | Situação | Fonte |
| --- | --- | --- |
| **E-mail de alerta ao cliente** quando o webhook falha repetidamente | **Adiado** para a próxima fase, após medir o impacto | [09:37] Marcos e Larissa |
| **Rate limiting de envio** por cliente | **Adiado**: "observar e decidir depois" | [09:38] Diego, [09:39] Larissa |
| **Dashboard visual** para o cliente acompanhar seus webhooks | **Descartado** nesta feature; é projeto separado do time de frontend | [09:39] Marcos, [09:40] Larissa |
| **Webhooks inbound** (cliente enviando para a plataforma) | **Descartado**; apenas outbound | [09:02] Sofia e Marcos |
| **Arquivamento** de eventos já entregues | Fora do escopo desta feature | [09:08] Diego |
| **Garantia de ordering global** e múltiplos workers | Limitação conhecida; ordering apenas por pedido e com worker único | [09:13] Larissa, [09:14] Marcos |
| **Itens do pedido** no payload | Descartado para manter o payload enxuto; cliente consulta `GET /orders/:id` | [09:43] Diego |
| **Permissões diferenciadas** para o CRUD de webhooks | Qualquer usuário autenticado por enquanto; endurecer depois | [09:37] Sofia |

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RF-01 | O usuário autenticado deve cadastrar um webhook informando `customerId`, URL e lista de status desejados; a plataforma gera a secret e a devolve apenas na criação | [09:31] Marcos, [09:32] Larissa |
| RF-02 | O usuário deve poder editar URL, lista de status e ativar/desativar um webhook (`PATCH`) | [09:33] Bruno |
| RF-03 | O usuário deve poder remover um webhook (`DELETE`) | [09:33] Bruno |
| RF-04 | O usuário deve poder listar os webhooks de um cliente (`GET`) | [09:33] Bruno |
| RF-05 | Cada webhook deve ter sua própria lista de status assinados; eventos de status não assinados não são gerados para aquele webhook | [09:33] Marcos, [09:34] Bruno |
| RF-06 | A cada mudança de status de pedido, a plataforma deve enviar um `POST` JSON ao webhook com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e totais do pedido, sem itens | [09:43] Diego |
| RF-07 | Toda entrega deve carregar os headers `X-Event-Id`, `X-Signature` (HMAC-SHA256 do corpo), `X-Timestamp` e `X-Webhook-Id` | [09:20] Sofia, [09:44] Diego, [09:44] Sofia |
| RF-08 | Cada webhook deve ter secret única; o usuário deve poder rotacioná-la via API, com a secret antiga válida por 24 h | [09:21] Sofia |
| RF-09 | O cliente deve poder consultar o histórico das últimas entregas de um webhook, com sucesso/falha, payload, resposta e tempo de resposta (`GET /webhooks/:id/deliveries`) | [09:34] Marcos |
| RF-10 | Entregas que falham devem ser retentadas automaticamente com backoff 1 min / 5 min / 30 min / 2 h / 12 h; esgotadas as tentativas, o evento vai para uma DLQ com payload, motivo e timestamp | [09:17] Larissa, [09:18] Diego |
| RF-11 | Um usuário com role `ADMIN` deve poder reprocessar um evento da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`); a ação deve registrar quem a executou | [09:18] Diego, [09:36] Sofia e Larissa |
| RF-12 | A plataforma deve rejeitar, com erro de validação, URLs de webhook que não sejam HTTPS | [09:23] Sofia |
| RF-13 | A plataforma garante entrega at-least-once; o mesmo evento pode chegar mais de uma vez com o mesmo `X-Event-Id`, e o cliente é responsável por deduplicar | [09:24] Diego, [09:26] Larissa |
| RF-14 | O conteúdo do evento deve refletir o estado do pedido no momento da mudança de status, mesmo que a entrega ocorra horas depois | [09:52] Larissa |

## 7. Requisitos não funcionais

| ID | Requisito | Valor | Fonte |
| --- | --- | --- | --- |
| RNF-01 | Latência entre mudança de status e início da entrega | < 10 s (esperado ~2 s) | [09:02] Marcos, [09:09] Diego |
| RNF-02 | Consistência | Evento registrado na mesma transação da mudança de status; nunca existe status alterado sem evento, nem evento sem status alterado | [09:06] Diego, [09:40] Bruno |
| RNF-03 | Isolamento | Lentidão ou indisponibilidade do cliente não afeta a API de pedidos nem outros clientes | [09:04] Bruno, [09:11] Diego |
| RNF-04 | Timeout por entrega | 10 s | [09:42] Diego |
| RNF-05 | Tamanho máximo do payload | 64 KB; acima disso, erro | [09:24] Diego e Larissa |
| RNF-06 | Transporte | HTTPS obrigatório | [09:23] Sofia |
| RNF-07 | Autenticidade e integridade | HMAC-SHA256 sobre o corpo, secret por endpoint | [09:22] Sofia |
| RNF-08 | Janela de resiliência | Até ~15 h de indisponibilidade do cliente sem perda de evento | [09:17] Diego |
| RNF-09 | Auditoria | Replay de DLQ registra o usuário executor; nenhuma secret aparece em logs | [09:36] Sofia, [09:22] Diego |
| RNF-10 | Infraestrutura | Nenhum componente novo (sem Redis, broker ou serviço adicional); MySQL e stack atuais | [09:07] Diego |
| RNF-11 | Padrões de código | Módulo, erros (`WEBHOOK_*`), logger e middlewares seguem o padrão existente do projeto | [09:30] Larissa |
| RNF-12 | Ordering | Garantido por pedido enquanto houver worker único; sem garantia global | [09:13] Larissa |
| RNF-13 | Prazo | Três sprints, com dois dias úteis de revisão de segurança antes do deploy | [09:46] Larissa e Sofia |

## 8. Decisões e trade-offs principais

Cada decisão tem um ADR com contexto, alternativas e consequências. Resumo em nível de produto:

| Decisão | O que ganhamos | O que aceitamos | ADR |
| --- | --- | --- | --- |
| Outbox no MySQL, na mesma transação do status | Consistência total sem infraestrutura nova | Latência mínima de ~2 s; tabela que precisará de arquivamento no futuro | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2 s, worker único | Isolamento da API; latência dentro do requisito | Ordering só por pedido; escala horizontal fica para depois | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas com backoff e DLQ com replay manual | Tolerância a manutenções reais dos clientes; nada se perde | Entrega pode levar até ~15 h; replay depende de ação humana | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação com 24 h de graça | Cliente valida origem e integridade; vazamento tem raio limitado | Cliente precisa implementar a verificação | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Simplicidade e padrão de mercado (Stripe, GitHub) | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Velocidade e consistência | Mudança no caminho crítico de `changeStatus` | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Payload como snapshot da mudança | Evento fiel ao momento, mesmo entregue tarde | Mais dados por linha na outbox | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| Filtro de status aplicado na geração do evento | Outbox menor; só o que será entregue | Mudar o filtro não é retroativo | [ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Observação |
| --- | --- | --- |
| Transação de `changeStatus` em `src/modules/orders/order.service.ts` | Código | Ponto único de geração de eventos; alteração obrigatória | 
| Padrões existentes (`AppError`, error middleware, Pino, Zod, `requireRole`) | Código | Reaproveitados sem alteração estrutural ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)) |
| MySQL 8 + Prisma 5.22 | Infra | Novas tabelas via migration; nada novo a provisionar |
| Node ≥ 20 | Runtime | HTTP client e criptografia nativos; sem dependência nova |
| Revisão de segurança (Sofia) | Pessoas | Mínimo dois dias úteis antes do deploy ([09:46] Sofia) |
| Portal de desenvolvedor | Documentação | Marcos documenta contrato, HMAC e deduplicação para os clientes ([09:26], [09:40] Marcos) |
| Clientes B2B | Externo | Precisam expor endpoint HTTPS, verificar HMAC e deduplicar por `event_id` |
| Usuários que representam o cliente | Produto | Cadastro é feito com JWT do OMS, não com credencial do cliente ([09:32] Marcos) |

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R1 | Não entregar até fim de novembro e a Atlas migrar para o concorrente ([09:00] Marcos) | Média | Alto | Escopo cortado deliberadamente (sem e-mail, rate limit, dashboard); estimativa de três sprints já inclui a revisão de segurança; prazo confirmado com o cliente ([09:47] Marcos) |
| R2 | Cliente não dedupica e processa o mesmo evento duas vezes ([09:25] Sofia) | Média | Médio | Contrato at-least-once documentado com destaque no portal ([09:26] Marcos); `event_id` no header e no body |
| R3 | Vazamento de secret por parte do cliente (já ocorreu, [09:22] Diego) | Média | Alto | Secret única por endpoint; rotação com 24 h de graça; nunca exposta em consultas nem em logs |
| R4 | Eventos parados na DLQ sem ninguém perceber | Média | Médio | Log de alerta a cada evento morto; histórico de entregas visível ao cliente; alerta por e-mail avaliado na próxima fase ([09:37] Larissa) |
| R5 | Transação de mudança de status ficar mais lenta ([09:04] Bruno) | Média | Médio | Uma consulta e uma inserção a mais, com índices; medir latência do `PATCH` antes e depois (objetivo O5) |
| R6 | Cliente lento (até 10 s) atrasa entregas de outros clientes no mesmo ciclo, por o worker ser único | Média | Baixo | Lotes pequenos e ciclo de 2 s limitam o atraso; limitação documentada; evolução para múltiplos workers já mapeada ([09:13] Diego) |
| R7 | Outbox cresce indefinidamente sem arquivamento ([09:08] Diego) | Alta (longo prazo) | Baixo (curto prazo) | Índices mantêm o polling barato; arquivamento registrado como questão em aberto no RFC |

## 11. Critérios de aceitação

A feature está pronta quando:

1. Um usuário autenticado consegue cadastrar, listar, editar e remover webhooks de um cliente via API, e a secret é exibida apenas na criação e na rotação.
2. Cadastro com URL `http://` é rejeitado com erro de validação.
3. Ao mudar o status de um pedido para um status assinado, o cliente recebe o `POST` em menos de 10 s, com os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e o payload definido em RF-06.
4. Status não assinados não geram entrega.
5. Se a mudança de status falhar, nenhum evento é enviado; se o evento não puder ser registrado, o status não muda.
6. Endpoint indisponível recebe o evento quando volta, dentro da sequência 1 min / 5 min / 30 min / 2 h / 12 h, sempre com o mesmo `X-Event-Id`.
7. Após esgotar as tentativas, o evento aparece na DLQ; um `ADMIN` consegue reprocessá-lo e a ação fica registrada com o usuário; um usuário sem role `ADMIN` recebe `403`.
8. O cliente consulta as últimas entregas com sucesso/falha, payload, resposta e tempo de resposta.
9. A rotação de secret mantém a antiga válida por 24 h e o cliente continua validando assinaturas durante a transição.
10. Nenhuma secret aparece em logs da aplicação.
11. A latência de `PATCH /orders/:id/status` não aumenta perceptivelmente.
12. A revisão de segurança de Sofia está concluída e sem pendências bloqueantes.
13. O portal de desenvolvedor documenta o contrato, a verificação HMAC e a deduplicação.

Os critérios técnicos detalhados (19 itens testáveis) estão no [FDD, seção 12](FDD.md#12-critérios-de-aceite-técnicos).

## 12. Estratégia de testes e validação

| Camada | O que valida | Como |
| --- | --- | --- |
| **Unitários** | Backoff, assinatura HMAC (inclusive com duas secrets no grace), filtro de status, limite de 64 KB, montagem do snapshot | `vitest`, já configurado em `vitest.config.ts` |
| **Integração da API** | CRUD, deliveries, rotação, replay com e sem role `ADMIN`, erros `WEBHOOK_*`, secret nunca exposta | `supertest` contra `buildApp`, no padrão de `tests/orders.test.ts`; `tests/setup.ts` passa a limpar as novas tabelas |
| **Integração do outbox** | `PATCH /orders/:id/status` cria linhas na outbox apenas para webhooks assinantes; falha na outbox faz rollback de status, histórico e estoque | Mesmos testes de pedidos, verificando `webhook_outbox` via Prisma |
| **Ponta a ponta do worker** | Entrega em até 2 s, headers, retry em `503`, timeout em 10 s, DLQ na 6ª falha, ordering por pedido | Servidor HTTP local de teste recebendo os `POST`; relógio controlado para os backoffs longos |
| **Segurança** | Geração de secret, HMAC, rotação, redaction de logs, autorização do replay | Revisão manual de Sofia (≥ 2 dias úteis, [09:46]) mais testes automatizados de autorização |
| **Regressão** | Testes existentes de pedidos e autenticação continuam passando | `npm test` no pipeline |
| **Validação com clientes** | Contrato compreensível e verificação HMAC implementável | Documentação no portal revisada com os três clientes; Marcos atualiza os clientes sobre o prazo ([09:49] Marcos) |
| **Pós-deploy** | Objetivos O1, O2 e O5 | Acompanhar `webhook_outbox_pending_age_ms`, taxa de sucesso e latência do `PATCH` pelos logs estruturados |
