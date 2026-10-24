# Da Reunião ao Documento — Webhooks de Notificação de Pedidos

> Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)

## Sobre o desafio

A tarefa era transformar a transcrição de uma reunião técnica de ~55 minutos (`TRANSCRICAO.md`) num pacote de design docs acionável para o time que vai construir o **Sistema de Webhooks de Notificação de Pedidos** de um OMS em Node.js/TypeScript/Prisma/MySQL. Na reunião, tech lead, PM, dois engenheiros e a engenheira de segurança fecharam a arquitetura (outbox, worker, retry/DLQ, HMAC, at-least-once). Nada disso tinha sido registrado.

A parte difícil não é escrever, é **filtrar**. A transcrição mistura decisões fechadas, requisitos, itens descartados (e-mail, dashboard), itens adiados (rate limiting, arquivamento), detalhes secundários e até ambiguidades (o que exatamente significa "5 tentativas"?). Cada documento do pacote opera numa altura diferente (PRD, RFC, ADRs, FDD), e tudo precisa ser rastreável a um timestamp da reunião ou a um arquivo real do código, sem nenhuma alteração em `src/`, `prisma/` ou `tests/`.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (CLI/VS Code, modelo Claude Opus 5.5) em **plan mode** | Leitura integral da transcrição e do código, construção do "mapa de fatos" com timestamps, plano de produção aprovado antes de escrever qualquer documento |
| **Claude Code**, sessão principal | Redação dos ADRs, RFC, FDD, PRD, Tracker e deste README |
| **Subagente revisor do Claude Code** (contexto isolado, sem acesso às minhas conclusões) | Revisão adversarial de rastreabilidade: conferiu cada citação `[hh:mm] Nome` contra a transcrição e cada afirmação sobre o código contra os arquivos |
| **Scripts Python ad hoc** (gerados e executados pelo Claude Code) | Verificação mecânica: timestamps e falantes existentes, caminhos de arquivo reais, percentuais do tracker, links entre documentos, nomes dos ADRs |

## Workflow adotado

1. **Contextualização em plan mode (só leitura).** A IA leu a transcrição inteira e todos os arquivos relevantes do código: `order.service.ts`, `order.status.ts`, erros, middlewares, `schema.prisma`, `app.ts`, `routes/index.ts`, `server.ts`, `config/`, logger e testes. Em vez de "resumir a reunião", pedi um **mapa de fatos classificado**: decisões fechadas, requisitos funcionais, descartados/adiados, alternativas rejeitadas, questões em aberto e ganchos no código, cada item com o timestamp e o falante. Esse mapa virou a fonte única de verdade para todos os documentos e foi aprovado como plano antes da escrita.
2. **ADRs primeiro (7).** As seis decisões principais e uma secundária (payload/snapshot/filtro), no formato MADR, com alternativas e trade-off explícito. Os ADRs formam o esqueleto do resto.
3. **RFC.** Consolida a proposta em nível de arquitetura, com as alternativas descartadas na reunião (cada uma com o trade-off), as questões em aberto e links para os ADRs. Regra imposta: nada de schema, payload ou endpoint detalhado no RFC.
4. **FDD.** Desce à implementação: modelo de dados, máquina de estados da outbox, fluxos com diagramas de sequência, 7 endpoints com request/response/status, contrato de saída, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção obrigatória de integração com 16 arquivos reais. O que a reunião não decidiu ficou marcado como **(proposta de implementação)** e consolidado na §14 do FDD.
5. **PRD por último** entre os grandes documentos: em linguagem de produto, remetendo aos ADRs e ao FDD em vez de repetir detalhes.
6. **Revisão adversarial independente** (subagente com contexto limpo) e correção dos achados.
7. **Tracker**, montado varrendo os documentos prontos e atualizado depois das correções.
8. **Verificação mecânica** com scripts e checklist dos critérios de aceite.

## Prompts customizados

**1. Prompt de abertura (plan mode).** Colei o enunciado completo do desafio com o Claude Code em plan mode, o que obriga a IA a explorar e propor um plano antes de escrever. O plano aprovado exigia este recorte da transcrição:

