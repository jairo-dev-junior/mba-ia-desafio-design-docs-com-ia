# FDD — Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e motivação técnica

O OMS não possui mecanismo de eventos, filas ou notificações externas. Mudanças de status são aplicadas por `OrderService.changeStatus` dentro de uma transação Prisma que também grava histórico e pode debitar ou repor estoque. **[FDD-CTX-01]** O desenho deve registrar a intenção de notificar na mesma transação, mas executar HTTP fora dela, de modo que indisponibilidade do cliente não bloqueie pedidos e uma mudança confirmada nunca fique sem evento persistido.

Este documento detalha a implementação das decisões consolidadas no [RFC](./RFC.md) e nos [ADRs](./adrs/).

## 2. Objetivos técnicos

- **[FDD-OBJ-01] Atomicidade:** para toda mudança de status confirmada que tenha endpoint ativo e inscrito, criar a outbox na mesma transação; falha na outbox reverte status, histórico e estoque.
- **[FDD-OBJ-02] Latência:** iniciar a primeira tentativa em menos de 10 segundos em condições normais, usando polling de 2 segundos.
- **[FDD-OBJ-03] Isolamento:** entrega HTTP roda em processo separado da API, com sua própria instância Prisma.
- **[FDD-OBJ-04] Segurança:** HTTPS, HMAC-SHA256, secret única por endpoint, rotação com sobreposição de 24 horas e payload máximo de 64 KB.
- **[FDD-OBJ-05] Recuperação:** retry progressivo, DLQ persistente e replay administrativo auditável.
- **[FDD-OBJ-06] Compatibilidade:** seguir módulos, Zod, `AppError`, Pino, JWT/RBAC, UUID e MySQL existentes.

## 3. Escopo e exclusões

### Incluído

- **[FDD-SCOPE-01]** CRUD autenticado de endpoints e filtro por status de pedido.
- **[FDD-SCOPE-02]** Rotação de secret e consulta das últimas 100 entregas.
- **[FDD-SCOPE-03]** Publicação de `order.status_changed`, outbox, worker, retry, DLQ e replay `ADMIN`.
- **[FDD-SCOPE-04]** Assinatura e headers de entrega; métricas, logs estruturados e correlação de tracing.

### Excluído

- **[FDD-OUT-01]** Webhooks inbound, e-mail de alerta/fallback e dashboard visual.
- **[FDD-OUT-02]** Rate limiting de saída, múltiplos workers e garantia de ordering global.
- **[FDD-OUT-03]** Arquivamento automático/retenção definitiva de entregues.
- **[FDD-OUT-04]** Inclusão de itens do pedido no payload; o consumidor usa `GET /orders/:id` para detalhes.

## 4. Arquitetura e modelo lógico

### 4.1 Componentes

