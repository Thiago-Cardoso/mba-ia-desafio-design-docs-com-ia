# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito — decidido em reunião técnica ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h").

## Contexto

Os webhooks expõem dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir validar duas coisas: que a requisição veio realmente de nós, e que ninguém adulterou o payload no caminho ([09:19] Sofia). Há histórico real de incidente: um cliente já vazou secret em log de aplicação ([09:22] Diego) — o raio de dano de um vazamento precisa ser contido.

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature`; o cliente verifica do lado dele com a secret compartilhada ([09:20] Sofia). SHA-256 é o padrão de mercado, com biblioteca disponível em qualquer stack séria ([09:20] Sofia).
2. **Secret única por endpoint de webhook**, não global da plataforma: "se vaza uma, vaza tudo" ([09:21] Sofia). A tabela de configuração armazena url + secret + customer_id + estado ativo ([09:21] Bruno/Sofia).
3. A secret é **gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
4. **Rotação via API**: endpoint para o cliente pedir nova secret. Durante a rotação, a secret antiga permanece válida por **24 horas em paralelo** (grace period) para dar tempo de migração; depois disso, morre ([09:21] Sofia).
5. Complementos de segurança decididos na mesma discussão:
   - **TLS obrigatório**: URL de webhook deve ser `https`; `http` é recusado com erro de validação no schema Zod ([09:23] Sofia).
   - Header `X-Timestamp` com o timestamp do envio, permitindo ao cliente detectar replay attack se quiser ([09:44] Diego).

## Alternativas Consideradas

### 1. Secret global da plataforma

Descartada explicitamente ([09:21] Sofia). Simplificaria o armazenamento e a rotação, mas um único vazamento comprometeria a autenticidade dos webhooks de **todos** os clientes. O incidente real de secret vazada em log de cliente ([09:22] Diego) mostrou que vazamento não é hipótese teórica. Trade-off: gestão de N secrets (uma por endpoint) em troca de raio de dano contido.

### 2. Nenhuma assinatura (confiar apenas em TLS)

Alternativa implícita rejeitada pela colocação do problema ([09:19] Sofia): TLS protege o transporte, mas não permite ao cliente autenticar a **origem** da chamada nem detectar um payload forjado por terceiro que conheça a URL. HMAC é o padrão de mercado para webhooks.

## Consequências

**Positivas**

- Cliente valida origem e integridade de cada entrega com verificação barata (HMAC local).
- Vazamento de uma secret compromete apenas um endpoint de um cliente.
- Rotação sem downtime de integração graças ao grace period de 24h.
- Validação de `https` é uma regra simples de schema, no padrão Zod já usado pelo projeto (`src/middlewares/validate.middleware.ts`).

**Negativas / Trade-offs**

- Durante o grace period de 24h existem duas secrets válidas por endpoint — a verificação de assinatura do lado do cliente e o armazenamento do nosso lado precisam lidar com esse estado duplo.
- A responsabilidade de verificar a assinatura é do cliente; exige documentação clara no portal de desenvolvedor ([09:26] Marcos).
- Geração e manuseio de secret são código sensível: a revisão de segurança da Sofia (mínimo 2 dias úteis antes do deploy) é etapa obrigatória do plano ([09:46] Sofia).

## Referências

- Transcrição: [09:19]–[09:23], [09:31], [09:44], [09:46]
- Código: `src/middlewares/validate.middleware.ts` (padrão de validação Zod)
