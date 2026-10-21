# Design Docs Gerados por IA — Processo de Produção

## Sobre o desafio

O desafio consistia em transformar a transcrição de uma reunião técnica (55 minutos, 5 participantes) em um pacote completo de documentação de design para uma nova feature: **Sistema de Webhooks de Notificação de Pedidos** em um Order Management System (OMS) existente. A transcrição da reunião decidiu os rumos técnicos mas nada além disso havia sido registrado.

A entrega é puramente documental: PRD, RFC, FDD, 6 ADRs, um Tracker de rastreabilidade e este README. A utilização de uma ferramenta de IA para a geração destes documentos possibilitou que o desafio fosse concluído muito mais rápido, com muito mais precisão e reduzindo a carga cognitiva drasticamente, uma vez que seria muito desgastante resumir manualmente todo o conteúdo para só então gerar os documentos.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (CLI)** | Agente principal. Leitura da transcrição e do código, mapa da arquitetura existente, geração dos documentos e revisão iterativa. |
| **Deepseek V4 Flash (modelo)** | Motor da análise: interpretação da transcrição, extração de decisões/requisitos/alternativas, formulação das seções técnicas. |
| **Comandos customizados (skills)** | `caveman` para reduzir a quantidade de output e economizar tokens; regras de contexto em `.claude/rules/` e memória em `.claude/memory/` para persistir as convenções do desafio entre sessões. |

## Workflow adotado

O trabalho seguiu a ordem sugerida pelo enunciado, mas com uma camada extra de governança: **regras de rastreabilidade definidas antes de qualquer documento ser escrito**.

1. **Setup de contexto** — extraídas para `CLAUDE.md` e `.claude/rules/` as regras do desafio: formato de cada artefato, fronteiras entre documentos (PRD/RFC/FDD/ADRs/Tracker), regras de rastreabilidade e critérios de aceite.
2. **Contextualização** — leitura completa da `TRANSCRICAO.md` e do código (`src/modules/orders/order.service.ts`, `order.status.ts`, `shared/errors`, middlewares, `schema.prisma`). Mapa mental da arquitetura e das 6 decisões principais.
3. **ADRs primeiro** — produzidos os 6 ADRs (outbox, retry/DLQ, HMAC, at-least-once, worker separado, reuso de padrões).
4. **RFC** — consolidação da proposta técnica em cima das decisões, com alternativas descartadas e questões em aberto.
5. **FDD** — o documento mais técnico: fluxos, contratos, matriz de erros, resiliência, observabilidade e a seção de integração com o código existente.
6. **PRD** — consolidação de produto (problema, escopo, requisitos, métricas), produzido por último entre os grandes documentos.
7. **Tracker** — montado **em paralelo**: cada documento pronto gerava linhas no tracker, mapeando item → origem na transcrição (timestamp) ou no código (caminho de arquivo). O tracker foi a ferramenta de checagem contra alucinação — se um item não tinha origem, era removido ou sinalizado como derivação.
8. **README** — este documento, por último.

## Prompts customizados

Os prompts abaixo são cópias fiéis das mensagens que eu escrevi durante a interação com a IA e que geraram os documentos e correções mais relevantes:

**Prompt para entendimento do contexto (após a leitura transversal da transcrição):**

```
primeiro me explique de forma resumida do que se trata a transcrição
lembre-se de que eu não li ela e não tenho contexto do assunto tratado na reunião. Explique de uma forma concisa e considere que não tenho conhecimentos sobre projeto backend, então preciso de uma apoio para entender cada passo que definirmos daqui pra frente
```

**Prompt para validação de rastreabilidade e correção do tracker:**

```
fiz uma correção no tracker pois estava quebrando a formatação da tabela
sobre os timestamps:
- faça uma varredura em todos timestamps identificados e garanta que todos os items anotados estão contidos no tracker e vice-versa. 
- garanta que as referências estão corretas
- garanta que não existem informações inventadas
- é importante essa validação dos dois documentos para podermos garantir o rastreio de onde cada fato veio e é um requisito do desafio
- ainda não remova o timestamps. Somente faça essas correções por enquanto
```

## Iterações e ajustes

A geração não saiu correta de primeira — houveram pelo menos **3 iterações principais de correção**:

**Iteração 1 — alternativas inventadas nos ADRs.** A primeira versão dos ADRs incluiu alternativas "plausíveis de mercado" que **não foram discutidas na reunião**: framework de webhook externo (Svix) no ADR-006, "JWT no header" e "Basic Auth" no ADR-003, "RabbitMQ" no ADR-005. A correção exigiu revisar cada alternativa contra a transcrição e remover as que não tinham origem, deixando apenas as realmente discutidas (webhook síncrono, Redis Streams, trigger MySQL, secret global, exactly-once, worker embutido, logger separado).

**Iteração 2 — RFC com recomendações e riscos não rastreáveis.** A primeira versão do RFC incluía "recomendações para revisão" (implementar rate limiting antes do launch, campo `shard_key` na outbox) que eram sugestões do autor, não decisões da reunião — removidas para cumprir a regra de rastreabilidade. Além disso, nomes de participantes foram removidos da narrativa: o documento passou a narrar os fatos de forma impessoal (mantendo os items do documento TRACKER como referência), para ser legível por quem não participou da call.

**Iteração 3 — migração de timestamps para o tracker.** Originalmente os documentos citavam `[09:xx] Nome` inline. Após revisão, decidiu-se que os timestamps viveriam **apenas no TRACKER**, e os documentos passaram a referenciar `(ver TRACKER: ID)`. Isso exigiu criar IDs granulares no tracker (PRD-RF-01, FDD-CONTRATO-03, RFC-ALT-02, etc.), substituir todas as citações e validar programaticamente que todos os IDs referenciados tinham linha correspondente.

**Iteração 4 — corrupção no PRD durante automação.** Um script de substituição em massa de timestamps no PRD gerou referências quebradas (`PRD-RF-02-PRD-RF-05`, `[09:44]-PRD-OBJ-03`). O PRD foi reconstruído, semântica por semântica, com mapeamento correto de cada fato para seu ID.

## Como navegar a entrega

Caminhos dos arquivos entregues:

| Arquivo | Conteúdo |
|---|---|
| `docs/PRD.md` | Produto: problema, público, escopo, 10 requisitos funcionais, 9 não funcionais, métricas, riscos |
| `docs/RFC.md` | Proposta técnica: TL;DR, contexto, visão geral, 3 alternativas descartadas, 2 questões em aberto |
| `docs/FDD.md` | Implementação: fluxos (outbox/worker/retry/DLQ), 6 endpoints, matriz de erros `WEBHOOK_*`, resiliência, observabilidade, integração com 11 arquivos reais |
| `docs/TRACKER.md` | Rastreabilidade: 88 linhas ligando cada item à transcrição (`[hh:mm] Nome`) ou ao código (caminho de arquivo) |
| `docs/adrs/ADR-001` a `ADR-006` | Decisões arquiteturais em formato MADR |
| `README.md` | Este documento (processo de produção) |

**Ordem sugerida de leitura:**

1. `docs/TRACKER.md` — entende da onde cada coisa veio
2. `docs/adrs/` — as decisões principais primeiro
3. `docs/RFC.md` — proposta consolidada
4. `docs/FDD.md` — o "como construir"
5. `docs/PRD.md` — o "porquê e o quê", por último (consolida tudo)
