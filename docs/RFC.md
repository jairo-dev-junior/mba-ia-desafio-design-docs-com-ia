# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| Autor | Equipe de Engenharia do OMS — consolidação da reunião técnica |
| Status | Proposto para revisão final |
| Data do documento | 2026-08-17 |
| Revisores | Larissa (Tech Lead), Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |

## Resumo executivo (TL;DR)

**[RFC-PROP-01]** Propomos webhooks outbound para mudanças de status de pedidos, desacoplados da API por uma outbox no MySQL gravada atomicamente com a mudança de status. Um único worker em processo separado consulta pendências a cada 2 segundos, entrega um snapshot JSON assinado com HMAC-SHA256 e aplica retry com backoff antes de enviar falhas permanentes para DLQ. A entrega é at-least-once, e o consumidor deduplica pelo `X-Event-Id`.

A solução reutiliza Prisma, MySQL, Zod, `AppError`, Pino, JWT/RBAC e a organização modular existentes. Evita broker adicional nesta fase e preserva a operação de pedidos quando um endpoint externo estiver lento ou fora do ar.

## Contexto e problema

**[RFC-CTX-01]** Atlas Comercial, MaxDistribuição e Nova Cargo hoje consultam `GET /orders` repetidamente. Eles pediram notificação de mudanças de status com atraso inferior a 10 segundos; a Atlas associou a entrega até o fim do trimestre ao risco de migração para concorrente.

O caminho atual de `OrderService.changeStatus` já executa em uma transação as regras de transição, estoque, atualização de `orders` e histórico. Inserir HTTP externo ali aumentaria a latência e faria falhas do cliente interferirem na operação válida do OMS. Ao mesmo tempo, persistir a notificação fora da transação permitiria status confirmado sem evento correspondente.

## Proposta técnica

### Visão geral

**[RFC-PROP-02]** Na transação da mudança de status, o OMS identifica endpoints ativos do cliente interessados no novo status e grava um evento por destino na outbox. O registro contém UUID, destino e snapshot já renderizado, preservando os dados do instante da mudança. Se a gravação falhar, toda a transação sofre rollback.

**[RFC-PROP-03]** Um entry point de worker separado usa sua própria instância de Prisma e o mesmo banco. Em configuração inicial single-worker, consulta eventos elegíveis mais antigos a cada 2 segundos. O processamento assina os bytes do JSON, exige HTTPS, limita o corpo a 64 KB e encerra a chamada após 10 segundos.

**[RFC-PROP-04]** Resposta `2xx` encerra a entrega. Falhas de rede, timeout e respostas não `2xx` seguem os intervalos `1 min, 5 min, 30 min, 2 h, 12 h`; ao esgotar, a evidência vai para tabela de dead letter. O replay manual é restrito a `ADMIN` e auditado.

**[RFC-PROP-05]** Cada destino possui secret própria e rotacionável, com 24 horas de sobreposição. O cliente recebe `X-Signature`, `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id`. Duplicatas podem ocorrer; exactly-once não é prometido.

O detalhamento de modelos, estados, contratos HTTP, payloads, matriz de erros e integração por arquivo está no [FDD](./FDD.md).

## Alternativas consideradas

### [RFC-ALT-01] Chamada síncrona na mudança de status

Descartada porque endpoints lentos bloqueariam uma transação que já altera pedido, histórico e estoque; indisponibilidade externa não deve provocar rollback da regra de negócio. O trade-off do processamento assíncrono é aceitar consistência temporal e operar worker/outbox.

### [RFC-ALT-02] Redis Streams ou cluster

Descartado na primeira fase por acrescentar infraestrutura e carga operacional a um time pequeno. O MySQL existente resolve atomicidade e volume inicial. O trade-off é polling no banco e menor capacidade de escala horizontal imediata.

### [RFC-ALT-03] Trigger no MySQL

Descartada porque trigger executa SQL, mas não acorda processo externo de forma nativa. Qualquer mecanismo improvisado adicionaria acoplamento sem superar o polling de 2 segundos para a meta atual.

### [RFC-ALT-04] Exactly-once

Descartado por exigir coordenação entre OMS e consumidores. At-least-once simplifica o produtor e reduz risco de perda, ao custo de exigir deduplicação do cliente.

### [RFC-ALT-05] Três tentativas ou retry indefinido

Três foram consideradas insuficientes para uma manutenção de duas horas; retry infinito manteria eventos abandonados para sempre. A política graduada com DLQ limita ambos os riscos, ao custo de atrasos longos antes da falha permanente.

## Questões em aberto

- **[RFC-OPEN-01] Rate limiting por cliente:** observar o tráfego real e decidir depois; não faz parte desta fase.
- **[RFC-OPEN-02] Escala e ordenação com múltiplos workers:** a primeira versão opera single-worker. Particionamento por `order_id` ou lock pessimista será avaliado somente quando houver necessidade de escala.
- **[RFC-OPEN-03] Arquivamento/retenção:** a reunião mencionou arquivar entregues “depois de 30 dias ou assim”, mas adiou o tema e não fechou prazo nem mecanismo.
- **[RFC-OPEN-04] Contagem de retries:** “5 tentativas” conflita com os cinco intervalos de retentativa enumerados. Este RFC interpreta provisoriamente como tentativa inicial + cinco retries para utilizar a sequência completa; revisores devem confirmar antes do desenvolvimento.
- **[RFC-OPEN-05] Proteção da secret em repouso:** HMAC, geração e rotação foram decididos, mas o mecanismo de criptografia/gestão da secret armazenada não foi especificado e deve ser fechado na revisão de segurança.
- **[RFC-OPEN-06] Assinatura durante a rotação:** a reunião exige 24 horas de validade em paralelo, mas não define se o OMS continua assinando com a chave anterior, envia duas assinaturas ou permite ao cliente escolher o instante da ativação. Sofia deve aprovar o contrato exato; uma única `X-Signature` não pode representar silenciosamente duas chaves.

E-mail de alerta/fallback e dashboard visual não são questões em aberto desta entrega: foram explicitamente adiados para fases/projetos futuros.

## Impacto e riscos

| Risco/impacto | Prob. | Impacto | Tratamento proposto |
|---|---:|---:|---|
| Crescimento da outbox e custo de polling | Média | Médio | índices por estado/data, lotes pequenos e métricas de backlog; retenção fica como decisão futura |
| Duplicatas no consumidor | Alta em cenários de falha | Médio | contrato at-least-once e `X-Event-Id` estável |
| Vazamento de secret em logs/armazenamento | Média | Alto | secret por endpoint, rotação, redaction e revisão obrigatória de Sofia |
| Single-worker limitar throughput | Baixa inicialmente | Médio | medir atraso e backlog; discutir particionamento somente quando necessário |
| Ambiguidade da contagem de retries | Alta até revisão | Médio | confirmar [RFC-OPEN-04] antes de implementar |

**[RFC-IMPACT-01]** A entrega foi estimada em três sprints, incluindo dois dias úteis reservados para a revisão de segurança antes do deploy.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado com polling](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](./adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — Assinatura HMAC por endpoint](./adrs/ADR-004-assinatura-hmac-por-endpoint.md)
- [ADR-005 — Entrega at-least-once](./adrs/ADR-005-entrega-at-least-once.md)
- [ADR-006 — Reuso dos padrões do OMS](./adrs/ADR-006-reuso-dos-padroes-do-oms.md)
