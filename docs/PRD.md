# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) precisa notificar clientes B2B quando o status de seus pedidos muda. Três clientes de grande porte — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente receber notificações em tempo real em vez de consultar a API repetidamente (ver TRACKER: `PRD-RF-01`).

Hoje esses clientes verificam mudanças de status fazendo chamadas periódicas ao `GET /orders`, o que é lento e custoso para ambos os lados. Um dos clientes (Atlas Comercial) condicionou a continuidade do contrato à entrega desta feature até o fim do trimestre (ver TRACKER: `PRD-RF-01`).

A feature entrega **webhooks outbound**: a plataforma notifica o cliente quando o status de um pedido muda, seguindo o endereço (URL) que o próprio cliente cadastra (ver TRACKER: `RFC-UNI-01`). A decisão técnica foi tomada pelo time (tech lead, PM, engenheiros e segurança) e documentada nos ADRs (ver `docs/adrs/`).

## 2. Problema e Motivação

**Problema:** clientes B2B precisam reagir em tempo real a mudanças de status de pedidos, mas o único mecanismo disponível é consulta manual/recorrente à API, que:
- Atras a identificação de mudanças (depende da frequência do polling do cliente);
- Gera carga desnecessária no OMS e nos clientes;
- Torna a integração lenta e cara (ver TRACKER: `PRD-RF-01`).

**Motivação adicional:** risco comercial — a Atlas Comercial ameaçou migrar para concorrente se a notificação não for entregue até o fim do trimestre (ver TRACKER: `PRD-OBJ-03`).

**O que motiva a proposta:** o requisito de "tempo real" dos clientes é atingível tecnicamente — qualquer coisa abaixo de 10 segundos já é considerado tempo real (ver TRACKER: `PRD-RNF-01`).

## 3. Público-Alvo e Cenários de Uso

**Público-alvo:**
- Clientes B2B do OMS, representados por usuários autenticados na plataforma (há usuários que representam o cliente; o identificador do cliente é passado no body ou path, não do JWT — ver TRACKER: `FDD-CUSTOMER-01`);
- Equipes de integração/desenvolvimento dos clientes, que consomem os eventos;
- Operação/engenharia interna (via endpoint de reprocessamento de DLQ com role ADMIN — ver TRACKER: `PRD-RF-08`).

**Cenários de uso:**
1. **Cliente acompanha entrega:** quando um pedido do cliente muda de status (ex: `PAID → SHIPPED → DELIVERED`), o cliente é notificado automaticamente em seu endpoint cadastrado (ver TRACKER: `PRD-RF-01`).
2. **Cliente filtra por evento:** um cliente que só quer saber de `SHIPPED` e `DELIVERED` cadastra apenas esses eventos e recebe só eles (ver TRACKER: `PRD-RF-06`).
3. **Operação reprocessa falhas:** um evento que falhou após todas as tentativas vai para a DLQ e é reprocessado manualmente por um admin (ver TRACKER: `PRD-RF-08`).
4. **Cliente valida origem:** o cliente verifica a assinatura HMAC para garantir que o evento veio da plataforma (ver TRACKER: `ADR-003`).

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta | Origem (TRACKER) |
|---|---|---|---|
| Notificar mudança de status em tempo real | Tempo entre mudança de status e chegada ao cliente | **< 10 segundos** | `PRD-RNF-01` |
| Reduzir atraso de propagação mantendo simplicidade | Latência mínima imposta pelo mecanismo (polling) | **~2 segundos** (pior caso aceito) | `PRD-RNF-02` |
| Reter cliente estratégico | Entrega da feature no prazo | **Até fim do trimestre** (novembro) | `PRD-OBJ-03` |

> Metas adicionais (ex: percentual de redução de chamadas, taxa de entrega) não foram definidas; os RNFs 6 e 7 (retry + DLQ, at-least-once) operam como garantias de qualidade de entrega (ver TRACKER: `PRD-RNF-06`, `PRD-RNF-07`).

## 5. Escopo

### 5.1 Incluso

- Registro de eventos de mudança de status em tabela de outbox, na mesma transação da mudança (ver TRACKER: `PRD-RF-01`).
- Entrega via worker em processo separado com polling de 2s (ver TRACKER: `PRD-RNF-02`).
- CRUD de configuração de webhooks por cliente (url, eventos desejados, secret) (ver TRACKER: `PRD-RF-02`).
- Retry com backoff e DLQ com reprocessamento manual (ver TRACKER: `PRD-RNF-06`).
- Segurança: HMAC-SHA256, secret por endpoint, rotação com grace period (ver TRACKER: `ADR-003`).
- Garantia at-least-once com `X-Event-Id` (ver TRACKER: `PRD-RNF-07`).
- Histórico de entregas consultável pelo cliente (ver TRACKER: `PRD-RF-07`).

