# Tracker de Rastreabilidade

Referência cruzada entre os documentos de design, a transcrição da reunião e o código existente. Cada linha indica a origem de um item (decisão, requisito, restrição) para garantir que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL: evento inserido na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Payload do evento é snapshot renderizado na inserção | TRANSCRICAO | [09:51]-[09:52] Larissa, Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | ID do evento é UUID, padrão das demais tabelas | TRANSCRICAO | [09:51] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Tabela outbox com índice em status e created_at; leitura em lotes pequenos | TRANSCRICAO | [09:08] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Ordenação implícita por order_id, sem garantir ordering global (single-worker) | TRANSCRICAO | [09:12]-[09:13] Diego, Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Mudança de status e evento na outbox na mesma transação (integração no changeStatus) | TRANSCRICAO | [09:40]-[09:41] |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | changeStatus executa a transação de mudança de status (ponto de integração da outbox) | CODIGO | src/modules/orders/order.service.ts:126 |
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
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
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
| RFC-TLDR | docs/RFC.md | Resumo | TL;DR: 3 clientes B2B demandam notificação, solução com outbox + worker + HMAC | TRANSCRICAO | [09:00]-[09:02] |
| RFC-ID-02 | docs/RFC.md | Requisito Funcional | Notificação deve chegar em até 10s | TRANSCRICAO | [09:02] |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Alternativa descartada: webhook síncrono (travaria outros pedidos) | TRANSCRICAO | [09:03]-[09:06] |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Alternativa descartada: Redis Streams (overengineering p/ time pequeno) | TRANSCRICAO | [09:07] |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Alternativa descartada: trigger MySQL (sem NOTIFY/LISTEN nativo) | TRANSCRICAO | [09:09] |
| RFC-QA-01 | docs/RFC.md | Questão Aberta | Rate limiting de envio por cliente — observar e decidir depois | TRANSCRICAO | [09:38]-[09:39] |
| RFC-QA-02 | docs/RFC.md | Questão Aberta | Escalabilidade para múltiplos workers — adiado | TRANSCRICAO | [09:12]-[09:13] |
| RFC-RISCO-01 | docs/RFC.md | Riscos | Cliente não implementa dedup pode gerar suporte | TRANSCRICAO | [09:25]-[09:26] |
| RFC-RISCO-02 | docs/RFC.md | Riscos | Vazamento de secret por cliente (já ocorreu) | TRANSCRICAO | [09:22] |
| RFC-OE-01 | docs/RFC.md | Restrição | Email fallback e dashboard visual estão fora de escopo | TRANSCRICAO | [09:37], [09:39]-[09:40] |
| RFC-UNI-01 | docs/RFC.md | Restrição | Entrega unilateral (sistema → cliente); cliente não envia webhooks | TRANSCRICAO | [09:02] |
| RFC-EVENTS-01 | docs/RFC.md | Requisito Funcional | Cliente pode escolher quais status deseja receber | TRANSCRICAO | [09:33] |
| RFC-DUP-01 | docs/RFC.md | Restrição | Cliente pode receber o mesmo evento mais de uma vez (dedup pelo cliente) | TRANSCRICAO | [09:24]-[09:26] |
| RFC-OUTBOX-01 | docs/RFC.md | Decisão | Outbox na mesma transação da mudança de status (snapshot renderizado) | TRANSCRICAO | [09:06]-[09:07] |
| RFC-WORKER-01 | docs/RFC.md | Decisão | Worker separado em processo próprio, polling 2s | TRANSCRICAO | [09:09]-[09:11] |
| RFC-RESIL-01 | docs/RFC.md | Decisão | Resiliência: retry backoff 5x, DLQ, HMAC, at-least-once | TRANSCRICAO | [09:15]-[09:25] |
| RFC-MERCADO-01 | docs/RFC.md | Referência | Padrão de mercado: Stripe e GitHub usam at-least-once + HMAC | TRANSCRICAO | [09:25] |
| RFC-RISCO-03 | docs/RFC.md | Riscos | Worker fora de ordem se múltiplas instâncias iniciadas | TRANSCRICAO | [09:12]-[09:13] |
| FDD-FLUXO-01 | docs/FDD.md | Requisito Funcional | Evento criado na outbox na mesma transação do changeStatus | TRANSCRICAO | [09:40]-[09:41] |
| FDD-FLUXO-02 | docs/FDD.md | Requisito Funcional | Worker processa eventos em polling de 2s | TRANSCRICAO | [09:09]-[09:11] |
| FDD-FLUXO-03 | docs/FDD.md | Requisito Funcional | Retry 5 tentativas 1m/5m/30m/2h/12h, depois DLQ | TRANSCRICAO | [09:15]-[09:17] |
| FDD-FLUXO-04 | docs/FDD.md | Requisito Funcional | Filtro de eventos por status na inserção da outbox | TRANSCRICAO | [09:33]-[09:34] |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /api/v1/webhooks (CRUD criação) | TRANSCRICAO | [09:31] |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | PATCH/DELETE/GET /api/v1/webhooks (CRUD) | TRANSCRICAO | [09:33] |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | GET /api/v1/webhooks/:id/deliveries (histórico) | TRANSCRICAO | [09:34] |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18], [09:35] |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Erros com prefixo WEBHOOK_* | TRANSCRICAO | [09:28]-[09:29] |
| FDD-ERRO-02 | docs/FDD.md | Restrição | URL http rejeitada (https obrigatório) | TRANSCRICAO | [09:23] |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout 10s por tentativa | TRANSCRICAO | [09:42] |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Observabilidade: logs Pino, métricas de entrega, tracing | TRANSCRICAO | [09:29] (padrão existente) |
| FDD-PAYLOAD-01 | docs/FDD.md | Restrição | Payload de saída: event_id, event_type order.status_changed, timestamp, order_id, order_number, from/to_status, customer_id, total_cents (sem items) | TRANSCRICAO | [09:43] |
| FDD-HEADER-01 | docs/FDD.md | Restrição | Headers de saída: X-Event-Id, X-Signature, X-Timestamp, Content-Type, X-Webhook-Id | TRANSCRICAO | [09:44]-[09:45] |
| FDD-DELIVERY-01 | docs/FDD.md | Requisito Funcional | Histórico de entregas: últimas 100, com payload, response e tempo | TRANSCRICAO | [09:34] |
| FDD-CUSTOMER-01 | docs/FDD.md | Restrição | customerId vai no body ou no path, não do JWT | TRANSCRICAO | [09:32] |
| FDD-ESC-01 | docs/FDD.md | Restrição | Fora de escopo: dashboard visual | TRANSCRICAO | [09:39]-[09:40] |
| FDD-ESC-02 | docs/FDD.md | Restrição | Fora de escopo: notificação por email | TRANSCRICAO | [09:37] |
| FDD-ESC-03 | docs/FDD.md | Restrição | Fora de escopo: rate limiting (obs. antes de implementar) | TRANSCRICAO | [09:38]-[09:39] |
| FDD-PAYLOAD-02 | docs/FDD.md | Restrição | Payload sem items (enxuto) | TRANSCRICAO | [09:43] |
| FDD-SEMANTICA-01 | docs/FDD.md | Requisito Funcional | Sucesso = resposta 2xx; timeout 10s = falha | TRANSCRICAO | [09:42] |
| FDD-ERRO-03 | docs/FDD.md | Restrição | WEBHOOK_DLQ_NOT_FOUND; endpoint replay definido | TRANSCRICAO | [09:18] |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Reuso do logger Pino existente | TRANSCRICAO | [09:29] |
| FDD-OBS-03 | docs/FDD.md | Requisito Não Funcional | Auditoria no replay de DLQ | TRANSCRICAO | [09:35]-[09:36] |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Backoff 5x 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15]-[09:17] |
| FDD-RESIL-03 | docs/FDD.md | Requisito Não Funcional | DLQ após 5 falhas com motivo e timestamp | TRANSCRICAO | [09:18] |
| FDD-RESIL-04 | docs/FDD.md | Requisito Não Funcional | At-least-once; X-Event-Id para dedup | TRANSCRICAO | [09:24]-[09:25] |
| FDD-RESIL-05 | docs/FDD.md | Requisito Não Funcional | Filtro de eventos na inserção da outbox | TRANSCRICAO | [09:34] |
| FDD-RISCO-01 | docs/FDD.md | Riscos | Cliente fora do ar por horas | TRANSCRICAO | [09:15]-[09:17] |
| FDD-RISCO-02 | docs/FDD.md | Riscos | Secret vazada | TRANSCRICAO | [09:21]-[09:22] |
| FDD-RISCO-03 | docs/FDD.md | Riscos | Evento duplicado processado | TRANSCRICAO | [09:25]-[09:26] |
| FDD-RISCO-04 | docs/FDD.md | Riscos | Worker derrubado junto com API | TRANSCRICAO | [09:11] |
| FDD-RISCO-05 | docs/FDD.md | Riscos | Acúmulo de eventos pendentes | TRANSCRICAO | [09:08] |
| FDD-INT-01 | docs/FDD.md | Decisão | changeStatus integra outbox na mesma transação | TRANSCRICAO | [09:40]-[09:41] |
| FDD-INT-02 | docs/FDD.md | Decisão | Módulo webhooks segue padrão controller/service/repository | TRANSCRICAO | [09:27] |
| FDD-INT-03 | docs/FDD.md | Decisão | Schema Prisma: webhook, outbox, dead_letter com UUID | TRANSCRICAO | [09:08], [09:51] |
| FDD-INT-04 | docs/FDD.md | Decisão | Script npm run worker como entry separada | TRANSCRICAO | [09:11] |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Notificação automática ao cliente quando status muda | TRANSCRICAO | [09:00]-[09:02] |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | CRUD webhook: cadastro com URL + eventos; sistema gera secret | TRANSCRICAO | [09:31] |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Editar webhook (URL, eventos, ativo) | TRANSCRICAO | [09:33] |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Remover webhook | TRANSCRICAO | [09:33] |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos na inserção da outbox | TRANSCRICAO | [09:33]-[09:34] |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Histórico de entregas (100, payload, response, tempo) | TRANSCRICAO | [09:34] |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Admin reprocessa DLQ com auditoria | TRANSCRICAO | [09:18], [09:35]-[09:36] |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period 24h | TRANSCRICAO | [09:21] |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | X-Event-Id para dedup do cliente | TRANSCRICAO | [09:24]-[09:25] |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Notificação em até 10 segundos | TRANSCRICAO | [09:02] |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Polling de 2s; latência mínima ~2s | TRANSCRICAO | [09:09]-[09:10] |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https | TRANSCRICAO | [09:23] |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB; erro se exceder | TRANSCRICAO | [09:23]-[09:24] |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa | TRANSCRICAO | [09:42] |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Retry 5 tentativas (1m/5m/30m/2h/12h) + DLQ | TRANSCRICAO | [09:15]-[09:18] |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | At-least-once (cliente deduplica) | TRANSCRICAO | [09:24]-[09:26] |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Erros padrão WEBHOOK_* | TRANSCRICAO | [09:28]-[09:29] |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Meta: tempo real < 10s | TRANSCRICAO | [09:02] |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Meta: latência mínima ~2s | TRANSCRICAO | [09:09]-[09:10] |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Meta: prazo fim do trimestre (novembro) | TRANSCRICAO | [09:00], [09:45] |
| PRD-OE-01 | docs/PRD.md | Restrição | Fora de escopo: email de falha | TRANSCRICAO | [09:37] |
| PRD-OE-02 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual | TRANSCRICAO | [09:39]-[09:40] |
| PRD-OE-03 | docs/PRD.md | Restrição | Fora de escopo: rate limiting | TRANSCRICAO | [09:38]-[09:39] |
| PRD-OE-04 | docs/PRD.md | Restrição | Fora de escopo: inbound webhooks | TRANSCRICAO | [09:02] |
| PRD-RISCO | docs/PRD.md | Riscos | Retenção de cliente estratégico (Atlas) com prazo | TRANSCRICAO | [09:00], [09:45]-[09:47] |

**Cobertura:** linhas cobrem os 6 ADRs, o RFC, o FDD e o PRD. O tracker será atualizado quando o README for produzido.