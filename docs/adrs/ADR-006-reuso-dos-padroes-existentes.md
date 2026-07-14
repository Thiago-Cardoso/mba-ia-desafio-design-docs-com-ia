# ADR-006 — Reuso máximo dos padrões e da infraestrutura existentes do projeto

## Status

Aceito — decidido em reunião técnica ([09:30] Larissa: "Decisão: reuso máximo do que já existe").

## Contexto

A codebase do OMS tem um padrão consolidado e uniforme ([09:27] Bruno: "A gente tem um padrão claro na codebase"): cada domínio é um módulo em `src/modules/` com `controller`, `service`, `repository`, `routes` e `schemas` (ver `src/modules/orders/`, `src/modules/customers/`, etc.). Erros, logging, validação e autenticação são transversais e centralizados. Introduzir convenções novas para o módulo de webhooks aumentaria a carga cognitiva do time sem benefício.

## Decisão

O módulo de webhooks **segue integralmente os padrões existentes**, sem introduzir nada novo:

| Aspecto | Padrão reusado | Onde está no código |
|---|---|---|
| Estrutura de módulo | Pasta `src/modules/webhooks/` com controller, service, repository, routes e schemas ([09:27] Bruno) | espelha `src/modules/orders/` |
| Erros | Subclasses de `AppError` com `errorCode`, como `InsufficientStockError` e `InvalidStatusTransitionError`; códigos novos com prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa) | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| Tratamento de erro HTTP | Middleware central que já trata `AppError`, `ZodError` e erros do Prisma — "vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno) | `src/middlewares/error.middleware.ts` |
| Logging | Pino, já presente no projeto inteiro; nada novo ([09:29] Bruno) | `src/shared/logger/index.ts` |
| Validação | Schemas Zod por módulo + middleware `validate` ([09:30] Larissa) | `src/middlewares/validate.middleware.ts` |
| Autenticação/autorização | `authenticate` + `requireRole` já existentes ([09:36] Larissa) | `src/middlewares/auth.middleware.ts` |
| Banco/ORM | MySQL + Prisma, mesma `DATABASE_URL`; worker abre **instância própria** de `PrismaClient` por ser outro processo ([09:30] Bruno) | `src/config/database.ts`, `prisma/schema.prisma` |
| Identificadores | UUID em todas as tabelas novas, "segue o padrão do resto do projeto" ([09:51] Larissa) | `prisma/schema.prisma` (`@default(uuid())`) |

## Alternativas Consideradas

### 1. Introduzir ferramentas novas para o módulo (logger próprio, camada de eventos dedicada)

Rejeitada na discussão ([09:29] Bruno: "Não vamos botar nada novo"). Qualquer ganho pontual de uma ferramenta especializada não compensa a fragmentação de padrões numa codebase pequena e uniforme.

### 2. Worker compartilhando a instância de `PrismaClient` da API

Descartada ([09:29]–[09:30]). `PrismaClient` é por processo; como o worker é um processo Node separado ([ADR-002](ADR-002-worker-separado-com-polling.md)), ele precisa de instância nova — mesmo banco, mesma stack, outro client ([09:30] Bruno).

## Consequências

**Positivas**

- Qualquer pessoa do time navega no módulo novo sem aprender convenção nova.
- O middleware de erro e o logger funcionam para webhooks **sem alteração** ([09:29] Bruno).
- Menos código novo para revisar na janela de segurança da Sofia ([09:46]).

**Negativas / Trade-offs**

- O módulo herda as limitações da stack atual (ex.: sem mecanismo de mensageria, o que já foi endereçado pela escolha da outbox no [ADR-001](ADR-001-padrao-outbox-no-mysql.md)).
- O prefixo `WEBHOOK_` precisa ser aplicado com disciplina para manter a convenção de códigos de erro consistente.

## Referências

- [ADR-001 — Padrão Outbox no MySQL](ADR-001-padrao-outbox-no-mysql.md), [ADR-002 — Worker separado com polling](ADR-002-worker-separado-com-polling.md)
- Transcrição: [09:27]–[09:30], [09:36], [09:51]
- Código: `src/modules/orders/` (módulo de referência), `src/shared/errors/`, `src/shared/logger/index.ts`, `src/middlewares/`
