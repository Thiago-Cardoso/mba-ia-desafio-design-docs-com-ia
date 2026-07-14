# FDD — Sistema de Webhooks de Notificação de Pedidos

> **Documentos relacionados:** [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/README.md)
> Este documento detalha o "como construir". O "por quê" está no PRD; a proposta em nível de arquitetura e as alternativas, no RFC.

## 1. Contexto e motivação técnica

O OMS muda o status de pedidos exclusivamente por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` ([09:04] Bruno). Não existe hoje nenhum mecanismo de notificação externa: clientes B2B descobrem mudanças fazendo polling em `GET /orders` ([09:00] Marcos).

A feature adiciona: publicação transacional de eventos (outbox), um worker de entrega em processo separado, e um módulo `src/modules/webhooks/` com CRUD de configuração — tudo dentro dos padrões existentes do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

## 2. Objetivos técnicos

1. **Atomicidade**: nenhuma mudança de status commitada sem evento correspondente na outbox, e vice-versa ([09:40] Bruno; [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md)).
2. **Latência**: evento entregue ao cliente em menos de 10s do commit em condições normais; ciclo de polling de 2s ([09:02] Marcos, [09:09] Diego).
3. **Resiliência**: nenhuma perda silenciosa — retry com backoff (5 tentativas) e DLQ persistida ([ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)).
4. **Segurança**: HMAC-SHA256 por endpoint, TLS obrigatório, rotação de secret com grace period ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).
5. **Zero regressão**: a API existente e seu comportamento transacional permanecem intactos, exceto pela inserção na outbox.

## 3. Escopo e exclusões

**No escopo**: tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`; módulo `src/modules/webhooks/`; entry-point `src/worker.ts`; alteração pontual em `OrderService.changeStatus`; endpoints CRUD + deliveries + rotação de secret + replay de DLQ.

**Fora do escopo** (decisões explícitas da reunião):

- Notificação por e-mail quando webhook falha — próxima fase ([09:37] Larissa).
- Dashboard visual para o cliente — projeto separado do time de frontend ([09:40] Larissa).
- Rate limiting de saída — observar e decidir depois ([09:39] Diego/Larissa).
- Arquivamento de linhas entregues da outbox (~30 dias) — fora do escopo da feature ([09:08] Diego).
- Múltiplos workers em paralelo — problema do futuro ([09:13] Diego).
- Webhooks inbound (cliente → nós) — só outbound ([09:02] Marcos).

## 4. Modelo de dados

