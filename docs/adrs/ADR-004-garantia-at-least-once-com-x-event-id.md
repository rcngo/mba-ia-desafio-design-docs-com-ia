# ADR-004 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Sofia (Segurança), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-005](ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

Com outbox, worker e retry, há situações em que o mesmo evento pode ser enviado mais de uma vez. Exemplos: o cliente processou mas a resposta estourou o timeout de 10s, o worker caiu depois do envio e antes de marcar como entregue, ou um admin fez replay da DLQ. Precisávamos escolher a semântica de entrega que prometemos ao cliente.

## Decisão

- A plataforma garante **at-least-once**. O cliente pode receber o mesmo evento duas vezes e precisa estar preparado ([09:24] Diego).
- Todo evento leva o header **`X-Event-Id`** com um UUID gerado quando o evento entra na outbox. O valor é único por evento e **se mantém igual em todos os reenvios**. O cliente deduplica por ele ([09:25] Diego).
- O PM vai destacar esse comportamento no portal do desenvolvedor ([09:26] Marcos).
- Decisão formalizada em [09:26] Larissa.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exige coordenação dos dois lados e fica muito mais complexo. At-least-once com `event_id` resolve 99% dos casos, e é o que Stripe e GitHub fazem ([09:25] Diego). |
| **At-most-once (sem retry)** | Incompatível com a política de retry já decidida ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) e com o objetivo de não perder notificações. Não chegou a ser defendida na reunião; aparece aqui como alternativa plausível descartada por consequência. |

## Consequências

**Positivas**
- O modelo é simples do nosso lado e compatível com retry, replay e crash do worker.
- Segue o padrão de mercado, que os clientes já conhecem.

**Negativas / trade-offs**
- A responsabilidade de idempotência fica com o cliente ([09:25] Sofia). Se ele não deduplicar, pode processar duas vezes.
- Depende de documentação clara no portal ([09:26] Marcos).
- O `event_id` precisa ser gerado na inserção e persistido. Gerar no envio quebraria a deduplicação.
