# Da Reunião ao Documento — Processo de Produção

> Este README documenta o processo de produção do pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos**, gerado a partir de [`TRANSCRICAO.md`](TRANSCRICAO.md) e do código do repositório. O enunciado original do desafio está preservado no histórico do git (commit `e7f6311` e anteriores).

## Sobre o desafio

O ponto de partida era uma situação comum em times reais: uma decisão técnica inteira tomada numa call de ~55 minutos — arquitetura, segurança, requisitos, escopo e prazo de um sistema de webhooks para um OMS em produção — e **nenhum registro além da transcrição literal**. A tarefa foi transformar essa transcrição, cruzada com o código existente (Node.js + TypeScript + Express + Prisma/MySQL), em um pacote completo de design docs: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A restrição central mudou a natureza do trabalho: **nada podia ser inventado**. Cada requisito, decisão ou limite tinha que ser rastreável a um timestamp da reunião ou a um arquivo real do repositório — inclusive (e principalmente) o que foi **descartado ou adiado** na reunião, que precisava ficar fora dos requisitos. A IA foi a ferramenta de produção; o trabalho humano foi direção, prompts dirigidos e revisão crítica.

## Ferramentas de IA utilizadas

- **Claude Code (CLI) com o modelo Claude Fable 5 (Anthropic)** — ferramenta única do processo, usada de ponta a ponta: exploração do código com as ferramentas de leitura/busca do agente, análise da transcrição, geração de cada documento, revisão cruzada entre documentos e verificação final contra os critérios de aceite. Trabalhar dentro do repositório (em vez de colar trechos num chat) fez diferença prática: o agente lê os arquivos reais, então referências como `order.service.ts` ou o alias `TxClient` saem verificadas, não imaginadas.

## Workflow adotado

A ordem de produção foi deliberadamente **inversa à altura dos documentos** — do mais concreto para o mais alto nível:

1. **Exploração** — leitura integral da `TRANSCRICAO.md` e dos arquivos-chave do código (`order.service.ts`, `order.status.ts`, `schema.prisma`, middlewares, errors, logger, `server.ts`, `package.json`). Saída: um mapa mental de decisões com timestamps + inventário de padrões do código.
2. **ADRs primeiro** (`docs/adrs/`) — as 6 decisões principais da reunião + a decisão de snapshot do payload viraram 7 ADRs. As decisões são o esqueleto de todo o resto; escrevê-las primeiro evitou que RFC e FDD "decidissem de novo".
3. **RFC** (`docs/RFC.md`) — proposta consolidada em cima dos ADRs já linkáveis. As alternativas descartadas ([09:04] síncrono, [09:07] Redis, [09:09] trigger, [09:25] exactly-once, [09:15]–[09:16] políticas de retry) e as questões deixadas em aberto (rate limiting, multi-worker, arquivamento, permissões) encontraram aqui seu lugar natural.
4. **FDD** (`docs/FDD.md`) — o "como construir": modelo de dados, fluxos, contratos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção obrigatória de integração com 7 caminhos reais do código.
5. **PRD** (`docs/PRD.md`) — por último entre os grandes documentos: com decisões e design prontos, virou consolidação de problema, escopo, métricas e riscos, sem duplicar o nível técnico.
6. **Tracker** (`docs/TRACKER.md`) — varredura final dos quatro documentos, mapeando ~100 itens a timestamps `[hh:mm] Falante` ou caminhos de arquivo.
7. **Revisão final** — checklist dos critérios de aceite item por item + caça a inconsistências entre documentos (foi onde apareceram os ajustes descritos abaixo).

A interação com a IA seguiu um padrão fixo: prompt dirigido com critérios explícitos → geração → leitura crítica contra transcrição/código → correção pontual → nova verificação.

## Prompts customizados

Dois exemplos representativos dos prompts usados (adaptados dos padrões do curso para o contexto do desafio):

**1. Extração dirigida com classificação de escopo** (usado antes de qualquer documento, para evitar que item descartado virasse requisito):

```text
Leia TRANSCRICAO.md na íntegra. Produza três listas separadas, cada item com
timestamp e falante:

1. DECIDIDO — decisões fechadas na reunião (o que foi decidido, por quem,
   e qual alternativa foi descartada no caminho, se houver).
2. DESCARTADO/ADIADO — tudo que foi explicitamente rejeitado ou empurrado
   para "próxima fase" / "observar depois". Estes itens NÃO podem aparecer
   como requisito em nenhum documento; só em "Fora de escopo" ou
   "Questões em aberto".
3. GANCHOS COM O CÓDIGO — cada menção a arquivo, classe, método ou padrão
   existente (ex.: changeStatus, AppError, requireRole, src/worker.ts).
   Depois verifique no repositório se cada gancho existe de fato e anote o
   caminho real do arquivo.

Não interprete além do que foi dito. Se algo for ambíguo, marque como
AMBÍGUO em vez de resolver por conta própria.
```

