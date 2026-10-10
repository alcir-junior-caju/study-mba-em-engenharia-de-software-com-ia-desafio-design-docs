# Da Reunião ao Documento: Design Docs Gerados por IA

> Entrega do desafio de design docs com IA. O enunciado original está preservado em [`docs/ENUNCIADO.md`](docs/ENUNCIADO.md).

## Sobre o desafio

Uma empresa que opera um Order Management System (OMS) em produção decidiu, em uma reunião de 55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, como construir um Sistema de Webhooks de Notificação de Pedidos. Nada foi registrado além da transcrição da call. O desafio foi transformar essa transcrição, cruzada com o código real da aplicação (Node.js, TypeScript, Express, Prisma, MySQL), em um pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A regra central é que nada pode ser inventado. Cada requisito, decisão e restrição precisa apontar para uma fala da transcrição ou para um arquivo do código. Identificar o que foi descartado ou adiado na reunião (e mantê-lo fora dos requisitos) vale tanto quanto identificar o que entra. O código da aplicação não foi alterado: ele serviu apenas de contexto e referência.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (CLI, modelo Claude Fable 5.1) | Única ferramenta de IA usada. Leu o repositório e a transcrição diretamente do disco, mapeou os padrões do código, produziu todos os documentos e rodou os scripts de verificação. |
| **Scripts shell gerados pela IA** (`grep`, `awk`, `find`) | Verificação automática a cada documento: existência de arquivos citados, validade de timestamps, links internos, contagens dos critérios de aceite. |

## Workflow adotado

Segui a ordem sugerida pelo enunciado, com um ciclo fixo por documento: **ler as fontes → produzir → verificar automaticamente → revisar**.

1. **Contextualização.** Pedi à IA uma análise do enunciado e um inventário do repositório antes de escrever qualquer coisa. Ela listou os seis entregáveis com seus critérios, confirmou que a transcrição tinha 155 falas no formato `[hh:mm] Nome`, localizou os ganchos citados no enunciado (`changeStatus`, `requireRole`, `AppError`, error middleware, Pino) e apontou duas ressalvas: o diretório não era um repositório git e o `docs/adrs/README.md` do boilerplate sugeria uma convenção de nome diferente da exigida.
2. **Leitura completa das fontes.** A IA leu a transcrição inteira e os arquivos de código relevantes (service, status machine, erros, middlewares, logger, schema Prisma, server, app, routes, testes) antes do primeiro documento.
3. **ADRs primeiro.** Oito ADRs em formato MADR: as seis decisões principais do enunciado mais duas decisões secundárias que tiveram alternativa real discutida na reunião (snapshot do payload e filtragem na inserção). Cada afirmação saiu com timestamp e falante, o que depois alimentou o tracker.
4. **RFC.** Conciso, em nível de arquitetura, apontando para os ADRs em vez de repetir detalhe. As alternativas descartadas e as questões em aberto da reunião ficaram aqui.
5. **FDD.** O documento mais longo. As lacunas que a reunião deixou foram fechadas seguindo o padrão do código e isoladas em uma seção própria ("Decisões de implementação tomadas neste FDD"), cada uma com a justificativa.
6. **PRD.** Produzido por último entre os grandes documentos, como consolidação.
7. **Tracker.** Varredura dos quatro documentos prontos, gerando 247 linhas.
8. **README.** Este documento.

A interação com a IA foi em turnos curtos e dirigidos ("comece pelos ADRs", "agora o RFC"), porque o enunciado já continha a especificação detalhada de cada documento e foi carregado como contexto no primeiro turno. O trabalho de direção ficou em revisar o que saía e manter a disciplina de rastreabilidade.

## Prompts customizados

**Prompt 1: contextualização antes de produzir.** Pedi análise e inventário, não geração.

```text
faça uma analise no @README.md entenda o desafio, e verifique se nesse
repositório temos tudo que precisamos para o desafio
```

**Prompt 2: ADRs como ponto de partida, com leitura obrigatória da fonte.**

```text
agora leia a TRANSCRICAO.md e comece pelos ADRs
```

**Prompt 3: verificação automática aplicada após cada documento.** Script gerado pela IA e reexecutado a cada entrega; qualquer linha `INVALIDO`, `FALTA` ou `QUEBRADO` exigia correção antes de seguir.

