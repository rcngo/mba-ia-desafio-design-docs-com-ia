# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Pronto para revisão técnica (Bruno, Diego) e de segurança (Sofia) |
| **Documentos** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> Os IDs entre colchetes (ex.: `FDD-CONTRATO-01`) estão mapeados à origem em [TRACKER.md](TRACKER.md).
> Onde este documento precisou **decidir um detalhe que a reunião não fechou**, o trecho aparece como **[Proposta FDD]** e aponta para a questão correspondente no RFC.

> **Arquivos novos vs. existentes:** `src/worker.ts` e tudo que fica em `src/modules/webhooks/` são **arquivos novos (a criar)** propostos na reunião ([09:11] Larissa, [09:27] Bruno). Todos os outros caminhos citados existem no repositório atual.

---

## 1. Contexto e motivação técnica

A mudança de status de pedido é feita em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que:

1. lê o pedido com itens;
2. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
3. debita ou repõe estoque (`shouldDebitStock` / `shouldReplenishStock`);
4. atualiza `orders.status` e grava `order_status_history`.

Não existe nenhum mecanismo de notificação externa. Os clientes B2B descobrem mudanças fazendo polling em `GET /api/v1/orders` ([09:00] Marcos). A feature cria uma notificação push confiável sem pôr I/O de rede nessa transação ([09:04] Bruno) e sem infraestrutura nova ([09:07] Diego).

## 2. Objetivos técnicos

| ID | Objetivo |
|---|---|
| FDD-OBJ-01 | Toda mudança de status commitada gera um evento persistido na mesma transação. Nenhuma mudança revertida gera evento ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)). |
| FDD-OBJ-02 | O evento é entregue ao endpoint do cliente em menos de 10s em condições normais ([09:02] Marcos), com polling de 2s ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). |
| FDD-OBJ-03 | Falhas do cliente são absorvidas por retry com backoff por ~15h e depois vão para a DLQ, que permite replay ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)). |
| FDD-OBJ-04 | O cliente consegue autenticar a origem e a integridade de cada chamada por HMAC-SHA256 ([ADR-005](adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md)). |
| FDD-OBJ-05 | O cliente consegue deduplicar reenvios pelo `X-Event-Id` ([ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)). |
| FDD-OBJ-06 | Zero dependências novas: Prisma, Zod, Pino, Express e `fetch` nativo do Node ≥ 20 (`package.json` → `engines.node`) ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)). |

## 3. Escopo e exclusões

