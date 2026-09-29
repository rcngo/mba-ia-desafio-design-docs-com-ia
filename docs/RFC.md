# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autora** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | Semana da reunião técnica (quinta-feira, 09:00) |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Documentos** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> **Arquivos novos vs. existentes:** `src/worker.ts` e tudo que fica em `src/modules/webhooks/` são **arquivos novos (a criar)** propostos na reunião ([09:11] Larissa, [09:27] Bruno). Todos os outros caminhos citados existem no repositório atual.

---

## 1. Resumo (TL;DR)

A proposta é notificar os clientes B2B, por **webhook outbound**, sempre que o status de um pedido mudar. No mesmo `$transaction` do `OrderService.changeStatus`, gravamos o evento numa **outbox no MySQL**. Um **worker em processo separado**, fazendo **polling a cada 2s**, envia cada evento por HTTPS com **assinatura HMAC-SHA256** (secret por endpoint). Quando o envio falha, o worker aplica **backoff exponencial** (1m/5m/30m/2h/12h). Depois da última tentativa, o evento vai para uma **DLQ** que só um ADMIN pode reprocessar. A entrega é **at-least-once**, e o cliente deduplica pelo `X-Event-Id`. Tudo isso vive em um módulo `src/modules/webhooks` que segue os padrões atuais do projeto. Não entra infraestrutura nova.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para ser avisados quando o status dos pedidos muda. Hoje eles fazem polling em `GET /orders`, e a integração fica lenta e cara. A Atlas sinalizou que pode migrar para um concorrente se não houver entrega até o fim do trimestre ([09:00] Marcos). Para os clientes, "tempo real" é qualquer coisa abaixo de 10 segundos ([09:02] Marcos).

A aplicação atual não tem nenhum mecanismo de notificação, evento, fila ou webhook. A mudança de status é uma transação Prisma que já atualiza `orders`, grava `order_status_history` e mexe no estoque (`src/modules/orders/order.service.ts`). Colocar I/O de rede ali atrasaria todos os pedidos por causa de um cliente lento ([09:04] Bruno).

## 3. Proposta técnica

### 3.1 Visão geral

```
 API (src/server.ts)                          Worker (src/worker.ts)
 ┌──────────────────────────────┐             ┌───────────────────────────────┐
 │ PATCH /orders/:id/status     │             │ loop a cada 2s                │
 │  └─ OrderService.changeStatus│             │  1. busca PENDING vencidos    │
 │      $transaction {          │             │  2. assina (HMAC-SHA256)      │
 │        update orders         │   MySQL     │  3. POST https (timeout 10s)  │
 │        insert status_history │  ┌───────┐  │  4. 2xx → DELIVERED           │
 │        stock debit/replenish │─▶│outbox │◀─│     falha → retry c/ backoff  │
 │        publishWebhookEvent() │  └───────┘  │     5ª falha → dead_letter    │
 │      }                       │  ┌───────┐  └──────────────┬────────────────┘
 │ CRUD /webhooks, deliveries   │  │  DLQ  │                 │ HTTPS + headers
 │ POST /admin/.../replay (ADM) │─▶└───────┘                 ▼
 └──────────────────────────────┘                 Endpoint do cliente B2B
```

### 3.2 Componentes

