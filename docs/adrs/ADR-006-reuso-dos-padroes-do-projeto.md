# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa, Bruno
- **Consultados:** Diego, Sofia

## Status

Aceito

## Contexto

O OMS tem convenções consistentes que qualquer módulo novo deveria seguir ([09:27] Bruno):

- **Módulos por domínio** em `src/modules/<dominio>/` com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts` (exemplo: `src/modules/orders/`, `src/modules/customers/`).
- **Erros de domínio** que estendem `AppError` (`src/shared/errors/app-error.ts`) com `statusCode` e `errorCode` em CONSTANT_CASE. Exemplos: `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) e `InsufficientStockError` (`INSUFFICIENT_STOCK`) em `src/shared/errors/http-errors.ts` ([09:28] Bruno).
- **Tratamento centralizado** em `src/middlewares/error.middleware.ts`, que converte `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002`, `P2025`) para o formato `{ error: { code, message, details? } }` ([09:29] Bruno).
- **Validação** com Zod via `validate({ body, query, params })` (`src/middlewares/validate.middleware.ts`).
- **Logger Pino** único (`src/shared/logger/index.ts`) com `redact` de campos sensíveis ([09:29] Bruno).
- **Autenticação/autorização** com `authenticate` e `requireRole(...)` (`src/middlewares/auth.middleware.ts`). O `requireRole('ADMIN')` já é usado em `src/modules/users/user.routes.ts`.
- **Composição manual** de dependências em `buildControllers` (`src/app.ts`) e montagem de rotas em `buildApiRouter` (`src/routes/index.ts`), sob o prefixo `/api/v1`.
- **Identificadores UUID** (`@default(uuid()) @db.Char(36)`) em todos os models de `prisma/schema.prisma` ([09:51] Larissa).

## Decisão

O módulo de webhooks é "um módulo igual aos outros" ([09:30] Larissa):

1. Pasta `src/modules/webhooks/` com controller, service, repository, routes e schemas ([09:27] Bruno). A lógica do worker vai em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) e o entry-point em `src/worker.ts` ([09:28] Bruno).
2. Erros novos estendem `AppError` (`src/shared/errors/app-error.ts`) ou suas subclasses de `src/shared/errors/http-errors.ts` e usam **códigos com prefixo `WEBHOOK_`**, como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa).
3. **Sem mudanças no `error.middleware.ts`**: os erros novos são `AppError` e já são tratados ([09:29] Bruno).
4. **Pino como único logger**, sem nenhuma lib nova de log ([09:29] Bruno).
5. Schemas Zod para todos os inputs, incluindo a validação de URL `https` ([09:23] Sofia).
6. Replay de DLQ protegido com o `requireRole('ADMIN')` existente ([09:36] Larissa). O resto do CRUD fica só com `authenticate` ([09:37] Sofia).
7. O worker cria seu próprio `PrismaClient` com a mesma `DATABASE_URL`, reaproveitando a fábrica `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno).
8. IDs UUID em todas as tabelas novas, incluindo a outbox ([09:51] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Estrutura própria para o módulo de webhooks** (por exemplo, um pacote ou serviço à parte, com tratamento de erro e logger próprios) | Aumenta a carga cognitiva e a manutenção. A reunião decidiu explicitamente por "reuso máximo do que já existe" ([09:30] Larissa). |
| **Injetar um repositório de webhooks no `OrderService`** | Preferiu-se uma função `publishWebhookEvent(tx, …)` que recebe o client da transação: "não precisa injetar repository inteiro" ([09:41] Diego). |
| **ID auto-incremental na outbox** | Pergunta de Diego em [09:51]. Foi decidido UUID para manter o padrão do projeto ([09:51] Larissa). |

## Consequências

**Positivas**
- Curva de aprendizado mínima: o módulo é previsível para quem já conhece `orders` ou `customers`.
- As respostas de erro dos endpoints de webhook seguem o mesmo formato da API inteira, sem código novo de tratamento.
- Logs da API e do worker com o mesmo formato e a mesma política de redaction.
- Os testes podem seguir o padrão de `tests/orders.test.ts` e `tests/helpers/factories.ts`.

**Negativas**
- O módulo herda as limitações dos padrões atuais. Por exemplo, não existe vínculo `User`→`Customer` no `prisma/schema.prisma`, então o `authenticate` não restringe um usuário aos webhooks "do seu" customer (aceito por ora, [09:37] Sofia).
- O worker não é um processo HTTP: as partes do padrão ligadas ao Express (middlewares, `requestLogger`) não se aplicam a ele, e o logging e o tratamento de erro do loop precisam ser feitos à mão.
- O `redact` do logger precisa ser estendido para a secret de webhook, uma mudança pequena num arquivo compartilhado.

**Trade-off explícito:** priorizamos consistência e velocidade de entrega (3 sprints, [09:46] Larissa) em vez de liberdade de desenho específica para o módulo.

## Referências
- Transcrição: [09:23], [09:27]–[09:30], [09:36]–[09:37], [09:41], [09:48], [09:51]
- Código: `src/modules/orders/`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/modules/users/user.routes.ts`, `src/shared/logger/index.ts`, `src/config/database.ts`, `src/app.ts`, `src/routes/index.ts`, `prisma/schema.prisma`
