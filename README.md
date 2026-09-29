# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

> Enunciado original: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia) (MBA em Engenharia de Software com IA, Full Cycle).

## Sobre o desafio

O ponto de partida era a transcrição de uma reunião de ~55 minutos (`TRANSCRICAO.md`) em que tech lead, PM, dois engenheiros e a engenheira de segurança decidiram como construir um **Sistema de Webhooks de Notificação de Pedidos** para um OMS em produção (Node.js + TypeScript + Prisma/MySQL). Nada tinha sido registrado além da gravação. A tarefa era transformar essa conversa, junto com o código existente, em um pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A maior dificuldade não era escrever, e sim **filtrar**. A reunião mistura decisões fechadas, ideias descartadas (e-mail, dashboard), pontos adiados (rate limiting, múltiplos workers), correções no meio da conversa (o `customer_id` "implícito no JWT" que depois deixa de ser) e ambiguidades que ninguém percebeu na hora. Cada afirmação dos documentos precisava apontar para uma fala `[hh:mm] Nome` ou para um arquivo real do repositório. Nenhum arquivo de `src/`, `prisma/` ou `tests/` foi alterado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (Claude Opus 5.5)** | Agente principal. Leu o repositório inteiro (módulos, schema Prisma, middlewares, testes), extraiu decisões da transcrição, redigiu todos os documentos e fez as revisões cruzadas. |
| **Claude in Chrome (extensão)** | Leitura do enunciado direto na plataforma Full Cycle (página autenticada), para trabalhar com os critérios de aceite exatos. |
| **Script Python de verificação** (gerado pela IA e rodado localmente) | Checagem automática contra alucinação: timestamps e falantes do tracker contra a transcrição, caminhos de código contra o disco, links entre documentos e cobertura de IDs. |

## Workflow adotado

1. **Ler o enunciado e os critérios.** Extraí o texto completo da página do desafio e transformei a seção "Critérios de Aceite" em checklist.
2. **Contextualização.** Clonei o repositório base e li a transcrição inteira e os arquivos que importam para a feature: `order.service.ts` (a transação de `changeStatus`), `order.status.ts`, `shared/errors/*`, `error.middleware.ts`, `validate.middleware.ts`, `auth.middleware.ts`, `logger`, `config/*`, `app.ts`, `routes/index.ts`, `schema.prisma` e `tests/setup.ts`.
3. **Inventário da transcrição.** Montei uma lista de todas as falas relevantes, classificadas em *decisão / requisito / restrição / descartado / adiado / ambíguo*, com o timestamp. Essa lista virou a espinha dorsal do tracker.
4. **ADRs primeiro** (7 ADRs): as 6 decisões principais listadas no enunciado, mais uma secundária com trade-off real (snapshot do payload + filtro na inserção).
5. **RFC** em cima dos ADRs: proposta em nível de arquitetura, 6 alternativas descartadas e 8 questões em aberto (5 vindas da reunião e 3 ambiguidades que encontrei ao cruzar falas).
6. **FDD**: modelo de dados, 8 contratos com exemplos, fluxos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente" com 16 pontos de integração em arquivos reais.
7. **PRD por último**, consolidando a visão de produto (14 RFs, 11 RNFs, métricas com metas numéricas).
8. **Tracker**, montado varrendo os IDs de todos os documentos.
9. **Verificação automática + revisão manual** item por item contra a checklist de aceite.

Para evitar duplicação entre documentos, usei esta regra: o **RFC** fala de "o quê e por quê" e aponta para o FDD, os **ADRs** guardam uma decisão cada, e o **FDD** concentra payloads, códigos e fluxos. Detalhes que precisei decidir porque a reunião não fechou estão marcados como **[Proposta FDD]** e ligados a uma questão em aberto do RFC.

## Prompts customizados

**Prompt 1 — Inventário filtrado da transcrição (antes de qualquer documento)**
```text
Você é um arquiteto de software revisando a transcrição TRANSCRICAO.md de uma reunião técnica.
NÃO gere documentos ainda. Produza uma tabela com TODAS as falas que contenham:
decisão fechada, requisito funcional, requisito não funcional, restrição, alternativa
descartada (com o motivo), item adiado/fora de escopo, ou ambiguidade/contradição.

Colunas: timestamp [hh:mm] | falante | categoria | conteúdo em 1 linha | status
(DECIDIDO / DESCARTADO / ADIADO / CORRIGIDO-DEPOIS / AMBÍGUO).

Regras:
- Se uma fala é corrigida depois (ex.: alguém propõe X e outro participante corrige),
  marque a primeira como CORRIGIDO-DEPOIS e aponte o timestamp da correção.
- Se dois trechos dão números que não fecham entre si, marque AMBÍGUO e explique.
- Não infira nada que não esteja dito. Se não há fala, não há linha.
```

