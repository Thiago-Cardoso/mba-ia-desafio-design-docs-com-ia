# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-07-14 |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) |

## TL;DR

Vamos notificar clientes B2B via **webhooks HTTP outbound** sempre que o status de um pedido mudar. A publicação usa o **padrão Transactional Outbox no MySQL existente**: o evento é inserido na tabela `webhook_outbox` dentro da mesma transação que já atualiza `orders`, `order_status_history` e estoque. Um **worker em processo separado** consome a outbox por polling a cada 2 segundos e entrega os eventos com **assinatura HMAC-SHA256**, **retry com backoff exponencial (5 tentativas)** e **DLQ** para falhas permanentes. Garantia **at-least-once**, com deduplicação pelo cliente via header `X-Event-Id`. Nenhuma infraestrutura nova; reuso integral dos padrões do projeto. Estimativa: **3 sprints**, incluindo revisão de segurança.

## Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente notificação em tempo real de mudanças de status dos seus pedidos. Hoje eles fazem polling no `GET /orders`, o que torna a integração lenta e cara do lado deles ([09:00] Marcos). "Tempo real" para esses clientes significa **latência abaixo de 10 segundos** ([09:02] Marcos). Há pressão comercial: a Atlas sinalizou migração para concorrente se não entregarmos até o fim do trimestre ([09:00] Marcos), com prazo pedido para fim de novembro ([09:45] Marcos).

O OMS atual (Node.js + TypeScript + Express + Prisma/MySQL) **não possui nenhum mecanismo de notificação externa, eventos, filas ou webhooks**. O ciclo de vida do pedido é bem controlado — máquina de estados em `src/modules/orders/order.status.ts`, transação de mudança de status em `src/modules/orders/order.service.ts` (`changeStatus`) com auditoria em `order_status_history` — mas nada sai do sistema.

Escopo confirmado: webhooks **apenas outbound** (nós → cliente); o cliente não envia nada para nós ([09:02] Marcos, [09:03] Sofia).

## Proposta técnica

### Visão geral

```
                         mesma transação SQL
┌─────────────────────────────────────────────────────┐
│ PATCH /orders/:id/status                            │
│  OrderService.changeStatus:                         │
│   1. update orders                                  │
│   2. insert order_status_history                    │
│   3. ajuste de estoque                              │
│   4. insert webhook_outbox  ← NOVO (se houver       │
│      webhook do customer inscrito no status)        │
└─────────────────────────────────────────────────────┘
                     │ commit
                     ▼
        ┌──────────────────────┐   polling 2s    ┌─────────────────────┐
        │   webhook_outbox     │◄────────────────│  Worker (processo   │
        │ (pendente/processando│                 │  separado,          │
        │  /falhou/entregue)   │────────────────►│  src/worker.ts)     │
        └──────────────────────┘   batch pequeno └──────────┬──────────┘
                     ▲                                      │ HTTP POST + HMAC
        replay admin │                                      ▼
        ┌────────────┴─────────┐    5 falhas     ┌─────────────────────┐
        │ webhook_dead_letter  │◄────────────────│  Endpoint do        │
        └──────────────────────┘                 │  cliente (https)    │
                                                 └─────────────────────┘
```

### Componentes

1. **Publicação transacional (outbox)** — Uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o transaction client da transação corrente e insere o evento na `webhook_outbox` ([09:41] Bruno/Diego). O `OrderService.changeStatus` a chama dentro do `$transaction` existente: se a inserção na outbox falhar, a mudança de status sofre rollback ([09:40] Bruno). O filtro de eventos é aplicado **na inserção**: se nenhum webhook do customer está inscrito naquele status, nada é inserido ([09:34] Bruno). Detalhes no [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md).

2. **Worker de entrega** — Processo Node separado (`src/worker.ts`, script `npm run worker`), mesma stack e mesmo banco, instância própria de `PrismaClient`. Poll a cada 2s dos eventos pendentes mais antigos, em batch pequeno; single-worker nesta fase, com ordering implícita por `order_id` ([09:09]–[09:13]). Timeout de 10s por chamada HTTP ([09:42] Diego). Detalhes no [ADR-002](adrs/ADR-002-worker-separado-com-polling.md).

3. **Resiliência** — Falha de entrega → retry com backoff exponencial 1m/5m/30m/2h/12h, 5 tentativas no total; depois disso, o evento vai para a tabela `webhook_dead_letter` com payload, motivo e timestamp. Replay manual via endpoint admin (role `ADMIN`, com log de auditoria) ([09:14]–[09:18], [09:36]). Detalhes no [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md).

4. **Segurança** — Assinatura HMAC-SHA256 do corpo em `X-Signature`, secret única por endpoint (gerada por nós, devolvida na criação), rotação via API com grace period de 24h; URLs `https` obrigatórias; payload limitado a 64KB ([09:19]–[09:24]). Detalhes no [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).

