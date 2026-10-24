# ADR-002 — Worker em processo separado, consumindo a outbox por polling de 2 segundos

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa, Diego, Bruno
- **Consultados:** Marcos

## Status

Aceito

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), falta decidir **como** e **onde** os eventos são consumidos e enviados.

- Requisito de latência: para os clientes, qualquer coisa abaixo de **10 segundos** já é "tempo real" ([09:02] Marcos).
- O MySQL não tem um mecanismo nativo de notificação para processos externos, como o `LISTEN/NOTIFY` do Postgres ([09:09] Diego).
- Hoje a aplicação tem um único entry-point, `src/server.ts`, que sobe o Express e controla o shutdown com `SIGINT`/`SIGTERM`. O Prisma Client é criado em `src/config/database.ts`.

## Decisão

1. **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos (ordenados por `created_at`), processa e marca o resultado ([09:09] Diego). O pior caso de latência até a primeira tentativa fica em ~2s, o que foi aceito ([09:10] Larissa, [09:10] Marcos).
2. **Processo separado da API.** Se a API reinicia, não pode levar o worker junto ([09:11] Diego). O novo entry-point é `src/worker.ts`, executado pelo script `npm run worker`, seguindo o modelo de `src/server.ts` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) ([09:28] Bruno).
3. **Mesmo banco e mesma stack, instância própria de Prisma.** O worker usa a mesma `DATABASE_URL`, mas cria seu próprio `PrismaClient`, porque o client é por processo ([09:11] Bruno, [09:11] Diego, [09:30] Bruno).
4. **Single-worker com ordering implícito por `order_id`.** Com um único worker processando em ordem de `created_at`, o cliente recebe os eventos de um mesmo pedido na ordem certa. **Não há garantia de ordering global**, e a garantia por pedido só vale enquanto houver um único worker. Isso é uma limitação conhecida ([09:12] Diego, [09:13] Larissa). Os clientes não pediram ordering global ([09:14] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no banco para um consumo reativo** | A trigger do MySQL só executa SQL e não notifica processo externo. Avisar o worker exigiria gambiarras (escrever em arquivo, chamar um endpoint) ([09:09] Bruno, [09:09] Diego). E o polling de 2s já atende os 10s com folga. |
| **Worker rodando dentro do processo da API** | Um restart ou crash da API derrubaria o worker junto ([09:11] Diego). |
| **Múltiplos workers em paralelo** | Perde a garantia de ordem por pedido. Particionar por `order_id` ou usar lock pessimista fica como evolução futura, "problema do futuro, não agora" ([09:13] Diego). |

## Consequências

**Positivas**
- Simples de implementar, operar e testar. Usa só Node, Prisma e MySQL.
- A latência fica bem abaixo do requisito de 10s.
- O ciclo de vida do worker é independente da API (deploy, restart, falha).
- A ordem por pedido vem de graça no modelo single-worker.

**Negativas**
- Consultas periódicas ao banco mesmo quando não há eventos (custo baixo, graças ao índice em `status`/`created_at`).
- O throughput fica limitado a um worker. Escalar horizontalmente exige rever a estratégia de ordering (questão em aberto no [RFC](../RFC.md)).
- Um novo processo para implantar e monitorar. Se o worker parar, os eventos acumulam como pendentes (não se perdem, mas atrasam).

**Trade-off explícito:** trocamos reatividade e escala horizontal por simplicidade operacional e ordering por pedido, aceitando até 2s de latência adicional.

## Referências
- Transcrição: [09:02], [09:08]–[09:14], [09:28], [09:30], [09:48]
- Código: `src/server.ts`, `src/config/database.ts`, `package.json` (seção `scripts`)