Quatro tabelas novas no `prisma/schema.prisma`, seguindo as convenções existentes (id `String @id @default(uuid()) @db.Char(36)`, `@@map` snake_case, timestamps — [09:51] Larissa):

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  FAILED       // falhou (aguardando retry)
  DELIVERED    // entregue
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)      // https obrigatório (validação Zod)
  secret                  String    @db.VarChar(128)       // gerada pela plataforma
  previousSecret          String?   @db.VarChar(128)       // válida durante grace period
  previousSecretExpiresAt DateTime?                        // rotação: antiga vale por 24h
  subscribedStatuses      Json                             // ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id          String              @id @default(uuid()) @db.Char(36)  // = event_id (X-Event-Id)
  webhookId   String              @db.Char(36)
  eventType   String              @db.VarChar(64)     // "order.status_changed"
  payload     Json                                    // snapshot renderizado na inserção (ADR-007)
  status      WebhookOutboxStatus @default(PENDING)
  attempts    Int                 @default(0)
  nextRetryAt DateTime?                               // agenda do backoff
  lastError   String?             @db.VarChar(500)
  createdAt   DateTime            @default(now())
  updatedAt   DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([status])       // [09:08] Diego: índice no status
  @@index([createdAt])    // [09:08] Diego: índice em created_at
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)        // outbox.id
  attempt        Int
  success        Boolean
  httpStatus     Int?                          // null em timeout/erro de rede
  requestPayload Json
  responseBody   String?  @db.Text
  durationMs     Int                           // "tempo de resposta" [09:34] Marcos
  attemptedAt    DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, attemptedAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id            String   @id @default(uuid()) @db.Char(36)
  eventId       String   @db.Char(36)         // outbox.id original
  webhookId     String   @db.Char(36)
  payload       Json                          // payload exato que falhou
  failureReason String   @db.VarChar(500)     // motivo da falha [09:18] Diego
  failedAt      DateTime @default(now())      // timestamp [09:18] Diego
  replayedAt    DateTime?                     // preenchido no replay

  @@map("webhook_dead_letter")
}
```

Estrutura do módulo (espelha `src/modules/orders/`):

```
src/modules/webhooks/
├── webhook.controller.ts
├── webhook.service.ts
├── webhook.repository.ts
├── webhook.routes.ts
├── webhook.schemas.ts
├── webhook.publisher.ts    // publishWebhookEvent(tx, order, fromStatus, toStatus)
└── webhook.processor.ts    // lógica do worker [09:28] Bruno
src/worker.ts               // entry-point do worker [09:11] Larissa
```

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Disparo: `OrderService.changeStatus` dentro do `$transaction` existente, após inserir em `order_status_history` e antes do commit ([09:40] Bruno).

1. `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` — função pura que recebe o `Prisma.TransactionClient` (tipo `TxClient` já usado em `order.service.ts`); não injeta repository ([09:41] Bruno/Diego).
2. `publishWebhookEvent` busca os `webhook_endpoints` ativos do `order.customerId` cujo `subscribedStatuses` contém `to`. **Filtro na inserção**: sem webhook inscrito naquele status → não insere nada, economiza linha ([09:34] Bruno).
3. Para cada endpoint elegível, renderiza o **snapshot** do payload (seção 6.2) e insere uma linha `PENDING` na `webhook_outbox` via `tx` ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)).
4. Se a inserção falhar, a exceção propaga e o `$transaction` inteiro sofre rollback — status não muda sem evento ([09:40] Bruno).
5. Guarda de tamanho: se o payload serializado exceder **64KB**, lança `WebhookPayloadTooLargeError` (`WEBHOOK_PAYLOAD_TOO_LARGE`) e a transação falha ([09:23] Sofia levanta o limite; [09:24] Diego propõe 64KB; [09:24] Larissa: "erro caso ultrapasse").

### 5.2 Processamento pelo worker

Loop do `webhook.processor.ts`, executado por `src/worker.ts` (processo separado, `PrismaClient` próprio — [09:30] Bruno):

1. A cada **2 segundos** ([09:09] Diego): `SELECT` dos eventos `PENDING` (ou `FAILED` com `nextRetryAt <= now()`) mais antigos por `createdAt`, em **batch pequeno** ([09:08] Diego).
2. Marca o batch como `PROCESSING`.
3. Para cada evento: monta headers (seção 6.3), assina o corpo com HMAC-SHA256 usando a secret **atual** do endpoint, e faz `POST` na `url` com **timeout de 10s** ([09:42] Diego).
4. Registra a tentativa em `webhook_deliveries` (sucesso ou falha, com `httpStatus`, `responseBody` e `durationMs`).
5. Resposta 2xx → evento vira `DELIVERED`.
6. Não-2xx, timeout ou erro de rede → fluxo de retry (5.3).
7. Single-worker: processamento em ordem de `createdAt` dá ordering implícita por `order_id`; não há garantia de ordering global (limitação documentada — [09:13] Larissa).

### 5.3 Retry

1. Falha → `attempts += 1`, `lastError` preenchido, status `FAILED`.
2. `nextRetryAt = now() + backoff[attempts]`, com `backoff = [1min, 5min, 30min, 2h, 12h]` ([09:17] Diego) — janela total de ~15h entre a primeira falha e a última tentativa.
3. O worker retoma o evento quando `nextRetryAt` vence (mesmo ciclo de polling).
4. Reenvios usam o **mesmo payload e o mesmo `X-Event-Id`** (snapshot — [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)); o cliente dedupica ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).

### 5.4 DLQ e replay

1. Falha na **5ª tentativa** → falha permanente: insere em `webhook_dead_letter` (payload, `failureReason`, `failedAt` — [09:18] Diego) e remove a linha da outbox (leitura da outbox principal fica limpa — [09:18] Diego). Feito em transação para não perder o evento entre as tabelas.
2. Replay manual: `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN` — [09:36] Sofia) recoloca o evento na outbox como `PENDING` com `attempts = 0` ([09:18] Diego), marca `replayedAt` na DLQ e **loga quem executou** para auditoria ([09:36] Sofia).

### 5.5 Rotação de secret

1. Cliente chama `POST /webhooks/:id/rotate-secret` ([09:21] Sofia: "Endpoint pro cliente conseguir pedir nova secret pela API").
2. Serviço gera secret nova, move a atual para `previousSecret` com `previousSecretExpiresAt = now() + 24h`, e devolve a nova na resposta.
3. Durante o grace period as duas secrets são válidas em paralelo; recomendação ao cliente: verificar `X-Signature` contra ambas. Passadas 24h, `previousSecret` é ignorada ("a antiga morre" — [09:21] Sofia).

> A assinatura de envio usa sempre a secret **atual**; o grace period existe para o lado verificador do cliente durante a migração.

## 6. Contratos públicos

Base path: `/api/v1` (padrão de `src/app.ts`). Todos os endpoints exigem `authenticate` (JWT Bearer); o de replay exige também `requireRole('ADMIN')`. O `customer_id` vai no body/path — o JWT é do usuário operador, não do cliente ([09:32] Larissa). Erros seguem o envelope do `error.middleware.ts`: `{ "error": { "code", "message", "details?" } }`.

### 6.1 Endpoints

#### POST /api/v1/webhooks — cadastrar webhook ([09:31] Marcos)

Request:

```json
{
  "customerId": "9f1b3c9a-0b6e-4c1a-9f0e-2a7d8b3c4d5e",
  "url": "https://api.atlascomercial.com.br/oms/webhooks",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (a `secret` é gerada pela plataforma e **devolvida apenas na criação** — [09:31] Marcos):

```json
{
  "id": "7c0d2e4f-6a8b-4c2d-9e1f-3b5a7c9d0e2f",
  "customerId": "9f1b3c9a-0b6e-4c1a-9f0e-2a7d8b3c4d5e",
  "url": "https://api.atlascomercial.com.br/oms/webhooks",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d",
  "active": true,
  "createdAt": "2026-07-14T12:00:00.000Z"
}
```

Status codes: `201` criado · `400 WEBHOOK_INVALID_URL` (url `http`) · `400 VALIDATION_ERROR` (Zod) · `401 UNAUTHORIZED` · `404 NOT_FOUND` (customer inexistente).

#### GET /api/v1/webhooks?customerId=... — listar webhooks de um customer ([09:33] Bruno)

Response `200 OK` (sem a secret; paginação no padrão de `src/shared/http/response.ts`):

```json
{
  "data": [
    {
      "id": "7c0d2e4f-6a8b-4c2d-9e1f-3b5a7c9d0e2f",
      "customerId": "9f1b3c9a-0b6e-4c1a-9f0e-2a7d8b3c4d5e",
      "url": "https://api.atlascomercial.com.br/oms/webhooks",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-14T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status codes: `200` · `401 UNAUTHORIZED`.

#### PATCH /api/v1/webhooks/:id — editar webhook ([09:33] Bruno)

Request (campos opcionais):

```json
{
  "url": "https://api.atlascomercial.com.br/oms/webhooks/v2",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200 OK`: objeto atualizado (sem secret). Status codes: `200` · `400 WEBHOOK_INVALID_URL` · `404 WEBHOOK_NOT_FOUND` · `401 UNAUTHORIZED`.

#### DELETE /api/v1/webhooks/:id — remover webhook ([09:33] Bruno)

Response `204 No Content`. Status codes: `204` · `404 WEBHOOK_NOT_FOUND` · `401 UNAUTHORIZED`.

#### GET /api/v1/webhooks/:id/deliveries — histórico de entregas ([09:34] Marcos)

"Os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" ([09:34] Marcos). Response `200 OK`:

```json
{
  "data": [
    {
      "id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
      "eventId": "e2a4c6d8-1b3d-5f7a-9c1e-2d4f6a8b0c1d",
      "attempt": 1,
      "success": false,
      "httpStatus": 503,
      "requestPayload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "responseBody": "Service Unavailable",
      "durationMs": 812,
      "attemptedAt": "2026-07-14T12:03:02.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Status codes: `200` · `404 WEBHOOK_NOT_FOUND` · `401 UNAUTHORIZED`.

#### POST /api/v1/webhooks/:id/rotate-secret — rotacionar secret ([09:21] Sofia)

Request: corpo vazio. Response `200 OK`:

```json
{
  "id": "7c0d2e4f-6a8b-4c2d-9e1f-3b5a7c9d0e2f",
  "secret": "whsec_1f2e3d4c5b6a7980aabbccddeeff0011",
  "previousSecretExpiresAt": "2026-07-15T12:00:00.000Z"
}
```

Status codes: `200` · `404 WEBHOOK_NOT_FOUND` · `401 UNAUTHORIZED`.

#### POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay de DLQ ([09:18] Diego, [09:36] Sofia)

Exige role `ADMIN` (`requireRole('ADMIN')`). Request: corpo vazio. Response `202 Accepted`:

```json
{
  "deadLetterId": "5e6f7a8b-9c0d-1e2f-3a4b-5c6d7e8f9a0b",
  "eventId": "e2a4c6d8-1b3d-5f7a-9c1e-2d4f6a8b0c1d",
  "status": "PENDING",
  "replayedBy": "admin@empresa.com",
  "replayedAt": "2026-07-14T15:30:00.000Z"
}
```

Status codes: `202` · `403 FORBIDDEN` (role insuficiente) · `404 WEBHOOK_DEAD_LETTER_NOT_FOUND` · `401 UNAUTHORIZED`. A execução é registrada em log de auditoria com o usuário do JWT ([09:36] Sofia).

### 6.2 Payload do evento (nós → cliente)

Formato definido em [09:43] Diego; snapshot renderizado na inserção ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)). **Não inclui items** — cliente que quiser detalhes consulta `GET /orders/:id` ([09:43] Diego). Tamanho máximo 64KB ([09:24]).

```json
{
  "event_id": "e2a4c6d8-1b3d-5f7a-9c1e-2d4f6a8b0c1d",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-14T12:03:00.000Z",
  "order_id": "b1c2d3e4-f5a6-7b8c-9d0e-1f2a3b4c5d6e",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "9f1b3c9a-0b6e-4c1a-9f0e-2a7d8b3c4d5e",
  "total_cents": 159900
}
```

### 6.3 Headers do envio (nós → cliente)

Definidos em [09:44] Diego/Sofia:

| Header | Conteúdo |
|---|---|
| `X-Event-Id` | UUID do evento (gerado na entrada na outbox) — chave de deduplicação ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)) |
| `X-Signature` | HMAC-SHA256 (hex) do corpo do request, com a secret do endpoint ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) |
| `X-Timestamp` | Timestamp do envio — permite ao cliente detectar replay attack |
| `X-Webhook-Id` | Id do endpoint de webhook — cliente com vários cadastros identifica qual recebeu ([09:44] Sofia) |
| `Content-Type` | `application/json` |

