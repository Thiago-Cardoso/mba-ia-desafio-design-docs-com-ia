# ADR-005 — Garantia de entrega at-least-once com deduplicação por X-Event-Id

## Status

Aceito — decidido em reunião técnica ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão").

## Contexto

Com outbox + retry, o sistema pode entregar o mesmo evento mais de uma vez — por exemplo, quando o cliente recebe e processa a chamada mas a resposta se perde (timeout), o evento entra em retry e é reenviado. É preciso definir a semântica de entrega prometida ao cliente e como ele distingue duplicatas ([09:24] Diego: "Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado").

## Decisão

1. A garantia oferecida é **at-least-once**: todo evento commitado será entregue pelo menos uma vez (dentro do limite de tentativas do [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)); duplicatas são possíveis ([09:24] Diego).
2. Cada evento carrega um **`event_id` (UUID)**, gerado no momento em que entra na outbox e único por evento, enviado no header **`X-Event-Id`** ([09:25] Diego). O UUID segue o padrão de identificadores do restante do projeto ([09:51] Larissa: "UUID, segue o padrão do resto do projeto").
3. A **deduplicação é responsabilidade do cliente**, usando o `event_id` ([09:25] Diego). Essa responsabilidade será documentada com destaque no portal de desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

### 1. Garantia exactly-once

Descartada ([09:25]). Exigiria coordenação entre os dois lados (produtor e consumidor) e tornaria o protocolo muito mais complexo. At-least-once com `event_id` "resolve 99% dos casos" e é o padrão de mercado — Stripe e GitHub fazem assim ([09:25] Diego).

### 2. At-most-once (enviar uma única vez, sem retry)

Alternativa implícita rejeitada pela própria decisão de retry ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)): sem reenvio, qualquer indisponibilidade do cliente significaria perda definitiva de eventos, incompatível com o propósito da feature (substituir o polling do cliente com confiabilidade).

## Consequências

**Positivas**

- Protocolo simples dos dois lados: nós reenviamos sem medo; o cliente dedupica por chave única.
- Alinhado ao padrão de mercado (Stripe, GitHub), o que reduz atrito de integração para os clientes B2B.
- O `event_id` também serve de chave de correlação para suporte e debug (aparece na outbox, nas entregas e na DLQ).

**Negativas / Trade-offs**

- Joga responsabilidade para o cliente ([09:25] Sofia) — cliente que ignorar a deduplicação processará eventos duplicados. Mitigação: documentação destacada no portal ([09:26] Marcos).
- Duplicatas consomem chamadas e processamento do cliente em cenários de timeout com sucesso silencioso.

## Referências

- [ADR-003 — Retry com backoff exponencial e DLQ](ADR-003-retry-backoff-exponencial-e-dlq.md)
- Transcrição: [09:24]–[09:26], [09:51]
- Código: `prisma/schema.prisma` (padrão de ids `@default(uuid())`)
