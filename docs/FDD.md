# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e Motivação Técnica

O OMS registra toda mudança de status de pedido dentro de uma transação única em `src/modules/orders/order.service.ts` (método `changeStatus`). Nessa transação o sistema já: atualiza `orders.status`, insere em `order_status_history` e gerencia estoque (débito em `PENDING → PAID`, restituição em `PAID → CANCELLED` e `PROCESSING → CANCELLED`, conforme `src/modules/orders/order.status.ts`).

Três clientes B2B exigem ser notificados em tempo real quando o status dos pedidos muda. A aplicação não possui nenhum mecanismo de notificação externa (sem webhooks, eventos ou filas). A feature preenche esse vácuo registrando um evento na tabela `webhook_outbox` **dentro da mesma transação** da mudança de status, garantindo atomicidade: ou o status muda e o evento existe, ou nada muda. Um worker lerá a outbox e fará as entregas HTTP.

## 2. Objetivos Técnicos

- Garantir que nenhuma mudança de status fique sem evento registrado (atômico com a transação existente).
- Entregar notificações para múltiplos endpoints por cliente, com filtro por status.
- Garantir entrega com retry (at-least-once) e persistência de falhas em DLQ para reprocessamento.
- Não introduzir infraestrutura nova: usar MySQL existente, mesmo Prisma Client e padrões do projeto.

## 3. Escopo e Exclusões

**Incluso:**
- Tabela `webhook_outbox` (eventos pendentes/entregues) e `webhook_dead_letter` (falhas permanentes).
- Tabela `webhook` de configuração (url, secret, customer_id, eventos desejados, ativo).
- CRUD de configuração autenticado.
- Endpoint admin para replay de DLQ.
- Worker separado (`src/worker.ts`) com polling periódico.

**Excluso:**
- Dashboard visual para clientes (fora de escopo, ver TRACKER: `FDD-ESC-01`).
- Notificação por email de falha (fora de escopo, ver TRACKER: `FDD-ESC-02`).
- Rate limiting de envio sobre o qual a equipe deve observar antes de implementar (ver TRACKER: `FDD-ESC-03`).

## 4. Fluxos Detalhados

### 4.1 Criação do evento na outbox (dentro da transação de `changeStatus`)

1. Cliente autenticado chama `PATCH /api/v1/orders/:id/status` com `{ toStatus }`.
2. `OrderService.changeStatus` inicia `prisma.$transaction` (existente).
3. Valida a transição (via `canTransition` de `order.status.ts`); valida estoque (débito/restituição).
4. Atualiza `orders.status`, insere em `order_status_history`.
5. **Novo passo:** para cada webhook ativo do `customerId` do pedido que deseje o novo status, insere linha em `webhook_outbox` com o payload já renderizado (snapshot do estado da order) — mesma transação.
6. Commit. Se qualquer passo falhar, rollback: status não muda e evento não é criado.

### 4.2 Processamento pelo worker

1. `src/worker.ts` consulta, a cada 2 segundos, eventos pendentes na outbox (status `PENDING`), ordenados por `created_at`, em lotes pequenos.
2. Para cada evento lê o destino (`webhook.url`) e a secret; monta payload + headers (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type).
3. Executa `POST` no destino.
4. Sucesso (HTTP 2xx) → marca evento como `DELIVERED`.
5. Falha → registra falha e agenda próxima tentativa conforme backoff.
6. Esgotadas 5 tentativas → move para `webhook_dead_letter` com motivo e timestamp, remove da outbox.

### 4.3 Retry

Baseado na tabela de tentativas:

| Tentativa | Intervalo | Acumulado |
|---|---|---|
| 1ª | 1 minuto | 1 min |
| 2ª | 5 minutos | 6 min |
| 3ª | 30 minutos | 36 min |
| 4ª | 2 horas | ~2h36 |
| 5ª | 12 horas | ~14h36 |

### 4.4 DLQ e reprocessamento

1. Evento em `webhook_dead_letter` com payload, motivo e timestamp.
2. Admin autenticado chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
3. O evento volta para `webhook_outbox` como `PENDING`, com contador de tentativas zerado.
4. A operação é logada (quem, quando) — auditoria.

## 5. Contratos Públicos

Endpoint para cada operação, com exemplos de request/response e status codes. Os nomes de rotas seguem o padrão `/api/v1` existente (`src/routes/index.ts`); paths não especificados na definição são detalhe de implementação deste FDD.

### 5.1 `POST /api/v1/webhooks` — criar webhook

