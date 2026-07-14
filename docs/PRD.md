# PRD — Sistema de Webhooks de Notificação de Pedidos

> **Documentos relacionados:** [RFC](RFC.md) (proposta técnica) · [FDD](FDD.md) (design de implementação) · [ADRs](adrs/README.md) (decisões) · [Tracker](TRACKER.md) (rastreabilidade)

## 1. Resumo e contexto da feature

O Order Management System (OMS) vai ganhar um mecanismo de **webhooks outbound**: sempre que o status de um pedido mudar (ex.: `PROCESSING → SHIPPED`), os clientes B2B inscritos recebem uma notificação HTTP assinada em seus próprios endpoints, em menos de 10 segundos. A feature inclui API de configuração (cadastro, edição, remoção, listagem e filtro de eventos por status), histórico de entregas consultável, reenvio automático com backoff em caso de falha e reprocessamento manual administrativo. A decisão técnica foi tomada em reunião entre tech lead, PM, engenharia e segurança (ver [TRANSCRICAO.md](../TRANSCRICAO.md)); este PRD registra o problema, o escopo e os critérios de sucesso.

## 2. Problema e motivação

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente notificação em tempo real de mudanças de status de seus pedidos ([09:00] Marcos). Hoje a única forma de acompanhar um pedido é **polling no `GET /orders`**, o que torna a integração deles lenta e cara ([09:00] Marcos) e carrega nossa API com consultas repetitivas sem valor.

A motivação é também comercial e urgente: a Atlas sinalizou que **pode migrar para o concorrente** se não entregarmos até o fim do trimestre ([09:00] Marcos), e pediu a entrega para **fim de novembro** ([09:45] Marcos).

Para esses clientes, "tempo real" significa **qualquer latência abaixo de 10 segundos** — o essencial é não depender de atualização manual ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

**Público-alvo:** times de integração/engenharia dos clientes B2B da plataforma (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), que consomem a API do OMS de sistema para sistema. O gerenciamento dos webhooks é feito **pela nossa API, com JWT do nosso sistema** — usuários que representam o cliente ([09:32] Marcos); não há painel visual nesta fase ([09:40] Larissa).

**Cenários de uso:**

1. **Integração recebe status em tempo real** — O sistema da Atlas cadastra um webhook para `SHIPPED` e `DELIVERED`. Quando um pedido é despachado, o endpoint deles recebe em segundos um POST assinado com número do pedido e transição de status, e dispara o fluxo logístico interno sem nenhum polling ([09:33] Marcos).
2. **Cliente escolhe só o que interessa** — A MaxDistribuição quer saber apenas de entregas: inscreve o webhook só em `DELIVERED` e não recebe ruído dos demais status ([09:33] Marcos).
3. **Cliente audita o que recebeu** — Após um incidente no sistema deles, a Nova Cargo consulta `GET /webhooks/:id/deliveries` e vê as últimas entregas com sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos).
4. **Cliente fica indisponível e nada se perde** — O endpoint da Atlas cai em manutenção de 2h ([09:16] Diego relata caso real). O sistema retenta com backoff e entrega quando o endpoint volta; se falhar 5 vezes, o evento vai para a DLQ e um admin pode reprocessá-lo ([09:17]–[09:18]).
5. **Segurança valida a origem** — O time da Atlas verifica o header `X-Signature` (HMAC-SHA256) com a secret exclusiva do endpoint e descarta qualquer chamada forjada ([09:19]–[09:21] Sofia).
6. **Rotação de credencial sem downtime** — Suspeita de vazamento de secret ([09:22] Diego relata caso real): o cliente pede nova secret pela API e migra em até 24h enquanto a antiga ainda vale ([09:21] Sofia).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
|---|---|---|---|
| O1 | Notificação percebida como tempo real | Latência entre commit da mudança de status e entrega no endpoint do cliente | **< 10 segundos** em condições normais ([09:02] Marcos); piso técnico de ~2s do polling aceito ([09:10] Larissa) |
| O2 | Eliminar o polling dos clientes integrados | Consumo de `GET /orders` pelos clientes com webhook ativo | Redução expressiva após adoção (baseline a medir; o polling é a dor relatada em [09:00]) |
| O3 | Confiabilidade de entrega | Eventos commitados com destino inscrito que são entregues ou registrados em DLQ (nenhuma perda silenciosa) | 100% — garantia estrutural do outbox ([09:06] Diego) |
| O4 | Prazo comercial | Data de disponibilização em produção | Fim de novembro ([09:45] Marcos); estimativa de 3 sprints ([09:46] Larissa) |
| O5 | Retenção dos clientes solicitantes | Atlas, MaxDistribuição e Nova Cargo ativos e integrados via webhook | 3 de 3 integrados; risco de churn da Atlas neutralizado ([09:00] Marcos) |

