# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) precisa notificar clientes B2B quando o status de seus pedidos muda. Três clientes de grande porte — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente receber notificações em tempo real em vez de consultar a API repetidamente ([09:00]).

Hoje esses clientes verificam mudanças de status fazendo chamadas periódicas ao `GET /orders`, o que é lento e custoso para ambos os lados. Um dos clientes (Atlas Comercial) condicionou a continuidade do contrato à entrega desta feature até o fim do trimestre ([09:00]-[09:02]).

A feature entrega **webhooks outbound**: a plataforma notifica o cliente quando o status de um pedido muda, seguindo o endereço (URL) que o próprio cliente cadastra ([09:02]). A decisão técnica foi tomada pelo time (tech lead, PM, engenheiros e segurança) e documentada nos ADRs (ver `docs/adrs/`).

## 2. Problema e Motivação

**Problema:** clientes B2B precisam reagir em tempo real a mudanças de status de pedidos, mas o único mecanismo disponível é consulta manual/recorrente à API, que:
- Atras a identificação de mudanças (depende da frequência do polling do cliente);
- Gera carga desnecessária no OMS e nos clientes;
- Torna a integração lenta e cara ([09:00]).

**Motivação adicional:** risco comercial — a Atlas Comercial ameaçou migrar para concorrente se a notificação não for entregue até o fim do trimestre ([09:00]-[09:02]).

**O que motiva a proposta:** o requisito de "tempo real" dos clientes é atingível tecnicamente — qualquer coisa abaixo de 10 segundos já é considerado tempo real ([09:02]).

## 3. Público-Alvo e Cenários de Uso

**Público-alvo:**
- Clientes B2B do OMS, representados por usuários autenticados na plataforma (há usuários que representam o cliente, [09:32]);
- Equipes de integração/desenvolvimento dos clientes, que consomem os eventos;
- Operação/engenharia interna (via endpoint de reprocessamento de DLQ, [09:18], [09:35]).

**Cenários de uso:**
1. **Cliente acompanha entrega:** quando um pedido do cliente muda de status (ex: `PAID → SHIPPED → DELIVERED`), o cliente é notificado automaticamente em seu endpoint cadastrado ([09:00], [09:43]).
2. **Cliente filtra por evento:** um cliente que só quer saber de `SHIPPED` e `DELIVERED` cadastra apenas esses eventos e recebe só eles ([09:33]).
3. **Operação reprocessa falhas:** um evento que falhou após todas as tentativas vai para a DLQ e é reprocessado manualmente por um admin ([09:18]).
4. **Cliente valida origem:** o cliente verifica a assinatura HMAC para garantir que o evento veio da plataforma ([09:19]-[09:20]).

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta | Origem |
|---|---|---|---|
| Notificar mudança de status em tempo real | Tempo entre mudança de status e chegada ao cliente | **< 10 segundos** | [09:02] |
| Reduzir atraso de propagação mantendo simplicidade | Latência mínima imposta pelo mecanismo (polling) | **~2 segundos** (pior caso aceito) | [09:09]-[09:10] |
| Reter cliente estratégico | Entrega da feature no prazo | **Até fim do trimestre** (novembro) | [09:00], [09:45] |

> Metas adicionais (ex: percentual de redução de chamadas, taxa de entrega) não foram definidas na discussão; os RNFs 6 e 7 (retry + DLQ, at-least-once) operam como garantias de qualidade de entrega ([09:15]-[09:26]).

## 5. Escopo

### 5.1 Incluso

- Registro de eventos de mudança de status em tabela de outbox, na mesma transação da mudança ([09:06]).
- Entrega via worker em processo separado com polling de 2s ([09:09]-[09:11]).
- CRUD de configuração de webhooks por cliente (url, eventos desejados, secret) ([09:31]-[09:33]).
- Retry com backoff e DLQ com reprocessamento manual ([09:15]-[09:18]).
- Segurança: HMAC-SHA256, secret por endpoint, rotação com grace period ([09:19]-[09:22]).
- Garantia at-least-once com `X-Event-Id` ([09:24]-[09:26]).
- Histórico de entregas consultável pelo cliente ([09:34]).

