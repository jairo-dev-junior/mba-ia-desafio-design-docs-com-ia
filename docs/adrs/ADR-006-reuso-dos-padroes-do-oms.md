# ADR-006 — Reuso dos padrões do OMS

- **Status:** Aceito
- **Data da decisão:** reunião técnica registrada em `TRANSCRICAO.md`

## Contexto

O OMS organiza domínios em módulos com controller, service, repository, routes e schemas. Já possui validação Zod, erros derivados de `AppError`, autenticação JWT/RBAC, Pino, middleware central de erros e criação de `PrismaClient`.

## Decisão

**[ADR-006]** Criar um módulo de domínio para webhooks seguindo a estrutura dos outros domínios e reutilizar:

- composição controller/service/repository/route de `src/app.ts` e `src/routes/index.ts`;
- autenticação e `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`;
- schemas Zod e middleware `validate` usados em `src/modules/orders/order.routes.ts`;
- `AppError` e `errorMiddleware`, com códigos específicos prefixados por `WEBHOOK_`;
- logger Pino de `src/shared/logger/index.ts`, ampliando a redaction para secrets/assinaturas;
- fábrica Prisma de `src/config/database.ts`, com uma instância por processo.

A integração crítica ocorre em `src/modules/orders/order.service.ts`: `changeStatus` chamará a publicação da outbox dentro do callback de `$transaction`, sem mover as regras de estoque ou de máquina de estados.

## Alternativas consideradas

- **Criar uma arquitetura, logger ou hierarquia de erros exclusivos para webhooks:** descartado por fragmentar padrões já conhecidos pelo time.
- **Injetar todo o repository de webhooks em `OrderService`:** preterido em favor de uma função estreita que recebe o transaction client, reduzindo o acoplamento entre domínios.

## Consequências

### Positivas

- Menor curva de aprendizado e integração coerente com o código existente.
- Respostas de erro e autorização permanecem uniformes.
- A feature reaproveita a transação que já protege status, histórico e estoque.

### Negativas e trade-offs

- `app.ts` e `routes/index.ts` precisarão conhecer o novo módulo.
- A redaction atual não cobre explicitamente `secret` e `signature` e deverá ser estendida.
- O worker exige composição própria, embora reutilize os mesmos componentes básicos.

## Referências

- Transcrição: [09:27]–[09:30], [09:35]–[09:36], [09:40]–[09:42], [09:48].
- Código: caminhos listados na decisão.
