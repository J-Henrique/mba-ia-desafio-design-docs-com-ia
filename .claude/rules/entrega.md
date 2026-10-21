# Regras de Entrega

## Formato
Todos os documentos devem ser entregues em **Markdown (.md)**.

## Escopo Exclusivamente Documental

**Não modificar** o código da aplicação existente em hipótese alguma:

- ❌ `src/` — código fonte
- ❌ `prisma/` — schema, migrations, seed
- ❌ `tests/` — testes existentes
- ❌ Configurações (`tsconfig.json`, `package.json`, `jest.config.*`, etc.)

O código existente serve **exclusivamente como contexto e referência** para produzir os documentos de design.

## Estrutura do Entregável

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
│       └── ... (até 8 ADRs)
├── src/                 (não alterar)
├── prisma/              (não alterar)
├── tests/               (não alterar)
└── ...                  (demais arquivos boilerplate)
```

## Penalidade
Qualquer alteração no código da aplicação viola as regras do desafio. A entrega é puramente documental.
