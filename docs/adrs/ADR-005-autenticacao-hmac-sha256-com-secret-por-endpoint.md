# ADR-005 — Autenticação por HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito
- **Data:** data da reunião técnica de webhooks (quinta-feira, 09:00)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-004](ADR-004-garantia-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Os webhooks mandam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar **autenticidade** (a requisição veio da gente) e **integridade** (o payload não foi adulterado no caminho) ([09:19] Sofia). O fluxo é só de saída, da plataforma para o cliente ([09:02] Marcos, [09:03] Sofia). Já tivemos cliente que vazou secret em log da própria aplicação ([09:22] Diego).

## Decisão

1. **HMAC-SHA256 sobre o corpo do request.** A assinatura vai no header `X-Signature` ([09:20] Sofia, [09:22] Sofia).
2. **Uma secret por endpoint de webhook**, sem secret global da plataforma. Se uma vazar, só aquela é afetada ([09:21] Sofia).
3. A secret é **gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos). A configuração guarda `url + secret + customer_id + estado ativo` ([09:21] Bruno, [09:21] Sofia).
4. **Rotação pela API:** o cliente pede uma nova secret, e a antiga continua válida em paralelo por **24 horas** para dar tempo de migrar. Depois disso, ela é invalidada ([09:21] Sofia).
5. **Só HTTPS:** URL `http://` é recusada com erro de validação no schema Zod. Isso não é decisão arquitetural, é validação, e entra aqui só como contexto ([09:23] Sofia).
6. Antes do deploy, o código de HMAC e de geração de secret passa por **revisão de segurança dedicada, de pelo menos 2 dias úteis** ([09:46] Sofia).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Um vazamento comprometeria todos os clientes ("se vaza uma, vaza tudo") ([09:21] Sofia). |
| **Rotação sem período de convivência** | Obrigaria o cliente a trocar a secret no mesmo instante, com risco de rejeitar eventos legítimos. O grace period de 24h existe para evitar isso ([09:21] Sofia). |
| **Outro algoritmo de assinatura** | Não houve disputa. SHA-256 foi escolhido por ser o padrão de mercado, com biblioteca disponível em qualquer stack de cliente ([09:20] Sofia). |

## Consequências

**Positivas**
- O cliente valida origem e integridade com bibliotecas padrão.
- O raio de impacto de um vazamento fica limitado a um endpoint.
- Rotação sem downtime do lado do cliente.

**Negativas / trade-offs**
- A secret precisa ser guardada **em forma recuperável** (não pode ser hash), porque o worker precisa dela para assinar. A proteção em repouso não foi discutida e ficou como questão para a revisão de segurança.
- Durante as 24h de rotação existem duas secrets válidas. O FDD define como as duas assinaturas são enviadas nesse período, para validação na revisão da Sofia.
- A secret não pode aparecer em log. Isso exige estender o `redact` do logger em `src/shared/logger/index.ts`, que hoje cobre `password`, `token` etc., mas não `secret`.
