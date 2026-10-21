# Definição dos Artefatos

Os documentos **não se repetem**: cada um opera em uma altura diferente. Conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

## PRD — Product Requirement Document

| Atributo | Valor |
|---|---|
| **Papel** | Problema, público, escopo e métricas de sucesso |
| **Altura** | Produto / negócio |
| **Pergunta** | *Por que e o quê?* |
| **Conteúdo** | Contexto de negócio, personas, definição do problema, escopo (dentro/fora), objetivos e métricas de sucesso, personas/usuários afetados |
| **O que NÃO vai** | Detalhes de implementação, diagramas técnicos, contratos de API |

### Seções obrigatórias do PRD (`docs/PRD.md`)

1. Resumo e contexto da feature
2. Problema e motivação
3. Público-alvo e cenários de uso
4. Objetivos e métricas de sucesso
5. Escopo: incluso e fora de escopo
6. Requisitos funcionais
7. Requisitos não funcionais
8. Decisões e trade-offs principais
9. Dependências
10. Riscos e mitigação
11. Critérios de aceitação
12. Estratégia de testes e validação

**Fora de escopo** deve listar **pelo menos 2 itens** descartados ou adiados na reunião.

## RFC — Request for Comments

| Atributo | Valor |
|---|---|
| **Papel** | Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto |
| **Altura** | Arquitetura |
| **Pergunta** | *Como pretendemos resolver, e o que ainda está em aberto?* |
| **Conteúdo** | Abordagem geral da solução, alternativas consideradas (prós/contras), questões em aberto para revisão, recomendações iniciais |
| **Extensão** | 2 a 4 páginas. Conciso, fala em **decisão**. |
| **O que NÃO vai** | Detalhe de implementação (isso é do FDD), especificação exaustiva de contratos |

### Seções obrigatórias do RFC (`docs/RFC.md`)

1. Metadados: autor, status, data, revisores (usar participantes da reunião)
2. Resumo executivo (TL;DR) da proposta
3. Contexto e problema
4. Proposta técnica — visão geral da solução, sem detalhe de implementação
5. Alternativas consideradas — **pelo menos 2** alternativas reais discutidas e descartadas na reunião, cada uma com trade-off que levou ao descarte
6. Questões em aberto — **pelo menos 2** pontos levantados na reunião e não decididos ou adiados
7. Impacto e riscos
8. Decisões relacionadas — links para os ADRs correspondentes

**Não duplicar** detalhamento do FDD. RFC responde "o que propomos e por quê".

## ADRs — Architecture Decision Records

| Atributo | Valor |
|---|---|
| **Papel** | Cada decisão arquitetural isolada, com contexto e consequências |
| **Altura** | Decisão pontual |
| **Pergunta** | *Por que decidimos exatamente assim?* |
| **Conteúdo** | Contexto, decisão, alternativas consideradas, consequências (positivas e negativas), status |
| **Quantidade** | 5 a 8 ADRs |
| **Formato** | ADR NNN: Título (ex: ADR-001: Uso de Webhook com Retry Exponencial) |

### Regras para ADRs (`docs/adrs/ADR-NNN-titulo-em-kebab-case.md`)

**Formato:** MADR (ou variante padrão) com no mínimo:
- Status
- Contexto
- Decisão
- Alternativas Consideradas (pelo menos 1 alternativa real ou plausível)
- Consequências (positivas e negativas, com trade-off explícito)

**Cobertura obrigatória:** mínimo 5 das 6 decisões principais:

1. Padrão Outbox no MySQL
2. Política de retry com backoff e DLQ
3. Autenticação HMAC-SHA256 com secret por endpoint
4. Garantia at-least-once com X-Event-Id
5. Worker em processo separado em polling
6. Reuso dos padrões existentes do projeto

**Pelo menos 1 ADR** deve referenciar arquivos, módulos ou padrões do código existente.

Decisões técnicas secundárias (payload, timeouts, headers) podem ficar apenas no FDD ou virar ADRs adicionais.

## FDD — Feature Design Document

| Atributo | Valor |
|---|---|
| **Papel** | Especificação de implementação: fluxos, contratos, erros, integração com o código |
| **Altura** | Implementação |
| **Pergunta** | *Como construir, em detalhe?* |
| **Conteúdo** | Fluxos detalhados, contratos de API (endpoints, payloads, headers, status codes), tabela de erros, integração com código existente, exemplos de uso |
| **O que NÃO vai** | Discussão de alternativas (isso é do RFC), justificativa de decisões (isso é dos ADRs) |

### Seções obrigatórias do FDD (`docs/FDD.md`)

1. Contexto e motivação técnica
2. Objetivos técnicos
3. Escopo e exclusões
4. Fluxos detalhados: criação do evento na outbox, processamento pelo worker, retry, DLQ
5. Contratos públicos: endpoints HTTP com payloads de exemplo, headers, status codes, semântica
6. Matriz de erros previstos com códigos no padrão `WEBHOOK_*`
7. Estratégias de resiliência: timeouts, retries, backoff, fallback
8. Observabilidade: métricas, logs, tracing
9. Dependências e compatibilidade
10. Critérios de aceite técnicos
11. Riscos e mitigação
12. **Integração com o sistema existente** (seção obrigatória específica do desafio): nomear **pelo menos 4 caminhos de arquivo reais** do código base e descrever como o módulo de webhooks se integra com cada um (ex: como `changeStatus` é estendido, como classes de erro existentes são reutilizadas)

## README — Documentação do Processo

| Atributo | Valor |
|---|---|
| **Papel** | Documentar a jornada de produção dos artefatos |
| **Altura** | Meta / processo |
| **Local** | `README.md` (raiz do repositório — substitui o original) |

### Estrutura obrigatória do README (`README.md`)

1. **Sobre o desafio:** 1 a 2 parágrafos descrevendo a tarefa
2. **Ferramentas de IA utilizadas:** lista com papel de cada uma
3. **Workflow adotado:** ordem de produção dos documentos, como organizou a interação com IA
4. **Prompts customizados:** pelo menos 2 prompts relevantes em blocos de código
5. **Iterações e ajustes:** momentos em que a IA gerou algo errado ou superficial e você corrigiu. Quantas iterações principais até o resultado final
6. **Como navegar a entrega:** caminho dos arquivos entregues e ordem sugerida de leitura

## Tracker — Rastreabilidade

| Atributo | Valor |
|---|---|
| **Papel** | Rastreabilidade de cada item ao código ou à transcrição |
| **Altura** | Transversal |
| **Pergunta** | *De onde veio cada coisa?* |
| **Conteúdo** | Tabela markdown ligando cada requisito/decisão/detalhe à sua origem |

### Regras do Tracker (`docs/TRACKER.md`)

**Formato obrigatório da tabela:**

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |

**Campos:**
- **ID:** identificador único (ex: `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002`)
- **Documento:** arquivo de origem (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-*.md`)
- **Tipo:** classificação do item (Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, etc.)
- **Conteúdo (resumo):** descrição de uma linha
- **Fonte:** `TRANSCRICAO` ou `CODIGO`
- **Localização:** timestamp + falante se TRANSCRICAO (ex: `[09:17] Diego`); caminho de arquivo se CODIGO (ex: `src/modules/orders/order.service.ts`)

**Cobertura mínima:** 80% dos itens identificáveis nos documentos devem ter linha no tracker.

**Função:** referência cruzada contra alucinações — garante alinhamento entre documentação, discussão e código.

## Em uma frase

- **RFC** propõe e abre para revisão
- **ADRs** registram cada decisão fechada
- **FDD** detalha como construir
