# Tracker de Rastreabilidade

Cada item registrado nos documentos do pacote mapeado à sua origem: a transcrição da reunião ([TRANSCRICAO.md](../TRANSCRICAO.md)) ou o código-fonte do repositório. Itens sem origem identificável foram removidos dos documentos durante a produção.

**Convenção de IDs:** `PRD-FR-*` (req. funcional), `PRD-NFR-*` (req. não funcional), `PRD-OBJ-*` (objetivo), `PRD-OOS-*` (fora de escopo), `PRD-RISK-*` (risco), `RFC-CTX-*` (contexto), `RFC-PROP-*` (proposta), `RFC-ALT-*` (alternativa), `RFC-QA-*` (questão em aberto), `ADR-NNN` (decisão), `FDD-FLUXO-*`, `FDD-CONTRATO-*`, `FDD-ERRO-*`, `FDD-INT-*` (integração), `FDD-NFR-*`, `FDD-OBS-*`, `FDD-TEST-*`.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo por notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling no GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Motivação | Atlas ameaça migrar para concorrente se não entregue até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | Escopo apenas outbound: webhooks saem de nós para o cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência de notificação < 10 segundos ("tempo real" para os clientes) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Piso de latência de 2s (pior caso do polling) aceito | TRANSCRICAO | [09:10] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Entrega em produção até fim de novembro; estimativa 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST: url, secret gerada e devolvida na criação, lista de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem dos webhooks de um customer via GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção na outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Notificação HTTP POST assinada a cada mudança de status inscrita | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico de entregas: últimos ~100 envios com sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry automático com backoff e DLQ para falha permanente | TRANSCRICAO | [09:15] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via endpoint admin com role ADMIN e auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Recusa de URL não-https com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | CRUD autenticado com JWT normal; customer_id no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10s; ciclo de polling de 2s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Mudança de status não pode ser bloqueada/revertida por cliente indisponível | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Consistência transacional: evento registrado sse status commitou | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com dedup pelo cliente via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256, secret única por endpoint, rotação com grace 24h | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório (somente https) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB com erro (nunca truncar) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP ao cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordering apenas por order_id e enquanto single-worker (limitação documentada) | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Índices em status e created_at na outbox; leitura em batch pequeno | TRANSCRICAO | [09:08] Diego |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Nenhuma infra nova; reuso da stack MySQL/Prisma e padrões do projeto | TRANSCRICAO | [09:07] Diego |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | Notificação por e-mail em falha de webhook — próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Dashboard visual — projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Arquivamento das linhas entregues da outbox (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Múltiplos workers em paralelo — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de Escopo | Webhooks inbound (cliente → nós) | TRANSCRICAO | [09:02] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Perda do prazo de novembro → churn da Atlas | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret (caso real de cliente que vazou em log) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente sem deduplicação processa duplicatas (at-least-once) | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-04 | docs/PRD.md | Risco | Worker parado acumula eventos e estoura o SLO de latência (mitigado por processo separado e monitoração da outbox) | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Acoplamento transacional: falha na escrita da outbox aborta a mudança de status (mitigado por publisher mínimo/função pura) | TRANSCRICAO | [09:41] Bruno |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia: ≥2 dias úteis antes do deploy (HMAC e secrets) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação da integração (dedup, assinatura) no portal de desenvolvedor pelo PM | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Sessão de revisão do design doc com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Testes ponta a ponta previstos na estimativa (sprint de integração) | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-02 | docs/PRD.md | Estratégia de Teste | Suite e helpers existentes (Vitest/supertest) como base de regressão e integração | CODIGO | tests/orders.test.ts |
| RFC-CTX-01 | docs/RFC.md | Contexto | Ciclo de vida do pedido controlado por máquina de estados; nada sai do sistema hoje | CODIGO | src/modules/orders/order.status.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox: inserir evento na mesma transação SQL que atualiza orders e history | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Função publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx client | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Decisão | Worker: entry-point src/worker.ts + script npm run worker, nos moldes do server.ts | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão | Worker com instância própria de PrismaClient (client é por processo) | TRANSCRICAO | [09:30] Bruno |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono no changeStatus — travaria a transação; rollback por cliente fora do ar inaceitável | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams — infra nova; overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger MySQL para acordar worker — banco não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Exactly-once — coordenação dos dois lados, complexidade; padrão de mercado é at-least-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | 3 tentativas de retry — insuficiente para indisponibilidade real de ~2h já observada | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Retry indefinido — evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída: observar em produção e implementar se virar problema | TRANSCRICAO | [09:39] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers: particionar por order_id ou lock pessimista, no futuro | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Arquivamento da outbox após ~30 dias: política fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em Aberto | Endurecimento futuro das permissões do CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Única alteração em código existente: chamada na transação do changeStatus | TRANSCRICAO | [09:40] Bruno |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Padrão Transactional Outbox no MySQL existente | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em processo separado, polling a cada 2s, single-worker | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID) para dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006b | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Referência de Código | Padrão de módulo (controller/service/repository/routes/schemas) que o webhook espelha | CODIGO | src/modules/orders/ |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro do $transaction do changeStatus; falha ⇒ rollback total | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker: poll 2s, eventos pendentes mais antigos, batch pequeno, marca entregue | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry: reagendamento com backoff crescente até 5 tentativas (~15h) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ: tabela separada com payload, motivo da falha e timestamp; replay recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Rotação de secret: antiga válida 24h em paralelo, depois morre | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-01 | docs/FDD.md | Modelo de Dados | Outbox com status pendente/processando/falhou/entregue e índices em status/created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-02 | docs/FDD.md | Modelo de Dados | Configuração de webhook: url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-03 | docs/FDD.md | Modelo de Dados | IDs UUID nas tabelas novas, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-04 | docs/FDD.md | Modelo de Dados | Convenções Prisma reusadas: @default(uuid()) @db.Char(36), @@map snake_case, @@index | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks — cadastro com secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks, PATCH /webhooks/:id, DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries — histórico com sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay — replay manual, role ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret — cliente pede nova secret pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Payload: event_id, event_type, timestamp ISO 8601, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Headers: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | X-Webhook-Id para cliente com vários cadastros identificar o endpoint | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Envelope de erro { error: { code, message, details } } e paginação reusados do projeto | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERRO-01 | docs/FDD.md | Matriz de Erros | Prefixo WEBHOOK_ em todos os códigos; exemplos citados: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Matriz de Erros | WEBHOOK_INVALID_URL para URL http (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Matriz de Erros | WEBHOOK_PAYLOAD_TOO_LARGE para payload > 64KB (erro, não trunca) | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-04 | docs/FDD.md | Matriz de Erros | Timeout de 10s tratado como falha de entrega (retry), não erro HTTP da nossa API | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-05 | docs/FDD.md | Matriz de Erros | Classes de erro como subclasses de AppError com errorCode, no padrão existente | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão do changeStatus: publishWebhookEvent(tx, ...) dentro do $transaction existente | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Enum OrderStatus e máquina de transições como universo dos eventos e validação dos filtros | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Error middleware já trata AppError/Zod/Prisma — nenhuma mudança necessária | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') reusados nas rotas do módulo e no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | src/worker.ts segue o esqueleto de bootstrap/graceful-shutdown do server.ts | CODIGO | src/server.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Registro do módulo em buildApiRouter/buildControllers como os demais | CODIGO | src/routes/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | PrismaClient próprio do worker via createPrismaClient() | CODIGO | src/config/database.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Logger Pino compartilhado, com redact de campos sensíveis a estender para secrets | CODIGO | src/shared/logger/index.ts |
| FDD-NFR-01 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-NFR-02 | docs/FDD.md | Resiliência | Single-worker processa em ordem de created_at; ordering por order_id; sem garantia global | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logging exclusivamente via Pino, já presente no projeto inteiro | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria do replay com o usuário que executou | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação ponta a ponta pelo event_id (API → worker → deliveries → DLQ → cliente) | TRANSCRICAO | [09:25] Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20 (fetch nativo), uuid já dependência; sem lib nova de runtime | CODIGO | package.json |
| FDD-PLAN-01 | docs/FDD.md | Plano | 3 sprints: outbox/DLQ; worker/retry; CRUD+deliveries, integração+e2e, HMAC/validações | TRANSCRICAO | [09:46] Larissa |
