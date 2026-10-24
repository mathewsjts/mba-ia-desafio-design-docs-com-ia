# ADR-007 — Payload enxuto renderizado como snapshot e filtrado por status na inserção da outbox

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa, Diego, Bruno
- **Consultados:** Marcos, Sofia

## Status

Aceito

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), três perguntas secundárias, mas que mexem no desenho da tabela e da transação, foram discutidas:

1. **Quando filtrar:** cada webhook escolhe quais status quer receber, por exemplo "só SHIPPED e DELIVERED" ([09:33] Marcos). O filtro é aplicado na inserção da outbox ou no envio? ([09:34] Diego)
2. **O que enviar:** quais campos vão no payload ([09:43] Marcos).
3. **Quando renderizar:** a outbox guarda o payload já pronto ou só o `order_id`, renderizando no envio? ([09:51] Bruno)

Os status possíveis vêm do enum `OrderStatus` (`prisma/schema.prisma`) e as transições válidas de `src/modules/orders/order.status.ts`.

## Decisão

1. **Filtrar na inserção.** Se nenhum webhook do customer quer aquele status de destino, nada é inserido na outbox ([09:34] Bruno, [09:34] Diego).
2. **Payload JSON enxuto:** `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Sem `items`**: quem quiser detalhes consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).
3. **Snapshot na inserção.** O payload é renderizado e gravado no momento em que o evento entra na outbox. Se o pedido mudar depois, o evento continua refletindo o estado do momento da transição ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
Relacionado, mas **fora deste ADR**: o limite de 64KB por payload, com erro em vez de truncamento, foi classificado na reunião como requisito não funcional, não como decisão arquitetural ([09:24] Larissa). Ele está no PRD (NFR-07). O payload enxuto deste ADR mantém os eventos bem abaixo desse teto ([09:24] Diego).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Filtrar no momento do envio** | Gravaria linhas que nunca seriam enviadas. Filtrar na inserção "economiza linha na tabela" ([09:34] Bruno). |
| **Guardar só `order_id` e renderizar no envio** | Com retries de até ~15h, o pedido pode ter mudado e o evento chegaria com dados que não batem com a transição: "tem caso esquisito" ([09:52] Larissa). |
| **Incluir `items` no payload** | Infla o payload sem necessidade ([09:43] Diego). |

## Consequências

**Positivas**
- Menos linhas na outbox e menos trabalho para o worker.
- O evento é imutável e fiel ao momento da transição, o que é coerente com retries longos e com o replay ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)).
- O payload pequeno reduz o custo de rede e o risco de expor dados.
- O worker não precisa consultar o pedido para enviar. Ele lê só a outbox e a configuração do endpoint (URL e secret).

**Negativas**
- A transação de `changeStatus` ganha uma leitura extra (webhooks ativos do customer que assinam o `to_status`).
- **Mudar o filtro não tem efeito retroativo**: eventos que não entraram na outbox não podem ser recuperados depois.
- O snapshot ocupa mais espaço por linha do que guardar só o `order_id`.
- Clientes que precisam dos itens fazem uma segunda chamada (`GET /orders/:id`).

**Trade-off explícito:** aceitamos uma leitura a mais na transação e linhas maiores em troca de eventos fiéis ao momento da mudança e de uma outbox sem lixo.

## Referências
- Transcrição: [09:23]–[09:24], [09:33]–[09:34], [09:43]–[09:44], [09:51]–[09:52]
- Código: `prisma/schema.prisma` (enum `OrderStatus`, model `Order`), `src/modules/orders/order.status.ts`