Semântica de resposta esperada do cliente: qualquer `2xx` = entregue; o corpo da resposta é armazenado em `webhook_deliveries` (truncado) para debug.

## 7. Matriz de erros

Códigos com prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa), implementados como subclasses de `AppError` (`src/shared/errors/app-error.ts`), tratados sem mudança pelo `error.middleware.ts` ([09:29] Bruno).

| Código | HTTP | Quando ocorre | Classe proposta |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook id inexistente em GET/PATCH/DELETE/deliveries/rotate | `WebhookNotFoundError` |
| `WEBHOOK_INVALID_URL` | 400 | URL não-`https` no cadastro/edição ([09:23] Sofia) | `WebhookInvalidUrlError` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação de assinatura sem secret válida configurada ([09:28] Bruno) | `WebhookSecretRequiredError` |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | `subscribedStatuses` com valor fora do enum `OrderStatus` | `WebhookInvalidEventFilterError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB na inserção ([09:24] Larissa) | `WebhookPayloadTooLargeError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de DLQ id inexistente | `WebhookDeadLetterNotFoundError` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno ao worker) | Cliente não respondeu em 10s; registrado em `webhook_deliveries.responseBody`/`lastError` e conta como falha para retry ([09:42] Diego) | valor de `failureReason`, não erro HTTP |
| `WEBHOOK_DELIVERY_REJECTED` | — (interno ao worker) | Cliente respondeu não-2xx; idem acima | valor de `failureReason`, não erro HTTP |

