# ADR-0002: Valores monetários em `BigInt` centavos

- **Data**: 2026-04-18
- **Status**: Aceito
- **Decisor(es)**: Roberto Chagas
- **Etapa do plan.md**: transversal (afeta Et 1+)

## Contexto

Sistema processa cobranças, repasses, multas e relatórios contábeis. Erros de
arredondamento em valores monetários geram diferenças que se acumulam ao longo
de meses, comprometem auditoria e podem gerar inconsistências fiscais.

JavaScript `Number` é IEEE 754 (64-bit float) — incapaz de representar exatamente
valores como `0.1 + 0.2`. Prisma `Decimal` resolve precisão mas força conversões
constantes e introduz bibliotecas client-side. `Float` em Postgres tem o mesmo
problema do Number.

## Decisão

**Todos os valores monetários são armazenados e manipulados como `BigInt` em
centavos** (R$ 12,34 → `1234n`).

- Schema Prisma: `BigInt` (mapeado para `bigint` no Postgres).
- TypeScript: `bigint` end-to-end no backend.
- DTO/API: serializar como `string` (JSON não suporta `bigint` nativo).
- Frontend: receber `string`, converter para `bigint` via `BigInt()`, formatar
  para exibição com helper único.

## Alternativas consideradas

- **`Number` (float)**: descartado — perde precisão.
- **`Decimal` (Prisma)**: descartado — força biblioteca extra no front, lento
  em aritmética, conversões implícitas mascaram bugs.
- **`String` decimal**: descartado — todo cálculo precisa parser, sem garantia
  de tipo em runtime.

## Consequências

### Positivas

- Aritmética exata, sem arredondamento.
- Type safety em tempo de compilação.
- `bigint` é nativo do JS/TS desde ES2020.

### Negativas / trade-offs aceitos

- `JSON.stringify` não serializa `bigint` — precisa de interceptor global.
- Operações entre `bigint` e `number` lançam `TypeError` — força disciplina.
- Helper único de formatação obrigatório (sem `toFixed(2)`).

### Neutras

- Reports financeiros precisam dividir por 100 ao gerar PDF/Excel.

## Referências

- `backend/src/common/interceptors/` (serialização de bigint para string)
- `frontend/src/utils/format-money.ts` (formatação)
- `backend/test/fixtures/cenarios-repasse.ts` (fonte de verdade contábil)
