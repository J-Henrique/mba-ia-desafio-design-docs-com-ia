# ADR-004: Garantia At-Least-Once com X-Event-Id

## Status

Aceito

## Contexto

Ao entregar eventos de webhook, é necessário definir qual garantia de entrega o sistema oferece. Foram contrastadas duas abordagens: garantir entrega exata única (**exactly-once**) ou garantir que o evento chegue pelo menos uma vez (**at-least-once**), aceitando duplicatas (ver TRACKER: ADR-004).

Garantir exactly-once exigiria coordenação entre os dois lados e é muito mais complexo; at-least-once resolve a maioria esmagadora dos casos (ver TRACKER: ADR-004). A pergunta central, levantada por Bruno, foi: como o cliente identifica eventos repetidos e consegue deduplicá-los? (ver TRACKER: ADR-004)

## Decisão

Adotar a garantia **at-least-once**: o sistema garante que o evento será enviado até ser entregue com sucesso ou esgotar as tentativas de retry, mas o cliente pode receber o mesmo evento mais de uma vez.

### Mecanismo de deduplicação

Cada evento, no momento em que é inserido na `webhook_outbox`, recebe um **UUID** como identificador único, gerado quando o evento entra na outbox (ver TRACKER: ADR-004) e no padrão UUID já usado nas demais tabelas do projeto (ver TRACKER: ADR-004). Esse UUID é enviado no header `X-Event-Id` em todas as tentativas de entrega daquele evento.

O cliente é responsável por:

1. Extrair o `X-Event-Id` do header de cada requisição recebida.
2. Manter um registro dos IDs que já processou.
3. Ignorar eventos cujo `X-Event-Id` já tenha sido processado.

O formato do header definido (ver TRACKER: ADR-004) inclui:
- `X-Event-Id`: UUID do evento
- `X-Signature`: assinatura HMAC-SHA256
- `X-Timestamp`: timestamp ISO 8601 do envio
- `Content-Type`: `application/json`
- `X-Webhook-Id`: ID do endpoint de webhook que gerou o envio (ver TRACKER: ADR-004)

### Justificativa da escolha

Ao afirmar que "é o padrão de mercado — Stripe faz assim, GitHub faz assim" (ver TRACKER: ADR-004), a equipe reconheceu que exigir that o cliente trate duplicatas é aceitável. O Product Manager comprometeu-se a documentar isso no portal do desenvolvedor para que clientes saibam como implementar a deduplicação (ver TRACKER: ADR-004).

## Alternativas Consideradas

### 1. Exactly-once com confirmação entre os dois lados

Garantir que o evento seja entregue exatamente uma vez, exigindo coordenação bidirecional entre o sistema OMS e o cliente. Descartada porque é muito mais complexa — "exigiria coordenação dos dois lados e fica muito mais complexo" — enquanto at-least-once com event_id resolve 99% dos casos (ver TRACKER: ADR-004).

## Consequências

### Positivas

- **Simplicidade de implementação:** o sistema OMS não precisa de protocolo de confirmação — ele tenta entregar e considera o evento entregue quando o cliente responde com sucesso; caso contrário, retenta conforme a ADR-002.
- **Padrão de mercado:** clientes B2B que já integram com Stripe, GitHub, ou plataformas similares já conhecem e implementam o padrão de idempotência por event_id.
- **Rastreabilidade:** cada evento tem um UUID único que pode ser referenciado no tracker, nos logs e no suporte.

### Negativas

- **Responsabilidade no cliente:** se o cliente não implementar deduplicação, ele pode processar eventos duplicados (ex: dar baixa no mesmo pedido duas vezes).
- **Documentação obrigatória:** o time de produto precisa documentar claramente que o cliente é responsável por deduplicar via `X-Event-Id` (ver TRACKER: ADR-004).
- **Suporte potencial:** clientes que não deduplicarem corretamente podem reportar comportamentos inesperados que exigirão suporte.

### Trade-off explícito

Simplicidade do sistema de origem vs. responsabilidade de deduplicação no cliente. At-least-once é o padrão de mercado porque o custo de implementar exactly-once supera o benefício na esmagadora maioria dos casos (ver TRACKER: ADR-004).