### 5.2 Fora de Escopo

1. **Notificação por email** quando o webhook falha — descartada; fica para uma próxima fase, após medição de impacto (ver TRACKER: `PRD-OE-01`).
2. **Dashboard visual** para o cliente acompanhar webhooks — descartada; é projeto separado do time de frontend (ver TRACKER: `PRD-OE-02`).
3. **Rate limiting de envio ao cliente** — adiado; a equipe vai observar antes de implementar (ver TRACKER: `PRD-OE-03`).
4. **Inbound webhooks** (cliente enviando eventos para a plataforma) — fora: os clientes querem receber, não enviar (ver TRACKER: `RFC-UNI-01`).

## 6. Requisitos Funcionais

| ID | Requisito | Origem (TRACKER) |
|---|---|---|
| RF-01 | O sistema deve notificar o cliente automaticamente quando o status de um pedido dele mudar | `PRD-RF-01` |
| RF-02 | O cliente deve poder cadastrar um webhook com URL e lista de status desejados; o sistema gera e devolve a secret | `PRD-RF-02` |
| RF-03 | O cliente deve poder editar um webhook (URL, eventos, estado ativo) | `PRD-RF-05` |
| RF-04 | O cliente deve poder remover um webhook | `PRD-RF-05` |
| RF-05 | O cliente deve poder listar os webhooks de um customer | `PRD-RF-05` |
| RF-06 | O filtro de eventos deve ocorrer na inserção da outbox: se nenhum webhook do cliente quer o status, o evento não é registrado | `PRD-RF-06` |
| RF-07 | O cliente deve poder consultar o histórico de entregas (últimas 100, sucesso/falha, payload, response, tempo de resposta) | `PRD-RF-07` |
| RF-08 | Um admin deve poder reprocessar manualmente um evento da DLQ, com registro de auditoria | `PRD-RF-08` |
| RF-09 | O cliente deve poder solicitar rotação da secret; a antiga permanece válida por 24h | `PRD-RF-09` |
| RF-10 | Cada evento deve carregar um `X-Event-Id` único para deduplicação pelo cliente | `PRD-RF-10` |

## 7. Requisitos Não Funcionais

| ID | Requisito | Origem (TRACKER) |
|---|---|---|
| RNF-01 | Notificação deve chegar ao cliente em até 10 segundos | `PRD-RNF-01` |
| RNF-02 | Worker consulta a outbox a cada 2 segundos (latência mínima ~2s) | `PRD-RNF-02` |
| RNF-03 | URL do webhook deve ser obrigatoriamente https; http é rejeitada | `PRD-RNF-03` |
| RNF-04 | Payload limitado a 64KB; excedeu, erro | `PRD-RNF-04` |
| RNF-05 | Timeout de 10 segundos por tentativa de envio | `PRD-RNF-05` |
| RNF-06 | Retry com backoff exponencial: 5 tentativas (1m/5m/30m/2h/12h); após esgotar, encaminhar à DLQ | `PRD-RNF-06` |
| RNF-07 | Garantia at-least-once: evento pode ser entregue mais de uma vez (cliente deduplica via `X-Event-Id`) | `PRD-RNF-07` |
| RNF-08 | Secret única por endpoint de webhook | `PRD-RF-09` |
| RNF-09 | Erros do módulo seguem o padrão `WEBHOOK_*` | `PRD-RNF-09` |

## 8. Decisões e Trade-offs Principais

Decisões de produto/negócio com impacto direto no cliente:

| Decisão | Trade-off | Origem (TRACKER) |
|---|---|---|
| **At-least-once** em vez de exactly-once | Cliente pode receber duplicatas e precisa deduplicar via `X-Event-Id`; em troca, solução simples e padrão de mercado | `PRD-RNF-07` |
| **Outbox em vez de entrega síncrona** | Notificação pode levar até ~2s para sair (janela de polling) em troca de nunca perder evento e não travar mudanças de status | `PRD-RNF-02` |
| **Retry limitado a 5 tentativas (~15h)** em vez de retry indefinido | Eventos de clientes "sumidos" param de ser tentados após ~15h e vão para DLQ, sinalizando problema | `PRD-RNF-06` |
| **Secret por endpoint** em vez de secret global | Cliente gerencia múltiplas secrets; em troca, vazamento de uma não compromete as demais | `PRD-RF-09` |
| **Sem dashboard visual nesta fase** | Clientes integram via API/documentação; painel fica para projeto futuro do time de frontend | `PRD-OE-02` |

