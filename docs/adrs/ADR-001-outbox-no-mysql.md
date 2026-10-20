# ADR-001: Uso do Padrão Outbox no MySQL

## Status

Aceito

## Contexto

O sistema precisa notificar clientes B2B quando o status dos pedidos deles muda. A opção de disparar a chamada HTTP de forma síncrona, dentro do serviço de pedidos, foi descartada porque a transação de mudança de status já é pesada — atualiza a order, insere em `order_status_history` e decrementa o estoque dos produtos — e uma chamada HTTP no meio dela faria qualquer cliente lento travar a mudança de status de outros pedidos ([09:04] Bruno). Em caso de cliente fora do ar, não haveria como dar rollback da mudança de status por causa de uma falha de notificação ([09:04] Bruno).

A aplicação atual não possui filas, eventos ou mecanismo de notificação externa. A solução precisa garantir que, se a transação principal commitar, o evento exista; se der rollback, o evento não exista — sem inconsistência possível ([09:06] Diego).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: quando o status do pedido muda, dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir também uma linha na tabela `webhook_outbox` com o evento ([09:06] Diego). Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego).

Cada linha da outbox carrega o **payload já renderizado (snapshot)** no momento da inserção, refletindo o estado do pedido quando o status mudou ([09:51]-[09:52] Larissa, Diego). O identificador do evento é **UUID**, seguindo o padrão das demais tabelas do projeto ([09:51] Larissa).

### Decisões complementares registradas

- A tabela tem índice nos campos `status` (pendente, processando, falhou, entregue) e `created_at`, para o worker ler apenas eventos pendentes em lotes pequenos ([09:08] Diego).
- Linhas entregues são arquivadas após ~30 dias, fora do escopo desta feature ([09:08] Diego).
- Enquanto houver um único worker, a ordenação é implícita por `order_id`; **não há garantia de ordering global**, e isso é uma limitação conhecida e aceita ([09:12]-[09:13] Diego, Larissa).

O ponto de integração no código é o método `changeStatus` em `src/modules/orders/order.service.ts`, que já executa toda a mudança de status dentro de um `prisma.$transaction` — a inserção na outbox entra na mesma transação, garantindo atomicidade com a mudança de status ([09:40] Bruno, [09:41] Diego).

## Alternativas Consideradas

### 1. Webhook síncrono dentro do service de orders

Disparar a chamada HTTP diretamente no `changeStatus`. Descartada porque a transação já é pesada; um cliente lento travaria a mudança de status de outros pedidos, e não faria sentido dar rollback da mudança de status se o cliente estiver fora do ar ([09:04] Bruno). A alternativa foi apoiada pela rejeição explícita de Larissa ([09:03]-[09:04]) e Diego ([09:06]).

### 2. Redis Streams (fila externa)

Usar Redis Streams como fila de eventos. Descartada porque exigiria subir infraestrutura adicional; para um time pequeno, subir Redis Cluster seria overengineering, enquanto o MySQL já existente resolve o problema ([09:07] Larissa, Diego).

### 3. Trigger do MySQL para reatividade

Usar trigger do banco para avisar o worker. Descartada porque MySQL não tem mecanismo de notificação nativo tipo `NOTIFY/LISTEN` do Postgres; trigger só executa SQL, não notifica processo externo, e improvisar isso seria frágil ([09:09] Diego).

## Consequências

### Positivas

- **Consistência garantida:** se a transação do status commitar, o evento está registrado; se der rollback, o evento some junto — não há caminho para "status mudou e evento não saiu" ([09:06] Diego).
- **Zero infraestrutura nova:** reutiliza o MySQL já existente, sem fila ou broker adicional ([09:07] Diego).
- **Latência aceitável:** o requisito de negócio é "tempo real" abaixo de 10 segundos; o polling de 2s do worker (ADR-005) atende com folga ([09:09]-[09:10]).

### Negativas

- **Latência mínima de ~2s** no pior caso, pois o evento só é disparado no próximo ciclo do worker ([09:10] Larissa).
- **Ordering limitado:** sem garantia de ordenação global; apenas implícita por `order_id` e desde que se mantenha um único worker. Escalar para múltiplos workers exige particionamento por `order_id` ou lock pessimista, decisão adiada para o futuro ([09:12]-[09:13] Diego).
- **Linhas acumuladas na outbox** exigem arquivamento/retenção periódico (fora do escopo atual) ([09:08] Diego).

### Trade-off explícito

Ganhamos consistência forte (nada se perde) e simplicidade operacional (sem infra nova) ao custo de uma latência mínima de ~2s e da ausência de ordering global — perdas aceitas explicitamente pela equipe na reunião ([09:10], [09:13]).