```bash
# timestamps citados existem na transcrição, com o falante certo?
grep -o '| TRANSCRICAO | \[[0-9:]*\] [A-Za-z]*' docs/TRACKER.md \
  | sed 's/| TRANSCRICAO | //' | sort -u \
  | while read ts nome; do grep -qF "$ts $nome:" TRANSCRICAO.md || echo "INVALIDO: $ts $nome"; done

# caminhos de código citados existem no repositório?
grep -ho '`src/[^`]*`\|`prisma/[^`]*`\|`tests/[^`]*`' docs/*.md docs/adrs/*.md \
  | tr -d '`' | sort -u | while read p; do [ -e "$p" ] || echo "FALTA $p"; done

# links relativos entre documentos resolvem?
grep -o '](\([^)#]*\.md\))' docs/*.md | sed 's/.*](\(.*\))/\1/' | sort -u \
  | while read l; do [ -f "docs/$l" ] || echo "QUEBRADO $l"; done
```

**Prompt 4 (adaptado do enunciado): fronteira entre documentos.** Antes do RFC, reforcei a regra de altura do enunciado para evitar duplicação.

```text
agora faça o RFC
(contexto já carregado: "O RFC é conciso, 2 a 4 páginas, fala em decisão;
não repita no RFC o nível de detalhe do FDD; aponte para os ADRs")
```

## Iterações e ajustes

Os ajustes relevantes aconteceram em cinco momentos. Nenhum deles foi "a IA errou um fato da transcrição"; foram lacunas e ambiguidades que só apareceram ao cruzar transcrição com código.

1. **Convenção de nome dos ADRs.** O `docs/adrs/README.md` do boilerplate sugeria `0001-titulo.md`, mas o enunciado exige `ADR-NNN-titulo-em-kebab-case.md`. Seguimos o enunciado e o índice da pasta foi reescrito para refletir isso.
2. **"5 tentativas" era ambíguo.** A transcrição diz "5 tentativas" e lista cinco intervalos (1m/5m/30m/2h/12h) que somam 14,6 h, "quase 15 horas entre primeira falha e última tentativa". Só a leitura "1 envio imediato + 5 retentativas" fecha a conta. O FDD adota essa interpretação e a marca explicitamente para confirmação com o Diego na revisão.
3. **`customer_id` ficou "no body ou no path".** A reunião não decidiu. A primeira versão do RFC registrou como questão em aberto; o FDD fechou seguindo `createOrderSchema` e `listOrdersQuerySchema` (body no POST, query no GET), com a justificativa anotada.
4. **`NotFoundError` não serve para `WEBHOOK_NOT_FOUND`.** O rascunho inicial da matriz de erros previa estender `NotFoundError`, mas o construtor em `src/shared/errors/http-errors.ts` fixa o código `NOT_FOUND`. Os erros 404 do módulo passaram a estender `AppError` diretamente. Mesmo tipo de correção: `orderId` da outbox ficou sem FK depois de constatar que `OrderService.delete` permite apagar pedidos `CANCELLED`, e o evento de cancelamento precisa sobreviver.
5. **Status `PENDING` nunca é destino de transição.** Ao mapear `order.status.ts`, ficou claro que criar um pedido não é mudança de status e que assinar `PENDING` nunca gera evento. Isso virou nota de integração no FDD e orientação para o portal, em vez de um requisito inventado de "evento de criação".

Foram **cinco ciclos principais** (contextualização, ADRs, RFC, FDD, PRD + tracker), cada um com rodada de verificação automática. As verificações passaram em todos os ciclos porque a disciplina de citar `[hh:mm] Nome` em cada afirmação foi adotada desde o primeiro documento.

## Como navegar a entrega

```
.
├── README.md                     ← este documento (processo)
├── TRANSCRICAO.md                ← fonte primária (não alterada)
├── docs/
│   ├── ENUNCIADO.md              ← enunciado original do desafio
│   ├── PRD.md                    ← por que e o quê (produto)
│   ├── RFC.md                    ← proposta técnica para revisão (arquitetura)
│   ├── FDD.md                    ← como construir (implementação)
│   ├── TRACKER.md                ← de onde veio cada item (247 linhas)
│   └── adrs/
│       ├── README.md             ← índice dos ADRs
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-retry-com-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes.md
│       ├── ADR-007-snapshot-do-payload-na-insercao.md
│       └── ADR-008-filtragem-de-eventos-na-insercao.md
├── src/, prisma/, tests/         ← código da aplicação (não alterado)
```

**Ordem de leitura sugerida:**

1. [`docs/PRD.md`](docs/PRD.md) para entender o problema, o escopo e o que ficou de fora.
2. [`docs/RFC.md`](docs/RFC.md) para a proposta, as alternativas descartadas e as questões em aberto.
3. [`docs/adrs/README.md`](docs/adrs/README.md) e os oito ADRs, na ordem 001 → 002 → 003 → 005 → 004 → 006 → 007 → 008.
4. [`docs/FDD.md`](docs/FDD.md) para modelo de dados, fluxos, contratos HTTP, erros `WEBHOOK_*` e a seção "Integração com o sistema existente".
5. [`docs/TRACKER.md`](docs/TRACKER.md) sempre que quiser conferir a origem de um item: cada ID aponta para `[hh:mm] Nome` na transcrição ou para um caminho real do código.
