# ADR-007 — Payload renderizado (snapshot) e filtro de eventos no momento da inserção na outbox

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-garantia-at-least-once-com-x-event-id.md)

## Contexto

Na hora de gravar o evento na outbox dentro de `changeStatus` (`src/modules/orders/order.service.ts`), havia duas perguntas:

1. **O que guardar:** o payload já montado, ou só o `order_id`, montando o payload no envio? ([09:51] Bruno)
2. **Quando filtrar:** cada webhook assina só alguns status (ex.: só `SHIPPED` e `DELIVERED`). O filtro acontece na inserção ou no envio? ([09:33] Marcos, [09:34] Diego)

## Decisão

1. **Snapshot na inserção.** A outbox guarda o payload JSON já renderizado. Se o pedido mudar depois, o evento continua mostrando o estado do momento da mudança de status ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
2. **Filtro na inserção.** `publishWebhookEvent` só grava linhas para webhooks **ativos** do customer que assinam o `to_status`. Se nenhum assina, nada é gravado ([09:34] Bruno, [09:34] Diego).
3. **Payload enxuto:** `event_id`, `event_type` (`order.status_changed`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. **Sem itens.** Se o cliente quiser detalhes, consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Gera "caso esquisito": um evento `PAID` enviado depois que o pedido já está `SHIPPED` sairia com dados que não batem com a transição ([09:52] Larissa). |
| **Filtrar no envio (worker)** | Gravaria linhas que nunca seriam enviadas. Filtrar na inserção "economiza linha na tabela" ([09:34] Bruno). |
| **Incluir os itens no payload** | Infla o payload sem necessidade. O detalhe está disponível via `GET /orders/:id` ([09:43] Diego). |

## Consequências

**Positivas**
- O evento é imutável e fiel à transição, o que ajuda auditoria e replay da DLQ.
- A outbox só tem trabalho útil.
- O payload fica pequeno, bem abaixo do limite de 64KB ([09:24] Diego).

**Negativas / trade-offs**
- Uma mudança nos filtros ou a desativação de um webhook **não afeta eventos já gravados**. Um webhook desativado depois da inserção ainda teria linhas pendentes, e o FDD define que o worker as descarta.
- A consulta de webhooks do customer entra na transação de `changeStatus` (uma leitura indexada a mais no caminho crítico).
- Replay da DLQ reenvia o snapshot original, não o estado atual do pedido.
