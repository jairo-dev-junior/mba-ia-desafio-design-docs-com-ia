# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e contexto da feature

O OMS permitirá que clientes B2B cadastrem endpoints HTTPS para receber notificações outbound quando pedidos mudarem de status. **[PRD-CTX-01]** Atlas Comercial, MaxDistribuição e Nova Cargo pediram essa capacidade; hoje precisam consultar `GET /orders` repetidamente, tornando suas integrações lentas e caras. Para esses clientes, uma notificação em menos de 10 segundos é percebida como tempo real.

## 2. Problema e motivação

O modelo de polling transfere custo e complexidade aos clientes e atrasa a reação a eventos relevantes, como `SHIPPED` e `DELIVERED`. A Atlas informou risco de migrar para um concorrente caso a capacidade não seja entregue até o fim do trimestre. **[PRD-PROB-01]** A oportunidade é reduzir polling e aumentar a previsibilidade das integrações sem permitir que falhas externas prejudiquem a mudança de status no OMS.

## 3. Público-alvo e cenários de uso

- **[PRD-AUD-01] Integradores de clientes B2B:** cadastram um endpoint, escolhem os status relevantes, validam a assinatura e atualizam seus sistemas.
- **[PRD-AUD-02] Usuários autenticados do OMS (`ADMIN` e `OPERATOR`):** gerenciam configurações de webhook para um `customer_id` informado; o JWT atual representa usuário, não cliente.
- **[PRD-AUD-03] Administradores do OMS:** investigam falhas e reprocessam uma dead letter com trilha de auditoria.
- **[PRD-USE-01] Expedição:** cliente inscrito em `SHIPPED` recebe a transição e inicia seu acompanhamento logístico.
- **[PRD-USE-02] Conciliação:** cliente consulta as últimas 100 tentativas para verificar sucesso, falha, payload, resposta e duração.
- **[PRD-USE-03] Segurança operacional:** cliente rotaciona uma secret exposta sem interrupção abrupta, graças à sobreposição de 24 horas.

## 4. Objetivos e métricas de sucesso

| Objetivo | Indicador | Meta |
|---|---|---:|
| **[PRD-METRIC-01]** Notificar com percepção de tempo real | tempo entre criação do evento e início da primeira tentativa, sem backlog | **< 10 s** |
| **[PRD-METRIC-02]** Não perder intenção de notificação após mudança confirmada | mudanças de status confirmadas com endpoint inscrito que possuem outbox correspondente | **100%** |
| **[PRD-METRIC-03]** Permitir manutenção segura de credencial | duração em que secret anterior e nova coexistem após rotação | **24 h** |
| **[PRD-METRIC-04]** Dar visibilidade operacional ao cliente | número máximo de tentativas recentes disponíveis por endpoint | **100** |

Não foi acordada meta de taxa de sucesso de entrega porque a disponibilidade do endpoint pertence ao cliente. Na primeira fase, backlog, latência, retries e DLQ serão medidos para definir metas operacionais futuras.

## 5. Escopo

### 5.1 Incluído

- Configuração de endpoints outbound por cliente e filtros por status.
- Notificação assíncrona de mudanças de status com snapshot enxuto do pedido.
- Assinatura, rotação de credenciais, retries, histórico, DLQ e replay administrativo.
- Reuso da autenticação, autorização, auditoria, erros e logging existentes.

### 5.2 Fora de escopo

- **[PRD-OUT-01] Webhooks inbound:** clientes não enviarão eventos ao OMS nesta feature; o fluxo é somente outbound.
- **[PRD-OUT-02] E-mail de alerta/fallback:** explicitamente adiado para uma próxima fase depois de medir impacto.
- **[PRD-OUT-03] Dashboard visual:** projeto separado do frontend; esta fase entrega apenas API.
- **[PRD-OUT-04] Rate limiting de saída:** será observado e decidido se virar problema.
- **[PRD-OUT-05] Escala horizontal do worker e ordenação global:** múltiplos workers, particionamento ou locking ficam para necessidade futura.
- **[PRD-OUT-06] Arquivamento/retenção:** limpeza de entregues foi mencionada, mas mecanismo e prazo não foram fechados.
- **[PRD-OUT-07] Itens no payload:** detalhes continuam disponíveis em `GET /orders/:id`.

