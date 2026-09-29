# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou devolvendo erro. Já tivemos cliente com duas horas de indisponibilidade por manutenção planejada ([09:16] Diego). Precisamos de uma política que aguente janelas longas de indisponibilidade sem deixar eventos pendurados para sempre.

## Decisão

1. **Backoff exponencial com teto de 5 tentativas**. Depois disso, a falha é tratada como permanente ([09:15] Diego, [09:16] Larissa).
2. **Progressão:** 1 min → 5 min → 30 min → 2 h → 12 h. São quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego, [09:17] Larissa). O PM considerou aceitável ([09:17] Marcos).
3. **Timeout de 10 segundos por chamada.** Se o cliente não responde nesse prazo, conta como falha e o evento vai para retry ([09:42] Diego).
4. **DLQ em tabela separada, `webhook_dead_letter`**, com payload, motivo da falha e timestamp. Serve de evidência para debug e reprocessamento ([09:18] Diego).
5. **Replay manual** pelo endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que devolve o evento à outbox como pendente ([09:18] Diego). Exige role `ADMIN` (via `requireRole` de `src/middlewares/auth.middleware.ts`), e quem fez o replay é registrado para auditoria ([09:36] Sofia, [09:36] Larissa).

> **Interpretação registrada.** A reunião fala em "5 tentativas" e lista 5 intervalos que somam cerca de 15h "entre primeira falha e última tentativa". A leitura consistente com os dois dados é: 1 envio inicial + 5 reenvios, um após cada intervalo. O FDD implementa assim, e a confirmação ficou como questão em aberto no [RFC](../RFC.md#questões-em-aberto).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento pode ficar pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Três tentativas cabem em uns 30 minutos. Uma indisponibilidade de manhã mataria o evento, e já houve cliente com 2h de manutenção ([09:16] Bruno, [09:16] Diego). |
| **Marcar como `failed` na própria outbox (sem tabela de DLQ)** | Mistura eventos vivos e mortos na tabela lida pelo worker. A tabela separada deixa a leitura da outbox mais limpa e guarda evidência para debug ([09:17] Larissa, [09:18] Diego). |

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15h sem intervenção.
- A DLQ separada mantém as queries do worker sobre a outbox enxutas e dá visibilidade clara do que falhou de vez.
- O replay manual dá controle operacional sem precisar mexer no banco.

**Negativas / trade-offs**
- Um evento pode chegar com até ~15h de atraso. O cliente pode receber, depois de se recuperar, uma rajada de eventos antigos.
- Os retries podem **quebrar a ordem por pedido**: um evento em backoff pode ser ultrapassado por outro evento mais novo do mesmo pedido. Está registrado como risco e questão em aberto no RFC.
- Reprocessar da DLQ é manual e depende de um ADMIN agir.
- O replay gera reenvio do mesmo `event_id`, então o cliente precisa deduplicar (ver [ADR-004](ADR-004-garantia-at-least-once-com-x-event-id.md)).
