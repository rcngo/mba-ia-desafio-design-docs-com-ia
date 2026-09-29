# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Engenharia** | Bruno (Pedidos), Diego (Plataforma) |
| **Segurança** | Sofia |
| **Status** | Aprovado em reunião técnica, em detalhamento |
| **Documentos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

O OMS vai passar a **avisar os sistemas dos clientes B2B, por webhook HTTPS, sempre que o status de um pedido mudar**. Cada cliente cadastra uma ou mais URLs, escolhe quais status quer receber e recebe um evento assinado com os dados essenciais do pedido. A entrega é assíncrona, tem retentativas automáticas, fica rastreável por histórico e pode ser reprocessada manualmente pela operação.

Hoje o OMS não tem nenhum mecanismo de notificação externa. A única forma de o cliente saber que algo mudou é consultar a API.

## 2. Problema e motivação

- Três clientes B2B (**Atlas Comercial, MaxDistribuição e Nova Cargo**) pediram formalmente para ser notificados em tempo real quando o status dos pedidos muda (PRD-CTX-01, [09:00] Marcos).
- Hoje eles fazem polling periódico em `GET /orders`, o que deixa a integração **lenta e cara** para eles (PRD-CTX-02, [09:00] Marcos).
- **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre (PRD-CTX-03, [09:00] Marcos). A data pedida depois foi **fim de novembro** ([09:45] Marcos).

## 3. Público-alvo e cenários de uso

**Personas**
| Persona | Necessidade |
|---|---|
| Integrador do cliente B2B (via usuário do OMS que representa o cliente, [09:32] Marcos) | Receber mudanças de status sem polling; cadastrar e gerir endpoints; auditar entregas |
| Operação/Admin do OMS | Reprocessar entregas que falharam de vez, com trilha de auditoria ([09:36] Sofia) |
| Time de desenvolvimento dos clientes | Validar autenticidade (HMAC) e deduplicar eventos, seguindo o portal do desenvolvedor ([09:26] Marcos) |

**Cenários**
1. **Acompanhar entregas.** A Nova Cargo cadastra um webhook só para `SHIPPED` e `DELIVERED` e passa a atualizar o rastreio dela sem polling ([09:33] Marcos).
2. **Cliente em manutenção.** O endpoint da MaxDistribuição fica 2h fora do ar em manutenção planejada. Os eventos são reenviados automaticamente quando ele volta ([09:16] Diego).
3. **Secret vazada.** Um cliente descobre que a secret apareceu no log da aplicação dele. Ele rotaciona pela API e tem 24h para migrar sem perder eventos ([09:21] Sofia, [09:22] Diego).
4. **Falha definitiva.** Depois de ~15h de falhas, o evento vai para a DLQ. Um admin corrige o problema com o cliente e dispara o replay ([09:18] Diego).
5. **Auditoria do cliente.** O cliente consulta as últimas 100 entregas para investigar um evento que diz não ter recebido ([09:34] Marcos).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Fonte |
|---|---|---|---|---|
| PRD-OBJ-01 | Notificação em "tempo real" na percepção do cliente | Latência entre o commit da mudança de status e a entrega bem-sucedida (cliente disponível) | **p95 < 10 s** | [09:02] Marcos |
| PRD-OBJ-02 | Reter e atender os clientes que pediram a feature | Clientes com webhook ativo em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até **fim de novembro** | [09:00] Marcos, [09:45] Marcos |
| PRD-OBJ-03 | Nenhuma mudança de status sem notificação | % de mudanças de status commitadas, com webhook inscrito, que geraram evento na outbox | **100%** (garantido por transação) | [09:40] Bruno |
| PRD-OBJ-04 | Entregar no prazo combinado | Sprints até o deploy, com revisão de segurança incluída | **≤ 3 sprints** | [09:46] Larissa |
| PRD-OBJ-05 | Diminuir a necessidade de polling | Volume de chamadas `GET /orders` dos 3 clientes | Tendência de queda depois da adoção (baseline a medir antes do go-live) | [09:00] Marcos |

## 5. Escopo

### 5.1 Incluso
- Cadastro, listagem, edição e remoção de webhooks por customer, com filtro de status.
- Envio automático de evento `order.status_changed` a cada mudança de status assinada.
- Assinatura HMAC-SHA256, secret por endpoint e rotação com grace period de 24h.
- Retentativas automáticas com backoff, DLQ e replay manual por ADMIN.
- Histórico das últimas 100 entregas por webhook.

### 5.2 Fora de escopo

