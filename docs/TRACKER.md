# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado no pacote de documentação e a sua origem: a reunião (`TRANSCRICAO.md`) ou o código-fonte do repositório.

- **Fonte = TRANSCRICAO** → `Localização` no formato `[hh:mm] Nome` (fala principal que sustenta o item).
- **Fonte = CODIGO** → `Localização` é o caminho real do arquivo no repositório.
- Itens marcados como **Proposta** no FDD (§14) são detalhes não decididos na reunião. A fonte indicada é a fala ou o arquivo que justifica a proposta.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Padrão Outbox no MySQL, evento gravado na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-D1 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Decisão fechada pela tech lead: "outbox em MySQL" | TRANSCRICAO | [09:08] Larissa |
| ADR-001-D2 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Restrição | Falha ao inserir na outbox faz rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| ADR-001-D3 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Função `publishWebhookEvent(tx, …)` em vez de injetar repositório | TRANSCRICAO | [09:41] Bruno |
| ADR-001-D4 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Índices em status e created_at; worker lê pendentes em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Contexto | `changeStatus` já faz update de orders, history e estoque numa `$transaction` | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa descartada | HTTP síncrono no service: cliente lento trava status, rollback inviável | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa descartada | Redis Streams: infra extra, overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-CONS | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Trade-off | Crescimento da tabela; arquivamento de 30 dias fora do escopo | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker consome outbox por polling a cada 2s | TRANSCRICAO | [09:09] Diego |
| ADR-002-D1 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Latência mínima de 2s no pior caso, aceita | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D2 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker em processo separado: `src/worker.ts` + `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D3 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Mesmo banco, PrismaClient próprio por ser outro processo | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D4 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Ordem garantida só por order_id e só com single-worker | TRANSCRICAO | [09:12] Diego |
| ADR-002-D5 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restrição | Limitação documentada: sem ordering global | TRANSCRICAO | [09:13] Larissa |
| ADR-002-CTX | docs/adrs/ADR-002-worker-separado-com-polling.md | Contexto | Entry-point atual e padrão de bootstrap/shutdown | CODIGO | src/server.ts |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa descartada | Trigger/LISTEN-NOTIFY: MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa descartada | Worker dentro da API: restart da API derruba o worker | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa adiada | Múltiplos workers (particionar por order_id / lock pessimista) | TRANSCRICAO | [09:13] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h, teto de 5 | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D1 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Interpretação: 1 envio + 5 retentativas (~14,6h, "quase 15 horas") | TRANSCRICAO | [09:17] Diego |
| ADR-003-D2 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela `webhook_dead_letter` (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| ADR-003-D2b | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Linha da outbox fica no status terminal "falhou" | TRANSCRICAO | [09:08] Diego |
| ADR-003-D3 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Replay manual via `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| ADR-003-D4 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Restrição | Replay exige ADMIN e loga quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido: evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas: esgota em ~30 min, menos que manutenção de 2h | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa descartada | Marcar "failed" na própria outbox em vez de tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-CONS | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Cliente fora por mais de 15h tem "problema sério dele" | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D1 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura enviada no header `X-Signature` | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D2 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint de webhook | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D3 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| ADR-004-D4 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Revisão de segurança de ≥ 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global: se vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa plausível (não discutida) | Secret sem rotação: histórico de vazamento em log de cliente | TRANSCRICAO | [09:22] Diego |
| ADR-004-ALT-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa plausível (não discutida) | Confiar só no TLS, sem assinatura | TRANSCRICAO | [09:19] Sofia |
| ADR-004-CONS | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret precisa ser recuperável; estender redaction do logger | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once | TRANSCRICAO | [09:24] Diego |
| ADR-005-D1 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | `X-Event-Id` UUID gerado na entrada da outbox, dedup pelo cliente | TRANSCRICAO | [09:25] Diego |
| ADR-005-D2 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Interpretação | event_id estável no replay ("recoloca na outbox") | TRANSCRICAO | [09:18] Diego |
| ADR-005-D3 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Fechamento da decisão pela tech lead | TRANSCRICAO | [09:26] Larissa |
| ADR-005-D4 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Dependência | Documentação destacada no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once: coordenação dos dois lados, muito complexo | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar fica com o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D1 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Pasta `src/modules/webhooks` com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Lógica do worker em `webhook.worker.ts`/`webhook.processor.ts` | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D3 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Prefixo `WEBHOOK_` em todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D4 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Pino como único logger; error middleware sem alterações | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D5 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | IDs UUID em todas as tabelas novas | TRANSCRICAO | [09:51] Larissa |
| ADR-006-CTX-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | `AppError` com statusCode/errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CTX-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | `InvalidStatusTransitionError`, `InsufficientStockError` com códigos CONSTANT_CASE | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CTX-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Middleware de erro trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CTX-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | `requireRole('ADMIN')` já usado em rotas de usuários | CODIGO | src/modules/users/user.routes.ts |
| ADR-006-CTX-05 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | IDs `@default(uuid()) @db.Char(36)` em todos os models | CODIGO | prisma/schema.prisma |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | Injetar repositório inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | ID auto-incremental na outbox | TRANSCRICAO | [09:51] Diego |
| ADR-006-CONS | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | CRUD com qualquer role autenticada por ora | TRANSCRICAO | [09:37] Sofia |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Decisão | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-D1 | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Decisão | Payload JSON enxuto, sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-D2 | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-D3 | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Restrição (NFR referenciado) | Limite de 64KB tratado como NFR, não como decisão do ADR | TRANSCRICAO | [09:24] Larissa |
| ADR-007-CTX | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Contexto | Status possíveis e transições válidas | CODIGO | src/modules/orders/order.status.ts |
| ADR-007-ALT-01 | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md | Alternativa descartada | Filtrar no momento do envio | TRANSCRICAO | [09:34] Diego |
| RFC-META-01 | docs/RFC.md | Metadado | Autoria da tech lead, que se comprometeu a abrir o doc de design e marcar revisão | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Três clientes B2B fazem polling em GET /orders e pediram notificação | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Requisito Não Funcional | "Tempo real" = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Webhook apenas de saída | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | docs/RFC.md | Contexto | Toda mudança de status passa por `OrderService.changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Proposta | Módulo de configuração com CRUD, rotação e histórico | TRANSCRICAO | [09:31] Marcos |
| RFC-PROP-02 | docs/RFC.md | Proposta | Publicação transacional via `publishWebhookEvent` | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Proposta | Worker separado com polling de 2s | TRANSCRICAO | [09:11] Diego |
| RFC-PROP-04 | docs/RFC.md | Proposta | Retry, DLQ e replay admin auditado | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-05 | docs/RFC.md | Proposta | Contrato: payload enxuto, headers, at-least-once | TRANSCRICAO | [09:44] Diego |
| RFC-PROP-06 | docs/RFC.md | Restrição | Nenhuma infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| RFC-PROP-07 | docs/RFC.md | Restrição | Segurança: TLS, HMAC, secret por endpoint, 64KB | TRANSCRICAO | [09:23] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila externa | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco reativa | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker no processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido / 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Aviso por e-mail após falhas: próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Escala multi-worker sem perder ordem | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | customer_id no body ou no path (proposta: body no POST, path na listagem) | TRANSCRICAO | [09:32] Larissa |
| RFC-Q-06 | docs/RFC.md | Questão em aberto | "5 tentativas" incluem o envio inicial? | TRANSCRICAO | [09:17] Diego |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | Assinatura no grace period e proteção da secret em repouso | TRANSCRICAO | [09:46] Sofia |
| RFC-Q-08 | docs/RFC.md | Questão em aberto | Retry pode inverter a ordem de eventos do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| RFC-IMP-01 | docs/RFC.md | Impacto | Única alteração em fluxo existente: chamada em changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-02 | docs/RFC.md | Impacto | Registro de rotas e controllers | CODIGO | src/routes/index.ts |
| RFC-IMP-03 | docs/RFC.md | Impacto | Prazo de três sprints com revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-IMP-04 | docs/RFC.md | Impacto | Meta da Atlas: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Mais carga na transação crítica de changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Ponto único de processamento (single-worker) | TRANSCRICAO | [09:13] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Semântica at-least-once depende do cliente | TRANSCRICAO | [09:26] Marcos |
| FDD-CTX-01 | docs/FDD.md | Contexto | Etapas atuais de changeStatus (validação, estoque, update, history, refresh) | CODIGO | src/modules/orders/order.service.ts |
| FDD-CTX-02 | docs/FDD.md | Contexto | Validação de transição por `canTransition` | CODIGO | src/modules/orders/order.status.ts |
| FDD-OT-01 | docs/FDD.md | Objetivo técnico | Status alterado ⇔ evento gravado (rollback conjunto) | TRANSCRICAO | [09:41] Diego |
| FDD-OT-02 | docs/FDD.md | Objetivo técnico | Primeira tentativa em ~2s, teto de 10s | TRANSCRICAO | [09:10] Larissa |
| FDD-OT-03 | docs/FDD.md | Objetivo técnico | Nenhum HTTP externo no processo/transação da API | TRANSCRICAO | [09:04] Bruno |
| FDD-OT-04 | docs/FDD.md | Objetivo técnico | At-least-once com X-Event-Id estável | TRANSCRICAO | [09:25] Diego |
| FDD-OT-05 | docs/FDD.md | Objetivo técnico | Retry com backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-OT-06 | docs/FDD.md | Objetivo técnico | Toda requisição assinada com HMAC-SHA256 | TRANSCRICAO | [09:22] Sofia |
| FDD-OT-07 | docs/FDD.md | Objetivo técnico | Zero dependências novas de runtime | TRANSCRICAO | [09:30] Larissa |
| FDD-EXC-01 | docs/FDD.md | Exclusão | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | docs/FDD.md | Exclusão | E-mail de alerta | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | docs/FDD.md | Exclusão | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | docs/FDD.md | Exclusão | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | docs/FDD.md | Exclusão | Arquivamento de 30 dias | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | docs/FDD.md | Exclusão | Múltiplos workers / ordering global | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-07 | docs/FDD.md | Exclusão | Restrição de role no CRUD | TRANSCRICAO | [09:37] Sofia |
| FDD-EXC-08 | docs/FDD.md | Exclusão | Evento na criação do pedido (create não passa por changeStatus) | CODIGO | src/modules/orders/order.service.ts |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | `webhook_endpoints`: url, secret, customer_id, ativo, events | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | `previousSecret` + expiração para o grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | `webhook_outbox`: status pendente/processando/falhou/entregue, índices | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | `webhook_deliveries`: sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | `webhook_dead_letter`: payload, motivo, timestamp (N por evento) | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | `replayedAt`/`replayedById` para auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-DADOS-07 | docs/FDD.md | Padrão | Convenções Prisma: uuid Char(36), @@map snake_case, enum OrderStatus | CODIGO | prisma/schema.prisma |
| FDD-DADOS-08 | docs/FDD.md | Proposta | Cascade para não bloquear exclusão de customer | CODIGO | src/modules/customers/customer.service.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox após history, dentro da mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro por status assinado na inserção | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Falha na outbox: log explícito + rethrow (middleware não loga AppError) | CODIGO | src/middlewares/error.middleware.ts |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Loop de polling a cada 2s | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Processamento sequencial por created_at (ordem por pedido) | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Validações pré-envio, incluindo 64KB sem truncar | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Envio com fetch nativo (Node ≥ 20) | CODIGO | package.json |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo | Timeout de 10s por tentativa | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-10 | docs/FDD.md | Fluxo | Registro de cada tentativa em webhook_deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-11 | docs/FDD.md | Fluxo | Tabela de backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-12 | docs/FDD.md | Fluxo | Movimento para DLQ em transação única | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-13 | docs/FDD.md | Fluxo | Replay recoloca o mesmo evento como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-14 | docs/FDD.md | Fluxo | Log de auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-15 | docs/FDD.md | Fluxo | Rotação com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks (secret só na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/customers/:customerId/webhooks (lista por customer) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries (as 100 mais recentes, sem paginação) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Payload de saída (event_id, event_type, timestamp, order_id, …, total_cents) | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | customerId não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | URL https validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Formato de erro `{ error: { code, message, details } }` | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato | Paginação `{ data, pagination }` | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-15 | docs/FDD.md | Contrato | API em camelCase, como os schemas existentes | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-CONTRATO-16 | docs/FDD.md | Contrato | CRUD com qualquer role autenticada; replay ADMIN | TRANSCRICAO | [09:36] Larissa |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (404) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_REPLAY_CONFLICT (409) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_PUBLISH_FAILED (500) com rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-ERRO-06 | docs/FDD.md | Erro | Falha de Zod responde VALIDATION_ERROR (middleware compartilhado) | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERRO-07 | docs/FDD.md | Erro | NotFoundError tem código fixo: 404 específicos estendem AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-08 | docs/FDD.md | Erro | FORBIDDEN via requireRole no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-ERRO-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR / NETWORK_ERROR (retentáveis) | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-11 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED → DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-12 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (> 64KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-13 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (defesa em profundidade) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-14 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED reservado (secret sempre gerada pela plataforma) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-15 | docs/FDD.md | Erro | WEBHOOK_INACTIVE (endpoint inativo no envio) | TRANSCRICAO | [09:21] Bruno |
| FDD-RES-01 | docs/FDD.md | Resiliência | Retries persistidos em nextAttemptAt | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-02 | docs/FDD.md | Resiliência | Fallback = DLQ + replay manual (sem e-mail) | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Recuperação de PROCESSING no boot (at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Restart da API não afeta o worker | TRANSCRICAO | [09:11] Diego |
| FDD-RES-05 | docs/FDD.md | Resiliência | Shutdown gracioso SIGINT/SIGTERM | CODIGO | src/server.ts |
| FDD-RES-06 | docs/FDD.md | Proposta | Banco indisponível: ciclo logado e retentado sem derrubar o worker | TRANSCRICAO | [09:06] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados com Pino, sem lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logger e redaction existentes | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Redaction de secret (vazamento em log de cliente) | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Métrica de latência vs meta de 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Métrica de DLQ como insumo para decisão do e-mail | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Métrica de volume por cliente como insumo para rate limiting | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | Tracing por correlação de event_id | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-08 | docs/FDD.md | Observabilidade | requestId (X-Request-Id) como correlação opcional | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-09 | docs/FDD.md | Observabilidade | Log de auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | Chamada a publishWebhookEvent dentro de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Erros do módulo estendem AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros do módulo estendem ConflictError etc. | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Barrel de erros existente | CODIGO | src/shared/errors/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | error middleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | validate({ body, query, params }) com Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Montagem de /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Composição em buildControllers | CODIGO | src/app.ts |
| FDD-INT-10 | docs/FDD.md | Integração | src/worker.ts espelha bootstrap/shutdown | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | createPrismaClient() no worker | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Worker exige o mesmo .env (JWT_SECRET, DATABASE_URL) | CODIGO | src/config/env.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Logger child + redactPaths estendidos | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | paginated() em listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Novos models, enum e migration | CODIGO | prisma/schema.prisma |
| FDD-INT-16 | docs/FDD.md | Integração | Máquina de estados inalterada | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-17 | docs/FDD.md | Integração | Scripts worker / worker:dev; uuid reaproveitado | CODIGO | package.json |
| FDD-INT-18 | docs/FDD.md | Integração | Padrão de testes e factory createTestWebhook | CODIGO | tests/helpers/factories.ts |
| FDD-INT-19 | docs/FDD.md | Integração | Testes de API no padrão supertest | CODIGO | tests/orders.test.ts |
| FDD-INT-20 | docs/FDD.md | Integração | Rotas admin no padrão requireRole de usuários | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-21 | docs/FDD.md | Integração | Estrutura de arquivos novos do módulo | TRANSCRICAO | [09:28] Bruno |
| FDD-DEP-01 | docs/FDD.md | Dependência | Runtime sem dependências novas (crypto, fetch, uuid, zod) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Restrição | Exatamente uma instância do worker | TRANSCRICAO | [09:12] Diego |
| FDD-DEP-03 | docs/FDD.md | Dependência | Portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| FDD-DEP-04 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-CA-01 | docs/FDD.md | Critério de aceite | 1 linha PENDING por endpoint assinante na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de aceite | Falha na outbox mantém status/histórico/estoque | TRANSCRICAO | [09:41] Diego |
| FDD-CA-03 | docs/FDD.md | Critério de aceite | Status não assinado não gera linha | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-04 | docs/FDD.md | Critério de aceite | Entrega em ≤ 2s com headers e assinatura válidos | TRANSCRICAO | [09:44] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de aceite | Backoff correto e DLQ após a última falha | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-06 | docs/FDD.md | Critério de aceite | Payload > 64KB vai à DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-07 | docs/FDD.md | Critério de aceite | Replay: mesmo id, ADMIN only, 403/404/409 | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-08 | docs/FDD.md | Critério de aceite | http recusado; secret só na criação | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de aceite | Rotação: antiga deixa de valer após 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério de aceite | Deliveries: mais recentes primeiro, máx. 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-11 | docs/FDD.md | Critério de aceite | Worker morto e reiniciado não perde evento | TRANSCRICAO | [09:06] Diego |
| FDD-CA-12 | docs/FDD.md | Critério de aceite | Nenhum log com secret | TRANSCRICAO | [09:22] Diego |
| FDD-CA-13 | docs/FDD.md | Critério de aceite | Ordem por order_id com single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Latência/lock na transação de changeStatus | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Worker parado sem ninguém perceber | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Duas instâncias do worker por engano | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Retry pode inverter a ordem por pedido | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-05 | docs/FDD.md | Risco | Cliente lento atrasa os demais (envio sequencial) | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | X-Timestamp fora do material assinado | TRANSCRICAO | [09:44] Diego |
| FDD-RISK-08 | docs/FDD.md | Risco | Crescimento das tabelas sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-09 | docs/FDD.md | Risco | Sem vínculo User→Customer: qualquer usuário gerencia qualquer customer | CODIGO | prisma/schema.prisma |
| FDD-PROP-01 | docs/FDD.md | Proposta | 1 linha de outbox por endpoint assinante | TRANSCRICAO | [09:44] Sofia |
| FDD-PROP-02 | docs/FDD.md | Proposta | 5 tentativas = 5 retentativas após o envio inicial (WEBHOOK_MAX_RETRIES = 5) | TRANSCRICAO | [09:17] Diego |
| FDD-PROP-03 | docs/FDD.md | Proposta | customerId no body (POST) e na query (GET) | TRANSCRICAO | [09:32] Larissa |
| FDD-PROP-04 | docs/FDD.md | Proposta | Path de rotação e 202 no replay | TRANSCRICAO | [09:21] Sofia |
| FDD-PROP-05 | docs/FDD.md | Proposta | Formato `sha256=<hex>` e segunda assinatura no grace period | TRANSCRICAO | [09:20] Sofia |
| FDD-PROP-06 | docs/FDD.md | Proposta | Falhas determinísticas direto para a DLQ | TRANSCRICAO | [09:23] Sofia |
| FDD-PROP-07 | docs/FDD.md | Proposta | DELETE físico em cascata; active=false para pausar | TRANSCRICAO | [09:21] Bruno |
| FDD-PROP-08 | docs/FDD.md | Proposta | Recuperação de PROCESSING no boot | TRANSCRICAO | [09:24] Diego |
| FDD-PROP-09 | docs/FDD.md | Proposta | Batch de 10, setTimeout encadeado, redirect: 'error' | TRANSCRICAO | [09:08] Diego |
| FDD-PROP-10 | docs/FDD.md | Proposta | create() (null → PENDING) não gera evento | TRANSCRICAO | [09:40] Bruno |
| FDD-PROP-11 | docs/FDD.md | Proposta | Corpo de resposta truncado em 4KB | TRANSCRICAO | [09:34] Marcos |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Polling em GET /orders deixa a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição de negócio | Atlas pode migrar para concorrente; pediu fim de novembro (a confirmar) | TRANSCRICAO | [09:45] Marcos |
| PRD-PER-01 | docs/PRD.md | Público-alvo | Integrador técnico do cliente B2B | TRANSCRICAO | [09:00] Marcos |
| PRD-PER-02 | docs/PRD.md | Público-alvo | Usuário da plataforma que representa o cliente (JWT do OMS) | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-03 | docs/PRD.md | Público-alvo | Administrador (role ADMIN) para replay | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | docs/PRD.md | Cenário | Assinar só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Cliente em manutenção de 2h recebe após retry | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Diagnóstico pelo histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-04 | docs/PRD.md | Cenário | Admin reenvia evento da fila de falhas | TRANSCRICAO | [09:18] Diego |
| PRD-CEN-05 | docs/PRD.md | Cenário | Troca de secret com 24h de sobreposição | TRANSCRICAO | [09:21] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Primeira tentativa < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 100% das mudanças assinadas com evento registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Janela de ~15h de retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Em produção até o fim de novembro (prazo pedido pela Atlas) | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de alerta por falhas (adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual (projeto do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída (observar) | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Ordem global / múltiplos workers | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Arquivamento de 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Restrição de perfil no cadastro | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook; secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | customer_id explícito, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Notificação a cada mudança de status assinada | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Conteúdo da notificação, sem items | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Notificação assinada (HMAC) com headers de identificação | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24h de validade da antiga | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retentativas automáticas (teto 5, 1m..12h; contagem em RFC Q-06) e fila de falhas | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Reenvio pelo admin, com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10s (polling de 2s) | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade status ⇔ notificação | TRANSCRICAO | [09:41] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Só URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64KB, sem truncar | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Secret única por webhook | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordem por pedido com processador único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Processamento separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Padrões da plataforma e prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Auditoria do reprocessamento manual | TRANSCRICAO | [09:36] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Resumo das decisões e trade-offs (remete aos ADRs) | TRANSCRICAO | [09:48] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Fluxo de mudança de status existente | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | MySQL existente, sem infra nova | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-03 | docs/PRD.md | Dependência | Novo processo de entrega | TRANSCRICAO | [09:11] Larissa |
| PRD-DEP-04 | docs/PRD.md | Dependência | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-05 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-06 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RSK-01 | docs/PRD.md | Risco | Perda da Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RSK-02 | docs/PRD.md | Risco | Cliente processa duplicata | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-03 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Cliente sobrecarregado por rajadas | TRANSCRICAO | [09:38] Diego |
| PRD-RSK-05 | docs/PRD.md | Risco | Cliente não percebe falhas (sem e-mail) | TRANSCRICAO | [09:37] Marcos |
| PRD-RSK-06 | docs/PRD.md | Risco | Processador de entregas parado | TRANSCRICAO | [09:11] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Notificação assinada em < 10s para status assinado | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Status não assinado não notifica | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | Atomicidade entre status e notificação | TRANSCRICAO | [09:40] Bruno |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Retentativas 1m..12h e fila de falhas | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | Reenvio ADMIN com mesmo identificador; OPERATOR negado | TRANSCRICAO | [09:36] Larissa |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | Secret só na criação/rotação; antiga vale 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Histórico de até 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | Restart da API não interrompe entregas | TRANSCRICAO | [09:11] Diego |
| PRD-TST-01 | docs/PRD.md | Estratégia de teste | Testes unitários com Vitest | CODIGO | vitest.config.ts |
| PRD-TST-02 | docs/PRD.md | Estratégia de teste | Integração de API com supertest e factories | CODIGO | tests/orders.test.ts |
| PRD-TST-03 | docs/PRD.md | Estratégia de teste | Revisão de segurança dedicada | TRANSCRICAO | [09:46] Sofia |
| PRD-TST-04 | docs/PRD.md | Estratégia de teste | Testes ponta a ponta da integração no order.service | TRANSCRICAO | [09:46] Larissa |