**Prompt 2 — Seção "Integração com o sistema existente" ancorada no código**
```text
Leia src/modules/orders/order.service.ts, src/shared/errors/*.ts,
src/middlewares/*.ts, src/shared/logger/index.ts, src/config/*.ts, src/app.ts,
src/routes/index.ts, prisma/schema.prisma e tests/setup.ts.
Para cada arquivo, diga EXATAMENTE como o módulo de webhooks vai se integrar:
qual função/classe é tocada, em que linha lógica entra a mudança, e se o arquivo
NÃO precisa mudar (e por quê). Aponte incompatibilidades entre o que a reunião
decidiu e o que o código faz hoje (ex.: um middleware que sobrescreve códigos de erro).
Proibido citar arquivo que não exista; arquivos novos devem ser marcados "(novo)".
```

**Prompt 3 — Revisão adversarial contra alucinação**
```text
Atue como revisor cético. Para cada linha de docs/PRD.md, docs/RFC.md, docs/FDD.md
e docs/adrs/*.md que afirme requisito, decisão, número ou restrição:
1) aponte a fala [hh:mm] Nome ou o arquivo de código que a sustenta;
2) se não houver fonte, classifique como INVENTADO ou PROPOSTA-DERIVADA;
3) verifique se algum item descartado/adiado na reunião (e-mail, dashboard, rate
   limiting, exactly-once, multi-worker, inbound) aparece como requisito.
Depois, gere um script Python que valide automaticamente timestamps+falantes do
TRACKER contra a transcrição, caminhos de código contra o disco e a cobertura de IDs.
```

## Iterações e ajustes

Foram **4 ciclos principais** de geração → revisão → correção, com os 5 ajustes mais relevantes abaixo:

1. **"5 tentativas" que não fechavam.** O primeiro rascunho do ADR-003 dizia "5 tentativas" e listava os 5 intervalos (1m/5m/30m/2h/12h). Na revisão cruzada, a conta não fechava: 5 intervalos "entre primeira falha e última tentativa" ([09:17] Diego) significam 6 envios, mas o resumo da Larissa diz "total 5 tentativas" ([09:48] Larissa). Em vez de escolher em silêncio, registrei a interpretação adotada no ADR-003 e no FDD (1 envio + 5 reenvios, com a tabela explícita) e abri a **RFC-OPEN-06** para confirmação.
2. **Validação de 64KB no lugar errado.** A primeira versão do fluxo validava o tamanho do payload dentro de `publishWebhookEvent`, ou seja, dentro da transação de `changeStatus`. Isso faria um evento grande **reverter a mudança de status do pedido**, o que contradiz o objetivo da feature. Movi a checagem para o worker, com ida direta para a DLQ (`WEBHOOK_PAYLOAD_TOO_LARGE`), e registrei o motivo no FDD §6.2.
3. **Conflito entre o código e a reunião nos códigos de erro.** Pela reunião, a validação de HTTPS "é só uma validação no schema Zod" ([09:23] Sofia) e o código deveria ser `WEBHOOK_INVALID_URL` ([09:28] Bruno). Lendo `validate.middleware.ts`, vi que **todo** `ZodError` vira `VALIDATION_ERROR`, então as duas coisas não funcionariam juntas do jeito ingênuo. O FDD passou a explicar a solução (schema Zod `httpsUrlSchema` aplicado no service). Da mesma forma, `NotFoundError` fixa `NOT_FOUND`, então não serve para `WEBHOOK_NOT_FOUND`, e as classes novas foram desenhadas herdando direto de `AppError`.
4. **Correção de rumo na própria transcrição.** A extração inicial pegava "customer_id implícito do JWT" ([09:31] Marcos), mas um minuto depois o time corrige: o JWT é do operador, e o `customer_id` vai explícito ([09:32] Larissa). Os contratos do FDD foram ajustados para seguir o padrão existente de `order.schemas.ts` (`customerId` no body/query).
5. **O script de verificação pegou dois problemas reais.** (a) A cobertura do tracker estava em 83%, com 32 IDs dos documentos sem linha (critérios de aceite, exclusões). Completei as linhas. (b) O script acusou `src/worker.ts` e `src/modules/webhooks/*.ts` como "inexistentes". Isso é correto, porque são arquivos a criar. Para não violar o critério "nenhum arquivo mencionado é inexistente", passei a marcá-los explicitamente como **(novo)** em todos os documentos.

## Como navegar a entrega

Ordem de leitura sugerida:

1. [`docs/PRD.md`](docs/PRD.md): por que e o quê (problema, escopo, métricas, requisitos).
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/README.md): uma decisão por arquivo.
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [ADR-003 Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
   - [ADR-004 At-least-once com X-Event-Id](docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)
   - [ADR-005 HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-006 Reuso dos padrões existentes](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md)
   - [ADR-007 Snapshot do payload e filtro na inserção](docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)
4. [`docs/FDD.md`](docs/FDD.md): como construir (dados, contratos, fluxos, erros, integração).
5. [`docs/TRACKER.md`](docs/TRACKER.md): origem de cada item.

```
.
├── README.md                 ← este arquivo (processo)
├── TRANSCRICAO.md            ← fonte (inalterada)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/ README.md + ADR-001 … ADR-007
└── src/ prisma/ tests/       ← código base (inalterado)
```
