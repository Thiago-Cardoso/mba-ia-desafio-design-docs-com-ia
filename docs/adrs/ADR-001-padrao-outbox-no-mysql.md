# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito — decidido em reunião técnica ([09:08] Larissa: "Tá decidido então: outbox em MySQL").

## Contexto

O OMS precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) quando o status de um pedido muda ([09:00] Marcos). A mudança de status hoje acontece dentro de uma transação Prisma pesada em `src/modules/orders/order.service.ts` (método `changeStatus`): atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos ([09:04] Bruno).

O problema central é garantir que **toda** mudança de status commitada gere exatamente um evento de notificação registrado — sem travar a transação principal com chamadas HTTP e sem criar inconsistência entre o estado do pedido e os eventos emitidos ([09:40] Bruno: "Não pode ter caso de status mudar e evento não sair").

Restrição relevante: o time é pequeno e não quer operar infraestrutura adicional ([09:07] Diego).

## Decisão

Adotar o **padrão Transactional Outbox usando o MySQL já existente**:

- Uma tabela `webhook_outbox` recebe uma linha por evento, **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history` ([09:06] Diego).
- Se a transação principal commita, o evento está registrado; se sofre rollback, o evento desaparece junto — não há inconsistência possível ([09:06] Diego).
- Um worker separado lê a tabela e dispara as chamadas HTTP (ver [ADR-002](ADR-002-worker-separado-com-polling.md)).
- A tabela terá índice no campo de status do evento (pendente, processando, falhou, entregue) e em `created_at`; o worker lê apenas pendentes em batch pequeno ([09:08] Diego).
- Arquivamento de linhas entregues (ex.: após 30 dias) fica **fora do escopo** desta feature ([09:08] Diego).

## Alternativas Consideradas

### 1. Disparo síncrono dentro do `OrderService.changeStatus`

Descartada ([09:04]–[09:06]). A transação já é pesada; um HTTP call no meio dela faria qualquer cliente lento travar mudanças de status de outros pedidos ([09:04] Bruno). Além disso, se o cliente estiver fora do ar não há resposta razoável — dar rollback na mudança de status por falha de notificação é inaceitável ([09:04] Bruno). Diego foi categórico: "Síncrono está fora de questão" ([09:06]).

### 2. Fila externa (Redis Streams ou similar)

Descartada ([09:07]). Resolveria o desacoplamento, mas exigiria subir e operar infraestrutura nova. Para um time pequeno, "subir Redis Cluster pra isso é overengineering" ([09:07] Diego). O trade-off aceito: menos capacidade de throughput/fan-out que uma fila dedicada, em troca de zero infra nova.

## Consequências

**Positivas**

- Atomicidade garantida entre mudança de status e registro do evento — a garantia central da feature ([09:41] Diego: "Se ficar fora da transação, perde a garantia toda").
- Nenhuma infraestrutura nova: reusa o MySQL e o Prisma já configurados em `prisma/schema.prisma` e `src/config/database.ts`.
- Evidência persistida e consultável (a outbox é uma tabela SQL comum), o que facilita debug e o histórico de entregas.

**Negativas / Trade-offs**

- A latência de entrega depende do ciclo de polling do worker (mínimo ~2s no pior caso, ver ADR-002) — aceito porque o requisito dos clientes é "abaixo de 10 segundos" ([09:02] Marcos).
- A tabela cresce indefinidamente até que exista rotina de arquivamento (adiada; mitigada pelos índices em status e `created_at` e leitura em batch pequeno — [09:08] Diego).
- Acopla o throughput de eventos ao banco transacional principal (linhas extras escritas em cada mudança de status).

## Referências

- [ADR-002 — Worker separado com polling](ADR-002-worker-separado-com-polling.md)
- Transcrição: [09:03]–[09:08]
- Código: `src/modules/orders/order.service.ts` (método `changeStatus`), `prisma/schema.prisma`