| ID | Item | Situação | Fonte |
|---|---|---|---|
| PRD-OUT-01 | **Notificação por e-mail** quando o webhook do cliente falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Larissa |
| PRD-OUT-02 | **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta fase. Projeto separado do time de frontend | [09:40] Larissa |
| PRD-OUT-03 | **Rate limiting de envio** por cliente | **Adiado** ("observar e decidir depois") | [09:39] Larissa |
| PRD-OUT-04 | **Webhooks inbound** (cliente enviando para nós) | **Descartado**. Só saída | [09:02] Marcos |
| PRD-OUT-05 | **Garantia de ordem global** e múltiplos workers | **Adiado**. Ordem só por pedido, com worker único | [09:13] Larissa, [09:14] Marcos |
| PRD-OUT-06 | **Exactly-once** | **Descartado** em favor de at-least-once | [09:25] Diego |
| PRD-OUT-07 | **Arquivamento** de eventos entregues (~30 dias) | Fora desta feature | [09:08] Diego |
| PRD-OUT-08 | **Itens do pedido no payload** | **Descartado**. O cliente consulta `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
|---|---|---|
| PRD-FR-01 | O usuário autenticado cadastra um webhook informando `customerId`, `url` e a lista de status que quer receber. | [09:31] Marcos, [09:32] Larissa |
| PRD-FR-02 | A plataforma **gera** a secret do webhook e a devolve **só na criação**. | [09:31] Marcos |
| PRD-FR-03 | O usuário lista os webhooks de um customer. | [09:33] Bruno |
| PRD-FR-04 | O usuário edita um webhook (URL, status assinados, ativo/inativo). | [09:33] Bruno |
| PRD-FR-05 | O usuário remove um webhook. | [09:33] Bruno |
| PRD-FR-06 | Cada webhook recebe só os status que assinou (ex.: só `SHIPPED` e `DELIVERED`). | [09:33] Marcos |
| PRD-FR-07 | Toda mudança de status de pedido gera um evento para cada webhook ativo do customer que assina o novo status. | [09:00] Marcos, [09:40] Bruno |
| PRD-FR-08 | O evento contém `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, **sem itens**. | [09:43] Diego |
| PRD-FR-09 | Cada envio leva assinatura HMAC-SHA256 do corpo em `X-Signature`, junto com `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id`. | [09:20] Sofia, [09:44] Diego, [09:44] Sofia |
| PRD-FR-10 | O cliente pode pedir uma nova secret pela API. A anterior continua válida por 24h. | [09:21] Sofia |
| PRD-FR-11 | Envios que falham são retentados automaticamente com intervalos crescentes (1m, 5m, 30m, 2h, 12h). | [09:17] Diego |
| PRD-FR-12 | Depois da última tentativa, o evento vai para uma fila de falhas definitivas (DLQ), com payload, motivo e horário. | [09:18] Diego |
| PRD-FR-13 | Um **ADMIN** pode reprocessar um item da DLQ. A ação fica registrada com o autor. | [09:18] Diego, [09:36] Sofia |
| PRD-FR-14 | O cliente consulta as **últimas 100 entregas** de um webhook, com sucesso/falha, payload, resposta e tempo de resposta. | [09:34] Marcos |

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
|---|---|---|
| PRD-NFR-01 | Latência de notificação **abaixo de 10s** em operação normal. Mínimo estrutural de até 2s (polling). | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | Entrega **at-least-once**. O cliente deduplica por `X-Event-Id`. | [09:24] Diego, [09:25] Diego |
| PRD-NFR-03 | A mudança de status e o registro do evento são **atômicos**. Nunca muda o status sem gerar evento. | [09:40] Bruno |
| PRD-NFR-04 | URL de webhook **só HTTPS**. `http` é recusado com erro de validação. | [09:23] Sofia |
| PRD-NFR-05 | Payload de no máximo **64KB**. Acima disso, o evento não é enviado e vira erro. | [09:23] Sofia, [09:24] Larissa |
| PRD-NFR-06 | **Timeout de 10s** por chamada ao cliente. | [09:42] Diego |
| PRD-NFR-07 | **Secret única por endpoint**. Nenhuma secret global. | [09:21] Sofia |
| PRD-NFR-08 | Ordem de entrega garantida **só por pedido** e enquanto houver um único worker. | [09:13] Larissa |
| PRD-NFR-09 | Retentativas cobrem indisponibilidades de até **~15h**. | [09:17] Diego, [09:17] Marcos |
| PRD-NFR-10 | **Sem infraestrutura nova**: usa o MySQL e a stack atuais. | [09:07] Diego |
| PRD-NFR-11 | O envio não pode afetar a disponibilidade da API. Roda em processo separado. | [09:11] Diego |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Outbox no MySQL em vez de envio síncrono ou Redis | Consistência total, com latência mínima de polling e tabela crescendo | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado com polling de 2s | Simplicidade, com até 2s de latência e worker único | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15h + DLQ | Cobre manutenções longas, mas a entrega pode chegar muito atrasada | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| At-least-once com `X-Event-Id` | Simples do nosso lado, e o cliente precisa deduplicar | [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) |
| HMAC-SHA256 com secret por endpoint e rotação de 24h | Segurança e contenção de vazamento, com duas secrets válidas durante a rotação | [ADR-005](adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| Reuso dos padrões do projeto | Rapidez e consistência, sem métricas dedicadas na v1 | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Snapshot do payload e filtro na inserção | Evento fiel ao momento da mudança, e mudanças de filtro valem só para eventos futuros | [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Fonte |
|---|---|---|
| PRD-DEP-01 | **Revisão de segurança** da Sofia (≥ 2 dias úteis) sobre HMAC e geração de secret, antes do deploy | [09:46] Sofia |
| PRD-DEP-02 | **Portal do desenvolvedor** atualizado pelo PM: integração via API, validação de HMAC, deduplicação por `X-Event-Id` | [09:26] Marcos, [09:40] Marcos |
| PRD-DEP-03 | **Confirmação do prazo** com a Atlas | [09:47] Marcos |
| PRD-DEP-04 | Novo processo `worker` no ambiente de deploy (mesmo banco) | [09:11] Larissa |
| PRD-DEP-05 | Usuários do OMS que representam os clientes (o cadastro é feito com JWT do nosso sistema) | [09:32] Marcos |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISK-01 | **Perda do cliente Atlas** por atraso na entrega | Média | Alto | Estimativa de 3 sprints já com revisão de segurança. Prazo confirmado com a Atlas. Escopo enxuto, com e-mail, dashboard e rate limit fora ([09:46] Larissa, [09:47] Marcos). |
| PRD-RISK-02 | **Cliente processa evento duplicado** por não implementar deduplicação | Média | Médio | `X-Event-Id` estável. Destaque no portal do desenvolvedor ([09:26] Marcos). |
| PRD-RISK-03 | **Vazamento de secret** do lado do cliente | Média | Alto | Secret por endpoint e rotação com 24h de convivência ([09:21] Sofia, [09:22] Diego). |
| PRD-RISK-04 | **Rajada de notificações** sobrecarrega um cliente (ex.: 50 pedidos/min) | Baixa | Médio | Observar volume por webhook. Decidir rate limit depois ([09:38] Diego, [09:39] Larissa). |
| PRD-RISK-05 | **Eventos fora de ordem** para o mesmo pedido em cenário de retry | Baixa | Médio | Payload traz `from_status`, `to_status` e `timestamp` para o cliente reconciliar. Ponto em aberto no RFC. |
| PRD-RISK-06 | **Worker parado** atrasa todas as notificações | Baixa | Alto | Eventos persistidos na outbox. Alerta de idade do evento pendente mais antigo (FDD §9). |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-AC-01 | Um cliente com webhook inscrito em `SHIPPED` recebe, em menos de 10s, um POST assinado quando um pedido dele passa para `SHIPPED`. |
| PRD-AC-02 | Esse cliente **não** recebe nada quando o pedido passa para um status que ele não assinou. |
| PRD-AC-03 | O cliente valida a assinatura com a secret recebida no cadastro. |
| PRD-AC-04 | Um cadastro com URL `http://` é recusado com mensagem de erro clara. |
| PRD-AC-05 | Com o endpoint do cliente fora do ar por 2h, o evento é entregue automaticamente depois da volta, sem ação manual. |
| PRD-AC-06 | Depois da rotação de secret, eventos assinados com a secret antiga continuam sendo aceitos pelo cliente por 24h. |
| PRD-AC-07 | O cliente consegue ver as últimas 100 entregas com resultado e tempo de resposta. |
| PRD-AC-08 | Só um ADMIN consegue reprocessar um item da DLQ, e o replay fica registrado com o autor. |
| PRD-AC-09 | Nenhuma mudança de status de pedido deixa de gerar evento para os webhooks inscritos. |

## 12. Estratégia de testes e validação

- **Testes automatizados** (Vitest + Supertest, padrão de `tests/orders.test.ts`): CRUD, validações (HTTPS, status), autorização (OPERATOR vs. ADMIN no replay), atomicidade outbox + mudança de status, filtro na inserção.
- **Testes do worker** com cliente HTTP simulado: sucesso, 5xx, timeout de 10s, progressão do backoff, ida para a DLQ, replay.
- **Teste ponta a ponta** integrando `order.service` e worker, já previsto na estimativa ("integração no order.service e testes ponta a ponta", [09:46] Larissa).
- **Revisão de segurança** dedicada de HMAC e geração de secret antes do deploy ([09:46] Sofia).
- **Validação com clientes:** homologação com a Atlas (cliente que puxa o prazo) usando o portal do desenvolvedor como guia ([09:47] Marcos).
- **Em produção:** acompanhar PRD-OBJ-01 (p95 < 10s) e o volume da DLQ nas primeiras semanas.

### Cronograma estimado ([09:46] Larissa)
| Bloco | Esforço |
|---|---|
| Modelagem de outbox e DLQ | 1 sprint |
| Worker e retry | 1 sprint |
| CRUD de configuração e deliveries | ½ sprint |
| Integração no `order.service` e testes ponta a ponta | ½ sprint |
| HMAC, schemas, validações e revisão de segurança | incluídos no total de **3 sprints** |
