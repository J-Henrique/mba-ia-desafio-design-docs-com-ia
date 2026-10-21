# ADR-001: Uso do Padrão Outbox no MySQL

## Status

Aceito

## Contexto

O sistema precisa notificar clientes B2B quando o status dos pedidos deles muda. A opção de disparar a chamada HTTP de forma síncrona, dentro do serviço de pedidos, foi descartada porque a transação de mudança de status já é pesada — atualiza a order, insere em `order_status_history` e decrementa o estoque dos produtos — e uma chamada HTTP no meio dela faria qualquer cliente lento travar a mudança de status de outros pedidos (ver TRACKER: ADR-001). Em caso de cliente fora do ar, não haveria como dar rollback da mudança de status por causa de uma falha de notificação (ver TRACKER: ADR-001).

A aplicação atual não possui filas, eventos ou mecanismo de notificação externa. A solução precisa garantir que, se a transação principal commitar, o evento exista; se der rollback, o evento não exista — sem inconsistência possível (ver TRACKER: ADR-001).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: quando o status do pedido muda, dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir também uma linha na tabela `webhook_outbox` com o evento (ver TRACKER: ADR-001). Um worker separado lê essa tabela e dispara as chamadas HTTP (ver TRACKER: ADR-001).

Cada linha da outbox carrega o **payload já renderizado (snapshot)** no momento da inserção, refletindo o estado do pedido quando o status mudou (ver TRACKER: ADR-001). O identificador do evento é **UUID**, seguindo o padrão das demais tabelas do projeto (ver TRACKER: ADR-001).

### Decisões complementares registradas

- A tabela tem índice nos campos `status` (pendente, processando, falhou, entregue) e `created_at`, para o worker ler apenas eventos pendentes em lotes pequenos (ver TRACKER: ADR-001).
- Linhas entregues são arquivadas após ~30 dias, fora do escopo desta feature (ver TRACKER: ADR-001).
- Enquanto houver um único worker, a ordenação é implícita por `order_id`; **não há garantia de ordering global**, e isso é uma limitação conhecida e aceita (ver TRACKER: ADR-001).

O ponto de integração no código é o método `changeStatus` em `src/modules/orders/order.service.ts`, que já executa toda a mudança de status dentro de um `prisma.$transaction` — a inserção na outbox entra na mesma transação, garantindo atomicidade com a mudança de status (ver TRACKER: ADR-001).

## Alternativas Consideradas

### 1. Webhook síncrono dentro do service de orders

Disparar a chamada HTTP diretamente no `changeStatus`. Descartada porque a transação já é pesada; um cliente lento travaria a mudança de status de outros pedidos, e não faria sentido dar rollback da mudança de status se o cliente estiver fora do ar (ver TRACKER: ADR-001). A alternativa foi apoiada pela rejeição explícita de Larissa e Diego (ver TRACKER: ADR-001).

### 2. Redis Streams (fila externa)

Usar Redis Streams como fila de eventos. Descartada porque exigiria subir infraestrutura adicional; para um time pequeno, subir Redis Cluster seria overengineering, enquanto o MySQL já existente resolve o problema (ver TRACKER: ADR-001).

### 3. Trigger do MySQL para reatividade

Usar trigger do banco para avisar o worker. Descartada porque MySQL não tem mecanismo de notificação nativo tipo `NOTIFY/LISTEN` do Postgres; trigger só executa SQL, não notifica processo externo, e improvisar isso seria frágil (ver TRACKER: ADR-001).

## Consequências

### Positivas

- **Consistência garantida:** se a transação do status commitar, o evento está registrado; se der rollback, o evento some junto — não há caminho para "status mudou e evento não saiu" (ver TRACKER: ADR-001).
- **Zero infraestrutura nova:** reutiliza o MySQL já existente, sem fila ou broker adicional (ver TRACKER: ADR-001).
- **Latência aceitável:** o requisito de negócio é "tempo real" abaixo de 10 segundos; o polling de 2s do worker (ADR-005) atende com folga (ver TRACKER: ADR-001).

### Negativas

- **Latência mínima de ~2s** no pior caso, pois o evento só é disparado no próximo ciclo do worker (ver TRACKER: ADR-001).
- **Ordering limitado:** sem garantia de ordenação global; apenas implícita por `order_id` e desde que se mantenha um único worker. Escalar para múltiplos workers exige particionamento por `order_id` ou lock pessimista, decisão adiada para o futuro (ver TRACKER: ADR-001).
- **Linhas acumuladas na outbox** exigem arquivamento/retenção periódico (fora do escopo atual) (ver TRACKER: ADR-001).

### Trade-off explícito

Ganhamos consistência forte (nada se perde) e simplicidade operacional (sem infra nova) ao custo de uma latência mínima de ~2s e da ausência de ordering global — perdas aceitas explicitamente pela equipe (ver TRACKER: ADR-001).