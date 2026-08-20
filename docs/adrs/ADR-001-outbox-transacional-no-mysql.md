# ADR-001 — Outbox transacional no MySQL

- **Status:** Aceito
- **Data da decisão:** reunião técnica registrada em `TRANSCRICAO.md`

## Contexto

A mudança de status de um pedido já ocorre em uma transação Prisma que altera `orders`, grava `order_status_history` e, em algumas transições, atualiza estoque. Fazer a chamada HTTP externa nesse caminho aumentaria a latência e faria a disponibilidade do cliente interferir na mudança de status. Também não se pode confirmar o status e perder seu evento correspondente.

## Decisão

**[ADR-001]** Persistir um snapshot do evento em uma tabela de outbox no MySQL, na mesma transação de `OrderService.changeStatus`. A função proposta `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `Prisma.TransactionClient`; se a gravação falhar, toda a mudança de status sofre rollback. Só se criam eventos para endpoints ativos do cliente cujo filtro inclua o novo status.

O evento e seu `event_id` usam UUID. A consulta do worker deve ser apoiada por índices de estado e data de criação.

## Alternativas consideradas

- **HTTP síncrono dentro de `changeStatus`:** descartado porque endpoint lento ou indisponível bloquearia a operação e não há motivo para reverter uma mudança válida por falha externa.
- **Redis Streams/cluster:** descartado porque acrescentaria infraestrutura e operação desnecessárias para a escala inicial; o MySQL já existente atende.
- **Gravar o evento depois do commit:** descartado porque abre uma janela em que o pedido muda sem o evento ser registrado.

## Consequências

### Positivas

- Atomicidade entre mudança de status e criação do evento.
- Nenhuma dependência de broker adicional na primeira versão.
- O snapshot preserva o estado observado no instante da transição.

### Negativas e trade-offs

- A outbox adiciona escrita e índices à transação de pedidos.
- O MySQL passa a acumular dados operacionais; arquivamento de entregues foi adiado e exigirá trabalho posterior.
- É necessário controlar concorrência e recuperar eventos que fiquem em processamento após falha do worker.

## Referências

- Transcrição: [09:03]–[09:08], [09:33]–[09:34], [09:40]–[09:41], [09:51]–[09:52].
- Código: `src/modules/orders/order.service.ts`, `prisma/schema.prisma`.