1. `webhook.routes/controller/service/repository/schemas`: configuração, rotação e histórico.
2. `webhook.publisher`: recebe `Prisma.TransactionClient`, seleciona inscrições e insere snapshots.
3. `webhook.processor`: serializa/assina uma vez, envia, registra tentativa e decide retry/DLQ.
4. Novo entry point do worker: bootstrap, loop de 2 segundos, shutdown e Prisma próprios.
5. Tabelas Prisma propostas: `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.

### 4.2 Persistência proposta

**[FDD-DATA-01] `webhook_endpoints`:** UUID, `customer_id`, URL, secret vigente, secret anterior e expiração, lista de `OrderStatus`, `active`, timestamps. Indexar `customer_id` e `active`. A representação criptográfica da secret em repouso deve ser aprovada por Sofia antes da migration; hash irreversível não atende porque o worker precisa recuperar a chave HMAC.

**[FDD-DATA-02] `webhook_outbox`:** UUID igual ao `event_id`, `webhook_id`, `order_id`, snapshot JSON, estado (`PENDING`, `PROCESSING`, `DELIVERED`, `FAILED`), contador de retries, `next_attempt_at`, último erro e timestamps. Indexar `(status, next_attempt_at, created_at)`; o worker ordena por `created_at`.

**[FDD-DATA-03] `webhook_deliveries`:** UUID, event/webhook IDs, número da tentativa, início/fim, latência, status HTTP nullable, resposta limitada para diagnóstico, resultado e erro. Suporta as últimas 100 entregas solicitadas.

**[FDD-DATA-04] `webhook_dead_letter`:** UUID, event/webhook/order IDs, snapshot, motivo final, contagem e timestamps. Replay registra ator e data na DLQ e recoloca a outbox original como pendente, mantendo tanto a evidência quanto o mesmo `event_id`.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

**[FDD-FLOW-01]** Dentro do callback já usado por `changeStatus`:

1. Validar a transição e aplicar débito/reposição de estoque como hoje.
2. Atualizar `orders` e criar `order_status_history`.
3. Carregar os dados básicos atualizados do pedido e endpoints do `customer_id` com `active=true` e filtro contendo `toStatus`.
4. Para cada destino, criar UUID e snapshot `order.status_changed`; se nenhum destino quiser o status, não inserir outbox.
5. Inserir as outboxes usando o mesmo `tx`. Qualquer erro propaga e causa rollback total.
6. Commit encerra a responsabilidade síncrona; nenhuma chamada externa ocorre na API.

Não publicar na criação inicial do pedido: o gatilho acordado é a mudança executada por `changeStatus`.

### 5.2 Processamento pelo worker

**[FDD-FLOW-02]** Em um único processo:

1. A cada 2 segundos buscar um lote pequeno de `PENDING` com `next_attempt_at <= now`, mais antigos primeiro.
2. Reservar o item como `PROCESSING` antes da chamada. Como robustez derivada da operação em processo separado, o bootstrap deve devolver a `PENDING` reservas abandonadas após queda; o limiar operacional precisa ser configurado acima do timeout HTTP.
3. Validar URL HTTPS e tamanho; serializar o snapshot exatamente uma vez.
4. Calcular `hex(HMAC-SHA256(secret, rawBody))` e enviar o mesmo `rawBody` com os headers do contrato.
5. Registrar cada tentativa em deliveries, sem secret e com resposta limitada.
6. Em `2xx`, marcar `DELIVERED`. Nos demais resultados, seguir retry.

O processo trata apenas um evento por vez na primeira versão, preservando a ordenação implícita por `created_at`. A garantia é somente por pedido enquanto for single-worker.

### 5.3 Retry

**[FDD-FLOW-03]** Timeout, falha de DNS/conexão e qualquer resposta não `2xx` são falhas de entrega. Agendar `next_attempt_at` conforme `1 min → 5 min → 30 min → 2 h → 12 h`, sem bloquear o loop.

A reunião contém uma ambiguidade: “5 tentativas” e cinco intervalos. A interpretação provisória deste FDD é uma tentativa inicial e até cinco **retentativas**; o contador representa retries já executados. Essa semântica deve ser confirmada pelos revisores antes de codificar ([RFC-OPEN-04](./RFC.md#questões-em-aberto)). Não se aplica retry indefinido.

### 5.4 DLQ e replay

**[FDD-FLOW-04]** Após falhar a última retentativa, em uma transação: copiar snapshot e diagnóstico para `webhook_dead_letter`, marcar a outbox `FAILED` e manter deliveries. O replay:

1. autentica JWT e exige `requireRole('ADMIN')`;
2. localiza a DLQ ou retorna `WEBHOOK_DEAD_LETTER_NOT_FOUND`;
3. redefine a outbox original como `PENDING`, zera a agenda de retry e conserva o `event_id`;
4. grava ator e timestamp na evidência original;
5. responde `202` com o novo identificador.

Replay não apaga a DLQ e pode gerar nova entrega do mesmo fato; o consumidor continua responsável por idempotência.

## 6. Contratos públicos

Todos os endpoints recebem `Authorization: Bearer <JWT>` e JSON sob `/api/v1`. CRUD pode ser usado por `ADMIN` ou `OPERATOR`; a reunião decidiu, nesta fase, que `customerId` vem no body/query e não é inferido do JWT. Replay exige `ADMIN`.

Os formatos abaixo são a concretização proposta das capacidades acordadas. UUIDs e timestamps são ilustrativos.

### 6.1 Criar endpoint — `POST /api/v1/webhooks`

**[FDD-API-01]** Request:

```http
POST /api/v1/webhooks
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "customerId": "4d0fe8be-cd22-4a8e-b4e1-e37013d9ae65",
  "url": "https://atlas.example.com/oms/events",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (a secret só é exibida aqui e na rotação):

