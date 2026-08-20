# ADR-003 — Retry com backoff e DLQ

- **Status:** Aceito, com uma clarificação pendente sobre a contagem
- **Data da decisão:** reunião técnica registrada em `TRANSCRICAO.md`

## Contexto

Endpoints de clientes podem ficar temporariamente indisponíveis. Três tentativas foram consideradas insuficientes para manutenções de até duas horas; retry indefinido deixaria eventos pendurados para sempre. Falhas permanentes precisam permanecer disponíveis para diagnóstico e replay.

## Decisão

**[ADR-003]** Aplicar backoff de `1 min / 5 min / 30 min / 2 h / 12 h`. Esgotada a política, copiar o snapshot, motivo e timestamp para uma tabela `webhook_dead_letter` separada. O replay é manual por `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, restrito a `ADMIN` e auditado; ele cria uma nova entrada pendente na outbox sem apagar a evidência original.

A reunião usa ao mesmo tempo “5 tentativas” e cinco intervalos de retry. Para não ocultar essa inconsistência, a especificação adota provisoriamente **uma tentativa inicial mais até cinco retentativas**, única leitura que utiliza toda a progressão, e exige confirmação dos revisores antes da implementação.

## Alternativas consideradas

- **Retry indefinido:** descartado porque eventos de endpoints abandonados nunca terminariam.
- **Três tentativas:** descartado porque uma indisponibilidade de duas horas poderia exceder rapidamente essa janela.
- **Marcar apenas `failed` na outbox:** descartado em favor de DLQ separada, mantendo a leitura operacional da outbox limpa e a evidência de falha disponível.

## Consequências

### Positivas

- Tolera indisponibilidades transitórias por uma janela longa.
- Isola falhas permanentes e permite diagnóstico/replay controlado.
- Mantém trilha de quem acionou o replay.

### Negativas e trade-offs

- A entrega pode permanecer atrasada por cerca de 15 horas.
- DLQ e replay adicionam estados, persistência e operação manual.
- A ambiguidade “tentativas versus retentativas” precisa ser encerrada antes do código.

## Referências

- Transcrição: [09:14]–[09:19], [09:35]–[09:36], [09:48].
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`).