**Incluso**
- Módulo `src/modules/webhooks/` com CRUD de configuração, rotação de secret, histórico de entregas e replay admin da DLQ.
- Tabelas `webhooks`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`, via migration Prisma.
- Entry-point `src/worker.ts` e script `npm run worker` ([09:11] Larissa).
- Integração em `OrderService.changeStatus` por meio de `publishWebhookEvent` ([09:41] Bruno).

**Excluído** (com origem)
| ID | Exclusão | Fonte |
|---|---|---|
| FDD-EXC-01 | Webhooks inbound (cliente enviando para nós) | [09:02] Marcos |
| FDD-EXC-02 | Notificação por e-mail em falhas repetidas | [09:37] Larissa |
| FDD-EXC-03 | Rate limiting de saída (só observar) | [09:39] Larissa |
| FDD-EXC-04 | Dashboard visual para o cliente | [09:40] Larissa |
| FDD-EXC-05 | Arquivamento/limpeza da outbox (~30 dias) | [09:08] Diego |
| FDD-EXC-06 | Múltiplos workers / ordem global | [09:13] Diego |
| FDD-EXC-07 | Evento na **criação** do pedido (`OrderService.create` grava `PENDING` com `fromStatus: null` sem passar por `changeStatus`). A reunião tratou só de "quando o status muda". | [09:40] Bruno + `src/modules/orders/order.service.ts` |
| FDD-EXC-08 | Listagem da DLQ via API. Só o replay por id foi discutido; a descoberta dos ids é por consulta ao banco/logs. | [09:18] Diego |

## 4. Modelo de dados

Novos modelos em `prisma/schema.prisma`, seguindo as convenções do arquivo: UUID `Char(36)`, `@@map` em snake_case, `createdAt`/`updatedAt` ([09:51] Larissa).

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  DELIVERED    // entregue
  FAILED       // falhou (DLQ ou webhook inativo)
}

model Webhook {                                   // FDD-DATA-01
  id                      String        @id @default(uuid()) @db.Char(36)
  customerId              String        @db.Char(36)
  url                     String        @db.VarChar(2048)
  secret                  String        @db.VarChar(128)
  previousSecret          String?       @db.VarChar(128)   // válida durante o grace period
  previousSecretExpiresAt DateTime?                        // now + 24h na rotação
  events                  Json          // OrderStatus[] assinados, ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean       @default(true)
  deletedAt               DateTime?     // [Proposta FDD] soft delete, preserva histórico/DLQ
  createdAt               DateTime      @default(now())
  updatedAt               DateTime      @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {                             // FDD-DATA-02
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @unique @db.Char(36)   // enviado em X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)        // "order.status_changed"
  payload       Json                                        // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)             // nº de falhas acumuladas
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook Webhook @relation(fields: [webhookId], references: [id])

  @@index([status])
  @@index([createdAt])
  @@index([status, nextAttemptAt, createdAt])   // query do worker
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {                           // FDD-DATA-03 — histórico por tentativa
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  webhookId    String   @db.Char(36)
  eventId      String   @db.Char(36)
  attempt      Int
  success      Boolean
  statusCode   Int?
  errorCode    String?  @db.VarChar(64)
  durationMs   Int
  responseBody String?  @db.Text          // [Proposta FDD] truncado em 4KB
  createdAt    DateTime @default(now())

  webhook Webhook @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {                         // FDD-DATA-04
  id             String    @id @default(uuid()) @db.Char(36)
  outboxId       String    @unique @db.Char(36)
  webhookId      String    @db.Char(36)
  eventId        String    @db.Char(36)
  payload        Json
  reason         String    @db.VarChar(500)   // errorCode + mensagem da última falha
  lastStatusCode Int?
  attempts       Int
  failedAt       DateTime  @default(now())
  replayedAt     DateTime?
  replayedById   String?   @db.Char(36)       // auditoria do replay ([09:36] Sofia)

  webhook Webhook @relation(fields: [webhookId], references: [id])

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Observações:
- **Uma linha de outbox por (evento × webhook destino).** O filtro por status é aplicado na inserção ([ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)), e cada linha tem seu próprio `eventId` e `X-Webhook-Id` ([09:44] Sofia).
- `Customer` ganha a relação `webhooks Webhook[]`. O model `Customer` fica igual em todo o resto.
- `events` reusa os valores do enum `OrderStatus` (`PENDING | PAID | PROCESSING | SHIPPED | DELIVERED | CANCELLED`).

## 5. Contratos públicos

Todas as rotas ficam sob `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <jwt>` (`authenticate`) e devolvem erros no formato do `errorMiddleware`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

O `customerId` é **explícito** (body ou query), não vem do JWT. O JWT é do usuário operador que representa o cliente ([09:32] Bruno, [09:32] Larissa). O padrão segue `createOrderSchema` (`customerId` no body) e `listOrdersQuerySchema` (`customerId` na query) de `src/modules/orders/order.schemas.ts`.

### FDD-CONTRATO-01 — Cadastrar webhook
`POST /api/v1/webhooks` · qualquer role autenticada ([09:37] Sofia)

Request:
```json
{
  "customerId": "5b1c8a52-0c7e-4a57-9a39-6c1f1f0c2b11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```
Response `201 Created`. A `secret` **só aparece aqui** e na rotação ([09:31] Marcos):
```json
{
  "id": "0f4e2c1a-7d9b-4b8e-9d6c-2a1f3e5b7c90",
  "customerId": "5b1c8a52-0c7e-4a57-9a39-6c1f1f0c2b11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_3q2+7w8kz1Yb0cJt4mQpLx9RrV6uN5sA0eHfGdTiKoM=",
  "createdAt": "2026-10-01T12:00:00.000Z",
  "updatedAt": "2026-10-01T12:00:00.000Z"
}
```
Status codes: `201`, `400 VALIDATION_ERROR` (body inválido, `events` vazio ou com status inexistente), `400 WEBHOOK_INVALID_URL` (URL não-HTTPS), `401 UNAUTHORIZED`, `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### FDD-CONTRATO-02 — Listar webhooks de um customer
`GET /api/v1/webhooks?customerId={uuid}&page=1&pageSize=20` ([09:33] Bruno)

Response `200 OK`, no formato `paginated()` de `src/shared/http/response.ts`. A `secret` **nunca** aparece:
```json
{
  "data": [
    {
      "id": "0f4e2c1a-7d9b-4b8e-9d6c-2a1f3e5b7c90",
      "customerId": "5b1c8a52-0c7e-4a57-9a39-6c1f1f0c2b11",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-01T12:00:00.000Z",
      "updatedAt": "2026-10-01T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Status codes: `200`, `400 VALIDATION_ERROR` (`customerId` ausente ou inválido), `401 UNAUTHORIZED`.

### FDD-CONTRATO-03 — Editar webhook
`PATCH /api/v1/webhooks/:id` ([09:33] Bruno)

Request (todos os campos opcionais, pelo menos um obrigatório):
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: mesmo corpo do item de FDD-CONTRATO-02, já atualizado.
Status codes: `200`, `400 VALIDATION_ERROR`, `400 WEBHOOK_INVALID_URL`, `401`, `404 WEBHOOK_NOT_FOUND`.
Semântica: a mudança vale **só para eventos futuros**. Linhas já gravadas na outbox não são refiltradas ([ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)).

### FDD-CONTRATO-04 — Remover webhook
`DELETE /api/v1/webhooks/:id` ([09:33] Bruno)

Response `204 No Content` (sem corpo). Status codes: `204`, `401`, `404 WEBHOOK_NOT_FOUND`.
Semântica **[Proposta FDD]**: soft delete (`deletedAt = now()`, `active = false`) para manter `webhook_deliveries` e `webhook_dead_letter` como evidência. O worker descarta linhas pendentes de webhook removido (ver §6.2).

### FDD-CONTRATO-05 — Rotacionar secret
`POST /api/v1/webhooks/:id/rotate-secret` ([09:21] Sofia)

Request: sem corpo. Response `200 OK`:
```json
{
  "id": "0f4e2c1a-7d9b-4b8e-9d6c-2a1f3e5b7c90",
  "secret": "whsec_Qm9vZ2xlLWhvb2stbmV3LXNlY3JldC0yMDI2LTEwLTAy",
  "previousSecretExpiresAt": "2026-10-03T09:00:00.000Z"
}
```
Status codes: `200`, `401`, `404 WEBHOOK_NOT_FOUND`, `409 WEBHOOK_ROTATION_IN_PROGRESS` **[Proposta FDD]** (já existe uma secret anterior em grace period; limita a no máximo duas secrets válidas).

### FDD-CONTRATO-06 — Histórico de entregas
`GET /api/v1/webhooks/:id/deliveries?limit=100` ([09:34] Marcos)

Devolve as **últimas 100** tentativas (`limit` entre 1 e 100, padrão 100), da mais recente para a mais antiga, com sucesso/falha, payload, resposta e tempo de resposta.
```json
{
  "data": [
    {
      "id": "9a7c1e44-2b1d-4f0e-8c33-51e7d2a9f6b0",
      "eventId": "c3d9f0a2-5e61-4b7a-9f14-0e2b8c7d6a55",
      "attempt": 2,
      "success": true,
      "statusCode": 200,
      "durationMs": 184,
      "errorCode": null,
      "payload": { "event_id": "c3d9f0a2-5e61-4b7a-9f14-0e2b8c7d6a55", "event_type": "order.status_changed", "...": "..." },
      "responseBody": "{\"received\":true}",
      "createdAt": "2026-10-01T12:01:03.412Z"
    }
  ]
}
```
Status codes: `200`, `400 VALIDATION_ERROR`, `401`, `404 WEBHOOK_NOT_FOUND`.

### FDD-CONTRATO-07 — Replay de item da DLQ (admin)
`POST /api/v1/admin/webhooks/dead-letter/:id/replay` · **`requireRole('ADMIN')`** ([09:18] Diego, [09:36] Sofia, [09:36] Larissa)

Request: sem corpo. Response `202 Accepted`:
```json
{
  "deadLetterId": "71b3c2d4-8e9f-4a01-b2c3-d4e5f6a7b8c9",
  "outboxId": "e2f4a6b8-1c3d-4e5f-8a9b-0c1d2e3f4a5b",
  "eventId": "c3d9f0a2-5e61-4b7a-9f14-0e2b8c7d6a55",
  "status": "PENDING",
  "replayedAt": "2026-10-02T14:30:00.000Z",
  "replayedById": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
}
```
Status codes: `202`, `401 UNAUTHORIZED`, `403 FORBIDDEN` (role ≠ ADMIN, vindo do `requireRole` existente), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
Semântica: a linha correspondente da outbox volta para `PENDING` com `attempts = 0` e `nextAttemptAt = now()`, **mantendo o mesmo `eventId`** ([ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)). O replay grava log de auditoria com `req.user.id` ([09:36] Sofia).

### FDD-CONTRATO-08 — Chamada enviada ao cliente (outbound)
`POST {webhook.url}`, enviada pelo worker.

Headers ([09:44] Diego, [09:44] Sofia):

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento, estável em todos os reenvios |
| `X-Webhook-Id` | `id` do cadastro de webhook |
| `X-Timestamp` | Instante do envio, em ISO 8601 (para detecção de replay pelo cliente) |
| `X-Signature` | `sha256=<hex(HMAC-SHA256(secret, rawBody))>` |

Durante o grace period de rotação, **[Proposta FDD]**: `X-Signature: sha256=<hmac_secret_nova>,sha256=<hmac_secret_anterior>`. O cliente aceita a requisição se qualquer uma das assinaturas bater. A validar na revisão de segurança (RFC-OPEN-08).

Body ([09:43] Diego):
```json
{
  "event_id": "c3d9f0a2-5e61-4b7a-9f14-0e2b8c7d6a55",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-01T12:00:58.120Z",
  "order_id": "8d2e6f10-3a4b-4c5d-9e8f-7a6b5c4d3e2f",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "5b1c8a52-0c7e-4a57-9a39-6c1f1f0c2b11",
  "total_cents": 18000
}
```
Semântica da resposta do cliente: **qualquer `2xx` dentro de 10s = entregue**. Qualquer outra resposta, timeout ou erro de rede conta como falha e vai para retry ([09:42] Diego).

## 6. Fluxos detalhados

### 6.1 FDD-FLOW-01 — Criação do evento na outbox (API, dentro da transação)

```
PATCH /orders/:id/status
  └─ OrderService.changeStatus(id, input, userId)
       prisma.$transaction(async (tx) => {
         order = tx.order.findUnique(... include items)        (existente)
         valida from≠to e canTransition(from,to)                (existente)
         debitStock / replenishStock                            (existente)
         tx.order.update(status = to)                           (existente)
         tx.orderStatusHistory.create(...)                      (existente)
         await publishWebhookEvent(tx, order, from, to)         ◀── NOVO
         refreshed = tx.order.findUnique(...)                   (existente)
       })
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` em `src/modules/webhooks/webhook.publisher.ts` (novo):
1. `tx.webhook.findMany({ where: { customerId: order.customerId, active: true, deletedAt: null } })`.
2. Filtra em memória os webhooks cujo `events` contém `toStatus` ([09:33] Marcos). Se não sobrar nenhum, **retorna sem inserir nada** ([09:34] Bruno).
3. Para cada webhook: gera `eventId = uuidv4()` (pacote `uuid`, já dependência), monta o payload snapshot (§5, FDD-CONTRATO-08) com `timestamp = new Date().toISOString()` e faz `tx.webhookOutbox.create({ status: PENDING, nextAttemptAt: now })`.
4. Qualquer exceção **propaga** e causa rollback de toda a mudança de status ([09:40] Bruno, [09:41] Diego).
5. Loga `webhook_event_enqueued` com `{ eventId, webhookId, orderId, toStatus }`.

### 6.2 FDD-FLOW-02 — Processamento pelo worker

`src/worker.ts` (novo, bootstrap) → `WebhookProcessor.run()` (`src/modules/webhooks/webhook.processor.ts`, novo).

```
startup:
  prisma = createPrismaClient()                       // instância própria (09:30 Bruno)
  UPDATE webhook_outbox SET status=PENDING WHERE status=PROCESSING   // recuperação de crash [Proposta FDD]
loop (enquanto !shuttingDown):
  batch = SELECT * FROM webhook_outbox
          WHERE status='PENDING' AND next_attempt_at <= NOW()
          ORDER BY created_at ASC
          LIMIT WEBHOOK_BATCH_SIZE                    // lote pequeno (09:08 Diego)
  para cada evento, SEQUENCIALMENTE (mantém ordem por order_id):
    marca PROCESSING
    webhook = findUnique(evento.webhookId)
    se webhook inativo/removido → status=FAILED, lastError=WEBHOOK_INACTIVE (sem DLQ); continue
    se !webhook.secret → falha não-retentável WEBHOOK_SECRET_REQUIRED → DLQ
    body = JSON.stringify(evento.payload)
    se Buffer.byteLength(body) > 64KB → falha não-retentável WEBHOOK_PAYLOAD_TOO_LARGE → DLQ
    headers = assinar(body, webhook)                   // §5 FDD-CONTRATO-08
    resp = fetch(url, { method:'POST', body, headers, signal: AbortSignal.timeout(10_000) })
    grava webhook_deliveries (attempt, success, statusCode, durationMs, responseBody truncado)
    se 2xx → status=DELIVERED, deliveredAt=now
    senão  → FLOW-03 (retry)
  aguarda WEBHOOK_POLL_INTERVAL_MS (2000) com setTimeout APÓS terminar o lote (sem sobreposição)
shutdown (SIGINT/SIGTERM): termina o evento em curso, para o loop, prisma.$disconnect()
```

Motivo do limite de 64KB ser checado no worker e não na inserção: se `publishWebhookEvent` lançasse erro por tamanho, a **mudança de status do pedido sofreria rollback** por causa de uma notificação. O evento vai direto para a DLQ como evidência de que "algo está errado" ([09:23] Sofia).

### 6.3 FDD-FLOW-03 — Retry com backoff

Tabela de backoff (`WEBHOOK_BACKOFF_SCHEDULE`), indexada pelo número de falhas ([09:17] Diego):

| Falha nº (`attempts` após incremento) | Próxima tentativa em | Envio nº |
|---|---|---|
| 1 (envio inicial falhou) | +1 min | 2 |
| 2 | +5 min | 3 |
| 3 | +30 min | 4 |
| 4 | +2 h | 5 |
| 5 | +12 h | 6 |
| 6 | → **DLQ** (FLOW-04) | — |

```
attempts = attempts + 1
se attempts <= 5:
   status = PENDING, nextAttemptAt = now + BACKOFF[attempts-1], lastError = <código>
   log webhook_retry_scheduled
senão:
   FLOW-04
```
A soma dos intervalos dá 14h36min, ou seja, "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). Essa interpretação (1 envio inicial + 5 reenvios) está pendente de confirmação em RFC-OPEN-06. Se a leitura for "5 envios no total", basta encurtar a tabela para 4 intervalos. A lógica não muda.

### 6.4 FDD-FLOW-04 — DLQ e replay

Movimentação para a DLQ, **em uma única transação**:
1. `webhook_outbox.status = FAILED`, `lastError = WEBHOOK_MAX_ATTEMPTS_EXCEEDED` (ou o código não-retentável).
2. `webhook_dead_letter.create({ outboxId, webhookId, eventId, payload, reason, lastStatusCode, attempts })` ([09:18] Diego).
3. Log `webhook_dead_lettered` em nível `warn`.

Replay (FDD-CONTRATO-07), **em uma única transação**:
1. Busca o dead letter. Se não existe → `WEBHOOK_DEAD_LETTER_NOT_FOUND`. Se `replayedAt != null` → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
2. `webhook_outbox`: `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`, `lastError = null` ("recoloca na outbox como pendente", [09:18] Diego).
3. `webhook_dead_letter`: `replayedAt = now()`, `replayedById = req.user.id`.
4. Log `webhook_replayed` com `{ deadLetterId, eventId, adminUserId }` ([09:36] Sofia).

Se o evento reprocessado falhar de novo em todas as tentativas, é criado um novo dead letter? Não: `outboxId` é `@unique` na DLQ. O registro existente é atualizado (`replayedAt = null`, novo `reason`, `failedAt = now()`), e o replay volta a ficar disponível.

### 6.5 FDD-FLOW-05 — Rotação de secret

1. Se `previousSecret != null` e `previousSecretExpiresAt > now` → `409 WEBHOOK_ROTATION_IN_PROGRESS`.
2. `previousSecret = secret`, `previousSecretExpiresAt = now + 24h`, `secret = gerarSecret()` ([09:21] Sofia).
3. O worker assina com as duas enquanto `previousSecretExpiresAt > now`. Depois disso, ignora `previousSecret` (e pode limpá-la no próximo uso).

Geração de secret: `crypto.randomBytes(32)` em base64 com prefixo `whsec_`, via módulo nativo `node:crypto`. **Revisão obrigatória da Sofia** ([09:46] Sofia).

## 7. Matriz de erros

Códigos novos seguem o padrão de `src/shared/errors/http-errors.ts` com prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa).

| ID | Código | HTTP | Onde | Retentável? | Quando |
|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | API | — | `:id` inexistente ou soft-deleted ([09:28] Bruno) |
| FDD-ERR-02 | `WEBHOOK_INVALID_URL` | 400 | API | — | URL não é `https://` ou é malformada ([09:23] Sofia, [09:28] Bruno) |
| FDD-ERR-03 | `WEBHOOK_SECRET_REQUIRED` | — (interno) | Worker | Não → DLQ | Webhook sem secret no momento de assinar ([09:28] Bruno) |
| FDD-ERR-04 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | — | `customerId` do cadastro não existe |
| FDD-ERR-05 | `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | API | — | Rotação pedida durante o grace period **[Proposta FDD]** |
| FDD-ERR-06 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API (admin) | — | `:id` da DLQ inexistente |
| FDD-ERR-07 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | API (admin) | — | Replay repetido do mesmo item |
| FDD-ERR-08 | `WEBHOOK_PAYLOAD_TOO_LARGE` | — (interno) | Worker | Não → DLQ | Corpo serializado > 64KB ([09:24] Larissa) |
| FDD-ERR-09 | `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | Worker | Sim | Sem resposta em 10s ([09:42] Diego) |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_FAILED` | — (interno) | Worker | Sim | Resposta não-2xx ([09:14] Larissa) |
| FDD-ERR-11 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | — (interno) | Worker | Sim | DNS, conexão recusada, erro TLS |
| FDD-ERR-12 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — (interno) | Worker | Não → DLQ | 6ª falha acumulada ([09:15] Diego) |
| FDD-ERR-13 | `WEBHOOK_INACTIVE` | — (interno) | Worker | Não (descarta, sem DLQ) | Webhook desativado/removido depois da inserção |

Códigos existentes reutilizados sem alteração: `VALIDATION_ERROR` (Zod via `validate.middleware.ts`), `UNAUTHORIZED` e `FORBIDDEN` (`auth.middleware.ts`), `INTERNAL_SERVER_ERROR` (`error.middleware.ts`).

Classes novas em `src/shared/errors/http-errors.ts` (exportadas em `src/shared/errors/index.ts`), no mesmo molde de `InsufficientStockError`:

```ts
export class WebhookNotFoundError extends AppError {
  constructor() { super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND'); }
}
export class WebhookInvalidUrlError extends BadRequestError {
  constructor(url: string) { super('Webhook URL must use https', 'WEBHOOK_INVALID_URL', { url }); }
}
export class WebhookRotationInProgressError extends ConflictError {
  constructor(expiresAt: Date) {
    super('Previous secret still in grace period', 'WEBHOOK_ROTATION_IN_PROGRESS', { previousSecretExpiresAt: expiresAt });
  }
}
// … WebhookCustomerNotFoundError, WebhookDeadLetterNotFoundError, WebhookDeadLetterAlreadyReplayedError
```

Observação sobre `WEBHOOK_INVALID_URL`: `validate.middleware.ts` transforma **qualquer** `ZodError` em `ValidationError` (`VALIDATION_ERROR`). Para emitir o código específico sem abrir mão de validar com Zod ([09:23] Sofia), o body schema valida `url` como `z.string().url()`, e o `WebhookService` aplica `httpsUrlSchema.safeParse(url)` (exportado de `webhook.schemas.ts`), lançando `WebhookInvalidUrlError` quando falha.

Erros internos do worker não viram resposta HTTP. Eles vão para `webhook_outbox.lastError`, `webhook_deliveries.errorCode` e `webhook_dead_letter.reason`, e aparecem nos logs.

## 8. Estratégias de resiliência

| ID | Estratégia | Valor | Fonte |
|---|---|---|---|
| FDD-RES-01 | Timeout por chamada HTTP | 10s (`AbortSignal.timeout`) | [09:42] Diego |
| FDD-RES-02 | Retries com backoff exponencial | 1m / 5m / 30m / 2h / 12h | [09:17] Diego |
| FDD-RES-03 | Teto de tentativas → DLQ | 5 reenvios, depois `webhook_dead_letter` | [09:15] Diego, [09:18] Diego |
| FDD-RES-04 | Fallback operacional | Replay manual pelo ADMIN | [09:18] Diego |
| FDD-RES-05 | Isolamento de falha | Worker em processo separado, API não depende dele | [09:11] Diego |
| FDD-RES-06 | Consistência transacional | Outbox no mesmo `$transaction`, falha → rollback | [09:40] Bruno |
| FDD-RES-07 | Recuperação de crash do worker | `PROCESSING → PENDING` no startup (seguro graças ao at-least-once) | **[Proposta FDD]** sobre [09:24] Diego |
| FDD-RES-08 | Sem sobreposição de ciclos | Próximo poll só depois de terminar o lote | **[Proposta FDD]** |
| FDD-RES-09 | Falhas não-retentáveis vão direto à DLQ | payload > 64KB, sem secret | [09:23] Sofia |

Não há fallback por e-mail nesta fase ([09:37] Larissa) nem rate limiting de saída ([09:39] Larissa).

Configuração em `src/config/env.ts`, com defaults vindos da reunião:

```ts
WEBHOOK_POLL_INTERVAL_MS: z.coerce.number().int().positive().default(2000),   // 09:09 Diego
WEBHOOK_HTTP_TIMEOUT_MS:  z.coerce.number().int().positive().default(10000),  // 09:42 Diego
WEBHOOK_MAX_PAYLOAD_BYTES: z.coerce.number().int().positive().default(65536), // 09:24 Diego
WEBHOOK_BATCH_SIZE:       z.coerce.number().int().positive().default(10),     // "batch pequeno" 09:08 — valor [Proposta FDD]
```

## 9. Observabilidade

O projeto não tem biblioteca de métricas nem de tracing, e a decisão foi não adicionar nada novo ([09:29] Bruno, [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)). A observabilidade da v1 usa **logs Pino estruturados** e **consultas SQL às tabelas novas**.

### 9.1 Logs (Pino, `src/shared/logger/index.ts`)

| ID | Evento (`msg`) | Nível | Campos | Processo |
|---|---|---|---|---|
| FDD-OBS-01 | `webhook_event_enqueued` | info | eventId, webhookId, orderId, fromStatus, toStatus | API |
| FDD-OBS-02 | `webhook_delivery_attempt` | info | eventId, webhookId, orderId, attempt, statusCode, durationMs, success, errorCode | Worker |
| FDD-OBS-03 | `webhook_retry_scheduled` | warn | eventId, attempts, nextAttemptAt, errorCode | Worker |
| FDD-OBS-04 | `webhook_dead_lettered` | warn | eventId, webhookId, reason, attempts | Worker |
| FDD-OBS-05 | `webhook_replayed` | info | deadLetterId, eventId, adminUserId | API (auditoria, [09:36] Sofia) |
| FDD-OBS-06 | `worker_started` / `worker_shutdown` / `worker_poll_cycle` | info/debug | batchSize, processed, cycleMs | Worker |

**Redação obrigatória:** adicionar `'*.secret'`, `'*.previousSecret'` e `'headers["x-signature"]'` ao `redactPaths` ([09:22] Diego).

### 9.2 Métricas (derivadas de logs e de consultas)

| Métrica | Como obter | Alerta sugerido |
|---|---|---|
| `webhook_outbox_pending` (gauge) | `COUNT(*) WHERE status='PENDING'` emitido a cada ciclo no `worker_poll_cycle` | crescimento contínuo |
| `webhook_oldest_pending_age_seconds` | `NOW() - MIN(created_at)` de pendentes com `attempts = 0` | > 10s (SLA, [09:02] Marcos) |
| `webhook_delivery_latency_ms` p50/p95 | `deliveredAt - createdAt` da outbox | p95 > 10s |
| `webhook_delivery_success_rate` | `webhook_deliveries.success` por webhook/janela | queda abrupta por cliente |
| `webhook_http_duration_ms` p95 | `webhook_deliveries.durationMs` | próximo de 10s |
| `webhook_dead_letter_total` | novas linhas/hora em `webhook_dead_letter` | > 0 |
| `webhook_events_per_webhook_per_minute` | contagem de `webhook_event_enqueued` | subsídio para a decisão de rate limit (RFC-OPEN-01) |

### 9.3 Tracing

- **Chave de correlação ponta a ponta:** `eventId` (`X-Event-Id`). Ele aparece no log de enfileiramento (API), em cada tentativa (worker), em `webhook_deliveries`, na DLQ e no request recebido pelo cliente. Isso permite reconstruir a jornada de um evento e cruzar com os logs do próprio cliente.
- `X-Webhook-Id` identifica o cadastro de destino ([09:44] Sofia).
- A relação com o request HTTP de origem é feita por `orderId` + horário, cruzando com o log `http_request` do `request-logger.middleware.ts` (que tem `requestId`). Propagar `req.id` até o `changeStatus` exigiria mudar sua assinatura e fica como melhoria futura. OpenTelemetry está fora desta fase.

## 10. Integração com o sistema existente

| ID | Arquivo existente | Como o módulo de webhooks se integra |
|---|---|---|
| FDD-INT-01 | `src/modules/orders/order.service.ts` | Em `changeStatus`, logo depois de `tx.orderStatusHistory.create(...)` e antes do `refreshed`, entra `await publishWebhookEvent(tx, order, from, to)`. O `order` carregado no início da transação já tem `id`, `orderNumber`, `customerId` e `totalCents`, que é tudo o que o payload precisa. Usa o `TxClient` (`Prisma.TransactionClient`) já tipado no arquivo. `create()` **não** muda (FDD-EXC-07). ([09:40] Bruno, [09:41] Bruno) |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | Sem alteração. Só transições aprovadas por `canTransition` chegam ao publish, então a máquina de estados continua sendo a fonte de verdade. `events` do webhook aceita só valores de `OrderStatus`. |
| FDD-INT-03 | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts` | Novas classes `Webhook*Error` herdando de `AppError`/`BadRequestError`/`ConflictError`, exportadas no `index.ts`. `NotFoundError` **não** é reaproveitado porque fixa `NOT_FOUND` (§7). ([09:28] Bruno) |
| FDD-INT-04 | `src/middlewares/error.middleware.ts` | **Sem alteração.** Os novos erros caem no ramo `err instanceof AppError` e saem como `{ error: { code: 'WEBHOOK_*', ... } }` ([09:29] Bruno). |
| FDD-INT-05 | `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas do módulo. `requireRole('ADMIN')` no replay ([09:36] Larissa). `req.user.id` preenche `replayedById`. |
| FDD-INT-06 | `src/middlewares/validate.middleware.ts` | `validate({ params, body, query })` com os schemas de `webhook.schemas.ts`, no molde de `order.routes.ts`. |
| FDD-INT-07 | `src/shared/logger/index.ts` | Reuso do `logger`. Inclusão de `*.secret`, `*.previousSecret` e `headers["x-signature"]` em `redactPaths`. |
| FDD-INT-08 | `src/config/database.ts` | O worker chama `createPrismaClient()` para ter sua própria instância. **Não** importa o singleton `prisma` junto com a API ([09:30] Bruno). |
| FDD-INT-09 | `src/config/env.ts` | Novas variáveis `WEBHOOK_*` no `envSchema` com defaults (§8), mantendo o fail-fast de `loadEnv()`. |
| FDD-INT-10 | `src/server.ts` → **novo** `src/worker.ts` | `src/worker.ts` replica o padrão de `bootstrap()` e shutdown por `SIGINT`/`SIGTERM` de `server.ts`, mas inicia `WebhookProcessor` em vez de `app.listen` ([09:11] Larissa). |
| FDD-INT-11 | `src/app.ts` | `buildControllers` instancia `WebhookRepository → WebhookService → WebhookController` (e `WebhookAdminController`) e os devolve no objeto `Controllers`. |
| FDD-INT-12 | `src/routes/index.ts` | `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. O tipo `Controllers` ganha `webhooks` e `webhooksAdmin`. |
| FDD-INT-13 | `src/shared/http/response.ts` | `paginated()` na listagem (FDD-CONTRATO-02). |
| FDD-INT-14 | `prisma/schema.prisma` + `prisma/migrations/` | Novos models e enum (§4), relação `Customer.webhooks`. Nova migration gerada por `npm run db:migrate`. |
| FDD-INT-15 | `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"`, espelhando `start`/`dev`. Nenhuma dependência nova. |
| FDD-INT-16 | `tests/setup.ts`, `tests/helpers/factories.ts` | `beforeEach` passa a limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhook` **antes** de `order`/`customer` (por causa das FKs). Nova factory `createTestWebhook()`. |

Estrutura do módulo (todos os arquivos abaixo são **novos**):
```
src/modules/webhooks/
├── webhook.controller.ts        # CRUD, rotate-secret, deliveries
├── webhook.admin.controller.ts  # replay DLQ
├── webhook.service.ts
├── webhook.repository.ts
├── webhook.routes.ts            # buildWebhookRouter / buildWebhookAdminRouter
├── webhook.schemas.ts           # createWebhookSchema, updateWebhookSchema, httpsUrlSchema, ...
├── webhook.publisher.ts         # publishWebhookEvent(tx, order, from, to)
├── webhook.signer.ts            # HMAC-SHA256, geração de secret
└── webhook.processor.ts         # loop do worker, retry, DLQ
src/worker.ts                    # entry-point do worker
```

## 11. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 (`fetch`, `AbortSignal.timeout` e `node:crypto` nativos). Nenhum pacote novo: `uuid`, `zod`, `pino` e `@prisma/client` já estão em `package.json`.
- **Banco:** MySQL 8 (`docker-compose.yml`). Migration só **aditiva**, sem mudança em tabelas existentes além da relação lógica com `customers`.
- **API existente:** nenhum contrato atual muda. `PATCH /orders/:id/status` continua com a mesma resposta. A latência sobe só pelo custo de uma leitura e N inserts.
- **Deploy:** um novo processo (`npm run worker`), com **uma única instância** ([09:12] Diego). A API e o worker podem ser publicados de forma independente, mas a migration precisa rodar antes dos dois.
- **Clientes:** precisam aceitar HTTPS, validar `X-Signature` e deduplicar por `X-Event-Id`. Isso vai para o portal do desenvolvedor ([09:26] Marcos, [09:40] Marcos).

## 12. Critérios de aceite técnicos

| ID | Critério |
|---|---|
| FDD-AC-01 | Mudar o status de um pedido cujo customer tem webhook ativo inscrito no `to_status` cria exatamente 1 linha `PENDING` na outbox por webhook inscrito, na mesma transação. |
| FDD-AC-02 | Se o insert na outbox falhar (simulado), o status do pedido, o histórico e o estoque ficam como estavam (rollback). |
| FDD-AC-03 | Se nenhum webhook assina o `to_status`, nenhuma linha é criada. |
| FDD-AC-04 | Com o worker rodando e o cliente respondendo 200, o evento fica `DELIVERED` em ≤ 10s, com uma linha `success=true` em `webhook_deliveries`. |
| FDD-AC-05 | O request enviado contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type`, e a assinatura bate com `HMAC-SHA256(secret, rawBody)`. |
| FDD-AC-06 | Um cliente que responde 500 ou estoura o timeout de 10s gera retries nos intervalos 1m/5m/30m/2h/12h e, depois da última falha, uma linha em `webhook_dead_letter` e outbox `FAILED`. |
| FDD-AC-07 | `POST /admin/webhooks/dead-letter/:id/replay` com OPERATOR devolve 403. Com ADMIN devolve 202, e a outbox volta a `PENDING` com o mesmo `eventId`. |
| FDD-AC-08 | `POST /webhooks` com `url` `http://` devolve 400 `WEBHOOK_INVALID_URL`. |
| FDD-AC-09 | A `secret` só aparece na resposta de criação e na de rotação, e nunca nos logs (verificado com `redact`). |
| FDD-AC-10 | Depois de uma rotação, os envios nas 24h seguintes carregam as assinaturas com as duas secrets. Depois disso, só a nova. |
| FDD-AC-11 | Um payload serializado acima de 64KB vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem nenhuma tentativa HTTP. |
| FDD-AC-12 | `GET /webhooks/:id/deliveries` devolve no máximo 100 itens, do mais recente para o mais antigo. |
| FDD-AC-13 | Com o worker parado, as mudanças de status continuam funcionando e os eventos se acumulam em `PENDING`. |

**Estratégia de testes:** Vitest + Supertest, no padrão de `tests/orders.test.ts`. O envio HTTP do `WebhookProcessor` é injetado como dependência (`sender`), o que permite simular 2xx, 5xx, timeout e erro de rede sem rede real. O relógio também é injetado, para validar a tabela de backoff sem esperar.

## 13. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RISK-01 | Outbox fora da transação por erro de implementação (perda silenciosa de eventos) | Baixa | Alto | `publishWebhookEvent` só aceita `tx`. Teste FDD-AC-02. Revisão de Bruno/Diego ([09:41] Diego). |
| FDD-RISK-02 | Worker único parado ou lento (timeouts de 10s em sequência atrasam o lote) | Média | Alto | Alerta `webhook_oldest_pending_age_seconds > 10`. Lote pequeno. Restart automático do processo. Os eventos persistem. |
| FDD-RISK-03 | Ordem por pedido quebrada durante retries (evento novo ultrapassa um em backoff) | Média | Médio | Documentado como limitação, e o cliente reconcilia por `timestamp`/`from_status`. Decisão final em RFC-OPEN-07. |
| FDD-RISK-04 | Vazamento de secret (logs, banco) | Baixa | Alto | `redact`, secret por endpoint, rotação com grace, revisão de segurança ≥ 2 dias úteis ([09:46] Sofia). Proteção em repouso em RFC-OPEN-08. |
| FDD-RISK-05 | Crescimento das tabelas `webhook_outbox`/`webhook_deliveries` | Alta | Médio | Índices adequados. Arquivamento fica para uma fase futura ([09:08] Diego). Monitorar tamanho. |
| FDD-RISK-06 | `X-Timestamp` não é coberto pela assinatura (HMAC só sobre o corpo, [09:22] Sofia), então a detecção de replay pelo cliente se apoia no `timestamp` do corpo | Média | Baixo | Documentar no portal que o `timestamp` assinado é o do corpo. Levar à revisão de segurança. |
| FDD-RISK-07 | Rajada de chamadas a um cliente com muitos pedidos | Média | Médio | Métrica `webhook_events_per_webhook_per_minute`. Decisão de rate limit adiada (RFC-OPEN-01). |
