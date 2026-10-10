# ADR-004 — Assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data** | 2026-10-10 |
| **Decisores** | Sofia (Segurança), Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos) |
| **Fonte** | `TRANSCRICAO.md` [09:19] a [09:23], [09:31], [09:44], [09:46] |
| **Relacionados** | [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) |

## Contexto

Os webhooks expõem dados de pedidos a endpoints fora da nossa infraestrutura. O cliente precisa conseguir validar que a requisição veio realmente de nós e que ninguém adulterou o payload no caminho ([09:19] Sofia). Já houve caso de cliente que vazou uma secret em log da própria aplicação ([09:22] Diego), o que mostra que o raio de impacto de um vazamento precisa ser contido.

## Decisão

- Cada request de webhook carrega o header **`X-Signature`** com o **HMAC-SHA256** calculado sobre o corpo do request, usando uma secret compartilhada entre a plataforma e o cliente ([09:20] Sofia, [09:22] Sofia).
- A **secret é única por endpoint de webhook cadastrado**, nunca global da plataforma: "se vaza uma, vaza tudo" ([09:21] Sofia). A tabela de configuração do webhook armazena `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno).
- A secret é **gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
- A secret é **rotacionável** por endpoint da API. Ao rotacionar, a secret antiga permanece válida por **24 horas** em paralelo com a nova, para o cliente migrar seus sistemas; depois disso, a antiga expira ([09:21] Sofia).
- O request também carrega **`X-Timestamp`** com o instante do envio, para que o cliente possa detectar replay attack se quiser ([09:44] Diego).
- Complementos de segurança que não são decisões arquiteturais, mas fazem parte do pacote: URL obrigatoriamente **HTTPS**, validada no schema Zod do módulo ([09:23] Sofia); **limite de 64 KB** de payload, com erro caso ultrapasse ([09:24] Diego e Larissa).
- A implementação de HMAC e de geração de secret passará por **revisão de segurança de Sofia** (pelo menos dois dias úteis) antes do deploy ([09:46] Sofia).

## Alternativas Consideradas

### A. Secret global da plataforma

Uma única secret compartilhada com todos os clientes.

- Descartada ([09:21] Sofia).
- Trade-off que motivou o descarte: um único vazamento comprometeria a autenticidade dos webhooks de todos os clientes; rotação seria um evento global e disruptivo.

### B. Rotação sem período de graça (troca imediata)

Ao rotacionar, invalidar a secret antiga na hora.

- Descartada implicitamente pela decisão de grace period ([09:21] Sofia).
- Trade-off: simplifica a verificação (uma secret por vez), mas força o cliente a trocar a secret em todos os seus sistemas no mesmo instante, com risco de rejeitar entregas legítimas durante a migração.

### C. Sem assinatura, confiando apenas em TLS

Alternativa plausível, não defendida na reunião.

- Trade-off: TLS protege o transporte, mas não prova ao cliente que a origem é a nossa plataforma nem permite verificar integridade fim a fim após proxies/gateways do próprio cliente. Não atende o requisito colocado por Sofia em [09:19].

## Consequências

### Positivas

- **Autenticidade e integridade verificáveis** pelo cliente com algoritmo padrão de mercado, com biblioteca disponível em qualquer stack ([09:20] Sofia).
- **Raio de impacto contido**: vazamento afeta apenas um endpoint de um cliente.
- **Rotação sem downtime** graças ao grace period de 24 h.
- **Defesa opcional contra replay** via `X-Timestamp`.

### Negativas / trade-offs assumidos

- **Verificação de assinatura durante o grace period é dupla**: o cliente (e nossa documentação) precisa lidar com duas secrets válidas por até 24 h; o modelo de dados precisa guardar secret atual, secret anterior e instante de expiração.
- **Responsabilidade do cliente**: ele precisa implementar a verificação HMAC; cliente que ignora o header perde a proteção.
- **Mais superfície de API**: endpoint de rotação, geração segura de secret e armazenamento seguro, tudo sujeito à revisão de segurança antes do deploy ([09:46] Sofia).
- **Custo de prazo**: a revisão de segurança está embutida na estimativa de três sprints ([09:47] Larissa).
