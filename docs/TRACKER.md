# Tracker de Rastreabilidade

Cada linha rastreia um item identificável dos documentos. Detalhes de implementação que concretizam uma decisão (por exemplo, nomes de métricas ou formato REST) apontam para a fala ou padrão de código que lhes dá origem; não devem ser confundidos com uma nova decisão de produto.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Outbox MySQL atômica e snapshot por destino inscrito | TRANSCRICAO | [09:06] Diego; [09:34] Bruno; [09:40] Bruno; [09:52] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Single-worker separado, polling de 2 s e ordem limitada | TRANSCRICAO | [09:09] Diego; [09:11] Diego; [09:12] Diego; [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Backoff finito, DLQ separada e replay ADMIN | TRANSCRICAO | [09:15] Diego–[09:18] Diego; [09:35] Diego–[09:36] Sofia |
| ADR-004 | docs/adrs/ADR-004-assinatura-hmac-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24 h, TLS e 64 KB | TRANSCRICAO | [09:20] Sofia–[09:24] Larissa |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once.md | Decisão | At-least-once e deduplicação por X-Event-Id | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-oms.md | Decisão | Reusar estrutura modular, erros, logger, Zod, auth e Prisma | TRANSCRICAO | [09:27] Bruno–[09:30] Larissa; [09:40] Bruno–[09:41] Diego |
| RFC-CTX-01 | docs/RFC.md | Contexto | Três clientes pedem notificação abaixo de 10 s | TRANSCRICAO | [09:00] Marcos–[09:02] Marcos |
| RFC-PROP-01 | docs/RFC.md | Proposta | Visão consolidada: outbox, worker, HMAC, retry e at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-02 | docs/RFC.md | Proposta | Snapshot por inscrição na mesma transação | TRANSCRICAO | [09:34] Bruno; [09:40] Bruno–[09:41] Diego; [09:52] Larissa |
| RFC-PROP-03 | docs/RFC.md | Proposta | Worker separado, polling, TLS, 64 KB e timeout | TRANSCRICAO | [09:09] Diego; [09:11] Diego; [09:23] Sofia–[09:24] Larissa; [09:42] Diego |
| RFC-PROP-04 | docs/RFC.md | Proposta | 2xx entrega; falhas seguem backoff e DLQ/replay | TRANSCRICAO | [09:15] Diego–[09:19] Larissa |
| RFC-PROP-05 | docs/RFC.md | Proposta | Secret por destino, rotação e headers de entrega | TRANSCRICAO | [09:20] Sofia–[09:22] Sofia; [09:44] Diego–[09:45] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono bloquearia pedidos | TRANSCRICAO | [09:03] Larissa–[09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis adicionaria infraestrutura excessiva | TRANSCRICAO | [09:07] Larissa–[09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger MySQL não notifica processo externo | TRANSCRICAO | [09:09] Bruno–[09:10] Larissa |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Exactly-once exigiria coordenação complexa | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Três retries são poucos e retry infinito não termina | TRANSCRICAO | [09:15] Diego–[09:17] Larissa |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting será observado | TRANSCRICAO | [09:38] Diego–[09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Escala/ordem com múltiplos workers foi adiada | TRANSCRICAO | [09:12] Diego–[09:13] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Retenção/arquivamento não teve prazo fechado | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-04 | docs/RFC.md | Ambiguidade | “5 tentativas” conflita com cinco intervalos enumerados | TRANSCRICAO | [09:15] Diego–[09:17] Larissa; [09:48] Larissa |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | Proteção da secret em repouso não foi especificada | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia; [09:46] Sofia |
| RFC-OPEN-06 | docs/RFC.md | Questão em aberto | Forma de assinar durante as 24 h de sobreposição não foi definida | TRANSCRICAO | [09:20] Sofia–[09:22] Sofia; [09:44] Diego |
| RFC-IMPACT-01 | docs/RFC.md | Prazo/dependência | Três sprints e dois dias de revisão de segurança | TRANSCRICAO | [09:45] Marcos–[09:47] Larissa |
| FDD-CTX-01 | docs/FDD.md | Contexto técnico | HTTP sai da transação, outbox permanece nela | TRANSCRICAO | [09:04] Bruno–[09:06] Diego; [09:40] Bruno–[09:41] Diego |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Atomicidade de status, histórico, estoque e evento | TRANSCRICAO | [09:40] Bruno–[09:41] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Primeira tentativa abaixo de 10 s com polling de 2 s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego–[09:10] Larissa |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Worker isolado da API | TRANSCRICAO | [09:11] Diego–[09:11] Larissa |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | HTTPS, HMAC, rotação e limite de payload | TRANSCRICAO | [09:20] Sofia–[09:24] Larissa |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Retry, DLQ e replay auditado | TRANSCRICAO | [09:15] Diego–[09:19] Larissa; [09:36] Sofia |
| FDD-OBJ-06 | docs/FDD.md | Compatibilidade | Reusar padrões e stack existentes | TRANSCRICAO | [09:27] Bruno–[09:30] Larissa |
| FDD-SCOPE-01 | docs/FDD.md | Escopo | CRUD e filtro por status | TRANSCRICAO | [09:31] Marcos–[09:34] Bruno |
| FDD-SCOPE-02 | docs/FDD.md | Escopo | Rotação e últimas 100 entregas | TRANSCRICAO | [09:21] Sofia; [09:34] Marcos |
| FDD-SCOPE-03 | docs/FDD.md | Escopo | Evento, outbox, worker, retry, DLQ e replay | TRANSCRICAO | [09:06] Diego; [09:15] Diego–[09:19] Larissa |
| FDD-SCOPE-04 | docs/FDD.md | Escopo técnico | Assinatura, headers e observabilidade com Pino | TRANSCRICAO | [09:20] Sofia; [09:29] Bruno; [09:44] Diego–[09:45] Diego |
| FDD-OUT-01 | docs/FDD.md | Fora de escopo | Inbound, e-mail e dashboard excluídos | TRANSCRICAO | [09:02] Marcos; [09:37] Larissa; [09:39] Marcos–[09:40] Larissa |
| FDD-OUT-02 | docs/FDD.md | Fora de escopo | Rate limiting e múltiplos workers adiados | TRANSCRICAO | [09:13] Diego; [09:38] Diego–[09:39] Larissa |
| FDD-OUT-03 | docs/FDD.md | Fora de escopo | Arquivamento automático adiado | TRANSCRICAO | [09:08] Diego |
| FDD-OUT-04 | docs/FDD.md | Fora de escopo | Payload sem items; detalhes via GET order | TRANSCRICAO | [09:43] Diego–[09:44] Bruno |
| FDD-DATA-01 | docs/FDD.md | Modelo de dados | Endpoint guarda customer, URL, secrets, filtros e ativo | TRANSCRICAO | [09:21] Bruno–[09:22] Sofia; [09:31] Marcos–[09:34] Bruno |
| FDD-DATA-02 | docs/FDD.md | Modelo de dados | Outbox UUID, snapshot, estados e índices de polling | TRANSCRICAO | [09:06] Diego–[09:08] Diego; [09:51] Larissa–[09:52] Larissa |
| FDD-DATA-03 | docs/FDD.md | Modelo de dados | Histórico contém tentativa, resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-04 | docs/FDD.md | Modelo de dados | DLQ conserva payload/motivo e replay preserva evidência | TRANSCRICAO | [09:18] Diego–[09:19] Diego; [09:36] Sofia |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Publicação filtrada dentro de changeStatus | TRANSCRICAO | [09:34] Bruno; [09:40] Bruno–[09:41] Diego |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Worker busca antigos, assina, envia e registra | TRANSCRICAO | [09:08] Diego–[09:12] Diego; [09:42] Diego–[09:45] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Falhas seguem progressão persistida de backoff | TRANSCRICAO | [09:15] Diego–[09:17] Larissa |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Esgotamento move para DLQ e replay exige ADMIN/auditoria | TRANSCRICAO | [09:17] Larissa–[09:19] Larissa; [09:35] Diego–[09:36] Sofia |
| FDD-API-01 | docs/FDD.md | Contrato HTTP | POST cria endpoint e devolve secret gerada | TRANSCRICAO | [09:31] Marcos–[09:32] Larissa |
| FDD-API-02 | docs/FDD.md | Contrato HTTP | GET lista webhooks de um customer | TRANSCRICAO | [09:32] Larissa–[09:33] Bruno |
| FDD-API-03 | docs/FDD.md | Contrato HTTP | PATCH edita endpoint e filtros | TRANSCRICAO | [09:33] Bruno–[09:33] Marcos |
| FDD-API-04 | docs/FDD.md | Contrato HTTP | DELETE remove endpoint | TRANSCRICAO | [09:33] Bruno |
| FDD-API-05 | docs/FDD.md | Contrato HTTP | Endpoint rotaciona secret com 24 h de sobreposição | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia |
| FDD-API-06 | docs/FDD.md | Contrato HTTP | GET retorna últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-API-07 | docs/FDD.md | Contrato HTTP | POST admin reprocessa dead letter | TRANSCRICAO | [09:18] Diego–[09:19] Larissa; [09:35] Diego–[09:36] Sofia |
| FDD-CONTRACT-01 | docs/FDD.md | Contrato outbound | Payload de status e quatro headers | TRANSCRICAO | [09:43] Diego–[09:45] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s | TRANSCRICAO | [09:42] Sofia–[09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Retry agendado com backoff | TRANSCRICAO | [09:15] Diego–[09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Duplicata aceita com event ID estável | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| FDD-RES-04 | docs/FDD.md | Fallback | DLQ/replay; e-mail não entra | TRANSCRICAO | [09:18] Diego; [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Worker separado possui ciclo de processo próprio | TRANSCRICAO | [09:11] Diego–[09:11] Larissa |
| FDD-RES-06 | docs/FDD.md | Resiliência | Lote pequeno e single-worker; rate limit adiado | TRANSCRICAO | [09:08] Diego; [09:12] Diego; [09:38] Diego–[09:39] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Métricas cobrem backlog, latência, retries e DLQ | TRANSCRICAO | [09:02] Marcos; [09:08] Bruno–[09:08] Diego; [09:15] Diego–[09:18] Diego |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs Pino correlacionam fluxo e auditam replay sem secrets | TRANSCRICAO | [09:29] Bruno; [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por event ID e lacuna atual de tracing | CODIGO | src/shared/logger/index.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Estender changeStatus dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reusar OrderStatus e máquina de estados | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Acrescentar modelos/índices MySQL com UUID | CODIGO | prisma/schema.prisma |
| FDD-INT-04 | docs/FDD.md | Integração | Compor controller/service/repository | CODIGO | src/app.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Montar rotas sob /api/v1 | CODIGO | src/routes/index.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Reusar JWT e requireRole ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Validar com Zod e enum existente | CODIGO | src/middlewares/validate.middleware.ts; src/modules/orders/order.schemas.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Erros WEBHOOK_* derivados de AppError | CODIGO | src/shared/errors/app-error.ts; src/middlewares/error.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Reusar Pino e ampliar redaction | CODIGO | src/shared/logger/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Worker espelha bootstrap/shutdown e cria Prisma próprio | CODIGO | src/config/database.ts; src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Futuro script de worker segue scripts Node existentes | CODIGO | package.json |
| FDD-INT-12 | docs/FDD.md | Integração | Estender testes e limpeza sem regressão | CODIGO | tests/orders.test.ts; tests/setup.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Stack Node/TS/Express/Prisma/MySQL/Zod/Pino | CODIGO | package.json; prisma/schema.prisma |
| FDD-DEP-02 | docs/FDD.md | Dependência | API e worker usam mesma URL e Prisma por processo | TRANSCRICAO | [09:11] Bruno; [09:29] Diego–[09:30] Bruno |
| FDD-DEP-03 | docs/FDD.md | Compatibilidade | Cliente HTTP não foi decidido; manter Node 20 | CODIGO | package.json |
| FDD-DEP-04 | docs/FDD.md | Dependência | Revisão de segurança por dois dias antes do deploy | TRANSCRICAO | [09:46] Sofia–[09:47] Larissa |
| FDD-AC-01 | docs/FDD.md | Critério de aceite | Commit/rollback atômico demonstrado | TRANSCRICAO | [09:40] Bruno–[09:41] Diego |
| FDD-AC-02 | docs/FDD.md | Critério de aceite | Sem interesse, sem outbox | TRANSCRICAO | [09:33] Marcos–[09:34] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de aceite | Worker 2 s e primeira tentativa <10 s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego–[09:10] Larissa |
| FDD-AC-04 | docs/FDD.md | Critério de aceite | JSON, HMAC e quatro headers estáveis | TRANSCRICAO | [09:20] Sofia; [09:25] Diego; [09:44] Diego–[09:45] Diego |
| FDD-AC-05 | docs/FDD.md | Critério de aceite | Timeout, backoff, DLQ e replay cobertos | TRANSCRICAO | [09:15] Diego–[09:19] Larissa; [09:42] Diego |
| FDD-AC-06 | docs/FDD.md | Critério de aceite | Replay só ADMIN e auditado | TRANSCRICAO | [09:35] Larissa–[09:36] Sofia |
| FDD-AC-07 | docs/FDD.md | Critério de aceite | Rotação 24 h e secret não exposta | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério de aceite | Payload >64 KB falha sem truncar | TRANSCRICAO | [09:23] Sofia–[09:24] Larissa |
| FDD-AC-09 | docs/FDD.md | Critério de aceite | Observar enqueue até entrega/DLQ | TRANSCRICAO | [09:29] Bruno; [09:34] Marcos; [09:36] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério de aceite | Regressão verde e revisão pré-deploy | TRANSCRICAO | [09:46] Sofia–[09:47] Larissa |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes fazem polling e querem <10 s | TRANSCRICAO | [09:00] Marcos–[09:02] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Reduzir polling sem afetar pedidos por falha externa | TRANSCRICAO | [09:00] Marcos; [09:04] Bruno–[09:06] Diego |
| PRD-AUD-01 | docs/PRD.md | Público | Integradores B2B consomem outbound | TRANSCRICAO | [09:00] Marcos–[09:03] Sofia |
| PRD-AUD-02 | docs/PRD.md | Público | Usuários JWT gerenciam customer informado | TRANSCRICAO | [09:31] Marcos–[09:37] Sofia |
| PRD-AUD-03 | docs/PRD.md | Público | ADMIN reprocessa e deixa auditoria | TRANSCRICAO | [09:35] Larissa–[09:36] Sofia |
| PRD-USE-01 | docs/PRD.md | Cenário | Cliente filtra SHIPPED/DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-USE-02 | docs/PRD.md | Cenário | Cliente consulta últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-USE-03 | docs/PRD.md | Cenário | Rotação mantém duas secrets por 24 h | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia |
| PRD-METRIC-01 | docs/PRD.md | Métrica | Primeira tentativa abaixo de 10 s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego–[09:10] Larissa |
| PRD-METRIC-02 | docs/PRD.md | Métrica | Toda mudança interessada confirmada tem outbox | TRANSCRICAO | [09:40] Bruno–[09:41] Diego |
| PRD-METRIC-03 | docs/PRD.md | Métrica | Sobreposição de secrets por 24 h | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia |
| PRD-METRIC-04 | docs/PRD.md | Métrica | Histórico expõe 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | Webhooks inbound excluídos | TRANSCRICAO | [09:02] Sofia–[09:03] Sofia |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | E-mail adiado | TRANSCRICAO | [09:37] Marcos–[09:38] Marcos |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Dashboard visual adiado | TRANSCRICAO | [09:39] Marcos–[09:40] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Rate limiting será observado | TRANSCRICAO | [09:38] Diego–[09:39] Larissa |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Multi-worker e ordering global adiados | TRANSCRICAO | [09:12] Diego–[09:13] Larissa |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Retenção não decidida | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo | Items fora do payload | TRANSCRICAO | [09:43] Diego–[09:44] Bruno |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint e gerar secret | TRANSCRICAO | [09:31] Marcos–[09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listar endpoints por customer sem secret | TRANSCRICAO | [09:32] Larissa–[09:33] Bruno; [09:21] Sofia |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar configuração e filtros | TRANSCRICAO | [09:33] Bruno–[09:33] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtrar antes de inserir outbox | TRANSCRICAO | [09:33] Marcos–[09:34] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Evento de mudança usa snapshot | TRANSCRICAO | [09:43] Diego; [09:51] Bruno–[09:52] Larissa |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | HMAC e quatro headers | TRANSCRICAO | [09:20] Sofia; [09:44] Diego–[09:45] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotacionar secret com 24 h | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Retry progressivo termina em DLQ | TRANSCRICAO | [09:15] Diego–[09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Consultar últimas 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay manual ADMIN auditado | TRANSCRICAO | [09:18] Diego–[09:19] Larissa; [09:35] Larissa–[09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Cliente deduplica pelo event ID | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência <10 s e polling 2 s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego–[09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade com outbox | TRANSCRICAO | [09:40] Bruno–[09:41] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Falha externa não bloqueia pedidos | TRANSCRICAO | [09:04] Bruno–[09:06] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | HTTPS, 64 KB e proteção de secret | TRANSCRICAO | [09:21] Sofia–[09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP 10 s | TRANSCRICAO | [09:42] Sofia–[09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | At-least-once sem ordering global | TRANSCRICAO | [09:13] Larissa; [09:24] Diego–[09:26] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Histórico, DLQ e replay auditável | TRANSCRICAO | [09:18] Diego; [09:34] Marcos; [09:36] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Reusar stack; sem Redis | TRANSCRICAO | [09:07] Diego; [09:27] Bruno–[09:30] Larissa |
| PRD-DEC-01 | docs/PRD.md | Decisão/trade-off | Outbox MySQL evita HTTP síncrono e Redis | TRANSCRICAO | [09:03] Larissa–[09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão/trade-off | Worker separado e único | TRANSCRICAO | [09:11] Diego–[09:13] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão/trade-off | Backoff finito e DLQ | TRANSCRICAO | [09:15] Diego–[09:18] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão/trade-off | HMAC e secret por endpoint | TRANSCRICAO | [09:20] Sofia–[09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão/trade-off | At-least-once versus exactly-once | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão/trade-off | Snapshot na inserção | TRANSCRICAO | [09:51] Bruno–[09:52] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Transação e dados de orders existentes | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | MySQL/Prisma e JWT/RBAC existentes | CODIGO | prisma/schema.prisma; src/middlewares/auth.middleware.ts |
| PRD-DEP-03 | docs/PRD.md | Dependência externa | Consumidor valida HMAC e deduplica | TRANSCRICAO | [09:20] Sofia; [09:25] Diego–[09:26] Marcos |
| PRD-DEP-04 | docs/PRD.md | Prazo/dependência | Três sprints e revisão de dois dias | TRANSCRICAO | [09:45] Marcos–[09:47] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência de decisão | Fechar retries e proteção da secret | TRANSCRICAO | [09:15] Diego–[09:17] Larissa; [09:46] Sofia |
| PRD-RISK-01 | docs/PRD.md | Risco | Indisponibilidade acumula backlog | TRANSCRICAO | [09:14] Larissa–[09:18] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Secret comprometida | TRANSCRICAO | [09:21] Sofia–[09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Duplicata no cliente | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| PRD-RISK-04 | docs/PRD.md | Risco | Single-worker limita escala | TRANSCRICAO | [09:12] Diego–[09:13] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Dados crescem sem retenção | TRANSCRICAO | [09:07] Bruno–[09:08] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Contagem de retries ambígua | TRANSCRICAO | [09:15] Diego–[09:17] Larissa; [09:48] Larissa |
| PRD-AC-01 | docs/PRD.md | Critério de aceitação | CRUD, filtro e HTTPS | TRANSCRICAO | [09:23] Sofia; [09:31] Marcos–[09:34] Bruno |
| PRD-AC-02 | docs/PRD.md | Critério de aceitação | Outbox atômica apenas para interessados | TRANSCRICAO | [09:34] Bruno; [09:40] Bruno–[09:41] Diego |
| PRD-AC-03 | docs/PRD.md | Critério de aceitação | <10 s sem bloquear status | TRANSCRICAO | [09:02] Marcos; [09:04] Bruno; [09:09] Diego |
| PRD-AC-04 | docs/PRD.md | Critério de aceitação | HMAC, headers, 64 KB e sem items | TRANSCRICAO | [09:20] Sofia; [09:23] Sofia–[09:24] Larissa; [09:43] Diego–[09:45] Diego |
| PRD-AC-05 | docs/PRD.md | Critério de aceitação | Backoff termina em DLQ | TRANSCRICAO | [09:15] Diego–[09:18] Diego |
| PRD-AC-06 | docs/PRD.md | Critério de aceitação | Histórico com 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-07 | docs/PRD.md | Critério de aceitação | Grace period de 24 h | TRANSCRICAO | [09:21] Sofia–[09:22] Sofia |
| PRD-AC-08 | docs/PRD.md | Critério de aceitação | Replay ADMIN auditado | TRANSCRICAO | [09:35] Larissa–[09:36] Sofia |
| PRD-AC-09 | docs/PRD.md | Critério de aceitação | Event ID estável para duplicatas | TRANSCRICAO | [09:24] Diego–[09:26] Larissa |
| PRD-AC-10 | docs/PRD.md | Critério de aceitação | Revisão de Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia–[09:49] Sofia |
| PRD-TEST-01 | docs/PRD.md | Validação | Validar CRUD, filtros, rotação, histórico e replay | TRANSCRICAO | [09:21] Sofia; [09:31] Marcos–[09:36] Sofia |
| PRD-TEST-02 | docs/PRD.md | Validação | Provar atomicidade com e sem inscrição | TRANSCRICAO | [09:34] Bruno; [09:40] Bruno–[09:41] Diego |
| PRD-TEST-03 | docs/PRD.md | Validação | Exercitar timeout, erros, retries e DLQ | TRANSCRICAO | [09:14] Larissa–[09:19] Larissa; [09:42] Diego |
| PRD-TEST-04 | docs/PRD.md | Validação de segurança | HMAC, TLS, 64 KB, RBAC e secrets | TRANSCRICAO | [09:19] Sofia–[09:24] Larissa; [09:36] Sofia |
| PRD-TEST-05 | docs/PRD.md | Validação de produto | Medir latência com os três clientes solicitantes | TRANSCRICAO | [09:00] Marcos–[09:02] Marcos |
| PRD-TEST-06 | docs/PRD.md | Regressão | Preservar status, estoque e auditoria | CODIGO | src/modules/orders/order.service.ts; tests/orders.test.ts |
