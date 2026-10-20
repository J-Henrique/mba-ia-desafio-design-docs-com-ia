# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

## Status

Aceito

## Contexto

Os eventos de webhook são enviados para endpoints HTTP fora da infraestrutura da empresa (servidores dos clientes B2B). O cliente precisa conseguir validar que a requisição veio realmente do sistema OMS e que o payload não foi adulterado durante o trânsito ([09:19] Sofia).

A reunião discutiu também o risco de vazamento de secrets: se uma única secret é usada para todos os endpoints de todos os clientes, o vazamento de uma comprometeria todos os webhooks ([09:21] Sofia). E, quando uma secret é trocada, o cliente precisa de tempo para migrar os sistemas dele sem perder entregas ([09:21] Diego, Sofia).

## Decisão

Adotar **HMAC-SHA256** sobre o corpo do request, com **secret única por endpoint de webhook**, armazenada na tabela de configuração.

### Geração e armazenamento

- A secret é gerada pelo sistema e devolvida ao cliente no momento da criação do webhook ([09:31] Marcos).
- A secret é armazenada no banco associada ao registro do webhook — a tabela de configuração armazena `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno).

### Header de assinatura

Cada requisição de webhook enviada inclui o header `X-Signature` contendo a assinatura HMAC-SHA256 do corpo JSON, codificada em hexadecimal.

### Rotação com grace period

O cliente pode solicitar uma nova secret via API. Quando a rotação ocorre:

1. A nova secret é ativada imediatamente para assinar novos envios.
2. A **secret anterior permanece válida por 24 horas** (grace period), permitindo que o cliente migre os sistemas dele sem perder a capacidade de validar eventos recebidos nesse intervalo.
3. Após as 24 horas, a secret antiga é descartada ([09:21]-[09:22] Sofia, Diego).

### TLS obrigatório

A URL do endpoint deve ser obrigatoriamente `https`. URLs com `http` são recusadas na validação (via schema Zod) no momento do cadastro ([09:23] Sofia). Essa validação acontece no schema do endpoint, não na camada de segurança propriamente dita.

## Alternativas Consideradas

### 1. Secret global única para todos os webhooks

Usar uma única secret HMAC compartilhada por todos os clientes. Descartada por risco de segurança: se um cliente vazar a secret em logs de aplicação (já ocorreu antes, [09:22] Diego), todos os endpoints ficam comprometidos ([09:21] Sofia).

## Consequências

### Positivas

- **Assinatura e integridade:** o cliente consegue verificar que o payload veio do sistema OMS e não foi alterado no meio do caminho.
- **Isolamento de vazamento:** se a secret de um endpoint é comprometida, os demais endpoints não são afetados.
- **Migração segura:** o grace period de 24h permite que o cliente troque a secret sem perder a capacidade de validar entregas durante a transição.

### Negativas

- **Gerenciamento de secrets:** o sistema precisa armazenar secrets em texto plano no banco (já que precisa assinar com elas), o que aumenta a superfície de risco no banco de dados.
- **Rotação adiciona complexidade:** lógica de validade paralela (24h) e expiração da secret antiga precisam ser implementadas e testadas.
- **Cliente precisa implementar HMAC:** a verificação não é feita pelo sistema, então o cliente precisa ter capacidade técnica de verificar HMAC-SHA256.

### Trade-off explícito

Secret por endpoint é mais seguro que secret global, mas exige do cliente gerenciar uma secret para cada cadastro de webhook que ele possui — complexidade aceitável pelo padrão de mercado ([09:21] Sofia).