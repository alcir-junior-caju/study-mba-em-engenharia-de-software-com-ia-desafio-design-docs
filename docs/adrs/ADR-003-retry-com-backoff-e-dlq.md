# ADR-003 — Retry com backoff exponencial (5 tentativas) e Dead Letter Queue em tabela separada

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM), Sofia (Segurança) |
| **Fonte** | `TRANSCRICAO.md` [09:14] a [09:18], [09:35] a [09:36], [09:42] |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Contexto

O worker fará chamadas HTTP para endpoints fora da nossa infraestrutura. Clientes podem estar offline, lentos ou em manutenção planejada; já houve cliente com indisponibilidade de duas horas ([09:16] Diego). É preciso definir o que fazer quando a entrega falha, quantas vezes tentar, e o que acontece quando as tentativas se esgotam ([09:14] Larissa).

## Decisão

### Política de retry

- Uma entrega é considerada **falha** quando o cliente responde erro ou quando o HTTP call ultrapassa o **timeout de 10 segundos** ([09:42] Diego).
- Em caso de falha, o evento é reagendado com **backoff exponencial**, no máximo **5 tentativas**, com a progressão **1 min, 5 min, 30 min, 2 h, 12 h** ([09:17] Diego, [09:17] Larissa). A janela total entre a primeira falha e a última tentativa é de quase 15 horas ([09:17] Diego), considerada aceitável pelo produto ([09:17] Marcos).

### Dead Letter Queue (DLQ)

- Esgotadas as 5 tentativas, o evento é movido para uma **tabela separada `webhook_dead_letter`**, contendo o payload, o motivo da falha e o timestamp ([09:18] Diego).
- O reprocessamento é **manual**, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego).
- O endpoint de replay exige **role `ADMIN`** no JWT, reaproveitando o `requireRole` de `src/middlewares/auth.middleware.ts`, e deve **logar quem fez o replay** para auditoria ([09:36] Sofia, [09:36] Larissa).

## Alternativas Consideradas

### A. Três tentativas, "mais agressivo"

Proposta de Bruno ([09:16]).

- Descartada ([09:16] Diego).
- Trade-off que motivou o descarte: três tentativas se esgotariam em cerca de 30 minutos; uma indisponibilidade matinal ou uma manutenção planejada de duas horas do cliente levaria eventos à DLQ desnecessariamente.

### B. Retry indefinido com backoff

Nunca desistir de entregar.

- Descartada ([09:15] Diego).
- Trade-off que motivou o descarte: evento fica "pendurado para sempre" se o cliente sumiu; a outbox acumularia eventos impossíveis de entregar sem um ponto de corte.

### C. DLQ como status `failed` na própria outbox

Em vez de tabela separada, marcar a linha como falha permanente na `webhook_outbox` ([09:17] Larissa levantou a opção).

- Descartada ([09:18] Diego).
- Trade-off que motivou o descarte: polui a leitura da outbox principal (o worker consulta pendentes a cada 2 s); tabela separada é mais limpa e serve como evidência para debug e reprocessamento.

## Consequências

### Positivas

- **Tolerância a indisponibilidades reais** dos clientes (até ~15 h) sem perder eventos.
- **Ponto de corte claro**: nenhum evento fica eternamente pendente; a DLQ é o destino final e auditável.
- **Recuperação operacional** por replay manual, protegido por role `ADMIN` e com trilha de auditoria.
- **Outbox enxuta**: a tabela consultada pelo worker contém apenas eventos vivos.

### Negativas / trade-offs assumidos

- **Entrega pode demorar até ~15 h** em cenário de falha persistente; o cliente não é avisado proativamente, pois notificação por e-mail ficou para fase futura ([09:37] Larissa).
- **Replay é manual**: depende de alguém com role `ADMIN` detectar e acionar o endpoint; não há automação nem alerta nesta fase.
- **Retries ampliam a duplicidade possível**: uma entrega que o cliente processou mas respondeu com erro (ou timeout) será reenviada, o que reforça a necessidade de dedup por `X-Event-Id` ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
- **Mais uma tabela e mais um endpoint** para modelar, migrar e manter.