| Componente | Responsabilidade | Decisão |
|---|---|---|
| **Outbox (`webhook_outbox`)** | Registro transacional do evento, gravado junto com a mudança de status | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **`publishWebhookEvent(tx, …)`** | Filtra os webhooks inscritos no `to_status` e grava o snapshot do payload | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |
| **Worker** | Processo separado, polling de 2s, single-worker | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| **Política de retry + DLQ** | 5 reenvios com backoff; depois, `webhook_dead_letter` e replay manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| **Assinatura** | HMAC-SHA256 no corpo, secret por endpoint, rotação com 24h de convivência | [ADR-005](adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| **Semântica de entrega** | At-least-once, `X-Event-Id` estável | [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) |
| **API de configuração** | CRUD de webhooks, rotação de secret, histórico de entregas, replay admin | FDD §5 |

### 3.3 Superfície pública (resumo)

- **Para o cliente:** cadastrar, listar, editar e remover webhooks; rotacionar secret; consultar histórico de entregas ([09:31] Marcos, [09:33] Bruno, [09:34] Marcos, [09:21] Sofia).
- **Para a operação (ADMIN):** fazer replay de um item da DLQ ([09:18] Diego, [09:36] Sofia).
- **O que o cliente recebe:** `POST` JSON com `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:44] Diego, [09:44] Sofia).

Contratos, schemas, códigos de erro e fluxos detalhados estão no [FDD](FDD.md).

### 3.4 Garantias e limites prometidos

- A latência alvo fica abaixo de 10s. O mínimo estrutural é de até 2s, por causa do polling ([09:10] Larissa).
- At-least-once: duplicatas podem acontecer, e perda de evento não pode.
- **A ordem só é garantida por `order_id`, e só enquanto houver um único worker.** Não há ordem global ([09:13] Larissa, [09:14] Marcos).
- O payload tem no máximo 64KB ([09:24] Larissa). URLs só HTTPS ([09:23] Sofia). Timeout de 10s ([09:42] Diego).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Fonte |
|---|---|---|---|
| RFC-ALT-01 | **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento trava a transação de status de todos os pedidos, e cliente offline levaria a um rollback impossível do status. Perde-se isolamento de falhas. | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | **Redis Streams / fila dedicada** | Entrega mais reativa, mas pede infraestrutura nova para um time pequeno ("overengineering"). A outbox no MySQL resolve com o que já temos. | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | **Trigger do MySQL notificando o worker** | O MySQL não tem `LISTEN/NOTIFY`. A trigger não avisa processo externo, então seria preciso improvisar. O polling de 2s já atende o requisito de latência. | [09:09] Bruno, [09:09] Diego |
| RFC-ALT-04 | **Retry indefinido** ou **só 3 tentativas** | Retry indefinido deixa evento pendurado para sempre. Com 3 tentativas, tudo cabe em uns 30 min e não cobre manutenção de 2h. | [09:15] Diego, [09:16] Diego |
| RFC-ALT-05 | **Exactly-once** | Exige coordenação dos dois lados, com complexidade muito maior. At-least-once + `event_id` resolve 99% dos casos. | [09:25] Diego |
| RFC-ALT-06 | **Secret global da plataforma** | Um vazamento compromete todos os clientes. | [09:21] Sofia |

## 5. Questões em aberto

| # | Questão | Situação | Fonte |
|---|---|---|---|
| RFC-OPEN-01 | **Rate limiting de saída por cliente.** Um cliente com 50 pedidos mudando em um minuto recebe 50 chamadas. | Fora do escopo. Decisão: "observar e decidir depois", com base na métrica de volume por webhook. | [09:38] Diego, [09:39] Larissa |
| RFC-OPEN-02 | **Notificação por e-mail quando o webhook do cliente falha repetidamente** | Adiada para a próxima fase, depois de medir o impacto. | [09:37] Marcos, [09:37] Larissa |
| RFC-OPEN-03 | **Escalar para múltiplos workers** (particionar por `order_id` ou usar lock pessimista) | Adiada ("problema do futuro"). Hoje é limitação conhecida. | [09:13] Diego, [09:13] Larissa |
| RFC-OPEN-04 | **Restringir o CRUD de webhooks por role** | Por enquanto, qualquer role autenticada. "Mais pra frente a gente pode endurecer." | [09:37] Sofia |
| RFC-OPEN-05 | **Arquivamento da outbox** (linhas entregues com mais de ~30 dias) | Fora do escopo desta feature. | [09:08] Diego |
| RFC-OPEN-06 | **Contagem exata de tentativas.** "5 tentativas" vs. 5 intervalos somando ~15h "entre primeira falha e última tentativa". O FDD adota 1 envio + 5 reenvios. **Confirmar com Diego/Larissa.** | Ambiguidade identificada ao cruzar as falas. | [09:15] Diego, [09:17] Diego, [09:48] Larissa |
| RFC-OPEN-07 | **Ordem por pedido durante retries.** Um evento em backoff pode ser ultrapassado por um evento mais novo do mesmo pedido. Bloquear a fila do pedido (head-of-line) ou aceitar e documentar? | Não discutido. O FDD documenta como limitação, e o cliente pode usar `timestamp`/`from_status` para reconciliar. | derivado de [09:12] Diego + [09:17] Larissa |
| RFC-OPEN-08 | **Proteção da secret em repouso** (precisa ser recuperável para assinar) | Não discutido. Levar para a revisão de segurança. | derivado de [09:21] Sofia, [09:46] Sofia |

## 6. Impacto e riscos

**Impacto no sistema existente**
- `OrderService.changeStatus` passa a chamar `publishWebhookEvent` dentro da transação: uma leitura e 0..N inserts a mais no caminho crítico ([09:40] Bruno).
- Nasce um segundo processo (`npm run worker`), com seu próprio pool de conexões no mesmo banco ([09:30] Bruno).
- Três tabelas novas e uma migration Prisma. Nenhuma mudança de contrato nos endpoints atuais.

**Riscos principais** (detalhes e mitigações no PRD §10 e FDD §11)

| Risco | Mitigação |
|---|---|
| Falha na inserção da outbox bloqueia mudança de status | É intencional: a consistência vem antes. Testes ponta a ponta e insert mínimo e indexado ([09:40] Bruno). |
| Worker parado acumula eventos e estoura o SLA de 10s | Os eventos não se perdem. Alarme por idade do evento mais antigo em `PENDING`. |
| Vazamento de secret | Secret por endpoint, rotação, `redact` no logger, revisão de segurança dedicada ([09:22] Diego, [09:46] Sofia). |
| Prazo (Atlas: fim de novembro) | Estimativa de 3 sprints, com a revisão de segurança incluída ([09:45] Marcos, [09:46] Larissa). |

## 7. Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — At-least-once com X-Event-Id](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)
- [ADR-005 — HMAC-SHA256 com secret por endpoint](adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload e filtro na inserção](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## 8. Próximos passos

1. Sessão de revisão deste RFC e do FDD com Bruno e Diego antes de começar a codar ([09:50] Larissa).
2. Fechar RFC-OPEN-06 e RFC-OPEN-07 nessa sessão.
3. Agendar a revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy ([09:46] Sofia, [09:49] Sofia).
