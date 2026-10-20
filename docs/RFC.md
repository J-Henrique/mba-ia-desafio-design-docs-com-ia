# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Autor** | Tech Lead |
| **Status** | Draft — submetido para revisão |
| **Data** | Quinta-feira (data não especificada na transcrição) |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Sofia (Eng. Segurança) |

## Resumo Executivo (TL;DR)

Este documento propõe a adição de um sistema de **webhooks de notificação de pedidos** ao Order Management System (OMS). Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) exigem ser notificados quando o status de seus pedidos muda, em vez de precisar consultar a API repetidamente ([09:00] PM).

A solução proposta usa o **padrão Outbox no MySQL**: eventos são registrados na mesma transação da mudança de status, e um worker separado os entrega via HTTP com garantia at-least-once. A segurança usa **HMAC-SHA256** com secret por endpoint. O conjunto de decisões foi fechado pelo time (tech lead, PM, engenheiros e segurança).

## Contexto e Problema

O OMS gerencia pedidos com uma máquina de estados (PENDING → PAID → PROCESSING → SHIPPED → DELIVERED, além de CANCELLED), conforme `src/modules/orders/order.status.ts`. Quando o status muda, o sistema já executa, dentro de uma transação: atualização da order, inserção em `order_status_history` e controle de estoque (débito ou restituição conforme a transição) — ver `changeStatus` em `src/modules/orders/order.service.ts`.

Hoje, clientes B2B que precisam acompanhar mudanças de status **consultam** a API repetidamente (`GET /orders`). Isso é lento e custoso para ambos os lados. Três clientes de grande porte demandam uma solução de notificação — um deles (Atlas Comercial) condicionou a continuidade do contrato à entrega da feature até o fim do trimestre ([09:00]-[09:02]).

A aplicação **não possui** nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Este vácuo é proposital e é exatamente o que esta RFC visa preencher.

### Requisitos de negócio capturados

- Notificação deve chegar ao cliente em até 10 segundos após a mudança de status ([09:02]).
- A entrega deve ser unilateral (sistema → cliente); cliente não envia webhooks de volta ([09:02]).
- Cliente pode escolher quais status deseja receber ([09:33]).
- É aceitável que o cliente receba o mesmo evento mais de uma vez, desde que consiga identificar duplicatas ([09:24]-[09:26]).

## Proposta Técnica

### Visão geral

A arquitetura tem três camadas:

1. **Outbox na mesma transação**: no método `changeStatus` (`src/modules/orders/order.service.ts`), dentro da transação que já atualiza a order, insere o histórico e controla estoque, adicionar uma linha na tabela `webhook_outbox` com o evento já renderizado (snapshot). Isso garante atomicidade: ou o status muda e o evento existe, ou nada muda ([09:06]-[09:07]).

2. **Worker de entrega**: um processo separado (`src/worker.ts`, rodando via `npm run worker`) consulta a outbox a cada 2 segundos e dispara os eventos via HTTP. Processo isolado para não ser derrubado em reinicializações da API ([09:09]-[09:11]).

3. **Resiliência e segurança**: retry com backoff exponencial (5 tentativas: 1m/5m/30m/2h/12h), DLQ em tabela separada para falhas permanentes, HMAC-SHA256 com secret por endpoint, garantia at-least-once com `X-Event-Id` para deduplicação pelo cliente ([09:15]-[09:25]).

### Decisões arquiteturais fechadas

As decisões foram discutidas e fechadas pelo time. Cada uma está registrada como ADR no pacote:

| Decisão | Resumo | ADR |
|---|---|---|
| Outbox no MySQL | Evento inserido na mesma transação da mudança de status, usando UUID como ID | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Retry + DLQ | 5 tentativas com backoff 1m/5m/30m/2h/12h; DLQ em tabela separada para reprocessamento | [ADR-002](adrs/ADR-002-retry-backoff-dlq.md) |
| HMAC-SHA256 | Assinatura do payload por endpoint; secret única por cadastro; rotação com grace period de 24h | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once | Garantia de entrega com possibilidade de duplicata; cliente deduplica via `X-Event-Id` | [ADR-004](adrs/ADR-004-at-least-once-x-event-id.md) |
| Worker separado | Processo independente com polling de 2s; entry point `src/worker.ts` + `npm run worker` | [ADR-005](adrs/ADR-005-worker-processo-separado-polling.md) |
| Reuso de padrões | Módulo em `src/modules/webhooks`; erros `WEBHOOK_*`; reuso de AppError, Pino, error middleware, requireRole | [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) |