## 6. Requisitos funcionais

- **[PRD-FR-01] Cadastrar endpoint:** usuário autenticado informa `customer_id`, URL HTTPS e lista não vazia de status; o OMS gera e devolve a secret.
- **[PRD-FR-02] Listar endpoints:** usuário autenticado lista as configurações de um customer sem expor secrets.
- **[PRD-FR-03] Editar endpoint:** usuário autenticado altera URL, filtro de status e estado ativo.
- **[PRD-FR-04] Remover endpoint:** usuário autenticado remove/desativa uma configuração para impedir novos eventos.
- **[PRD-FR-05] Filtrar na origem:** só criar evento quando um endpoint ativo do customer estiver inscrito no novo status; caso contrário, não ocupar a outbox.
- **[PRD-FR-06] Notificar mudança:** produzir um `order.status_changed` para cada destino interessado com snapshot do momento da transição.
- **[PRD-FR-07] Assinar entrega:** enviar HMAC-SHA256 com secret única por endpoint e os headers `X-Signature`, `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id`.
- **[PRD-FR-08] Rotacionar secret:** gerar nova secret e aceitar a anterior por 24 horas.
- **[PRD-FR-09] Retentar falhas:** reagendar entregas nos intervalos acordados e encerrar em DLQ; a contagem exata tentativa/retry precisa da confirmação registrada no RFC.
- **[PRD-FR-10] Consultar histórico:** expor as últimas 100 tentativas de um endpoint, com sucesso/falha, payload, resposta e tempo.
- **[PRD-FR-11] Reprocessar DLQ:** permitir replay manual somente a `ADMIN`, registrando quem executou.
- **[PRD-FR-12] Deduplicar no consumidor:** manter o mesmo event ID nas tentativas, permitindo que o cliente trate duplicatas esperadas pela garantia at-least-once.

## 7. Requisitos não funcionais

- **[PRD-NFR-01] Latência:** iniciar a entrega abaixo de 10 segundos em operação normal; polling base de 2 segundos.
- **[PRD-NFR-02] Atomicidade:** mudança confirmada e outbox são uma unidade transacional.
- **[PRD-NFR-03] Disponibilidade:** lentidão/indisponibilidade externa não bloqueia nem reverte pedidos já válidos.
- **[PRD-NFR-04] Segurança:** aceitar apenas HTTPS, limitar payload a 64 KB sem truncar e não expor secrets em listagens/logs.
- **[PRD-NFR-05] Timeout:** considerar falha quando o endpoint não responder em 10 segundos.
- **[PRD-NFR-06] Semântica de entrega:** at-least-once, sem promessa de exactly-once ou ordenação global.
- **[PRD-NFR-07] Auditabilidade:** conservar histórico de tentativas, DLQ e identidade do administrador que fizer replay.
- **[PRD-NFR-08] Compatibilidade:** manter Node.js/TypeScript, MySQL/Prisma e padrões modulares do OMS, sem Redis nesta fase.

## 8. Decisões e trade-offs principais

- **[PRD-DEC-01] Outbox no MySQL em vez de HTTP síncrono/Redis:** garante atomicidade e evita nova infraestrutura; adiciona polling e crescimento de tabela.
- **[PRD-DEC-02] Worker separado e single-worker:** isola a API e preserva ordem por pedido na escala inicial; limita throughput e não sustenta a mesma ordem ao escalar.
- **[PRD-DEC-03] Backoff finito + DLQ:** tolera falhas temporárias sem reter eventos para sempre; pode atrasar a conclusão por muitas horas e requer replay operacional.
- **[PRD-DEC-04] HMAC por endpoint:** reduz raio de vazamento e autentica o corpo; exige gestão segura de secrets por ambos os lados.
- **[PRD-DEC-05] At-least-once:** reduz risco de perda e complexidade; clientes devem deduplicar.
- **[PRD-DEC-06] Snapshot na inserção:** reflete o estado no instante correto; consome mais armazenamento que renderizar depois.

