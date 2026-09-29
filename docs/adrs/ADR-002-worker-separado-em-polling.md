# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

> **Arquivos novos vs. existentes:** `src/worker.ts` e tudo que fica em `src/modules/webhooks/` são **arquivos novos (a criar)** propostos na reunião ([09:11] Larissa, [09:27] Bruno). Todos os outros caminhos citados existem no repositório atual.

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e fazer as chamadas HTTP. Os clientes consideram "tempo real" qualquer coisa abaixo de 10 segundos ([09:02] Marcos). Hoje a aplicação só tem um entry-point, `src/server.ts`, que sobe a API Express e cria o `PrismaClient` compartilhado de `src/config/database.ts`.

## Decisão

1. **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos em lote pequeno, processa e marca o resultado ([09:08] Diego, [09:09] Diego). A latência mínima no pior caso é de 2s, e isso foi aceito ([09:10] Larissa, [09:10] Marcos).
2. **Processo separado da API.** Se a API reiniciar, o worker não pode cair junto ([09:11] Diego). O novo entry-point é `src/worker.ts`, com o script `npm run worker`, seguindo o modelo de `src/server.ts` ([09:11] Larissa, [09:28] Bruno).
3. **Mesma stack, mesmo banco, instância própria de Prisma.** O worker usa a mesma `DATABASE_URL`, mas cria um `PrismaClient` novo, porque o client é por processo ([09:11] Bruno, [09:30] Bruno).
4. **Instância única (single-worker).** A ordem de entrega por `order_id` depende de haver um só worker processando em ordem de `created_at` ([09:12] Diego). Isso fica registrado como **limitação conhecida**: não há garantia de ordem global, só por `order_id` e só enquanto houver um único worker ([09:13] Larissa).
5. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.processor.ts`. O `src/worker.ts` só faz bootstrap e shutdown ([09:28] Bruno).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger do banco para ser mais reativo** | O MySQL não tem listener nativo como o `LISTEN/NOTIFY` do Postgres. A trigger só executa SQL e não avisa processo externo. Seria preciso improvisar (arquivo, chamada HTTP) ([09:09] Bruno, [09:09] Diego). |
| **Worker rodando dentro do processo da API** | Um restart da API derruba o worker ([09:11] Diego). |
| **Múltiplos workers em paralelo** | Quebram a ordem por pedido. Particionar por `order_id` ou usar lock pessimista fica para o futuro ([09:13] Diego). |

## Consequências

**Positivas**
- Simples de implementar e operar. Não depende de nenhum recurso específico do banco.
- Com polling de 2s, a latência esperada fica bem abaixo do requisito de 10s ([09:09] Diego).
- Isola falhas: problemas de envio não afetam a disponibilidade da API.

**Negativas / trade-offs**
- Há latência mínima de até 2s mesmo sem carga, e queries periódicas no banco mesmo quando a outbox está vazia.
- Um único worker é gargalo de throughput e ponto único de falha do envio. Se ele cair, os eventos ficam acumulados, mas **não se perdem** (ficam na outbox).
- Mais um processo para deploy e monitoramento.
- Escalar horizontalmente exige outra decisão (particionamento/lock). Isso fica como dívida conhecida.