Request (JWT autenticado — qualquer role):
```http
POST /api/v1/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json
```
```json
{
  "customerId": "uuid-do-cliente",
  "url": "https://cliente.example.com/webhook",
  "events": ["PAID", "SHIPPED", "DELIVERED"]
}
```
Response `201 Created`:
```json
{
  "id": "uuid-do-webhook",
  "customerId": "uuid-do-cliente",
  "url": "https://cliente.example.com/webhook",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "secret-gerada-uma-vez",
  "active": true
}
```
Status codes: `201` sucesso; `400` validação (url não-https, secret), `401` sem JWT, `404` cliente não existe.

### 5.2 `PATCH /api/v1/webhooks/:id` — editar webhook

Request (JWT autenticado — qualquer role):
```http
PATCH /api/v1/webhooks/:id
Authorization: Bearer <jwt>
Content-Type: application/json
```
```json
{
  "url": "https://cliente.example.com/webhook-v2",
  "events": ["PAID", "DELIVERED"],
  "active": true
}
```
Response `200 OK` — mesmo formato de 5.1 (sem `secret`).
Status: `200` sucesso, `404` webhook não existe, `400` validação.

### 5.3 `DELETE /api/v1/webhooks/:id` — remover webhook

Request:
```http
DELETE /api/v1/webhooks/:id
Authorization: Bearer <jwt>
```
Response `204 No Content`. Status: `204`, `404`.

### 5.4 `GET /api/v1/customers/:customerId/webhooks` — listar webhooks de um cliente

O `customerId` vai no path: o identificador do cliente é passado no **body ou no path**, nunca derivado do JWT (ver TRACKER: `FDD-CUSTOMER-01`).

Request:
```http
GET /api/v1/customers/:customerId/webhooks
Authorization: Bearer <jwt>
```
Response `200 OK`:
```json
{
  "data": [
    {
      "id": "uuid",
      "customerId": "uuid",
      "url": "https://cliente.example.com/webhook",
      "events": ["PAID"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1 }
}
```
Status: `200`, `400` (sem customerId), `401`.

### 5.5 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Retorna as **últimas 100 entregas** do webhook, com sucesso/falha, payload, response e tempo de resposta (ver TRACKER: `FDD-DELIVERY-01`).
Request:
```http
GET /api/v1/webhooks/:id/deliveries
Authorization: Bearer <jwt>
```
Response `200 OK`:
```json
{
  "data": [
    {
      "id": "delivery-uuid",
      "webhookId": "webhook-uuid",
      "eventId": "event-uuid",
      "deliveryStatus": "SUCCESS",
      "payload": {
        "event_id": "event-uuid",
        "event_type": "order.status_changed",
        "order_id": "order-uuid"
      },
      "responseStatus": 200,
      "durationMs": 340
    }
  ],
  "limit": 100
}
```
Status: `200`, `404`, `401`.

### 5.7 Contrato de entrega do webhook (payload e headers de saída)

É o contrato real consumido pelos clientes — como a feature **chama** o endpoint do cliente. Payload enviado no corpo da requisição `POST` do worker para `webhook.url`.

**Payload** (ver TRACKER: `FDD-PAYLOAD-01`):
```json
{
  "event_id": "uuid-do-evento",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-20T10:00:00.000Z",
  "order_id": "uuid-do-pedido",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "uuid-do-cliente",
  "total_cents": 12345
}
```
Sem `items`, para manter payload enxuto (ver TRACKER: `FDD-PAYLOAD-02`).

**Headers da requisição de saída** (ver TRACKER: `FDD-HEADER-01`):

| Header | Valor | Finalidade |
|---|---|---|
| `Content-Type` | `application/json` | Tipo do corpo |
| `X-Event-Id` | UUID do evento | Deduplicação pelo cliente (ADR-004) |
| `X-Signature` | HMAC-SHA256 hex do corpo | Autenticação (ADR-003) |
| `X-Timestamp` | ISO 8601 do envio | Detecção de replay attack |
| `X-Webhook-Id` | ID do endpoint webhook | Client com vários webhooks distingue origem |

**Semântica de sucesso:** considera-se entregue quando o cliente responde `2xx`. Timeout de **10 segundos** por tentativa; não responder em 10s é falha e aciona retry (ADR-002) (ver TRACKER: `FDD-SEMANTICA-01`).

Request:
```http
POST /api/v1/admin/webhooks/dead-letter/:id/replay
Authorization: Bearer <jwt-de-admin>
```
Response `200 OK`:
```json
{ "status": "REQUEUED" }
```
Status: `200` recolocado na outbox, `403` role diferente de ADMIN, `404` DLQ item não existe.

