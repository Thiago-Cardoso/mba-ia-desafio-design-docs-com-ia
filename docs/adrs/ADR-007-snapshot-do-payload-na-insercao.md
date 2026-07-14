# ADR-007 — Payload do evento renderizado como snapshot na inserção da outbox

## Status

Aceito — decidido em reunião técnica ([09:52] Bruno: "Beleza, snapshot. Decidido").

## Contexto

Ao inserir um evento na `webhook_outbox`, existem duas formas de guardar o conteúdo: armazenar o payload JSON **já renderizado**, ou armazenar apenas o `order_id` e montar o payload na hora do envio pelo worker ([09:51] Bruno levantou a dúvida no fim da reunião).

A diferença importa porque há uma janela de tempo entre a inserção do evento e o envio (polling de 2s no melhor caso, até ~15h no pior caso com retries — [ADR-002](ADR-002-worker-separado-com-polling.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)). Nessa janela o pedido pode mudar de novo.

## Decisão

O payload é **renderizado no momento da inserção** na outbox (snapshot): "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" ([09:52] Larissa).

O formato do payload foi definido na mesma reunião ([09:43] Diego): JSON com `event_id`, `event_type` (ex.: `order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`. **Não inclui os items** do pedido, para não inflar o payload; cliente que quiser detalhes consulta `GET /orders/:id` ([09:43] Diego). Limite de tamanho: 64KB, com erro caso ultrapasse ([09:24] Larissa).

## Alternativas Consideradas

### 1. Guardar apenas `order_id` e renderizar o payload no envio

Descartada ([09:51]–[09:52]). Menos bytes na outbox e payload sempre "fresco", mas cria o "caso esquisito": um evento de `PAID → PROCESSING` enviado horas depois (por retry) carregaria o estado **atual** do pedido (ex.: já `SHIPPED`), contradizendo o próprio evento. Retries deixariam de ser idempotentes no conteúdo, minando a deduplicação por `X-Event-Id` ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).

## Consequências

**Positivas**

- Todo envio (e reenvio) de um mesmo `event_id` carrega exatamente os mesmos bytes — consistência com a semântica at-least-once e com a assinatura HMAC (que é sobre o corpo — [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md)).
- O worker não precisa consultar `orders` no envio; lê a linha da outbox e dispara.
- A DLQ guarda o payload exato que falhou, evidência completa para debug ([09:18] Diego).

**Negativas / Trade-offs**

- Payload duplicado em disco (a informação existe em `orders` e na outbox) — mitigado pelo payload enxuto, sem items ([09:43] Diego).
- Mudanças no formato do payload exigem cuidado com eventos antigos ainda pendentes na outbox (formato anterior).

## Referências

- [ADR-004 — HMAC-SHA256 com secret por endpoint](ADR-004-hmac-sha256-secret-por-endpoint.md), [ADR-005 — At-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
- Transcrição: [09:43], [09:51]–[09:52]
- Código: `prisma/schema.prisma` (modelo `Order` — campos `orderNumber`, `totalCents`, `customerId` que compõem o snapshot)