## 5. Escopo (incluso e fora de escopo)

### Incluso

- Notificação HTTP outbound de **mudança de status de pedido** (`order.status_changed`) para endpoints cadastrados ([09:02] Marcos — só saindo de nós para eles).
- CRUD de configuração de webhook por customer: criar (com secret gerada e devolvida), editar, remover, listar ([09:31]–[09:33]).
- Filtro de eventos por lista de status, aplicado na inserção ([09:33]–[09:34]).
- Entrega assinada (HMAC-SHA256), TLS obrigatório, limite de payload de 64KB ([09:19]–[09:24]).
- Retry automático com backoff e DLQ; replay manual por admin com auditoria ([09:15]–[09:18], [09:36]).
- Histórico de entregas por webhook ([09:34]).
- Rotação de secret via API com grace period de 24h ([09:21]).

### Fora de escopo (descartado ou adiado na reunião)

| Item | Decisão | Origem |
|---|---|---|
| **Notificação por e-mail** quando o webhook do cliente falha repetidamente | Adiado para próxima fase, "depois que a gente medir o impacto" | [09:37] Larissa |
| **Dashboard visual** para o cliente acompanhar seus webhooks | Fora de escopo; painel é projeto separado do time de frontend | [09:40] Larissa |
| **Rate limiting de envio** por cliente | Não entra; "observar e decidir depois" | [09:39] Diego/Larissa |
| **Arquivamento** das linhas entregues da outbox (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| **Múltiplos workers** em paralelo (escala horizontal do consumo) | "Problema do futuro"; fase atual é single-worker | [09:13] Diego |
| **Webhooks inbound** (cliente enviando eventos para nós) | Fora de escopo; só outbound | [09:02] Marcos |
| **Garantia de ordering global** de eventos | Não requerida pelos clientes; garantia limitada a por-pedido com single-worker, documentada como limitação | [09:13]–[09:14] |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| RF-01 | Cadastrar webhook via `POST`, informando URL (https), lista de status de interesse e customer; a secret é **gerada pela plataforma e devolvida na criação** | [09:31] Marcos |
| RF-02 | Editar configuração de webhook via `PATCH` (URL, filtros, estado ativo) | [09:33] Bruno |
| RF-03 | Remover webhook via `DELETE` | [09:33] Bruno |
| RF-04 | Listar os webhooks de um customer via `GET` | [09:33] Bruno |
| RF-05 | Cada webhook escolhe quais status quer receber (ex.: só `SHIPPED` e `DELIVERED`); o filtro é aplicado **na inserção na outbox** — status sem inscritos não gera evento | [09:33] Marcos, [09:34] Bruno |
| RF-06 | Ao mudar o status de um pedido, notificar via HTTP POST assinado todos os webhooks inscritos do customer, com payload contendo event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id e total_cents (sem items) | [09:31] Marcos, [09:43] Diego |
| RF-07 | Consultar histórico de entregas por webhook (`GET /webhooks/:id/deliveries`): últimas ~100 entregas com sucesso/falha, payload, response e tempo de resposta | [09:34] Marcos |
| RF-08 | Retentar entregas com falha automaticamente (backoff 1m/5m/30m/2h/12h, 5 tentativas) e mover falhas permanentes para DLQ persistida | [09:15]–[09:18] Diego |
| RF-09 | Reprocessar item da DLQ manualmente via endpoint admin (`POST /admin/webhooks/dead-letter/:id/replay`), restrito a role ADMIN e com registro de quem executou | [09:18] Diego, [09:36] Sofia |
| RF-10 | Rotacionar a secret de um webhook via API, mantendo a anterior válida por 24h | [09:21] Sofia |
| RF-11 | Recusar cadastro/edição com URL não-https (erro de validação) | [09:23] Sofia |
| RF-12 | CRUD de configuração acessível a qualquer usuário autenticado (JWT); `customer_id` informado no body/path, não derivado do JWT | [09:32] Larissa, [09:37] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| RNF-01 | Latência de notificação < 10s em condições normais; ciclo de polling do worker de 2s (piso de latência aceito) | [09:02] Marcos, [09:10] Larissa |
| RNF-02 | A mudança de status **não pode ser bloqueada nem revertida** por indisponibilidade do cliente — entrega assíncrona e desacoplada | [09:04] Bruno |
| RNF-03 | Consistência: evento registrado se e somente se a mudança de status commitou (outbox transacional) | [09:06] Diego, [09:40] Bruno |
| RNF-04 | Garantia de entrega **at-least-once**; deduplicação pelo cliente via `X-Event-Id` (documentada no portal) | [09:24]–[09:26] |
| RNF-05 | Autenticidade e integridade: HMAC-SHA256 do corpo, secret única por endpoint, rotação com grace de 24h | [09:20]–[09:22] Sofia |
| RNF-06 | Transporte exclusivamente TLS (https) | [09:23] Sofia |
| RNF-07 | Payload limitado a 64KB; acima disso, erro (nunca truncar) | [09:23]–[09:24] |
| RNF-08 | Timeout de 10s por chamada ao cliente; estouro conta como falha | [09:42] Diego |
| RNF-09 | Ordering garantida apenas por pedido e enquanto single-worker; sem ordering global (limitação documentada) | [09:12]–[09:13] |
| RNF-10 | Worker em processo separado da API; restart da API não interrompe entregas | [09:11] Diego |
| RNF-11 | Performance da outbox: índices em status e created_at, leitura em batch pequeno | [09:08] Diego |
| RNF-12 | Nenhuma infraestrutura nova (sem broker/Redis); reuso da stack MySQL + Prisma + padrões do projeto | [09:07] Diego, [09:30] Larissa |

## 8. Decisões e trade-offs principais

Resumo — cada decisão tem ADR próprio com contexto e consequências completas:

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Outbox transacional no MySQL (vs. fila externa/síncrono) | Latência atrelada a polling e crescimento de tabela, em troca de atomicidade e zero infra nova | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) |
| Worker separado, polling 2s (vs. trigger/mesmo processo) | Consultas constantes ao banco e piso de 2s, em troca de simplicidade e isolamento de falha | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| 5 tentativas com backoff + DLQ (vs. 3 tentativas/retry infinito) | Até ~15h para declarar falha permanente, em troca de tolerância a indisponibilidades reais | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint (vs. secret global) | Gestão de N secrets, em troca de raio de dano contido em vazamento | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` (vs. exactly-once) | Cliente precisa deduplicar, em troca de protocolo simples e padrão de mercado | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso integral dos padrões do projeto | Herda limitações da stack, em troca de consistência e velocidade | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Payload snapshot na inserção (vs. render no envio) | Duplicação de dados em disco, em troca de eventos imutáveis e coerentes | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

## 9. Dependências

- **Código existente**: transação de `changeStatus` em `src/modules/orders/order.service.ts` (ponto único de integração), máquina de estados (`src/modules/orders/order.status.ts`), middlewares de auth/erro/validação, logger Pino, schema Prisma/MySQL.
- **Times/pessoas**: revisão de segurança da Sofia (≥2 dias úteis antes do deploy, foco em HMAC e geração de secret — [09:46]); Marcos documenta a integração (dedup por `X-Event-Id`, verificação de assinatura) no portal de desenvolvedor e alinha prazo com os clientes ([09:26], [09:47]).
- **Clientes**: precisam expor endpoint `https` e implementar verificação de HMAC + deduplicação — condição de sucesso da integração.
- **Sessão de revisão do design**: Larissa marca revisão do doc com Bruno e Diego antes de começar a codar ([09:50] Larissa).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Perda do prazo de fim de novembro → churn da Atlas ([09:00]) | Média (3 sprints estimados, sem folga — [09:46]) | Alto — perda de cliente para concorrente | Escopo enxuto já negociado (e-mail, dashboard e rate limiting cortados); revisão de segurança embutida na estimativa; PM confirma prazo com o cliente imediatamente ([09:47]) |
| Vazamento de secret de cliente (já ocorreu antes — [09:22] Diego) | Média | Alto — chamadas forjadas em nome da plataforma | Secret única por endpoint (contém o raio de dano — [09:21] Sofia), rotação via API com grace de 24h, secrets fora de logs (Pino redact), revisão de segurança dedicada ([09:46]) |
| Worker parado sem detecção → eventos acumulam e SLO de 10s estoura | Baixa | Médio — notificações atrasadas em massa | Processo separado com restart independente ([09:11]); monitorar profundidade/idade da outbox (FDD §9); eventos não se perdem (outbox persistente) |
| Cliente sem deduplicação processa duplicatas (at-least-once) | Média | Baixo/Médio — efeitos colaterais duplicados no sistema do cliente | Documentação destacada no portal de desenvolvedor ([09:26] Marcos); `X-Event-Id` estável e payload imutável entre reenvios |
| Acoplamento transacional: bug na escrita da outbox bloqueia mudança de status | Baixa | Alto — operação principal do OMS afetada | Publisher mínimo e função pura ([09:41]); testes da transação estendida cobrindo rollback (FDD §12); suite existente como rede de proteção |

## 11. Critérios de aceitação

1. Cliente cadastra webhook com URL https e filtros de status, e recebe a secret na resposta de criação (RF-01); URL http é recusada (RF-11).
2. Mudança de status de pedido com webhook inscrito gera notificação entregue em < 10s, com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e payload no formato definido (RF-06, RNF-01).
3. Mudança para status não inscrito não gera evento nem entrega (RF-05).
4. Endpoint indisponível: sistema retenta em ~1m/5m/30m/2h/12h; após a 5ª falha o evento aparece na DLQ com payload e motivo (RF-08).
5. Admin (e somente admin) reprocessa item da DLQ; a ação fica registrada com o usuário executor (RF-09).
6. Histórico de entregas exibe sucesso/falha, payload, response e tempo de resposta das últimas entregas (RF-07).
7. Rotação de secret: nova secret emitida, antiga válida por exatamente 24h em paralelo (RF-10).
8. Falha do cliente não bloqueia nem reverte mudança de status de pedido (RNF-02); rollback da transação não deixa evento fantasma (RNF-03).
9. Cliente que recebe duplicata consegue deduplicar pelo `X-Event-Id` — mesmo id e mesmo corpo em qualquer reenvio (RNF-04).
10. Nenhum item da seção "Fora de escopo" foi implementado nesta fase.

## 12. Estratégia de testes e validação

- **Unitários** (Vitest, já configurado — `vitest.config.ts`): geração/verificação de HMAC-SHA256; cálculo do agendamento de backoff; filtro de inscrição por status; validação Zod (https, filtros válidos, limite de 64KB).
- **Integração** (padrão da suite existente em `tests/orders.test.ts` com supertest): CRUD completo de webhooks; `changeStatus` inserindo na outbox na mesma transação — incluindo caso de rollback (falha simulada na outbox ⇒ status não muda); replay de DLQ com e sem role ADMIN (reusando `tests/helpers/factories.ts`).
- **Ponta a ponta** (previsto na estimativa da sprint 3 — "integração no order.service e testes ponta a ponta" [09:46] Larissa): pedido muda de status ⇒ endpoint fake recebe POST assinado; endpoint fake respondendo 5xx ⇒ ciclo completo de retry até DLQ (com relógio/backoff controlado em teste).
- **Resiliência**: kill do worker no meio de um batch ⇒ nenhum evento perdido após restart; timeout de 10s tratado como falha.
- **Validação de segurança** (obrigatória antes do deploy): revisão da Sofia, ≥2 dias úteis, com foco em HMAC e geração de secret ([09:46] Sofia).
- **Validação com cliente**: integração piloto com a Atlas (cliente mais crítico) antes do anúncio geral; Marcos comunica os clientes ([09:47], [09:49]).
- **Regressão**: suite existente (`tests/auth.test.ts`, `tests/orders.test.ts`) verde — contrato atual da API preservado.