## 6. Matriz de Erros Previstos (padrão `WEBHOOK_*`)

O *padrão* `WEBHOOK_*` foi definido com exemplos (ver TRACKER: `FDD-ERRO-01`): `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`. Os demais códigos abaixo seguem o mesmo padrão e as subclasses de `AppError` existentes, mas **não foram citados literalmente** — são derivação deste FDD a partir da lógica dos endpoints.

| Código | HTTP | Quando | Origem |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook inexistente em GET/PATCH/DELETE | Transcrição (ver TRACKER: `FDD-ERRO-01`) |
| `WEBHOOK_INVALID_URL` | 400 | URL sem `https` | Transcrição (ver TRACKER: `FDD-ERRO-02`) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret ausente em cadastro | Transcrição (ver TRACKER: `FDD-ERRO-01`) |
| `WEBHOOK_INVALID_EVENT` | 400 | Evento de status inválido (não enum) | Derivação FDD (padrão Zod existente) |
| `WEBHOOK_ALREADY_EXISTS` | 409 | Duplicidade de URL para o mesmo customer | Derivação FDD (padrão ConflictError) |
| `WEBHOOK_NOT_ACTIVE` | 422 | Envio para webhook inativo | Derivação FDD (padrão UnprocessableEntityError) |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Item DLQ inexistente no replay | Derivação FDD (ver TRACKER: `FDD-ERRO-03` — endpoint replay) |
| `WEBHOOK_DELIVERY_FAILED` | 502 | Falha na entrega (registrada para retry) | Derivação FDD (lógica do worker) |

Todos seguem o formato do `AppError` (`message`, `statusCode`, `errorCode`, `details`) já existente em `src/shared/errors/app-error.ts`.

## 7. Estratégias de Resiliência

- **Timeout HTTP:** 10 segundos por chamada (ver TRACKER: `FDD-SEMANTICA-01`). Cliente que não responde em 10s → falha → retry.
- **Retry com backoff exponencial:** 5 tentativas: 1m/5m/30m/2h/12h (ver TRACKER: `FDD-RESIL-02`).
- **DLQ:** após 5 falhas, evento vai para `webhook_dead_letter` com motivo e timestamp (ver TRACKER: `FDD-RESIL-03`).
- **At-least-once e dedup:** eventos podem chegar duplicados; `X-Event-Id` único permite dedup no cliente (ver TRACKER: `FDD-RESIL-04`).
- **Filtro de eventos na inserção:** se nenhum webhook do customer quer o status, nem insere na outbox — evita trabalho desnecessário (ver TRACKER: `FDD-RESIL-05`).
- **Fallback/out-of-scope:** notificação por email fica para fase futura (ver TRACKER: `FDD-ESC-02`).

## 8. Observabilidade

Foi definido o **reuso do logger Pino existente** (ver TRACKER: `FDD-OBS-02`) e o padrão de logs com request id já presente na aplicação (via `src/middlewares/request-logger.middleware.ts`). O detalhamento abaixo (métricas e logs estruturados) é derivação deste FDD a partir desses padrões; nomes/campos exatos são sugestão de implementação, não decisão fechada.

### Métricas

Sugeridas para monitoramento das entregas (a aplicar conforme infraestrutura de observabilidade do time, ainda não existente na aplicação):
- Contadores por status de entrega: `webhook_delivered_total`, `webhook_failed_total`.
- Contador de eventos em DLQ: `webhook_dead_letter_total`.
- Latência por envio: histograma `webhook_delivery_duration_seconds`.

### Logs

- Usar o logger Pino existente (`src/shared/logger/index.ts`) — decisão do time (ver TRACKER: `FDD-OBS-02`).
- Campos estruturados sugeridos nos eventos de entrega: `eventId`, `webhookId`, `customerId`, `status`, `attempt`, `durationMs`, `responseStatus`.
- **Auditoria obrigatória no replay de DLQ** (ver TRACKER: `FDD-OBS-03`): registrar quem executou (admin), o evento reprocessado e o timestamp.

### Tracing

- Cada requisição já recebe `X-Request-Id` via `request-logger.middleware.ts`; reutilizar o id nos logs do worker para correlação.
- `requestLogger` registra `method`, `path`, `statusCode`, `durationMs`, `userId` — padrão a ser seguido pelo worker nos logs de ciclo.

## 9. Dependências e Compatibilidade

