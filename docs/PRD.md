# PRD — Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião técnica, aguardando implementação |
| **Data** | 2026-10-24 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS vai avisar automaticamente os clientes B2B, por **webhook**, sempre que o status de um pedido deles mudar (por exemplo, de `PROCESSING` para `SHIPPED`). Cada cliente cadastra uma ou mais URLs, escolhe quais status quer receber e passa a ser notificado em segundos, com uma assinatura que comprova que a mensagem veio da plataforma. Hoje a plataforma não tem nenhum mecanismo de notificação externa: a única forma de saber de uma mudança é consultar a API.

## 2. Problema e motivação

- Três clientes B2B (**Atlas Comercial, MaxDistribuição e Nova Cargo**) fizeram um pedido formal para receber as mudanças de status em tempo real ([09:00] Marcos).
- Hoje eles fazem polling periódico em `GET /orders`. A integração fica **lenta e cara** para eles ([09:00] Marcos).
- **Risco comercial:** a Atlas indicou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos). Ela **pediu a entrega para o fim de novembro** ([09:45] Marcos), e o Marcos vai confirmar esse prazo com o cliente ([09:47] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Necessidade |
|---|---|
| **Integrador do cliente B2B** (equipe técnica de Atlas, MaxDistribuição, Nova Cargo) | Receber as mudanças de status sem polling, validar a autenticidade e consultar o histórico de entregas |
| **Usuário da plataforma que representa o cliente** (autenticado com JWT do OMS) | Cadastrar e manter os webhooks do cliente pela API ([09:32] Marcos) |
| **Administrador da plataforma** (role `ADMIN`) | Reprocessar eventos que falharam definitivamente ([09:36] Sofia) |

**Cenários**
1. **Acompanhar envio:** um cliente cadastra um webhook só para `SHIPPED` e `DELIVERED` e recebe uma chamada segundos depois de cada uma dessas mudanças ([09:33] Marcos).
2. **Cliente em manutenção:** o endpoint de um cliente fica 2 horas fora do ar em manutenção planejada. As notificações são retentadas automaticamente e entregues quando ele volta ([09:16] Diego).
3. **Diagnóstico:** o integrador de um cliente consulta as últimas entregas (sucesso ou falha, resposta, tempo) para entender por que um evento não foi processado ([09:34] Marcos).
4. **Recuperação:** depois de uma indisponibilidade longa (mais de ~15h), um administrador reenvia um evento que foi para a fila de falhas ([09:18] Diego).
5. **Troca de credencial:** o cliente suspeita que a secret vazou, pede uma nova e tem 24h para atualizar os sistemas dele ([09:21] Sofia, [09:22] Diego).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| OBJ-01 | Notificação percebida como "tempo real" | Tempo entre a mudança de status e a primeira tentativa de entrega | **< 10 segundos** (desenho: ~2s de polling) | [09:02] Marcos, [09:10] Larissa |
| OBJ-02 | Nenhuma mudança de status sem notificação | % de mudanças de status assinadas com evento registrado | **100%** (garantido pela transação) | [09:40] Bruno |
| OBJ-03 | Tolerar indisponibilidades do cliente sem perder eventos | Janela coberta por retentativas automáticas antes de exigir intervenção | **~15 horas** (1m/5m/30m/2h/12h) | [09:17] Diego |
| OBJ-04 | Atender o prazo pedido pela Atlas e reduzir o risco de churn | Data de entrada em produção | Até o **fim de novembro** (prazo pedido, a confirmar com o cliente) | [09:00], [09:45], [09:47] Marcos |
| OBJ-05 | Entregar no prazo combinado | Sprints até produção | **3 sprints**, incluindo ≥ 2 dias úteis de revisão de segurança | [09:46] Larissa, [09:46] Sofia |

Observação: a redução do polling em `GET /orders` e a adesão dos três clientes são efeitos esperados, mas a reunião não definiu meta numérica para eles. Por isso não aparecem como metas aqui.

## 5. Escopo

### 5.1 Incluso
- Cadastro, edição, listagem e remoção de webhooks por cliente, com filtro de status.
- Notificação automática a cada mudança de status de pedido.
- Assinatura de cada envio, secret por webhook e rotação de secret.
- Retentativas automáticas e fila de falhas com reprocessamento manual por administrador.
- Histórico de entregas consultável pelo cliente.

### 5.2 Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| **Aviso por e-mail ao cliente quando o webhook falha repetidamente** | Adiado para a próxima fase, depois de medir o impacto | [09:37] Marcos, [09:37] Larissa |
| **Dashboard visual para o cliente gerenciar webhooks** | Descartado nesta fase. É projeto separado do time de frontend. Nesta fase, só API. | [09:39] Marcos, [09:40] Larissa |
| **Rate limiting de envio por cliente** | Não entra. Vamos observar e decidir depois. | [09:38] Diego, [09:39] Larissa |
| **Webhooks de entrada** (cliente enviando para a plataforma) | Descartado. O fluxo é só de saída. | [09:02] Marcos |
| **Ordem global de eventos** e múltiplos workers | Não garantido. A ordem vale só por pedido. | [09:13] Larissa, [09:14] Marcos |
| **Arquivamento de eventos antigos** (~30 dias) | Fora desta feature | [09:08] Diego |
| **Restringir o cadastro de webhooks por perfil** | Por ora, qualquer usuário autenticado. Endurecer no futuro. | [09:37] Sofia |

## 6. Requisitos funcionais

| ID | Requisito | Prioridade | Origem |
|---|---|---|---|
| FR-01 | O usuário autenticado cadastra um webhook informando o cliente, a URL de destino e os status que quer receber. A **secret é gerada pela plataforma e só é exibida na criação** (e na rotação, FR-10). | Must | [09:31] Marcos |
| FR-02 | O cliente dono do webhook é informado explicitamente na requisição (body ou path), **não deduzido do JWT**, porque o JWT é do usuário operador | Must | [09:32] Bruno, [09:32] Larissa |
| FR-03 | O usuário edita um webhook existente (URL, status assinados, ativo/inativo) | Must | [09:33] Bruno |
| FR-04 | O usuário remove um webhook | Must | [09:33] Bruno |
| FR-05 | O usuário lista os webhooks de um cliente | Must | [09:33] Bruno |
| FR-06 | Cada webhook define **quais status** deseja receber. Mudanças para status não assinados não geram notificação. | Must | [09:33] Marcos, [09:34] Bruno |
| FR-07 | Toda mudança de status de pedido gera uma notificação para cada webhook ativo do cliente que assina aquele status | Must | [09:00] Marcos, [09:40] Bruno |
| FR-08 | A notificação traz a identificação do evento, o tipo, a data/hora, o pedido (id e número), o status anterior e o novo, o cliente e o valor total. **Não traz os itens** (detalhes via `GET /orders/:id`). | Must | [09:43] Diego |
| FR-09 | Cada notificação é **assinada** (HMAC-SHA256) com a secret do webhook e identifica o evento e o webhook em headers | Must | [09:20] Sofia, [09:44] Diego, [09:44] Sofia |
| FR-10 | O usuário pode **rotacionar a secret** pela API. A secret anterior continua válida por **24 horas**. | Must | [09:21] Sofia |
| FR-11 | Entregas que falham são **retentadas automaticamente**, com teto de 5 tentativas e backoff de 1m, 5m, 30m, 2h e 12h, e, ao esgotar, vão para uma **fila de falhas**. Se o envio inicial entra ou não nessa conta está em aberto no [RFC Q-06](RFC.md#5-questões-em-aberto). | Must | [09:17] Larissa, [09:18] Diego |
| FR-12 | O cliente consulta o **histórico de entregas** de um webhook (últimas 100): sucesso/falha, payload, resposta e tempo de resposta | Must | [09:34] Marcos |
| FR-13 | Um **administrador** (role ADMIN) pode reenviar um evento da fila de falhas. A ação fica registrada com quem a executou. | Must | [09:18] Diego, [09:36] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| NFR-01 | **Latência:** primeira tentativa de entrega em menos de 10s após a mudança de status (polling de 2s) | [09:02] Marcos, [09:09] Diego |
| NFR-02 | **Consistência:** a mudança de status e o registro da notificação são atômicos. Se um falha, o outro não acontece. | [09:40] Bruno, [09:41] Diego |
| NFR-03 | **Isolamento:** cliente lento ou fora do ar nunca atrasa nem impede uma mudança de status | [09:04] Bruno |
| NFR-04 | **Garantia de entrega at-least-once:** o cliente pode receber duplicatas e deduplica pelo identificador do evento (`X-Event-Id`) | [09:24]–[09:25] Diego |
| NFR-05 | **Timeout** de 10s por tentativa de entrega | [09:42] Diego |
| NFR-06 | **Segurança de transporte:** só URLs `https`. Cadastro com `http` é recusado com erro de validação. | [09:23] Sofia |
| NFR-07 | **Tamanho máximo do payload:** 64KB. Acima disso o evento não é enviado (erro, sem truncar). | [09:23] Sofia, [09:24] Larissa |
| NFR-08 | **Segredo por webhook:** cada webhook tem uma secret única, nunca uma secret global | [09:21] Sofia |
| NFR-09 | **Ordem:** os eventos de um mesmo pedido chegam em ordem enquanto houver um único processador. Não há garantia de ordem global. | [09:12] Diego, [09:13] Larissa |
| NFR-10 | **Resiliência operacional:** o processamento roda separado da API. Reinícios da API não interrompem as entregas. | [09:11] Diego |
| NFR-11 | **Consistência com a plataforma:** mesmos padrões de erro, log e validação da API atual. Códigos de erro com prefixo `WEBHOOK_`. | [09:29] Larissa, [09:30] Larissa |
| NFR-12 | **Auditoria:** o reprocessamento manual registra o administrador responsável | [09:36] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Registrar a notificação no próprio banco, junto da mudança de status (outbox) | Consistência sem infraestrutura nova, em troca de um pequeno custo na transação | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Processador separado verificando a cada 2s | Simplicidade em troca de até 2s de atraso e escala limitada a um processador | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| 5 retentativas em ~15h e depois fila de falhas com reenvio manual | Janela longa de recuperação em troca de possível atraso e intervenção humana | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| Assinatura HMAC com secret por webhook e rotação de 24h | Segurança e raio de impacto contido em troca de esforço de verificação no cliente | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega "pelo menos uma vez" | Simplicidade em troca de o cliente precisar tratar duplicatas | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Seguir os padrões atuais da plataforma | Velocidade e consistência em troca de herdar as limitações atuais (por exemplo, permissões amplas) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Notificação enxuta e "congelada" no momento da mudança | Payload pequeno e fiel, em troca de o cliente buscar detalhes à parte | [ADR-007](adrs/ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Origem |
|---|---|---|
| Fluxo atual de mudança de status de pedidos (`src/modules/orders/order.service.ts`) | Técnica, interna | [09:40] Bruno |
| Banco MySQL existente (nenhuma infraestrutura nova) | Técnica | [09:07] Diego |
| Novo processo de entrega para implantar e operar | Operação | [09:11] Larissa |
| Revisão de segurança da Sofia (≥ 2 dias úteis antes do deploy) | Processo | [09:46] Sofia |
| Documentação no portal do desenvolvedor (assinatura, deduplicação, integração via API) | Produto/comunicação | [09:26] Marcos, [09:40] Marcos |
| Confirmação de prazo com a Atlas | Comercial | [09:47] Marcos |

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| RSK-01 | **Perda da Atlas** por atraso em relação ao fim de novembro | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting fora), estimativa de 3 sprints, Marcos confirma o prazo com o cliente ([09:46]–[09:47]) |
| RSK-02 | **Cliente processa a mesma notificação duas vezes** por não deduplicar | Média | Médio | Documentação destacada no portal sobre `X-Event-Id` ([09:26] Marcos) |
| RSK-03 | **Vazamento de secret** de um cliente | Baixa | Alto | Secret por webhook, rotação com 24h de sobreposição, nunca logar secrets, revisão de segurança ([09:21]–[09:22], [09:46]) |
| RSK-04 | **Cliente sobrecarregado** por rajadas de notificações (por exemplo, 50 em 1 minuto) | Média | Médio | Fora do escopo agora. Monitorar volume por cliente e decidir sobre rate limiting ([09:38]–[09:39]) |
| RSK-05 | **Cliente não percebe que o webhook está falhando** (sem aviso por e-mail nesta fase) | Média | Médio | Histórico de entregas consultável (FR-12). Admin acompanha a fila de falhas. E-mail na próxima fase ([09:37]) |
| RSK-06 | **Processador de entregas parado** atrasa todas as notificações | Baixa | Alto | Eventos não se perdem (ficam pendentes). Monitoramento de fila descrito no [FDD §9](FDD.md#9-observabilidade) |

## 11. Critérios de aceitação

1. **Dado** um webhook ativo de um cliente assinando `SHIPPED`, **quando** um pedido desse cliente muda para `SHIPPED`, **então** o endpoint recebe uma notificação assinada em menos de 10 segundos.
2. **Dado** um webhook assinando apenas `DELIVERED`, **quando** o pedido muda para `PAID`, **então** nenhuma notificação é enviada.
3. **Dado** que a mudança de status falha (por exemplo, estoque insuficiente), **então** nenhuma notificação é gerada. **Dado** que o registro da notificação falha, **então** o status não muda.
4. **Dado** um endpoint fora do ar, **quando** ocorre uma mudança de status, **então** o evento é retentado seguindo a progressão 1m, 5m, 30m, 2h e 12h, até o teto de tentativas (RFC Q-06). Persistindo a falha, o evento vai para a fila de falhas.
5. **Dado** um evento na fila de falhas, **quando** um ADMIN solicita o reenvio, **então** o evento é reenviado com o **mesmo** identificador e a ação fica registrada. Um usuário OPERATOR recebe acesso negado.
6. **Dado** um cadastro com URL `http://`, **então** a API recusa com erro de validação.
7. **Dado** um webhook criado, **então** a secret aparece apenas na resposta de criação (ou de rotação). Depois de rotacionar, a antiga deixa de valer em 24h.
8. **Dado** um webhook com entregas, **quando** o cliente consulta o histórico, **então** vê as 100 mais recentes, com resultado, resposta, tempo e payload.
9. **Dado** que a API é reiniciada, **então** as entregas pendentes continuam sendo processadas.

## 12. Estratégia de testes e validação

| Nível | O que valida | Como |
|---|---|---|
| Unitário | Assinatura HMAC, cálculo do backoff, montagem do payload, limite de 64KB, filtro de status | Vitest (`vitest.config.ts` existente) |
| Integração (API) | CRUD de webhooks, validação de https, secret só na criação, replay só ADMIN, histórico | Supertest + factories, no padrão de `tests/orders.test.ts` e `tests/helpers/factories.ts` |
| Integração (transação) | Mudança de status ⇔ evento na outbox, inclusive rollback | Testes sobre `PATCH /orders/:id/status` com MySQL de teste |
| Ponta a ponta | Integração no `order.service` e worker entregando a um endpoint de teste: sucesso, 5xx, timeout, DLQ e replay ([09:46] Larissa) | Endpoint HTTP local de teste + relógio controlado para o backoff |
| Segurança | Geração e armazenamento de secret, assinatura, ausência de secret em logs | Revisão da Sofia, ≥ 2 dias úteis antes do deploy ([09:46]) |

Os critérios técnicos detalhados estão no [FDD §12](FDD.md#12-critérios-de-aceite-técnicos).
