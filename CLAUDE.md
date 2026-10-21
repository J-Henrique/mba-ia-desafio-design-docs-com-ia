# Projeto: Design Docs Gerados por IA

## Descrição

Transformar a transcrição de uma reunião técnica em um pacote completo de design docs, usando IA como ferramenta principal de produção.

## Cenário

Uma empresa que opera um **Order Management System (OMS)** em produção vai construir uma nova feature: **Sistema de Webhooks de Notificação de Pedidos**.

A decisão técnica já foi tomada em uma reunião entre tech lead, PM, engenheiros e segurança, mas nada foi registrado além da transcrição da call (`TRANSCRICAO.md`).

## Objetivo

Produzir, a partir da transcrição e do código existente, a documentação técnica da feature, em nível acionável o suficiente para o time de engenharia iniciar a implementação.

## Entregáveis

Pacote de documentação a ser produzido:

1. **PRD** — Product Requirement Document da feature (`docs/PRD.md`, 12 seções — ver `.claude/rules/artefatos.md`)
2. **RFC** — Request for Comments com a proposta técnica (`docs/RFC.md`, 8 seções — ver `.claude/rules/artefatos.md`)
3. **FDD** — Feature Design Document da feature (`docs/FDD.md`, 12 seções — ver `.claude/rules/artefatos.md`)
4. **ADRs** — 5 a 8 Architecture Decision Records em `docs/adrs/ADR-NNN-*.md` — ver `.claude/rules/artefatos.md`
5. **Tracker** — Rastreabilidade em `docs/TRACKER.md` — tabela ligando cada item à transcrição ou código (mín. 80% cobertura)
6. **README** — Documentação do processo de produção (`README.md`, 6 seções — ver `.claude/rules/artefatos.md`)

### Resumo dos artefatos

| Documento | Altura | Pergunta |
|---|---|---|
| PRD | Produto/negócio | Por que e o quê? |
| RFC | Arquitetura | Como pretendemos resolver? |
| ADRs | Decisão pontual | Por que decidimos assim? |
| FDD | Implementação | Como construir? |
| Tracker | Transversal | De onde veio cada coisa? |

> Fronteiras completas em `.claude/rules/artefatos.md`.

## Formato de Entrega

- **Formato:** Markdown (.md)
- **Escopo:** Exclusivamente documental — **não modificar** o código da aplicação (`src/`, `prisma/`, `tests/`, configurações)
- **Código existente:** serve apenas como contexto e referência para os documentos
- **Entrega:** fork do repositório base, repositório público no GitHub

### Estrutura do entregável

```
.
├── README.md            (substituído — documentação do processo)
├── TRANSCRICAO.md       (não alterar)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── ADR-001-*.md
│       ├── ADR-002-*.md
│       └── ... (até 8)
├── src/                 (não alterar)
├── prisma/              (não alterar)
├── tests/               (não alterar)
└── ...                  (demais arquivos boilerplate)
```

## Stack

- **Runtime:** Node.js + TypeScript
- **Banco:** MySQL via Prisma ORM
- **Logger:** Pino
- **Autenticação:** middleware `requireRole`
- **Tratamento de erros:** middleware centralizado + classes de erro próprias + padrão de códigos de erro
- **Módulos existentes:** Autenticação, Usuários, Clientes, Produtos, Pedidos

## Arquitetura Existente

- Order Management System funcional com ciclo de vida de pedidos controlado por **máquina de estados**
- **Controle transacional de estoque** na atualização de status
- **Auditoria de mudanças de status** registrada
- **Não há** mecanismo de notificação externa, eventos, filas ou webhooks — esse vácuo é proposital e é o que a feature discutida na reunião visa preencher

### Componentes-chave do código (mapear via IA)

- Estrutura modular da aplicação
- Máquina de estados do pedido
- Transação `changeStatus`
- Classes de erro e padrão de códigos de erro
- Middleware `requireRole`
- Error middleware centralizado
- Logger Pino

## Papéis

- **Humano (maestro):** definir o que precisa ser feito, formular bons prompts, revisar criticamente o que a IA entrega, corrigir e refinar até o resultado ficar consistente.
- **IA:** ferramenta principal de produção — ler código, analisar transcrição, estruturar documentos, gerar conteúdo final.

> Critérios de aceite completos em `.claude/memory/projeto.md` (seção "Critérios de Aceite").