Erros de autenticação/autorização e validação genérica reusam os códigos existentes (`UNAUTHORIZED`, `FORBIDDEN`, `VALIDATION_ERROR`, `NOT_FOUND`) de `src/shared/errors/http-errors.ts`.

## 8. Estratégias de resiliência

| Mecanismo | Configuração | Origem |
|---|---|---|
| Timeout HTTP do worker | 10s por chamada; estouro = falha, entra em retry | [09:42] Diego |
| Retry | Backoff exponencial 1m / 5m / 30m / 2h / 12h, máx. 5 tentativas (~15h de janela) | [09:17] Diego |
| Fallback final (DLQ) | Tabela `webhook_dead_letter` com payload, motivo e timestamp; replay manual por admin | [09:18] Diego |
| Atomicidade | Outbox na mesma transação do `changeStatus`; falha na outbox = rollback total | [09:40] Bruno |
| Isolamento de processo | Worker separado da API; crash/restart de um não afeta o outro | [09:11] Diego |
| Idempotência | `X-Event-Id` estável + payload snapshot imutável entre reenvios | [09:25] Diego, [09:52] Larissa |
| Proteção contra eventos anômalos | Limite de 64KB com erro (não trunca) | [09:23]–[09:24] |
| Recuperação de crash do worker | Eventos presos em `PROCESSING` além de um limiar (ex.: 2× timeout) voltam a ser elegíveis no polling seguinte — decorrência necessária do at-least-once ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)) | derivado |

