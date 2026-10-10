# ADR-007 — Payload do evento renderizado e armazenado como snapshot no momento da inserção na outbox

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos) |
| **Fonte** | `TRANSCRICAO.md` [09:23] a [09:24], [09:43] a [09:44], [09:51] a [09:52] |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) |

## Contexto

A linha da `webhook_outbox` pode guardar apenas a referência ao pedido (`order_id`) e montar o JSON no momento do envio, ou guardar o JSON já renderizado. A dúvida foi levantada por Bruno ao final da reunião ([09:51]). A decisão interage com o retry: uma entrega pode acontecer até ~15 h depois da mudança de status ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), período em que o pedido pode ter mudado de novo.

O formato do payload já havia sido definido: JSON com `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`; **sem `items`**, para não inflar; o cliente busca detalhes em `GET /orders/:id` ([09:43] Diego). O limite de tamanho é 64 KB, com erro se ultrapassar ([09:24]).

## Decisão

- O payload é **renderizado na inserção** e armazenado na linha da outbox como **snapshot** do estado do pedido no instante da mudança de status ([09:52] Larissa, Diego e Bruno).
- O worker envia o snapshot tal como armazenado, em todas as tentativas de entrega do mesmo evento. Isso também garante que o HMAC ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)) é calculado sobre bytes estáveis.
- A validação do limite de 64 KB acontece na inserção, pois é o momento em que o tamanho do payload é conhecido.
- O snapshot é propositalmente enxuto (sem `items`), conforme o formato definido em [09:43].

## Alternativas Consideradas

### A. Guardar apenas `order_id` e renderizar no envio

Proposta explicitada por Bruno como a outra opção ([09:51]).

- Descartada ([09:52] Larissa).
- Trade-off que motivou o descarte: se o pedido mudar entre a inserção e o envio (ou entre retries), o cliente receberia um evento `from_status → to_status` cujo corpo reflete um estado posterior, "um caso esquisito". Também obrigaria o worker a consultar `orders` a cada envio.

## Consequências

### Positivas

- **Fidelidade histórica**: o evento reflete exatamente o estado de quando o status mudou, mesmo se entregue horas depois ([09:52] Larissa).
- **Worker mais simples e barato**: lê e envia, sem joins em `orders` e `customers` no caminho quente de 2 s.
- **Assinatura e dedup estáveis**: mesmos bytes em todas as tentativas; o `X-Event-Id` identifica um conteúdo imutável.
- **DLQ autossuficiente**: a `webhook_dead_letter` guarda o payload exato que falhou, útil para debug e replay ([09:18] Diego).

### Negativas / trade-offs assumidos

- **Mais bytes por linha** na outbox e na DLQ, o que torna o arquivamento de linhas entregues (fora de escopo nesta fase, [09:08]) mais relevante a médio prazo.
- **Mudanças no formato do payload não retroagem**: eventos já enfileirados saem no formato antigo; versionar o `event_type` ou o schema pode ser necessário no futuro.
- **Trabalho extra dentro da transação de status**: montar o JSON e validar o tamanho acontece dentro de `changeStatus`, que já é considerada pesada ([09:04] Bruno).