5. **Semântica de entrega** — At-least-once; deduplicação pelo cliente via `X-Event-Id` (UUID gerado na inserção da outbox). Payload é snapshot renderizado na inserção ([09:24]–[09:26], [09:52]). Detalhes nos [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) e [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md).

6. **API de configuração** — CRUD de webhooks (`POST`/`PATCH`/`DELETE`/`GET`) autenticado com JWT normal, `customer_id` no body/path (o JWT é do usuário operador, não do cliente — [09:32] Larissa); histórico de entregas em `GET /webhooks/:id/deliveries`; novo módulo `src/modules/webhooks/` seguindo o padrão da codebase ([09:27]–[09:33]). Detalhes no [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md); contratos completos no [FDD](FDD.md).

## Alternativas consideradas

| Alternativa | Trade-off que motivou o descarte | Origem |
|---|---|---|
| **Disparo síncrono dentro do `changeStatus`** | A transação já é pesada (orders + history + estoque); um HTTP call no meio faria cliente lento travar mudanças de status de outros pedidos, e cliente fora do ar não pode causar rollback de mudança de status. | [09:04] Bruno, [09:06] Diego |
| **Fila externa (Redis Streams)** | Resolveria o desacoplamento, mas exige subir e operar infra nova. Para um time pequeno é overengineering; a outbox no MySQL existente entrega a mesma garantia sem infra adicional. | [09:07] Larissa/Diego |
| **Trigger no MySQL para acordar o worker** | MySQL não notifica processo externo (sem equivalente ao NOTIFY/LISTEN do Postgres); trigger só executa SQL. O ganho de reatividade não compensa o improviso, e polling de 2s já atende o requisito de <10s. | [09:09] Bruno/Diego |
| **Garantia exactly-once** | Exigiria coordenação dos dois lados e protocolo muito mais complexo. At-least-once com `event_id` é o padrão de mercado (Stripe, GitHub) e resolve 99% dos casos. | [09:25] Diego |
| **Retry indefinido** ou **apenas 3 tentativas** | Indefinido deixa evento pendurado para sempre se o cliente sumiu; 3 tentativas (~30 min) mataria eventos em indisponibilidades reais de ~2h já observadas. 5 tentativas/~15h equilibra. | [09:15]–[09:16] Diego |

## Questões em aberto

1. **Rate limiting de saída** — Se um cliente tem 50 pedidos mudando de status em um minuto, hoje ele receberá 50 chamadas. Decidido **observar em produção e implementar se virar problema** ([09:38]–[09:39] Diego/Larissa).
2. **Escala para múltiplos workers** — Single-worker garante ordering por `order_id`; escalar exigirá particionamento por `order_id` ou lock pessimista. Explicitamente adiado: "problema do futuro" ([09:13] Diego). Registrado como limitação conhecida ([09:13] Larissa).
3. **Arquivamento da outbox** — Linhas entregues devem ser arquivadas "depois de 30 dias ou assim", mas a política exata está fora do escopo desta feature ([09:08] Diego).
4. **Endurecimento de permissões do CRUD** — Por enquanto qualquer role autenticada gerencia webhooks; "mais pra frente a gente pode endurecer" ([09:37] Sofia).

## Impacto e riscos

- **Impacto no código existente**: pontual e concentrado — a única alteração em código existente é a chamada a `publishWebhookEvent` dentro da transação do `changeStatus` em `src/modules/orders/order.service.ts` ([09:40] Bruno). Todo o resto é módulo novo (`src/modules/webhooks/`), entry-point novo (`src/worker.ts`) e tabelas novas.
- **Impacto no banco**: 3 tabelas novas (configuração de webhooks, outbox, DLQ) + escrita adicional por mudança de status. Índices em status e `created_at` na outbox controlam o custo de leitura do worker ([09:08] Diego).
- **Impacto operacional**: um processo novo para deployar e monitorar (worker). Se o worker parar, eventos acumulam na outbox — não se perdem, mas a latência de entrega cresce até o restart.
- **Risco de segurança**: geração e manuseio de secrets e HMAC são código sensível; a Sofia reservará **pelo menos 2 dias úteis** para revisão de segurança antes do deploy ([09:46] Sofia).
- **Risco de prazo**: estimativa de 3 sprints ([09:46] Larissa) contra o prazo de fim de novembro da Atlas ([09:45] Marcos) — apertado, mas factível; a revisão de segurança já está dentro da estimativa.
- **Risco de integração do cliente**: at-least-once exige deduplicação do lado do cliente; mitigado com documentação destacada no portal de desenvolvedor ([09:26] Marcos).

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-padrao-outbox-no-mysql.md)
- [ADR-002 — Worker separado com polling de 2s](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload na inserção](adrs/ADR-007-snapshot-do-payload-na-insercao.md)