**2. Geração do FDD ancorada no código real** (o prompt proíbe explicitamente referência não verificada):

```text
Escreva docs/FDD.md da feature de webhooks usando somente: (a) as decisões
dos ADRs já escritos em docs/adrs/, (b) a lista DECIDIDO da extração, e
(c) os arquivos reais do repositório que você leu.

Regras rígidas:
- A seção "Integração com o sistema existente" deve citar no mínimo 4
  caminhos de arquivo que EXISTEM no repositório; para cada um, descreva a
  mudança concreta (ex.: onde exatamente a chamada publishWebhookEvent
  entra no $transaction de changeStatus, e por que recebe o tx).
- Todo código de erro novo usa prefixo WEBHOOK_ e segue o padrão de
  subclasse de AppError com errorCode, igual a InsufficientStockError.
- Contratos: mínimo 4 endpoints com request/response de exemplo e status
  codes coerentes com o error.middleware existente (envelope
  { error: { code, message, details } }).
- Nada de seção genérica: se uma afirmação não tem origem na transcrição
  ou no código, remova-a.
- Não repita o conteúdo do RFC; aqui é implementação.
```

## Iterações e ajustes

O resultado final saiu em **4 ciclos principais** de geração → revisão crítica → correção:

1. **Snippet de integração tecnicamente impossível (FDD)** — a primeira versão da seção 10.1 propunha `publishWebhookEvent(tx, refreshed ?? order, from, to)` inserido "após o `orderStatusHistory.create`". Relendo o `changeStatus` real, a variável `refreshed` só é criada **depois** desse ponto da transação — o exemplo não compilaria no lugar indicado. Corrigido para `publishWebhookEvent(tx, order, from, to)`, com nota explicando por que a entidade carregada no início da transação basta para o snapshot (os campos usados não mudam na transição).
2. **Referências cruzadas quebradas** — a primeira geração dos documentos usava atalhos como `[ADR-007]` sem alvo de link em vários pontos do FDD e do ADR-005 (renderizariam como texto morto no GitHub). Ciclo de revisão converteu todos em links relativos reais para os arquivos em `docs/adrs/`.
3. **Referência interna inexistente (FDD)** — a seção de riscos citava "item 12.1" como mitigação, mas os critérios de aceite da seção 12 são numerados 1–12, sem subitens. Ajustado para "critérios 1 e 12 da seção 12".
4. **Disciplina de escopo negativo** — na estruturação do PRD, a tentação natural era listar "notificar cliente sobre webhook quebrado" como requisito (o Marcos pediu em [09:37]); a transcrição mostra a Larissa negando na mesma fala. O prompt de extração com a lista DESCARTADO/ADIADO existiu exatamente para isso: e-mail, dashboard, rate limiting, arquivamento e multi-worker terminaram em "Fora de escopo"/"Questões em aberto", cada um com seu timestamp — nenhum virou requisito.

O tracker funcionou como instrumento de verificação na prática: cada linha exigia preencher "Localização", e item sem timestamp ou arquivo identificável era sinal de invenção — foi reescrito ou removido antes da versão final.

## Como navegar a entrega

```
docs/
├── PRD.md          ← por quê e o quê (produto, escopo, métricas, riscos)
├── RFC.md          ← proposta técnica, alternativas descartadas, questões em aberto
├── FDD.md          ← como construir (fluxos, contratos, erros, integração com o código)
├── TRACKER.md      ← rastreabilidade de cada item à transcrição ou ao código
└── adrs/
    ├── README.md   ← índice das 7 decisões
    └── ADR-001…007 ← uma decisão por arquivo (formato MADR)
```

**Ordem de leitura sugerida** (do contexto para o detalhe):

1. [`TRANSCRICAO.md`](TRANSCRICAO.md) — a matéria-prima (opcional, mas dá contexto a tudo).
2. [`docs/PRD.md`](docs/PRD.md) — o problema e o que será entregue.
3. [`docs/RFC.md`](docs/RFC.md) — a proposta de solução e o que ficou em aberto.
4. [`docs/adrs/README.md`](docs/adrs/README.md) — as decisões, uma a uma.
5. [`docs/FDD.md`](docs/FDD.md) — o detalhe de implementação, para quem vai codar.
6. [`docs/TRACKER.md`](docs/TRACKER.md) — auditoria: de onde veio cada coisa.

O código da aplicação (`src/`, `prisma/`, `tests/`) permanece intacto — serviu exclusivamente de contexto e referência, conforme a regra do desafio.