```text
Antes de escrever qualquer documento, monte um MAPA DE FATOS a partir da TRANSCRICAO.md
e do código, que será a fonte única para PRD, RFC, FDD, ADRs e Tracker. Classifique cada item em:
- Decisões fechadas (quem fechou e quando)
- Requisitos funcionais explícitos
- Fora de escopo: separe DESCARTADO de ADIADO
- Alternativas rejeitadas, com o motivo do descarte dito na reunião
- Questões em aberto / ambiguidades (inclusive contradições internas da própria reunião)
- Ganchos no código: arquivo real + o que muda nele
Cada item precisa de [hh:mm] Nome ou de um caminho de arquivo existente.
Se não houver origem, o item não entra. Se for um detalhe necessário para implementar
que a reunião não decidiu, marque como "(proposta de implementação)".
```

**2. Prompt do revisor adversarial** (subagente com contexto isolado, sem receber as minhas conclusões):

```text
Você é um revisor adversarial de documentação técnica.
Fontes de verdade: TRANSCRICAO.md (formato [hh:mm] Nome: fala) e o código em src/, prisma/,
tests/, package.json. Documentos sob revisão: docs/PRD.md, docs/RFC.md, docs/FDD.md,
docs/adrs/ADR-*.md.
Regras: toda informação precisa ser rastreável; nada de requisito/decisão inventado;
descartados/adiados NÃO podem aparecer como requisito; nenhum arquivo existente citado com
caminho errado. O RFC é nível arquitetura e não repete detalhe do FDD.
Procure com ceticismo:
1. Citações [hh:mm] Nome que não batem (timestamp, falante, ou a fala não diz aquilo).
2. Afirmações apresentadas como decididas que não foram (propostas marcadas são permitidas).
3. Itens descartados/adiados aparecendo como requisito.
4. Afirmações erradas sobre o código (verifique lendo o código).
5. Contradições entre documentos (números, endpoints, códigos de erro, tentativas).
6. Duplicação excessiva entre RFC e FDD.
Não edite nada. Liste achados concretos: arquivo, trecho, problema, evidência, correção.
```

## Iterações e ajustes

Foram **cinco iterações principais**: (1) plano e mapa de fatos; (2) primeira versão completa; (3) correções da revisão adversarial; (4) correções da verificação mecânica e da checklist estrutural; (5) auditoria final contra o enunciado completo, item a item. Os ajustes concretos mais relevantes:

1. **Ambiguidade "5 tentativas" (detectada já no planejamento).** Diego propôs 5 intervalos (1m/5m/30m/2h/12h ≈ 14,6h, "quase 15 horas") e Larissa resumiu como "total 5 tentativas". As duas coisas só fecham com 1 envio + 5 retentativas. Em vez de escolher em silêncio, registrei a interpretação no ADR-003, a pergunta no RFC (Q-06) e constantes ajustáveis no FDD. Na primeira versão, o PRD ainda tinha resolvido a dúvida por conta própria ("5 retentativas") e o FDD sugeria um "plano B" que eliminava o intervalo de 12h, contradizendo a decisão. A revisão adversarial pegou os dois, e foram corrigidos.
2. **`customer_id` fora do JWT.** Marcos começou dizendo "customer_id implícito do JWT", mas Bruno e Larissa corrigiram em seguida ([09:32]: o JWT é do operador). A primeira versão do FDD colocava o `customerId` na *query* da listagem, uma terceira opção que ninguém citou na reunião ("body ou path"). Mudei a listagem para `GET /customers/:customerId/webhooks` e documentei como montar a rota sem conflitar com o router de `/customers` existente.
3. **Bug de modelagem na DLQ.** A revisão apontou que `webhook_dead_letter.outboxId @unique`, somado ao replay reaproveitando o mesmo evento, faria um replay que falha de novo quebrar com violação de unicidade (P2002). Tirei a unicidade (N dead letters por evento, histórico por ciclo) e ajustei o ADR-003 para explicitar o modelo híbrido: status "falhou" na outbox e evidência na DLQ.
4. **Meta inventada no PRD.** A primeira versão tinha "3 de 3 clientes integrados até fim de novembro" e tratava o prazo como "combinado". A reunião só diz que a Atlas *quer* para fim de novembro e que Marcos *vai confirmar*. A meta foi reescrita como "em produção até fim de novembro (prazo pedido, a confirmar)", e a validação com clientes, que não foi discutida, saiu da estratégia de testes.
5. **Observabilidade sem stack inventada.** O enunciado pede métricas, logs e tracing, mas o projeto só tem Pino e o `requestId`, e a reunião decidiu "não botar nada novo". Em vez de inventar Prometheus ou OpenTelemetry, o FDD define logs estruturados, métricas *derivadas* de logs e consultas SQL (com nomes prontos para uma stack futura) e tracing por correlação (`event_id`/`X-Event-Id`, `order_id`, `requestId` opcional).
6. **Leitura fina do código, que mudou o desenho:**
   - `validate.middleware.ts` converte qualquer `ZodError` em `VALIDATION_ERROR`, então a URL http responde `VALIDATION_ERROR`, e `WEBHOOK_INVALID_URL` ficou para a defesa em profundidade no worker.
   - `NotFoundError` tem código fixo, então os 404 `WEBHOOK_*` estendem `AppError`.
   - `error.middleware.ts` não loga `AppError`, então é preciso um log explícito antes de `WEBHOOK_PUBLISH_FAILED`.
   - `env.ts` exige `JWT_SECRET`, então o worker precisa do mesmo `.env`.
   - `WEBHOOK_SECRET_REQUIRED`, citado por Bruno como exemplo, nunca dispara num desenho em que a secret é gerada pela plataforma e é NOT NULL. Ficou documentado como código reservado em vez de um cenário fictício.