As justificativas completas estão nos [ADRs](./adrs/).

## 9. Dependências

- **[PRD-DEP-01]** Transação de mudança de status e dados de pedidos/clientes existentes no OMS.
- **[PRD-DEP-02]** MySQL/Prisma, autenticação JWT e roles `ADMIN`/`OPERATOR` existentes.
- **[PRD-DEP-03]** Consumidores capazes de expor HTTPS, validar HMAC-SHA256 e deduplicar por event ID.
- **[PRD-DEP-04]** Três sprints de execução estimadas e dois dias úteis de revisão de segurança por Sofia antes do deploy.
- **[PRD-DEP-05]** Confirmação da semântica de contagem de retries e da proteção da secret em repouso antes da implementação correspondente.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---:|---:|---|
| **[PRD-RISK-01]** Endpoint indisponível acumular backlog | Média | Alto | backoff finito, DLQ e monitoramento de idade/volume |
| **[PRD-RISK-02]** Secret comprometida | Média | Alto | secret por endpoint, rotação com 24 h, TLS, redaction e revisão de segurança |
| **[PRD-RISK-03]** Duplicata gerar efeito repetido no cliente | Alta em falhas | Médio | contrato explícito at-least-once e `X-Event-Id` para deduplicação |
| **[PRD-RISK-04]** Single-worker não acompanhar volume | Baixa inicialmente | Médio | medir backlog/latência e avaliar particionamento somente com evidência |
| **[PRD-RISK-05]** Crescimento de dados sem política de retenção | Média | Médio | métricas de volume e decisão posterior de arquivamento |
| **[PRD-RISK-06]** Divergência na contagem de retries | Alta até revisão | Médio | resolver questão aberta do RFC antes de codificar e testar a sequência |

## 11. Critérios de aceitação

- **[PRD-AC-01]** CRUD autenticado permite cadastrar, listar, editar e remover endpoints com filtro de status e HTTPS obrigatório.
- **[PRD-AC-02]** Uma transição confirmada cria eventos somente para destinos ativos/interessados, na mesma transação.
- **[PRD-AC-03]** Em condição normal, a primeira tentativa começa em menos de 10 segundos e não bloqueia a resposta da mudança de status.
- **[PRD-AC-04]** Cliente valida HMAC e observa os quatro headers; payload respeita 64 KB e não contém itens.
- **[PRD-AC-05]** Falha/timeout dispara a política de backoff e termina em DLQ após o limite aprovado.
- **[PRD-AC-06]** Histórico mostra até 100 tentativas com os campos solicitados.
- **[PRD-AC-07]** Rotação mantém credencial anterior válida por 24 h sem revelar secrets em listagens.
- **[PRD-AC-08]** Replay falha para não-admin, funciona para `ADMIN` e registra o ator.
- **[PRD-AC-09]** Duplicatas conservam `X-Event-Id`, e a documentação informa a responsabilidade de deduplicação.
- **[PRD-AC-10]** Sofia conclui a revisão de HMAC e geração/armazenamento de secret antes do deploy.

## 12. Estratégia de testes e validação

- **[PRD-TEST-01] Funcional:** validar CRUD, filtros, rotação, histórico e replay contra os critérios de aceitação.
- **[PRD-TEST-02] Integração:** simular mudanças com/sem inscrição e provar commit/rollback atômico.
- **[PRD-TEST-03] Resiliência:** simular sucesso, timeout de 10 s, `4xx`, `5xx`, queda e recuperação do worker, retries e DLQ.
- **[PRD-TEST-04] Segurança:** verificar HMAC com vetor conhecido, HTTPS/64 KB, RBAC, ausência de secrets em responses/logs e grace period.
- **[PRD-TEST-05] Validação de produto:** medir tempo até primeira tentativa e confirmar com Atlas, MaxDistribuição e Nova Cargo que o contrato atende suas integrações.
- **[PRD-TEST-06] Regressão:** executar a suíte atual de autenticação e pedidos, preservando máquina de estados, estoque e auditoria.
