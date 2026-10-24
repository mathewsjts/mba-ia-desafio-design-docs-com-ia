# ADR-005 — Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa, Diego
- **Consultados:** Sofia, Marcos, Bruno

## Status

Aceito

## Contexto

A combinação de outbox ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), worker ([ADR-002](ADR-002-worker-separado-com-polling.md)) e retry/replay ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)) cria janelas em que o mesmo evento pode ser enviado mais de uma vez. Por exemplo: o cliente processa a requisição, mas a resposta não chega dentro do timeout de 10s, e o worker agenda um retry. Também há o replay manual da DLQ. É preciso definir qual garantia a plataforma oferece e como o cliente lida com duplicatas.

## Decisão

1. **A garantia oferecida é at-least-once.** O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado ([09:24] Diego).
2. **Cada evento tem um `event_id` (UUID) gerado no momento em que entra na outbox**, único por evento, enviado no header **`X-Event-Id`** (e também no corpo, como `event_id`) ([09:25] Diego, [09:43] Diego). O cliente deduplica por esse identificador ([09:25] Diego).
3. **O `event_id` não muda em retentativas nem em replay.** *Interpretação derivada, não dita literalmente:* Diego disse que o replay "recoloca na outbox como pendente" ([09:18]), e o `event_id` só cumpre o papel de chave de deduplicação se for o mesmo em todo reenvio do evento ([09:25] Diego).
4. **Comunicação ao cliente:** Marcos documenta a semântica com destaque no portal do desenvolvedor ([09:26] Marcos).

Larissa fechou a decisão em [09:26] ("At-least-once com X-Event-Id pra dedup do lado do cliente").

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação dos dois lados e fica muito mais complexo ([09:25] Diego). |
| **At-most-once** (enviar uma vez, sem retry) | Incompatível com a política de retry decidida em [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md) e com o objetivo de o cliente não perder mudanças de status ([09:00] Marcos). Alternativa plausível, não discutida explicitamente. |

## Consequências

**Positivas**
- É o padrão de mercado: "Stripe faz assim, GitHub faz assim" ([09:25] Diego). Clientes integradores já conhecem o modelo.
- A implementação do lado da plataforma fica simples, sem estado distribuído compartilhado com o cliente.
- Resolve "99% dos casos" ([09:25] Diego).

**Negativas**
- **Joga responsabilidade para o cliente** ([09:25] Sofia), que precisa guardar os `event_id` já processados.
- Um cliente que não implementar a deduplicação pode processar a mesma mudança de status duas vezes.
- A documentação passa a fazer parte do contrato (dependência do portal do desenvolvedor).

**Trade-off explícito:** trocamos a garantia mais forte (exactly-once) por simplicidade de implementação, transferindo a idempotência para o consumidor.

## Referências
- Transcrição: [09:18], [09:24]–[09:26], [09:43]–[09:44], [09:48]
