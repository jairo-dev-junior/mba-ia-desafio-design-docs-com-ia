# ADR-002 — Worker separado com polling

- **Status:** Aceito
- **Data da decisão:** reunião técnica registrada em `TRANSCRICAO.md`

## Contexto

Os eventos persistidos precisam ser entregues sem acoplar o processamento ao ciclo de vida da API. O requisito percebido pelos clientes é receber a notificação em menos de 10 segundos. O MySQL não oferece mecanismo equivalente a `LISTEN/NOTIFY` para acordar um processo externo.

## Decisão

**[ADR-002]** Executar um único worker Node.js em processo separado da API, com novo entry point próprio, sua própria instância de `PrismaClient` e a mesma `DATABASE_URL`. O worker consulta, a cada 2 segundos, um lote pequeno dos eventos elegíveis mais antigos, processa-os e atualiza seu estado.

Enquanto houver um único worker, eventos são lidos por `created_at`, preservando ordem por pedido de forma implícita. Isso não constitui garantia de ordenação global nem permanece válido ao escalar para múltiplos workers.

## Alternativas consideradas

- **Worker dentro do processo da API:** descartado por acoplar entrega e reinícios/deploys da API.
- **Trigger MySQL para notificar o worker:** descartado porque triggers executam SQL, mas não notificam processos externos sem uma improvisação adicional.
- **Vários workers desde a primeira versão:** adiado; exigiria particionamento por `order_id` ou locking e a escala atual não justificou essa complexidade.

## Consequências

### Positivas

- Polling de 2 segundos cabe na meta de latência de 10 segundos em condições normais.
- Falhas e reinícios da API e do worker ficam isolados.
- Reutiliza runtime, banco e fábrica do Prisma já adotados.

### Negativas e trade-offs

- Polling gera consultas mesmo quando não há eventos.
- Um único worker limita throughput e é ponto único de processamento.
- A ordenação por pedido é uma limitação operacional, não uma garantia ao escalar.

## Referências

- Transcrição: [09:08]–[09:13], [09:48].
- Código: `src/server.ts`, `src/config/database.ts`.
