# ADR-004 — Autenticação de origem por HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Sofia (Segurança), Larissa
- **Consultados:** Bruno, Diego

## Status

Aceito

## Contexto

Os webhooks enviam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar que (a) a requisição veio de nós e (b) o payload não foi adulterado no caminho ([09:19] Sofia). Já tivemos um cliente que vazou uma secret em log da própria aplicação ([09:22] Diego), então o vazamento de uma credencial é um cenário real, não teórico.

O logger do projeto (`src/shared/logger/index.ts`) já mascara campos sensíveis (`*.password`, `*.token` e outros) via `redact`, mas não conhece nenhum campo de secret de webhook.

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature`. O cliente recalcula a assinatura do lado dele ([09:20] Sofia, [09:20] Bruno, [09:22] Sofia).
2. **Secret única por endpoint de webhook**, nunca uma secret global da plataforma ([09:21] Sofia). A configuração de cada webhook guarda `url`, `secret`, `customer_id` e o estado ativo ([09:21] Bruno, [09:21] Sofia).
3. **A secret é gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
4. **Rotação via API:** o cliente pede uma nova secret por um endpoint. A antiga continua válida em paralelo por **24 horas** (grace period) e depois é descartada ([09:21] Sofia).
5. **Revisão de segurança obrigatória:** pelo menos dois dias úteis da Sofia para revisar HMAC e geração de secret antes do deploy ([09:46] Sofia, [09:47] Larissa).

Decisões complementares de segurança, tratadas como validação ou requisito não funcional e não como ADR próprio ([09:23] Sofia, [09:24] Larissa): URL obrigatoriamente `https` (validada no schema Zod) e payload limitado a 64KB.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Se vaza uma, vaza tudo ([09:21] Sofia). |
| **Secret fixa, sem rotação** *(alternativa plausível, não discutida explicitamente)* | O histórico de vazamento em log de cliente ([09:22] Diego) mostra que é preciso poder trocar a credencial sem downtime. Por isso a rotação com grace period ([09:21] Sofia). |
| **Sem assinatura, confiando só no TLS** *(alternativa plausível, não discutida explicitamente)* | O TLS protege o transporte, mas não deixa o cliente verificar que a origem somos nós ([09:19] Sofia). Sofia tratou o HMAC como padrão ([09:20]) e o TLS como obrigatório *além* dele ([09:23]). |

## Consequências

**Positivas**
- HMAC-SHA256 é padrão de mercado: "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
- O raio de impacto de um vazamento fica limitado a um único endpoint.
- A rotação com sobreposição de 24h permite trocar a secret sem perder entregas.

**Negativas**
- A secret precisa ficar **recuperável** no banco para assinar cada envio (não dá para guardar só um hash, como `passwordHash`). A proteção em repouso fica para a revisão da Sofia (questão em aberto no [RFC](../RFC.md)).
- Durante o grace period existem duas secrets válidas. Como isso aparece na assinatura de saída (por exemplo, assinar com as duas) não foi detalhado na reunião e também é questão em aberto.
- É preciso garantir que a secret nunca apareça em logs: estender os `redactPaths` de `src/shared/logger/index.ts` e nunca devolver a secret em listagens.
- Parte do esforço vai para o cliente, que precisa implementar a verificação.

**Trade-off explícito:** aceitamos o custo de guardar secrets recuperáveis e gerenciar a rotação em troca de autenticação de origem por endpoint e um raio de impacto contido.

## Referências
- Transcrição: [09:19]–[09:24], [09:31], [09:44], [09:46]–[09:47], [09:48]
- Código: `src/shared/logger/index.ts` (`redactPaths`), `src/middlewares/validate.middleware.ts` (validação Zod)