## 9. Observabilidade

Tudo via **Pino** (`src/shared/logger/index.ts`), já presente no projeto inteiro — nada novo de ferramenta ([09:29] Bruno). O logger já redige campos sensíveis (`redactPaths`); adicionar `*.secret` à lista ao implementar.

**Logs estruturados (worker e API):**

- `webhook_event_enqueued` — event_id, webhook_id, order_id, to_status (na inserção da outbox).
- `webhook_delivery_attempt` / `webhook_delivery_succeeded` / `webhook_delivery_failed` — event_id, webhook_id, attempt, http_status, duration_ms.
- `webhook_event_dead_lettered` — event_id, failure_reason.
- `webhook_dlq_replayed` — dead_letter_id, event_id, **user id/email do admin** (auditoria obrigatória — [09:36] Sofia).

**Métricas** (derivadas das tabelas e dos logs; sem stack nova de métricas nesta fase):

- Profundidade da outbox (`COUNT` de `PENDING`/`FAILED`) — indicador de worker parado ou cliente em massa fora do ar.
- Taxa de sucesso/falha de entregas e `durationMs` (percentis) a partir de `webhook_deliveries`.
- Latência ponta a ponta: `webhook_deliveries.attemptedAt - webhook_outbox.createdAt` — valida o objetivo de <10s.
- Tamanho da DLQ e idade do item mais antigo.

**Tracing / correlação:** o `event_id` é o identificador de correlação ponta a ponta — aparece no log de inserção (processo da API), nos logs do worker, em `webhook_deliveries`, na DLQ e no header `X-Event-Id` recebido pelo cliente. Na API, os logs de request já carregam o `requestId` do `request-logger.middleware.ts`; o log de `webhook_event_enqueued` amarra `requestId` → `event_id`, permitindo rastrear do `PATCH /orders/:id/status` original até a entrega no cliente.

## 10. Integração com o sistema existente

Seção obrigatória: como o módulo de webhooks se acopla ao código real do repositório.

### 10.1 `src/modules/orders/order.service.ts` — extensão do `changeStatus`

Única alteração em código existente ([09:40] Bruno: "a alteração crítica é dentro do service de orders, no método changeStatus"). Hoje o método executa, dentro de `this.prisma.$transaction`: validação da transição (`canTransition`), débito/reposição de estoque, `tx.order.update` e `tx.orderStatusHistory.create`. A extensão adiciona **uma chamada** antes do fim da transação:

```ts
// dentro do $transaction de changeStatus, após tx.orderStatusHistory.create(...)
await publishWebhookEvent(tx, order, from, to);
```

> `order` é a entidade carregada no início da transação (`tx.order.findUnique`); os campos usados no snapshot (`id`, `orderNumber`, `customerId`, `totalCents`) não mudam na transição — apenas `status`, que o payload representa explicitamente via `from`/`to`.