As decisões técnicas detalhadas (com alternativas descartadas e consequências) estão nos ADRs: `docs/adrs/ADR-001` a `ADR-006`.

## 9. Dependências

- **Código existente do OMS:** método `changeStatus` em `src/modules/orders/order.service.ts` é o ponto de integração da outbox, dentro da transação já existente (ver TRACKER: `FDD-INT-01`).
- **Máquina de estados:** os eventos de webhook usam os status do enum `OrderStatus` (`src/modules/orders/order.status.ts`).
- **Padrões de infraestrutura:** Prisma/MySQL existentes (sem nova infraestrutura — ver TRACKER: `RFC-ALT-02`); logger Pino, error middleware e `requireRole` reutilizados (ver TRACKER: `ADR-006`).
- **Prazo de negócio:** entrega no fim do trimestre (novembro) para reter a Atlas Comercial (ver TRACKER: `PRD-OBJ-03`).

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação | Origem (TRACKER) |
|---|---|---|---|---|
| Atraso na entrega põe em risco o contrato da Atlas | Média | Alto (perda de cliente) | Prazo estimado de 3 sprints; escopo contido (dashboard e email ficam de fora) | `PRD-RISCO` |
| Cliente do webhook fica fora do ar por horas | Média | Moderado (eventos não entregues) | Retry com backoff de até ~15h + DLQ para reprocessamento | `PRD-RNF-06` |
| Vazamento de secret de webhook (já ocorrido com outro cliente) | Baixa | Alto (comprometimento de integridade) | Secret única por endpoint limita o dano; rotação com grace period de 24h para troca | `PRD-RF-09` |
| Cliente processa evento duplicado | Média | Moderado (inconsistência no cliente) | `X-Event-Id` em todas as entregas; orientação documentada no portal | `PRD-RNF-07` |

## 11. Critérios de Aceitação

1. RF-01 a RF-10 implementados e cobertos por testes de integração no padrão existente (ver seção 12).
2. Uma mudança de status válida gera, na mesma transação, o evento para os webhooks do cliente que escutam aquele status (ver TRACKER: `PRD-RF-01`, `PRD-RF-06`).
3. Transição inválida (ex: `PENDING → SHIPPED`) não gera evento (validação existente de máquina de estados em `src/modules/orders/order.status.ts`).
4. Notificação chega em até 10 segundos (ver TRACKER: `PRD-RNF-01`).
5. Evento sem sucesso após 5 tentativas aparece na DLQ e pode ser reprocessado por admin com auditoria (ver TRACKER: `PRD-RNF-06`, `PRD-RF-08`).
6. Entrega carrega `X-Event-Id`, `X-Signature` (HMAC-SHA256), `X-Timestamp` e `X-Webhook-Id` (ver TRACKER: `FDD-HEADER-01`).
7. Cadastro com URL `http` é recusado (ver TRACKER: `PRD-RNF-03`).

## 12. Estratégia de Testes e Validação

Baseada no padrão de testes existente no projeto (Vitest + Supertest, ver `tests/orders.test.ts` e `tests/auth.test.ts`) e nos pontos definidos na discussão:

- **Testes de integração da transação:** validar que `changeStatus` cria o evento na outbox na mesma transação (sucesso e falha/rollback) — usando o padrão dos testes existentes de pedidos.
- **Testes de contrato do worker:** entrega com sucesso, falha e retry; após 5 tentativas, migração para DLQ; replay da DLQ via endpoint admin.
- **Testes de API (CRUD):** criação, edição, remoção e listagem de webhooks; rejeição de URL não-https; rotação de secret.
- **Testes de segurança:** verificação da assinatura HMAC e da validação de peso de payload (64KB).
- **Validação ponta a ponta:** integração no `order.service` com testes de ponta a ponta (citado no planejamento, ver TRACKER: `PRD-RISCO`).
- **Revisão de segurança:** dois dias úteis de revisão da engenharia de segurança antes do deploy (ver TRACKER: `PRD-RISCO`).