### Integração com o código existente (alto nível)

A alteração crítica é no `changeStatus` do `OrderService`: a inserção na webhook_outbox entra dentro do `$transaction` já existente (detalhamento completo no FDD). O worker reusa a `DATABASE_URL` existente e as mesmas classes de erro, middleware e logger do projeto. O CRUD de configuração de webhooks segue a estrutura `controller → service → repository → routes → schemas` dos demais módulos.

## Alternativas Consideradas

As seguintes abordagens foram discutidas e descartadas antes de fechar a proposta acima:

### 1. Webhook síncrono dentro do service de orders

Disparar a chamada HTTP diretamente no método `changeStatus`, sem outbox ou fila. Descartada porque a transação já é pesada (atualiza order, insere histórico, controla estoque); qualquer cliente lento travaria a mudança de status de outros pedidos. Além disso, se o cliente estivesse fora do ar, não faria sentido dar rollback na mudança de status por causa de uma notificação ([09:03]-[09:06]).

### 2. Redis Streams (fila externa)

Usar Redis Streams como fila de eventos entre a API e o worker. Descartada porque exigiria subir Redis Cluster, adicionando complexidade operacional para um time pequeno. O MySQL já instalado resolve o problema com o padrão Outbox, sem infraestrutura nova ([09:07]).

### 3. Trigger do MySQL para acionar o worker

Usar trigger do banco para avisar o worker quando um novo evento fosse inserido. Descartada porque MySQL não tem mecanismo de notificação nativo (`NOTIFY/LISTEN` do Postgres). Uma trigger só executa SQL; improvisar um mecanismo de notificação seria frágil e complexo ([09:09]).

## Questões em Aberto

Dois pontos foram levantados mas **não foram decididos**, ficando para avaliação futura:

### 1. Rate limiting de envio por cliente

Foi levantado o cenário de um cliente com muitos pedidos mudando de status em um curto intervalo ("50 pedidos mudando em um minuto" — o cliente receberia 50 chamadas seguidas). A equipe decidiu **observar e implementar se virar problema**, sem definição de limite no momento ([09:38]-[09:39]).

### 2. Escalabilidade para múltiplos workers

Enquanto houver um único worker, a ordenação dos eventos é implícita por `order_id`. Foi levantada a possibilidade de escalar para múltiplos workers no futuro; as opções surgidas na discussão foram particionamento por `order_id` ou lock pessimista, mas a decisão foi **adiada** para quando houver necessidade ([09:12]-[09:13]).

> Outras questões levantadas foram **fechadas como fora de escopo**: notificação por email quando o webhook falha ([09:37]) e dashboard visual para clientes ([09:39]-[09:40]) ficam para fases futuras.

## Impacto e Riscos

### Impacto positivo

- Clientes B2B deixam de consultar a API repetitivamente, reduzindo carga no OMS.
- Diferencial competitivo: a feature é requisito para continuidade de contratos existentes (Atlas Comercial).
- Padrão de mercado: Stripe, GitHub e outras plataformas usam webhooks com at-least-once e assinatura HMAC — clientes B2B já conhecem o modelo ([09:25]).

### Riscos

| Risco | Mitigação |
|---|---|
| Cliente não implementa deduplicação e processa evento duplicado | Documentar claramente no portal do desenvolvedor; incluir `X-Event-Id` em toda comunicação ([09:26]) |
| Vazamento de secret de webhook por parte do cliente | Secret por endpoint (limita dano); grace period de 24h para rotação ([09:21]-[09:22]) |
| Worker processa fora de ordem se múltiplas instâncias forem iniciadas | Single-worker é default; ordenação explícita por `created_at`; campo `order_id` para agrupamento ([09:12]-[09:13]) |

### Decisões relacionadas

Todas as decisões arquiteturais estão registradas como ADRs no diretório `docs/adrs/`. Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) para o detalhamento de cada escolha, alternativas e consequências.