`publishWebhookEvent(tx, order, fromStatus, toStatus)` é função pura exportada por `src/modules/webhooks/webhook.publisher.ts`, recebendo o `Prisma.TransactionClient` (o alias `TxClient` já existe em `order.service.ts:24`) — sem injetar repository no `OrderService` ([09:41] Bruno/Diego). Exceção lançada dentro dela propaga e aborta a transação inteira ([09:40] Bruno).

### 10.2 `src/modules/orders/order.status.ts` — fonte dos tipos de evento

O enum `OrderStatus` (Prisma) e a máquina de transições em `order.status.ts` definem o universo de eventos possíveis: cada transição válida (`PENDING→PAID`, `PAID→PROCESSING`, `PROCESSING→SHIPPED`, `SHIPPED→DELIVERED`, `*→CANCELLED`) pode gerar um `order.status_changed`. O schema Zod de `subscribedStatuses` valida contra esse mesmo enum — status inexistente é rejeitado com `WEBHOOK_INVALID_EVENT_FILTER`.

### 10.3 `src/shared/errors/` — reuso das classes de erro

Novas classes em `src/shared/errors/http-errors.ts` (ou arquivo `webhook-errors.ts` no mesmo diretório) seguem o padrão de `InvalidStatusTransitionError`/`InsufficientStockError`: subclasse de `AppError` com `errorCode` fixo prefixado `WEBHOOK_` ([09:28] Bruno). Como o `errorMiddleware` (`src/middlewares/error.middleware.ts`) já trata qualquer `AppError`, Zod e Prisma, **nenhuma mudança de middleware é necessária** ([09:29] Bruno: "Vai pegar nossos erros sem precisar mudar nada").

### 10.4 `src/middlewares/auth.middleware.ts` — autenticação e autorização

As rotas do módulo usam `authenticate` como todos os módulos (ver `src/modules/orders/order.routes.ts`, que aplica `router.use(authenticate)`). O endpoint de replay adiciona `requireRole('ADMIN')`, reaproveitando a função existente ([09:36] Larissa: "a gente reaproveita o requireRole que já existe"). O `req.user` populado pelo middleware fornece o id/email do admin para o log de auditoria do replay.

### 10.5 `src/server.ts` + `package.json` — novo entry-point do worker

`src/worker.ts` segue o esqueleto de `src/server.ts`: bootstrap com `logger`, `PrismaClient` próprio via `createPrismaClient()` de `src/config/database.ts` (instância nova — client é por processo, [09:30] Bruno), e graceful shutdown em `SIGINT`/`SIGTERM` (parar o loop, aguardar o batch corrente, `$disconnect`). Novo script no `package.json`: `"worker": "tsx watch --env-file=.env src/worker.ts"` (dev) e `node dist/worker.js` (prod), no padrão dos scripts existentes ([09:11] Larissa: "criar um src/worker.ts e um script npm run worker").

### 10.6 `src/routes/index.ts` e `src/app.ts` — registro do módulo

`buildWebhookRouter` entra no `buildApiRouter` (`src/routes/index.ts`) como os demais módulos (`router.use('/webhooks', ...)` e `router.use('/admin/webhooks', ...)`); controller/service/repository são instanciados em `buildControllers` (`src/app.ts`), seguindo o padrão de construção manual de dependências já usado para orders/customers/products.

### 10.7 `prisma/schema.prisma` — convenções de modelagem

As 4 tabelas novas seguem as convenções do schema existente: ids `@default(uuid()) @db.Char(36)` ([09:51] Larissa: "UUID, segue o padrão do resto do projeto"), `@@map` para snake_case, `@@index` explícitos, relações nomeadas. Migração via `npm run db:migrate` (script existente).

## 11. Dependências e compatibilidade

- **Sem dependência nova de runtime**: HMAC-SHA256 via `node:crypto` (`createHmac`); HTTP do worker via `fetch` nativo (Node ≥ 20, já exigido em `package.json` → `engines.node >= 20`); UUID via `uuid` (já dependência).
- **Banco**: MySQL existente (`docker-compose.yml`), mesma `DATABASE_URL` para API e worker; migração Prisma aditiva (só tabelas novas + nada alterado nas existentes) — compatível com rollback de deploy da API.
- **Compatibilidade da API**: nenhum endpoint existente muda; `PATCH /orders/:id/status` mantém contrato e passa a ter o efeito colateral interno da outbox.
- **Deploy**: worker é artefato do mesmo build (`tsc -p tsconfig.build.json`), processo separado no mesmo host/orquestrador.
- **Dependência de processo**: revisão de segurança da Sofia (≥2 dias úteis) antes do deploy, focada em HMAC e geração de secret ([09:46] Sofia).