```json
{
  "id": "cc6dd84c-02ef-433a-b927-279f767eacbb",
  "customerId": "4d0fe8be-cd22-4a8e-b4e1-e37013d9ae65",
  "url": "https://atlas.example.com/oms/events",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_<valor-gerado-pelo-oms>",
  "createdAt": "2026-08-17T12:00:00.000Z"
}
```

Erros: `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_EVENTS_REQUIRED`, `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### 6.2 Listar endpoints — `GET /api/v1/webhooks?customerId=<uuid>`

**[FDD-API-02]** Request sem body. Response `200 OK` não expõe secrets:

```json
{
  "items": [
    {
      "id": "cc6dd84c-02ef-433a-b927-279f767eacbb",
      "customerId": "4d0fe8be-cd22-4a8e-b4e1-e37013d9ae65",
      "url": "https://atlas.example.com/oms/events",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ]
}
```

Erros: `400 WEBHOOK_CUSTOMER_ID_REQUIRED`.

### 6.3 Editar endpoint — `PATCH /api/v1/webhooks/:id`

**[FDD-API-03]** Request:

```json
{
  "url": "https://atlas.example.com/oms/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200 OK`:

```json
{
  "id": "cc6dd84c-02ef-433a-b927-279f767eacbb",
  "customerId": "4d0fe8be-cd22-4a8e-b4e1-e37013d9ae65",
  "url": "https://atlas.example.com/oms/webhooks",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_EVENTS_REQUIRED`. Secret não é alterada por este endpoint.

### 6.4 Remover endpoint — `DELETE /api/v1/webhooks/:id`

**[FDD-API-04]** Request sem body. Response `204 No Content`, sem body. Retorna `404 WEBHOOK_NOT_FOUND` se não existir. A remoção é lógica (`active=false`): impede novos eventos sem invalidar snapshots, deliveries ou secrets necessários às entregas já registradas.

### 6.5 Rotacionar secret — `POST /api/v1/webhooks/:id/rotate-secret`

**[FDD-API-05]** Request com body vazio `{}`. Response `200 OK`:

```json
{
  "id": "cc6dd84c-02ef-433a-b927-279f767eacbb",
  "secret": "whsec_<novo-valor-gerado-pelo-oms>",
  "previousSecretValidUntil": "2026-08-18T12:00:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`, `500 WEBHOOK_SECRET_GENERATION_FAILED`.

### 6.6 Histórico — `GET /api/v1/webhooks/:id/deliveries`

**[FDD-API-06]** Retorna no máximo as últimas 100 tentativas, mais recentes primeiro. Request sem body. Response `200 OK`:

```json
{
  "items": [
    {
      "eventId": "ca2a8b8a-f5d3-4596-969e-c27a93e308f4",
      "attempt": 1,
      "result": "DELIVERED",
      "httpStatus": 204,
      "response": "",
      "durationMs": 183,
      "payload": { "event_type": "order.status_changed", "order_id": "991a433b-e657-472a-b8d8-b3c396bdba4e" },
      "attemptedAt": "2026-08-17T12:05:02.000Z"
    }
  ]
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.7 Replay administrativo — `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

**[FDD-API-07]** Request com body vazio `{}`. Response `202 Accepted`:

```json
{
  "deadLetterId": "52d4673f-f7a8-4e4c-861d-68cc55e2a22a",
  "eventId": "ca2a8b8a-f5d3-4596-969e-c27a93e308f4",
  "status": "PENDING"
}
```

Erros: `403` do RBAC existente para não-admin; `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 6.8 Entrega ao consumidor

**[FDD-CONTRACT-01]** Exemplo de request outbound:

```http
POST /oms/events HTTP/1.1
Content-Type: application/json
X-Event-Id: ca2a8b8a-f5d3-4596-969e-c27a93e308f4
X-Webhook-Id: cc6dd84c-02ef-433a-b927-279f767eacbb
X-Timestamp: 2026-08-17T12:05:02.000Z
X-Signature: <hex-hmac-sha256-do-corpo-exato>

{
  "event_id": "ca2a8b8a-f5d3-4596-969e-c27a93e308f4",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-17T12:05:00.000Z",
  "order_id": "991a433b-e657-472a-b8d8-b3c396bdba4e",
  "order_number": "ORD-000127",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "4d0fe8be-cd22-4a8e-b4e1-e37013d9ae65",
  "total_cents": 25990
}
```

Semântica:

- `timestamp` é o instante do fato/snapshot; `X-Timestamp` é o instante desta tentativa.
- `event_id` e `X-Event-Id` são idênticos e estáveis entre retries.
- Qualquer `2xx` é sucesso. Timeout de 10 s, transporte ou não `2xx` falham.
- Entrega é at-least-once; consumidor deduplica por event ID.
- Corpo nunca inclui itens e nunca ultrapassa 64 KB; acima disso o evento falha sem truncar.

## 7. Matriz de erros previstos

| Código | HTTP/contexto | Quando ocorre | Tratamento |
|---|---:|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | configuração não existe | não repetir operação da API |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | customer informado não existe | corrigir customer |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | DLQ não existe | não criar replay |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, inválida ou sem HTTPS | corrigir cadastro |
| `WEBHOOK_CUSTOMER_ID_REQUIRED` | 400 | listagem sem customer | informar UUID |
| `WEBHOOK_EVENTS_REQUIRED` | 400 | lista vazia/inválida de status | informar `OrderStatus` válidos |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | worker | corpo excede 64 KB | não truncar; retry/DLQ e log seguro |
| `WEBHOOK_DELIVERY_TIMEOUT` | worker | destino não responde em 10 s | retry |
| `WEBHOOK_DELIVERY_FAILED` | worker | transporte ou resposta não `2xx` | retry/DLQ |
| `WEBHOOK_SIGNATURE_FAILED` | worker | falha ao assinar | não enviar corpo sem assinatura; retry/DLQ |
| `WEBHOOK_SECRET_GENERATION_FAILED` | 500 | geração/rotação falha | não alterar secret vigente |

Erros HTTP são subclasses de `AppError`; Zod valida UUID, enum e HTTPS e o `errorMiddleware` preserva o envelope `{ error: { code, message, details? } }`. Erros internos do worker usam os mesmos códigos em logs/deliveries, sem passar pelo middleware HTTP.

## 8. Estratégias de resiliência

- **[FDD-RES-01] Timeout:** abortar a chamada após 10 segundos e classificá-la como retryable.
- **[FDD-RES-02] Retry/backoff:** persistir `next_attempt_at`; nunca usar `sleep` por evento nem retry inline.
- **[FDD-RES-03] At-least-once:** event ID estável e estado persistido antes/depois do HTTP; duplicata é aceitável.
- **[FDD-RES-04] Fallback:** após esgotamento, DLQ persistente e replay manual; e-mail não é fallback nesta fase.
- **[FDD-RES-05] Queda do processo:** shutdown fecha Prisma como `server.ts`; reservas abandonadas precisam voltar à elegibilidade no reinício.
- **[FDD-RES-06] Backpressure:** lote pequeno, processamento serial e índices. Rate limiting fica apenas sob observação.

## 9. Observabilidade

### Métricas

**[FDD-OBS-01]** Expor/registrar contadores `webhook_events_created_total`, `webhook_delivery_attempts_total{result,http_class}`, `webhook_delivered_total`, `webhook_retries_total` e `webhook_dlq_total`; gauges `webhook_outbox_pending` e idade do evento pendente mais antigo; histogramas de tempo entre criação e primeira tentativa e `webhook_delivery_duration_ms`. Alerta operacional prioritário: p95 de criação→primeira tentativa aproximando-se de 10 segundos ou backlog/idade crescendo continuamente.

### Logs

**[FDD-OBS-02]** Reusar Pino e emitir eventos estruturados `webhook_enqueued`, `webhook_attempt_started`, `webhook_delivered`, `webhook_retry_scheduled`, `webhook_moved_to_dlq` e `webhook_dlq_replayed`, com `eventId`, `webhookId`, `orderId`, attempt, duração, status HTTP/código de erro e, no replay, `actorUserId`. Nunca registrar secret, `X-Signature`, Authorization nem payload/resposta integral; ampliar redaction para esses campos.

### Tracing e correlação

**[FDD-OBS-03]** O projeto não possui SDK de tracing hoje. Preservar correlação assíncrona persistindo `event_id` e, se disponível no request que muda status, `trace_id` na outbox. Delimitar spans lógicos `webhook.enqueue`, `webhook.dequeue`, `webhook.sign`, `webhook.http` e `webhook.persist_result`; até existir backend de tracing, os mesmos IDs nos logs permitem reconstruir o fluxo. A adoção de biblioteca/exporter é dependência futura, não motivo para introduzir infraestrutura nova nesta entrega.

## 10. Integração com o sistema existente

| Caminho real | Integração necessária |
|---|---|
| **[FDD-INT-01]** `src/modules/orders/order.service.ts` | Em `changeStatus`, após update/histórico e antes do commit, chamar `publishWebhookEvent(tx, refreshedOrder, from, to)`. Manter no mesmo `$transaction` as regras existentes de estoque. |
| **[FDD-INT-02]** `src/modules/orders/order.status.ts` | Reusar `OrderStatus`/transições como universo válido do filtro; não criar enum divergente para eventos. |
| **[FDD-INT-03]** `prisma/schema.prisma` | Adicionar os quatro modelos, relações com Customer/Order quando aplicável, UUIDs `Char(36)` e índices de polling. Preservar configurações removidas logicamente enquanto houver outbox/histórico dependente. O datasource já é MySQL. |
| **[FDD-INT-04]** `src/app.ts` | Compor repository/service/controller de webhooks em `buildControllers`; não iniciar worker no `buildApp`. |
| **[FDD-INT-05]** `src/routes/index.ts` | Acrescentar controller e montar rotas `/webhooks` e `/admin/webhooks`, mantendo o prefixo `/api/v1`. |
| **[FDD-INT-06]** `src/middlewares/auth.middleware.ts` | Reusar `authenticate`; aplicar `requireRole('ADMIN')` somente ao replay, conforme decisão da reunião. |
| **[FDD-INT-07]** `src/middlewares/validate.middleware.ts` e `src/modules/orders/order.schemas.ts` | Seguir validação Zod existente para params/body/query e reutilizar `OrderStatus`; impor UUID, lista não vazia, HTTPS e 64 KB. |
| **[FDD-INT-08]** `src/shared/errors/app-error.ts` e `src/middlewares/error.middleware.ts` | Criar erros do módulo derivados de `AppError`, prefixados `WEBHOOK_`; conservar envelope e tratamento central. |
| **[FDD-INT-09]** `src/shared/logger/index.ts` | Reusar Pino e ampliar `redactPaths` para `*.secret`, `*.previousSecret`, `*.signature` e headers de assinatura. |
| **[FDD-INT-10]** `src/config/database.ts` e `src/server.ts` | Fazer o futuro entry point do worker espelhar bootstrap/shutdown do servidor, mas criar Prisma por processo. |
| **[FDD-INT-11]** `package.json` | Na implementação futura, adicionar scripts equivalentes a `dev:worker`/`worker`; nenhum código/configuração é alterado por este design doc. |
| **[FDD-INT-12]** `tests/orders.test.ts` e `tests/setup.ts` | Estender testes transacionais e preparar limpeza/factories das novas tabelas, sem regressão na suíte de pedidos. |

## 11. Dependências e compatibilidade

- **[FDD-DEP-01]** Node.js ≥20, TypeScript/ESM, Express 4, Prisma 5, MySQL, Zod e Pino permanecem a base.
- **[FDD-DEP-02]** API e worker compartilham `DATABASE_URL`, schema e código do módulo, mas não `PrismaClient` em memória.
- **[FDD-DEP-03]** O envio HTTP e HMAC podem usar capacidades do Node 20; a escolha de cliente HTTP não foi discutida e deve evitar dependência sem necessidade.
- **[FDD-DEP-04]** A revisão de Sofia reserva dois dias úteis antes do deploy e bloqueia a aprovação final de geração/armazenamento/rotação de secrets.

## 12. Estratégia de testes

1. **Unitários:** schemas HTTPS/status; snapshot; HMAC sobre bytes conhecidos; classificação `2xx`/erro; cálculo de agenda; limite de 64 KB.
2. **Integração com MySQL/Prisma:** commit cria histórico + outbox; falha ao inserir outbox faz rollback de status/histórico/estoque; filtro evita evento; DLQ/replay preserva evidência.
3. **Worker com servidor HTTP controlado:** sucesso, 4xx, 5xx, timeout, conexão interrompida, duplicata com mesmo event ID e rotação durante grace period.
4. **API/Supertest:** pelo menos os sete contratos, envelopes `WEBHOOK_*`, ausência de secrets em GET e RBAC do replay.
5. **Regressão:** executar a suíte atual de autenticação e pedidos.

## 13. Critérios de aceite técnicos

- **[FDD-AC-01]** Mudança com inscrição cria snapshot/outbox atomicamente; rollback é demonstrado por teste.
- **[FDD-AC-02]** Mudança sem endpoint interessado não cria outbox.
- **[FDD-AC-03]** Worker separado consulta a cada 2 s e a primeira tentativa ocorre em menos de 10 s em teste sem backlog.
- **[FDD-AC-04]** Entrega contém JSON e quatro headers definidos, HMAC verificável e event ID estável entre retries.
- **[FDD-AC-05]** HTTP é abortado em 10 s; sequência de backoff e DLQ/replay é coberta com relógio falso.
- **[FDD-AC-06]** Apenas `ADMIN` reprocessa DLQ e o ator fica auditado.
- **[FDD-AC-07]** O mecanismo aprovado na revisão de segurança demonstra transição sem interrupção durante 24 h, encerra o uso da secret anterior depois da janela e não expõe secrets em listagem/log; o formato exato está em [RFC-OPEN-06](./RFC.md#questões-em-aberto).
- **[FDD-AC-08]** Payload acima de 64 KB retorna/classifica `WEBHOOK_PAYLOAD_TOO_LARGE`, sem truncamento.
- **[FDD-AC-09]** Métricas, logs e IDs de correlação permitem acompanhar enqueue→tentativas→entrega/DLQ.
- **[FDD-AC-10]** Suíte existente continua verde e a revisão de segurança é concluída antes do deploy.

## 14. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---:|---:|---|
| Secret exposta em log ou storage | Média | Alto | redaction, resposta única, rotação e revisão de segurança; fechar proteção em repouso |
| Backlog elevar latência além de 10 s | Média | Alto | índices, lote pequeno, idade do mais antigo e p95; escalar desenho só com evidência |
| Duplicata causar efeito repetido no cliente | Alta sob falhas | Médio | documentação at-least-once e deduplicação por event ID |
| Worker cair com item `PROCESSING` | Média | Alto | recuperação de reserva no bootstrap e estado persistente |
| Outbox/DLQ crescer sem retenção | Média | Médio | medir volume e abrir decisão posterior de arquivamento |
| Política de retries implementada com contagem errada | Alta até revisão | Médio | encerrar [RFC-OPEN-04] e testar cada intervalo com relógio falso |
