# Tracker de Rastreabilidade

Referência cruzada entre os documentos de design, a transcrição da reunião e o código existente. Cada linha indica a origem de um item (decisão, requisito, restrição) para garantir que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL: evento inserido na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Payload do evento é snapshot renderizado na inserção | TRANSCRICAO | [09:51]-[09:52] Larissa, Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | ID do evento é UUID, padrão das demais tabelas | TRANSCRICAO | [09:51] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Tabela outbox com índice em status e created_at; leitura em lotes pequenos | TRANSCRICAO | [09:08] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Ordenação implícita por order_id, sem garantir ordering global (single-worker) | TRANSCRICAO | [09:12]-[09:13] Diego, Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Integração no changeStatus (src/modules/orders/order.service.ts) — mesma transação | CODIGO | src/modules/orders/order.service.ts:126 |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: webhook síncrono dentro do service (travaria outros pedidos) | TRANSCRICAO | [09:03]-[09:04] Bruno, Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: Redis Streams (infra nova, overengineering) | TRANSCRICAO | [09:07] Larissa, Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Alternativa descartada: trigger MySQL (sem NOTIFY/LISTEN nativo) | TRANSCRICAO | [09:09] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | Retry com backoff exponencial: 5 tentativas, 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Reflexão | Alternativa descartada: 3 tentativas (janela curta, ~30min) | TRANSCRICAO | [09:16] Bruno, Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Trade-off | Alternativa descartada: retry indefinido (evento pendurado pra sempre) | TRANSCRICAO | [09:15] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | DLQ em tabela separada (webhook_dead_letter) com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Trade-off | Alternativa descartada: marcar failed na própria outbox (poluiria leitura) | TRANSCRICAO | [09:18] Diego, Bruno |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Requisito Funcional | Endpoint admin POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Restrição | Replay DLQ exige role ADMIN; operação logada para auditoria | TRANSCRICAO | [09:35]-[09:36] Sofia |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Requisito Não Funcional | Timeout HTTP de 10s por tentativa | TRANSCRICAO | [09:42] Diego, Sofia |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Autenticação HMAC-SHA256 sobre o payload | TRANSCRICAO | [09:20] Sofia |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint (não global) | TRANSCRICAO | [09:21] Sofia |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21]-[09:22] Sofia, Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Trade-off | Alternativa descartada: secret global (vazamento comprometeria tudo) | TRANSCRICAO | [09:21] Sofia |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Restrição | TLS obrigatório; URL com http recusada na validação Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Requisito Funcional | Tabela de configuração armazena url, secret, customer_id, estado ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Decisão | Garantia at-least-once (evento pode chegar duplicado) | TRANSCRICAO | [09:24]-[09:25] Diego |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Decisão | X-Event-Id com UUID gerado na entrada da outbox para dedup do cliente | TRANSCRICAO | [09:25] Diego |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Trade-off | Alternativa descartada: exactly-once (complexo, coordenação bidirecional) | TRANSCRICAO | [09:25] Diego |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Requisito Funcional | Headers: X-Event-Id, X-Signature, X-Timestamp, Content-Type, X-Webhook-Id | TRANSCRICAO | [09:44]-[09:45] Diego, Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Restrição | Dedup é responsabilidade do cliente, documentado no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo separado, polling a cada 2s | TRANSCRICAO | [09:09]-[09:11] Diego, Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Requisito Não Funcional | Latência máx. 10s para notificação (negócio) | TRANSCRICAO | [09:02] Marcos |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Trade-off | Alternativa descartada: worker embutido na API (perde worker em restart) | TRANSCRICAO | [09:11] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Trade-off | Alternativa descartada: trigger MySQL para reatividade (sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Trade-off | Alternativa descartada: fila externa Redis Streams (overengineering) | TRANSCRICAO | [09:07] Diego, [09:10] Marcos |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Entry point src/worker.ts + script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Restrição | Worker usa mesma DATABASE_URL, PrismaClient por processo | TRANSCRICAO | [09:29]-[09:30] Bruno, Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Ordenação: created_at dentro de cada order_id, sem garantia global | TRANSCRICAO | [09:12]-[09:13] Diego, Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo src/modules/webhooks com controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Restrição | Erros com prefixo WEBHOOK_ (ex: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL) | TRANSCRICAO | [09:28]-[09:29] Bruno |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Trade-off | Alternativa descartada: logger separado para webhooks (fragmentaria logs) | TRANSCRICAO | [09:29] Bruno |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso do error middleware centralizado existente (sem alteração) | TRANSCRICAO | [09:29] Bruno |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso do middleware requireRole (ADMIN no replay) | TRANSCRICAO | [09:35]-[09:36] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | CRUD de webhook com autenticação JWT normal (qualquer role) | TRANSCRICAO | [09:36]-[09:37] Sofia, Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Restrição | Erros seguem padrão AppError existente | CODIGO | src/shared/errors/app-error.ts:3 |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Restrição | Error middleware trata AppError, ZodError e Prisma sem alteração | CODIGO | src/middlewares/error.middleware.ts:14 |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Logger Pino do módulo (sem instância própria) | CODIGO | src/shared/logger/index.ts:32 |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Restrição | requireRole('ADMIN') existente para endpoint admin | CODIGO | src/middlewares/auth.middleware.ts:49 |

**Cobertura:** linhas até o momento cobrem os 6 ADRs produzidos. O tracker será atualizado conforme PRD, RFC, FDD e README forem produzidos (varredura no fim de cada documento).