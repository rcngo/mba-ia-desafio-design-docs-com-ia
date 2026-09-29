# Tracker de Rastreabilidade — Webhooks de Notificação de Pedidos

Esta tabela liga cada item identificável dos documentos à sua origem: a fala na [`TRANSCRICAO.md`](../TRANSCRICAO.md) (`[hh:mm] Nome`) ou um arquivo real do código.
Itens marcados como **[Proposta FDD]** nos documentos também aparecem aqui, ligados à fala ou ao arquivo de onde foram derivados.

> **Arquivos novos vs. existentes:** `src/worker.ts` e tudo que fica em `src/modules/webhooks/` são **arquivos novos (a criar)** propostos na reunião ([09:11] Larissa, [09:27] Bruno). Todos os outros caminhos citados existem no repositório atual.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; a integração fica lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco de negócio | Atlas pode migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência p95 < 10s ("tempo real" para o cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 3 de 3 clientes integrados até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Métrica | 100% das mudanças de status com evento (atomicidade) | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Entrega em ≤ 3 sprints, com revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Redução do polling em GET /orders (baseline a medir) | TRANSCRICAO | [09:00] Marcos |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | E-mail em falha repetida adiado para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual descartado (time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída adiado ("observar") | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Webhooks inbound descartados; só saída | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Ordem global e múltiplos workers adiados | TRANSCRICAO | [09:13] Larissa |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo | Arquivamento da outbox (~30 dias) fora da feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-08 | docs/PRD.md | Fora de escopo | Itens do pedido fora do payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com customerId explícito, url e status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01a | docs/PRD.md | Restrição | customerId vai no body/path, não vem do JWT (JWT é do operador) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida só na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro por lista de status assinados | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Evento gerado a cada mudança de status para webhooks inscritos | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Campos do payload, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Headers X-Signature, X-Event-Id, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com 24h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retentativas 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Replay da DLQ só por ADMIN, com registro do autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10s; mínimo de até 2s pelo polling | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre status e evento | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Só HTTPS, validação no schema Zod | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, com erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordem só por order_id, com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Janela de retentativa de ~15h aceita pelo PM | TRANSCRICAO | [09:17] Marcos |
| PRD-NFR-10 | docs/PRD.md | Restrição | Sem infraestrutura nova (MySQL existente) | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Envio em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal do desenvolvedor documentando dedup e integração | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Novo processo worker no deploy | TRANSCRICAO | [09:11] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Usuários do OMS que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Perda da Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não deduplica eventos | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret do lado do cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de notificações (50 pedidos/min) | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Eventos fora de ordem em retry | TRANSCRICAO | [09:12] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Worker parado atrasa notificações | TRANSCRICAO | [09:11] Diego |
| PRD-AC-05 | docs/PRD.md | Critério de aceite | Indisponibilidade de 2h coberta por retry automático | TRANSCRICAO | [09:16] Diego |
| PRD-TEST-01 | docs/PRD.md | Estratégia de teste | Testes ponta a ponta de order.service + worker incluídos na estimativa | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-02 | docs/PRD.md | Estratégia de teste | Testes no padrão Vitest + Supertest existente | CODIGO | tests/orders.test.ts |
| RFC-META-01 | docs/RFC.md | Metadado | Revisores = participantes; sessão de revisão com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-PROP-01 | docs/RFC.md | Proposta | Visão geral: outbox + worker + HMAC + retry + DLQ + at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-02 | docs/RFC.md | Contexto | A transação de changeStatus atualiza orders, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila dedicada | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger MySQL para notificar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido / 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | E-mail de alerta de falha na próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (particionar/lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Endurecer roles do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-06 | docs/RFC.md | Questão em aberto | Ambiguidade "5 tentativas" vs. 5 intervalos (~15h) | TRANSCRICAO | [09:17] Diego |
| RFC-OPEN-07 | docs/RFC.md | Questão em aberto | Ordem por pedido durante retries | TRANSCRICAO | [09:12] Diego |
| RFC-OPEN-08 | docs/RFC.md | Questão em aberto | Proteção da secret em repouso, para revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| RFC-RISK-01 | docs/RFC.md | Risco | Prazo da Atlas em fim de novembro | TRANSCRICAO | [09:45] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, na mesma transação de mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-C1 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em status e created_at na outbox | TRANSCRICAO | [09:08] Diego |
| ADR-001-C2 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | PK UUID seguindo o padrão do schema | TRANSCRICAO | [09:51] Larissa |
| ADR-001-C3 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | changeStatus em prisma.$transaction | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado com polling de 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-C1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry-point src/worker.ts e npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-C2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio no worker, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-C3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Single-worker como limitação conhecida de ordem | TRANSCRICAO | [09:13] Larissa |
| ADR-002-C4 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | Entry-point atual da API usado como modelo | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-C1 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-C2 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | Marcar failed na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-C3 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Contexto | requireRole existente usado no replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-004-C1 | docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup fica com o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005 | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 no corpo, secret por endpoint, rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005-C1 | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Restrição | Config guarda url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-005-C2 | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Redact do logger não cobre "secret" hoje | CODIGO | src/shared/logger/index.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo: módulo, AppError, Pino, error middleware, Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-006-C1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo src/modules/webhooks com a estrutura padrão | TRANSCRICAO | [09:27] Bruno |
| ADR-006-C2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Prefixo WEBHOOK_ em todos os códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-C3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | publishWebhookEvent(tx, …) como função pura | TRANSCRICAO | [09:41] Diego |
| ADR-006-C4 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | AppError e subclasses de domínio | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-C5 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Error middleware trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-C1 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Filtro de status na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-C2 | docs/adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md | Decisão | Payload enxuto, sem itens | TRANSCRICAO | [09:44] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Entrega < 10s com polling de 2s | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-06 | docs/FDD.md | Restrição | Zero dependências novas; Node ≥ 20 (fetch nativo) | CODIGO | package.json |
| FDD-EXC-07 | docs/FDD.md | Fora de escopo | Sem evento na criação do pedido (create não passa por changeStatus) | CODIGO | src/modules/orders/order.service.ts |
| FDD-EXC-08 | docs/FDD.md | Fora de escopo | Listagem da DLQ via API não discutida | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-01 | docs/FDD.md | Modelo de dados | Tabela webhooks (url, secret, customer, ativo, eventos) | TRANSCRICAO | [09:21] Bruno |
| FDD-DATA-02 | docs/FDD.md | Modelo de dados | Tabela webhook_outbox com status pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | docs/FDD.md | Modelo de dados | Tabela webhook_deliveries para histórico (payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-04 | docs/FDD.md | Modelo de dados | Tabela webhook_dead_letter com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-05 | docs/FDD.md | Restrição | Convenções UUID Char(36), @@map e enum OrderStatus reutilizado | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks (201, secret só na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks?customerId (paginado, sem secret) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id (204) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay (ADMIN, 202) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Chamada outbound: headers e body JSON | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08a | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-09 | docs/FDD.md | Restrição | customerId explícito, seguindo o padrão de createOrderSchema/listOrdersQuerySchema | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro de changeStatus; falha → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Loop do worker: lote pequeno dos pendentes mais antigos | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-02a | docs/FDD.md | Decisão | Limite de 64KB verificado no worker (erro, não truncar) | TRANSCRICAO | [09:23] Sofia |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Tabela de backoff e contagem de tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Movimentação para a DLQ e replay de volta a PENDING | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Rotação com previousSecret por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLOW-05a | docs/FDD.md | Restrição | Geração de secret revisada pela segurança | TRANSCRICAO | [09:46] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (não-HTTPS) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-02a | docs/FDD.md | Restrição | validate.middleware converte todo ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (prefixo WEBHOOK_ no módulo) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_ROTATION_IN_PROGRESS [Proposta FDD] | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (64KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED (não-2xx) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR (cliente offline) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-13 | docs/FDD.md | Erro | WEBHOOK_INACTIVE (webhook desativado depois da inserção) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-14 | docs/FDD.md | Restrição | NotFoundError fixa NOT_FOUND; são necessárias classes novas | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Backoff exponencial | TRANSCRICAO | [09:15] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Teto de tentativas e DLQ | TRANSCRICAO | [09:16] Larissa |
| FDD-RES-04 | docs/FDD.md | Resiliência | Replay manual como fallback | TRANSCRICAO | [09:18] Diego |
| FDD-RES-05 | docs/FDD.md | Resiliência | Isolamento por processo separado | TRANSCRICAO | [09:11] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Consistência transacional | TRANSCRICAO | [09:41] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Recuperação PROCESSING → PENDING no startup [Proposta FDD] | TRANSCRICAO | [09:24] Diego |
| FDD-RES-09 | docs/FDD.md | Resiliência | Falhas não-retentáveis direto para a DLQ | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-10 | docs/FDD.md | Configuração | Variáveis WEBHOOK_* no envSchema com defaults | CODIGO | src/config/env.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Log webhook_event_enqueued via Pino | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Log de auditoria do replay com adminUserId | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | Redact de secret e X-Signature nos logs | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-08 | docs/FDD.md | Observabilidade | Alerta de idade do pendente mais antigo > 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-09 | docs/FDD.md | Observabilidade | Métrica de eventos por webhook/minuto para decidir rate limit | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-10 | docs/FDD.md | Observabilidade | Tracing por X-Event-Id / X-Webhook-Id | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-11 | docs/FDD.md | Observabilidade | Correlação com requestId do request logger | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | publishWebhookEvent em changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados sem alteração (canTransition) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas classes Webhook*Error | CODIGO | src/shared/errors/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate() com schemas do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | redactPaths ampliado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | createPrismaClient() no worker | CODIGO | src/config/database.ts |
| FDD-INT-09 | docs/FDD.md | Integração | envSchema com WEBHOOK_* | CODIGO | src/config/env.ts |
| FDD-INT-10 | docs/FDD.md | Integração | src/worker.ts espelhando bootstrap/shutdown | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | buildControllers instancia os controllers do módulo | CODIGO | src/app.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Montagem de /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-13 | docs/FDD.md | Integração | paginated() na listagem | CODIGO | src/shared/http/response.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Novos models e migration | CODIGO | prisma/schema.prisma |
| FDD-INT-15 | docs/FDD.md | Integração | Scripts worker / worker:dev | CODIGO | package.json |
| FDD-INT-16 | docs/FDD.md | Integração | Limpeza das novas tabelas nos testes e factory | CODIGO | tests/setup.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Deploy com uma única instância de worker | TRANSCRICAO | [09:12] Diego |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8 existente | CODIGO | docker-compose.yml |
| FDD-AC-02 | docs/FDD.md | Critério de aceite | Rollback total se a outbox falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-07 | docs/FDD.md | Critério de aceite | OPERATOR recebe 403 no replay | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério de aceite | Duas assinaturas durante as 24h de rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-13 | docs/FDD.md | Critério de aceite | Worker parado não impede mudança de status | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Outbox fora da transação | TRANSCRICAO | [09:41] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Worker único parado ou lento | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Ordem quebrada durante retries | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-04 | docs/FDD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Crescimento das tabelas | TRANSCRICAO | [09:07] Bruno |
| FDD-RISK-06 | docs/FDD.md | Risco | X-Timestamp fora da assinatura (HMAC só no corpo) | TRANSCRICAO | [09:44] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | Rajada de chamadas a um cliente | TRANSCRICAO | [09:38] Diego |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Evento persistido na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Retry por ~15h e depois DLQ com replay | TRANSCRICAO | [09:17] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Autenticidade e integridade por HMAC-SHA256 | TRANSCRICAO | [09:19] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Deduplicação por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-EXC-01 | docs/FDD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | docs/FDD.md | Fora de escopo | E-mail em falhas repetidas | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | docs/FDD.md | Fora de escopo | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | docs/FDD.md | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | docs/FDD.md | Fora de escopo | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | docs/FDD.md | Fora de escopo | Múltiplos workers / ordem global | TRANSCRICAO | [09:13] Diego |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log por tentativa com statusCode e durationMs | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log de retry agendado | TRANSCRICAO | [09:15] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Log de ida para a DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Logs de ciclo, start e shutdown do worker (logger Pino existente) | CODIGO | src/shared/logger/index.ts |
| FDD-RES-08 | docs/FDD.md | Resiliência | Poll seguinte só depois do lote terminar (polling em loop) [Proposta FDD] | TRANSCRICAO | [09:09] Diego |
| FDD-AC-01 | docs/FDD.md | Critério de aceite | 1 linha na outbox por webhook inscrito, na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de aceite | Nenhuma linha quando ninguém assina o status | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-04 | docs/FDD.md | Critério de aceite | Entrega em ≤ 10s com cliente respondendo 200 | TRANSCRICAO | [09:02] Marcos |
| FDD-AC-05 | docs/FDD.md | Critério de aceite | Headers presentes e assinatura válida | TRANSCRICAO | [09:44] Diego |
| FDD-AC-06 | docs/FDD.md | Critério de aceite | Progressão do backoff e DLQ após a última falha | TRANSCRICAO | [09:17] Larissa |
| FDD-AC-08 | docs/FDD.md | Critério de aceite | URL http:// recusada com WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-09 | docs/FDD.md | Critério de aceite | Secret nunca aparece em logs | TRANSCRICAO | [09:22] Diego |
| FDD-AC-11 | docs/FDD.md | Critério de aceite | Payload > 64KB vai para a DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-12 | docs/FDD.md | Critério de aceite | Deliveries limitado a 100 itens | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-01 | docs/PRD.md | Critério de aceite | POST assinado em < 10s para status inscrito | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de aceite | Nada enviado para status não inscrito | TRANSCRICAO | [09:33] Marcos |
| PRD-AC-03 | docs/PRD.md | Critério de aceite | Cliente valida a assinatura com a secret | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-04 | docs/PRD.md | Critério de aceite | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de aceite | Secret antiga aceita por 24h depois da rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de aceite | Consulta das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-08 | docs/PRD.md | Critério de aceite | Replay só por ADMIN, com autor registrado | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-09 | docs/PRD.md | Critério de aceite | Nenhuma mudança de status sem evento | TRANSCRICAO | [09:41] Diego |