### 5.2 Fora de Escopo

1. **Notificação por email** quando o webhook falha — descartada; fica para uma próxima fase, após medição de impacto ([09:37]).
2. **Dashboard visual** para o cliente acompanhar webhooks — descartada; é projeto separado do time de frontend ([09:39]-[09:40]).
3. **Rate limiting de envio ao cliente** — adiado; a equipe vai observar antes de implementar ([09:38]-[09:39]).
4. **Inbound webhooks** (cliente enviando eventos para a plataforma) — fora: os clientes querem receber, não enviar ([09:02]).

## 6. Requisitos Funcionais

| ID | Requisito | Origem |
|---|---|---|
| RF-01 | O sistema deve notificar o cliente automaticamente quando o status de um pedido dele mudar | [09:00]-[09:02] |
| RF-02 | O cliente deve poder cadastrar um webhook com URL e lista de status desejados; o sistema gera e devolve a secret | [09:31] |
| RF-03 | O cliente deve poder editar um webhook (URL, eventos, estado ativo) | [09:33] |
| RF-04 | O cliente deve poder remover um webhook | [09:33] |
| RF-05 | O cliente deve poder listar os webhooks de um customer | [09:33] |
| RF-06 | O filtro de eventos deve ocorrer na inserção da outbox: se nenhum webhook do cliente quer o status, o evento não é registrado | [09:33]-[09:34] |
| RF-07 | O cliente deve poder consultar o histórico de entregas (últimas 100, sucesso/falha, payload, response, tempo de resposta) | [09:34] |
| RF-08 | Um admin deve poder reprocessar manualmente um evento da DLQ, com registro de auditoria | [09:18], [09:35]-[09:36] |
| RF-09 | O cliente deve poder solicitar rotação da secret; a antiga permanece válida por 24h | [09:21] |
| RF-10 | Cada evento deve carregar um `X-Event-Id` único para deduplicação pelo cliente | [09:24]-[09:25] |

## 7. Requisitos Não Funcionais

| ID | Requisito | Origem |
|---|---|---|
| RNF-01 | Notificação deve chegar ao cliente em até 10 segundos | [09:02] |
| RNF-02 | Worker consulta a outbox a cada 2 segundos (latência mínima ~2s) | [09:09]-[09:10] |
| RNF-03 | URL do webhook deve ser obrigatoriamente https; http é rejeitada | [09:23] |
| RNF-04 | Payload limitado a 64KB; excedeu, erro | [09:23]-[09:24] |
| RNF-05 | Timeout de 10 segundos por tentativa de envio | [09:42] |
| RNF-06 | Retry com backoff exponencial: 5 tentativas (1m/5m/30m/2h/12h); após esgotar, encaminhar à DLQ | [09:15]-[09:18] |
| RNF-07 | Garantia at-least-once: evento pode ser entregue mais de uma vez (cliente deduplica via `X-Event-Id`) | [09:24]-[09:26] |
| RNF-08 | Secret única por endpoint de webhook | [09:21] |
| RNF-09 | Erros do módulo seguem o padrão `WEBHOOK_*` | [09:28]-[09:29] |

## 8. Decisões e Trade-offs Principais

Decisões de produto/negócio com impacto direto no cliente:

| Decisão | Trade-off | Origem |
|---|---|---|
| **At-least-once** em vez de exactly-once | Cliente pode receber duplicatas e precisa deduplicar via `X-Event-Id`; em troca, solução simples e padrão de mercado | [09:24]-[09:26] |
| **Outbox em vez de entrega síncrona** | Notificação pode levar até ~2s para sair (janela de polling) em troca de nunca perder evento e não travar mudanças de status | [09:04]-[09:10] |
| **Retry limitado a 5 tentativas (~15h)** em vez de retry indefinido | Eventos de clientes "sumidos" param de ser tentados após ~15h e vão para DLQ, sinalizando problema | [09:15]-[09:17] |
| **Secret por endpoint** em vez de secret global | Cliente gerencia múltiplas secrets; em troca, vazamento de uma não compromete as demais | [09:21] |
| **Sem dashboard visual nesta fase** | Clientes integram via API/documentação; painel fica para projeto futuro do time de frontend | [09:39]-[09:40] |