7. **Citações esticadas e falsas alternativas.** A revisão encontrou:
   - alternativas no ADR-004 que não foram discutidas (agora marcadas como "plausível, não discutida");
   - "zero dependências novas" atribuído a uma fala do Bruno que só tratava do logger (recitado para [09:30] Larissa);
   - o argumento de atomicidade contra o Redis, que é análise minha e não fala da reunião (agora rotulado);
   - o limite de 64KB como decisão de ADR, quando Larissa disse explicitamente que é NFR (movido).
8. **Verificação mecânica.** O script achou duas faixas de timestamp cujo falante não falava no fim da faixa (uma faixa atribuída a Diego terminando às 09:26 e outra a Sofia terminando às 09:24), que foram corrigidas. Confirmou também que todos os caminhos citados existem, exceto os arquivos novos planejados (`src/worker.ts`, `src/modules/webhooks/*`). A checklist estrutural levou a tornar "Status" uma seção própria nos ADRs e "Metadados" uma seção explícita no RFC.
9. **Duplicação entre documentos.** Os riscos do RFC foram reduzidos aos arquiteturais, com links para os riscos de produto (PRD) e de implementação (FDD). O cálculo das ~15h ficou concentrado no ADR-003. O arquivamento de 30 dias saiu das "questões em aberto" do RFC, porque é fora de escopo, não questão aberta.
10. **Auditoria final contra o enunciado.** Pedi à IA, numa sessão nova, que conferisse o pacote contra cada critério de aceite e contra o código. Ela encontrou quatro ajustes:
    - o FDD listava "sem secret" entre as falhas que vão direto para a DLQ, contradizendo a própria nota de que `WEBHOOK_SECRET_REQUIRED` nunca dispara;
    - o FDD expunha `MAX_ATTEMPTS = 6`, que à primeira vista contradiz o "total 5 tentativas" de [09:48]. O desenho não mudou, mas a constante passou a ser `WEBHOOK_MAX_RETRIES = 5`, no mesmo vocabulário da reunião;
    - o tracker estava dividido em quatro tabelas, e o enunciado pede uma. Virou uma tabela única;
    - cenários e critérios do PRD atribuíam a clientes específicos (Atlas, Nova Cargo, MaxDistribuição) exemplos que a reunião deu de forma genérica. Passaram a falar em "um cliente".

## Como navegar a entrega

| Ordem | Arquivo | O que é |
|---|---|---|
| 1 | [docs/PRD.md](docs/PRD.md) | Por que e o quê: problema, público, métricas, escopo, requisitos |
| 2 | [docs/RFC.md](docs/RFC.md) | Proposta técnica para revisão: abordagem, alternativas descartadas, questões em aberto |
| 3 | [docs/adrs/](docs/adrs/) | Uma decisão por arquivo (ADR-001 a ADR-007) |
| 4 | [docs/FDD.md](docs/FDD.md) | Como construir: dados, fluxos, contratos, erros, resiliência, observabilidade, integração com o código |
| 5 | [docs/TRACKER.md](docs/TRACKER.md) | Origem de cada item (transcrição ou código) |

ADRs:
- [ADR-001 — Outbox transacional no MySQL](docs/adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado com polling de 2s](docs/adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Payload enxuto, snapshot e filtro na inserção](docs/adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md)

Quem vai implementar pode ir direto ao FDD §10 (integração) e §14 (propostas a validar), consultando o RFC §5 para o que ainda depende de decisão do time.
