# ADR-005: Worker em Processo Separado com Polling

## Status

Aceito

## Contexto

O padrão outbox (ADR-001) define que eventos de mudança de status são registrados em uma tabela `webhook_outbox`, mas quem lê e dispara as chamadas HTTP para os clientes precisa ser definido. Foram discutidas duas questões principais: o mecanismo de leitura da outbox (event-driven vs. polling) e o ciclo de vida do processo que executa essa leitura (ver TRACKER: ADR-005).

O requisito de negócio é que a notificação chegue ao cliente em **até 10 segundos** após a mudança de status (ver TRACKER: ADR-005). Qualquer solução precisa atender a esse limite com folga.

Um ponto crítico levantado por Diego foi que o worker **precisa rodar em processo separado** da API principal: se a API reiniciar (por deploy, crash, etc.), o worker não pode ser derrubado junto (ver TRACKER: ADR-005).

## Decisão

Implementar um **worker em processo separado que executa polling na tabela `webhook_outbox` a cada 2 segundos**.

### Arquitetura do worker

- O worker é uma **entry point independente** da aplicação, localizada em `src/worker.ts`, análoga ao `src/server.ts` existente (ver TRACKER: ADR-005).
- É executado via comando próprio: `npm run worker` (ver TRACKER: ADR-005).
- Conecta ao **mesmo banco de dados**, usando o **mesmo Prisma Client**, mas em uma instância separada porque `PrismaClient` é por processo (ver TRACKER: ADR-005).

### Ciclo de polling

1. A cada **2 segundos**, o worker consulta a `webhook_outbox` pelos eventos pendentes mais antigos, em lotes pequenos.
2. Para cada evento, tenta entregar via HTTP para o endpoint configurado.
3. Se a entrega for bem-sucedida (HTTP 2xx), marca o evento como "entregue".
4. Se falhar, registra a falha e agenda uma nova tentativa conforme a política de retry (ADR-002) ou move para a DLQ se esgotaram-se as tentativas.

### Decisões de ordenação

- Enquanto existir **um único worker**, a ordenação dos eventos respeita `created_at` dentro de cada `order_id`, sem garantia de ordenação global (ver TRACKER: ADR-005).
- O particionamento para múltiplos workers no futuro (por `order_id` ou lock pessimista) é uma decisão adiada (ver TRACKER: ADR-005).

## Alternativas Consideradas

### 1. Processo único com a API (worker embutido)

Executar o worker na mesma instância da API. Descartada porque se a API reiniciar (deploy, crash, etc.), o worker perde o ciclo de processamento — exatamente o problema que a decisão de processo separado visa evitar (ver TRACKER: ADR-005).

### 2. Notificação via trigger do MySQL

Usar trigger do MySQL para avisar o worker quando um novo evento for inserido. Descartada porque MySQL não tem listener nativo (`NOTIFY/LISTEN` do Postgres), e improvisar um mecanismo seria frágil e complexo (ver TRACKER: ADR-005).

### 3. Fila externa com consumo em tempo real (Redis Streams)

Subir uma infraestrutura de fila separada que notificasse o worker sob demanda. Descartada por overengineering para o tamanho do time — a latência de 2s do polling já atende o requisito de "abaixo de 10 segundos" com margem confortável (ver TRACKER: ADR-005).

## Consequências

### Positivas

- **Isolamento de ciclo de vida:** a API e o worker são processos independentes — reinicializações de um não afetam o outro (ver TRACKER: ADR-005).
- **Latência dentro do requisito:** 2s de polling deixa margem de ~8s para a entrega HTTP vs. o limite de 10s definido pelo negócio (ver TRACKER: ADR-005).
- **Simplicidade operacional:** não requer infraestrutura de fila adicional — apenas a entry point nova e o comando npm.

### Negativas

- **Latência mínima de ~2s** mesmo que o evento pudesse ser processado antes — é o pior caso aceito pela equipe (ver TRACKER: ADR-005).
- **Dois processos para gerenciar:** a operação precisa monitorar e manter tanto a API quanto o worker separadamente.
- **Conexão extra ao banco:** cada instância do worker mantém seu próprio `PrismaClient`, consumindo uma conexão adicional do pool do MySQL.
- **Escalabilidade limitada:** múltiplos workers em paralelo exigiriam coordenação (particionamento ou lock) — decisão adiada (ver TRACKER: ADR-005).

### Trade-off explícito

Simplicidade operacional (polling 2s, sem fila externa) vs. latência mínima de 2s e necessidade de gerenciar um processo extra. O trade-off foi aceito porque o requisito de latência (<10s) é confortavelmente atendido e o time é pequeno demais para justificar uma fila externa (ver TRACKER: ADR-005).