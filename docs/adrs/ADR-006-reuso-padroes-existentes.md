# ADR-006: Reuso dos Padrões Existentes do Projeto

## Status

Aceito

## Contexto

A aplicação OMS existente tem padrões bem definidos de arquitetura e tratamento de erros. A feature de webhooks poderia ser implementada com bibliotecas ou abordagens diferentes, mas foi definido explicitamente que seguir os padrões já estabelecidos reduz surpresa, acelera a implementação e mantém a legibilidade do código para o time (ver TRACKER: ADR-006).

Bruno listou os padrões existentes e propôs reutilizá-los (ver TRACKER: ADR-006) e a proposta foi aceita (ver TRACKER: ADR-006).

## Decisão

Reutilizar os padrões existentes do projeto para o módulo de webhooks, conforme detalhado abaixo.

### Estrutura de módulo

O módulo de webhooks segue exatamente a estrutura dos demais módulos da aplicação (ver TRACKER: ADR-006):

```
src/modules/webhooks/
├── webhook.controller.ts
├── webhook.service.ts
├── webhook.repository.ts
├── webhook.routes.ts
└── webhook.schemas.ts
```

Isso replica o padrão já usado em `orders/`, `customers/`, `products/`, `users/` e `auth/` (verificado em `src/modules/` do repositório).

### Classes de erro — padrão AppError e prefixo WEBHOOK_

O projeto tem uma hierarquia de erros em `src/shared/errors/`:
- `AppError` — classe base que recebe `(message, statusCode, errorCode, details)` (em `app-error.ts`)
- Subclasses: `NotFoundError`, `ConflictError`, `ValidationError`, `UnprocessableEntityError`, etc. (em `http-errors.ts`)

Os erros do módulo de webhooks usam as **mesmas subclasses** com código no **prefixo `WEBHOOK_`** (ver TRACKER: ADR-006):
- `WEBHOOK_NOT_FOUND`
- `WEBHOOK_INVALID_URL`
- `WEBHOOK_SECRET_REQUIRED`
- etc.

### Error middleware centralizado

O middleware em `src/middlewares/error.middleware.ts` já trata:
- `AppError` → retorna `{ error: { code, message, details } }`
- `ZodError` → retorna `{ error: { code: 'VALIDATION_ERROR', details } }`
- `Prisma.PrismaClientKnownRequestError` → casos P2002 (409) e P2025 (404)

Esse middleware **não precisa ser alterado** — erros do módulo de webhooks que disparam `AppError` ou `ZodError` serão capturados automaticamente (ver TRACKER: ADR-006).

### Logger Pino

O logger do projeto (`src/shared/logger/index.ts`) usa Pino com redact de campos sensíveis, ISO timestamp, e metadados de serviço. O módulo de webhooks utiliza a mesma instância exportada, sem criar logger próprio (ver TRACKER: ADR-006).

### Autenticação e autorização — `requireRole`

O middleware `requireRole` em `src/middlewares/auth.middleware.ts` verifica se o usuário autenticado possui uma das roles necessárias (ver TRACKER: ADR-006):
- Endpoints CRUD de configuração de webhook exigem **autenticação JWT** (qualquer role) (ver TRACKER: ADR-006).
- Endpoint de replay de DLQ exige **role `ADMIN`** via `requireRole('ADMIN')` (ver TRACKER: ADR-006).

### Schemas Zod

As validações seguem o padrão Zod já utilizado nos demais módulos, com schemas específicos para cada endpoint. Inclui validação de URL obrigatoriamente `https` (ver TRACKER: ADR-006).

### Configuração de rotas

As rotas de webhook seguem o padrão `buildRouter(controller)` já utilizado, registradas dentro do router principal em `src/routes/index.ts`.

### PrismaClient

O worker mantém sua própria instância de `PrismaClient` (separada da API), conectando ao mesmo banco com a mesma `DATABASE_URL` (ver TRACKER: ADR-006).

## Alternativas Consideradas

### 1. Logger separado para webhooks

Criar uma instância de logger específica do módulo. Descartada por consistência: o Pino configurado já atende todos os módulos, e criar um separado fragmentaria os logs (ver TRACKER: ADR-006).

## Consequências

### Positivas

- **Consistência entre módulos:** engenheiros que conhecem o módulo de pedidos reconhecem a estrutura no módulo de webhooks.
- **Zero alteração em infraestrutura compartilhada:** error middleware, logger, auth, validação — tudo já captura e trata os novos erros sem modificação.

### Negativas

- **Acoplamento ao padrão existente:** se o módulo de webhooks precisar de comportamentos que os padrões existentes não cobrem, pode exigir adaptações (ex: validação customizada de HMAC).
- **Worker não segue o mesmo padrão de controller/service:** porque é um loop contínuo, não um request handler — exigirá estrutura diferente dentro do mesmo módulo (ex: `webhook.worker.ts` ou `webhook.processor.ts` como sugerido por Bruno, (ver TRACKER: ADR-006)).

### Trade-off explícito

Reusar padrões maximiza consistência e minimiza surpresas, ao custo de possível rigidez se o módulo de webhooks exigir comportamentos atípicos. O trade-off foi aceito com a ressalva de que o worker, por ser um processo contínuo, terá estrutura diferente da API (ver TRACKER: ADR-006).