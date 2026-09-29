# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

> **Arquivos novos vs. existentes:** `src/worker.ts` e tudo que fica em `src/modules/webhooks/` são **arquivos novos (a criar)** propostos na reunião ([09:11] Larissa, [09:27] Bruno). Todos os outros caminhos citados existem no repositório atual.

## Contexto

O OMS já tem convenções consolidadas, que dá para ver no código:

| Padrão | Onde está no código |
|---|---|
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/`… |
| Erros de domínio herdando de `AppError`, com `errorCode` em SCREAMING_SNAKE (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| Tratamento centralizado de `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` | `src/middlewares/error.middleware.ts` |
| Validação de body/query/params com Zod | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` |
| Logger Pino estruturado com `redact` | `src/shared/logger/index.ts` |
| Autenticação JWT e autorização por role | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` |
| Composição manual de dependências e montagem de rotas | `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`) |
| IDs UUID | `prisma/schema.prisma` (`@default(uuid()) @db.Char(36)`) |

A feature podia ter trazido bibliotecas ou estruturas novas (fila, logger próprio, handler de erro próprio). O time preferiu que isso não acontecesse.

## Decisão

"Reuso máximo do que já existe", e o webhook vira um módulo igual aos outros ([09:30] Larissa):

- **Módulo `src/modules/webhooks/`** com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` ([09:27] Bruno), mais `webhook.processor.ts` para a lógica do worker ([09:28] Bruno).
- **Erros** como subclasses de `AppError`/`ConflictError`/`UnprocessableEntityError`, com **prefixo `WEBHOOK_`** em todos os códigos do módulo (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa).
- **Pino** como logger, sem biblioteca nova. O `errorMiddleware` já trata os erros novos sem alteração, porque eles herdam de `AppError` ([09:29] Bruno).
- **Schemas Zod** no mesmo estilo de `order.schemas.ts`, incluindo a validação de URL HTTPS ([09:23] Sofia).
- **`requireRole('ADMIN')`** existente no endpoint de replay ([09:36] Larissa).
- **Worker** no mesmo repositório e stack, com `PrismaClient` próprio criado pela `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno).
- A integração com pedidos é uma **função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o `Prisma.TransactionClient`. Assim não é preciso injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Injetar um `WebhookRepository` no construtor do `OrderService`** | Aumenta o acoplamento e muda a composição em `src/app.ts`. A função pura recebendo `tx` resolve com menos atrito ([09:41] Bruno, [09:41] Diego). |
| **Estrutura/serviço separado (outro repositório, outro logger, outro padrão de erro)** | Contraria a decisão explícita de reuso e aumentaria o custo de manutenção para um time pequeno ([09:07] Diego, [09:30] Larissa). Alternativa plausível, não defendida na reunião. |

## Consequências

**Positivas**
- A curva de aprendizado é zero para o time. Code review e testes seguem os mesmos moldes (`tests/*.test.ts` com Supertest).
- As respostas de erro já saem no formato `{ error: { code, message, details } }` do `errorMiddleware`.
- Menos dependências novas, menos superfície de segurança.

**Negativas / trade-offs**
- `NotFoundError` (`src/shared/errors/http-errors.ts`) fixa o código `NOT_FOUND`. Para ter `WEBHOOK_NOT_FOUND`, precisamos de classes novas que herdem direto de `AppError`, em vez de reaproveitar `NotFoundError`.
- O projeto não tem biblioteca de métricas. Seguindo este ADR, a observabilidade da primeira versão se apoia em logs Pino estruturados e em consultas às tabelas (ver FDD). Métricas de verdade ficam para depois.
- O worker herda as limitações da stack (processo Node único, pool Prisma próprio).
