# ADR-002: Política de Retry com Backoff Exponencial e DLQ

## Status

Aceito

## Contexto

Quando o worker tenta entregar um webhook e o cliente está fora do ar ou responde com erro, é necessário decidir o que fazer com aquele evento. Tentar uma vez só e desistir não é aceitável porque clientes podem ter indisponibilidades temporárias (manutenção, queda de rede, etc.). Tentar infinitamente também não é desejável porque eventos falhos acumulariam na outbox e não haveria sinalização clara de que o cliente tem um problema permanente ([09:14]-[09:15] Diego).

Além disso, eventos que falham definitivamente precisam ser preservados para auditoria e reprocessamento manual, sem poluir a tabela de eventos pendentes ([09:15] Diego).

Foram discutidos diferentes números de tentativas (3 vs. 5) e onde armazenar eventos que esgotaram as tentativas ([09:15]-[09:18]).

## Decisão

Adotar **5 tentativas** com backoff exponencial progressivo e, após o esgotamento, mover o evento para uma **tabela separada de Dead Letter Queue (DLQ)**.

### Progressão do backoff

As tentativas seguem os seguintes intervalos a partir da primeira falha:

| Tentativa | Intervalo | Tempo acumulado |
|---|---|---|
| 1ª | 1 minuto | 1 minuto |
| 2ª | 5 minutos | 6 minutos |
| 3ª | 30 minutos | 36 minutos |
| 4ª | 2 horas | ~2h36min |
| 5ª (última) | 12 horas | ~14h36min |

Entre a primeira falha e a última tentativa decorrem aproximadamente **15 horas** ([09:17] Diego). Esse total foi aceito pelo Product Manager como razoável — se um cliente fica offline por 15 horas, o problema é dele ([09:17] Marcos).

### DLQ — Dead Letter Queue

Eventos que falham em todas as 5 tentativas são movidos para a tabela `webhook_dead_letter`, contendo:

- Payload do evento
- Motivo da falha
- Timestamp

A DLQ é separada da tabela `webhook_outbox` para manter a leitura da outbox principal limpa e servir como evidência para debug e reprocessamento ([09:18] Diego).

### Reprocessamento manual

Um endpoint administrativo permite recolocar eventos da DLQ de volta na outbox como pendentes:

```
POST /admin/webhooks/dead-letter/:id/replay
```

Acesso restrito a usuários com role `ADMIN`, utilizando o middleware `requireRole('ADMIN')` já existente na aplicação. A operação deve ser logada para auditoria ([09:18] Diego, [09:35]-[09:36] Sofia).

### Timeout da chamada HTTP

O timeout para cada tentativa de entrega é de **10 segundos**. Clientes que não responderem em 10s são tratados como falha e acionam o mecanismo de retry ([09:42] Diego, Sofia).

## Alternativas Consideradas

### 1. Três tentativas (backoff mais agressivo)

Proposto por Bruno ([09:16]). Descartada porque 3 tentativas cobririam apenas uma janela curta de indisponibilidade (~30 minutos); há registros de clientes com manutenção planejada de 2 horas. As 5 tentativas cobrem aproximadamente 15 horas, o que já inclui janelas longas de manutenção ([09:16] Diego, Larissa).

### 2. Retry indefinido com backoff

Descartada porque eventos ficariam "pendurados" para sempre se o cliente desaparecer permanentemente, sem dar visibilidade ao time de operação de que há um problema ativo ([09:15] Diego).

### 3. Marcar como "failed" na própria outbox (sem DLQ separada)

Descartada porque poluiria a tabela da outbox principal — manter a DLQ em tabela separada mantém a leitura da outbox mais rápida e organiza o debug ([09:18] Diego, apoiado por Bruno).

## Consequências

### Positivas

- **Cobertura de indisponibilidades longas:** as ~15 horas de janela de retry cobrem a grande maioria dos cenários de manutenção ou queda temporária de clientes.
- **Separação de responsabilidades:** a outbox contém apenas eventos pendentes ou entregues; eventos que falharam definitivamente vão para a DLQ, facilitando monitoramento e debug.
- **Reprocessamento auditável:** o endpoint admin com role `ADMIN` e log obrigatório garante rastreabilidade de intervenções manuais.

### Negativas

- **Atraso de até 15 horas** para que um evento seja considerado falha definitiva e movido para DLQ.
- **Complexidade adicional** do mecanismo de retry com backoff, cálculo de próxima tentativa, e movimentação entre tabelas.
- **DLQ requer endpoint e tratamento de auditoria**, aumentando o escopo de implementação.

### Trade-off explícito

Mais tentativas = maior chance de entrega mesmo em janelas longas de indisponibilidade, ao custo de maior latência para declarar falha definitiva (~15h) e maior complexidade de implementação. As 5 tentativas foram o ponto de equilíbrio aceito pelo time ([09:16]-[09:17]).