# ADR-005 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM) |
| **Fonte** | `TRANSCRICAO.md` [09:24] a [09:26], [09:44] a [09:45] |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) |

## Contexto

Com outbox, worker em polling e política de retry, existem janelas em que o mesmo evento pode ser enviado mais de uma vez: por exemplo, o cliente processa o request mas responde erro ou estoura o timeout de 10 s, e o worker reagenda a entrega ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)). É preciso definir qual semântica de entrega a plataforma garante e como o cliente lida com duplicatas ([09:24] Diego).

## Decisão

- A plataforma garante **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e **deve estar preparado** para isso ([09:24] Diego).
- Cada evento recebe um **UUID gerado no momento em que entra na outbox**, único por evento. Esse identificador é enviado no header **`X-Event-Id`** em todas as tentativas de entrega do mesmo evento. O cliente faz a deduplicação pelo `event_id` do lado dele ([09:25] Diego).
- O request também carrega **`X-Webhook-Id`** com o id do cadastro de webhook, para que clientes com vários endpoints saibam qual cadastro originou o envio ([09:44] Sofia, [09:45] Diego).
- O comportamento será **documentado com destaque no portal de desenvolvedor** para os clientes ([09:26] Marcos).

## Alternativas Consideradas

### A. Exactly-once

Garantir que cada evento seja entregue exatamente uma vez.

- Descartada ([09:25] Diego).
- Trade-off que motivou o descarte: exigiria coordenação dos dois lados (protocolo de confirmação/idempotência negociado com cada cliente), tornando a solução muito mais complexa. At-least-once com `event_id` "resolve 99% dos casos" e é o padrão de mercado adotado por Stripe e GitHub.

### B. At-most-once (sem retry)

Enviar uma única vez e nunca reenviar.

- Não defendida na reunião; incompatível com a política de retry já decidida ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)).
- Trade-off: elimina duplicatas, mas perde eventos sempre que o cliente estiver indisponível, o que contraria o motivo da feature.

## Consequências

### Positivas

- **Nenhum evento é perdido** por indisponibilidade transitória do cliente, dentro da janela de retry.
- **Deduplicação simples e barata**: um único identificador estável por evento, em header, independente do conteúdo do payload.
- **Alinhado ao mercado**, reduzindo atrito de integração para clientes que já consomem webhooks de outros provedores.

### Negativas / trade-offs assumidos

- **Responsabilidade transferida para o cliente**, como Sofia apontou ([09:25]): ele precisa persistir os `event_id` já processados e ignorar repetições. Cliente que não dedupica pode processar o mesmo evento duas vezes.
- **Dependência de documentação**: a garantia só funciona se o portal deixar o contrato explícito ([09:26] Marcos).
- **Sem garantia de ordering global**: duplicatas somadas ao ordering apenas por `order_id` ([ADR-002](ADR-002-worker-separado-em-polling.md)) exigem que o cliente trate também eventos fora de ordem em cenários de retry.
