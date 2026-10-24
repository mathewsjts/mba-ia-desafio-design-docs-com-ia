# ADR-001 — Padrão Outbox transacional no MySQL existente

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)

## Status

Aceito

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status dos seus pedidos muda, em vez de fazer polling em `GET /orders` ([09:00] Marcos). A notificação é apenas de saída, da plataforma para o cliente ([09:02] Marcos, [09:03] Sofia).

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já faz bastante coisa: valida a transição (`src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders` e grava `order_status_history` ([09:04] Bruno). Duas forças entram em conflito:

1. **Consistência:** se o status mudou, o evento tem que existir. Se a transação deu rollback, o evento não pode existir ([09:06] Diego, [09:40] Bruno).
2. **Isolamento:** uma chamada HTTP a um cliente lento ou fora do ar não pode travar nem desfazer a mudança de status ([09:04] Bruno).

A aplicação não tem nenhuma fila, broker ou mecanismo de eventos hoje. O único armazenamento é o MySQL acessado via Prisma (`prisma/schema.prisma`).

## Decisão

Adotar o **padrão Outbox** no MySQL que já existe:

- Criar a tabela `webhook_outbox`. Na **mesma transação SQL** de `changeStatus`, depois de gravar `order_status_history`, inserir nela uma linha por evento a ser entregue ([09:06] Diego, [09:40] Bruno).
- Se a inserção na outbox falhar, a transação inteira dá rollback. Não pode existir um status alterado sem o evento correspondente ([09:40] Bruno, [09:41] Diego).
- A inserção é feita por uma função que recebe o client da transação corrente, `publishWebhookEvent(tx, order, fromStatus, toStatus)`, em vez de injetar um repositório inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at`. O consumidor lê só os pendentes, em batches pequenos ([09:08] Diego).
- O envio HTTP fica com um worker separado (ver [ADR-002](ADR-002-worker-separado-com-polling.md)).

Larissa fechou a decisão em [09:08] e a repetiu no resumo de [09:48].

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento trava a mudança de status de outros pedidos, e um cliente fora do ar forçaria um rollback do status, o que não faz sentido ([09:04] Bruno). "Síncrono está fora de questão" ([09:06] Diego). |
| **Redis Streams (ou fila similar)** | Exige subir e operar mais infraestrutura. Para um time pequeno, um Redis Cluster só para isso é overengineering ([09:07] Larissa, [09:07] Diego). *Análise complementar, não dita na reunião:* publicar num broker externo de dentro da transação deixaria de ser atômico com o commit do pedido. |

## Consequências

**Positivas**
- Atomicidade total entre a mudança de status e o registro do evento, sem 2PC e sem estados inconsistentes ([09:06] Diego).
- Nenhuma infraestrutura nova: mesmo MySQL, mesmo Prisma, mesmo backup.
- A latência de clientes externos fica totalmente fora do caminho crítico da API.
- A outbox também serve de trilha de auditoria do que foi gerado.

**Negativas**
- A transação de `changeStatus`, que já é pesada, ganha mais um `INSERT` (e a consulta de quais webhooks assinam aquele status, ver [ADR-007](ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md)).
- A tabela cresce continuamente. O arquivamento das linhas entregues (aprox. 30 dias) foi explicitamente deixado **fora do escopo** desta feature ([09:08] Diego), então existe uma dívida operacional conhecida.
- A entrega passa a ser assíncrona, com latência mínima igual ao intervalo de polling ([ADR-002](ADR-002-worker-separado-com-polling.md)).

**Trade-off explícito:** aceitamos o custo de um `INSERT` a mais na transação crítica e o crescimento da tabela em troca de consistência garantida sem infraestrutura nova.

## Referências
- Transcrição: [09:03]–[09:08], [09:40]–[09:41], [09:48]
- Código: `src/modules/orders/order.service.ts` (`changeStatus`), `src/modules/orders/order.status.ts`, `prisma/schema.prisma`
