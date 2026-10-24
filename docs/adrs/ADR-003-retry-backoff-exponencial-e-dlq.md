# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e Dead Letter Queue em tabela separada

- **Data:** reunião técnica de quinta-feira (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa, Diego, Bruno
- **Consultados:** Marcos, Sofia

## Status

Aceito

## Contexto

O endpoint do cliente pode estar lento, fora do ar ou em manutenção. Já houve cliente com duas horas de indisponibilidade por manutenção planejada ([09:16] Diego). O worker ([ADR-002](ADR-002-worker-separado-com-polling.md)) precisa de uma política que:

- dê tempo para o cliente se recuperar, sem perder o evento;
- não deixe eventos "pendurados para sempre" quando o cliente sumiu ([09:15] Diego);
- deixe evidência para debug e reprocessamento ([09:18] Diego).

O timeout por chamada HTTP é de **10 segundos**. Uma resposta acima disso conta como falha e o evento vai para retry ([09:42] Diego).

## Decisão

1. **Backoff exponencial com intervalos fixos de 1 min, 5 min, 30 min, 2 h e 12 h**, com teto de **5 tentativas** ([09:17] Diego, [09:17] Larissa).
   - *Interpretação registrada:* os cinco intervalos começam a contar na **primeira falha**. Diego estimou "quase 15 horas entre primeira falha e última tentativa" ([09:17]), e 1+5+30+120+720 min ≈ 14,6 h, o que só fecha com **1 envio inicial + 5 retentativas**. O resumo de [09:48] fala em "total 5 tentativas". A ambiguidade está listada como questão em aberto no [RFC](../RFC.md) e deve ser confirmada antes da implementação.
2. **Falha permanente vai para a DLQ.** Esgotadas as tentativas, o evento vai para a tabela separada **`webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp ([09:15] Diego, [09:18] Diego). Na outbox, a linha do evento passa para o status terminal "falhou", um dos quatro estados que Diego listou para a tabela ([09:08]). Assim ela sai do polling, que só lê pendentes, e o replay pode reaproveitá-la (item 3).
3. **Reprocessamento manual** pelo endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). Ele exige a role `ADMIN` e registra quem executou o replay, para auditoria ([09:36] Sofia, [09:36] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento pode ficar pendurado para sempre quando o cliente sumiu ([09:15] Diego). |
| **3 tentativas** (sugestão de Bruno, [09:16]) | Uma indisponibilidade de manhã seria esgotada em cerca de 30 minutos, e já houve manutenção planejada de 2h ([09:16] Diego). |
| **Só marcar como `failed` na própria outbox**, sem tabela separada | Pergunta feita por Larissa em [09:17]. A tabela separada deixa a leitura da outbox principal mais limpa e funciona como evidência para debug e reprocessamento ([09:18] Diego). O status "falhou" continua existindo na outbox como estado terminal, mas o motivo e a evidência ficam na DLQ. |

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15h sem intervenção. Marcos considera aceitável que um cliente fora por mais tempo que isso já tenha um "problema sério dele" ([09:17] Marcos).
- O polling lê só pendentes. A DLQ concentra, com o motivo da falha, só os casos que precisam de atenção humana.
- O replay é auditável e restrito a administradores.

**Negativas**
- Um evento pode chegar com até ~15h de atraso. Para o cliente, isso é preferível a perdê-lo, mas o evento pode chegar "velho" (o payload é um snapshot, ver [ADR-007](ADR-007-payload-snapshot-enxuto-filtrado-na-insercao.md)).
- O replay é manual: exige alguém olhando a DLQ. O aviso automático ao cliente (e-mail após falhas seguidas) ficou **fora do escopo desta fase** ([09:37] Larissa).
- Retentativas e replay podem gerar entregas duplicadas, o que é coerente com a garantia at-least-once ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).
- Um timeout de 10s por tentativa, com um único worker, faz um cliente lento consumir tempo do ciclo de polling.

**Trade-off explícito:** priorizamos uma janela longa de recuperação (~15h) e uma outbox limpa, aceitando atraso potencial e reprocessamento manual.

## Referências
- Transcrição: [09:14]–[09:19], [09:35]–[09:36], [09:42], [09:48]
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`), `src/shared/logger/index.ts` (log de auditoria do replay)