As decisões técnicas detalhadas (com alternativas descartadas e consequências) estão nos ADRs: `docs/adrs/ADR-001` a `ADR-006`.

## 9. Dependências

- **Código existente do OMS:** método `changeStatus` em `src/modules/orders/order.service.ts` é o ponto de integração da outbox, dentro da transação já existente ([09:40]-[09:41]).
- **Máquina de estados:** os eventos de webhook usam os status do enum `OrderStatus` (`src/modules/orders/order.status.ts`).
- **Padrões de infraestrutura:** Prisma/MySQL existentes (sem nova infraestrutura, [09:07]); logger Pino, error middleware e `requireRole` reutilizados ([09:28]-[09:36]).
- **Prazo de negócio:** entrega no fim do trimestre (novembro) para reter a Atlas Comercial ([09:00], [09:45]).

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação | Origem |
|---|---|---|---|---|
| Atraso na entrega põe em risco o contrato da Atlas | Média | Alto (perda de cliente) | Prazo estimado de 3 sprints; escopo contido (dashboard e email ficam de fora) | [09:00], [09:45]-[09:47] |
| Cliente do webhook fica fora do ar por horas | Média | Moderado (eventos não entregues) | Retry com backoff de até ~15h + DLQ para reprocessamento | [09:14]-[09:18] |
| Vazamento de secret de webhook (já ocorrido com outro cliente) | Baixa | Alto (comprometimento de integridade) | Secret única por endpoint limita o dano; rotação com grace period de 24h para troca | [09:21]-[09:22] |
| Cliente processa evento duplicado | Média | Moderado (inconsistência no cliente) | `X-Event-Id` em todas as entregas; orientação documentada no portal | [09:24]-[09:26] |

## 11. Critérios de Aceitação

1. RF-01 a RF-10 implementados e cobertos por testes de integração no padrão existente (ver seção 12).
2. Uma mudança de status válida gera, na mesma transação, o evento para os webhooks do cliente que escutam aquele status (RF-01, RF-06; [09:06], [09:33]).
3. Transição inválida (ex: `PENDING → SHIPPED`) não gera evento ([09:38] — validação existente de máquina de estados).
4. Notificação chega em até 10 segundos (RNF-01; [09:02]).
5. Evento sem sucesso após 5 tentativas aparece na DLQ e pode ser reprocessado por admin com auditoria (RF-08, RNF-06; [09:15]-[09:18]).
6. Entrega carrega `X-Event-Id`, `X-Signature` (HMAC-SHA256), `X-Timestamp` e `X-Webhook-Id` (RF-10, RNF-08; [09:44]-[09:45]).
7. Cadastro com URL `http` é recusado (RNF-03; [09:23]).

## 12. Estratégia de Testes e Validação

Baseada no padrão de testes existente no projeto (Vitest + Supertest, ver `tests/orders.test.ts` e `tests/auth.test.ts`) e nos pontos definidos na discussão ([09:46]):

- **Testes de integração da transação:** validar que `changeStatus` cria o evento na outbox na mesma transação (sucesso e falha/rollback) — usando o padrão dos testes existentes de pedidos.
- **Testes de contrato do worker:** entrega com sucesso, falha e retry; após 5 tentativas, migração para DLQ; replay da DLQ via endpoint admin.
- **Testes de API (CRUD):** criação, edição, remoção e listagem de webhooks; rejeição de URL não-https; rotação de secret.
- **Testes de segurança:** verificação da assinatura HMAC e da validação de peso de payload (64KB).
- **Validação ponta a ponta:** integração no `order.service` com testes de ponta a ponta (citado no planejamento, [09:46]).
- **Revisão de segurança:** dois dias úteis de revisão da engenharia de segurança antes do deploy ([09:46], [09:49]).