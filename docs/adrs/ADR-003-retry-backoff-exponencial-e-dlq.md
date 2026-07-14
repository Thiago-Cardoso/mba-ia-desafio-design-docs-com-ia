# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Status

Aceito — decidido em reunião técnica ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").

## Contexto

Webhooks são entregues a endpoints fora da nossa infraestrutura. Clientes ficam offline — já houve caso real de cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). É preciso definir o que acontece quando a entrega falha: quantas vezes retentar, com que espaçamento, e o destino dos eventos que esgotam as tentativas ([09:14] Larissa: "Se o cliente tá offline, o que a gente faz?").

## Decisão

1. **Retry com backoff exponencial**: após cada falha, o evento é reagendado com intervalos crescentes ([09:15] Diego).
2. **5 tentativas no total**, com progressão **1 min / 5 min / 30 min / 2 h / 12 h** — cerca de 15 horas entre a primeira falha e a última tentativa ([09:17] Diego).
3. Esgotadas as tentativas, o evento é considerado **falha permanente** e movido para uma **DLQ persistida em tabela separada** (`webhook_dead_letter`), guardando payload, motivo da falha e timestamp ([09:18] Diego).
4. **Reprocessamento manual** via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role `ADMIN` e registra em log quem executou o replay, para auditoria ([09:36] Sofia) — reusa o `requireRole` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
5. Timeout de 10 segundos por chamada HTTP: cliente que não responde em 10s conta como falha e entra no ciclo de retry ([09:42] Diego).

## Alternativas Consideradas

### 1. Apenas 3 tentativas (mais agressivo)

Proposta pelo Bruno ([09:16]) e descartada. Com 3 tentativas o ciclo fecharia em ~30 minutos; um cliente com indisponibilidade de duas horas (caso real já ocorrido) perderia eventos definitivamente ([09:16] Diego: "3 é pouco").

### 2. Retry indefinido com backoff

Mencionada e descartada ([09:15]). Evita perda de eventos, mas cria eventos "pendurados pra sempre" quando o cliente simplesmente sumiu, poluindo a outbox e mascarando problemas ([09:15]–[09:16] Diego). Cinco tentativas cobrem janela de 12–24 horas, suficiente ([09:17] Marcos: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele").

### 3. Marcar como "failed" na própria outbox, sem tabela separada

Discutida e descartada ([09:17]–[09:18]). Tabela `webhook_dead_letter` separada mantém a leitura da outbox principal limpa e serve de evidência para debug e reprocessamento ([09:18] Diego).

## Consequências

**Positivas**

- Tolerância a indisponibilidades reais de clientes (janela de ~15h cobre manutenções longas).
- Nenhum evento é perdido silenciosamente: falha permanente vira linha na DLQ, auditável e reprocessável.
- Outbox principal permanece enxuta (falhas permanentes saem dela).

**Negativas / Trade-offs**

- Um evento pode levar ~15h até ser declarado falha permanente — durante esse período o cliente não sabe que está perdendo eventos (alerta por e-mail foi explicitamente adiado para fase futura — [09:37] Larissa).
- Reprocessamento é manual e depende de um admin — sem automação nesta fase.
- Mais uma tabela para modelar e manter.

## Referências

- [ADR-001 — Padrão Outbox no MySQL](ADR-001-padrao-outbox-no-mysql.md)
- Transcrição: [09:14]–[09:18], [09:35]–[09:36], [09:42]
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`)
