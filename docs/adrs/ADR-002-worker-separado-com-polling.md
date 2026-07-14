# ADR-002 — Worker em processo separado consumindo a outbox por polling

## Status

Aceito — decidido em reunião técnica ([09:10] Larissa: "Vamos registrar isso como uma decisão. Worker em polling, 2s").

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), é preciso definir **como** os eventos pendentes serão consumidos e entregues: qual mecanismo de leitura da tabela e onde esse consumidor roda.

Dois requisitos moldam a decisão:

1. Latência percebida como "tempo real" pelos clientes: qualquer coisa abaixo de 10 segundos atende ([09:02] Marcos).
2. O consumidor não pode morrer junto com a API: "se a API reinicia, perde o worker" ([09:11] Diego).

## Decisão

- O worker roda como **processo separado** da API, com entry-point próprio: `src/worker.ts` (nos moldes do `src/server.ts` existente) e script `npm run worker` no `package.json` ([09:11] Larissa).
- Consumo por **polling em loop: a cada 2 segundos**, busca os eventos pendentes mais antigos, processa em batch pequeno e marca como entregue ([09:09] Diego; [09:08] Diego).
- O worker conecta no **mesmo banco** (mesma `DATABASE_URL`), mas com **instância própria de `PrismaClient`**, pois o client é por processo ([09:30] Bruno). A lógica de processamento vive dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
- **Single-worker** nesta fase: um único worker processa em ordem de `created_at`, o que dá ordering implícita por `order_id` ([09:12] Diego). Escalar para múltiplos workers (particionamento por `order_id` ou lock pessimista) é problema do futuro ([09:13] Diego).

## Alternativas Consideradas

### 1. Trigger no banco para notificar o worker (mais reativo)

Descartada ([09:09]). MySQL não tem listener nativo equivalente ao NOTIFY/LISTEN do Postgres; triggers só executam SQL e não notificam processo externo. Avisar o worker exigiria improvisos (escrever em arquivo, bater em endpoint) — "fica esquisito" ([09:09] Diego). Trade-off aceito: latência mínima de até 2s do polling, que atende com folga o requisito de <10s.

### 2. Worker dentro do mesmo processo da API

Descartada ([09:11]). Um `setInterval` na própria instância da API seria mais simples, mas o ciclo de vida do consumidor ficaria acoplado ao da API: restart ou crash da API interrompe a entrega de webhooks ([09:11] Diego: "não pode ser o mesmo processo").

## Consequências

**Positivas**

- Isolamento de falhas: API e worker reiniciam de forma independente.
- Simplicidade operacional: nenhum broker, apenas mais um processo Node com a mesma stack (mesmo banco, mesmo padrão de código — [09:11] Diego).
- Ordering por `order_id` garantida enquanto for single-worker ([09:12] Diego).

**Negativas / Trade-offs**

- Latência mínima de ~2s no pior caso, aceita explicitamente ([09:10] Larissa: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos").
- Polling gera consultas constantes ao banco mesmo sem eventos pendentes (mitigado por índice em status/`created_at` e batch pequeno — [09:08] Diego).
- **Limitação conhecida e documentada**: não há garantia de ordering global; a garantia é por `order_id` e apenas enquanto houver um único worker ([09:13] Larissa). Os clientes nunca pediram ordering global ([09:14] Marcos).
- Um novo processo para operar (deploy, monitoração e restart próprios).

## Referências

- [ADR-001 — Padrão Outbox no MySQL](ADR-001-padrao-outbox-no-mysql.md)
- Transcrição: [09:08]–[09:14], [09:28]–[09:30]
- Código: `src/server.ts` (modelo de entry-point), `src/config/database.ts` (`createPrismaClient`), `package.json` (scripts)