- **Node/Prisma:** reutiliza `prisma` de `src/config/database.ts` e o schema em `prisma/schema.prisma` (novas tabelas via migration).
- **Sem novas dependências npm:** HMAC via Node `crypto` nativo.
- **Compatibilidade:** API continua servindo os mesmos endpoints; a feature só adiciona rotas novas em `/api/v1/webhooks` e `/api/v1/admin/webhooks`.
- **Zod:** validações seguem `src/middlewares/validate.middleware.ts` e schemas em `*.schemas.ts`.

## 10. Critérios de Aceite Técnicos

1. `PATCH /:id/status` bem-sucedido cria evento na outbox na mesma transação (sem evento órfão).
2. Transição inválida (ex: `PENDING → SHIPPED`) não cria evento.
3. Filtro por status: webhook que não escuta o status não recebe evento.
4. Worker entrega evento em até ~2s (polling) após o commit, com latência total <10s.
5. Após 5 tentativas com falha, evento vai para DLQ.
6. Replay admin recoloca evento na outbox como pendente.
7. Erros de webhook devolvem código `WEBHOOK_*`.
8. URL `http` é rejeitada no cadastro (400).
9. Entrega duplicada usa o mesmo `X-Event-Id` (dedup possível no cliente).

## 11. Riscos e Mitigação

| Risco | Mitigação |
|---|---|
| Cliente fora do ar por horas | Retry com backoff de até ~15h + DLQ (ver TRACKER: `FDD-RISCO-01`) |
| Secret vazada (já ocorreu em outro cliente) | Secret por endpoint + rotação com grace period 24h (ver TRACKER: `FDD-RISCO-02`) |
| Evento duplicado processado pelo cliente | `X-Event-Id` + documentação no portal (ver TRACKER: `FDD-RISCO-03`) |
| Worker derrubado junto com a API | Processo separado (`src/worker.ts`) com entry point próprio (ver TRACKER: `FDD-RISCO-04`) |
| Acúmulo de eventos pendentes | Índice em `status` + `created_at` na outbox; leitura em lotes pequenos (ver TRACKER: `FDD-RISCO-05`) |

## 12. Integração com o Sistema Existente

| Arquivo/caminho | Como o módulo de webhooks se integra |
|---|---|
| `src/modules/orders/order.service.ts` | Dentro do `$transaction` do método `changeStatus` (linha 126), após o `tx.order.update` e o insert em `order_status_history`, chamar a função de *enqueue* do webhook (ex: `publishWebhookEvent(tx, order, fromStatus, toStatus)`) que insere na `webhook_outbox` com o payload snapshot. Se a inserção falhar, rollback total (ver TRACKER: `FDD-INT-01`). |
| `src/modules/orders/order.status.ts` | Reusar o enum `OrderStatus` e os helpers (`canTransition`, `allowedTransitions`) para validar os eventos de webhook e filtrar por status desejado pelo cliente. |
| `src/shared/errors/app-error.ts` + `src/shared/errors/http-errors.ts` | Estender ou reutilizar `AppError` e as subclasses (`ConflictError`, `UnprocessableEntityError`, `NotFoundError`) para lançar erros com código `WEBHOOK_*`. |
| `src/middlewares/error.middleware.ts` | Sem alteração: o middleware já trata `AppError` (formata `error: { code, message, details }`) e `ZodError` (400) — os erros do módulo webhook caem nele automaticamente. |
| `src/middlewares/auth.middleware.ts` | Usar `authenticate` para autenticação JWT; `requireRole('ADMIN')` para o endpoint de replay de DLQ. |
| `src/routes/index.ts` | Registrar `buildWebhookRouter` e o router admin no `buildApiRouter`, montando `/api/v1/webhooks` e `/api/v1/admin/webhooks`. |
| `src/modules/webhooks/` (novo) | Seguir o padrão `controller → service → repository → routes → schemas` dos demais módulos (ver TRACKER: `FDD-INT-02`). |
| `src/shared/logger/index.ts` | Usar o Pino existente (mesma instância exportada), com `eventId`/`webhookId` nos logs de entrega. |
| `prisma/schema.prisma` | Adicionar `webhook`, `webhook_outbox`, `webhook_dead_letter` com UUID padrão (`@default(uuid()) @db.Char(36)`), índices em `status`+`created_at` (ver TRACKER: `FDD-INT-03`). |
| `package.json` | Adicionar script `"worker": "tsx watch --env-file=.env src/worker.ts"` seguindo o padrão do `dev` (entry separada) (ver TRACKER: `FDD-INT-04`). |