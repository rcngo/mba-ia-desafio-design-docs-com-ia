# Architectural Decision Records

Este diretório guarda os ADRs do projeto, no formato MADR simplificado (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).
Cada arquivo segue o padrão `ADR-NNN-titulo-em-kebab-case.md`.

## Índice — Sistema de Webhooks de Notificação de Pedidos

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s, single-worker | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-garantia-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | Aceito |
| [ADR-005](ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com grace de 24h | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes (módulos, AppError, Pino, Zod, requireRole) | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) | Snapshot do payload e filtro de eventos na inserção | Aceito |