## 12. Critérios de aceite técnicos

1. Mudança de status commitada com webhook inscrito ⇒ linha `PENDING` na `webhook_outbox` na mesma transação; rollback da transação ⇒ nenhuma linha ([09:40]–[09:41]).
2. Mudança para status **não** inscrito por nenhum webhook do customer ⇒ nenhuma linha na outbox ([09:34]).
3. Evento pendente com endpoint saudável ⇒ entregue em < 10s do commit (polling 2s + envio) ([09:02], [09:09]).
4. Entrega carrega `X-Event-Id`, `X-Signature` (HMAC-SHA256 verificável com a secret), `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:44]).
5. Endpoint retornando 5xx ⇒ tentativas em ~1m, 5m, 30m, 2h e 12h após a primeira falha; após a 5ª falha, linha em `webhook_dead_letter` com payload e motivo, e evento fora da outbox ([09:17]–[09:18]).
6. Reenvio de um mesmo evento ⇒ mesmo `X-Event-Id` e mesmo corpo byte a byte ([09:25], [09:52]).
7. Cadastro com URL `http://` ⇒ `400 WEBHOOK_INVALID_URL`; payload > 64KB ⇒ erro `WEBHOOK_PAYLOAD_TOO_LARGE`, sem envio truncado ([09:23]–[09:24]).
8. Rotação de secret ⇒ nova secret na resposta; assinatura da antiga aceita pelo verificador do cliente por até 24h; após 24h, somente a nova ([09:21]).
9. Replay de DLQ sem role ADMIN ⇒ `403 FORBIDDEN`; com ADMIN ⇒ evento de volta na outbox como `PENDING` e log de auditoria com o usuário ([09:36]).
10. `GET /webhooks/:id/deliveries` retorna as entregas com sucesso/falha, payload, response e `durationMs` ([09:34]).
11. Kill do worker durante processamento ⇒ nenhum evento perdido; ao reiniciar, eventos são retomados (duplicata permitida — at-least-once).
12. Suite existente (`tests/orders.test.ts`, `tests/auth.test.ts`) permanece verde — comportamento atual do `changeStatus` preservado.

## 13. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Worker parado sem ninguém perceber → eventos acumulam e latência estoura o SLO de 10s | Monitorar profundidade da outbox e idade do evento pendente mais antigo (seção 9); processo separado facilita restart isolado ([09:11]) |
| Vazamento de secret de cliente (caso real já ocorrido — [09:22] Diego) | Secret por endpoint (raio de dano contido), rotação com grace de 24h, redação de secrets nos logs (Pino `redact`) ([ADR-004]) |
| Cliente sem deduplicação processa eventos duplicados | Documentação destacada no portal de desenvolvedor sobre at-least-once + `X-Event-Id` ([09:26] Marcos) |
| Crescimento da `webhook_outbox`/`webhook_deliveries` degrada o polling | Índices em `status` e `createdAt`, batch pequeno ([09:08]); arquivamento fica como questão em aberto ([RFC — Questões em aberto](RFC.md#questões-em-aberto)) |
| Bug na inserção da outbox derruba mudança de status (acoplamento transacional) | Cobertura de testes da transação estendida (critérios 1 e 12 da seção 12); código do publisher mínimo e puro ([09:41]) |
| Janela de retry de ~15h atrasa a detecção de integração quebrada do cliente | Histórico de deliveries consultável via API ([09:34]); alerta por e-mail explicitamente adiado para próxima fase ([09:37]) |

## 14. Plano de entrega (referência)

Estimativa da reunião: **3 sprints**, com a revisão de segurança da Sofia incluída no fim ([09:46]–[09:47] Larissa):

1. Sprint 1 — modelagem de outbox e DLQ.
2. Sprint 2 — worker e retry.
3. Sprint 3 — CRUD de configuração e deliveries (½), integração no `order.service` + testes ponta a ponta (½), HMAC/schemas/validações, revisão de segurança (≥2 dias úteis — [09:46] Sofia).
