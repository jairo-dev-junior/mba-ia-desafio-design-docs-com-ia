# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

Este repositório transforma a transcrição de uma reunião técnica e o código de um Order Management System em um pacote de documentação acionável para um Sistema de Webhooks de Notificação de Pedidos. A entrega separa problema de produto, proposta arquitetural, decisões pontuais e especificação de implementação, mantendo a origem de cada requisito, restrição e decisão visível.

O trabalho é exclusivamente documental. `TRANSCRICAO.md` e a aplicação em `src/`, `prisma/`, `tests/` e arquivos de configuração foram usados como evidência e não foram alterados.

## Ferramentas de IA utilizadas

- **OpenAI Codex:** ferramenta principal para explorar o repositório, classificar as falas da reunião, comparar decisões com o código, redigir os documentos e revisar consistência/rastreabilidade.
- **Ferramentas locais conduzidas pelo Codex:** `rg` para inventário e conferência de referências, leitura dirigida de arquivos, `git diff/status` para controlar o escopo e scripts de validação somente-leitura para verificar checklist, links e caminhos.

## Workflow adotado

1. Inventariei o repositório e li a transcrição inteira, separando decisões fechadas, requisitos, alternativas descartadas, itens adiados e ambiguidades.
2. Inspecionei a transação de `OrderService.changeStatus`, máquina de estados, schema Prisma, composição de módulos, JWT/RBAC, Zod, erros, logger, bootstrap e testes. Esse mapa impediu referências a classes ou caminhos inexistentes.
3. Escrevi primeiro seis ADRs, um para cada decisão arquitetural principal. Eles formaram o esqueleto do RFC.
4. Mantive o RFC conciso e orientado a proposta/revisão; alternativas e questões em aberto ficaram nele, enquanto modelos, fluxos, contratos e erros foram aprofundados no FDD.
5. Consolidei no PRD apenas o problema, o público, o escopo, os resultados e critérios verificáveis, evitando copiar detalhes internos do FDD.
6. Atribuí IDs aos itens e montei o tracker cruzando-os com timestamp + falante ou caminho real de código.
7. Fiz uma revisão final automatizada da estrutura, quantidade de ADRs, seções, endpoints, fontes, links e alterações Git.

## Prompts customizados

O primeiro prompt foi usado como roteiro de extração, antes de redigir qualquer documento:

```text
Leia TRANSCRICAO.md inteira e devolva uma matriz com cinco classes separadas:
(1) decisão fechada, (2) requisito funcional explícito, (3) restrição/NFR,
(4) alternativa descartada e seu motivo, (5) item adiado ou genuinamente aberto.
Para cada linha, copie somente um resumo e indique [hh:mm] + falante.
Não promova perguntas, sugestões ou exemplos a requisitos. Se duas falas se
contradisserem, preserve a ambiguidade e liste as duas fontes.
```

O segundo prompt direcionou a exploração do código e a integração do FDD:

```text
Inspecione o OMS sem editar arquivos. Localize o método transacional de mudança
de status, histórico e estoque; a composição controller/service/repository/routes;
os schemas Zod; AppError/error middleware; JWT e requireRole; logger; Prisma;
bootstrap e testes. Para cada ponto de integração do módulo de webhooks, cite um
caminho que realmente exista e explique a menor alteração futura necessária.
Marque como lacuna tudo que a reunião pede mas o código ainda não oferece.
```

Um terceiro prompt orientou a revisão cruzada:

```text
Audite PRD, RFC, FDD e ADRs contra TRANSCRICAO.md e o código. Procure:
duplicação entre alturas documentais, requisito sem fonte, item adiado tratado
como escopo, arquivo inexistente, número contraditório e contrato insuficiente.
Exija no FDD quatro ou mais endpoints com request/response/status, erros
WEBHOOK_*, integração com quatro ou mais caminhos reais e observabilidade.
Para cada problema, proponha correção e uma linha correspondente no TRACKER.md.
```

## Iterações e ajustes

Foram quatro iterações principais:

1. **Extração e classificação:** a primeira leitura misturava decisões com ideias futuras. E-mail, dashboard, rate limiting, multi-worker e retenção foram reclassificados como fora de escopo ou questões futuras, conforme as falas explícitas.
2. **Confronto com o código:** a ideia inicial de inferir `customer_id` do JWT foi corrigida. O código mostra que o token representa usuário, e a reunião fecha que o customer deve vir no body ou path/query. Também foram substituídas referências genéricas por caminhos reais como `src/modules/orders/order.service.ts` e `src/middlewares/auth.middleware.ts`.
3. **Separação dos documentos:** detalhes de tabelas, payloads, headers, estados e erros que deixavam o RFC longo foram movidos para o FDD. O RFC ficou com proposta, alternativas, riscos e pontos para revisão; ADRs passaram a conter apenas uma decisão cada.
4. **Auditoria de números e rastreabilidade:** foi detectada uma inconsistência da própria reunião entre “5 tentativas” e cinco intervalos de retentativa. Em vez de escolher silenciosamente, o RFC/ADR/FDD registram a leitura provisória e pedem confirmação. A última varredura cobriu IDs, fontes e caminhos citados.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [PRD](docs/PRD.md) — problema, público, escopo, requisitos e resultados esperados.
2. [RFC](docs/RFC.md) — proposta para revisão, alternativas e questões abertas.
3. [ADRs](docs/adrs/) — seis decisões arquiteturais isoladas e seus trade-offs.
4. [FDD](docs/FDD.md) — fluxos, contratos, erros, resiliência, observabilidade e integração com o OMS.
5. [Tracker](docs/TRACKER.md) — referência cruzada para a transcrição e o código.

A fonte primária da reunião permanece em [TRANSCRICAO.md](TRANSCRICAO.md). O enunciado original está no [repositório-base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
