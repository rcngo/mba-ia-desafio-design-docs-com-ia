# ADR-001 — Padrão Outbox no MySQL existente para eventos de webhook

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-004](ADR-004-garantia-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](ADR-007-snapshot-do-payload-e-filtro-na-insercao.md)

## Contexto

Os clientes B2B precisam ser notificados quando o status de um pedido muda. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já faz três coisas: atualiza `orders`, insere em `order_status_history` e mexe no estoque (`debitStock`/`replenishStock`).

Na reunião apareceram dois problemas com disparar o HTTP direto nesse ponto:

- a transação já é pesada. Um cliente lento travaria a mudança de status de outros pedidos ([09:04] Bruno);
- se o cliente estiver fora do ar, não faz sentido desfazer a mudança de status ([09:04] Bruno).

Precisamos de um jeito de garantir que **toda mudança de status confirmada gere um evento**, e que **nenhum evento exista para uma mudança desfeita**, sem colocar I/O de rede dentro da transação.

## Decisão

Usar o **padrão Transactional Outbox** sobre o **MySQL que já temos** ([09:06] Diego, [09:08] Larissa):

- Dentro da mesma transação de `changeStatus`, inserir uma linha na tabela `webhook_outbox` com o evento ([09:06] Diego, [09:40] Bruno).
- Se a inserção na outbox falhar, a transação inteira faz rollback: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno, [09:41] Diego).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).
- O envio HTTP fica a cargo de um worker separado (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- A chave primária é UUID, igual ao resto do schema (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`) ([09:51] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo síncrono no `changeStatus`** | Acopla a latência do cliente à transação de status, e não há resposta boa para cliente offline (fazer rollback do status?). "Síncrono está fora de questão" ([09:04] Bruno, [09:06] Diego). |
| **Redis Streams / fila dedicada** | Exige subir e operar mais infraestrutura. Para um time pequeno, Redis Cluster seria overengineering ([09:07] Larissa, [09:07] Diego). |

## Consequências

**Positivas**
- A atomicidade vem de graça: se o commit passou, o evento existe; se houve rollback, o evento some junto ([09:06] Diego).
- Nenhuma infraestrutura nova. Reusa MySQL, Prisma e a mesma `DATABASE_URL`.
- A outbox serve como registro auditável do que foi emitido.

**Negativas / trade-offs**
- A tabela cresce sem parar. O arquivamento de linhas entregues (depois de uns 30 dias) ficou **fora do escopo** desta feature ([09:08] Diego), então o volume precisa ser acompanhado.
- A transação de `changeStatus` ganha um `INSERT` a mais (custo pequeno, mas no caminho crítico).
- A latência passa a depender do intervalo de polling do worker (ver ADR-002). Trocamos "tempo real estrito" por consistência.
